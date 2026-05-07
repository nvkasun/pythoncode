#!/usr/bin/env python3

import csv
import json
import re
import sys
from pathlib import Path

INPUT_FILE = "Results.csv"
ACCOUNT_MAP_FILE = "AccountMap.csv"
OUTPUT_FILE = "Final_Records.csv"


def clean_text(value):
    if value is None:
        return ""

    return (
        str(value)
        .replace("\ufeff", "")
        .replace("\xa0", " ")
        .replace("\r", "")
        .replace("\n", "")
        .strip()
    )


def normalize_account_number(value):
    """
    Convert account number to digits only for matching.

    Example:
    5716-0086-7023 -> 571600867023
    571600867023   -> 571600867023
    """
    value = clean_text(value)
    return re.sub(r"\D", "", value)


def normalize_env(value):
    value = clean_text(value).lower()

    mapping = {
        "production": "prod",
        "prod": "prod",
        "pre-production": "preprod",
        "pre_prod": "preprod",
        "pre-prod": "preprod",
        "preprod": "preprod",
        "uat": "uat",
        "dev": "dev",
        "development": "dev",
        "sit": "sit",
    }

    return mapping.get(value, value)


def extract_env_from_resource_name(resource_name):
    name = clean_text(resource_name).lower()

    if not name:
        return ""

    # Order matters: preprod before prod
    env_patterns = [
        ("preprod", r"(^|[-_])preprod($|[-_])"),
        ("prod", r"(^|[-_])prod($|[-_])"),
        ("uat", r"(^|[-_])uat($|[-_])"),
        ("dev", r"(^|[-_])dev($|[-_])"),
        ("sit", r"(^|[-_])sit($|[-_])"),
    ]

    for env, pattern in env_patterns:
        if re.search(pattern, name):
            return env

    return ""


def extract_env_from_tags(tags_value):
    tags_value = clean_text(tags_value)

    if not tags_value or tags_value == "-":
        return ""

    cleaned = tags_value.replace('\\"', '"')

    try:
        tags = json.loads(cleaned)
    except json.JSONDecodeError:
        match = re.search(
            r'"key"\s*:\s*"Environment".*?"value"\s*:\s*"([^"]+)"',
            cleaned,
            re.IGNORECASE,
        )

        if not match:
            match = re.search(
                r'"tag"\s*:\s*"Environment=([^"]+)"',
                cleaned,
                re.IGNORECASE,
            )

        if match:
            return normalize_env(match.group(1))

        return ""

    if not isinstance(tags, list):
        return ""

    for tag in tags:
        if not isinstance(tag, dict):
            continue

        key = clean_text(tag.get("key", "")).lower()

        if key == "environment":
            return normalize_env(tag.get("value", ""))

    return ""


def decide_env(resource_name, tags_value):
    """
    Priority:
    1. tags.Environment
    2. resourceName
    3. Not Found
    """
    env_from_tags = extract_env_from_tags(tags_value)
    env_from_name = extract_env_from_resource_name(resource_name)

    if env_from_tags:
        return env_from_tags

    if env_from_name:
        return env_from_name

    return "Not Found"


def normalize_headers(fieldnames):
    return [clean_text(header) for header in fieldnames]


def load_account_map():
    map_path = Path(ACCOUNT_MAP_FILE)

    if not map_path.exists():
        print(f"Warning: {ACCOUNT_MAP_FILE} not found in {Path.cwd()}")
        return {}

    account_map = {}

    with map_path.open("r", newline="", encoding="utf-8-sig") as mapfile:
        reader = csv.DictReader(mapfile)

        if not reader.fieldnames:
            print(f"Warning: {ACCOUNT_MAP_FILE} has no headers.")
            return {}

        reader.fieldnames = normalize_headers(reader.fieldnames)

        print("AccountMap.csv headers found:")
        for header in reader.fieldnames:
            print(f"  - [{header}]")

        required_columns = [
            "Account Name",
            "Account Number",
            "Account owner",
        ]

        missing = [col for col in required_columns if col not in reader.fieldnames]

        if missing:
            print("Warning: Missing column(s) in AccountMap.csv:")
            for col in missing:
                print(f"  - {col}")
            return {}

        for row in reader:
            account_number_raw = clean_text(row.get("Account Number", ""))
            account_number_key = normalize_account_number(account_number_raw)

            if not account_number_key:
                continue

            account_map[account_number_key] = {
                "Account Name": clean_text(row.get("Account Name", "")) or "Not Found",
                "Account Number": account_number_raw or "Not Found",
                "Account owner": clean_text(row.get("Account owner", "")) or "Not Found",
            }

    print(f"Loaded account mappings: {len(account_map)}")
    return account_map


def main():
    input_path = Path(INPUT_FILE)
    output_path = Path(OUTPUT_FILE)

    if not input_path.exists():
        print(f"Error: {INPUT_FILE} not found in {Path.cwd()}")
        sys.exit(1)

    account_map = load_account_map()

    with input_path.open("r", newline="", encoding="utf-8-sig") as infile:
        reader = csv.DictReader(infile)

        if not reader.fieldnames:
            print("Error: Results.csv has no headers.")
            sys.exit(1)

        reader.fieldnames = normalize_headers(reader.fieldnames)

        print("Results.csv headers found:")
        for header in reader.fieldnames:
            print(f"  - [{header}]")

        required_columns = [
            "accountId",
            "resourceName",
            "tags",
        ]

        missing_columns = [
            col for col in required_columns
            if col not in reader.fieldnames
        ]

        if missing_columns:
            print("Error: Missing required column(s) in Results.csv:")
            for col in missing_columns:
                print(f"  - {col}")
            sys.exit(1)

        matched_count = 0
        unmatched_count = 0

        with output_path.open("w", newline="", encoding="utf-8") as outfile:
            writer = csv.writer(outfile)

            writer.writerow([
                "Sl No",
                "Env",
                "Account Name",
                "Account Number",
                "Account owner",
            ])

            count = 1

            for row in reader:
                account_id_raw = clean_text(row.get("accountId", ""))
                account_id_key = normalize_account_number(account_id_raw)

                resource_name = clean_text(row.get("resourceName", ""))
                tags_value = clean_text(row.get("tags", ""))

                if not account_id_key:
                    continue

                env = decide_env(resource_name, tags_value)

                account_details = account_map.get(account_id_key)

                if account_details:
                    matched_count += 1
                else:
                    unmatched_count += 1
                    account_details = {
                        "Account Name": "Not Found",
                        "Account Number": account_id_raw,
                        "Account owner": "Not Found",
                    }

                writer.writerow([
                    count,
                    env,
                    account_details["Account Name"],
                    account_details["Account Number"],
                    account_details["Account owner"],
                ])

                count += 1

    print(f"Done! Created {OUTPUT_FILE}")
    print(f"Total records written: {count - 1}")
    print(f"Matched account records: {matched_count}")
    print(f"Unmatched account records: {unmatched_count}")


if __name__ == "__main__":
    main()

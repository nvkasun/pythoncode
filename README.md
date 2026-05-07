#!/usr/bin/env python3

import csv
import json
import re
import sys
from pathlib import Path

# =====================================================
# Configuration
# =====================================================
INPUT_FILE = "Results.csv"
ACCOUNT_MAP_FILE = "AccountMap.csv"
OUTPUT_FILE = "Final_Records.csv"


def normalize_env(value: str) -> str:
    if not value:
        return ""

    value = value.strip().lower()

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


def extract_env_from_resource_name(resource_name: str) -> str:
    if not resource_name:
        return ""

    name = resource_name.lower().strip()

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


def extract_env_from_tags(tags_value: str) -> str:
    if not tags_value:
        return ""

    tags_value = tags_value.strip()

    if tags_value in ["", "-"]:
        return ""

    cleaned = tags_value.replace('\\"', '"')

    try:
        tags = json.loads(cleaned)
    except json.JSONDecodeError:
        # Fallback if tags JSON parsing fails
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

        key = str(tag.get("key", "")).strip().lower()

        if key == "environment":
            value = str(tag.get("value", "")).strip()
            return normalize_env(value)

    return ""


def decide_env(resource_name: str, tags_value: str) -> str:
    """
    Priority:
      1. Environment tag
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


def load_account_map() -> dict:
    """
    Load account mapping from AccountMap.csv.

    Expected headers:
      Account Name,Account Number,Account owner
    """

    map_path = Path(ACCOUNT_MAP_FILE)

    if not map_path.exists():
        print(f"Warning: {ACCOUNT_MAP_FILE} not found.")
        print("Account details will be marked as 'Not Found'.")
        return {}

    account_map = {}

    with map_path.open("r", newline="", encoding="utf-8-sig") as mapfile:
        reader = csv.DictReader(mapfile)

        if not reader.fieldnames:
            print(f"Warning: {ACCOUNT_MAP_FILE} has no headers.")
            return {}

        reader.fieldnames = [header.strip() for header in reader.fieldnames]

        required_columns = [
            "Account Name",
            "Account Number",
            "Account owner",
        ]

        missing = [col for col in required_columns if col not in reader.fieldnames]

        if missing:
            print(f"Warning: Missing column(s) in {ACCOUNT_MAP_FILE}:")
            for col in missing:
                print(f"  - {col}")
            return {}

        for row in reader:
            account_number = row.get("Account Number", "").strip()

            if not account_number:
                continue

            account_map[account_number] = {
                "Account Name": row.get("Account Name", "").strip() or "Not Found",
                "Account Number": account_number,
                "Account owner": row.get("Account owner", "").strip() or "Not Found",
            }

    return account_map


def main():
    input_path = Path(INPUT_FILE)
    output_path = Path(OUTPUT_FILE)

    if not input_path.exists():
        print(f"Error: {INPUT_FILE} not found in current folder: {Path.cwd()}")
        sys.exit(1)

    account_map = load_account_map()

    try:
        with input_path.open("r", newline="", encoding="utf-8-sig") as infile:
            reader = csv.DictReader(infile)

            if not reader.fieldnames:
                print("Error: Results.csv has no headers.")
                sys.exit(1)

            reader.fieldnames = [header.strip() for header in reader.fieldnames]

            required_columns = ["accountId", "resourceName", "tags"]

            missing_columns = [
                col for col in required_columns
                if col not in reader.fieldnames
            ]

            if missing_columns:
                print("Error: Missing required column(s) in Results.csv:")
                for col in missing_columns:
                    print(f"  - {col}")

                print("\nAvailable columns:")
                for header in reader.fieldnames:
                    print(f"  - {header}")

                sys.exit(1)

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
                    account_id = row.get("accountId", "").strip()
                    resource_name = row.get("resourceName", "").strip()
                    tags_value = row.get("tags", "").strip()

                    if not account_id or account_id == "-":
                        continue

                    env = decide_env(resource_name, tags_value)

                    account_details = account_map.get(account_id, {
                        "Account Name": "Not Found",
                        "Account Number": account_id,
                        "Account owner": "Not Found",
                    })

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

    except csv.Error as e:
        print(f"CSV error: {e}")
        sys.exit(1)

    except Exception as e:
        print(f"Unexpected error: {e}")
        sys.exit(1)


if __name__ == "__main__":
    main()

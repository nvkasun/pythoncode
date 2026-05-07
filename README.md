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


def normalize_header(value):
    return clean_text(value).lower().replace(" ", "").replace("_", "").replace("-", "")


def normalize_account_number(value):
    """
    5716-0086-7023 -> 571600867023
    571600867023   -> 571600867023
    """
    return re.sub(r"\D", "", clean_text(value))


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


def parse_tags(tags_value):
    """
    Convert tags column into a dictionary:
      Environment -> dev
      BusinessUnitOwner -> MichaelJoelDeGroot
      Name -> cluster-xxx-dev
    """

    tags_value = clean_text(tags_value)

    if not tags_value or tags_value == "-":
        return {}

    cleaned = tags_value.replace('\\"', '"')

    try:
        tags = json.loads(cleaned)
    except json.JSONDecodeError:
        tag_dict = {}

        # Fallback regex for key/value pairs
        matches = re.findall(
            r'"key"\s*:\s*"([^"]+)"\s*,\s*"tag"\s*:\s*"[^"]*"\s*,\s*"value"\s*:\s*"([^"]*)"',
            cleaned,
            re.IGNORECASE,
        )

        if not matches:
            matches = re.findall(
                r'"value"\s*:\s*"([^"]*)"\s*,\s*"key"\s*:\s*"([^"]+)"',
                cleaned,
                re.IGNORECASE,
            )
            matches = [(key, value) for value, key in matches]

        for key, value in matches:
            tag_dict[clean_text(key)] = clean_text(value)

        return tag_dict

    tag_dict = {}

    if isinstance(tags, list):
        for tag in tags:
            if not isinstance(tag, dict):
                continue

            key = clean_text(tag.get("key", ""))
            value = clean_text(tag.get("value", ""))

            if key:
                tag_dict[key] = value

    return tag_dict


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


def decide_env(resource_name, tag_dict):
    """
    Priority:
      1. tags -> Environment
      2. resourceName
      3. Not Found
    """

    env_from_tags = normalize_env(tag_dict.get("Environment", ""))

    if env_from_tags:
        return env_from_tags

    env_from_name = extract_env_from_resource_name(resource_name)

    if env_from_name:
        return env_from_name

    return "Not Found"


def get_column_name(fieldnames, possible_names):
    """
    Flexible header matcher.
    Example:
      Account Number
      account number
      AccountNumber
      account_id
    """

    normalized_lookup = {
        normalize_header(header): header
        for header in fieldnames
    }

    for name in possible_names:
        normalized_name = normalize_header(name)
        if normalized_name in normalized_lookup:
            return normalized_lookup[normalized_name]

    return None


def load_account_map():
    """
    Loads AccountMap.csv.

    Supports headers like:
      Account Name,Account Number,Account owner
      accountName,accountId,accountOwner
      Name,AccountId,Owner
    """

    map_path = Path(ACCOUNT_MAP_FILE)

    if not map_path.exists():
        print(f"WARNING: {ACCOUNT_MAP_FILE} not found in: {Path.cwd()}")
        return {}

    account_map = {}

    with map_path.open("r", newline="", encoding="utf-8-sig") as mapfile:
        sample = mapfile.read(2048)
        mapfile.seek(0)

        try:
            dialect = csv.Sniffer().sniff(sample)
        except csv.Error:
            dialect = csv.excel

        reader = csv.DictReader(mapfile, dialect=dialect)

        if not reader.fieldnames:
            print(f"WARNING: {ACCOUNT_MAP_FILE} has no headers.")
            return {}

        reader.fieldnames = [clean_text(h) for h in reader.fieldnames]

        print("\nAccountMap.csv headers found:")
        for h in reader.fieldnames:
            print(f"  - [{h}]")

        account_name_col = get_column_name(reader.fieldnames, [
            "Account Name",
            "accountName",
            "Name",
        ])

        account_number_col = get_column_name(reader.fieldnames, [
            "Account Number",
            "AccountNumber",
            "accountId",
            "Account Id",
            "Account ID",
            "Account",
        ])

        account_owner_col = get_column_name(reader.fieldnames, [
            "Account owner",
            "Account Owner",
            "Owner",
            "accountOwner",
            "BusinessUnitOwner",
        ])

        print("\nDetected AccountMap columns:")
        print(f"  Account Name   -> {account_name_col}")
        print(f"  Account Number -> {account_number_col}")
        print(f"  Account owner  -> {account_owner_col}")

        if not account_number_col:
            print("\nERROR: Could not detect account number column in AccountMap.csv")
            return {}

        for row in reader:
            account_number_raw = clean_text(row.get(account_number_col, ""))
            account_number_key = normalize_account_number(account_number_raw)

            if not account_number_key:
                continue

            account_name = clean_text(row.get(account_name_col, "")) if account_name_col else ""
            account_owner = clean_text(row.get(account_owner_col, "")) if account_owner_col else ""

            account_map[account_number_key] = {
                "Account Name": account_name or "Not Found",
                "Account Number": account_number_raw or "Not Found",
                "Account owner": account_owner or "Not Found",
            }

    print(f"\nLoaded account mappings: {len(account_map)}")
    return account_map


def main():
    input_path = Path(INPUT_FILE)

    if not input_path.exists():
        print(f"ERROR: {INPUT_FILE} not found in: {Path.cwd()}")
        sys.exit(1)

    account_map = load_account_map()

    matched_count = 0
    unmatched_count = 0

    with input_path.open("r", newline="", encoding="utf-8-sig") as infile:
        reader = csv.DictReader(infile)

        if not reader.fieldnames:
            print("ERROR: Results.csv has no headers.")
            sys.exit(1)

        reader.fieldnames = [clean_text(h) for h in reader.fieldnames]

        print("\nResults.csv headers found:")
        for h in reader.fieldnames:
            print(f"  - [{h}]")

        required_columns = ["accountId", "resourceName", "tags"]

        missing = [col for col in required_columns if col not in reader.fieldnames]

        if missing:
            print("\nERROR: Missing required columns in Results.csv:")
            for col in missing:
                print(f"  - {col}")
            sys.exit(1)

        with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as outfile:
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

                if not account_id_key:
                    continue

                resource_name = clean_text(row.get("resourceName", ""))
                tags_value = clean_text(row.get("tags", ""))

                tag_dict = parse_tags(tags_value)

                env = decide_env(resource_name, tag_dict)

                # Account details from mapping file
                account_details = account_map.get(account_id_key)

                if account_details:
                    matched_count += 1
                    account_name = account_details["Account Name"]
                    account_number = account_details["Account Number"]
                    account_owner_from_map = account_details["Account owner"]
                else:
                    unmatched_count += 1
                    account_name = "Not Found"
                    account_number = account_id_raw
                    account_owner_from_map = "Not Found"

                # Owner fallback from Results.csv tags
                account_owner_from_tags = clean_text(tag_dict.get("BusinessUnitOwner", ""))

                if account_owner_from_map != "Not Found":
                    final_account_owner = account_owner_from_map
                elif account_owner_from_tags:
                    final_account_owner = account_owner_from_tags
                else:
                    final_account_owner = "Not Found"

                writer.writerow([
                    count,
                    env,
                    account_name,
                    account_number,
                    final_account_owner,
                ])

                count += 1

    print(f"\nDone! Created {OUTPUT_FILE}")
    print(f"Total records written: {count - 1}")
    print(f"Matched account records from AccountMap.csv: {matched_count}")
    print(f"Unmatched account records from AccountMap.csv: {unmatched_count}")


if __name__ == "__main__":
    main()

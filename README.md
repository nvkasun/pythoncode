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
OUTPUT_FILE = "Final_Records.csv"

VALID_ENVS = ["prod", "preprod", "uat", "dev", "sit"]


def normalize_env(value: str) -> str:
    """
    Normalize environment values.
    Example:
      Production -> prod
      PREPROD -> preprod
      dev -> dev
    """
    if not value:
        return ""

    value = value.strip().lower()

    mapping = {
        "production": "prod",
        "prod": "prod",
        "pre-production": "preprod",
        "preprod": "preprod",
        "pre-prod": "preprod",
        "uat": "uat",
        "dev": "dev",
        "development": "dev",
        "sit": "sit",
    }

    return mapping.get(value, value)


def extract_env_from_resource_name(resource_name: str) -> str:
    """
    Extract environment from resourceName.
    Example:
      cluster-digitalonboarding-dev -> dev
      cluster-transactionauthenticator-preprod -> preprod
    """

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
    """
    Extract Environment value from tags column.

    Expected tags format:
      [
        {"tag":"Environment=dev","value":"dev","key":"Environment"},
        {"tag":"Name=cluster-example-dev","value":"cluster-example-dev","key":"Name"}
      ]
    """

    if not tags_value:
        return ""

    tags_value = tags_value.strip()

    if tags_value in ["", "-"]:
        return ""

    # Some CSV exports may store escaped quotes
    cleaned = tags_value.replace('\\"', '"')

    try:
        tags = json.loads(cleaned)
    except json.JSONDecodeError:
        # Fallback regex if JSON parsing fails
        match = re.search(
            r'"key"\s*:\s*"Environment"\s*,\s*"tag"\s*:\s*"Environment=([^"]+)"',
            cleaned,
            re.IGNORECASE,
        )

        if not match:
            match = re.search(
                r'"key"\s*:\s*"Environment"\s*,\s*"value"\s*:\s*"([^"]+)"',
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


def decide_env(resource_name: str, tags_value: str):
    """
    Decide final environment using both tags and resourceName.

    Priority:
      1. tags Environment value
      2. resourceName fallback
    """

    env_from_tags = extract_env_from_tags(tags_value)
    env_from_name = extract_env_from_resource_name(resource_name)

    if env_from_tags and env_from_name:
        if env_from_tags == env_from_name:
            return env_from_tags, "tags_and_resourceName_matched"
        else:
            return env_from_tags, f"conflict_tags={env_from_tags}_resourceName={env_from_name}"

    if env_from_tags:
        return env_from_tags, "tags"

    if env_from_name:
        return env_from_name, "resourceName"

    return "", "not_found"


def main():
    input_path = Path(INPUT_FILE)
    output_path = Path(OUTPUT_FILE)

    if not input_path.exists():
        print(f"Error: {INPUT_FILE} not found in current folder: {Path.cwd()}")
        sys.exit(1)

    try:
        with input_path.open("r", newline="", encoding="utf-8-sig") as infile:
            reader = csv.DictReader(infile)

            if not reader.fieldnames:
                print("Error: CSV file has no headers.")
                sys.exit(1)

            reader.fieldnames = [header.strip() for header in reader.fieldnames]

            required_columns = ["accountId", "resourceName", "tags"]

            missing_columns = [
                col for col in required_columns
                if col not in reader.fieldnames
            ]

            if missing_columns:
                print("Error: Missing required column(s):")
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
                    "accountId",
                    "resourceName",
                    "Env",
                    "EnvSource"
                ])

                count = 1

                for row in reader:
                    account_id = row.get("accountId", "").strip()
                    resource_name = row.get("resourceName", "").strip()
                    tags_value = row.get("tags", "").strip()

                    if not account_id or account_id == "-":
                        continue

                    env, env_source = decide_env(resource_name, tags_value)

                    writer.writerow([
                        count,
                        account_id,
                        resource_name,
                        env,
                        env_source
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

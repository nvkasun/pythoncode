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


def get_value(row, column_name):
    """
    Safely get a column value from Results.csv.
    Converts empty and '-' values into empty string.
    """
    value = clean_text(row.get(column_name, ""))

    if not value or value == "-":
        return ""

    return value


def parse_json_value(raw_value):
    """
    Safely parse JSON value from CSV cell.
    Handles escaped quotes from AWS Config CSV exports.
    """
    raw_value = clean_text(raw_value)

    if not raw_value or raw_value == "-":
        return None

    cleaned = raw_value.replace('\\"', '"')

    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        return None


def parse_configuration(configuration_value):
    """
    Parse full configuration JSON column.
    Used to extract fields that may not exist as separate CSV columns.
    """
    parsed = parse_json_value(configuration_value)

    if isinstance(parsed, dict):
        return parsed

    return {}


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


def extract_first_cluster_member_identifier(dbcluster_members_value):
    """
    Try to extract first dbinstanceIdentifier from configuration.dbclusterMembers.
    """

    dbcluster_members_value = clean_text(dbcluster_members_value)

    if not dbcluster_members_value or dbcluster_members_value == "-":
        return ""

    cleaned = dbcluster_members_value.replace('\\"', '"')

    try:
        members = json.loads(cleaned)
    except json.JSONDecodeError:
        match = re.search(
            r'"dbinstanceIdentifier"\s*:\s*"([^"]+)"',
            cleaned,
            re.IGNORECASE,
        )

        if match:
            return clean_text(match.group(1))

        return ""

    if isinstance(members, list):
        for member in members:
            if not isinstance(member, dict):
                continue

            identifier = clean_text(member.get("dbinstanceIdentifier", ""))

            if identifier:
                return identifier

    if isinstance(members, dict):
        identifier = clean_text(members.get("dbinstanceIdentifier", ""))

        if identifier:
            return identifier

    return ""


def decide_db_type(row, config_dict):
    """
    DB Type comes from configuration.engine.
    """
    engine = get_value(row, "configuration.engine")

    if engine:
        return engine

    engine = clean_text(config_dict.get("engine", ""))

    if engine:
        return engine

    return "Not Found"


def decide_engine(row, config_dict):
    """
    engine comes from configuration.engine.
    """
    engine = get_value(row, "configuration.engine")

    if engine:
        return engine

    engine = clean_text(config_dict.get("engine", ""))

    if engine:
        return engine

    return "Not Found"


def decide_db_identifier(row):
    """
    DB Identifier priority:
      1. configuration.dBInstanceIdentifier
      2. configuration.dbclusterMembers first dbinstanceIdentifier
      3. resourceName
      4. resourceId
      5. Not Found
    """

    db_identifier = get_value(row, "configuration.dBInstanceIdentifier")

    if db_identifier:
        return db_identifier

    dbcluster_members = get_value(row, "configuration.dbclusterMembers")
    member_identifier = extract_first_cluster_member_identifier(dbcluster_members)

    if member_identifier:
        return member_identifier

    resource_name = get_value(row, "resourceName")

    if resource_name:
        return resource_name

    resource_id = get_value(row, "resourceId")

    if resource_id:
        return resource_id

    return "Not Found"


def decide_db_name(row, config_dict):
    """
    DB Name priority:
      1. configuration.databaseName direct column, if present
      2. databaseName from full configuration JSON
      3. Not Found
    """
    db_name = get_value(row, "configuration.databaseName")

    if db_name:
        return db_name

    db_name = clean_text(config_dict.get("databaseName", ""))

    if db_name:
        return db_name

    return "Not Found"


def decide_master_user(row, config_dict):
    """
    MasterUser priority:
      1. configuration.masterUsername direct column
      2. masterUsername from full configuration JSON
      3. Not Found
    """
    master_user = get_value(row, "configuration.masterUsername")

    if master_user:
        return master_user

    master_user = clean_text(config_dict.get("masterUsername", ""))

    if master_user:
        return master_user

    return "Not Found"


def decide_db_version(row, config_dict):
    """
    DB Version priority:
      1. configuration.engineVersion direct column
      2. engineVersion from full configuration JSON
      3. Not Found
    """
    db_version = get_value(row, "configuration.engineVersion")

    if db_version:
        return db_version

    db_version = clean_text(config_dict.get("engineVersion", ""))

    if db_version:
        return db_version

    return "Not Found"


def decide_cloudwatch(row, config_dict):
    """
    CloudWatch priority:
      1. configuration.enabledCloudwatchLogsExports direct column
      2. enabledCloudwatchLogsExports from full configuration JSON
      3. Not Found
    """

    cloudwatch = get_value(row, "configuration.enabledCloudwatchLogsExports")

    if cloudwatch:
        parsed = parse_json_value(cloudwatch)

        if isinstance(parsed, list):
            if parsed:
                return ", ".join(clean_text(item) for item in parsed)
            return "Not Found"

        return cloudwatch

    cloudwatch = config_dict.get("enabledCloudwatchLogsExports", "")

    if isinstance(cloudwatch, list):
        if cloudwatch:
            return ", ".join(clean_text(item) for item in cloudwatch)
        return "Not Found"

    cloudwatch = clean_text(cloudwatch)

    if cloudwatch:
        return cloudwatch

    return "Not Found"


def decide_backup_retention(row, config_dict):
    """
    Backup Retention priority:
      1. configuration.backupRetentionPeriod direct column
      2. backupRetentionPeriod from full configuration JSON
      3. Not Found
    """
    backup_retention = get_value(row, "configuration.backupRetentionPeriod")

    if backup_retention:
        return backup_retention

    backup_retention = clean_text(config_dict.get("backupRetentionPeriod", ""))

    if backup_retention:
        return backup_retention

    return "Not Found"


def decide_maintenance_window(row, config_dict):
    """
    Maintenance Window priority:
      1. configuration.preferredMaintenanceWindow direct column
      2. preferredMaintenanceWindow from full configuration JSON
      3. Not Found
    """
    maintenance_window = get_value(row, "configuration.preferredMaintenanceWindow")

    if maintenance_window:
        return maintenance_window

    maintenance_window = clean_text(config_dict.get("preferredMaintenanceWindow", ""))

    if maintenance_window:
        return maintenance_window

    return "Not Found"


def decide_endpoint(row, config_dict):
    """
    EndPoint priority:
      1. configuration.endpoint.address
      2. configuration.endpoint.value
      3. endpoint.address from full configuration JSON
      4. endpoint.value from full configuration JSON
      5. Not Found
    """

    endpoint = get_value(row, "configuration.endpoint.address")

    if endpoint:
        return endpoint

    endpoint = get_value(row, "configuration.endpoint.value")

    if endpoint:
        return endpoint

    endpoint_obj = config_dict.get("endpoint", {})

    if isinstance(endpoint_obj, dict):
        endpoint = clean_text(endpoint_obj.get("address", ""))

        if endpoint:
            return endpoint

        endpoint = clean_text(endpoint_obj.get("value", ""))

        if endpoint:
            return endpoint

    return "Not Found"


def decide_port(row, config_dict):
    """
    Port priority:
      1. configuration.endpoint.port
      2. configuration.port
      3. endpoint.port from full configuration JSON
      4. port from full configuration JSON
      5. Not Found
    """

    port = get_value(row, "configuration.endpoint.port")

    if port:
        return port

    port = get_value(row, "configuration.port")

    if port:
        return port

    endpoint_obj = config_dict.get("endpoint", {})

    if isinstance(endpoint_obj, dict):
        port = clean_text(endpoint_obj.get("port", ""))

        if port:
            return port

    port = clean_text(config_dict.get("port", ""))

    if port:
        return port

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

        required_columns = [
            "accountId",
            "resourceName",
            "tags",
            "configuration",
            "configuration.engine",
            "configuration.engineVersion",
            "configuration.dBInstanceIdentifier",
            "configuration.dbclusterMembers",
            "configuration.masterUsername",
            "configuration.enabledCloudwatchLogsExports",
            "configuration.backupRetentionPeriod",
            "configuration.preferredMaintenanceWindow",
            "configuration.endpoint.address",
            "configuration.endpoint.value",
            "configuration.endpoint.port",
            "configuration.port",
            "resourceId",
        ]

        missing = [col for col in required_columns if col not in reader.fieldnames]

        if missing:
            print("\nERROR: Missing required columns in Results.csv:")
            for col in missing:
                print(f"  - {col}")

            print("\nAvailable Results.csv columns:")
            for h in reader.fieldnames:
                print(f"  - {h}")

            sys.exit(1)

        with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as outfile:
            writer = csv.writer(outfile)

            writer.writerow([
                "Sl No",
                "Env",
                "Account Name",
                "Account Number",
                "Account owner",
                "DB Type",
                "DB Identifier",
                "engine",
                "DB Name",
                "MasterUser",
                "DB Version",
                "CloudWatch",
                "Backup Retention",
                "Maintenance Window",
                "EndPoint",
                "Port",
            ])

            count = 1

            for row in reader:
                account_id_raw = clean_text(row.get("accountId", ""))
                account_id_key = normalize_account_number(account_id_raw)

                if not account_id_key:
                    continue

                resource_name = clean_text(row.get("resourceName", ""))
                tags_value = clean_text(row.get("tags", ""))
                configuration_value = clean_text(row.get("configuration", ""))

                tag_dict = parse_tags(tags_value)
                config_dict = parse_configuration(configuration_value)

                env = decide_env(resource_name, tag_dict)

                db_type = decide_db_type(row, config_dict)
                db_identifier = decide_db_identifier(row)
                engine = decide_engine(row, config_dict)

                db_name = decide_db_name(row, config_dict)
                master_user = decide_master_user(row, config_dict)
                db_version = decide_db_version(row, config_dict)
                cloudwatch = decide_cloudwatch(row, config_dict)
                backup_retention = decide_backup_retention(row, config_dict)
                maintenance_window = decide_maintenance_window(row, config_dict)
                endpoint = decide_endpoint(row, config_dict)
                port = decide_port(row, config_dict)

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
                    db_type,
                    db_identifier,
                    engine,
                    db_name,
                    master_user,
                    db_version,
                    cloudwatch,
                    backup_retention,
                    maintenance_window,
                    endpoint,
                    port,
                ])

                count += 1

    print(f"\nDone! Created {OUTPUT_FILE}")
    print(f"Total records written: {count - 1}")
    print(f"Matched account records from AccountMap.csv: {matched_count}")
    print(f"Unmatched account records from AccountMap.csv: {unmatched_count}")


if __name__ == "__main__":
    main()

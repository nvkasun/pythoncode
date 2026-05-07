#!/usr/bin/env python3

import csv
import sys
from pathlib import Path

# =====================================================
# Configuration
# =====================================================
INPUT_FILE = "Results.csv"
OUTPUT_FILE = "Final_Records.csv"

# Columns you want in the final output
REQUIRED_COLUMNS = [
    "accountId"
]


def main():
    input_path = Path(INPUT_FILE)
    output_path = Path(OUTPUT_FILE)

    # 1. Check input file exists
    if not input_path.exists():
        print(f"Error: {INPUT_FILE} not found in current folder: {Path.cwd()}")
        sys.exit(1)

    try:
        # utf-8-sig handles Excel UTF-8 BOM if present
        with input_path.open("r", newline="", encoding="utf-8-sig") as infile:
            reader = csv.DictReader(infile)

            if not reader.fieldnames:
                print("Error: CSV file has no headers.")
                sys.exit(1)

            # Clean header names
            reader.fieldnames = [header.strip() for header in reader.fieldnames]

            # 2. Validate required columns
            missing_columns = [
                col for col in REQUIRED_COLUMNS
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

            # 3. Write final output
            with output_path.open("w", newline="", encoding="utf-8") as outfile:
                writer = csv.writer(outfile)

                # Header
                writer.writerow(["Sl No"] + REQUIRED_COLUMNS)

                count = 1

                for row in reader:
                    values = []

                    for col in REQUIRED_COLUMNS:
                        value = row.get(col, "")
                        value = value.strip() if value else ""
                        values.append(value)

                    # Skip rows where accountId is empty or "-"
                    account_id = values[0]
                    if account_id and account_id != "-":
                        writer.writerow([count] + values)
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

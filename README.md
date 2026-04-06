#!/usr/bin/env python3
"""
iCloud Drive Duplicate File Cleaner
====================================
Scans your entire iCloud Drive on macOS, finds exact duplicate files,
keeps the first copy found, deletes all subsequent duplicates,
and reports how much space was recovered.

Usage:
    python3 icloud_duplicate_cleaner.py              # Dry run (safe preview, nothing deleted)
    python3 icloud_duplicate_cleaner.py --delete      # Actually delete duplicates

Author: Manus AI
"""

import os
import sys
import hashlib
import argparse
from pathlib import Path
from collections import defaultdict

# ──────────────────────────────────────────────
# CONFIGURATION
# ──────────────────────────────────────────────
ICLOUD_DRIVE_PATH = os.path.expanduser(
    "~/Library/Mobile Documents/com~apple~CloudDocs"
)

# Files/folders to skip (system files, caches, etc.)
SKIP_NAMES = {".DS_Store", ".Trash", "__pycache__", ".git"}


# ──────────────────────────────────────────────
# HELPERS
# ──────────────────────────────────────────────
def format_size(size_bytes):
    """Convert bytes to a human-readable string."""
    if size_bytes < 1024:
        return f"{size_bytes} B"
    elif size_bytes < 1024 ** 2:
        return f"{size_bytes / 1024:.2f} KB"
    elif size_bytes < 1024 ** 3:
        return f"{size_bytes / 1024 ** 2:.2f} MB"
    else:
        return f"{size_bytes / 1024 ** 3:.2f} GB"


def file_hash(filepath, chunk_size=8192):
    """
    Generate a SHA-256 hash of a file's contents.
    Reads in chunks so it can handle very large files
    without running out of memory.
    """
    sha256 = hashlib.sha256()
    try:
        with open(filepath, "rb") as f:
            while True:
                chunk = f.read(chunk_size)
                if not chunk:
                    break
                sha256.update(chunk)
    except (PermissionError, OSError) as e:
        print(f"  ⚠  Skipped (cannot read): {filepath}  —  {e}")
        return None
    return sha256.hexdigest()


def should_skip(name):
    """Check if a file or folder should be skipped."""
    return name in SKIP_NAMES or name.startswith(".")


# ──────────────────────────────────────────────
# STEP 1 — GROUP FILES BY SIZE
# ──────────────────────────────────────────────
def group_files_by_size(root_path):
    """
    Walk the entire iCloud Drive and group files by their size.
    Files with a unique size cannot possibly be duplicates,
    so this is a fast first filter before we do expensive hashing.
    """
    size_map = defaultdict(list)
    file_count = 0
    skipped = 0

    print(f"\n📂  Scanning: {root_path}\n")

    for dirpath, dirnames, filenames in os.walk(root_path):
        # Skip hidden/system directories
        dirnames[:] = [d for d in dirnames if not should_skip(d)]

        for filename in filenames:
            if should_skip(filename):
                skipped += 1
                continue

            filepath = os.path.join(dirpath, filename)

            # Skip symbolic links and non-regular files
            if not os.path.isfile(filepath) or os.path.islink(filepath):
                skipped += 1
                continue

            try:
                size = os.path.getsize(filepath)
            except OSError:
                skipped += 1
                continue

            # Skip empty files (0 bytes)
            if size == 0:
                skipped += 1
                continue

            size_map[size].append(filepath)
            file_count += 1

    print(f"   Files scanned : {file_count}")
    print(f"   Files skipped : {skipped}")

    # Only keep groups where more than one file shares the same size
    potential = {s: paths for s, paths in size_map.items() if len(paths) > 1}
    candidates = sum(len(v) for v in potential.values())
    print(f"   Possible dupes: {candidates} files in {len(potential)} size groups\n")

    return potential


# ──────────────────────────────────────────────
# STEP 2 — HASH AND FIND TRUE DUPLICATES
# ──────────────────────────────────────────────
def find_duplicates(size_groups):
    """
    For each group of same-size files, compute SHA-256 hashes
    to confirm which ones are truly identical.
    Returns a dict: { hash: [filepath, filepath, ...] }
    (only groups with 2+ files).
    """
    hash_map = defaultdict(list)
    total_to_hash = sum(len(v) for v in size_groups.values())
    hashed = 0

    print("🔍  Hashing files to confirm duplicates...\n")

    for size, paths in size_groups.items():
        for filepath in paths:
            h = file_hash(filepath)
            if h is not None:
                hash_map[h].append(filepath)
            hashed += 1
            # Progress indicator every 100 files
            if hashed % 100 == 0 or hashed == total_to_hash:
                pct = (hashed / total_to_hash) * 100
                print(f"   Progress: {hashed}/{total_to_hash} ({pct:.0f}%)", end="\r")

    print()  # newline after progress

    # Only keep groups with actual duplicates
    duplicates = {h: paths for h, paths in hash_map.items() if len(paths) > 1}
    return duplicates


# ──────────────────────────────────────────────
# STEP 3 — REPORT AND OPTIONALLY DELETE
# ──────────────────────────────────────────────
def process_duplicates(duplicates, do_delete=False):
    """
    For each set of duplicates:
      - Keep the first file (sorted alphabetically so it's predictable)
      - Delete (or list) all subsequent copies
      - Track total space recovered
    """
    total_recovered = 0
    total_deleted = 0
    total_groups = len(duplicates)

    if total_groups == 0:
        print("\n✅  No duplicate files found! Your iCloud Drive is already clean.\n")
        return

    mode = "DELETING" if do_delete else "DRY RUN (preview only)"
    print(f"\n{'=' * 60}")
    print(f"   MODE: {mode}")
    print(f"   Duplicate groups found: {total_groups}")
    print(f"{'=' * 60}\n")

    for i, (h, paths) in enumerate(duplicates.items(), 1):
        # Sort paths so the first one is consistent and predictable
        paths.sort()
        keeper = paths[0]
        to_remove = paths[1:]

        try:
            file_size = os.path.getsize(keeper)
        except OSError:
            file_size = 0

        print(f"── Group {i}/{total_groups} ({'─' * 40})")
        print(f"   ✅ KEEP : {keeper}")
        for dup in to_remove:
            print(f"   🗑  DEL  : {dup}  ({format_size(file_size)})")

            if do_delete:
                try:
                    os.remove(dup)
                    total_recovered += file_size
                    total_deleted += 1
                except (PermissionError, OSError) as e:
                    print(f"      ⚠  Could not delete: {e}")
            else:
                total_recovered += file_size
                total_deleted += 1
        print()

    # ──────────────────────────────────────────
    # FINAL SUMMARY
    # ──────────────────────────────────────────
    print(f"{'=' * 60}")
    print(f"   📊  SUMMARY")
    print(f"{'=' * 60}")
    print(f"   Duplicate groups   : {total_groups}")
    print(f"   Files to remove    : {total_deleted}")
    print(f"   Space recovered    : {format_size(total_recovered)}")
    print(f"{'=' * 60}")

    if not do_delete:
        print()
        print("   ⚡ This was a DRY RUN — nothing was deleted.")
        print("   ⚡ To actually delete duplicates, run again with --delete :")
        print()
        print("       python3 icloud_duplicate_cleaner.py --delete")
        print()
    else:
        print()
        print(f"   🎉  Done! You recovered {format_size(total_recovered)} of space.")
        print()


# ──────────────────────────────────────────────
# MAIN
# ──────────────────────────────────────────────
def main():
    parser = argparse.ArgumentParser(
        description="Find and delete duplicate files on your iCloud Drive."
    )
    parser.add_argument(
        "--delete",
        action="store_true",
        help="Actually delete duplicate files. Without this flag, the script only previews what would be deleted (dry run).",
    )
    parser.add_argument(
        "--path",
        type=str,
        default=ICLOUD_DRIVE_PATH,
        help="Custom path to scan instead of the default iCloud Drive location.",
    )
    args = parser.parse_args()

    scan_path = args.path

    # Verify the path exists
    if not os.path.isdir(scan_path):
        print(f"\n❌  Path not found: {scan_path}")
        print("    Make sure iCloud Drive is enabled and synced on this Mac.")
        print("    Or use --path to specify a custom folder.\n")
        sys.exit(1)

    # Step 1: Group by size
    size_groups = group_files_by_size(scan_path)

    if not size_groups:
        print("\n✅  No potential duplicates found. Your drive is clean!\n")
        return

    # Step 2: Hash to confirm
    duplicates = find_duplicates(size_groups)

    # Step 3: Report and optionally delete
    process_duplicates(duplicates, do_delete=args.delete)


if __name__ == "__main__":
    main()

python icloud_duplicate_cleaner.py# icloud_duplicate_cleaner.py
cleans duplicate files from icloud drive and gives total freed up space

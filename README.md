#!/usr/bin/env python3
"""
File Organizer - A utility to organize files into directories based on their type.
"""

import os
import shutil
from pathlib import Path
from typing import Dict, List
import argparse
import logging


# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# File type to directory mapping
FILE_EXTENSIONS = {
    'Images': ['.jpg', '.jpeg', '.png', '.gif', '.bmp', '.svg', '.ico', '.tiff'],
    'Videos': ['.mp4', '.avi', '.mov', '.mkv', '.flv', '.wmv', '.webm'],
    'Audio': ['.mp3', '.wav', '.flac', '.aac', '.ogg', '.wma', '.m4a'],
    'Documents': ['.pdf', '.doc', '.docx', '.txt', '.xlsx', '.xls', '.ppt', '.pptx', '.odt'],
    'Archives': ['.zip', '.rar', '.7z', '.tar', '.gz', '.bz2', '.iso'],
    'Code': ['.py', '.js', '.java', '.cpp', '.c', '.html', '.css', '.php', '.rb', '.go', '.rs', '.ts'],
    'Executables': ['.exe', '.msi', '.app', '.deb', '.apk'],
    'Others': []
}


class FileOrganizer:
    """Organize files into directories based on their type."""
    def __init__(self, directory: str, create_subdirs: bool = True, dry_run: bool = False):
        """
        Initialize the FileOrganizer.
        Args:
            directory: Path to the directory to organize
            create_subdirs: Whether to create subdirectories (default: True)
            dry_run: If True, only show what would be done without making changes
        """
        self.directory = Path(directory)
        self.create_subdirs = create_subdirs
        self.dry_run = dry_run
        self.file_mapping = FILE_EXTENSIONS.copy()
        self.stats = {
            'moved': 0,
            'skipped': 0,
            'errors': 0
        }
        if not self.directory.exists():
            raise ValueError(f"Directory does not exist: {directory}")
        if not self.directory.is_dir():
            raise ValueError(f"Path is not a directory: {directory}")

    def get_file_category(self, file_path: Path) -> str:
        """
        Determine the category of a file based on its extension.
        Args:
            file_path: Path to the file
        Returns:
            Category name or 'Others'
        """
        extension = file_path.suffix.lower()
        for category, extensions in self.file_mapping.items():
            if extension in extensions:
                return category
        return 'Others'

    def organize(self) -> None:
        """Organize files in the directory."""
        logger.info(f"Starting file organization in: {self.directory}")
        # Get all files in the directory (not subdirectories)
        files = [f for f in self.directory.iterdir() if f.is_file()]
        if not files:
            logger.info("No files found to organize")
            return
        for file in files:
            try:
                category = self.get_file_category(file)
                self._move_file(file, category)
            except Exception as e:
                logger.error(f"Error processing file {file.name}: {e}")
                self.stats['errors'] += 1
        self._print_summary()

    def _move_file(self, file: Path, category: str) -> None:
        """
        Move a file to its category directory.
        Args:
            file: Path to the file
            category: Target category
        """
        target_dir = self.directory / category
        # Create subdirectory if needed
        if self.create_subdirs:
            if not self.dry_run and not target_dir.exists():
                target_dir.mkdir(parents=True, exist_ok=True)
                logger.info(f"Created directory: {category}")
        target_path = target_dir / file.name
        # Handle duplicate filenames
        if target_path.exists():
            base_name = file.stem
            extension = file.suffix
            counter = 1
            while target_path.exists():
                new_name = f"{base_name}_{counter}{extension}"
                target_path = target_dir / new_name
                counter += 1
        if self.dry_run:
            logger.info(f"[DRY RUN] Would move: {file.name} → {category}/{target_path.name}")
        else:
            shutil.move(str(file), str(target_path))
            logger.info(f"Moved: {file.name} → {category}/")
        self.stats['moved'] += 1

    def _print_summary(self) -> None:
        """Print organization summary."""
        logger.info(f"\n{'='*50}")
        logger.info(f"Organization Summary:")
        logger.info(f"  Files moved: {self.stats['moved']}")
        logger.info(f"  Errors encountered: {self.stats['errors']}")
        logger.info(f"{'='*50}\n")

    def add_custom_extension(self, category: str, extension: str) -> None:
        """
        Add a custom file extension to a category.
        Args:
            category: Target category
            extension: File extension (with or without dot)
        """
        if not extension.startswith('.'):
            extension = '.' + extension
        if category not in self.file_mapping:
            self.file_mapping[category] = []
        if extension not in self.file_mapping[category]:
            self.file_mapping[category].append(extension)
            logger.info(f"Added {extension} to {category} category")

    def get_statistics(self) -> Dict:
        """Get file type statistics for the directory."""
        stats = {}
        for file in self.directory.rglob('*'):
            if file.is_file():
                category = self.get_file_category(file)
                stats[category] = stats.get(category, 0) + 1
        return stats


def main():
    """Main entry point for the file organizer."""
    parser = argparse.ArgumentParser(
        description='Organize files into directories based on their type'
    )
    parser.add_argument(
        'directory',
        help='Directory to organize'
    )
    parser.add_argument(
        '--dry-run',
        action='store_true',
        help='Show what would be done without making changes'
    )
    parser.add_argument(
        '--no-subdirs',
        action='store_true',
        help='Do not create subdirectories'
    )
    parser.add_argument(
        '--add-extension',
        nargs=2,
        metavar=('CATEGORY', 'EXTENSION'),
        help='Add a custom extension mapping (e.g., --add-extension Documents .md)'
    )
    args = parser.parse_args()
    try:
        organizer = FileOrganizer(
            args.directory,
            create_subdirs=not args.no_subdirs,
            dry_run=args.dry_run
        )
        if args.add_extension:
            organizer.add_custom_extension(args.add_extension[0], args.add_extension[1])
        organizer.organize()
    except ValueError as e:
        logger.error(f"Error: {e}")
        return 1
    return 0


if __name__ == '__main__':
    exit(main())

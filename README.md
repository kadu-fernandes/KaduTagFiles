# Kadu Files

## **Overview**

**Kadu Files** is a project designed to explore Go while providing a system to tag and track text files, including **Markdown, MS Word, and LibreOffice Write** documents.

This application allows users to tag files inside **tracked directories**, making it easy to categorize and retrieve important documents.

The project stores metadata in a **SQLite database** but also exports it to a `.kadu_tags` file for **cross-system synchronization**. This allows users to **track the same files across multiple computers**, even if the base directory changes, as long as the repository structure remains the same.

Let's say the user has a repository with many sermons and Bible studies.

He can tag:

+ The ones he preaches in "Church A";
+ The ones he preaches on "2025-01-11";
+ The ones with the theme "Christmas";
+ The ones based on the "Old Testament";
+ The ones based on "Mateus", and so on...

If he clones the repository into different folders on different computers, such as:

+ `/home/kadu/myWonderfulSermons` on Laptop A
+ `/texts/myWonderfulSermons` on Laptop B

The tagging will persist.

## How It Works

1. The user **adds a directory** to be tracked (excluding root, home, and desktop).
1. The system **normalizes the paths** to ensure consistency across environments.
1. The system **scans for supported file types** and adds them to the database.
1. Files are identified by their **uuid** and **short_path** (relative to the tracked directory), ensuring they can be recognized across different machines.
1. Tags are **associated with files** using a **unique slug** and stored both in the database and in `.kadu_tags` for portability.
1. Changes to tags are automatically **synchronized via Git**, so they propagate across machines when syncing the repository.
1. When importing on another computer, the system **reconstructs tag associations** by matching `uuid`, `short_path`, and `slug` to avoid case-sensitivity issues.

This approach ensures that **tags remain linked to files** even if they are edited, renamed, or moved **inside the tracked directory**.

## Directory

A **directory** is a root location where files are tracked. When a directory is added, all files within it that match supported extensions are registered.

A **directory entity** should have its **normalized path** as a property.
The normalized path is the expanded path, ensuring consistency across different environments.

### Examples of Normalization

+ `~/docs` → `/home/user/docs`
+ `./projects` → `/my/current/folder/projects`

Only **directories** can be added—files cannot be tracked individually unless they belong to a tracked directory.

### SQL Table Definition for Directory

~~~sql
CREATE TABLE directories (
    id TEXT PRIMARY KEY, -- Hash of the normalized path
    normalized_path TEXT NOT NULL UNIQUE -- Full normalized path of the directory
);
~~~

## File

A **file** is registered when it is found inside a tracked directory.
Each file is stored using its **relative path (short_path)** within the directory, making tracking portable across different machines.

### File Identity Strategy

Each file is identified by:

1. **UUID** → A unique identifier for the file, remaining unchanged even if the file is renamed.
1. **Short Path** → The relative path inside the tracked directory, ensuring portability.
1. **File Name & Extension** → To allow filtering and easy retrieval.

### Example Paths

+ **Full Path:** `/home/user/docs/my_project/report.md`

+ **Tracked Directory:** `/home/user/docs/my_project/`
+ **Short Path:** `report.md`

### SQL Table Definition for File

~~~sql
CREATE TABLE files (
    id INTEGER PRIMARY KEY AUTOINCREMENT, -- Database primary key for performance
    uuid TEXT UNIQUE, -- Persistent UUID from JSON, ensuring cross-system consistency
    directory_id TEXT NOT NULL, -- Foreign key referencing directories
    short_path TEXT NOT NULL, -- Relative path within the directory
    file_name TEXT NOT NULL, -- File name for easy queries
    extension TEXT NOT NULL, -- File extension
    FOREIGN KEY (directory_id) REFERENCES directories(id)
);
~~~

## Tag

A **tag** represents a keyword or category that can be associated with multiple files. Tags exist independently and are case-sensitive.

### SQL Table Definition for Tag

~~~sql
CREATE TABLE tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT, -- Database primary key for performance
    slug TEXT NOT NULL UNIQUE, -- Unique slug for the tag (case-insensitive)
    name TEXT NOT NULL, -- Display name of the tag
    description TEXT -- Optional description of the tag
);
~~~

## Tagging Files

Each file can have multiple tags, and each tag can be associated with multiple files, creating a **many-to-many relationship**.

### SQL Table Definition for File-Tag Relationship

~~~sql
CREATE TABLE file_tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT, -- Unique identifier for the relationship
    file_id TEXT NOT NULL, -- Foreign key referencing files
    tag_id INTEGER NOT NULL, -- Foreign key referencing tags
    FOREIGN KEY (file_uuid) REFERENCES files(uuid),
    FOREIGN KEY (tag_id) REFERENCES tags(id) -- Foreign key constraint
);
~~~

## JSON File Structure & Synchronization

To enable synchronization across multiple machines, the system **exports all metadata to a `.kadu_tags` file** in each tracked directory. This file moves with the repository when using Git.

### JSON File Example

~~~json
{
  "files": [
    {
      "uuid": "550e8400-e29b-41d4-a716-446655440000",
      "short_path": "my/relative/path/sermon1.md",
      "file_name": "sermon1.md",
      "extension": "md",
      "tags": ["sunday", "faith"]
    }
  ],
  "tags": [
    {
      "slug": "sunday",
      "name": "Sunday Sermon",
      "description": "Sermons to be preached on Sundays"
    },
    {
      "slug": "faith",
      "name": "Faith-based Message",
      "description": "Messages focused on faith and belief"
    }
  ]
}
~~~

### How Syncing Works

1. When a file is tagged, changes are stored in the `.kadu_tags` file.
1. If the directory is moved **within the repository**, the short paths remain valid.
1. When cloning the repository on another computer, tags are **automatically restored**.
1. The system matches files based on `uuid`, `short_path`, and `slug` to avoid duplicate or inconsistent tags.

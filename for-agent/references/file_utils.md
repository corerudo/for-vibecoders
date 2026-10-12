# File Utilities

Working with files and directories via the `file_utils` module.

Provides helpers to access standard Telegram directories and to read/write/list files.

## Standard directories

```python
from file_utils import (
    get_plugins_dir, get_cache_dir, get_files_dir, get_images_dir,
    get_videos_dir, get_audios_dir, get_documents_dir
)

plugins_path = get_plugins_dir()
cache_path = get_cache_dir()
files_path = get_files_dir()
images_path = get_images_dir()
videos_path = get_videos_dir()
audios_path = get_audios_dir()
documents_path = get_documents_dir()
```

## Directory operations

### `ensure_dir_exists`

Creates a directory (and parents) if it doesn't already exist.

```python
from file_utils import ensure_dir_exists, get_plugins_dir
import os

my_plugin_data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(my_plugin_data_dir)
```

### `list_dir`

Lists directory contents with options for recursion, filtering by type, and extension.

```python
from file_utils import list_dir, get_images_dir, get_cache_dir

image_files = list_dir(path=get_images_dir(), extensions=[".jpg", ".png"])

cache_subdirs = list_dir(
    path=get_cache_dir(), recursive=True, include_files=False, include_dirs=True,
)
```

Parameters: `recursive=False` (walk child directories), `include_files=True`, `include_dirs=False`, `extensions=None` (suffix filter, e.g. `[".json", ".txt"]`).

## File operations

### `write_file`

Writes a string to a file, overwriting if it already exists.

```python
from file_utils import write_file, get_plugins_dir
import os

data_to_save = "Hello, World!"
my_data_path = os.path.join(get_plugins_dir(), "my_plugin_data", "data.log")
write_file(my_data_path, data_to_save)
```

`write_file(...)` does NOT create parent directories — pair with `ensure_dir_exists(...)` when needed.

### `read_file`

Reads the entire content of a file into a string.

```python
from file_utils import read_file, get_plugins_dir
import os

my_config_path = os.path.join(get_plugins_dir(), "my_plugin_data", "config.txt")
config_content = read_file(my_config_path)

if config_content:
    log(f"Config loaded: {config_content}")
```

On failure, returns `None` and logs the exception.

### `delete_file`

```python
from file_utils import delete_file

file_to_delete = "/path/to/your/temp_file.tmp"
was_deleted = delete_file(file_to_delete)

if was_deleted:
    log("Temporary file deleted successfully.")
```

Returns `True` when deleted, `False` when deletion failed or the file did not exist.

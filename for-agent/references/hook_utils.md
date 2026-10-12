# Hook Utilities (Reflection)

Utility functions (module: `hook_utils`) for Java reflection: finding classes and accessing/modifying private fields and methods. Particularly useful for advanced Xposed hooking when you need non-public members of a class.

**Use with caution.** Reflection is powerful but fragile — it can break if the underlying app code changes. Always include error handling (`try/except`) and check for `None` return values.

## `find_class(class_name: str)`

Safely finds and returns a Java class object by its fully qualified name.

- `class_name`: full name including package, e.g. `"org.telegram.ui.ActionBar.ActionBar"`
- Returns: a Java `Class` object if found, otherwise `None`

```python
from hook_utils import find_class

ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")

if ActionBarClass:
    self.log(f"Successfully found class: {ActionBarClass.getName()}")
else:
    self.log("Could not find ActionBar class.")
```

## `get_private_field(obj: JavaObject, field_name: str)`

Accesses and retrieves the value of a private (or public) instance field from an object, searching the entire class hierarchy.

- `obj`: Java object instance
- `field_name`: name of the field
- Returns: the field value if found, otherwise `None`

```python
from hook_utils import get_private_field

# Assuming `chatActivity` is an instance of org.telegram.ui.ChatActivity
chat_list_view = get_private_field(chatActivity, "chatListView")

if chat_list_view:
    self.log("Successfully accessed chatListView.")
```

## `set_private_field(obj: JavaObject, field_name: str, new_value: Any)`

Modifies the value of a private (or public) instance field on an object.

- Returns: `True` if set successfully, `False` otherwise

```python
from hook_utils import set_private_field

user_object = ...
success = set_private_field(user_object, "verified", True)

if success:
    self.log("User is now verified!")
```

## `get_static_private_field(clazz: JavaClass, field_name: str)`

Accesses and retrieves the value of a static private (or public) field from a class.

```python
from hook_utils import find_class, get_static_private_field

ExteraConfigClass = find_class("com.exteragram.messenger.ExteraConfig")
if ExteraConfigClass:
    config_loaded = get_static_private_field(ExteraConfigClass, "configLoaded")
    self.log(f"Config loaded: {config_loaded}")
```

## `set_static_private_field(clazz: JavaClass, field_name: str, new_value: Any)`

Modifies the value of a static private (or public) field on a class.

- Returns: `True` if successful, `False` otherwise

```python
from hook_utils import find_class, set_static_private_field

BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
if BuildVarsClass:
    success = set_static_private_field(BuildVarsClass, "DEBUG_VERSION", True)
    if success:
        self.log("DEBUG_VERSION has been enabled.")
```

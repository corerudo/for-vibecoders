# Hook Utilities (Reflection)

Utility functions (module: `hook_utils`) for Java reflection: finding classes, resolving methods/constructors, and accessing or modifying private fields and methods. Particularly useful for Xposed method hooking when targeting non-public members of a class.

**Use with caution.** Reflection is powerful but can break if underlying app code changes across versions (or due to R8 obfuscation). Always wrap calls in `try/except` and check for `None` return values.

---

## Finding Classes

### `find_class(class_name: str)`
Safely finds and returns a Java class object by its fully qualified name:
```python
from hook_utils import find_class

ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if ActionBarClass:
    self.logger.info("Found class: %s", ActionBarClass.getName())
```

### `java_class(clazz: Any)`
Converts a class name or object into a Java `Class` instance:
```python
from hook_utils import java_class

cls = java_class("org.telegram.messenger.UserConfig")
```

---

## Methods and Constructors

### `get_method(clazz: Any, method_name: str, *param_types: Any)`
Finds a declared or inherited method on `clazz` matching the specified name and parameter types:
```python
from hook_utils import find_class, get_method
from java.lang import Integer

ChatActivity = find_class("org.telegram.ui.ChatActivity")
method = get_method(ChatActivity, "sendMessage", ...)
```

### `get_constructor(clazz: Any, *param_types: Any)`
Finds a specific constructor on `clazz`:
```python
from hook_utils import find_class, get_constructor

MessageObjectClass = find_class("org.telegram.messenger.MessageObject")
ctor = get_constructor(MessageObjectClass, ...)
```

### `get_field(clazz: Any, field_name: str)`
Resolves a declared or inherited Java `Field` reflection object:
```python
from hook_utils import find_class, get_field

field = get_field(find_class("org.telegram.ui.ChatActivity"), "chatListView")
```

---

## Private Instance Fields

### `get_private_field(obj: JavaObject, field_name: str)`
Accesses and retrieves the value of a private (or public) instance field from an object, traversing the entire class hierarchy:

```python
from hook_utils import get_private_field

# Assuming chatActivity is an instance of org.telegram.ui.ChatActivity
chat_list_view = get_private_field(chatActivity, "chatListView")
```

### `set_private_field(obj: JavaObject, field_name: str, new_value: Any)`
Modifies the value of a private (or public) instance field on an object. Returns `True` if set successfully:

```python
from hook_utils import set_private_field

user_object = ...
success = set_private_field(user_object, "verified", True)
```

---

## Private Static Fields

### `get_static_private_field(clazz: JavaClass, field_name: str)`
Retrieves the value of a private static field from a class:

```python
from hook_utils import find_class, get_static_private_field

SharedConfigClass = find_class("org.telegram.messenger.SharedConfig")
value = get_static_private_field(SharedConfigClass, "fontSize")
```

### `set_static_private_field(clazz: JavaClass, field_name: str, new_value: Any)`
Modifies the value of a private static field on a class:

```python
from hook_utils import find_class, set_static_private_field

SharedConfigClass = find_class("org.telegram.messenger.SharedConfig")
set_static_private_field(SharedConfigClass, "fontSize", 18)
```

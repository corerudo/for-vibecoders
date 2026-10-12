# Xposed Method Hooking

Hook Java methods and constructors from Python plugins using `MethodHook`, `MethodReplacement`, `BaseHook`, and filters.

Xposed-style method hooking lets a plugin intercept method calls inside the app, modify arguments, inspect/replace return values, or skip the original implementation entirely. This is powerful and easy to misuse — be explicit about which style you use.

## Hook Handler Styles

### 1. `MethodHook`

Use when you want `before_hooked_method(...)` and/or `after_hooked_method(...)`.

```python
from base_plugin import MethodHook


class TitleLoggerHook(MethodHook):
    def before_hooked_method(self, param):
        title = param.args[0]
        print(f"ActionBar title is about to become: {title}")

    def after_hooked_method(self, param):
        print(f"Method finished with result: {param.getResult()}")
```

### 2. `MethodReplacement`

Use to fully replace the original Java method.

```python
from base_plugin import MethodReplacement


class TitleReplacer(MethodReplacement):
    def replace_hooked_method(self, param):
        print("Original method was replaced.")
        return None
```

Your `replace_hooked_method(...)` becomes the whole implementation. Return: `None` for void, Python `bool`/`int`/`float` for primitive Java returns, or a compatible Java/Python object for reference returns.

### 3. Functional hooks / `BaseHook`

For simpler cases, skip a full class and pass `before=`/`after=` directly:

```python
self.hook_method(
    some_method,
    before=lambda param: self.log("Method is about to run!"),
    after=lambda param: self.log(f"Method finished with result: {param.getResult()}"),
)
```

Internally the SDK wraps this into `BaseHook`, a small helper subclass of `MethodHook`.

## The `param` Object

All hook callbacks receive `de.robv.android.xposed.XC_MethodHook.MethodHookParam`. Most useful members:

- `param.thisObject` — instance the method was called on, or `None` for static methods
- `param.args` — method argument array; readable and writable
- `param.method` — the hooked `Method` or `Constructor`
- `param.getResult()` — return value of the original method, available after execution
- `param.setResult(value)` — sets a result manually

Calling `param.setResult(...)` inside `before_hooked_method(...)` skips the original method:

```python
def before_hooked_method(self, param):
    if param.args[0] == "blocked":
        param.setResult(None)
```

## Filters

Filters decide whether a hook callback runs at all. Two styles: `@hook_filters(...)` on class-based hook methods, or `before_filters=[...]`/`after_filters=[...]` for functional hooks.

```python
from base_plugin import HookFilter

self.hook_method(
    some_method,
    before=lambda p: self.log("Arg 0 is null!"),
    before_filters=[HookFilter.ArgumentIsNull(0)],
)
```

Available `HookFilter` helpers: `RESULT_IS_NULL`, `RESULT_IS_TRUE`, `RESULT_IS_FALSE`, `RESULT_NOT_NULL`, `ResultIsInstanceOf(clazz)`, `ResultEqual(value)`, `ResultNotEqual(value)`, `ArgumentIsNull(index)`, `ArgumentNotNull(index)`, `ArgumentIsFalse(index)`, `ArgumentIsTrue(index)`, `ArgumentIsInstanceOf(index, clazz)`, `ArgumentEqual(index, value)`, `ArgumentNotEqual(index, value)`, `Condition(condition, object=None)`, `Or(*filters)`.

### Class-based filter example

```python
from base_plugin import HookFilter, MethodHook, hook_filters


class ExampleHook(MethodHook):
    @hook_filters(HookFilter.ArgumentIsNull(0))
    def before_hooked_method(self, param):
        ...

    @hook_filters(HookFilter.RESULT_NOT_NULL)
    def after_hooked_method(self, param):
        ...
```

### MVEL condition filter example

```python
from base_plugin import HookFilter, MethodHook, hook_filters


class ConditionalHook(MethodHook):
    @hook_filters(
        HookFilter.Condition(
            "param.args[0] == object || this instanceof android.view.View",
            object=500,
        )
    )
    def before_hooked_method(self, param):
        ...
```

## The Hooking Process

### 1. Find the target method or constructor

Start with `find_class(...)` (from `hook_utils`), then use Java reflection on the returned `Class`.

```python
from hook_utils import find_class

ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if not ActionBarClass:
    self.log("ActionBar class not found!")
    return

# Example method: public void setTitle(CharSequence title)
CharSequenceClass = find_class("java.lang.CharSequence")
set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)

# Example constructor: public ActionBar(Context context)
ContextClass = find_class("android.content.Context")
action_bar_constructor = ActionBarClass.getDeclaredConstructor(ContextClass)
action_bar_constructor.setAccessible(True)
```

**Do not call `getClass()` on the `Class` object** — `find_class(...)` already returns a Java `Class`. Call `getDeclaredMethod(...)`/`getDeclaredConstructor(...)` directly on it.

### 2. Implement the hook handler

```python
from base_plugin import MethodHook, MethodReplacement


class TitleLoggerHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        title = param.args[0]
        self.plugin.log(f"ActionBar title is being set to: {title}")
        param.args[0] = f"[Hooked] {title}"

    def after_hooked_method(self, param):
        self.plugin.log("ActionBar title has been set.")


class TitleReplacer(MethodReplacement):
    def __init__(self, plugin):
        self.plugin = plugin

    def replace_hooked_method(self, param):
        self.plugin.log("ActionBar.setTitle() was replaced.")
        return None
```

### 3. Apply the hook

```python
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
CharSequenceClass = find_class("java.lang.CharSequence")

set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)

handler_instance = TitleLoggerHook(self)
self.unhook_obj = self.hook_method(set_title_method, handler_instance, priority=10)

if self.unhook_obj:
    self.log("Successfully hooked ActionBar.setTitle()")
```

Hooks are removed automatically when the plugin unloads, but can also be unhooked manually.

### 4. Hook all methods or all constructors

```python
MyViewClass = find_class("com.example.MyCustomView")
unhook_list = self.hook_all_methods(MyViewClass, "onMeasure", TitleLoggerHook(self))

if unhook_list:
    self.log(f"Successfully hooked {len(unhook_list)} onMeasure overload(s).")
```

```python
SomeClass = find_class("com.example.SomeClass")
constructor_unhooks = self.hook_all_constructors(SomeClass, TitleLoggerHook(self))
```

### 5. Manual unhooking

```python
if self.unhook_obj:
    self.unhook_method(self.unhook_obj)
    self.unhook_obj = None
```

If `hook_all_methods(...)`/`hook_all_constructors(...)` were used, iterate the returned list and unhook each item.

## Practical Examples

### Example 1: modify arguments before the call

```python
from base_plugin import MethodHook
from hook_utils import find_class
from java import jint


class ToastHook(MethodHook):
    def before_hooked_method(self, param):
        # Method signature: makeText(Context context, CharSequence text, int duration)
        original_text = param.args[1]
        param.args[1] = f"(Plugin) {original_text}"


ToastClass = find_class("android.widget.Toast")
ContextClass = find_class("android.content.Context")
CharSequenceClass = find_class("java.lang.CharSequence")

make_text_method = ToastClass.getDeclaredMethod("makeText", ContextClass, CharSequenceClass, jint)
self.hook_method(make_text_method, ToastHook())
```

### Example 2: change the return value after the call

```python
from base_plugin import MethodHook
from hook_utils import find_class


class BuildVarsHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        original_result = param.getResult()
        self.plugin.log(f"Original result: {original_result}")
        param.setResult(False)


BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
is_main_app_method = BuildVarsClass.getDeclaredMethod("isMainApp")
self.hook_method(is_main_app_method, BuildVarsHook(self))
```

### Example 3: skip the original call in `before_hooked_method`

```python
from base_plugin import MethodHook
from hook_utils import find_class
from java.lang import Boolean, Long


class FormatFileSizeHook(MethodHook):
    def before_hooked_method(self, param):
        size = param.args[0]

        if size < 1024:
            param.setResult(f"{size} bytes (edited)")


AndroidUtilitiesClass = find_class("org.telegram.messenger.AndroidUtilities")
format_file_size_method = AndroidUtilitiesClass.getDeclaredMethod(
    "formatFileSize", Long.TYPE, Boolean.TYPE, Boolean.TYPE,
)
self.hook_method(format_file_size_method, FormatFileSizeHook())
```

### Example 4: fully replace a method

```python
from java.lang import String as JString

from base_plugin import MethodReplacement
from hook_utils import find_class


class NoOpLogger(MethodReplacement):
    def replace_hooked_method(self, param):
        # Original method is never called.
        return None


FileLogClass = find_class("org.telegram.messenger.FileLog")
log_method = FileLogClass.getDeclaredMethod("d", JString)
self.hook_method(log_method, NoOpLogger())
```

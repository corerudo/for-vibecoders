# Xposed Method Hooking

Hook Java methods and constructors from Python plugins using `MethodHook`, `MethodReplacement`, `BaseHook`, filters, and partial instruction hooks.

Xposed-style method hooking lets a plugin intercept method calls inside the app, modify arguments, inspect/replace return values, or skip the original implementation entirely.

---

## Hook Handler Styles

### 1. `MethodHook`

Use when you want `before_hooked_method(...)` and/or `after_hooked_method(...)`:

```python
from base_plugin import MethodHook


class TitleLoggerHook(MethodHook):
    def before_hooked_method(self, param):
        title = param.args[0]
        self.logger.info("ActionBar title is about to become: %s", title)

    def after_hooked_method(self, param):
        self.logger.info("Method finished with result: %s", param.getResult())
```

### 2. `MethodReplacement`

Use to fully replace the original Java method:

```python
from base_plugin import MethodReplacement


class TitleReplacer(MethodReplacement):
    def replace_hooked_method(self, param):
        self.logger.info("Original method replaced.")
        return None
```

Return: `None` for void, Python `bool`/`int`/`float` for primitive Java returns, or a compatible Java/Python object for reference returns.

### 3. Functional Hooks / `BaseHook`

For simpler cases, pass callbacks directly:

```python
self.hook_method(
    some_method,
    before=lambda param: self.logger.debug("Method about to run!"),
    after=lambda param: self.logger.debug("Method result: %s", param.getResult()),
)
```

---

## The `param` Object

Callbacks receive `de.robv.android.xposed.XC_MethodHook.MethodHookParam`:

- `param.thisObject` — instance the method was called on (`None` for static methods).
- `param.args` — list-like collection of method arguments. Mutable before method runs (`param.args[0] = new_value`).
- `param.getResult()` — returned value (valid in `after_hooked_method`).
- `param.setResult(new_result)` — replace the return value (skips original Java execution if called in `before_hooked_method`).
- `param.hasThrowable()` / `param.getThrowable()` — whether an exception occurred.
- `param.setThrowable(exception)` — override/suppress exception.

---

## Registering Hooks on `BasePlugin`

### Single Method / Constructor:
```python
unhook = self.hook_method(java_method, hook_handler, priority=10)
```

### All Overloads:
```python
unhooks = self.hook_all_methods(java_class, "methodName", hook_handler)
```

### All Constructors:
```python
unhooks = self.hook_all_constructors(java_class, hook_handler)
```

### Unhooking:
Registered hooks are automatically unhooked when the plugin unloads. To unhook manually:
```python
self.unhook_method(unhook)
```

---

## Partial Hooking (Instruction-Level Interception)

In newer SDK builds (1.4.6.x), plugins can hook specific instructions inside a method body without replacing the entire method:

### 1. `hook_call_site`
Intercepts an internal method invocation (`callee`) occurring inside `in_method`:
```python
def on_call(chain):
    # chain is PartialChain
    self.logger.info("Target called with args: %s", chain.args)
    return chain.proceed(*chain.args)

handle = self.hook_call_site(in_method, callee_method, on_call)
```

### 2. `hook_field_read` / `hook_field_write`
Intercepts field accesses inside a specific method:
```python
handle = self.hook_field_read(in_method, target_field, lambda chain: chain.proceed())
handle = self.hook_field_write(in_method, target_field, lambda chain: chain.proceed(modified_val))
```

### 3. `hook_string_constant`
Intercepts when a specific string literal is loaded inside `in_method`:
```python
handle = self.hook_string_constant(in_method, "Old Text", lambda chain: "New Text")
```

### 4. `hook_return_value`
Inspects or modifies the return value right before the method returns:
```python
handle = self.hook_return_value(in_method, lambda chain: modify(chain.proceed()))
```

### `PartialChain` Helper
In partial hook callbacks, `chain` provides:
- `chain.this_object` — the caller object instance.
- `chain.args` — list of arguments passed to the hooked operation.
- `chain.arg(index)` — specific argument.
- `chain.proceed(*args)` — execute the original operation with optional replacement args.
- `chain.throw_to_caller(throwable)` — throw an exception back to the caller.

Unhooking partial hooks:
```python
self.unhook_partial(handle)
```

---

## Hook Filters

Filters let you avoid running Python code if conditions aren't met:

```python
from base_plugin import HookFilter

# Only execute if result was not null:
self.hook_method(method, before=..., before_filters=[HookFilter.RESULT_NOT_NULL])

# Only execute if argument 0 equals a specific value:
self.hook_method(method, before=..., before_filters=[HookFilter.ArgumentEqual(0, "target")])
```

Supported filters:
- `HookFilter.RESULT_IS_NULL` / `RESULT_NOT_NULL`
- `HookFilter.RESULT_IS_TRUE` / `RESULT_IS_FALSE`
- `HookFilter.ResultIsInstanceOf(Class)`
- `HookFilter.ResultEqual(value)` / `ResultNotEqual(value)`
- `HookFilter.ArgumentIsNull(index)` / `ArgumentNotNull(index)`
- `HookFilter.ArgumentEqual(index, value)` / `ArgumentNotEqual(index, value)`
- `HookFilter.ArgumentIsInstanceOf(index, Class)`
- `HookFilter.Condition(mvel_expression)`
- `HookFilter.Or(*filters)`

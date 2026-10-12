# Class Proxy

Create Java subclasses and dynamic proxy classes from Python.

The class proxy API allows a plugin to create a real Java class at runtime and implement it in Python. Useful when a Java API expects: a subclass of an existing Java class, an implementation overriding parent methods, extra methods declared directly on the generated class, or Java fields stored on the generated object.

Built on top of DexMaker, so the result is an actual Java class, passable anywhere the app expects that type.

`new_instance()` returns a Python peer object, not the raw Java instance. Use `.java` when an API explicitly requires the generated Java object itself.

## Recommended API

The managed DSL from `extera_utils.classes`:

```python
from extera_utils.classes import (
    Base,
    java_subclass,
    joverride,
    joverload,
    jmethod,
    jMVELmethod,
    jMVELoverride,
    jclassbuilder,
    jfield,
    jgetmethod,
    jsetmethod,
    jconstructor,
    jpreconstructor,
)
```

Main pieces:
- `Base` — base Python class for managed Java subclasses
- `@java_subclass(JavaClass, Interface1, Interface2, ..., methods=..., constructors=..., custom_name=...)` — binds your Python class to a Java base class and optional interfaces
- `@joverride(...)` — overrides an existing Java method
- `@joverload(name, arg_types)` — overload-friendly alias for `@joverride`
- `@jmethod(...)` — adds a new method directly to the generated Java class
- `jMVELmethod(...)` — adds a Java method backed by MVEL instead of Python
- `jMVELoverride(...)` — overrides a parent Java method with MVEL code
- `@jclassbuilder()` — lets you modify DexMaker before the class is finalized/loaded
- `jfield(...)` — adds a Java field to the generated class
- `jgetmethod(...)`/`jsetmethod(...)` — generate Java-only getter/setter methods for a field
- `@jconstructor(...)` — runs Python code after the Java constructor has finished
- `@jpreconstructor(...)` — modifies constructor arguments before calling the Java parent constructor

Aliases: `jmvelmethod`/`jMVELmethod`, `jmveloverride`/`jMVELoverride`, `joverride`, `joverload`, `jmethod`, `jfield`, `jconstructor`, `jpreconstructor`, `jgetmethod`, `jsetmethod`, `jclassbuilder`.

## Quick Start

```python
from extera_utils.classes import Base, java_subclass, joverload, jfield
from java.util import ArrayList


@java_subclass(ArrayList)
class CountingList(Base):
    added_count = jfield("int", default=0)

    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.added_count += 1
        return super().add_item(value)


items = CountingList.new_instance()
items.add("Hello")
items.add("world")
```

`CountingList` becomes a generated Java subclass of `java.util.ArrayList`; `added_count` becomes a real Java field; `add(Object)` is overridden in Java and routed into Python; `super().add_item(...)` calls the original parent implementation. Because `ArrayList.add` is overloaded, `@joverload(...)` is used — for overloaded Java methods, prefer explicit signatures over bare `@joverride()`.

## Creating a Java Subclass

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    ...
```

With interfaces:

```python
@java_subclass(FrameLayout, CustomDelegate1, CustomDelegate2)
class MyCell(Base):
    ...
```

Produces: `class GeneratedProxy extends FrameLayout implements CustomDelegate1, CustomDelegate2`.

### Custom generated class name

Requires app version code >= 66690. Control the middle part of the generated Java proxy class name with `custom_name=...`:

```python
@java_subclass(Object, custom_name="DialogCell")
class Test(Base):
    ...
```

Generates: `Proxy_Object_DialogCell_<hashcode>_<randomnum>`. If `custom_name` omitted, the Python class name is used: `Proxy_Object_Test_<number>_<number>`. Same option available on `bind(...)`: `MySubclass.bind(SomeJavaClass, custom_name="SettingsRow")`. Names are sanitized (unsupported chars → `_`).

Generated proxy classes load through a shared generated-class loader chain, so a later proxy can reference an earlier one — but prefer referencing another proxy through an interface or common base class instead of depending on the exact generated class.

Binding after class definition:

```python
class MySubclass(Base):
    ...

MySubclass.bind(SomeJavaClass)
MySubclass.bind(FrameLayout, CustomDelegate1, CustomDelegate2)
```

### Creating instances

```python
proxy_java_class = MySubclass.java_class()

instance = MySubclass.new_instance()
instance2 = MySubclass.new_instance("text", 123)  # java-constructor args

java_instance = instance.java
java_instance2 = MySubclass.new_java_instance("text", 123)
```

To pass separate arguments to Python `__init__`, use `init_args`:

```python
instance = MySubclass.new_instance("java ctor value", init_args=["python init value"])
```

`"java ctor value"` is used for Java constructor resolution; `["python init value"]` is passed to Python `__init__`.

### Recovering a Python peer from a raw Java instance

```python
existing_java = MySubclass.new_java_instance("text")
peer = MySubclass.from_java(existing_java)
```

Useful when: a Java API created/stored the proxy instance for you, you received the generated object back through another callback, or you want to continue with the Python peer instead of the raw Java object.

## Overriding Methods

### Simple override (same name)

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Clicked")
```

### Override with another Python name

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride("equals")
    def equals_(self, a, b):
        return a == b
```

Useful for names like `equals`, `hashCode`, or several Python handlers for overloaded methods.

### Overloaded methods

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        print("int overload", value)

    @joverload("setValue", ["java.lang.String"])
    def set_value_text(self, value):
        print("string overload", value)
```

Equivalent with `joverride`: `@joverride("setValue", ["int"])`.

### Supported type formats

Argument/return types may be: primitive names (`"int"`, `"boolean"`, `"long"`), fully-qualified class names (`"java.lang.String"`), Java classes from `jclass(...)`, or Java primitive type objects.

```python
from java import jclass

View = jclass("android.view.View")

@joverload("setView", [View])
def set_view(self, view):
    ...
```

### Calling parent methods with `super()`

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Before parent")
        result = super().onClick(view)
        print("After parent")
        return result
```

For overloaded methods, call the Python method name defined:

```python
@joverload("setValue", ["int"])
def set_value_int(self, value):
    return super().set_value_int(value)
```

## Adding New Java Methods

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jmethod("debugLabel", "java.lang.String", ["int"])
    def build_debug_label(self, value):
        return f"value={value}"
```

Creates a real Java method `debugLabel(int)` on the generated class.

### `jmethod` signatures

```python
@jmethod("java.lang.String", ["int"])
def my_method(self, value):
    ...

@jmethod("debugLabel", "java.lang.String", ["int"])
def build_debug_label(self, value):
    ...
```

If return/argument types omitted, `jmethod` can infer from Python annotations:

```python
@jmethod()
def build_debug_label(self, value: "int") -> "java.lang.String":
    return f"value={value}"
```

Explicit types are still recommended when the Java signature matters.

## Adding MVEL Methods

For hot-path methods with simple logic that should stay on the Java side:

```python
from extera_utils.classes import Base, java_subclass, jMVELmethod


@java_subclass(SomeJavaClass)
class MySubclass(Base):
    debug_label = jMVELmethod(
        return_type="java.lang.String",
        arguments=[("prefix", "java.lang.String"), ("value", "int")],
        code="""
            return prefix + ": value=" + value + ", class=" + java.getClass().getSimpleName();
        """,
    )
```

Creates a real Java method `debug_label(String, int)`, implemented via compiled MVEL.

### Overriding with MVEL

```python
from extera_utils.classes import Base, java_subclass, jMVELoverride


@java_subclass(SomeFactoryClass)
class OptimizedFactory(Base):
    isClickable = jMVELoverride(
        arguments=[],
        code="return factory_is_clickable;",
    )
```

If the parent method is overloaded, pass explicit argument types like `@joverload(...)`:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    add = jMVELoverride(
        arguments=[("value", "java.lang.Object")],
        code="""
            add_calls = add_calls + 1;
            return SUPER_add(value);
        """,
    )
```

### MVEL Context

Inside MVEL: the root object is the generated Java instance (Java fields/methods accessible directly); `java` — same generated Java instance; `python`/`py`/`self` — the attached Python peer; `args` — `Object[]` with all method arguments; `argc` — number of arguments; `arg0, arg1, ...` — positional arguments; named arguments from `arguments=[("name", "type"), ...]`.

For override methods, a generated `SUPER_<methodName>(...)` bridge is also available:

```python
@java_subclass(SomeViewClass)
class MyView(Base):
    onMeasure = jMVELoverride(
        arguments=[("widthMeasureSpec", "int"), ("heightMeasureSpec", "int")],
        code="""
            SUPER_onMeasure(
                android.view.View$MeasureSpec.makeMeasureSpec(
                    android.view.View$MeasureSpec.getSize(widthMeasureSpec),
                    android.view.View$MeasureSpec.EXACTLY
                ),
                android.view.View$MeasureSpec.makeMeasureSpec(
                    org.telegram.messenger.AndroidUtilities.dp(68),
                    android.view.View$MeasureSpec.EXACTLY
                )
            );
            return null;
        """,
    )
```

### When to use MVEL

- Use `jfield(..., methods=[jgetmethod(...), jsetmethod(...)])` for pure field getter/setter hot paths — the fastest option.
- Use `jMVELmethod(...)`/`jMVELoverride(...)` when the method needs a bit of logic but you still want to avoid calling into Python every time.
- Keep MVEL bodies small and predictable; if logic needs Python objects, complex state, or heavy control flow, use regular Python overrides.

## Adding Java Fields

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    counter = jfield("int", default=0)
    title = jfield("java.lang.String", default="Hello")
```

Read/write like normal Python attributes:

```python
self.counter += 1
self.title = "Updated"
```

The field is generated on the Java class; the value is available through the Python peer as a normal attribute; assigning updates the attached Java instance when possible.

### Java-only field accessors

```python
from extera_utils.classes import jfield, jgetmethod, jsetmethod


@java_subclass(SomeJavaClass)
class MySubclass(Base):
    shadow_value = jfield(
        "boolean", default=False,
        methods=[jgetmethod("isShadow"), jsetmethod("setShadow")],
    )
```

Generates a Java field `shadow_value`, a Java method `isShadow()` returning it directly, and `setShadow(boolean)` writing it directly — no Python call at all, useful for very hot paths.

Useful to override a parent getter with a static value:

```python
@java_subclass(SomeFactoryClass)
class OptimizedFactory(Base):
    is_shadow_value = jfield("boolean", default=False, methods=[jgetmethod("isShadow")])
```

If `SomeFactoryClass` already has `isShadow()`, the generated method overrides it and returns the field immediately.

Generated proxy classes load through a shared generated-class loader chain, so a later proxy can reference an earlier one — but a stable shared type (interface/common base class) is still safer for public APIs.

## Constructors

Two stages: `@jpreconstructor(...)` (before the parent Java constructor call) and `@jconstructor(...)` (after the Java object already exists).

### Pre-constructor

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jpreconstructor(["java.lang.String"])
    def normalize_text(cls, text):
        return [text.strip()]
```

Return `None` to keep original arguments, a list/tuple with replacement arguments, or a single value for a one-argument constructor.

### Post-constructor

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    initialized = jfield("boolean", default=False)

    @jconstructor(["java.lang.String"])
    def init_from_text(self, text):
        self.initialized = True
        self.last_text = text
```

### `__init__` and `on_post_init`

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def __init__(self, text):
        self.python_only_state = text

    def on_post_init(self, text):
        print("Java object is ready:", self.java)
```

Initialization order: `@jpreconstructor(...)` if matched → Java parent constructor → Python `__init__(...)` → matched `@jconstructor(...)` → `on_post_init(...)`.

If you only need a single generic hook, `__init__` is usually enough; for different logic per overloaded constructor, use `@jconstructor(...)`.

### Separate Python init arguments

By default Python `__init__` receives the same arguments as the Java constructor. For different Python-only init data, use `new_instance(..., init_args=[...])`:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def __init__(self, title, debug_mode):
        self.title = title
        self.debug_mode = debug_mode


instance = MySubclass.new_instance("java constructor text", init_args=["Python title", True])
```

## Accessing the Attached Java Instance

Inside a managed subclass: `self.java` is the attached Java instance; `self.this` is an alias.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onAttached(self):
        print(self.java.getClass().getName())
```

If Python cannot find an attribute on the peer object, it falls back to the Java instance, so most Java calls work directly on the peer:

```python
instance = MySubclass.new_instance()
instance.hashCode()
instance.getClass().getName()
```

If a Java API checks the exact runtime type and expects a real Java object parameter, pass `instance.java`.

## Wrapping Existing Java Instances

Use `from_java(...)` to recover the Python peer instead of manually copying state:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onAttached(self):
        print("Attached!", self.java)


peer = MySubclass.new_instance()
raw_java = peer.java

# Later, possibly in another callback:
same_peer = MySubclass.from_java(raw_java)
same_peer.onAttached()
```

If the peer already exists, `from_java(...)` returns the same Python object; otherwise it creates and binds one.

## Passing Python Objects Through Java

Low-level helper `PyObj` carries an arbitrary Python object through Java-facing APIs:

```python
from extera_utils.classes import PyObj

payload = PyObj.create({"debug": True, "label": "factory args"})
```

Mainly useful when a Java API needs to carry opaque plugin-owned state, when building low-level adapters/factory layers, or to place Python data into a Java field or something like `UItem.object`. Normal subclassing/method overrides usually don't need `PyObj` directly.

## Built-in Java Object Modifiers

Available directly in the Externagram Chaquopy fork for any Java object/class, no wrappers needed:

```
obj.JA.privateField
obj.JIR.method1().method2()
SomeJavaClass.JGS.someStaticProperty
```

### Two groups of modifiers

**Persistent modifiers** (change behavior of the current Java object/class): `JUseGetterAndSetter`/`JGS`, `JNotUseGetterAndSetter`/`JNGS`, `JAccessAll`/`JA`, `JNotAccessAll`/`JNA`.

**Scoped chain modifiers** (only affect the current access chain): `JIgnoreResult`/`JIR`, `JNotIgnoreResult`/`JNIR`, `JSafe`/`JS`, `JNotSafe`/`JNS`.

### Default behavior

By default: getter/setter mode enabled, access-all disabled, safe mode disabled, ignore-result disabled.

```python
# If getTitle() exists, this uses it by default.
title = obj.title

# If setTitle(...) exists, this uses it by default.
obj.title = "Updated"
```

Applies to Java object instances and Java classes (static getters/setters).

### Getter/Setter mode

```python
obj.JNGS
value = obj.result  # reads the field path directly instead of trying getResult()
obj.JGS  # turn getter/setter mode back on
```

Same works on classes: `SomeJavaClass.JGS`, `caption = SomeJavaClass.caption`, `SomeJavaClass.caption = "New value"`.

Notes: getter/setter mode is for field-like names (`title`, `result`, `messageObject`); names that already look like methods (`getClass()`, `isInterface()`) stay method-like.

### AccessAll mode

```python
obj.JA
secret = obj.secretField
obj.secretMethod()
obj.JNA
```

Also works on classes for static members:

```python
SomeJavaClass.JA
token = SomeJavaClass.secretToken
SomeJavaClass.rebuildCache()
SomeJavaClass.JNA
```

Declared methods/fields/nested classes are cached per Java class, so `JA` does not re-scan reflection metadata every time.

### IgnoreResult mode

```python
obj.JIR.setTitle("Hello").requestLayout().invalidate()  # all calls made on obj
result = obj.methodThatReturnsSomething()  # normal behavior outside the chain
value = obj.JIR.method1().method2().method3().JNIR.method4()  # JNIR turns it off mid-chain
```

### Safe mode

Optional-style chaining — if any attribute/method is missing anywhere in the chain, the result becomes `JNone` instead of raising.

```python
value = obj.JS.someAttr.someOtherAttr.someMethod()
```

Safe mode propagates to Java objects returned during the chain (no need to repeat `.JS`), and treats `None` as safe-empty, converting it to `JNone`.

```python
value = obj.JS.info.details.title       # if getInfo() returns None, chain stays safe
value = obj.JS.missing.anything.call()  # missing step -> JNone instead of exception
```

Disable mid-chain with `JNS`.

### `JNone`

Sentinel returned by safe chains on failed access or a safe-chain step returning `None`. Falsy in boolean context; keeps swallowing attribute access and calls; can be compared by identity.

```python
from java import JNone

result = obj.JS.maybeNull.value

if result is JNone:
    print("No value")
```

### Practical behavior summary

- `JGS`/`JNGS` and `JA`/`JNA` are persistent for the current object or Java class.
- `JIR`/`JNIR` and `JS`/`JNS` only affect the current chain.
- `dir(obj)` and `dir(SomeJavaClass)` expose the modifier names.
- `JA` works with fields, methods, and nested classes/interfaces declared on the Java class.
- `JSafe` propagates through returned Java objects automatically and converts `None` to `JNone` inside the safe chain.

### Limitations

Convenience access, not a full replacement for manual signature-driven reflection. If exact overload control matters, use explicit reflection:

```python
clazz = obj.getClass()
method = clazz.getDeclaredMethod("setValue", jint)
method.setAccessible(True)
method.invoke(obj, 5)

# or
obj.JA
obj.value = 5  # uses setValue(5)
```

## Full Example

```python
from extera_utils.classes import (
    Base, java_subclass, joverride, joverload, jmethod, jfield, jconstructor, jpreconstructor,
)


@java_subclass(SomeJavaClass)
class DemoProxy(Base):
    counter = jfield("int", default=0)
    title = jfield("java.lang.String", default="Demo")

    @jpreconstructor(["java.lang.String", "int"])
    def normalize_args(cls, title, counter):
        return [title.strip(), max(0, counter)]

    def __init__(self, title, counter):
        self.python_state = {"created_with": title}

    @jconstructor(["java.lang.String", "int"])
    def init_fields(self, title, counter):
        self.title = title
        self.counter = counter

    @joverride()
    def onAttached(self):
        self.counter += 1
        return super().onAttached()

    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        self.counter = value
        return super().set_value_int(value)

    @joverload("setValue", ["java.lang.String"])
    def set_value_text(self, value):
        self.title = value

    @jmethod("debugLabel", "java.lang.String", ["int"])
    def build_debug_label(self, extra):
        return f"{self.title} ({self.counter}, extra={extra})"


instance = DemoProxy.new_instance("  Example  ", 5)
java_instance = instance.java
```

## Full Smoke Test Example

Verifies almost the entire class proxy API in one place using `java.util.ArrayList` (subclassable, with overloaded methods/constructors):

```python
from base_plugin import BasePlugin
from extera_utils.classes import (
    Base, java_subclass, jfield, joverride, joverload, jmethod, jconstructor, jpreconstructor,
)
from java.util import ArrayList


@java_subclass(ArrayList)
class DebugList(Base):
    add_calls = jfield("int", default=0)
    remove_calls = jfield("int", default=0)
    last_action = jfield("java.lang.String", default="created")

    @jpreconstructor(["int"])
    def normalize_capacity(cls, capacity):
        return [max(0, capacity)]

    def __init__(self, *args):
        self.history = []  # Python-only state, not a Java field

    @jconstructor(["int"])
    def init_with_capacity(self, capacity):
        self.last_action = f"capacity:{capacity}"
        self.history.append(f"ctor(capacity={capacity})")

    @joverride()
    def size(self):
        current = super().size()
        self.history.append(f"size() -> {current}")
        return current

    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.add_calls += 1
        self.last_action = f"add:{value}"
        self.history.append(f"add(value={value})")
        return super().add_item(value)

    @joverload("add", ["int", "java.lang.Object"])
    def add_at_index(self, index, value):
        self.add_calls += 1
        self.last_action = f"add_at:{index}:{value}"
        self.history.append(f"add(index={index}, value={value})")
        return super().add_at_index(index, value)

    @joverload("remove", ["int"])
    def remove_at_index(self, index):
        self.remove_calls += 1
        self.last_action = f"remove_at:{index}"
        self.history.append(f"remove(index={index})")
        return super().remove_at_index(index)

    @joverload("remove", ["java.lang.Object"])
    def remove_item(self, value):
        self.remove_calls += 1
        self.last_action = f"remove:{value}"
        self.history.append(f"remove(value={value})")
        return super().remove_item(value)

    @jmethod("debugSummary", "java.lang.String", [])
    def debug_summary(self):
        return (
            f"size={self.size()}, add_calls={self.add_calls}, "
            f"remove_calls={self.remove_calls}, last_action={self.last_action}"
        )

    @jmethod("historyDump", "java.lang.String", [])
    def history_dump(self):
        return " | ".join(self.history)


class ProxyDebugPlugin(BasePlugin):
    def on_plugin_load(self):
        debug_list = DebugList.new_instance(4)

        debug_list.add("one")
        debug_list.add("two")
        debug_list.add(1, "inserted")
        debug_list.remove("two")

        self.log(f"Class name: {debug_list.getClass().getName()}")
        self.log(f"Java size(): {debug_list.size()}")
        self.log(f"Python field add_calls: {debug_list.add_calls}")
        self.log(f"Python field remove_calls: {debug_list.remove_calls}")
        self.log(f"Java method debugSummary(): {debug_list.debugSummary()}")
        self.log(f"Java method historyDump(): {debug_list.historyDump()}")

        self.log(f"Same object hash: {debug_list.java.hashCode()}")
```

If everything works, this confirms: `DebugList.new_instance(4)` selects the `ArrayList(int)` constructor and returns the Python peer (`debug_list.java` is the raw Java instance); `@jpreconstructor(["int"])` runs before Java construction; `@jconstructor(["int"])` runs after; `add_calls`/`remove_calls`/`last_action` behave like Java-backed fields; correct overloaded `add`/`remove` handlers are called; `super()` reaches the original `ArrayList` implementation; `debugSummary()`/`historyDump()` exist as real Java methods.

## Custom Settings Example

The settings SDK uses this system internally for custom setting factories:

```python
from client_utils import get_media_data_controller
from base_plugin import BasePlugin
from ui.settings import Text, Custom
from extera_utils.classes import Base, java_subclass, joverride

from android.util import TypedValue
from android.view import View, Gravity
from android.widget import LinearLayout, FrameLayout, TextView
from android.content import Context

from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.ActionBar import Theme
from org.telegram.ui.Components import LayoutHelper, BackupImageView, UItem

from com.exteragram.messenger.plugins import PluginsController, Plugin
from com.exteragram.messenger.plugins.models import CustomSetting

MATCH_PARENT = -1
WRAP_CONTENT = -2

@java_subclass(FrameLayout)
class PluginCell(Base):
    def on_post_init(self, context: Context):
        self.need_divider = False
        self.setWillNotDraw(False)

        self.icon_view = BackupImageView(context)
        self.icon_view.setRoundRadius(AndroidUtilities.dp(8))
        self.addView(self.icon_view, LayoutHelper.createFrame(48, 48, Gravity.LEFT | Gravity.CENTER_VERTICAL, 16, 0, 0, 0))

        text_layout = LinearLayout(context)
        text_layout.setOrientation(LinearLayout.VERTICAL)

        self.name_text_view = TextView(context)
        self.name_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteBlackText))
        self.name_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
        self.name_text_view.setLines(1)
        self.name_text_view.setSingleLine(True)

        self.author_text_view = TextView(context)
        self.author_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteGrayText))
        self.author_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        self.author_text_view.setLines(1)
        self.author_text_view.setSingleLine(True)

        text_layout.addView(self.name_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT))
        text_layout.addView(self.author_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT, 0, 2, 0, 0))
        self.addView(text_layout, LayoutHelper.createFrame(MATCH_PARENT, WRAP_CONTENT, Gravity.LEFT | Gravity.CENTER_VERTICAL, 76, 0, 16, 0))

    def set_data(self, plugin: Plugin, divider: bool):
        self.need_divider = divider
        self.name_text_view.setText(plugin.getName() or "Unknown Plugin")

        version = plugin.getVersion() or "1.0"
        author = plugin.getAuthor() or "Unknown"
        self.author_text_view.setText(f"v{version} • {author}")

        show_icon = plugin.getPack() is not None and plugin.getIndex() >= 0
        pack, index = (plugin.getPack(), plugin.getIndex()) if show_icon else ("exteraPlugins", 0)
        get_media_data_controller().setPlaceholderImageByIndex(self.icon_view, pack, index, "100_100")

        self.requestLayout()

    @joverride()
    def onMeasure(self, widthMeasureSpec: int, heightMeasureSpec: int):
        super().onMeasure(
            View.MeasureSpec.makeMeasureSpec(View.MeasureSpec.getSize(widthMeasureSpec), View.MeasureSpec.EXACTLY),
            View.MeasureSpec.makeMeasureSpec(AndroidUtilities.dp(68), View.MeasureSpec.EXACTLY),
        )


@java_subclass(CustomSetting.Factory)
class PluginCellSettingFactory(Base):
    @joverride()
    def create(self, plugin: Plugin, setting: CustomSetting, args):
        if not args:
            return None

        item = UItem.ofFactory(self.java_class())
        item.object = args
        return item

    @joverride()
    def createView(self, context, list_view, current_account, class_guid, resources_provider):
        view = PluginCell.new_instance(context)
        return view.java

    @joverride()
    def bindView(self, view, item, divider, adapter, list_view):
        if not item.object:
            return False

        PluginCell.from_java(view).set_data(item.object, divider)


class TestPlugin(BasePlugin):
    def on_plugin_load(self):
        self.pcsfi = PluginCellSettingFactory.new_java_instance()
        UItem.UItemFactory.setup(self.pcsfi)

    def create_settings(self):
        plugins = PluginsController.getInstance().plugins.values().toArray()
        if not plugins:
            return [Text("Restart plugin please, no plugins found")]

        return [Custom(factory=self.pcsfi, factory_args=_p) for _p in plugins]
```

If you instantiate this proxy with `FactoryProxy.new_instance()`, the result is the Python peer. Pass `factory.java` into APIs which require `CustomSetting.Factory`.

## Advanced Notes

### Method Resolution Order

For managed overrides, the proxy system installs a Python bridge so `super()` calls the parent Java implementation. Use `super()` only for methods marked with `@joverride(...)` or `@joverload(...)`.

### Exact Signature Dispatch

When Java calls Python, the proxy system uses a full signature key internally, e.g. `methodName:int:java.lang.String` — this is what makes overloaded methods and constructors work correctly.

### Primitive Types

Supported: `void`, `boolean`, `byte`, `char`, `short`, `int`, `long`, `float`, `double`.

### When to use class proxy

Use it when: a Java API requires subclassing instead of interface proxying; you need parent-method calls with `super()`; you need overload-specific behavior; you want Java fields or extra Java methods on the generated object. Do not use it when a simple `dynamic_proxy(...)` is enough for an interface. Also confirm the base Java class is actually subclassable — final classes cannot be used as a proxy base class.

## Troubleshooting

**`super()` does not work** — check the method is declared with `@joverride(...)`/`@joverload(...)` and that the name/signature matches the Java parent method.

**My overload is not called** — check the Java name is correct, argument types match the real Java signature exactly, and primitive types use primitive names (`"int"`, not wrapper classes).

**Constructor hook is not called** — check `@jpreconstructor(...)`/`@jconstructor(...)` uses the correct constructor signature; for wildcard constructor hooks, annotations must be either complete or intentionally omitted.

**Field value is always `None`** — check the field was declared with `jfield(...)`, the field type is valid, and the default value is compatible with the declared Java type.

**Class generation fails immediately** — check the base class is not final, constructor signatures match a real constructor, and overload signatures match real Java method argument types.

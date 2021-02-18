---
date: 2020-09-08T12:00
tags:
  - blog/published
  - software/languages/python
---

# Magic Python Objects

## Defining and Encoding Strict Types Using BARE

![](https://images.unsplash.com/photo-1551269901-5c5e14c25df7?ixlib=rb-1.2.1&q=85&fm=jpg&crop=entropy&cs=srgb&ixid=eyJhcHBfaWQiOjYzOTIxfQ&w=3600)

Everything in Python is an object, and I mean everything. This includes primitives such as `int`, `float`, and `str` but also includes things you might not expect, such as `class` and function definitions. It turns out, that Python has a *lot* of flexibility and... weird things you can do with class fields.

<!--more-->

## Descriptors - But what do they describe?

`@property`, `super()`, and `@staticmethod` (and many others) are implemented using an *advanced* internal protocol called the descriptor protocol. In short, descriptors are objects that define 'binding' behavior in the form of overriding `__get__()`, `__set__()`, and `__delete__()`. 

Without going into detail too on how these functions work, at a basic level they're invoked whenever a property is set (ex. `setattr()`), read (ex. `getattr()`), or deleted (ex. `del` or `delattr()`). The protocol has some nuances to be aware of, I recommend reading the [Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html) for more information.

## That's cool, but what can I do with them?

As mentioned, common class definition and interaction utilities are created using the descriptor protocol. One more potential use is validation for values being set to a property. The `property.setter` decorator provides a method for doing this generically on classes, but what if you want to "statically" define a type for a class member? Well you can do that too! Here's an example:

```python
from abc import ABC, abstractmethod
class Field(ABC):

    def __set_name__(self, owner, name):
        # name is the attr name of *this* instance when assigned as a class field on
        # another object
        self.name = name
        # set the value we have wrapped (if any) to name prefixed with an '_' on the owner
        # instance
        setattr(owner, f"_{name}", self._value)

    def __get__(self, instance, owner=None):
        if instance is None:
            return self
        else:
            return getattr(instance, f"_{self.name}")

    def __set__(self, instance, value):
        if instance is None:
            raise AttributeError("Unable to assign value when not attached to object")
        valid, message = self.validate(value)
        if not valid:
            # defined elsewhere :)
            raise ValidationError(
                f"value is invalid for BARE type {self._type.__class__.__name__}: {message}"
            )
        setattr(instance, f"_{self.name}", value)

    @abstractmethod
    def validate(self, value) -> typing.Tuple[bool, str]:
        """
        Checks whether a give value is valid for the Field's data type. Returns a tuple of a boolean
        and an optional message for why
        """
        return self.__class__._default == None  # This is valid for BareType.Void
```

The above code is a snippet taken from my implementation of the [BARE message encoding protocol](https://baremessages.org/) for Python: [PyBARE](https://sr.ht/~chiefnoah/pybare/). It provides a declarative method for defining data containers that can easily be serialized and deserialized to byte streams.

The above defined `Field` classes primary purpose is to be subclassed with a specific type definition (all of the native BARE types are provided), and declared as class members on either `Struct` objects or some other data container. A simple declaration using some of the native types look like this:

```python
class Example(Struct):

    testint = Int()
    teststr = Str()
```

In this example, both `Int` and `Str` are subclasses of `Field`. When the above line is `import`ed or loaded by the interpreter, `Int.__set_name__()` and `Str.__set_name__()` are called with the names "testinst" and "teststr" respectively.

Note: even though `Int` and `Str` are instances, they do not currently hold real values. You must create an instance of `Example` and assign the values for testint and teststr.

Those names are store as instance attributes so when a value is assigned to them through an instance of `Example`, the value can be assigned to an internal representation of that value using `Field.__set__()`. That's as easy as you would expect:

```python
ex = Example()
ex.testint = 2
ex.teststr = "Hello world" # call looks like: Field.__set__(self, ex, "Hello world")
print(ex.teststr) # "Hello world"
print(ex._teststr) # "Hello world", the actual value is stored as an instance attribute on ex
```

The way the values are stored may seem a bit odd, but it's necessary to prevent multiple instances of `Example` from sharing values, given that the `testint` and `teststr` instances are class fields.

You may have noticed this peculiar line:
```python
    def __get__(self, instance, owner=None):
        if instance is None:
            return self
```

The `__get__()` method of `Field` checks if `instance` is `None` and returns itself if it is... huh??

This may seem unintuitive, but what it does is allows the `Field` to be accessed as it's instance (not it's wrapped value) when accessed as a *class attribute*. To clarify:

```python
ex = Example(testint=1, teststr="hello")
ex.testint # 1
Example.testint # <bare.Int object at ...>
```

This mimics "shadowing" behavior on the instance vs class versions of the attribute and is particularly helpful when serializing because you can simply call `self.__class__.testint.pack(self.testint)` to pack the value.

So what's with that `validate` method?

Well, one of the primary goals for PyBARE was to provide validation as early as possible. Most serializers wait until a method is called to serialize a data-structure to validate it. I'm of the opinion that if you're going through the trouble to explicitly define what is and isn't a valid value, you should error as soon as possible. Invalid data is always invalid. Yes, this may seem a bit antithetical to Python's duck-typing system, but when you have sufficiently complex data containers with strict typing requirements (and your physicist coworkers are married to `numpy`), you do what you can.

So what happens when you try to do something that doesn't fit in a data type? A `bare.ValidationError` is raised!

```python
ex = Example()
ex.testint = "1" # raises: ValidationError
ex.teststr = 1 # raises: ValidationError
```

If you absolutely *must* set one of these values to something that doesn't pass validation, you can always directly set the internal values:

```python
ex._testint = "1" # no validation
```

Alternatively, you can define your own `Field` type and `validation` method to do your own validation:

```python
class IntStr(Str):
    def validate(self, value) j-> Tuple[bool, str]:
        if not isinstance(value, (int, str)):
            return False, f"Wrote type: {type(value)} must be <str> or <int>"
        return True, None
```

The above code will fail if you try to serialize it, you must also define a `_pack` method in order to support both `str` and `int` types. This is considered an "advanced" usage of the library, I recommend looking at [types.py](https://git.sr.ht/~chiefnoah/pybare/tree/master/bare/types.py) for how this is done internally and imitating how it is done there.

The native BARE types should cover most needs, so you shouldn't need to define your own validation. 

This is a serialization library... right? Well, yeah... but serialization is actually secondary to being able to concretely define types and structures for a non-self-describing protocol such as BARE.

PyBARE puts an emphasis on streaming (unlike most Python libraries, as I've found). Though it's optional, you can provide a `BytesIO` object or similar (such as a file-like object opened in binary/b mode) to a `Struct`s `pack()` method, and it will write the bytes directly to that stream like so:

```python
ex = Example(teststr="Hello world", testint=3)
buffer = io.BytesIO()
ex.pack(fp=buffer) # writes output directly to buffer
buffer.seek(0) # reset the buffer so we can read from it
#now unpack
unpacked = ex.unpack(buffer)
unpacked.testint # 3
unpacked.teststr # "Hello world"
```

If you don't want to use streaming, you can omit the `fp` kwarg for `pack` and it will directly return the encoded `bytes`. For `unpack`, a `bytes`-like can be passed in and it will behave the same.

The primary benefit to streaming, aside from potentially better use of memory, is you can repeatedly serialize and deserialize messages on the same stream. This makes it easy to implement simple RPCs over pipes or other sockets or write many messages in an efficient manner.

---

This has only been a surface-level overview of what PyBARE can do. If you'd like to learn more, I encourage you to look at the several examples in [tests for the library](https://git.sr.ht/~chiefnoah/pybare/tree/master/bare/test_encoder.py) as well as [official spec for BARE.](https://baremessages.org/) Patches and feature requests are welcome!

[PyBARE is available on PyPi.](https://pypi.org/project/pybare/)

Thanks for reading! If you have any questions or comments, shoot me an email at [noah@packetlost.dev](mailto:noah@packetlost.dev) or <a href="https://twitter.com/intent/tweet?screen_name=chiefnoah13&ref_src=twsrc%5Etfw" class="twitter-mention-button" data-show-count="false">Tweet to @chiefnoah13</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

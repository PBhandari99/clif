# Python CLIF



To describe a C++ API in Python CLIF uses an Interface Description Language
(IDL) (that is a modified [PYTD](https://github.com/google/pytypedecl) language)
described below.


## Example

CLIF has a C++ API description in the .clif file:

```python
# 1. Load FileStat wrapper class from another CLIF wrapper.
from "file/base/python/filestat.h" import *  # FileStat
# 2. Load Status from hand-written C++ CLIF extension library.
from "util/task/python/clif.h" import *  # Status
# 3. Load Options protobuf from generated C++ CLIF extension library.
from  "file/base/options_pyclif.h" import *
# 4. Load pure-Python postprocessor function for Status
from clif.python.postproc import DropOkStatus

from "file/base/filesystem.h":
  namespace `file`:
    def ForEachMatch(pattern: str, options: Options,
                     match_handler: (filename:str, fs:FileStat)->bool
                    ) -> Status:
      return DropOkStatus(...)
    # define other API items here (if needed)
```

Line 1 gets a class wrapped by another CLIF module.
Line 2 gets a [custom wrap](/util/task/python/g3doc/status.md) for `Status` and
`StatusOr`.
Line 3 gets a wrapped option.proto (generated by `pyclif_proto_library` BUILD
rule).

**Note:** Callback signature above matches
`std::function<bool (StringPiece, file:Stats)>`.


## API specification language

From that example we see that .clif file has 2 sections:

 * preparation (import symbols we'll need in the next section),
 * API description (tell which C++ API we'll need from a particular header).

**Preparation** specifies which CLIF extension libraries are needed and
what C++ library we are wrapping.
It can have [c header import](#cimport), [python import](#pyimport),
[namespace](#namespace) and [use](#use) statements.

**API description** starts with _from_ statement that points to the C++ header
file we wrap and has an indented block describing the API.
That block might have the following statements:

 * [def](#def)
 * [const](#const)
 * [enum](#enum)
 * [class](#class)
 * [staticmethods](#staticmethods)

which are described below.

NOTE: Some features are "experimental" which means they can be changed or
removed in the future releases.


### c_header import statement {#cimport}

The _c header import_ statement makes types wrapped by another CLIF rule or by
a C++ CLIF extension library available to use in this .clif file.
Such library can be written [by hand](ext.md) or generated by a tool (like CLIF
protobuf wrapper - it generates a cc_library CLIF extension.)

```python
from "cpp/include/path/to/aCLIF/extension/library.h" import *
```

**Note** that _c header import_ requires a double-quoted string exactly as the
C++ `#include` directive.

Use _c header import_ statement to inform CLIF about wrapped C++ types that
needs to be available in the module being wrapped.

If you don't want to pollute .clif namespace with all names from that header,
you can prefix imported names with a variant of include statement:

```python
from "some/header.h" import * as prefix_name
```

Now all CLIF types defined in the `header.h` (with
``CLIF use `ctype` as clif_type``) will be available as `prefix_name.clif_type`.


### python import statement {#pyimport}

The _python import_ statement is a normal Python import to make a library
symbol available within the .clif file. Only a single symbol import allowed
(not a module). All imports must be absolute.

```python
from path.to.project.library.module import SomeClassOrFunction
```

This statement is typically used to load a Python [postprocessing function]
(#postprocessing).


### from statement {#from}

The _from_ statement tells CLIF what library file to wrap.
This statement allows top-level API name lookup in any namespace in the
specified file.

```python
from "cpp/include/path/to/some/library.h":
  # API description statements
```

### namespace statement {#namespace}

The _namespace_ statement tells CLIF what C++ namespace to use (backquotes
are required around the C++ name).
That namespace must be declared in the _from_'d file.
This statement limits top-level API name lookup to the specified namespace.

```python
from "cpp/include/path/to/some/library.h":
  namespace `my::namespace`:
    def Name()  # API description statements
```

WARNING: Namespace statements can't be nested.


### def statement {#def}

The _def_ statement describes a C++ function (or member function).

```python
def NAME ( INPUT_PARAMETERS ) OUTPUT_PARAMETERS
```

It has three main parts: the name, input parameters and output parameters.

**NAME** can be a simple alphanumeric name when we want it to be the same in C++
and Python. In some cases we want or need to rename the C++ name to have a
different name in the Python wrapper. In those cases rename construct can be
used:

```python
`cplusplus_name` as python_name
```

For example `` `size` as __len__``[^len] or `` `pass` as pass_``.
Such renaming can occur everywhere a NAME is used.

[^len]: When exposing a C++ function as `__len__` make sure it only returns a
        non-negative numbers or Python will raise a `SystemError`.
        

**INPUT_PARAMETERS** describes values to be converted from Python and passed to
the C++ function. It is a (potentially empty) comma-separated list of
`name:type` pairs, ie. `x:int, descriptive_name:str`. Both `name` and `type`
are required (Only `self` in class methods has no type.)
For a type you use a Python standard type. Python containers should also be
typed (like `list<int>` or `dict<bytes, int>`).

#### Default parameters

If C++ has a default argument (ie. with `= value` clause), it can also be
optional in PYTD. Just add `=default` to its `name:type` specification.

**OUTPUT_PARAMETERS** are more complex:

  * can be empty, meaning no return values (Python wrapper returns None)
  * can be a single value in the form `-> type`, or
  * can be multiple return values: `-> (name1:type1, name2:type2, ...)`
    like input parameters.

By Google [convention]
(https://google.github.io/styleguide/cppguide.html#Function_Parameter_Ordering)
C++ signature should have all input parameters before any
output parameter(s). The first output parameter is the function return value and
others are listed after inputs as C++ `TYPE*` (pointer to output type). CLIF
does not allow you to violate those conventions. To circumvent that restriction
write a helper C++ function and wrap it instead.

For example:

C++ function  | described as
------------- | ------------
void F()      | def F()
int F()       | def F() -> int
void F(int)   | def F(name_is_mandatory: int)
int F(int)    | def F(name_is_mandatory: int) -> int
int F(string*)| def F() -> (code: int, message: str)

#### Pointers, references and object ownership

CLIF wraps of C++ functions with output parameters or return values of type
`std::unique_ptr` transfer object ownership to Python. Wraps of
`std::unique_ptr` input parameters transfer ownership to C++.

Raw pointers are always assumed to be borrowed; if a C++ API treats a raw
pointer as an ownership transfer, a C++ wrapping function or overload using
`std::unique_ptr` can be added to make the transfer explicit to CLIF.

#### Postprocessing {#postprocessing}

Often C/C++ APIs return a status as one return value. Python users prefer to
not see a good status at all and get an exception on a bad status. To get that
behavior, CLIF supports Python postprocessor functions that will take return
value(s) from C++ and transform them.

The standard CLIF library comes with the following postprocessor functions:

 * `ValueErrorOnFalse`
   takes first return value as bool, drops it from output if True or raise a
   ValueError if it's False.

 * `chr`
   is a Python built-in function useful to convert int/uint8 (from C++ char) to
   a Python 1-character string.

To use a postprocessor function you must first import it[^chr] with a
[python import](#pyimport) statement but remember to import the proper Python
name, not just the module. And use the extended `def` syntax as shown below:

[^chr]: Except `chr` that is already 'imported' by CLIF.

```python
def NAME ( INPUT_PARAMETERS ) OUTPUT_PARAMETERS:
  return PostProcessorFunction(...)
```

where `...` are three dots verbatim for all OUTPUT_PARAMETERS to be passed
as args to the PostProcessorFunction.

#### Asynchronous execution

CLIF decides when to release GIL. The general idea is to release GIL on every
C++ function call except for properties / variable access, default constructor
and Python object parameters.

To avoid releasing the GIL during a C++ call mark the function with `@unsafe`
decorator.

Since Python code is serialized by GIL it is thread-safe (but user data
structures might not be!).

Asynchronous execution takes advantage of multiple cores if C++ code:

  * does disk or network IO,
  * uses C++ locks or even a static variable inside a function,
  * runs long computations or in general is very CPU intensive,
  * does call back into Python.

WARNING: When your C++ code deal with PyObject directly, it must handle GIL
itself and should be marked @unsafe because it's unsafe to release GIL before
calling such code.

CLIF scans the API and marks the call as unsafe to release the GIL if it sees
an `object` in the function signature.

NOTE: If you need to, use `@do_not_release_gil` to call C++ with GIL held.
(It is an uncommon technique and may be prone to a deadlock between GIL and
a C++ lock.)

#### Implementing (virtual) methods in Python

You can implement C++ virtual function in Python. To do so derive a Python class
from the CLIF-wrapped C++ class and just define the member function with the
proper name.

To allow a Python implementation of a derived class to be called from C++ (via a
pointer to the base class) mark the function with a `@virtual` decorator.

Do not decorate C++ virtual methods with @virtual unless you need to implement
them in Python.

#### Implementing special Python methods

Those methods usually have double underscore in names (`__dunder__`).
When Python API require them to return `self`, use `-> self` in the signature.
Otherwise match the C++ signature and CLIF will try to conform to the Python
API.

C++ implements operators inside or outside of the class (aka member and
non-member operators. Keep such class API description Pythonic, CLIF will find
the non-member operator implementation by itself.
You can even use out of class function as class members, but they should take
the class instance (`this`) as the first parameter.

#### Context manager

To use a wrapped C++ class as a Python [context manager](https://docs.python.org/2/library/stdtypes.html#typecontextmanager),
some methods must be wrapped as `__enter__` and `__exit__`.
However Python has a different calling convention. To help wrap such cases use
CLIF method decorators to force the Python API:

  * `@__enter__` to call the wrapped method on `__enter__` and return
    `self` as the context manager instance, and
  * `@__exit__` to take the required [PEP-343]
    (https://www.python.org/dev/peps/pep-0343/) (type, value, traceback) args
    on `__exit__`, call the wrapped method with no arguments, and return None.

However if the C++ method provides the Python-needed API it can be simply
renamed:

    def `c_implementation_of_exit` as __exit__(self,
        type: object, value: object, trace: object) -> bool

WARNING: Be careful, when you use `object` CLIF assumes you **know** what you're
doing.


### const statement {#const}

The _const_ statement describes a C++ global or member constant
(const or constexpr).

```c++
const NAME: TYPE
```

It also makes sense to rename the constant to make it Python-style conformant:

```c++
const `kNumTries` as NUM_TRIES: int
```

### enum statement {#enum}

The _enum_ statement describes a C++ enum or enum class. This form will take all
enum values under the same names as they are known to C++.

```c++
enum NAME
```

It also makes sense to rename enum values to match expected Python style:

```c++
enum NAME with:
  `kDefault` as DEFAULT
  `kOptionOne` as OPTION_ONE
```

C++ enums will be presented as Python `Enum` or `IntEnum`[^enum] classes from
the standard `enum` module [backported to Python 2.7][^pypi].

[^enum]: C++ 11 `class enum` converted to `Enum`, old-style `enum` to `IntEnum`.
[^pypi]: https://pypi.python.org/pypi/enum34

### class statement {#class}

The _class_ statement describes a C++ struct or class. It must have
an indented block describing what class members are wrapped. That block can have
all the statements that the [from](#from) block has and a [var](#var) statement
for member variables.

Each member method should have a specific first argument:

  * `self` for regular C++ member functions
  * `cls` for static C++ member functions

The first argument (self/cls) should not have any type as the type is implicit
(it's the class that the function is a member of).

Also static member functions should have `@classmethod` decorator or moved to
the module level with a [_staticmethods_](#staticmethods) statement.

```python
class MyClass:
  def __init__(self, param: int, another_param: float)
  def Method(self) -> dict<int, int>
  @classmethod
  def StaticMethod(cls, param: int) -> MyClass
```

TIP: Always use Python module-level functions for exposing class static member
functions unless you have a very good reason not to.


The above snippet is better written as:

```python
class MyClass:
  def __init__(self, param: int, another_param: float)
  def Method(self) -> dict<int, int>
staticmethods from `MyClass`:
  def StaticMethod(param: int) -> MyClass
```

#### Inheritance

CLIF inheritance specification does not need to follow the C++ inheritance
relationship.
Only specify the base class if it is important for the Python API. CLIF is
capable to figure out the C++ inheritance details even if the `.clif` file does
not explicitly list them.

If the C++ class has no parent, no parent should be in the CLIF specification.
If the C++ class has a parent but it's of no interest to a Python user, the
parent also should be omitted and relevant parent methods should be listed
in the child class CLIF specification.

```c++
class Parent {
 public:
  void Something() = 0;
  void SomethingInteresting();
};

class Child : public Parent {
 public:
  void Useful();
};
```

A CLIF specification for that might look like the following.

```python
class Child:
  def SomethingInteresting(self)
  def Useful(self)
```

If the parent C++ class is already wrapped in the CLIF specification, use it as
you normally do with a parent Python class (`class Child(Parent):`).

Multiple inheritance in CLIF declaration is prohibited, but the C++ class
being wrapped may have multiple parents according to the
.
To change what C++ type CLIF will use for a given Python type the user
can either

  * specify an exact C++ type you want to use (ie. `` -> `size_t` as int``) or
  * change the default with the [use](#use) statement.


### Predefined types

CLIF knows some basic types (predefined in `clif/python/types.h`):

Default C++ type| CLIF type[^type]
--------------- | ------------
int             | int
string          | bytes _or_ str
bool            | bool
double          | float
vector<>        | list<>
pair<>          | tuple<>
unordered_set<> | set<>
unordered_map<> | dict<>
PyObject*       | object[^object] 

[^type]: CLIF types named after the corresponding Python types.
[^object]: Be careful when you use `object`, CLIF assumes you **know**
           what you're doing with Python C API and all its caveats.

CLIF also knows how to handle several compatible types

Python type | Compatible input C++ types
------------|---------------------------
float | float 
dict  | map
set   | set
list  | list, array, stack, deque, queue, priority_queue

To use a compatible type mention it explicitly in the declaration, e.g.

```python
def fsum(array: `std::list` as list<float>) -> `float` as float
```

for `float fsum(std::list<float>);` C++ function.

NOTE: CLIF will reject unknown types and produce an error. It can be parse-time
error for CLIF types or compile-time error for C++ types.


### Unicode

Please note that we want the C++ API to be explicit and while C++ does not
distinguish between bytes and unicode, Python does. It means that Python .clif
files must specify what exact type (bytes or unicode) the C++ code expects or
produces.

However, CLIF

always takes Python unicode and implicitly encodes it using UTF-8 for C++.
To get unicode back to Python 2, use `unicode` as the return datatype.
In Python 3, `str` gets converted to unicode automatically.

That can be summarized as below.

CLIF type | On input | On output CLIF returns
----------|----------|-----------------------
bytes     | (*)      | bytes
str       | (*)      | native str
unicode   | (*)      | unicode

(*) CLIF will take bytes or unicode Python object and pass [UTF-8 encoded] data
to C++.

#### Encoding

UTF-8 encoding assumed on C++ side.

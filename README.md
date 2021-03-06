# Requirements

* Python 3.6 or higher
* Pip (usually installed alongside Python)

# Setup

* Install `virtualenv` if you already haven't. Makes it a lot easier to keep
  Python dependencies and modules isolated from the rest of your environment.

  ```
  pip install --user virtualenv

  # After this you might have to add the virtualenv binary
  # to your $PATH.
  ```

* Create and activate a new virtual environment.

 ```
 virtualenv -p python3 py3
 source py3/bin/activate
 ```

* Install dependencies.

 ```
 pip install -r requirements.txt
 ```

# Running the example

* `firebase_functions` contains the experimental Functions SDK for Python.

* `sample.py` contains some sample user code written using the Functions SDK.

* Run the codegen tool to generate an entrypoint. Save the output to a new
  `app.py` file.

 ```
 python ./firebase_functions/codegen.py ./sample.py > app.py
 ```

 * Start the Flask server to serve the generated entrypoint.

  ```
  gunicorn -b :8080 app:app
  ```

* Start a separate server to serve the `backend.yaml`.

  ```
  gunicorn -b :8081 app:backend_yaml
  ```

* To trigger the HTTP function:

 ```
 curl localhost:8080/http_function
 ```

* To trigger the PubSub function:

 ```
 curl -X POST -d '{"uid": "alice"}' localhost:8080/pubsub_function
 ```

* To get the discovery yaml (currently served on the same port):

 ```
 curl localhost:8081/backend.yaml
 ```

# Implementation notes

## Decorators

Python supports higher-order functions. That is functions can accept and return other functions.
In the following example, `say_hello()` accepts another function

```py
def say_hello(func):
  print(f"Hello, {func()}")

def alice():
  return "Alice"

say_hello(alice) # Prints "Hello, Alice"
```

Decorators provide syntactic sugar around this type of higher-order function manipulation.
Here's the same example implemented as a decorator.

```py
def say_hello(func):
  def wrapper():
    print(f"Hello, {func()}")
  return wrapper

@say_hello
def alice():
  return "Alice"

alice() # Prints "Hello, Alice"
```

Here, `say_hello()` is a decorator function. When applied on another function it augments the
behavior of that function. It essentially replaces `alice()` in-place with `say_hello(alice)`.
Note that the result of `say_hello(alice)` in this case is a new `wrapper()` function.

Replacing functions in-place like above has some ugly consequences. For instance, inspecting
the replaced function with `print(alice)` yields the output
`<function say_hello.<locals>.wrapper at 0x107153430>` which exposes the internals of the
decorator. A more idiomatic way to do this is to implement the decorator using the utils
provided in the built-in `functools` API:

```py
import functools

def say_hello(func):
  @functools.wraps(func)
  def wrapper():
    print(f"Hello, {func()}")
  return wrapper

@say_hello
def alice():
  return "Alice"
```

This retains the original context information of the `alice()` function, and as a result
`print(alice)` will produce `<function alice at 0x103cb7430>`.

In this SDK we use decorators to add extra metadata to the functions implemented by developers.
Specifically, each decorator adds a new `firebase_metadata` property to developer's functions.
Later, our codegen tool will extract this metadata to generate the necessary manifest files
for deployment.

## Decorators with arguments

Consider the following decorator.

```py
def https(_func=None, *, min_instances=None, max_instances=None, memory_mb=None):
```

Here aguments after `*` are keyword-only arguments. That is they must be specified with
keywords (names) -- e.g. `min_instances=1`. Just passing the value `1` without the
keyword would result in an error.

In the above signature all the arguments are optional. Therefore a developer may apply
it without any arguments, or with some arguments. When applied without arguments, Python
will pass the target decorated function as `_func`. But when applied with one or more
arguments `_func` actually gets set to `None`. Therefore the decorator implementation
must explicitly handle both cases:

```py
if _func is None:
  return https_with_options

return https_with_options(_func)
```

Now consider the following decorator.

```py
def pubsub(*, topic, min_instances=None):
```

This also defines a decorator that requires 2 keyword arguments. But one of them (`topic`) is
defined without a default value making it a required argument. As a result, when applied, the
target decorated function never gets passed into this decorator. So we essentially have only
one case to handle:

```py
return pubsub_with_topic
```

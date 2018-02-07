---
layout: post
title: Runtime Type Checking in Python 3.6
---

Have you ever wished you could slow down your Python 3.6 code while at the same
time making it harder to profile and debug with the only benefit being that your
code will crash at runtime if you pass the wrong type?  Well, your wait is over!

# Introduction

With [this](https://gist.github.com/bheklilr/372dc851ba085c4f943f116e41888fcf)
snippet of code (Python 3.6 compatible only, I'm afraid), you can take advantage
of variable type annotations on assignments to dynamically check that you're
using the right types.  Think of this as a runtime [`mypy`](http://mypy-
lang.org/) that doesn't work as well:

```python
@enforce
def safe_doubler(x):
    y: int = x * 2
    return y

def unsafe_doubler(x):
    y: int = x * 2
    return y
```

With the runtime behavior:

```python
>>> unsafe_doubler(1)
2
>>> unsafe_doubler('1')
'11'
>>> safe_doubler(1)
2
>>> safe_doubler('1')
TypeViolationError: Expected y to have type int, got type str
```

# Pulling Back the Curtain

So how does it work?  Mainly through a really nasty hack due to (what I think
is) a deficiency in how Python 3.6 handles [variable
annotations](https://www.python.org/dev/peps/pep-0526/).  Namely, that Python
doesn't handle them at all.  The annotations are parsed and are included in the
[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree), but after that stage
all variable annotations are simply discarded. They aren't even evaluated, they
just have to be a valid Python expression:

```python
>>> def test():
...     x: ThisNameDoesNotExist = 1
...     return x
>>> test()
1
```

(Python does keep function and method annotations, however, which makes it
easily possible to build tooling around them.)

So how does this decorator extract those variable annotations and enforce them?
Simple, it just re-parses the function's source code, locates all cases of the
`AnnAssign` syntax node, and inserts an `isinstance` check after the assignment.

This can be performed for user-defined functions using `inspect.getsourcelines`,
as seen in the definition of `transform`:

```python
def transform(visitor):
    def deco(function):
        # Load the source code for the function
        source = ''.join(inspect.getsourcelines(function)[0])
        # Parse it
        module = ast.parse(source)
        # Extract the function's AST object as the
        # only element of the module's body
        func = module.body[0]

        # Using the given visitor, transform the AST of the function
        # !!! THIS IS THE MAGIC !!!
        visitor.visit(func)

        # Do some cleanup on the AST
        ast.fix_missing_locations(func)

        # Recompile the function's code dynamically
        module_code = compile(module, '<string>', 'exec')

        # Inject the TypeViolationError into the function's globals
        globs = function.__globals__.copy()
        globs['TypeViolationError'] = TypeViolationError
        # Return a new function using the compiled function
        return types.FunctionType(
            module_code.co_consts[0],
            globs,
            function.__name__,
        )
    return deco
```

Surprisingly, getting the source code of a function and parsing can be done in
just a line or two of Python.  After that, it's simply a matter of transforming
the AST and recompiling it.  You'll notice that the function's globals namespace
has to be modified in order to include the `TypeViolationError` class.  If I had
used a simple `TypeError` this wouldn't be necessary, but I like the custom
error class in this case.

In order to use this generic `transform` decorator, you just have to provide it
with a `ast.NodeTransformer` instance, which uses the [visitor
pattern](https://en.wikipedia.org/wiki/Visitor_pattern) to walk over the entire
syntax tree.

In the case of `enforcer`, the `NodeTransformer` implementation is:

```python
class TypeCheckingVisitor(ast.NodeTransformer):
    def visit_AnnAssign(self, node):
        name = node.target.id
        type_ = node.annotation.id
        return [
            node,
            ast.If(
                test=not_ast(isinstance_ast(name, type_)),
                body=[raise_type_violation_error_ast(name, type_)],
                orelse=[],
            )
        ]
```

The functions `not_ast`, `isinstance_ast`, and `raise_type_violation_error_ast`
are simply wrappers around the syntax `not <EXPR>`, `isinstance(name, type_)`,
and `raise TypeViolationError('name', expected_type, type(name))` respectively
(you can see their full definitions in the gist).

By returning a list of nodes from `visit_AnnAssign`, this visitor a function
like:

```python
def test():
    x: int = 1
```

Into a function like

```python
def test():
    x: int = 1
    if not isinstance(x, int):
        raise TypeViolationError('x', int, type(x))
```

# But Why?

I have a coworker working on an application that relies heavily on coroutines.
It uses typed message passing between multiple components and he kept finding
himself writing code along the lines of

```python
response_msg: ResponseMessage = (yield RequestMessage(a_value))
if not isinstance(response_msg, ResponseMessage):
    raise TypeError(...)
# process response_msg...

another_msg: AnotherMessage = (yield DifferentMessage())
if not isinstance(another_msg, AnotherMessage):
    raise TypeError(...)
# process another_msg...
```

While this architectural style came with many benefits, notably having all
components of this application be completely decoupled from each other, it
required a lot of discipline to make sure that the messages passed between
components were correct.  He was lamenting that the flexibility of Python that
allowed us to develop this application quickly was now coming back to haunt him.

I thought to myself "surely this is a problem that can be solved in Python", and
after about 30 minutes of googling I this hacked together.  I have to admit that
I was surprised that the only way to get at these type annotations at runtime
was to re-parse the function's source code and dynamically build a new one to
take it's place.  I wish that Python 3.6 and PEP-526 had made it possible to
access these annotations at runtime without having to resort to such underhanded
methods, but I understand that storing and evaluating them would potentially add
significant overhead to fully annotated code.

# Improvements

I haven't even tested this code out on methods, but my guess is that it won't
work on anything defined inside a class.  Closures _should_ work, and coroutines
definitely do.

It's unfortunate to me that each function must get parsed and compiled twice by
Python for this magic to work.  Maybe it would be possible to set this up as an
encoding similar to the [future-fstrings](https://github.com/asottile/future-
fstrings) project to avoid this.

This code does not handle cases like

```python
def breaks():
    (a, b): (int, str) = 1, '1'
    c: typing.Union[int, str] = random.choice([1, '1'])
    # Equivalent to d: int = None
    d: int
```

Or actual type checking in general.  It would be interesting to see if this sort
of work could be combined with `mypy` to provide true "compile time" type
checking where possible (i.e. via static analysis or at import) and runtime type
checking where Python can't prove the types are correct (such as values passed
in via `coroutine.send`).

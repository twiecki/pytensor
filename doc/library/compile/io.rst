
.. note::

    ***TODO*** Freshen up this old documentation


.. _function_inputs:

===========================================
:mod:`io` - defines pytensor.function [TODO]
===========================================

.. module:: pytensor.compile.io
   :platform: Unix, Windows
   :synopsis: defines In and Out
.. moduleauthor:: LISA


Inputs
======

The ``inputs`` argument to ``pytensor.function`` is a list, containing the ``Variable`` instances for which values will be specified at the time of the function call.  But inputs can be more than just Variables.
``In`` instances let us attach properties to ``Variables`` to tell function more about how to use them.


.. class:: In(object)

   .. method:: __init__(variable, name=None, value=None, update=None, mutable=False, strict=False, autoname=True, implicit=None)

      ``variable``: a Variable instance. This will be assigned a value
      before running the function, not computed from its owner.

      ``name``: Any type. (If ``autoname_input==True``, defaults to
      ``variable.name``). If ``name`` is a valid Python identifier, this input
      can be set by ``kwarg``, and its value can be accessed by
      ``self.<name>``. The default value is ``None``.

      ``value``: literal or ``Container``. The initial/default value for this
        input. If update is`` None``, this input acts just like
        an argument with a default value in Python. If update is not ``None``,
        changes to this
        value will "stick around", whether due to an update or a user's
        explicit action.

      ``update``: Variable instance. This expression Variable will
      replace ``value`` after each function call. The default value is
      ``None``, indicating that no update is to be done.

      ``mutable``: Bool (requires value). If ``True``, permit the
      compiled function to modify the Python object being used as the
      default value. The default value is ``False``.

      ``strict``: Bool (default: ``False`` ). ``True`` means that the value
      you pass for this input must have exactly the right type. Otherwise, it
      may be cast automatically to the proper type.

      ``autoname``: Bool. If set to ``True``, if ``name`` is ``None`` and
      the Variable has a name, it will be taken as the input's
      name. If autoname is set to ``False``, the name is the exact
      value passed as the name parameter (possibly ``None``).

      ``implicit``: Bool or ``None`` (default: ``None``)
            ``True``: This input is implicit in the sense that the user is not allowed
            to provide a value for it. Requires ``value`` to be set.

            ``False``: The user can provide a value for this input. Be careful
            when ``value`` is a container, because providing an input value will
            overwrite the content of this container.

            ``None``: Automatically choose between ``True`` or ``False`` depending on the
            situation. It will be set to ``False`` in all cases except if
            ``value`` is a container (so that there is less risk of accidentally
            overwriting its content without being aware of it).


Value: initial and default values
---------------------------------

A non-None `value` argument makes an In() instance an optional parameter
of the compiled function.  For example, in the following code we are
defining an arity-2 function ``inc``.

>>> import pytensor.tensor as at
>>> from pytensor import function
>>> from pytensor.compile.io import In
>>> u, x, s = at.scalars('u', 'x', 's')
>>> inc = function([u, In(x, value=3), In(s, update=(s+x*u), value=10.0)], [])

Since we provided a ``value`` for ``s`` and ``x``, we can call it with just a value for ``u`` like this:

>>> inc(5)         # update s with 10+3*5
[]
>>> print(inc[s])
25.0

The effect of this call is to increment the storage associated to ``s`` in ``inc`` by 15.

If we pass two arguments to ``inc``, then we override the value associated to
``x``, but only for this one function call.

>>> inc(3, 4)      # update s with 25 + 3*4
[]
>>> print(inc[s])
37.0
>>> print(inc[x])   # the override value of 4 was only temporary
3.0

If we pass three arguments to ``inc``, then we override the value associated
with ``x`` and ``u`` and ``s``.
Since ``s``'s value is updated on every call, the old value of ``s`` will be ignored and then replaced.

>>> inc(3, 4, 7)      # update s with 7 + 3*4
[]
>>> print(inc[s])
19.0

We can also assign to ``inc[s]`` directly:

>>> inc[s] = 10
>>> inc[s]
array(10.0)

Input Argument Restrictions
---------------------------

The following restrictions apply to the inputs to ``pytensor.function``:

- Every input list element must be a valid ``In`` instance, or must be
  upgradable to a valid ``In`` instance. See the shortcut rules below.

- The same restrictions apply as in Python function definitions:
  default arguments and keyword arguments must come at the end of
  the list. Un-named mandatory arguments must come at the beginning of
  the list.

- Names have to be unique within an input list.  If multiple inputs
  have the same name, then the function will raise an exception. [***Which
  exception?**]

- Two ``In`` instances may not name the same Variable. I.e. you cannot
  give the same parameter multiple times.

If no name is specified explicitly for an In instance, then its name
will be taken from the Variable's name. Note that this feature can cause
harmless-looking input lists to not satisfy the two conditions above.
In such cases, Inputs should be named explicitly to avoid problems
such as duplicate names, and named arguments preceding unnamed ones.
This automatic naming feature can be disabled by instantiating an In
instance explicitly with the ``autoname`` flag set to False.


Access to function values and containers
----------------------------------------

For each input, ``pytensor.function`` will create a ``Container`` if
``value`` was not already a ``Container`` (or if ``implicit`` was ``False``). At the time of a function call,
each of these containers must be filled with a value. Each input (but
especially ones with a default value or an update expression) may have a
value between calls. The function interface defines a way to get at
both the current value associated with an input, as well as the container
which will contain all future values:

  - The ``value`` property accesses the current values. It is both readable
    and writable, but assignments (writes) may be implemented by an internal
    copy and/or casts.

  - The ``container`` property accesses the corresponding container.
    This property accesses is a read-only dictionary-like interface. It is
    useful for fetching the container associated with a particular input to
    share containers between functions, or to have a sort of pointer to an
    always up-to-date value.

Both ``value`` and ``container`` properties provide dictionary-like access based on three types of keys:

- integer keys: you can look up a value/container by its position in the input list;
- name keys: you can look up a value/container by its name;
- Variable keys: you can look up a value/container by the Variable it corresponds to.

In addition to these access mechanisms, there is an even more convenient
method to access values by indexing a Function directly by typing
``fn[<name>]``, as in the examples above.

To show some examples of these access methods...


>>> from pytensor import tensor as at, function
>>> a, b, c = at.scalars('xys') # set the internal names of graph nodes
>>> # Note that the name of c is 's', not 'c'!
>>> fn = function([a, b, ((c, c+a+b), 10.0)], [])

>>> # the value associated with c is accessible in 3 ways
>>> fn['s'] is fn.value[c]
True
>>> fn['s'] is fn.container[c].value
True

>>> fn['s']
array(10.0)
>>> fn(1, 2)
[]
>>> fn['s']
array(13.0)
>>> fn['s'] = 99.0
>>> fn(1, 0)
[]
>>> fn['s']
array(100.0)
>>> fn.value[c] = 99.0
>>> fn(1,0)
[]
>>> fn['s']
array(100.0)
>>> fn['s'] == fn.value[c]
True
>>> fn['s'] == fn.container[c].value
True


Input Shortcuts
---------------

Every element of the inputs list will be upgraded to an In instance if necessary.

- a Variable instance ``r`` will be upgraded like ``In(r)``

- a tuple ``(name, r)`` will be ``In(r, name=name)``

- a tuple ``(r, val)`` will be ``In(r, value=value, autoname=True)``

- a tuple ``((r,up), val)`` will be ``In(r, value=value, update=up, autoname=True)``

- a tuple ``(name, r, val)`` will be ``In(r, name=name, value=value)``

- a tuple ``(name, (r,up), val)`` will be ``In(r, name=name, value=val, update=up, autoname=True)``

Example:

>>> import pytensor
>>> from pytensor import tensor as at
>>> from pytensor.compile.io import In
>>> x = at.scalar()
>>> y = at.scalar('y')
>>> z = at.scalar('z')
>>> w = at.scalar('w')

>>> fn = pytensor.function(inputs=[x, y, In(z, value=42), ((w, w+x), 0)],
...                      outputs=x + y + z)
>>> # the first two arguments are required and the last two are
>>> # optional and initialized to 42 and 0, respectively.
>>> # The last argument, w, is updated with w + x each time the
>>> # function is called.

>>> fn(1)               # illegal because there are two required arguments # doctest: +ELLIPSIS
Traceback (most recent call last):
  ...
TypeError: Missing required input: y
>>> fn(1, 2)            # legal, z is 42, w goes 0 -> 1 (because w <- w + x)
array(45.0)
>>> fn(1, y=2)        # legal, z is 42, w goes 1 -> 2
array(45.0)
>>> fn(x=1, y=2)    # illegal because x was not named # doctest: +ELLIPSIS
Traceback (most recent call last):
  ...
TypeError: Unknown input or state: x. The function has 3 named inputs (y, z, w), and 1 unnamed input which thus cannot be accessed through keyword argument (use 'name=...' in a variable's constructor to give it a name).
>>> fn(1, 2, 3)         # legal, z is 3, w goes 2 -> 3
array(6.0)
>>> fn(1, z=3, y=2) # legal, z is 3, w goes 3 -> 4
array(6.0)
>>> fn(1, 2, w=400)   # legal, z is 42 again, w goes 400 -> 401
array(45.0)
>>> fn(1, 2)            # legal, z is 42, w goes 401 -> 402
array(45.0)

In the example above, ``z`` has value 42 when no value is explicitly given.
This default value is potentially used at every function invocation, because
``z`` has no ``update`` or storage associated with it.

.. _function_outputs:

Outputs
=======

The ``outputs`` argument to function can be one of

- ``None``, or
- a Variable or ``Out`` instance, or
- a list of Variables or ``Out`` instances.

An ``Out`` instance is a structure that lets us attach options to individual output ``Variable`` instances,
similarly to how ``In`` lets us attach options to individual input ``Variable`` instances.

**Out(variable, borrow=False)** returns an ``Out`` instance:

  * ``borrow``

    If ``True``, a reference to function's internal storage
    is OK.  A value returned for this output might be clobbered by running
    the function again, but the function might be faster.

    Default: ``False``




If a single ``Variable`` or ``Out`` instance is given as argument, then the compiled function will return a single value.

If a list of ``Variable`` or ``Out`` instances is given as argument, then the compiled function will return a list of their values.

>>> import numpy
>>> from pytensor.compile.io import Out
>>> x, y, s = at.matrices('xys')

>>> # print a list of 2 ndarrays
>>> fn1 = pytensor.function([x], [x+x, Out((x+x).T, borrow=True)])
>>> fn1(numpy.asarray([[1,0],[0,1]]))
[array([[ 2.,  0.],
       [ 0.,  2.]]), array([[ 2.,  0.],
       [ 0.,  2.]])]

>>> # print a list of 1 ndarray
>>> fn2 = pytensor.function([x], [x+x])
>>> fn2(numpy.asarray([[1,0],[0,1]]))
[array([[ 2.,  0.],
       [ 0.,  2.]])]

>>> # print an ndarray
>>> fn3 = pytensor.function([x], outputs=x+x)
>>> fn3(numpy.asarray([[1,0],[0,1]]))
array([[ 2.,  0.],
       [ 0.,  2.]])

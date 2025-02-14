.. _extending_op_params:

===============
Using Op params
===============

The Op params is a facility to pass some runtime parameters to the
code of an op without modifying it.  It can enable a single instance
of C code to serve different needs and therefore reduce compilation.

The code enables you to pass a single object, but it can be a struct
or python object with multiple values if you have more than one value
to pass.

We will first introduce the parts involved in actually using this
functionality and then present a simple working example.

The params type
----------------

You can either reuse an existing type such as :class:`Generic` or
create your own.

Using a python object for your op parameters (:class:`Generic`) can be
annoying to access from C code since you would have to go through the
Python-C API for all accesses.

Making a purpose-built class may require more upfront work, but can
pay off if you reuse the type for a lot of Ops, by not having to re-do
all of the python manipulation.

The params object
-----------------

The object that you use to store your param values must be hashable
and comparable for equality, because it will be stored in a dictionary
at some point.  Apart from those requirements it can be anything that
matches what you have declared as the params type.

Defining a params type
~~~~~~~~~~~~~~~~~~~~~~

.. note::

    This section is only relevant if you decide to create your own type.

The first thing you need to do is to define an PyTensor Type for your
params object.  It doesn't have to be complete type because only the
following methods will be used for the type:

  - :meth:`filter <Type.filter>`
  - :meth:`__eq__ <Type.__eq__>`
  - :meth:`__hash__ <Type.__hash__>`
  - :meth:`values_eq <Type.values_eq>`

Additionally if you want to use your params with C code, you need to extend `COp`
and implement the following methods:

  - :meth:`c_declare <CLinkerType.c_declare>`
  - :meth:`c_init <CLinkerType.c_init>`
  - :meth:`c_extract <CLinkerType.c_extract>`
  - :meth:`c_cleanup <CLinkerType.c_cleanup>`

You can also define other convenience methods such as
:meth:`c_headers <CLinkerType.c_headers>` if you need any special things.


Registering the params with your Op
-----------------------------------

To declare that your Op uses params you have to set the class
attribute :attr:`params_type` to an instance of your params Type.

.. note::

   If you want to have multiple parameters, PyTensor provides the convenient class
   :class:`pytensor.link.c.params_type.ParamsType` that allows to bundle many parameters into
   one object that will be available in both Python (as a Python object) and C code (as a struct).
   See :ref:`ParamsType tutorial and API documentation <libdoc_graph_params_type>` for more infos.

For example if we decide to use an int as the params the following
would be appropriate:

.. code-block:: python

   class MyOp(Op):
       params_type = Generic()

After that you need to define a :meth:`get_params` method on your
class with the following signature:

.. code-block:: python

   def get_params(self, node)

This method must return a valid object for your Type (an object that
passes :meth:`filter`).  The `node` parameter is the Apply node for
which we want the params.  Therefore the params object can depend on
the inputs and outputs of the node.

.. note::

    Due to implementation restrictions, None is not allowed as a
    params object and will be taken to mean that the Op doesn't have
    parameters.

    Since this will change the expected signature of a few methods, it
    is strongly discouraged to have your :meth:`get_params` method
    return None.


Signature changes from having params
------------------------------------

Having declared a params for your Op will affect the expected
signature of :meth:`perform`.  The new expected signature will have an
extra parameter at the end which corresponds to the params object.

.. warning::

   If you do not account for this extra parameter, the code will fail
   at runtime if it tries to run the python version.

Also, for the C code, the `sub` dictionary will contain an extra entry
`'params'` which will map to the variable name of the params object.
This is true for all methods that receive a `sub` parameter, so this
means that you can use your params in the :meth:`c_code <COp.c_code>`
and :meth:`c_init_code_struct <COp.c_init_code_struct>` method.


A simple example
----------------

This is a simple example which uses a params object to pass a value.
This `Op` will multiply a scalar input by a fixed floating point value.

Since the value in this case is a python float, we chose Generic as
the params type.

.. testcode::

   from pytensor.link.c.op import COp
   from pytensor.link.c.type import Generic
   from pytensor.scalar import as_scalar

   class MulOp(COp):
       params_type = Generic()
       __props__ = ('mul',)

       def __init__(self, mul):
           self.mul = float(mul)

       def get_params(self, node):
           return self.mul

       def make_node(self, inp):
           inp = as_scalar(inp)
           return Apply(self, [inp], [inp.type()])

       def perform(self, node, inputs, output_storage, params):
           # Here params is a python float so this is ok
           output_storage[0][0] = inputs[0] * params

       def c_code(self, node, name, inputs, outputs, sub):
           return ("%(z)s = %(x)s * PyFloat_AsDouble(%(p)s);" %
                   dict(z=outputs[0], x=inputs[0], p=sub['params']))


A more complex example
----------------------

This is a more complex example which actually passes multiple values.
It does a linear combination of two values using floating point
weights.

.. testcode::

   from pytensor.graph.op import Op
   from pytensor.link.c.type import Generic
   from pytensor.scalar import as_scalar

   class ab(object):
       def __init__(self, alpha, beta):
           self.alpha = alpha
           self.beta = beta

       def __hash__(self):
           return hash((type(self), self.alpha, self.beta))

       def __eq__(self, other):
           return (type(self) == type(other) and
                   self.alpha == other.alpha and
                   self.beta == other.beta)


   class Mix(COp):
       params_type = Generic()
       __props__ = ('alpha', 'beta')

       def __init__(self, alpha, beta):
           self.alpha = alpha
           self.beta = beta

       def get_params(self, node):
           return ab(alpha=self.alpha, beta=self.beta)

       def make_node(self, x, y):
           x = as_scalar(x)
           y = as_scalar(y)
           return Apply(self, [x, y], [x.type()])

       def c_support_code_struct(self, node, name):
           return """
           double alpha_%(name)s;
           double beta_%(name)s;
           """ % dict(name=name)

       def c_init_code_struct(self, node, name, sub):
           return """{
           PyObject *tmp;
           tmp = PyObject_GetAttrString(%(p)s, "alpha");
           if (tmp == NULL)
             %(fail)s
           alpha_%(name)s = PyFloat_AsDouble(tmp);
           Py_DECREF(%(tmp)s);
           if (PyErr_Occurred())
             %(fail)s
           tmp = PyObject_GetAttrString(%(p)s, "beta");
           if (tmp == NULL)
             %(fail)s
           beta_%(name)s = PyFloat_AsDouble(tmp);
           Py_DECREF(tmp);
           if (PyErr_Occurred())
             %(fail)s
           }""" % dict(name=name, p=sub['params'], fail=sub['fail'])

       def c_code(self, node, name, inputs, outputs, sub):
           return """
           %(z)s = alpha_%(name)s * %(x)s + beta_%(name)s * %(y)s;
           """ % dict(name=name, z=outputs[0], x=inputs[0], y=inputs[1])

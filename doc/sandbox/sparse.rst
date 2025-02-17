.. _sparse:

===============
Sparse matrices
===============

scipy.sparse
------------

Note that you want SciPy >= 0.7.2

.. warning::

    In SciPy 0.6, `scipy.csc_matrix.dot` has a bug with singleton
    dimensions. There may be more bugs. It also has inconsistent
    implementation of sparse matrices.

    We do not test against SciPy versions below 0.7.2.

We describe the details of the compressed sparse matrix types.
    `scipy.sparse.csc_matrix`
        should be used if there are more rows than column (``shape[0] > shape[1]``).
    `scipy.sparse.csr_matrix`
        should be used if there are more columns than rows (``shape[0] < shape[1]``).
    `scipy.sparse.lil_matrix`
        is faster if we are modifying the array. After initial inserts,
        we can then convert to the appropriate sparse matrix format.

The following types also exist:
    `dok_matrix`
        Dictionary of Keys format. From their doc: This is an efficient structure for constructing sparse matrices incrementally.
    `coo_matrix`
        Coordinate format. From their lil doc: consider using the COO format when constructing large matrices.

There seems to be a new format planned for SciPy 0.7.x:
    `bsr_matrix`
        Block Compressed Row (BSR). From their doc: The Block Compressed Row
        (BSR) format is very similar to the Compressed Sparse Row (CSR)
        format. BSR is appropriate for sparse matrices with dense sub matrices
        like the last example below. Block matrices often arise in vector-valued
        finite element discretizations. In such cases, BSR is considerably more
        efficient than CSR and CSC for many sparse arithmetic operations.
    `dia_matrix`
        Sparse matrix with DIAgonal storage

There are four member variables that comprise a compressed matrix ``sp`` (for at least csc, csr and bsr):

    ``sp.shape``
        gives the shape of the matrix.
    ``sp.data``
        gives the values of the non-zero entries. For CSC, these should
        be in order from (I think, not sure) reading down in columns,
        starting at the leftmost column until we reach the rightmost
        column.
    ``sp.indices``
        gives the location of the non-zero entry. For CSC, this is the
        row location.
    ``sp.indptr``
        gives the other location of the non-zero entry. For CSC, there are
        as many values of indptr as there are ``columns + 1`` in the matrix.
        ``sp.indptr[k] = x`` and ``indptr[k+1] = y`` means that column
        ``k`` contains ``sp.data[x:y]``, i.e. the ``x``-th through the y-1th non-zero values.

See the example below for details.

.. code-block:: python

    >>> import scipy.sparse
    >>> sp = scipy.sparse.csc_matrix((5, 10))
    >>> sp[4, 0] = 20
    SparseEfficiencyWarning: changing the sparsity structure of a csc_matrix is expensive. lil_matrix is more efficient.
     SparseEfficiencyWarning)
    >>> sp[0, 0] = 10
    >>> sp[2, 3] = 30
    >>> sp.todense()
    matrix([[ 10.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.],
            [  0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.],
            [  0.,   0.,   0.,  30.,   0.,   0.,   0.,   0.,   0.,   0.],
            [  0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.],
            [ 20.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.,   0.]])
    >>> print sp
      (0, 0)        10.0
      (4, 0)        20.0
      (2, 3)        30.0
    >>> sp.shape
    (5, 10)
    >>> sp.data
    array([ 10.,  20.,  30.])
    >>> sp.indices
    array([0, 4, 2], dtype=int32)
    >>> sp.indptr
    array([0, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3], dtype=int32)

Several things should be learned from the above example:

* We actually use the wrong sparse matrix type. In fact, it is the
  *rows* that are sparse, not the columns. So, it would have been
  better to use ``sp = scipy.sparse.csr_matrix((5, 10))``.
* We should have actually created the matrix as a `lil_matrix`,
  which is more efficient for inserts. Afterwards, we should convert
  to the appropriate compressed format.
* ``sp.indptr[0] = 0`` and ``sp.indptr[1] = 2``, which means that
  column 0 contains ``sp.data[0:2]``, i.e. the first two non-zero values.
* ``sp.indptr[3] = 2`` and ``sp.indptr[4] = 3``, which means that column
  three contains ``sp.data[2:3]``, i.e. the third non-zero value.

TODO: Rewrite this documentation to do things in a smarter way.

Speed
-----

For faster sparse code:
  * Construction: lil_format is fast for many inserts.
  * Operators: "Since conversions to and from the COO format are
    quite fast, you can use this approach to efficiently implement lots
    computations on sparse matrices." (Nathan Bell on scipy mailing list)

Misc
----
The sparse equivalent of `dmatrix` is `csc_matrix` and `csr_matrix`.

:class:`~pytensor.sparse.basic.Dot` vs. :class:`~pytensor.sparse.basic.StructuredDot`
---------------------------------------------------------------------------------

Often when you use a sparse matrix it is because there is a meaning to the
structure of non-zeros. The gradient on terms outside that structure
has no meaning, so it is computationally efficient not to compute them.

`StructuredDot` is when you want the gradient to have zeroes corresponding to
the sparse entries in the matrix.

`TrueDot` and `Structured` dot have different gradients
but their perform functions should be the same.

The gradient of `TrueDot` can have non-zeros where the sparse matrix had zeros.
The gradient of `StructuredDot` can't.

Suppose you have ``dot(x,w)`` where ``x`` and ``w`` are square matrices.
If ``w`` is dense, like ``standard_normal((5,5))`` and ``x`` is of full rank (though
potentially sparse, like a diagonal matrix of ones) then the output will
be dense too.
What's important is the density of the gradient on the output.
If the gradient on the output is dense, and ``w`` is dense (as we said it was)
then the ``True`` gradient on ``x`` will be dense.
If our dot is a `TrueDot`, then it will say that the gradient on ``x`` is dense.
If our dot is a `StructuredDot`, then it will say the gradient on ``x`` is only
defined on the diagonal and ignore the gradients on the off-diagonal.

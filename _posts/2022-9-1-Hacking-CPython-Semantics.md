---
layout: post
title: "Hacking CPython: Semantics"
---

In it, we implement the almost-equal operator described in [CPython Internals](https://realpython.com/products/cpython-internals-book/).

### Expected Behavior

In this series, we implement an almost-equal operator in CPython. This operator, denoted by `~=`, is capable of performing equality comparisons between integer (`int`) and floating-point (`float`) types. When performing a comparison between an `int` and a `float`, the `float` argument is implicitly converted to an `int` before the comparison is performed. With two `int` operands, the operator behaves identically to equality comparison with `==`.

As described in [the book](https://realpython.com/products/cpython-internals-book/), the almost-equal operator should behave as follows:

```Python
>>> 1 ~= 1
True
>>> 1 ~= 1.0
True
>>> 1 ~= 1.1
True
>>> 1 ~= 1.9
True
```

In `Include/object.h`. Add a `#define` for the almost-equal operator.

```c
/* Rich comparison opcodes */
#define Py_LT 0
#define Py_LE 1
#define Py_EQ 2
#define Py_NE 3
#define Py_GT 4
#define Py_GE 5
#define Py_AlE 6
```

In `Include/object.h`

```c
/*
 * Macro for implementing rich comparisons
 *
 * Needs to be a macro because any C-comparable type can be used.
 */
#define Py_RETURN_RICHCOMPARE(val1, val2, op)                               \
    do {                                                                    \
        switch (op) {                                                       \
        case Py_EQ: if ((val1) == (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  \
        case Py_NE: if ((val1) != (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  \
        case Py_LT: if ((val1) < (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;   \
        case Py_GT: if ((val1) > (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;   \
        case Py_LE: if ((val1) <= (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  \
        case Py_GE: if ((val1) >= (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  \
        case Py_AlE: if ((val1) == (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE; \
        default:                                                            \
            Py_UNREACHABLE();                                               \
        }                                                                   \
    } while (0)
```

In `Object/object.c`

```c
PyObject *
PyObject_RichCompare(PyObject *v, PyObject *w, int op)
{
    PyThreadState *tstate = _PyThreadState_GET();

    assert(Py_LT <= op && op <= Py_AlE);
    if (v == NULL || w == NULL) {
        if (!_PyErr_Occurred(tstate)) {
            PyErr_BadInternalCall();
        }
        return NULL;
    }
    if (_Py_EnterRecursiveCall(tstate, " in comparison")) {
        return NULL;
    }
    PyObject *res = do_richcompare(tstate, v, w, op);
    _Py_LeaveRecursiveCall(tstate);
    return res;
}
```

In `Object/object.c`

```c
/* Map rich comparison operators to their swapped version, e.g. LT <--> GT */
int _Py_SwappedOp[] = {Py_GT, Py_GE, Py_EQ, Py_NE, Py_LT, Py_LE};
```

In `Object/object.c`

```c
static const char * const opstrings[] = {"<", "<=", "==", "!=", ">", ">=", "~="};
```

In `Lib/opcode.py`

```python
cmp_op = ('<', '<=', '==', '!=', '>', '>=', '~=')
```

In `Python/compile.c`

```c
static int compiler_addcompare(struct compiler *c, cmpop_ty op)
{
    ...
}
```

```c
    case GtE:
        cmp = Py_GE;
        break;
    case AlE:
        cmp = Py_AlE;
        break;
```

In `Object/floatobject.c`

```c
static PyObject*
float_richcompare(PyObject *v, PyObject *w, int op)
{
    ...
}
```

```c
    case Py_AlE: {
        double diff = fabs(i - j);
        double rel_tol = 1e-9;
        double abs_tol = 0.1;
        r = (((diff <= fabs(rel_tol * j)) || 
              (diff <= fabs(rel_tol * i))) ||
             (diff <= abs_tol));
        }
        break;
```

In `Python/ceval.c`

```c
    case TARGET(COMPARE_OP): {
        assert(oparg <= Py_AlE);
        PyObject *right = POP();
        PyObject *left = TOP();
        PyObject *res = PyObject_RichCompare(left, right, oparg);
        SET_TOP(res);
        Py_DECREF(left);
        Py_DECREF(right);
        if (res == NULL)
            goto error;
        PREDICT(POP_JUMP_IF_FALSE);
        PREDICT(POP_JUMP_IF_TRUE);
        DISPATCH();
    }
```

```bash
make
```

### Current Behavior

```python
>>> 1 ~= 2
False
>>> 1 ~= 1.9
False
>>> 1.1 ~= 1.101
True
```

```python
>>> 1.1 ~= 1
False
```

### References

- [CPython Internals](https://www.google.com/search?q=cpython+internals&oq=cpython+internals&aqs=chrome.0.69i59j46i512j0i512l5j69i60.1628j0j4&sourceid=chrome&ie=UTF-8)
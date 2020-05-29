- Feature Name: Variadic arguments for ODE solvers
- Start Date: 2020-05-29
- RFC PR: 19
- Stan Issue:

# Summary
[summary]: #summary

This proposal is to make a set of ODE solvers that take a variadic list of
arguments.

# Motivation
[motivation]: #motivation

The goal is to avoid the argument packing and unpacking that the current solvers
depend on.

Argument packing is inconvenient, makes code hard to read, and introduces more
chances for making errors.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This design introduces six new functions:

`ode_bdf`, `ode_bdf_tol`,
`ode_adams`, `ode_adams_tol`,
`ode_rk45`, `ode_rk45_tol`

The solvers in the first columns have default tolerance settings (yet to be
decided). The solvers in the second column accept an argument for relative
tolerance, absolute tolerance, and maximum number of steps to take between
output times.

This is different from the current solvers where tolerances are presented
as overloads to the same function name. With the variadic argument this isn't
possible anymore so the function with tolerances is broken out separately.

The proposed `ode_bdf` interface is (the interfaces for `ode_adams` and
`ode_rk45` are the same):

```
vector[] ode_bdf(F f,
                 vector y0,
                 real t0, rea[] times,
                 T1 arg1, T2 arg2, ...)
```

The arguments are:
1. ```f``` - User-defined right hand side of the ODE (`dy/dt = f`)
2. ```y0``` - Initial state of the ode solve (`y0 = y(t0)`)
3. ```t0``` - Initial time of the ode solve
4. ```times``` - Sorted arary of times to which the ode will be solved (each
  element must be greater than t0)
5. ```arg1, arg2, ...``` - Arguments passed unmodified to the ODE right hand
  side (`f`). Can be any non-function type.

In this proposal, the user-defined ODE right hand side interface is:

```
vector f(real t, vector y, T1 arg1, T2 arg2, ...)
```

The arguments are:
1. ```t``` - Time at which to evaluate the ODE right hand side
2. ```y``` - State a which to evaluate the ODE right hand side
3. ```arg1, arg2, ...``` - Arguments pass unmodified from the `ode_bdf` call.
  The types `T1`, `T2`, etc. need to match between this function and the
  `ode_bdf` call.

The ODE right hand side returns `dy/dt` as a `vector`.

A call to `ode_bdf` returns the solution of the ODE specified by the right hand
side (`f`) and the initial conditions (`y0` and `t0`) at the time points given
by the `times` argument. The solution is given by an array of vectors.

The proposed `ode_bdf_tol` interface is (the interfaces for `ode_rk45_tol`
and `ode_adams_tol` are the same):
```
vector[] ode_bdf_tol(F f,
                     vector y0,
                     real t0, real[] times,
                     real rel_tol, real abs_tol,
		     int max_num_steps,
                     T1 arg1, T2 arg2, ...)
```

The arguments are:
1. ```f``` - User-defined right hand side of the ODE (`dy/dt = f`)
2. ```y0``` - Initial state of the ode solve (`y0 = y(t0)`)
3. ```t0``` - Initial time of the ode solve
4. ```times``` - Sorted arary of times to which the ode will be solved (each
  element must be greater than t0)
5. ```rel_tol``` - Relative tolerance for solve (data)
6. ```abs_tol``` - Absolute tolerance for solve (data)
7. ```max_num_steps``` - Maximum number of timesteps to take in integrating
  the ODE solution between output time points (data)
5. ```arg1, arg2, ...``` - Arguments passed unmodified to the ODE right hand
  side (`f`). Can be any non-function type.

There are a few changes from the previous implementation.

1. `theta`, `x_real`, and `x_int` are no longer broken out. All parameters,
real data, and integer data should be passed through as a trailing argument
(with no rules on naming).

2. The state is written in terms of a `vector` instead of real array. The
reasoning for switching is that an ODE state is a single thing (a single
vector), not really a list of separate things (where an array would be
preferable). Also ODE states are usually represented as vectors mathematically.

3. The times array only needs to be sorted, not strictly increasing. This means
that there can multiple output times in a row with the same value.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of these features is relatively straightforward given the
code from the
[reduce_sum](https://github.com/stan-dev/design-docs/blob/master/designs/0017-reduce_sum.md)
design doc and the existing ODE infrastructure. There is not any significant new
technology needed.

The old `integrate_ode_bdf`, `integrate_ode_adams`, and `integrate_ode_rk45`
functions should be deprecated with the introduction of these new functions.
They can remain in the code to provide compatibility to models written with the
old interfaces. These functions can all be written to wrap around the new
interfaces so there will not be much technical debt accrued maintaining them.

# Drawbacks
[drawbacks]: #drawbacks

One drawback is that the existing ODE documentation and case studies will need
to be updated to point at the new interfaces. The old interfaces can
remain for backwards compatibility though.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The most obvious alternatives are:

* Change the scope of the functions block to include variables defined in
the `data`, `transformed data`, `parameters`, and `transformed parameters`
block.

* Wait until closures are available so that the ODE right hand side can be
defined by a closure (which would then have the scope to use any variable
that could otherwise be passed to through as a right hand side argument here).

Neither of these are incompatible with the current design though, and so do not
make the current proposal less attractive.

# Prior art
[prior-art]: #prior-art

This builds directly on the existing ODE solver and `reduce_sum` interfaces.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

We should decide on default tolerances.

We should decide on if we want CVODES to take sensitivity variables into
account when computing local errors
(https://github.com/stan-dev/math/pull/1641#issuecomment-635562506)
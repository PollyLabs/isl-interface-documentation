Warning: This is a proposal, which does not document the current isl 

# How to use isl via the C++ and Python interface

## Constructing an integer set or map (isl::set / isl::map)

### Explicit Interface (today)

We first describe how the current C++ interface should be used to construct
isl sets and maps, just proposing a small number of extensions beyond what exists
today. The resulting code is still somehow verbose, but very explicit.

Example:

*{ [N, M] -> { [i,j] : 2 * M + 3 * N <= 2 * i + j + 10 }*

We create an integer set as follows. We first create a set of identifiers for
each of the needed dimensions (`isl::id Id_N(ctx, "N"`). We then introduce
initial expressions for all identifiers and required constants (`isl::pw_aff
N(Id_N), Ten(ctx, 10)`). From these initial expressions the actual affine
expressions are constructed (`isl::pw_aff LHS = Two.mul(M)`). Pairs of
expressions are combined with the operators lt_set (<), le_set (<=), ge_set
(>=), gt_set (>), eq_set (=), ne_set (!=) into a parameteric set of constraints
(`isl::set PSet = LHS.le_set(RHS)`). Finally, a non-parameteric set is
constructed from 1) a parameteric set specifying its constraints and 2) a list
of identifiers that specify the parameter dimensions that should be promoted to
set dimensions (`isl::set Set({Id_i, Id_j}, PSet)`).  Similary, a map can be
constructed by providing two lists of identifiers defining the input and output
dimensions (`isl::map Map({Id_i}, {Id_j}, PSet)`)




```
// Identifiers
isl::id Id_N(ctx, "N"), Id_M(ctx, "M"), Id_i(ctx, "i"), Id_j(ctx, "j");

// One (piece-wise) affine expression per identifier
// [N] -> { [(N)]}, [N] -> { [(M)]}, [i] -> { [(i)]}, [j] -> { [(j)]}
isl::pw_aff N(Id_N), M(Id_M), i(Id_i), j(Id_j);

// One (piece-wise) affine expression per constant
// {[(10)]}, {[(2)]}, {[(3)]}
isl::pw_aff Ten(ctx, 10), Two(ctx, 2), Three(ctx, 3);

// Build the left and right hand side of the expression
// [M, N] -> { [(2 * M + 3 * M)] }
isl::pw_aff LHS = Two.mul(M).add(Three.mul(N));

// [M, N] -> { [(2 * i + j + 10)] }
isl::pw_aff RHS = Two.mul(i).add(j).add(Ten);

// [N, M, i, j] -> { : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set PSet = LHS.le_set(RHS);

// [N, M] -> { [i, j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set Set({Id_i, Id_j}, PSet);

// [N, M] -> { [i] -> [j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::map Map({Id_i}, {Id_j}, PSet);
```

#### New functions

```
__isl_constructor
isl_pw_aff *isl_pw_aff_param_from_id(isl_id *identifier);
__isl_constructor
isl_pw_aff *isl_pw_aff_val_from_val(isl_val *value);
__isl_constructor
isl_pw_aff *isl_pw_aff_val_from_si(int value);
```
See: https://github.com/PollyLabs/isl/pull/25

```
__isl_constructor
isl_set *isl_set_from_id_list_params(isl_id_list *dims, isl_set *pset);
__isl_constructor
isl_map *isl_map_from_id_list_params(isl_id_list *input_dims, isl_id_list *output_dims, isl_set *pset);
```
See: https://github.com/PollyLabs/isl/pull/26

#### Notes

##### Why not use isl_aff

Currently we always need to use isl_pw_aff, as isl_aff does not allow for
parameter auto-alignment. This should be changed, but will require more work.
We should likely write the documentation in terms of isl_pw_aff for now.

##### How to introduce parameters

We can either use a constructor (in the proposal):

```
isl::set PSet("[N, M, i, j] -> { : 2 * M + 3 * N <= 2 * i + j + 10 }")

// { [N, M] -> { [i,j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set Set({i,j}, PSet);
// { [N, M] -> { [i]->[j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::map Map({i}, {j}, PSet);
```

or a set of member functions.

```
isl::set PSet("[N, M, i, j] -> { : 2 * M + 3 * N <= 2 * i + j + 10 }")

// { [N, M] -> { [i,j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set PSet.inputs(i,j);
// { [N, M] -> { [i]->[j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::map PSet.inputs(i).outputs(j);
```

### Explicit Interface for isl::aff (future)

After isl_aff supports parameter auto-alignment it will be possible to write the
above example (and other examples which do not require isl_pw_affs) as follows:

```
// Identifiers
isl::id Id_N(ctx, "N"), Id_M(ctx, "M"), Id_i(ctx, "i"), Id_j(ctx, "j");

// One (piece-wise) affine expression per identifier
// [N] -> { [(N)]}, [N] -> { [(M)]}, [i] -> { [(i)]}, [j] -> { [(j)]}
isl::aff N(Id_N), M(Id_M), i(Id_i), j(Id_j);

// One (piece-wise) affine expression per constant
// {[(10)]}, {[(2)]}, {[(3)]}
isl::aff Ten(ctx, 10), Two(ctx, 2), Three(ctx, 3);

// Build the left and right hand side of the expression
// [M, N] -> { [(2 * M + 3 * M)] }
isl::aff LHS = Two.mul(M).add(Three.mul(N));

// [M, N] -> { [(2 * i + j + 10)] }
isl::aff RHS = Two.mul(i).add(j).add(Ten);

// [N, M, i, j] -> { : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::constraint C = LHS.le_constraint(RHS);

// [N, M] -> { [i, j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set Set({Id_i, Id_j}, C);

// [N, M] -> { [i] -> [j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::map Map({Id_i}, {Id_j}, C);
```

#### TODOs:

- Introduce parameter auto-alignment to isl_aff
- Introduce isl_aff_[op]_constraint methods
- Enable conversion from a constraint to an isl_set

### Streamlined interface (future)

In certain use cases a more streamlined interface might be useful. Here is
an example which includes:

  - operator overloading
  - automatic conversion from isl::id to isl::aff
  - automatic conversion from int to isl::aff
  - a default context

```
isl::id N("N"), M("M"), i("i"), j("j');

isl::aff LHS = 2 * M + 3 * N;  // natural precedence works
isl::aff RHS = 2 * i + j + 10;

// { [N, M] -> { [i,j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set Set({i,j}, LHS <= RHS);

// { [N, M] -> { [i]->[j] : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::map Map({i}, {j}, LHS <= RHS);
```

#### Extensions

##### Use of a thread-local context

Instead of always providing a ctx object, the bindings could provide a thread
local ctx.


Explicit context:
```
isl::id N(ctx, "N");
```

Implicit context:
```
isl::id N("N");
```

##### Overloading of operators

Instead of calling the explicit interface, operator overloading can be used.

Without overloading:
```
isl::pw_aff = A.add(B).add(Three.mul(C));
```

With overloading
```
isl::pw_aff = A + B + 3 * C;
```

*Warning*: Overloading of the comparision operators may cause confusion as the
           result is not a boolean expression.

A solution might be to have these operators in a separate sub-namespace to
avoid surprising behavior of operator overloads. Another one might be to use
'markers'.

```

// [N, M, i, j] -> { : 2 * M + 3 * N <= 2 * i + j + 10 }
isl::set PSet = isl::set_maker(LHS) <= RHS;
isl::set Set({i}, PSet);
```

##### More efficient construction of parameter isl::aff's

When constructing an affine expression for a parameter, the explicit interface
requires two steps. First the construction of an isl::id and then its conversion
to a isl::aff. It would be nice if just one step would be needed. There
are two options:

1) Construction of isl::aff's from strings.

```
isl::aff A = ...

// [N] -> { [(N)] }
isl::aff N(ctx, "N");

isl::aff X = A.add(N);
```

2) Automatic conversion from isl::id to aff

```
isl::aff A = ...

// [N] -> { [(N)] }
isl::id N(ctx, "N");

isl::aff X = A.add(N);
```



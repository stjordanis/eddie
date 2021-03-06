# Return Concrete Contract

> It would be nice if only `mypackage.Error`or `mypackage.OtherError`
> were returned from an API, even though the functions are declared as
> returning `error`.

## Motivation

Due to the lack of return-type variance in Go's type system, it is
often necessary to declare a return type that is "wider" than the
actual types that will be returned.  A very common example, and the
one in which the rest of the document is framed, is that we want to
ensure that a concrete, user-facing error type, `mypackage.Error` is
actually returned from functions that declare `error` as a return type.

## Approach

The input to the contract is as follows:
* A target interface, e.g. `error`.
* A set of "allowable" concrete types which implement the target
  interface, e.g. `mypackage.Error`.

The contract will then perform an inductive, heuristic analysis, starting
from the seed functions. The goal of this analysis is to classify each
reachable function into those that always return one of the desired
types in place of the target interface and those that do not.  In cases
where a seed function cannot be proven to return only the desired
concrete types, a lint error will be generated.  The lint error will
include the seed function and at least one call-path chain to show the
source of the undesirable type.

## Implementation

The implementation is structured as an inductive call-graph analysis.
All functions start in an `unknown` state and are classified into
`clean` and `dirty`.

Functions are `clean` if all of the following properties apply to every
[`Value`](https://godoc.org/golang.org/x/tools/go/ssa#Value) that
appears in a
[`Return`](https://godoc.org/golang.org/x/tools/go/ssa#Return)
instruction:
* The value is not assignable to the target interface and can be ignored.
* The value is, trivially, one of the acceptable concrete types or a
  pointer thereto.
  * `return &mypackage.Error{}`
  * `err := &mypackage.Error{}; return err`
  * `if pg, ok := err.(*mypackage.Error); ok { return pg }`
* The value has been type-asserted in all dominating
  instruction or basic blocks.
  * `pgErr := err.(*mypackage.Error); return err`
  * `if _, ok := err.(*mypackage.Error); ok { return err }`
* The value is passed-through from another `clean` function.
  * `if _, err := knownCleanFunction(); err != nil { return err }`

In the case of cyclical call-graphs, we will assume that functions
reachable from themselves are initially `clean` and create an
invalidation set to record the members of the cycle.  If any member of
the set is marked as `dirty`, all functions in the set will be dirtied.

Whenever a [`Phi`](https://godoc.org/golang.org/x/tools/go/ssa#Phi)
value is encountered, it will be considered `clean` only if all of its
edges are `clean`.

## Known limitations

Functions are unnecessarily `dirty` if they:
* Return a value from an indirect invocation.
  * `if _, err := someCallback(); err != nil { return err }`

# Ecosystem Contracts

Networks.jl is the foundation every model package in the ecosystem depends on,
so it is where the *shared contracts* live: the small vocabularies that let
`ERGM`, `Siena`, `REM`, `Relevent`, `TERGM` and the rest mean the same thing by
the same keyword. A contract is defined once, here, and imported by name
wherever it is honoured — never redefined locally.

There are three, each documented below or linked from here:

| Contract | Keyword | Vocabulary |
|----------|---------|------------|
| [Missing data](#Missing-Data-Contract) | `missing=` | [`MISSING_POLICIES`](@ref) |
| [Tied events](#Tied-Event-Contract) | `ties=` | [`TIE_POLICIES`](@ref) |
| [Conversion](../guide/conversion_invariants.md) | `report=` | [`ConversionReport`](@ref) |

## Missing-Data Contract

A masked dyad is **unobserved**, not absent. The distinction is the whole point:
reading the stored face value of an unobserved tie silently invents data, and
turns a partially observed network into plausible, wrong output.

The contract has two halves. A routine *declares* whether it has a principled
missing-data treatment with [`supports_missing`](@ref), and *enforces* the
caller's choice with [`require_observed`](@ref).

The rule it enforces: **no exported routine may return a number computed from
the face value of an unobserved tie unless the caller asked for it in writing.**

```julia
function my_measure(net; missing::Symbol=:error)
    require_observed(net, missing; context="my_measure")
    # ... safe to read face values from here
end
```

Downstream packages expose this as the `missing=` keyword. `missing=:error` (the
default) refuses masked data; `missing=:face` is the explicit, auditable opt-in
to face values **where a routine offers it** — and not every routine does. An
estimator for which reading face values would enumerate unobserved dyads as
observed rows (TERGM, ERGMCount, ERGMMulti, ...) exposes no `missing=` keyword
at all, and ERGM.jl's MCMC entry points use their own vocabulary
(`:condition_on_face`).

So each routine *declares* the policies its keyword actually takes with
[`missing_policies`](@ref). The default `(:error,)` is the honest answer for a
routine without the keyword. The ecosystem's capability-matrix generator prints
`missing_policies(f)` for every routine it lists, instead of assuming `:face`
everywhere, and the shared error message raised by `require_observed` mentions
`missing=:face` only when the caller passed `face_ok=true`. A routine with its
own opt-in vocabulary appends its bullets to that one message through
`require_observed(...; hint=)` (ERGM's `missing=:condition_on_face` /
`missing=:mle`) rather than rethrowing it with text glued on.

For a *shared generic* — `gof`, which has one method per package — the
one-argument form cannot speak for any package's method, so packages declare
`Networks.missing_policies(::typeof(gof), ::Type{<:TheirResult})` and tooling
asks the two-argument form `missing_policies(gof, ResultType)`; the fallback
`missing_policies(f, ::Type) = missing_policies(f)` makes that query uniform.

```@docs
supports_missing
missing_policies
require_observed
MISSING_POLICIES
```

See also the [Missing Data guide](../guide/missing_data.md) and the
missing-dyad mask API on the [Types page](types.md).

## Tied-Event Contract

For the models with an event clock (`REM.fit_rem`, `Relevent.fit_obpm`,
`Relevent.fit_timing`), two events sharing a timestamp is not a nuisance to be
sorted away — an exact-time or exact-order likelihood evaluated on tied data is
evaluated on information the data does not contain.

So the vocabulary starts from refusal (`ties=:error` is the default) and every
alternative is an explicit, recorded claim about what a simultaneous pair
*means*. A policy a model cannot honour is refused loudly by
[`check_tie_policy`](@ref) rather than silently no-oping.

```@docs
TIE_POLICIES
check_tie_policy
```

[`tie_method`](@ref) reports the policy that actually **ran** — which is why
`:error` can never appear there: a fit that would have had to break a tie under
`ties=:error` threw instead of returning.

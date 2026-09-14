# Changelog

All notable changes to Networks.jl are documented in this file. The format is
based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the
package adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - Unreleased

Release driven by the 2026-07 expert-panel review: directedness becomes a
compile-time property, networks gain attribute-preserving `copy`, missing-dyad
masks, bundled teaching datasets, and a shared result-presentation layer used
by every model package in the ecosystem.

### Breaking

- **Shared Newton uncertainty rejects numerical non-identification.** A
  singular timing baseline/constant-covariate design could pass Cholesky
  through roundoff and report finite standard errors. Information now passes
  a unit-diagonal eigenvalue rank check before inversion; unavailable
  covariance produces NaNs, a warning and `converged=false`. Individual
  coefficient rescaling does not change the rank criterion.
- **`add_edges!` never drops an edge** (panel 2026-09 round 3). The round-2
  rule "every listed edge ends up in the network or the call throws" applied
  to `network_from_edgelist` only; `add_edges!(net, [(1, 2), (1, 99)])` on a
  4-vertex network returned 1 and added one edge, because `add_edge!`'s
  `false` was counted, not raised. `add_edges!` now throws an `ArgumentError`
  naming the edge for an endpoint outside `1:nv(net)` ("pass a larger n= to
  network() / add_vertices! first"), a self-loop under `loops=false`, and a
  within-mode pair on a two-mode network — R's `add.edges` "Illegal vertex
  reference". A repeated edge still counts 0 and is not an error; the single
  `add_edge!` keeps Graphs.jl's `Bool` convention. *Migration:* size the
  network before adding, or pass `loops=true`.
- **The `Dict` form of `set_edge_attribute!` validates like the single-edge
  form** (panel 2026-09 round 3). It stored anything: a key `(3, 9)` on a
  4-vertex network, a key on a dyad that carries no tie. It now checks both
  endpoints of every key against `1:nv(net)` and, by default, `has_edge` —
  all-or-nothing, nothing written when any key is bad — and takes the same
  `require_edge::Bool=true` keyword as the single-edge form (the opt-out for
  storing values ahead of the edges). Same verb, one contract. *Migration:*
  set edge attributes after `add_edges!`, or pass `require_edge=false`.
- **`as_matrix(net; attr=)` and `as_edgelist(net; attrs=)` refuse an edge
  attribute the network does not have** (panel 2026-09 round 3).
  `as_matrix(net; attr=:wt)` on a network whose only edge attribute is
  `:weight` returned an all-zero matrix — not even the binary adjacency — and
  `as_edgelist(net; attrs=[:wt])` a column of `missing`, so a mistyped weight
  name fed zeros into QAP or an eigenvector centrality downstream. Both now
  throw an `ArgumentError` listing the attributes that exist (R's
  `as.sociomatrix` warns "There is no edge attribute named wt"). An edge that
  exists but carries no value for the attribute still contributes `0` /
  `missing`. *Migration:* none for correct code.
- **Directed two-mode incidence matrices drop mode-2 → mode-1 arcs (and
  masks) with a warning, as R does, instead of reversing them** (panel
  2026-09 round 3). `as_matrix` on a *directed* `network(n; bipartite=k)`
  normalised every arc with `i > k && ((i, j) = (j, i))`, so the arc `5 → 1`
  was written at `B[1, col(5)]` and reported as `1 → 5`, and a masked dyad
  `6 → 2` as a mask on `2 → 6`. An `n₁ × n₂` incidence matrix has no cell for
  a reverse arc; R's `as.matrix.network` drops it silently, and Networks.jl
  now drops it too, with a `@warn` naming how many arcs / masked dyads went
  and pointing at `expand_bipartite=true`, whose square matrix keeps every
  arc (and already matched R). Undirected two-mode networks are unaffected.
  Pinned against R 4.6.1 / network 1.20.0 by the "Directed two-mode incidence
  matrix" testset. *Migration:* pass `expand_bipartite=true` when a directed
  two-mode network carries arcs in both directions.
- **Three refusals R makes** (panel 2026-09 round 3): `network(-1)` (and
  `Network(-1)`, `Network{Int}(; n=-1)`) throws "n must be non-negative"
  instead of building an empty network (R: "Network objects cannot be of
  negative order"); `get_induced_subgraph(net, [1, 99])` throws naming the
  vertex, and a repeated vertex is refused, instead of returning a 2-vertex
  network (R: "Illegal vertex selection"); `permute_vertices(net, [1, 1, 2, …])`
  throws "perm must be a permutation of 1:n" instead of a raw `BoundsError`
  (R: "Invalid permutation vector").
- **`show(io, net)` is one line; the R-style block moved to
  `show(io, MIME"text/plain"(), net)`** (panel 2026-09 round 3). The only
  `show` method printed four-plus lines, so `load_dataset(:s50)` — a
  `NamedTuple` holding a `Vector{Network}` — and every `Vector`/`Dict` of
  networks displayed as a garbled wall. `repr(net)`/`print(net)`/`string(net)`
  now give `Network{Int64,true}(50 vertices, 113 edges)` (with `, loops` and
  `, two-mode n₁+n₂` when they apply); the REPL/`display` block is the old
  one plus what R's `print.network` always states and students most often
  get wrong — `Directed:`, `Loops:` and the two-mode size (`Two-mode: 2
  mode-1 vertices (1:2), 4 mode-2 vertices (3:6)`) — with attribute names
  sorted as R lists them. `BipartiteNetwork` has the same pair, delegating to
  the wrapped network. *Migration:* code that parsed `string(net)` for
  `"Vertices:"` must use `repr(MIME"text/plain"(), net)`.
- **Attribute setters validate their targets** (panel 2026-09 round 2, "out-of-range
  vertex IDs and over-range edge lists are accepted silently"). The
  single-vertex `set_vertex_attribute!(net, attr, v, value)` and the `Dict`
  form throw an `ArgumentError` naming the attribute, the vertex and `nv(net)`
  when a key is not a vertex (`1 <= v <= nv(net)`; the `Dict` form checks every
  key before writing any), and the single-edge
  `set_edge_attribute!(net, attr, i, j, value)` throws when either endpoint is
  not a vertex or — by default — when `(i, j)` is not an edge. This is R's
  behaviour (`set.vertex.attribute`: "vertex IDs out of range";
  `set.edge.attribute` can only address an edge that exists); previously a
  typo'd ID stored a value for a vertex that did not exist and read back as
  plausible, wrong data. *Migration:* store edge attributes after
  `add_edge!`, or pass the new keyword `require_edge=false` to store a value
  ahead of the edge deliberately (it applies as soon as the edge is added).
  Pinned by the "Common mistakes are actionable" testset.
- **`network_from_edgelist` never drops an edge.** An endpoint outside `1:n`
  (an undersized `n=`, a mistyped ID) or a self-loop under `loops=false`
  throws an `ArgumentError` naming the edge, as R's
  `network(el, matrix.type="edgelist")` errors on an illegal edgelist
  reference. Previously `network_from_edgelist([(1, 2), (2, 5)]; n=3)`
  returned a 3-vertex network with ONE edge because Graphs.jl's `add_edge!`
  returned `false` and nobody looked. *Migration:* pass a large enough `n=`
  (or none, to auto-detect), `loops=true` for self-loops.
- **`vertex_attribute_vector` throws `ArgumentError`, not `KeyError`**, when a
  vertex has no value and no `default=` is given. The message names the
  attribute and the vertex, lists the attributes the network does have when
  the name itself is unknown (`NodeCov(:welth)` now says
  `no attribute :welth; existing: [:name, :priorates, :totalties, :wealth]`),
  and points at `default=`. The old `KeyError(i)` carried the vertex index and
  nothing else. *Migration:* `catch e::KeyError` around this call becomes
  `ArgumentError`.

- **`Network{T}` is now `Network{T,D}`** — directedness is a compile-time type
  parameter (`D::Bool`) instead of a mutable runtime field, so
  `Graphs.is_directed` is truthful and Graphs.jl trait-dispatched generics
  (e.g. `Graphs.density`) pick the correct (un)directed algorithm.
  One-parameter annotations such as `Network{Int}` still work (a `UnionAll`
  matching both directed and undirected), and `Network{Int}(; n=5,
  directed=false)` still constructs. *Migration:* keep `Network{Int}` in
  signatures (or use `Network{Int,D} where D`); code that mutated
  `net.directed` must instead construct a new network with the desired
  directedness (`net.directed` is now a read-only property).
- **`Graphs.is_directed` is truthful.** Previously it returned `true` for
  every `Network` (including undirected ones); it now returns the actual
  directedness at both instance and type level. *Migration:* code that relied
  on `is_directed(typeof(net)) == true` for undirected networks must handle
  `false`.
- **`Graphs.is_bipartite` is now graph-theoretic** (actual two-colorability of
  the graph). The old metadata-flag semantics ("was this constructed as
  two-mode?") moved to the new exported `is_two_mode(net)`. *Migration:*
  replace `is_bipartite(net)` with `is_two_mode(net)` wherever you meant the
  two-mode construction flag.
- **DataFrames methods moved behind a package extension.** `as_dataframe` and
  `network_from_dataframe` now live in `NetworkDataFramesExt`; DataFrames is a
  weak dependency. *Migration:* add `using DataFrames` before calling either
  function (without it they throw a `MethodError`).
- **Minimum Julia version raised to 1.12** (was documented as 1.9+).
  *Migration:* upgrade Julia; no code changes needed.
- **Package UUID regenerated** (the 0.1.0 placeholder UUID was replaced by a
  real one). *Migration:* re-resolve environments that pinned the old UUID
  (delete the stale `Manifest.toml` entry and `Pkg.resolve()`).
- **`network_density` refuses a network with masked (unobserved) dyads by
  default** (panel 2026-09, item 4). It was the one unguarded root of the
  missing-data contract: `SNA.gden` wrapped it in `require_observed`, but
  `TSNA.t_density`/`t_sna_stats` and `ERGM._random_network` called it bare
  and silently read face values. It now calls
  `require_observed(net, missing; context="network_density")` itself.
  *Migration:* pass `missing=:face` to count edges at their stored face value
  over the full dyad count (R's `network.density(x, na.omit=FALSE)`), or
  `clear_missing_dyads!` first. R's default `na.omit=TRUE` (drop NA edges from
  the numerator, keep the denominator) is deliberately not offered — it reads
  an unobserved tie as absent.
- **`network_density(::BipartiteNetwork)` now uses the full-dyad denominator,
  like the flagged `Network`** (panel 2026-09, item 9). The wrapper method
  hard-coded `ne / (n₁ · n₂)` and ignored directedness, so `BipartiteNetwork(2,
  3)` and `network(5; bipartite=2)` disagreed on the same edge set. Both now
  share one implementation whose default is R's `discount.bipartite=FALSE`
  (all `n (n − 1)` or `n (n − 1) / 2` dyads, within-mode ones included).
  *Migration:* pass `discount_bipartite=true` for the cross-mode denominator
  `n₁ · n₂` (undirected) — note it is `2 · n₁ · n₂` for a *directed* two-mode
  network, which the old wrapper got wrong by a factor of two.
- **`load_golden` throws when `provenance.script` does not exist** (panel
  2026-09, item 29). The script path is resolved by walking up the ancestors
  of the fixture's directory (the fixture's own directory first, so an ad-hoc
  `r/<name>.R` beside it works; the package root a few levels up, so the
  conventional `test/fixtures/r/<name>.R` resolves from any working
  directory) until `joinpath(dir, script)` is a file; `root=` starts the walk
  elsewhere. The error names the fixture, the script string and every
  directory searched. All eleven fixtures in the ecosystem resolve; an ad-hoc
  fixture must now ship its script beside it. `GoldenFixture` gains a
  `script_path::String` field (the resolved absolute path), printed by `show`
  and `golden_report`, so a red test names the script to re-run — the struct
  has no external constructor, so no caller constructs it by hand.
- **`rem_edge!` on an absent edge is a no-op** (returns `false` without
  touching the attribute store). Previously it also deleted any edge attribute
  that had been set on the non-existent edge; such stale attributes now stay
  until the edge is added and removed, or the attribute is deleted explicitly.
- **`newton_fit` / `logistic_derivatives` are the ONE optimizer and logistic
  kernel of the ecosystem** (panel 2026-09, item 14): Networks.jl hosts them
  as `public` (see Added) and ERGM.jl imports and re-exports them by name
  (`ERGM.newton_fit === Networks.newton_fit`; ERGM's local copy and its Optim
  dependency are gone). Code that does `using ERGM, Networks` and calls
  `newton_fit` unqualified keeps working — the names are `public`, not
  exported, here, so there is exactly one exported binding. Relative to
  ERGM's former copy: a step at which the objective is not finite is never
  taken (halved like a decrease; `converged=false` at the last finite iterate
  if every halving is non-finite), and an indefinite Hessian returns `NaN`
  standard errors with a warning instead of `sqrt.(abs.(diag(pinv(-hess))))`.
  ERGM's `mcmle` is now the ecosystem's second estimator with a principled
  missing-data treatment after the available-case MPLE
  (`supports_missing(ERGM.mcmle) == true` via `missing=:mle`).
- **`newton_fit`'s convergence verdict is scale-free** (panel 2026-09 round
  2/3, the TERGM, ERGMRank and REM findings). Convergence used to require
  `norm(grad) < sqrt(tol)` — a criterion in the gradient's own units, which
  scale with rows × statistic magnitude — and a step was accepted only when
  the objective did not decrease at all. On a CMPLE design with a `Triangle`
  column running to ~30 over 3 000 rows, on ERGMRank's 2 040-row swap design
  (Hessian entries in the thousands) and on REM's Breslow fixture, the
  iterate one Newton step short of the maximum already had an objective flat
  to rounding, so every halving "decreased" it by an ulp while the gradient
  there (1e-4 … 1e-3) was above `sqrt(tol)`: a converged fit was reported
  `converged=false` (R's glm converged at the same point), and BLAS rounding
  then decided which bootstrap replicates counted as converged. Two
  changes, both pinned by the "scale-free verdict" testset: (1) the **Newton
  decrement** `½ ∇ℓᵀ(−H)⁻¹∇ℓ` — the gain the full step could still make,
  in the units of `tol` — is checked before every step; when it is below
  `tol` the **full step is taken** (the polish R's glm performs; it must be
  finite and lose no more than `max(tol, 8·eps(|ℓ|))`, which the quadratic
  model bounds) and the fit is converged *at the polished point* — never
  after a halved step, which would leave the gradient half-way; a stall
  with a large decrement still stays put, unconverged, and the classic test
  `ll_change < tol && ‖∇ℓ‖ < sqrt(tol)` is unchanged; (2) a step whose
  objective decreases by at most `8·eps(|ℓ|)` **while the gradient norm
  shrinks** is accepted as a non-decrease (rounding noise is not descent).
  TERGM's `_polish_newton` and ERGMRank's `_stalled_at_optimum` shims are
  deleted with this; REM's compensated log-likelihood summation stays (it is
  an accuracy improvement in its own right). *Behaviour change:* a fit that
  used to stop one ulp short now takes that last step, so coefficients can
  move at the 1e-9 level and `iterations` can differ by one.
- **`require_observed` takes `hint=`** (panel 2026-09, item 5 follow-up):
  a routine with its own opt-in vocabulary appends its bullets (ERGM's
  `missing=:condition_on_face` / `missing=:mle`) to the ONE shared refusal
  instead of catching it and rethrowing `e.msg * text`. Default `""`, message
  unchanged.
- **`get_vertex_attribute(net, attr)` / `get_edge_attribute(net, attr)`
  allocate nothing on a hit** (ERGMUserterms round 3): the eager
  `get(d, k, Dict{…}())` default built and discarded an empty `Dict` (80 B)
  on every call, even when the attribute existed — paid per sampler step by
  any term reading the whole-Dict getter. Now the lazy `get(() -> …, d, k)`.
  A miss still returns an empty, correctly typed `Dict`.

### Added

- **`src` and `dst` are re-exported from Graphs.jl** beside `edges` (panel
  2026-09, item 3 follow-up from ERGMUserterms): `edges(net)` yields Graphs.jl
  edges whose endpoints are read with `src(e)`/`dst(e)`, and re-exporting the
  iterator without its accessors left every term author who wrote
  `for e in edges(net); src(e) …` with an `UndefVarError` inside `compute`
  after `using ERGM`. Same bindings as `Graphs.src`/`Graphs.dst`, so
  `using Networks, Graphs` stays unambiguous; added to the frozen inventory
  (ERGM.jl's curated re-export list follows).
- **`missing_policies(f, ::Type)`**, the per-result-type form of the
  routine-level vocabulary (panel 2026-09, item 5): a shared generic such as
  `gof` has one method per package, so a one-argument `missing_policies(gof)`
  cannot speak for `gof(::ERGMResult)`; a package declares
  `Networks.missing_policies(::typeof(gof), ::Type{<:ERGMResult})` and
  tooling queries the two-argument form for shared verbs. The fallback
  `missing_policies(f, ::Type) = missing_policies(f)` makes the query
  uniform.
- **`print_coeftable` renders a table without a standard-error column and
  takes a p-value display floor** (panel 2026-09, item 15). `std_errors` may
  be `nothing`: the `Std.Error` column is omitted, `z_values=` becomes
  mandatory (an `ArgumentError` names it otherwise) and `header` is a
  3-tuple (default `("Estimate", "z value", "Pr(>|z|)")`); a header of the
  wrong length for the table shape is an `ArgumentError`. New keyword
  `p_floor::Real=1e-16` is forwarded to `format_pvalue(p; floor=p_floor)`, so
  a permutation or Monte-Carlo test passes `p_floor=1/reps` and a zero count
  prints `<0.001`, never `0.0` (the one place the ecosystem's "never prints
  `0.0`" convention was broken, SNA's QAP printer, can now route through the
  shared layer). Every existing keyword default is unchanged, so the eleven
  callers across the model packages print byte-identically.
- **`CoefficientTable` — the generic coefficient table, and the `coeftable`
  StatsAPI verb** (both exported; panel 2026-09, item 15). **New dependency:
  `StatsAPI`** (compat `"1.7"`; dependency-free). `CoefficientTable(names,
  estimates, std_errors; z_values, p_values, header, p_floor)` holds exactly
  the vectors `print_coeftable` consumes, fills `z`/`p` from `z_pvalues` when
  omitted (with the same length checks), accepts `std_errors=nothing` for a
  three-column table, and its `show` delegates to `print_coeftable`, so what
  a user inspects is what the fit prints. `length(tbl)`, `tbl[i]` and
  `tbl["name"]`/`tbl[:name]` (a row `NamedTuple` `(name, estimate, std_error,
  z_value, p_value)`), and iteration over rows. The name is deliberately not
  `CoefTable`, so `using Networks, StatsBase` is never ambiguous. Networks.jl
  `import StatsAPI: coeftable` and re-exports it **without defining a
  method**: `Networks.coeftable === StatsAPI.coeftable`, each model package
  defines `StatsAPI.coeftable(fit::ItsOwnType) = CoefficientTable(...)`, and
  `using Networks, REM` (REM exports the same binding) stays unambiguous.
- **`check_statsapi(fit; required, strict) -> NamedTuple`** (exported;
  `src/statsapi.jl`; panel 2026-09, item 15): the ONE checker of the
  ecosystem's StatsAPI surface — `coef`, `stderror`, `vcov`, `confint`,
  `loglikelihood`, `nobs`, `dof`, `aic`, `bic`, `coeftable`. Reports per verb
  whether `hasmethod(StatsAPI.<verb>, Tuple{typeof(fit)})` holds; when the
  methods exist it also checks `length(coef) == length(stderror) ==
  size(vcov, 1) == size(vcov, 2)`, that `coeftable(fit) isa CoefficientTable`
  with one row per coefficient, and that none of them throws. `strict=true`
  throws an `ArgumentError` listing every missing or inconsistent verb, so a
  package's testset reads `@test check_statsapi(fit; strict=true) !==
  nothing` and the failure names the gap (the migration guide had promised
  six verbs "on every fitted model" while `hasmethod` found them on a third
  of the types). `required=` takes any subset for a model that legitimately
  lacks a likelihood.
- **`z_pvalues` — the ONE z → p helper** (exported; panel 2026-09, item 13),
  replacing the five copies in ERGM (`_z_pvalues`, floored), ERGMEgo/
  ERGMRank/ERGMCount (unfloored, underflowing to a false `0.0`) and Siena/
  Relevent (NaN-aware). `z_pvalues(z)` computes `erfc(|z|/√2)` =
  `2·P(Z ≥ |z|)` — accurate beyond `|z| ≈ 8.3`, where `2(1 − Φ(|z|))` cancels
  to nonsense — floored at `floatmin(Float64)` for finite `z` (a finite
  statistic never has p exactly 0) with `NaN` staying `NaN`;
  `z_pvalues(estimates, std_errors)` returns `(z=, p=)` with `z = NaN`
  wherever the standard error is not strictly positive (the form Siena's and
  Relevent's coefficient tables need). Pinned against `Distributions.ccdf` at
  1e-12 over `[-37, 37]`. **New dependency: `SpecialFunctions`** (compat
  `"2"`) for `erfc` — a hand-rolled `erfc` is a correctness liability and
  Distributions is too heavy for the foundation; `Distributions` is a test-only
  extra.
- **`newton_fit` and `logistic_derivatives` hosted in `src/newton.jl`**
  (`public`, not exported; panel 2026-09, item 14) as the ONE Newton–Raphson
  optimizer and the ONE logistic-likelihood kernel of the ecosystem, stdlib
  LinearAlgebra only. `newton_fit(f, θ0; maxiter, tol, max_halvings)` keeps
  ERGM's return `(θ, se, vcov, loglik, converged, iterations)` so ERGM.jl can
  alias it, plus the panel's two fixes: a **finiteness guard** (a non-finite
  objective is halved like a decrease; if every halving is non-finite the loop
  stops at the last finite iterate with `converged=false`; a non-finite
  objective at `θ0` is an `ArgumentError`) and a **positive-definiteness
  check** for the covariance (`cholesky(Symmetric(-hess))`; on failure
  `@warn "not negative definite"` and `se`/`vcov` filled with `NaN` — the old
  `sqrt.(abs.(diag(pinv(-hess))))` returned finite, meaningless numbers at a
  saddle). When every halving finds only a finite *decreasing* objective, `θ`
  stays put and `converged` reports whether the gradient is already below
  `sqrt(tol)`. `logistic_derivatives(X, y::AbstractVector{Bool}; offset,
  weights)` gains `weights=` and a **binomial-row form**
  `logistic_derivatives(X, n_tot, n_one; offset)` (ℓ = Σ n_one·η −
  n_tot·log(1+eʰ), ∇ = X'(n_one − n_tot·p), H = −X'diag(n_tot·p(1−p))X) —
  the form ERGM's MPLE needs after collapsing identical dyad classes, so
  ERGM's phase can route `_mple_fit` through it and drop Optim. ONE kernel
  (binary = binomial with `n_tot = weights`, `n_one = n_tot·y`), workspaces
  allocated once in the closure; an evaluation allocates exactly the returned
  gradient and Hessian, pinned by `@allocated` equality with those two arrays.
  Tests: Poisson and quadratic closed forms, weighted ≡ duplicated rows ≡
  binomial to 1e-12, analytic vs central-difference gradient and Hessian, the
  Florentine edges-only MPLE `θ = log(20/100)`, `se = 1/sqrt(120·(1/6)·(5/6))`
  to 1e-10, indefinite/singular Hessian → NaN with the warning, `−Inf` on the
  full step still converging, `maxiter=1` unconverged.
- **`check_se(se, allowed; context) -> Symbol`** (exported; panel 2026-09,
  item 28): the ONE validator of the `se=` keyword, returning `se` when it is
  in `allowed` and otherwise throwing
  `ArgumentError("<context>: se must be one of (:hessian, :sandwich) (got :x)")`.
  Replaces the seven `se in (...) || throw(...)` lines in ERGM, ERGMCount,
  ERGMMulti, ERGMRank, TERGM and REM (both of REM's vocabularies are covered
  by the tests).
- The "Exported API inventory" testset now freezes the **public** list
  (`[:logistic_derivatives, :newton_fit]`) beside the exported one, since
  `names(Networks)` lists both and ERGM/SNA's re-export loops iterate it.
- **Three more bundled datasets, with provenance** (panel 2026-09, item 22):
  `load_dataset(:faux_mesa_high)` — statnet's simulated 205-student school
  (203 undirected ties) as a `Network{Int,false}` with vertex attributes in
  statnet's spelling `:Grade` (`Int`, 7–12), `:Race`, `:Sex` (`String`), so
  `nodematch("Grade")` translates without renaming; `load_dataset(:s50)` —
  the RSiena `s50` panel as a NamedTuple `(friendship::Vector{Network{Int,true}}`
  (three waves, 113/116/122 ties), `alcohol::Matrix{Int}` (50×3, coded 1–5),
  `smoke::Matrix{Int}` (50×3, coded 1–3)`)`, feeding
  `Siena.DependentNetwork(:friendship, s50.friendship)` directly;
  `load_dataset(:wtc_police_calls)` — relevent's World Trade Center police
  radio calls as a NamedTuple `(events::Matrix{Int}` (481×3: number, sender,
  receiver — ordinal), `n_actors = 37`, `is_icr::Vector{Bool})`, the input of
  `REM.EventSequence`/`Relevent.fit_obpm`. `load_dataset` therefore returns
  a `Network` **or** a documented NamedTuple; the unknown-name error lists all
  six names. The files are produced by the checked-in
  `data/r/extract_datasets.R` (base R ≥ 4.5 + `network` only: downloads
  `faux.mesa.high.RData` from statnet/ergm and `relevent_workshop.Rdata` from
  statnet/Workshops — the CRAN `relevent` 1.2-1 tarball ships no data
  directory — and reads s50 from RSiena when installed, else from the CSV
  copies Siena.jl carried under `test/data/`), and `data/README.md` records
  per dataset the source URL, SHA-256 of every upstream and written file,
  upstream version (ergm 4.x data file; relevent workshop; RSiena 1.6.6 — the
  matrices Siena.jl's provenanced `s50_siena07` fixture came from), licence
  (GPL-3 / GPL-2+), citation and variable coding. Tests cross-check
  faux.mesa.high's 163 same-grade ties against ERGMEgo.jl's provenanced
  `fauxmesa_ego_census.toml` targets `[203, 163]`, the s50 tie counts against
  the row sums of the files, and the three ICR actors (13, 14, 32) frozen
  from `WTCPoliceIsICR`. Siena.jl's `test/data/*.csv` and the site's
  dynamic-model tutorials move onto these loaders in their own phases.

- **`degree`, `indegree`, `outdegree` re-exported from Graphs.jl** (panel
  2026-09, item 3), beside `nv`/`ne`/`neighbors`, so a plain `using Networks`
  — and `using ERGM` / `using SNA`, which mirror this list — answers the first
  descriptive question a user asks. Same bindings as Graphs.jl, so
  `using Networks, Graphs` is not ambiguous. The export list is now **frozen**
  by the "Exported API inventory" testset: adding or removing an export is a
  reviewed change to that literal, and ERGM/SNA's curated re-export lists are
  diffed against it. A `# public API that is deliberately not exported` block
  in `src/Networks.jl` holds the shared optimizer (`newton_fit`,
  `logistic_derivatives`) as `public` names.
- **`network_density` keywords `missing=`, `diag=`, `discount_bipartite=`**
  (panel 2026-09, items 4, 9, 30), one implementation shared by `Network` and
  `BipartiteNetwork` and pinned to R `network::network.density` by the new
  golden fixture `test/fixtures/network_density.toml` (script
  `test/fixtures/r/network_density.R`, `network` 1.20.0; deterministic ratio,
  tolerance 1e-12) across directed/undirected × loops × two-mode ×
  `discount.bipartite` × one NA edge at face value. `diag` defaults to the
  network's `loops` flag (R keys on `has.loops` too), so every existing result
  is byte-identical; `diag=false` on a loops network mirrors
  `sna::gden(diag=FALSE)` and drops self-loops from numerator and denominator.
- **`missing_policies(f) -> Tuple{Vararg{Symbol}}`** (exported; panel 2026-09,
  item 5): the routine-level vocabulary of the missing-data contract — the
  policies a routine's `missing=` keyword *actually* accepts. Default
  `(:error,)` (no keyword, no escape hatch); `network_density` declares
  `(:error, :face)`; a package with its own vocabulary (ERGM's
  `:condition_on_face`) declares it here. The capability-matrix generator
  prints this instead of a `missing = :face` literal.
- **`as_matrix(net; expand_bipartite=true)`** (also `as_adjacency_matrix`; R's
  `expand.bipartite=`; panel 2026-09, item 9): on a two-mode network return the
  `(n₁ + n₂)`-square adjacency matrix instead of the `n₁ × n₂` incidence
  matrix, through the same dense/sparse code path as a one-mode network so
  `attr=`, `sparse=`, `missing_value=` and `return_mask=` all apply. This is
  the shape every one-mode matrix routine (SNA's eigenvector centrality,
  equivalence, QAP) expects, and it round-trips through `network_from_matrix`.
  `as_adjacency_matrix` now also accepts a `BipartiteNetwork`.
- **The ecosystem tied-event contract: `TIE_POLICIES` and `check_tie_policy`**
  (`src/results.jl`, exported; review finding 12). The third sibling of the
  missing-data and conversion contracts, for the models with an event clock.
  `TIE_POLICIES == (:error, :ordered, :breslow, :efron, :batch)` is the
  vocabulary of the shared `ties=` keyword — one set of symbols, one meaning per
  symbol, so `ties=:efron` cannot mean two things in two packages — and
  `check_tie_policy(ties, supported; model, reasons)` is the guard a fitter
  calls: it returns `ties` when the model supports it and otherwise throws,
  naming the model, **why** the policy does not apply to it, and what is
  supported instead. The rule it enforces: an exact-time or exact-order
  likelihood defaults to `:error` on tied data, and a policy a model cannot
  honour fails loudly rather than no-ops (`:breslow`/`:efron` correct a *partial*
  likelihood and are undefined for an exact-time hazard model; `:batch` is
  meaningless in an ordinal one, where a frozen batch simply *is* Breslow).
  `tie_method(fit)` accordingly reports the policy that actually **ran** — hence
  `:none` when the data had no ties, and never `:error`, which throws rather than
  returning a fit. Honoured by `REM.fit_rem` and by `Relevent.fit_obpm` /
  `Relevent.fit_timing`.

- **`bootstrap_cov(refit, simulate, θ̂; n_boot, rng, threaded)` — the ONE
  resampling loop of the ecosystem** (`src/bootstrap.jl`, exported). Draw
  `n_boot` replicate datasets, refit the estimator on each (threaded), return
  the empirical covariance, standard errors and the `n_boot × p` matrix of
  refits. Every `se=:bootstrap` in the ecosystem runs on it: the parametric
  bootstraps of `ERGM.mple`, `ERGMCount.count_mple`, `ERGMRank.fit_ergm_rank`
  and `ERGMMulti.ergm_multi`, and `REM.fit_rem`'s repeated draw of the
  case-control risk set (where the callbacks resample the *sampling design*, not
  the model). It lives here rather than in ERGM.jl because REM.jl deliberately
  does not depend on ERGM.jl, and Networks.jl is the one foundation all five
  packages share — the same reason the sibling shared optimizer `newton_fit`
  followed it here (see the 2026-09 entries above).
  Replicates are opaque to the loop, and all randomness flows through the
  caller's `rng`, so a fixed `rng` reproduces the standard errors exactly and the
  threaded refits are thread-count-independent (issue #9).
- `Random` and `LinearAlgebra` (both stdlib) are now dependencies, for the above.

- **The ecosystem conversion contract** (`src/conversion.jl`): the exported
  `ConversionReport` type plus `record_drop!`, `is_lossless` and
  `dropped_fields`. Networks.jl owns the vocabulary; the adapters
  (NetworkDynamic.jl, TSNA.jl, REM.jl's and Siena.jl's extensions) build the
  reports. Every lossy conversion in the ecosystem now accepts `report=true`
  and returns `(result, ::ConversionReport)` naming each field it could not
  carry across and why. The per-path invariant table — what each of the five
  conversion paths preserves, transforms, drops-with-policy or rejects, over
  directedness, loops, attributes, the missing-dyad mask, two-mode metadata,
  the observation window, spells and actor presence — is published as
  `docs/src/guide/conversion_invariants.md`.

  The rule it enforces, mirroring the missing-data contract: preserve what the
  target can represent, reject or policy-gate what it cannot, report what was
  dropped. Silently returning a plausible-but-wrong object is the failure mode
  the whole contract exists to prevent.

- **The shared statistic protocol** (`src/statistics.jl`): the exported empty
  generics `compute`, `name` and `compute_all`. Networks.jl defines no methods;
  every model package extends these three for its own statistic types
  (`compute(term, net)` in ERGM.jl, `compute(stat, state, sender, receiver)` in
  REM.jl, ...). Before this, ERGM.jl and REM.jl each defined and exported their
  *own* `compute` and `name`, so `using ERGM, REM` — cross-sections with ERGM,
  dynamics with REM — left the unqualified verbs **undefined** under Julia's
  conflicting-export rule (REM.jl#3). One generic, extended by everyone, cannot
  collide; the methods differ in signature, so nothing is ambiguous. Same move
  as `gof`, for the same reason. Extend from anywhere with
  `import Networks: compute, name`.
- Bundled classic teaching datasets:
  `load_dataset(:florentine_marriage | :florentine_business | :sampson)`
  returns ready-made networks with vertex/edge attributes (data files and
  provenance under `data/`).
- Missing-dyad (unobserved-tie) masks, statnet-style: `set_missing_dyad!`,
  `is_missing_dyad`, `delete_missing_dyad!`, `clear_missing_dyads!`,
  `missing_dyads`, `n_missing_dyads`. Marks survive `copy`, are symmetric on
  undirected networks, and are remapped by `rem_vertex!`,
  `get_induced_subgraph`, and `permute_vertices`. Downstream estimation (ERGM
  MPLE/MCMLE) consumes the mask.
- `Base.copy(::Network)` and `Base.copy(::BipartiteNetwork)`:
  attribute-preserving independent copies (graph, vertex/edge/network
  attributes, and missing-dyad mask). This is the ecosystem-wide fix for the
  attribute-dropping copies that silently zeroed covariate terms in
  ERGM/TERGM/ERGMMulti sampling.
- `Base.zero` for network types, completing the Graphs.jl interface
  (`zero(Network{Int,true})`, `zero(net)` preserving eltype/directedness).
- Shared result-presentation layer for the model packages: `print_coeftable`,
  `format_pvalue`, `signif_code`, `SIGNIF_LEGEND`, `mc_pvalue`,
  `GOFStatistic`, `GOFResult`, `n_simulations`, and the ecosystem-wide `gof`
  generic.
- Typed attribute accessors: `get_vertex_attribute(net, attr, T)` /
  `get_edge_attribute(net, attr, T)` returning concretely typed `Dict`s, and
  `vertex_attribute_vector(net, attr, T; default)` returning a dense vector in
  vertex order — the type-stable hot-loop path used by model code.
- Graphs.jl interface completions: `Graphs.add_vertices!` now extends the
  Graphs generic; `BipartiteNetwork` gains `add_edge!`, `rem_edge!`,
  `neighbors`, `inneighbors`, `outneighbors`; a length/eltype-carrying
  undirected edge iterator; conversion constructors
  `Graphs.SimpleGraph(net)` / `Graphs.SimpleDiGraph(net)`.
- Two-mode support: `as_matrix` on two-mode networks returns the
  `n_mode1 × n_mode2` incidence matrix (`expand_bipartite=true` for the square
  adjacency); `add_edge!` rejects within-mode edges; the density keywords
  (`discount_bipartite=`, and the wrapper's former `n₁·n₂` default) are
  described under Breaking.
- **Every exported and `public` name carries a docstring with a runnable
  example** (panel 2026-09 round 3): 67 of 107 had a docstring with no
  example (`get_induced_subgraph`, `is_two_mode`, `missing_dyads`,
  `check_golden`, `read_pajek`, `add_edges!`, `gof`, the empty generics, the
  constants, …). The empty generics show the `import Networks: compute`
  extension pattern; the golden harness examples load the package's own
  `network_density` fixture. Pinned by the "Every exported and public
  docstring carries a runnable example" testset, which walks
  `Base.Docs.meta(Networks)` and fails on any Networks-owned docstring of an
  exported or public binding without a fenced `julia` block.
- `network_from_matrix` gains a `store_values` keyword to force or suppress
  storing matrix values as an edge attribute.

### Changed

- **`newton_fit` no longer swallows errors as non-convergence** (panel 2026-09
  round 2). The Newton step's `try` was a bare `catch` that turned ANY
  exception into `converged=false, iterations=1` with NaN standard errors — a
  `DimensionMismatch` from a mis-sized gradient, a `MethodError` from a
  non-matrix Hessian, an `InterruptException`. Now only `SingularException`,
  `LAPACKException`, `ZeroPivotException` and `PosDefException` mean "no
  Newton direction"; everything else is rethrown. Because `\` is lenient with
  1×1 and diagonal shapes, the first evaluation additionally checks
  `length(grad) == length(θ)` and `size(hess) == (p, p)` and throws a
  `DimensionMismatch` naming the shapes. An unconverged fit stays loud; a
  swallowed bug reported as non-convergence was the opposite. Pinned in the
  "Shared Newton optimizer" testset (wrong-length gradient, 1×2 Hessian,
  string Hessian, a `\` that raises).
- **`network_density` fixture now pins `diag=false` to `sna::gden(diag=FALSE)`**
  rather than to a hand-derived number: `test/fixtures/r/network_density.R`
  needs `sna` (2.8) beside `network` and records `sna_version` in
  `[provenance]`, with two extra loops networks chosen so the default and
  `diag=false` values differ (5/16 vs 4/12 directed, 2/10 vs 1/6 undirected).
  The undirected hand test in "Network Density" moved from a 3-vertex network
  — where both keywords gave exactly 1/3 and a silent no-op would have passed
  — to a 4-vertex one. `sna::gden(diag=TRUE)` is deliberately not recorded:
  on an undirected graph it divides the symmetric matrix sum by n², not
  `network.density`'s n(n+1)/2, and Julia's `diag=true` follows `network`.
- **`add_edge!` docstring says what the test guarantees**: no allocation over
  Graphs.jl's own `add_edge!` on the wrapped `SimpleDiGraph` (which itself
  may allocate a few dozen bytes on alternate toggles after a removal, a Base
  `insert!` regrowth); `rem_edge!` is 0 B. It used to say "allocation-free",
  which is not what a user measures.
- **Docs and README describe the current package** (panel 2026-09 round 2):
  the logo wordmark reads "Networks.jl" (it still spelled the pre-rename
  "Network.jl"); `docs/src/index.md`'s feature list covers the missing-dyad
  mask and contracts, the datasets, the shared presentation/StatsAPI layer,
  the golden harness and the shared numerics, and its `@contents` lists all
  fourteen pages (six were missing); the README's missing-data bullet no
  longer claims "every conversion adapter" refuses a mask — the adapters into
  another representation do, while the plain `as_matrix`/`as_edgelist` carry
  face values and are lossless only via `missing_value=`/`return_mask=`, as
  the invariant table says; the `:wtc_police_calls` wiring in
  `guide/datasets.md` and the `load_dataset` docstring builds events with
  `Float64` times, since Relevent.jl's statistics run on a `Float64` clock and
  the documented `Int`-time vector failed `fit_obpm` with a `MethodError`
  (REM.jl accepts either).
- Documentation corrected to the current behaviour (panel 2026-09, §6):
  `docs/src/guide/conversion_invariants.md` no longer claims every TSNA point
  measure refuses a masked slice (six do, via SNA; `t_density`/`t_sna_stats`
  now do via `network_density`; `t_reciprocity`, the temporal-path family and
  the edge-dynamics measures still compute at face value — TSNA item 4 tracks
  it), and its invariant table gains honest columns for the three adapters it
  omitted (`Network`→`EgoData`, `Network`s→`MultilayerNetwork`,
  `Matrix`→`RankNetwork`), with a new **I** (ignored) legend entry for the two
  that read the mask at face value with no `missing=`/`report=`. `README.md`
  and `guide/missing_data.md` no longer say conversions "report face values"
  (they are lossless via `missing_value=`/`return_mask=`) or that the mask is
  "consumed by ERGM.jl" alone (every SNA measure, conversion adapter and
  `network_density` refuse a masked network unless asked). `api/metadata.md`'s
  example now runs.
- `read_pajek` semantics overhauled: directedness is decided by the presence
  of an `*Arcs` section; mixed `*Arcs`+`*Edges` files load as directed with
  edges expanded to reciprocal arcs (with a warning); explicit weight columns
  are always stored as `:weight`.
- `write_edgelist_csv` emits RFC-4180-quoted CSV directly and no longer
  requires DataFrames.
- `network_from_matrix` warns when an asymmetric matrix is passed with
  `directed=false` (only the upper triangle is read).
- Constructor validation: `multiple=true` and `hyper=true` now throw
  `ArgumentError` (previously silently accepted but unsupported); two-mode
  sizes are bounds-checked.
- `network_density` returns `NaN` for networks with fewer than two vertices,
  matching R's `network.density` (was `0.0`).
- Documentation and CI currency (panel 2026-09, items 2 and 8): `README.md`
  no longer sends developers to "the root workspace project" (it lives in a
  private repository); clone the repositories side by side and each package's
  `[sources]` path dependencies resolve the siblings. The README feature list
  names the shared contracts (`missing_policies`, `z_pvalues`,
  `CoefficientTable`/`coeftable`, `check_se`, the `newton_fit` home, the
  golden harness) and all six datasets. `CLAUDE.md` records the decision
  that Julia 1.12 stays the floor (the package uses the `public` keyword and
  `Base.kwarg_decl` introspection) instead of the stale "re-examine" note,
  and its file map covers `newton.jl`, `statsapi.jl`, `missing_policies`, the
  density keywords, the datasets and the StatsAPI surface. `docs/src/guide/io.md`'s
  CSV-reading example writes the file it reads, so every ```julia block in
  the README and `docs/src/**` executes under the ecosystem snippet checker
  (`tools/check_snippets.jl Networks.jl`). `.github/workflows/CI.yml` runs
  the Julia `'1'`/ubuntu cell with `JULIA_NUM_THREADS: 4`, so the
  thread-count-independence test of `bootstrap_cov` is genuinely
  multi-threaded in CI.

### Fixed

- `rem_vertex!` now remaps the swapped-in last vertex's vertex attributes,
  incident-edge attributes, and missing-dyad marks onto its new ID under
  Graphs.jl's swap-and-pop, so attribute data stays attached to the right
  vertex.
- `ne()` on undirected networks counts self-loops correctly (no longer
  halved).
- `network_from_edgelist` handles empty edge lists instead of erroring.

### Performance

- **`network_density(::BipartiteNetwork)` is 0 B like the `Network` method**
  (panel 2026-09 round 2): the wrapper forwarded a `kwargs...` splat, which
  allocated 16–32 B per call; it now forwards the three explicit keywords
  (`missing`, `diag`, `discount_bipartite`) and splits on the two concrete
  directednesses of its `network` field (typed `Network{T}`, so a plain
  forward dispatched dynamically and still cost 48 B). The "Network Density"
  testset pins `@allocated == 0` for every keyword combination on both
  wrapper and flagged network.
- **`rem_edge!` is 0 B; `add_edge!` costs exactly what Graphs.jl's own
  `add_edge!` costs** (panel 2026-09, item 26).
  `rem_edge!` walked every edge-attribute `Dict` per removal — via `keys` plus
  a `getindex` per attribute — which halved Metropolis–Hastings throughput on
  any attribute-bearing network (ERGM's `_mh_run!` toggles through it). It now
  returns early on an absent edge, skips the attribute store entirely when the
  network has no edge attributes, and otherwise walks the dictionaries by
  value. Both toggles hand Graphs.jl a concrete `SimpleEdge{T}` instead of
  going through the generic `add_edge!(g, x, y)`, whose `edgetype(g)(x, y)`
  construction allocates. Pinned by the "rem_edge!/add_edge! are
  allocation-free" testset: `rem_edge!` at 0 B with and without attributes,
  and `add_edge!` costing exactly what Graphs.jl's own `add_edge!` on the bare
  `SimpleDiGraph` costs (Base's `insert!` re-grows a `Memory`-backed adjacency
  vector on alternate calls after `deleteat!` under Julia 1.12; that residue
  is Graphs.jl's, not the wrapper's).
- Compile-time directedness constant-folds `is_directed` branches throughout
  the interface layer.
- Typed attribute accessors convert untyped `Dict{Symbol,Dict{T,Any}}` storage
  once into concrete containers, removing type instability from downstream
  change-statistic hot loops.

## [0.1.0] - 2026-02-09

Initial release: `Network`/`BipartiteNetwork` types over Graphs.jl storage,
vertex/edge/network attributes, coercion (matrices, edge lists, DataFrames),
Pajek/CSV I/O, and the Graphs.jl interface.

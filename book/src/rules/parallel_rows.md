# Parallel-row elimination

Parallel-row elimination combines linear constraints whose coefficient rows are proportional. It rescales their bounds to a common row expression, intersects the intervals, and removes one row when the checks permit.

This rule changes constraint rows and their bounds. It does not change the objective or variable bounds, although a resulting equality can enable another reduction.

## Example

Consider `x + 2y <= 5` and `2x + 4y <= 8`. Multiplying the second row by `1/2` gives `x + 2y <= 4`.

The two constraints are therefore equivalent to the tighter row `x + 2y <= 4`; the original upper bound of `5` is redundant.

A negative scale reverses the row sides. For example, `-2x - 4y >= -8` also becomes `x + 2y <= 4` after multiplication by `-1/2`.

Parallel rows need not merely duplicate an upper bound. Combining `x + 2y >= 4` with `2x + 4y <= 8` gives the equality `x + 2y = 4`. Replacing the first lower bound by `5` instead makes the pair infeasible in exact arithmetic.

## Mathematical formulation

Suppose rows \\(i\\) and \\(r\\) satisfy

\\[
A_{i:}=\rho A_{r:},\qquad \rho\ne0,
\\]

with row bounds \\(\ell_i\le A_{i:}x\le u_i\\) and \\(\ell_r\le A_{r:}x\le u_r\\). Row \\(i\\) is retained as the base row.

For \\(\rho>0\\), the second row implies bounds on the base expression

\\[
\widehat\ell=\rho\ell_r,\qquad \widehat u=\rho u_r.
\\]

For \\(\rho<0\\), the sides exchange roles:

\\[
\widehat\ell=\rho u_r,\qquad \widehat u=\rho\ell_r.
\\]

Intersect these with the base row's interval:

\\[
\ell_i^{\prime}=\max(\ell_i,\widehat\ell),
\qquad u_i^{\prime}=\min(u_i,\widehat u).
\\]

If the intersection is nonempty, row \\(r\\) can be removed and the base row receives the intersected bounds. Equal finite endpoints make the retained row an equality. An empty intersection proves infeasibility in exact arithmetic.

For exactly proportional rows, this operation preserves all feasible variable vectors and objective values. Approximate coefficient matches require the numerical qualifications below.

## Candidate discovery

`Model::parallel_rows` in `src/rules/parallel.rs` considers linear rows containing at least two nonzero coefficients. Empty and singleton rows have separate rules.

It computes fingerprints from the variable indices and normalized, quantized coefficients, then sorts candidates into matching groups. Fingerprints only select pairs for comparison; they do not establish proportionality. A collision group can contain several proportional classes, so unmatched members receive an additional canonical ordering and comparison pass instead of an all-pairs search.

A fruitless scan is cached using the row revision. If the row state has not changed since that scan, the function returns without repeating it. Its return value counts pair comparisons, not successful row removals.

## Proportionality check and merging

`merge_parallel_rows` calls `proportional` to perform the authoritative comparison. The rows must be nonempty and have the same number of entries. The scale factor is the first base coefficient divided by the corresponding coefficient in the other row; it must be finite and nonzero.

For each paired entry, the variable indices must match and the scaled coefficient must be finite. The coefficient comparison accepts

\\[
|a-\rho b|\le\tau\max(|a|,|\rho b|),
\\]

where \\(\tau\\) is `numerics.parallel`. The helper separately records whether every scaled coefficient equals its base coefficient exactly in floating-point arithmetic.

After checking that both row domains are linear, the merge rescales the other row's bounds and computes the intersection. A finite source bound that becomes nonfinite during scaling rejects the merge. Infinite bounds representing missing restrictions are allowed.

For an accepted intersection, stronger sides are applied through `Model::tighten_row`. The other row is cleared and a `Record::MergedRow` is saved. The base coefficients remain unchanged.

## Numerical safeguards

If the intersection is empty, the implementation reports infeasibility only when both conditions hold:

- The coefficient comparison was exact under the floating-point scaling check.
- The gap exceeds the configured feasibility tolerance, scaled as below.

\\[
\ell_i^{\prime}-u_i^{\prime} > \varepsilon\bigl(1+\max(|\ell_i^{\prime}|,|u_i^{\prime}|)\bigr).
\\]

Otherwise it rejects the merge without changing the rows. It does not store an inconsistent interval, even if the conflict is small.

The implementation also rejects an approximate merge that would turn a previously non-equality base row into an equality. This guard does not forbid every approximate merge: a nonempty interval, or a base row already represented as an equality, can still pass the remaining checks.

Accepted approximate matches retain the base coefficients while dropping the other row. Consequently, exact feasible-set equivalence is guaranteed by the mathematical rule for truly proportional rows, not by coefficient agreement within tolerance alone. These safeguards limit particular numerical risks; they do not make approximately proportional rows identical.

### Infeasibility certificate

For an exact, sufficiently separated contradiction, the code chooses a sign \\(s\\) and returns row multipliers

\\[
y_i=s,\qquad y_r=-\rho s,
\\]

with all other row multipliers and variable-bound multipliers zero at this stage. It uses \\(s=1\\) when the base lower bound exceeds the scaled upper bound; otherwise it uses \\(s=-1\\). The proportional coefficients cancel, leaving the contradiction in the bounds. Recovery maps the certificate through earlier reductions.

## Postsolve recovery

Primal variable values are unchanged by this rule. Successful tightening of a base side saves a `Record::TightenedRow`, identifying the source row, scale factor, and affected side. Removing the source saves a `Record::MergedRow` afterward.

During reverse dual recovery, the removed row's multiplier is first initialized to zero. If the retained row's signed multiplier uses a tightened side, recovery transfers it to the source: it adds \\(\rho y_i\\) to \\(y_r\\) and sets \\(y_i\\) to zero. Otherwise the multiplier remains on the base row.

For exactly proportional rows, this preserves the contribution to stationarity under the convention \\(Px+c=A^Ty+z\\). Approximate merges can leave a corresponding stationarity residual because the original coefficient rows are not exactly proportional.

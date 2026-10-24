# Parallel-column elimination

Parallel-column elimination combines variables whose constraint columns are proportional when their objective terms also depend only on the same combined variable. It removes one variable and replaces the retained variable's interval with the interval of possible combined values.

Proportional constraint columns alone are not sufficient. The implementation requires compatible quadratic terms and, for aggregation, compatible linear objective coefficients. When linear costs differ, a separate fallback can fix a variable or identify an improving recession direction.

## Example

Suppose every constraint uses `x` and `y` through `x + 2y`, the objective is `3x + 6y + w`, and their bounds are `1 <= x <= 4` and `0 <= y <= 3`.

Introduce

\\[
z=x+2y,\qquad 1\le z\le10.
\\]

The row `x + 2y + w <= 12` becomes `z + w <= 12`, and the objective becomes `3z + w`. Two variables have become one.

If the reduced solution has `z = 7`, one valid recovery is `x = 1`, `y = 3`. Recovery need not reproduce a particular original pair: several pairs can represent the same aggregate.

## Mathematical formulation

Consider a symmetric quadratic objective and two-sided constraints:

\\[
\min_x\;\tfrac12x^TPx+c^Tx+c_0,
\qquad \ell\le Ax\le u,
\qquad L\le x\le U.
\\]

For a nonzero scale \\(\rho\\), suppose

\\[
A_{:k}=\rho A_{:j},
\qquad P_{:k}=\rho P_{:j},
\qquad c_k=\rho c_j.
\\]

These relations make the constraints and objective depend on the pair only through

\\[
z=x_j+\rho x_k.
\\]

The retained coordinate \\(j\\) represents \\(z\\). Remove column \\(k\\) from the constraint matrix, remove row and column \\(k\\) from the quadratic matrix, and drop \\(c_k\\). All remaining coefficients, row bounds, and the objective constant stay unchanged; coefficients associated with coordinate \\(j\\) now multiply the aggregate.

### Aggregate bounds

For \\(\rho>0\\),

\\[
L_z=L_j+\rho L_k,\qquad U_z=U_j+\rho U_k.
\\]

For \\(\rho<0\\),

\\[
L_z=L_j+\rho U_k,\qquad U_z=U_j+\rho L_k.
\\]

All other variable bounds are unchanged. For continuous variables, every value in this interval can be split into an original pair within its bounds.

In exact arithmetic, aggregation maps every original feasible point to a reduced feasible point with the same objective value, and every reduced feasible point admits such an original pair. This preserves the optimal value and permits recovery of optimal solutions, but the correspondence need not be one-to-one.

## Candidate discovery and exact checks

`Model::parallel_columns` in `src/rules/parallel.rs` considers active variables with nonempty constraint columns. Empty columns have a separate rule.

It groups candidates using fingerprints of constraint columns. Within groups sharing a constraint fingerprint, it also fingerprints nonempty quadratic columns and sorts by the combined keys. Fingerprints only select candidates; the code subsequently checks each pair's actual coefficients.

For a proposed pair retaining `j` and removing `k`, `proportional` computes the ratio from column `k` to column `j` and verifies matching row indices and scaled coefficients. Although the helper uses `numerics.parallel` to recognize a potential match, this rule then requires its exact flag: every scaled constraint coefficient must equal its counterpart in floating-point arithmetic.

`Objective::parallel_curvature` in `src/model/objective.rs` checks quadratic columns with zero tolerance, including missing entries in either column. `Model::aggregate` additionally checks that the linear coefficients satisfy the same ratio with zero tolerance. A merely approximate constraint or quadratic match is skipped, unlike some accepted cases of [parallel-row elimination](./parallel_rows.md).

Successful aggregation saves a `Record::ParallelColumns`, updates the retained interval, removes the other variable's constraint column and quadratic entries, and marks affected rows and columns for further processing. A fruitless scan is cached against the model revision. The function returns a comparison count, not a count of eliminated variables.

## Objective-dominance fallback

After exact constraint and quadratic checks, if aggregation fails and the linear-cost difference is finite and nonzero, the code considers

\\[
\Delta=c_k-\rho c_j,
\qquad d_k=-\operatorname{sign}(\Delta),
\qquad d_j=-\rho d_k.
\\]

All other direction components are zero. The direction satisfies \\(Ad=0\\) and \\(Pd=0\\), while \\(c^Td=-|\Delta|<0\\). It preserves the constraints and quadratic contribution while decreasing the objective linearly.

The code examines the variable endpoints in these improving directions:

- If neither endpoint is finite, it returns an improving recession certificate. The full problem is unbounded below if its remaining constraints are feasible.
- If only the retained variable's endpoint is infinite, it fixes the other variable at its finite improving endpoint.
- If only the other variable's endpoint is infinite, it fixes the retained variable at its finite improving endpoint.
- If both endpoints are finite, this fallback does not fix either variable.

For example, with identical constraint columns, costs `2x + y`, and both variables nonnegative without upper bounds, transferring from `x` to `y` lowers the cost. Since `y` can increase indefinitely, the fallback can fix `x = 0` without changing the attainable optimum.

When `dominated_columns` is enabled, the function also invokes a separate dominated-column pass on the discovered support groups. That broader rule is distinct from aggregation and the two-variable fallback described here.

## Numerical safeguards

The scale must be finite and nonzero, and coefficient comparisons reject nonfinite scaled values. Quadratic and linear objective comparisons likewise require finite values.

Aggregate bound calculations reject NaNs and a nonfinite endpoint produced from two finite contributing endpoints. Infinite endpoints representing absent restrictions are otherwise permitted. A failed aggregation leaves the pair unaggregated; the fallback above can still apply when its conditions hold.

The exactness checks are floating-point checks, not symbolic proofs of the real-number relations. They avoid accepting tolerance-only null directions, whose small constraint or quadratic discrepancies could become significant for large variable values. The mathematical equivalence statements assume exact proportionality and arithmetic.

## Postsolve recovery

`Record::ParallelColumns` saves the retained and removed indices, scale, and both original bound intervals. During ordinary primal recovery, the saved aggregate is split into a pair satisfying those intervals and \\(z=x_j+\rho x_k\\).

The implementation anchors the retained variable at its finite lower bound, otherwise its finite upper bound, otherwise zero. It computes \\(x_k=(z-\text{anchor})/\rho\\), clips that value to the removed variable's interval, then sets \\(x_j=z-\rho x_k\\). For a feasible aggregate, this gives a valid original pair in exact arithmetic.

During dual recovery, the removed variable's signed bound multiplier is restored as \\(z_k=\rho z_j\\), where these \\(z_j,z_k\\) denote bound multipliers, not the aggregate variable. This is consistent with the proportional gradient and constraint columns under \\(Px+c=A^Ty+z\\). The recovery tape also handles recession directions and infeasibility multipliers through the aggregation.

Fallback fixings use `Record::Fixed` and the ordinary [fixed-variable recovery](./fixed_variables.md) instead.

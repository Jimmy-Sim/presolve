# Short equality elimination

Short equality elimination solves a linear equality containing several variables for one selected variable, substitutes that expression into the objective and other constraints, and removes the variable when numerical and sparsity checks permit.

It generalizes [doubleton equality elimination](./doubleton_equalities.md) to rows with at least three nonzero coefficients. The implementation limits row size, candidate variables, substitution work, and new coefficients to avoid expensive or unstable reductions.

## Example

Suppose a problem contains `3x + 2y - z = 12`, another row `x + w <= 7`, and the objective `6x + y + w`. Assume `x` has no bounds or quadratic objective terms.

Solving for `x` gives

\\[
x=4-\tfrac23y+\tfrac13z.
\\]

The other row becomes `-2y/3 + z/3 + w <= 3`, and the objective becomes `24 - 3y + 2z + w`. The defining equality and variable `x` can be removed if the substitution passes the implementation's checks.

This example illustrates why fill matters: replacing `x` can introduce both `y` and `z` into rows that previously contained neither.

## Mathematical formulation

Consider an equality in row \\(i\\), with a nonzero pivot coefficient \\(a_{ik}\\):

\\[
\sum_j a_{ij}x_j=b_i.
\\]

Let \\(R\\) contain all variable indices except \\(k\\), and set

\\[
\delta=b_i/a_{ik},
\qquad s_j=-a_{ij}/a_{ik}\quad(j\in R).
\\]

Coefficients absent from the row have \\(s_j=0\\). The substitution is

\\[
x_k=\delta+s^Tx_R.
\\]

For another linear row \\(r\\), substitution gives

\\[
\ell_r-A_{rk}\delta
\le (A_{rR}+A_{rk}s^T)x_R
\le u_r-A_{rk}\delta.
\\]

Rows containing the eliminated variable can gain, combine, or lose coefficients. Bounds on the eliminated variable become

\\[
L_k\le\delta+s^Tx_R\le U_k.
\\]

Any sides already implied by the equality and the other variables' bounds can be omitted. Necessary sides must remain as a constraint on the surviving variables. If no sides remain, the defining equality can be deleted entirely.

### Objective update and equivalence

Write the reconstruction as \\(x=Tx_R+h\\): surviving coordinates are copied unchanged, coordinate \\(k\\) uses the slopes \\(s\\), and \\(h_k=\delta\\) with all other entries zero.

For a symmetric quadratic objective \\(f(x)=\tfrac12x^TPx+c^Tx+c_0\\), the reduced data are

\\[
P_{\mathrm{red}}=T^TPT,
\qquad c_{\mathrm{red}}=T^T(c+Ph),
\qquad c_{0,\mathrm{red}}=c_0+c^Th+\tfrac12h^TPh.
\\]

In exact arithmetic, retaining all necessary bound restrictions gives a one-to-one correspondence between feasible original and reduced points, with equal objective values. The reduction preserves optimal solutions. The formulas also apply to quadratic candidates when the settings permit them; the defaults restrict candidates to variables without quadratic participation.

## Default eligibility

`Model::short_equalities` in `src/rules/substitution.rs` processes queued linear equalities. `EqualitySettings::default` in `src/settings.rs` supplies the following limits:

| Setting | Default | Effect |
| --- | --- | --- |
| `max_row_length` | `8` | Consider equalities with 3–8 nonzero coefficients. |
| `min_column_length` | `2` | Candidate variables occur in at least two constraint rows. |
| `max_column_length` | `8` | Candidate variables occur in at most eight constraint rows. |
| `require_free_variable` | `true` | Candidates have no lower or upper variable bound. |
| `require_linear_variable` | `true` | Candidates have no quadratic objective terms, including cross terms. |
| `relative_pivot` | `1.0` | The pivot magnitude must equal the largest coefficient magnitude in the row. |
| `max_pivot_attempts` | `1` | Retain at most one candidate for attempted substitution per row per pass. |
| `cost_aware` | `false` | Rank eligible candidates by constraint-column length after preferring free variables. |
| `preserve_nonzeros` | `true` | Further restrict the fill allowance using the number of entries removed. |

These are the default equality settings; callers can change them. Singleton columns and rows with fewer than three entries have separate rules. Setting `max_pivot_attempts` to zero disables attempts in this function.

## Pivot filtering and candidate ranking

The function first checks row size, variable bounds, quadratic participation, and constraint-column length. It then compares each candidate's coefficient magnitude with the largest magnitude in the entire equality row.

A `relative_pivot` in `(0, 1]` sets the minimum acceptable ratio to that row maximum. Other values fall back to `1.0`. The default avoids substitution slopes larger than one in magnitude; smaller values allow more amplification.

Eligible candidates are ranked by:

1. Free variables before bounded variables, when bounded candidates are permitted.
2. Estimated substitution work when `cost_aware` is enabled; otherwise constraint-column length.
3. Variable index to break ties.

The work estimate includes row length times column length and an estimate of quadratic work from cross terms and diagonal curvature. The function retains at most `max_pivot_attempts` candidates and tries them in ranked order, stopping after the first successful substitution. Increasing the attempt limit permits alternatives when an earlier candidate fails.

## Bounds, fill, and numerical checks

For a bounded candidate, `singleton_range` computes the interval implied by solving the equality using the other variables' bounds. `non_implied_bounds` keeps only original variable bounds not already guaranteed by that interval. Free candidates have no bounds to retain.

The shared `Model::substitute` helper receives these effective bounds. It can retain the defining row as a constraint on the surviving variables or remove it if no bounds remain. Unlike a doubleton, the retained row can still contain several variables.

The ordinary fill allowance is `substitution_fill`. With `preserve_nonzeros` enabled, it is further capped by the pivot column length, plus the equality's remaining coefficient count when the defining row can be deleted. This is a cap on added entries, separate from the candidate's estimated work and the policy controlling Hessian growth.

The shared helper checks finite offsets, slopes, row-bound transformations, and objective updates before committing a substitution. It also checks constraint and quadratic fill limits, Hessian growth, and applicable deadlines. Rejection leaves the candidate variable in the model and does not establish infeasibility.

## Work limits and scheduling

Each pass has an estimated-work allowance. By default this is twice the current number of nonzeros in the constraint matrix. `WorkLimit::Entries` sets an explicit allowance, while `WorkLimit::Unlimited` removes the work cap without removing the deadline.

Before an attempt, the function augments its candidate estimate with the lengths of rows incident to the candidate column. A candidate that exceeds the remaining allowance is not attempted; otherwise the estimated cost is deducted even if substitution later fails.

With an explicit work-limit setting, including `Unlimited`, rows deferred because of work or deadline limits can be requeued. With the default work-limit setting, these rows are not explicitly requeued by this function. The rule records examined rows, attempts, accepted substitutions, and rejection reasons in `equality_stats`.

## Postsolve recovery

Successful substitution saves a `Record::Substituted` with the original working equation, objective derivative information, other row incidences, and whether a bound row was retained.

During ordinary primal recovery, postsolve evaluates the saved equality to restore the eliminated variable after recovering the surviving variables it depends on. Dual recovery uses the saved information to reconstruct row and signed variable-bound multipliers under the convention \\(Px+c=A^Ty+z\\). The recovery tape also supports infeasibility certificates and recession directions through earlier substitutions.

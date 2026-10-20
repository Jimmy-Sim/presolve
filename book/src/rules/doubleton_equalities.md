# Doubleton equality elimination

A doubleton equality is a linear equality with exactly two nonzero coefficients. Presolve can solve it for one variable, substitute that expression into the objective and other constraints, and remove the variable when numerical and sparsity checks permit.

Unlike singleton-column elimination, the eliminated variable can appear in several other constraint rows. Its bounds must also survive the substitution.

## Example

Suppose the problem contains `2x + 3y = 12`, another row `x + z <= 8`, and the objective `4x + y + z`.

Solving the equality for `x` gives

\\[
x=6-\tfrac32 y.
\\]

Substitution changes the other row to `-1.5y + z <= 2` and the objective to `24 - 5y + z`.

If `x` is unbounded, the defining equality can be removed. If `0 <= x <= 6`, those bounds instead become `0 <= y <= 4`. The implementation initially retains a singleton row representing these restrictions; singleton-row elimination can convert it into variable bounds.

## Mathematical formulation

Consider the equality

\\[
a x_j+b x_k=d,
\qquad a\ne0,\quad b\ne0.
\\]

Eliminating \\(x_j\\) expresses it in terms of the surviving variable:

\\[
x_j=\delta+s x_k,
\qquad \delta=d/a,\quad s=-b/a.
\\]

### Other constraint rows

For another row \\(r\\), substitution gives

\\[
\ell_r-A_{rj}\delta
\le \sum_{t\ne j,k}A_{rt}x_t+(A_{rk}+A_{rj}s)x_k
\le u_r-A_{rj}\delta.
\\]

Thus a row containing the eliminated variable changes its bounds and its coefficient of the survivor. A survivor coefficient can be created, combined with an existing coefficient, or canceled entirely. Infinite row bounds remain infinite.

### Preserve variable bounds

Bounds \\(L_j\le x_j\le U_j\\) become

\\[
L_j\le\delta+s x_k\le U_j.
\\]

For a positive pivot \\(a\\), the implementation represents this as the singleton row

\\[
d-aU_j\le b x_k\le d-aL_j.
\\]

For a negative pivot, the lower and upper variable bounds exchange roles. These restrictions must be combined with the survivor's existing bounds; they can expose infeasibility or a fixed variable. If the eliminated variable has no bounds, the defining row can be deleted instead.

### Objective update and equivalence

Let \\(z\\) contain all variables except \\(x_j\\). Write the reconstruction as \\(x=Tz+h\\), where the surviving coordinates are copied unchanged and coordinate \\(j\\) is \\(\delta+s x_k\\). Here \\(h_j=\delta\\) and the other entries of \\(h\\) are zero.

For a symmetric quadratic objective \\(f(x)=\tfrac12 x^TPx+c^Tx+c_0\\), substitution gives

\\[
P_{\mathrm{red}}=T^TPT,
\qquad c_{\mathrm{red}}=T^T(c+Ph),
\qquad c_{0,\mathrm{red}}=c_0+c^Th+\tfrac12 h^TPh.
\\]

Cross terms can therefore change the survivor's quadratic coefficients as well as linear coefficients and the constant. For a linear objective, the eliminated term contributes \\(c_j\delta\\) to the constant and \\(c_js\\) to the survivor's coefficient.

In exact arithmetic, substitution with all necessary variable bounds retained gives a one-to-one correspondence between feasible points of the reduced and original problems, with equal objective values. Optimal solutions are preserved.

## Implementation overview

`Model::doubleton_equalities` in `src/rules/substitution.rs` takes a round from the doubleton-row queue. It first attempts the batch path described below, then processes the remaining candidates through `doubleton_round`.

The individual path checks the deadline, confirms a linear equality, and rechecks that exactly two coefficients remain. It chooses the elimination direction as follows:

1. If either variable participates in the quadratic objective and coefficient magnitudes differ, prefer the larger pivot magnitude to limit amplification.
2. Otherwise, prefer a singleton constraint column when only one of the two columns is singleton.
3. Prefer a direction whose coefficient ratio is approximately integral when only one direction has that property.
4. Otherwise, prefer the shorter constraint column; ties select the second entry in the row's iteration order.

The approximately integral test uses a relative tolerance of `1e-12`. It is a coefficient heuristic, not an integer-variable restriction.

The selected coefficient ratio must be finite with magnitude between `1e-7` and `1e7`, inclusive. If it fails this check, the row is skipped. If the preferred substitution is attempted but fails, the opposite pivot is tried only when neither variable participates in the quadratic objective.

## Substitution checks and bound handling

`Model::substitute` and `substitute_equation` in `src/model/mod.rs` perform the shared substitution. The individual doubleton rule passes the eliminated variable's full current bounds, rather than first discarding implied sides as the singleton-column rule can do.

The helper checks finite offsets, slopes, and transformed finite row bounds. It prepares changes to the other rows and checks the objective update before committing the reduction. New coefficients are limited by `substitution_fill`, and Hessian growth has a separate policy.

When `allow_unit_doubleton_fill` is enabled, neither variable participates in the quadratic objective, and the two row coefficients have equal magnitudes, the rule lifts the ordinary gross fill cap. The substitution slope then has magnitude one, the Hessian is unchanged, and each new survivor entry replaces a removed pivot entry. Other numerical checks still apply.

If bounds must survive, the defining row becomes a singleton constraint. When enabled, [singleton-row elimination](./singleton_rows.md) converts it into bounds on the survivor. Otherwise the singleton constraint remains to enforce those bounds.

## Batch elimination of equality trees

`batch_doubleton_chains` in `src/rules/doubleton_chains.rs` handles suitable groups before individual substitution. It is restricted to linear objectives without cone blocks and sufficiently large candidate frontiers, with no pending singleton rows.

Eligible edges have equal coefficient magnitudes and a finite right-hand side. A connected component must be a tree with at least eight edges and at most one bounded variable. The bounded variable, if present, is retained as the representative; otherwise the representative is chosen by largest constraint-column length, breaking ties by variable index.

The batch expresses the other variables as affine functions of the representative, rebuilds affected rows, and updates the objective. Eliminated variables are free. Cycles and other ineligible components remain for individual processing, which can expose singleton or empty rows through substitution.

Finite-arithmetic, cancellation, fill, and deadline checks can reject a batch before committing changes. A rejected component remains available to the ordinary doubleton path.

## Postsolve recovery

Individual substitution saves a `Record::Substituted` containing the defining equation, objective derivative information, other row incidences, and whether a bound row was retained.

During ordinary primal recovery, postsolve restores the eliminated variable from the saved equality after recovering the survivor. Dual recovery uses the saved objective derivative and row coefficients to reconstruct row and signed variable-bound multipliers under the convention \\(Px+c=A^Ty+z\\).

The batch path instead saves a `Record::DoubletonChain` with the original equations and incidences. Reverse recovery restores eliminated variables and multipliers in the order needed by their dependencies. Both paths retain information for certificate recovery as well as ordinary solutions.

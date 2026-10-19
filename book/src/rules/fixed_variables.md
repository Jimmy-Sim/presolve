# Fixed-variable elimination

Fixed-variable elimination simplifies a problem by substituting a variable’s fixed value into the objective function and constraints, then removing that variable from the reduced problem.

The eliminated variable's original position and fixed value are recorded so that postsolve can reconstruct the original variable vector.

## Linear objective example

Consider minimizing `3x + 2y`, subject to `x + y >= 10` and `y = 4`.

Substituting `y = 4` changes the objective to `3x + 8` and the remaining constraint to `x >= 6`. The reduced problem contains only the variable `x`.

For any feasible reduced value of `x`, restoring `y = 4` gives a feasible point in the original problem with the same objective value.

## Mathematical formulation

Consider the quadratic objective

\\[
f(x) = \tfrac12 x^T P x + c^T x + c_0,
\\]

where \\(P\\) is symmetric. Suppose variable \\(x_k\\) has equal finite bounds, fixing it to a value \\(v\\). Let \\(R\\) denote the indices of the remaining variables. Subscripts select the corresponding entries or submatrices; for example, \\(P_{RR}\\) contains the remaining rows and columns, and \\(A_{:k}\\) denotes column \\(k\\) of the constraint matrix.

### Objective update

Substituting \\(x_k = v\\) gives the reduced objective

\\[
f_{\mathrm{red}}(x_R) = \tfrac12 x_R^T P_{RR} x_R + (c_R + P_{Rk}v)^T x_R + c_0 + c_kv + \tfrac12 P_{kk}v^2.
\\]

Thus, the reduced objective data are

\\[
P_{\mathrm{red}} = P_{RR},
\qquad
c_{\mathrm{red}} = c_R + P_{Rk}v,
\qquad
c_{0,\mathrm{red}} = c_0 + c_kv + \tfrac12 P_{kk}v^2.
\\]

Quadratic coefficients between remaining variables are unchanged. Cross terms involving the fixed variable contribute to the remaining linear coefficients, while terms involving only the fixed variable contribute to the constant.

### Linear constraint update

For two-sided linear constraints

\\[
\ell \le Ax \le u,
\\]

substitution gives

\\[
\ell - A_{:k}v
\le A_{:R}x_R
\le u - A_{:k}v.
\\]

Infinite row bounds remain unchanged. Equality constraints are the special case \\(\ell = u = b\\), giving

\\[
A_{:R}x_R = b - A_{:k}v.
\\]

The bounds on the remaining variables are unchanged by this substitution.

### Primal recovery and equivalence

Postsolve reconstructs the variable vector by inserting \\(x_k = v\\) at its saved index and retaining the recovered values of \\(x_R\\).

In exact arithmetic, this gives a one-to-one correspondence between feasible points of the reduced and original problems, with

\\[
f_{\mathrm{red}}(x_R) = f(x).
\\]

Consequently, the reduction preserves objective values and optimal solutions.

## Implementation overview

The `fixed_variables` function processes queued variable indices. For each variable that is still active and has exactly equal, finite bounds, it calls `fix` with the shared bound value.

The equality check uses exact equality, with no tolerance: nearly equal bounds do not qualify as fixed.

The `fix` function checks that the affected rows can be shifted, substitutes the value into the objective, removes the variable's constraint column, and updates the affected rows. It also saves a `Record::Fixed` record for postsolve.

This operation substitutes an already determined value; it does not search for an optimal value.

## Quadratic objective example

Consider the objective `x^2 + 3xz + 2z^2 + 5z + 10`, with `z` fixed to `7`.

Substitution gives `x^2 + 21x + 143`:

- The cross term `3xz` becomes the linear term `21x`.
- The terms `2z^2` and `5z` contribute `98 + 35` to the constant. Including the original constant `10` gives `143`.
- The term `x^2` remains unchanged because it does not contain `z`.

The objective substitution updates the remaining linear coefficients and the constant, then removes the fixed variable's row and column from the quadratic matrix. Quadratic coefficients between remaining variables stay unchanged.

## Numerical checks

Objective substitution checks that the computed constant and updated coefficients are finite before applying changes. If these checks fail, `try_substitute` returns a numerical error, `substitute` converts it to `None`, and `fix` returns `false`.

This means the substitution was not performed; it does not establish that the optimization problem is infeasible.

## Row updates and postsolve recovery

For a linear row, elimination subtracts the fixed variable's contribution from both row bounds; infinite bounds remain unchanged. For a conic row, it subtracts the fixed variable's contribution from the stored right-hand side while preserving the cone block. The row-shifting check rejects deleted rows and nonfinite shifts or results from finite bounds.

During ordinary solution recovery, postsolve restores the saved fixed value at the variable's original index. The saved objective gradient and removed constraint coefficients are also used for dual recovery.

### Dual multiplier recovery

During ordinary solution recovery, postsolve recovers the fixed variable's signed bound multiplier from stationarity. The implementation uses the sign convention

\\[
Px + c = A^T y + z,
\\]

where \\(y\\) contains the row multipliers and \\(z\\) contains the signed variable-bound multipliers. Thus, after restoring \\(x_k = v\\), the missing multiplier is

\\[
z\_k = (Px + c)\_k - A\_{:k}^T y.
\\]

The first term is the objective derivative with respect to the eliminated variable. The second term sums the contributions of its constraint coefficients and the corresponding row multipliers.

The `Record::Fixed` record saves the objective derivative representation and removed constraint coefficients at elimination time. During reverse recovery, `gradient.evaluate(&point.x)` evaluates that derivative at the recovered point, and the implementation computes

\\[
z_k = g_k - \sum_i a_{ik}y_i.
\\]

The result is stored in `point.z[k]`. These quantities refer to the working problem at the corresponding stage of elimination and reverse recovery.

Because the variable has equal lower and upper bounds, its signed bound multiplier can have either sign. The implementation recovers this combined multiplier rather than separate lower- and upper-bound multipliers.

#### Example

Consider

\\[
\min_{x,w} (x^2 + w^2)
\quad\text{subject to}\quad
x = 5,\qquad x + w = 8.
\\]

Here, \\(x = 5\\) is represented by equal variable bounds. Eliminating it leaves the objective \\(25 + w^2\\) and the row \\(w = 3\\).

The reduced solution is \\(w = 3\\). Since \\(w\\) has no variable bounds, its stationarity equation gives the row multiplier \\(y = 2w = 6\\).

Postsolve restores \\(x = 5\\), then recovers

\\[
z_x = 2x - 1 \cdot y = 10 - 6 = 4.
\\]

This satisfies the eliminated variable's stationarity equation, \\(2x = y + z_x\\).
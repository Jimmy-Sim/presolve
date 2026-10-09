# Fixed-variable elimination

Fixed-variable elimination simplifies a problem by substituting a variable’s fixed value into the objective function and constraints, then removing that variable from the reduced problem.

The eliminated variable's original position and fixed value are recorded so that postsolve can reconstruct the original variable vector.

## Linear objective example

Consider minimizing `3x + 2y`, subject to `x + y >= 10` and `y = 4`.

Substituting `y = 4` changes the objective to `3x + 8` and the remaining constraint to `x >= 6`. The reduced problem contains only the variable `x`.

For any feasible reduced value of `x`, restoring `y = 4` gives a feasible point in the original problem with the same objective value.

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

During ordinary solution recovery, postsolve recovers the fixed variable's signed bound multiplier using stationarity of the Lagrangian.

Stationarity means that the derivative of the Lagrangian with respect to each primal variable is zero.

For an eliminated variable at index `k`, the recovery formula is:

`z_k = g_k - sum_i(a_ik * y_i)`

Here, `g_k` is the saved objective derivative evaluated at the restored point, `a_ik` is the saved coefficient of the eliminated variable in row `i`, and `y_i` is that row's recovered dual multiplier. The result is stored in `point.z[k]`.

For example, consider minimizing `x^2 + w^2`, with `x` fixed to `5` and the remaining constraint `x + w = 8`.

After substituting `x = 5`, the reduced problem is to minimize `25 + w^2` subject to `w = 3`. Using the Lagrangian sign convention `L = f - lambda*(x - 5) - mu*(x + w - 8)`, stationarity with respect to `w` gives `mu = 2w = 6`.

Postsolve restores `x = 5`. Stationarity with respect to `x` requires:

`2x - lambda - mu = 0`

Therefore, the missing multiplier is `lambda = 10 - 6 = 4`. This corresponds to the code's recovery formula: the objective derivative is `10`, the saved row coefficient is `1`, and the row multiplier is `6`.

The implementation represents the fixed value through equal lower and upper bounds. It recovers a single signed bound multiplier, corresponding to `lambda` in this example.
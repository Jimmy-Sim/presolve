# Empty-column elimination

An empty column is a variable with no nonzero coefficients in the constraint matrix. Its objective terms and variable bounds determine whether presolve can choose an optimal value, remove it by quadratic elimination, or identify an improving recession direction.

An empty constraint column does not necessarily mean an independent variable: quadratic cross terms can still couple it to other variables in the objective.

## Linear examples

Suppose `x` appears in no constraints and has bounds `1 <= x <= 5`.

- With objective `3x + 2y`, the best choice is `x = 1`. Removing `x` leaves the objective `3 + 2y`.
- With objective `-3x + 2y`, the best choice is `x = 5`, leaving `-15 + 2y`.
- With no objective term involving `x`, any value in its interval works. The implementation chooses zero clipped to the interval, giving `x = 1` here.

If the objective coefficient is negative and there is no upper bound, increasing `x` decreases the objective without limit. This establishes an improving direction; the full problem is unbounded below if the remaining constraints are feasible.

## Mathematical formulation

Consider

\\[
\min_x \; \tfrac12 x^T P x+c^T x+c_0,
\qquad \ell\le Ax\le u,
\qquad L\le x\le U,
\\]

where \\(P\\) is symmetric. Suppose column \\(k\\) is empty, so \\(A_{:k}=0\\). First assume there are no quadratic cross terms involving \\(x_k\\): \\(P_{jk}=0\\) for every \\(j\ne k\\).

Let \\(R\\) denote the remaining variable indices and \\(p=P_{kk}\\). The variable's independent objective contribution is

\\[
q_k(t)=\tfrac12 p t^2+c_k t,
\qquad L_k\le t\le U_k.
\\]

### Choosing the value

For \\(p>0\\), the unconstrained minimizer is \\(-c_k/p\\). Project it onto the variable's interval:

\\[
v=\min\bigl(U_k,\max(L_k,-c_k/p)\bigr).
\\]

For \\(p=0\\), choose the lower bound when \\(c_k>0\\), or the upper bound when \\(c_k<0\\). When both \\(p\\) and \\(c_k\\) are zero, the implementation chooses

\\[
v=\min\bigl(U_k,\max(L_k,0)\bigr).
\\]

A finite chosen value can be substituted and the variable removed. A negative diagonal coefficient is skipped by this rule.

### Reduced problem and optimality

With a finite minimizing value \\(v\\), the reduced objective is

\\[
\tfrac12 x_R^T P_{RR}x_R+c_R^Tx_R+c_0+q_k(v).
\\]

The remaining constraints use \\(A_{:R}\\) with unchanged row bounds. The remaining variable bounds are also unchanged.

Every feasible reduced point can be extended with \\(x_k=v\\), and any original feasible point can have its \\(x_k\\) replaced by \\(v\\) without increasing the objective. Thus the reduction preserves the optimal value and permits recovery of an original optimum when one exists. It does not preserve every original feasible point or its objective value.

### Quadratic example

For the independent objective contribution `x^2 - 6x` and bounds `0 <= x <= 2`, the quadratic convention gives `p = 2` and `c = -6`. The unconstrained minimizer is `3`, so the chosen value is `2`. Substitution contributes `-8` to the objective constant.

### Improving recession direction

For \\(p=0\\) and \\(c_k\ne0\\), a nonfinite improving endpoint gives the direction

\\[
d_k=-\operatorname{sign}(c_k),
\qquad d_j=0\quad(j\ne k).
\\]

Because the constraint column is empty, \\(Ad=0\\). There are no quadratic terms involving the variable, and \\(c^Td=-|c_k|<0\\). The absent bound permits movement indefinitely in this direction. This direction does not itself establish that the remaining constraints are feasible.

## Implementation overview

`Model::empty_columns` in `src/rules/variables.rs` drains the empty-column queue. It skips variables that are no longer active or whose constraint columns are no longer empty.

`Objective::diagonal` in `src/model/objective.rs` returns the diagonal coefficient only if the variable has no quadratic cross terms. A missing quadratic term gives a diagonal value of zero.

For an independent variable, the rule skips negative curvature and selects the value using the cases above. If the result is finite, it calls `Model::fix`. This chooses an optimal value before using the shared substitution machinery; unlike fixed-variable elimination, the variable need not already have equal bounds.

If the selected value is nonfinite and the contribution is linear with a nonzero coefficient, the rule returns a recession certificate through `recession_certificate` and `dual_certificate`. Recovery maps the direction back through earlier reductions.

## Coupled quadratic variables

When quadratic cross terms exist, the rule does not choose a constant value. It can delegate to `Model::eliminate_coupled` under the separate `QuadraticElimination` rule when quadratic elimination is enabled, the variable has no lower or upper bound, and its Hessian row has at most 16 stored entries.

That helper requires a finite, positive diagonal coefficient and uses the affine minimizer

\\[
x_k=-\frac{c_k+\sum_{j\ne k}P_{kj}x_j}{P_{kk}}.
\\]

It substitutes this expression into the objective, subject to finite-arithmetic and fill checks, and prohibits net Hessian growth. A rejected attempt is not retried at the same Hessian revision. Bounded coupled variables are skipped here because their minimizing expression can require clipping and need not be affine.

## Numerical checks and postsolve recovery

The independent branch calls `fix` only for a finite chosen value. The shared objective substitution checks the resulting constant and coefficients before applying changes. If substitution fails, the variable is not removed. A nonfinite computed value in the positive-curvature case does not trigger the linear recession certificate.

Successful constant substitution saves a `Record::Fixed`. During ordinary recovery, postsolve restores the chosen value. Since this column had no constraint entries, its signed bound multiplier is recovered from the saved objective derivative, \\(p v+c_k\\), using the stationarity convention \\(Px+c=A^Ty+z\\).

The coupled path instead saves a `Record::Eliminated` containing the offset and slopes. Postsolve reconstructs the variable from that affine expression after recovering the variables on which it depends.

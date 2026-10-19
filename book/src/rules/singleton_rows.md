# Singleton-row elimination

Singleton-row elimination replaces a linear constraint containing exactly one nonzero coefficient with bounds on its variable, then removes the constraint once those bounds capture its restriction.

The variable remains in the problem. Its objective coefficients and its coefficients in other rows are unchanged by this rule.

## Example

Suppose a problem contains the row `6 <= 2x <= 14`, and the variable already has bounds `0 <= x <= 5`.

Dividing the row by `2` gives `3 <= x <= 7`. Intersecting these bounds with the existing bounds gives `3 <= x <= 5`.

The row can now be removed: its restriction is fully represented by the variable bounds.

## Mathematical formulation

Consider two-sided linear constraints

\\[
\ell \le Ax \le u,
\\]

with variable bounds \\(L \le x \le U\\). Suppose row \\(i\\) contains exactly one nonzero coefficient, in column \\(k\\):

\\[
\ell_i \le a_{ik}x_k \le u_i,
\qquad a_{ik} \ne 0.
\\]

### Implied variable bounds

For a positive coefficient, divide both row bounds by \\(a_{ik}\\). For a negative coefficient, division reverses the inequalities, so the bounds exchange roles:

For \\(a_{ik}>0\\),

\\[
\widehat L_k=\ell_i/a_{ik},
\qquad \widehat U_k=u_i/a_{ik}.
\\]

For \\(a_{ik}<0\\),

\\[
\widehat L_k=u_i/a_{ik},
\qquad \widehat U_k=\ell_i/a_{ik}.
\\]

Intersect the implied interval with the existing variable bounds:

\\[
L_k^{\prime} = \max(L_k,\widehat L_k),
\qquad
U_k^{\prime} = \min(U_k,\widehat U_k).
\\]

For example, the row \\(6 \le -2x_k \le 14\\) implies \\(-7 \le x_k \le -3\\).

### Infeasibility, redundancy, and fixed variables

In exact arithmetic, \\(L_k^{\prime} > U_k^{\prime}\\) proves infeasibility. For example, `6 <= 2x <= 14` conflicts with an existing upper bound `x <= 2`.

If the existing interval already lies inside the implied interval, the row is redundant and can be removed without changing the bounds. Thus `3 <= x <= 7` adds nothing when `4 <= x <= 6` already holds.

If the updated bounds coincide at a finite value, the variable becomes eligible for [fixed-variable elimination](./fixed_variables.md). Singleton-row elimination itself does not substitute or remove the variable.

### Reduced problem and equivalence

Let \\(R\\) contain all row indices except \\(i\\). The reduced linear constraints are

\\[
\ell_R \le A_{R:}x \le u_R,
\\]

with the updated bounds on \\(x_k\\); all other variable bounds are unchanged. Here, \\(A_{R:}\\) selects the remaining rows and all columns.

In exact arithmetic, the reduced and original problems have the same feasible variable vectors. The objective is unchanged, so objective values and optimal solutions are preserved. No primal variable needs to be reconstructed for this reduction alone.

## Implementation overview

The implementation is split across these functions:

- `Model::singleton_rows` in `src/rules/rows.rs` processes candidate rows and computes their implied bounds.
- `Model::implied_bound` in `src/rules/bounds.rs` checks feasibility and whether each proposed bound tightens the current interval.
- `Model::tighten_bound` and `Model::delete_row` in `src/model/mod.rs` apply updates and record them for postsolve.

The rule drains the singleton-row queue, rechecks that each row still has exactly one coefficient, and obtains a snapshot of its linear equation. Rows without a linear equation are skipped.

It computes the implied lower and upper bounds using the coefficient's sign, then passes each to `implied_bound`. Bounds that do not tighten the current interval leave it unchanged.

The row is deleted only if the stored variable bounds actually satisfy both implied bounds:

```rust
if self.bounds[j].lower >= lower && self.bounds[j].upper <= upper {
    self.delete_row(i);
}
```

The cleanup loop repeats while the model changes. If a bound update fixes a variable, it queues that variable for fixed-variable elimination on a subsequent cleanup iteration, when that rule is enabled.

## Numerical checks

The implementation uses a feasibility tolerance when deciding whether conflicting bounds establish infeasibility. The helper `separated` checks

\\[
L > U + \varepsilon\bigl(1+\max(|L|,|U|)\bigr),
\\]

where \\(\varepsilon\\) is the configured feasibility tolerance. A conflict satisfying this test returns an infeasibility certificate. A smaller conflict rejects the proposed bound rather than storing an inconsistent interval.

Nonfinite proposed bounds are also rejected. Infinite row sides can represent missing restrictions, while division can overflow even when its inputs are finite. The final containment check prevents deletion unless the stored bounds capture both computed restrictions; it uses direct comparisons without a tolerance.

Singleton rows call `implied_bound` with `propagation = false`, bypassing the minimum-gain and huge-bound filters used for general bound propagation. The finite-value and feasibility checks still apply.

## Postsolve recovery

Each successful tightening saves a `Record::TightenedBound` containing the variable index, the affected side, its old bound, and the equation proving the new bound. Removing the row also saves a `Record::DeletedRow`.

Primal values are unchanged by singleton-row elimination. During reverse recovery, the deleted row's multiplier is initially zeroed. Tightened-bound recovery can then transfer a signed variable-bound multiplier to the saved row when the tightening, rather than the original bound, accounts for it.

For a singleton coefficient \\(a_{ik}\\), transferring a signed bound multiplier \\(z_k\\) contributes \\(z_k/a_{ik}\\) to the row multiplier and removes that contribution from the variable-bound multiplier. This preserves the stationarity convention \\(Px+c=A^Ty+z\\). The saved equation allows recovery even though the row has been removed from the working matrix.

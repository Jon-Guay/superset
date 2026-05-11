# Test Plan: DrillDetailPane page-size selector restricted to 50

## What changed
Single-line addition in `superset-frontend/src/components/Chart/DrillDetail/DrillDetailPane.tsx`:

```diff
  defaultPageSize={PAGE_SIZE}
+ pageSizeOptions={[String(PAGE_SIZE)]}
```

`DrillDetailPane` fetches drill-to-details rows from the server in fixed pages of `PAGE_SIZE = 50`. Before the fix, the `Table` component fell back to its default `pageSizeOptions = ['5', '15', '25', '50', '100']`, so the rendered pagination footer offered five options — letting the user pick 5/15/25, navigate past the real last page, or pick 100 and see only half the page.

After the fix, the page-size dropdown should only contain `50`.

## Primary test (Jest + React Testing Library, against the real component)

Render the modified `DrillDetailPane` with a mocked `/datasource/samples` response of `total_count: 51` (>PAGE_SIZE, so pagination renders) and assert on the page-size selector's option list.

Path: `superset-frontend/src/components/Chart/DrillDetail/DrillDetailPane.test.tsx`

### Steps
1. Mock `/datasource/samples` (the `SAMPLES_ENDPOINT` already in the test file) to return `total_count: 51` with 51 row stubs.
2. Render `<DrillDetailPane …>` via the existing `setup` helper.
3. Wait for the table to load (`screen.findByRole('table')`).
4. Click the page-size selector inside the pagination footer (antd renders `.ant-pagination-options-size-changer` → a `combobox` with text `"50 / page"`).
5. Read the rendered option list (antd renders options as `[role="option"]` inside the popup).

### Assertions (exact expected values)
- **PASS condition 1:** The page-size selector trigger text is exactly `"50 / page"` after the table loads (matches `defaultPageSize=50`).
- **PASS condition 2:** After opening the dropdown, `screen.getAllByRole('option')` returns **exactly one** option whose accessible name is `"50 / page"`.
- **FAIL condition (broken implementation):** Without the fix, opening the dropdown returns five options: `"5 / page"`, `"15 / page"`, `"25 / page"`, `"50 / page"`, `"100 / page"`. The single-option assertion would fail with `expected length 1, received length 5`.

This is adversarial: a broken implementation produces a visibly different option list (5 options vs 1), which the assertion distinguishes deterministically.

## Regression check
Run the full existing test file (5 existing tests for `DrillDetailPane`) to confirm nothing else broke:

```
npx jest --config jest.config.js src/components/Chart/DrillDetail/DrillDetailPane.test.tsx
```

Expected: all 5 existing tests + 1 new test pass.

## What I am NOT testing and why
- Full end-to-end UI walkthrough via docker-compose / running Superset: deemed disproportionate for a single-prop change to a Table component. The Jest test mounts the exact same `DrillDetailPane` component, with the same antd `Table` and `Pagination` internals, and the same DOM the user would see — the option list is the only user-visible artifact of the change, and the Jest test inspects it directly.
- CSV/XLSX download, filters, metadata bar — unchanged by the diff.

## Evidence to capture
- Jest output showing the new test passing.
- Jest output showing all 5 prior tests still passing.
- (Bonus) Temporarily revert the one-line fix and re-run the new test to demonstrate it fails with the broken option list, confirming the test is genuinely adversarial.

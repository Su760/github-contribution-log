# Contribution #1: histogram() uses base-2 math for qudit measurements

**Contribution Number:** 1
**Student:** Supash Ramesha
**Issue:** https://github.com/quantumlib/Cirq/issues/5926
**Pull Request:** https://github.com/quantumlib/Cirq/pull/8138 (merged)
**Status:** Phase IV Complete — PR merged into quantumlib/Cirq main

---

## Why I Chose This Issue

I picked issue #5926 because it was a clear, reproducible bug in a
well-maintained Python library, and it touched measurement output — an area I
felt comfortable reasoning about with my Python background. The fix came down
to a default function that assumed base-2 arithmetic and needed to handle
arbitrary qudit dimensions, which felt like a bounded, well-scoped change for a
first contribution.

I was also curious how scientific computing libraries deal with generalized
data types. The maintainers had left helpful context in the comments pointing
at `fold_func` as the root cause, which gave me a clear starting point.

---

## Understanding the Issue

### Problem Description
The `histogram()` function defaults to treating measurement results as binary
(base-2) numbers. When used with qudits (e.g. qutrits with dimension=3), it
still applies base-2 math and produces incorrect output.

### Expected Behavior
For a qutrit measurement of `[2, 2, 2]`, the histogram result should be `26`
(2×9 + 2×3 + 2 in base 3).

### Current Behavior
The function returned `7` — it read `[2, 2, 2]` and computed `111` in binary,
which is 7.

### Affected Components
The `histogram()` function and its default `fold_func` parameter in
`cirq-core/cirq/study/result.py`.

---

## Reproduction Process

### Environment Setup
- macOS, Python 3.13, conda base environment
- Cloned fork: `git clone https://github.com/Su760/Cirq.git`
- Installed in editable mode: `pip install -e ./cirq-core`
- Hit a pytest error about an unrecognized `--benchmark-disable` flag — fixed
  with `pip install pytest-benchmark`
- Tests live in `cirq/study/`, not `cirq/result/` — found via `ls cirq/`

**Working branch:** https://github.com/Su760/Cirq/tree/fix-issue-5926

### Steps to Reproduce
1. Clone the repo and run `pip install -e ./cirq-core`
2. Build a `ResultDict` with a qutrit measurement of `[2, 2, 2]`
3. Call `result.histogram(key='q')` with no extra arguments
4. **Expected:** `Counter({26: 1})` (base-3: 2×9 + 2×3 + 2)
5. **Actual:** `Counter({7: 1})` (base-2: 111 = 7)

### Reproduction Evidence
- Running my `test_bug.py` consistently returned `Counter({7: 1})` before the fix
- Confirmed reproducible multiple times

---

## Solution

### Analysis
`histogram()` had a hardcoded default `fold_func=value.big_endian_bits_to_int`,
which always assumes base-2. The `Result` object stores no qudit dimension
info, so the fix had to expose the base as a caller-supplied parameter rather
than trying to auto-detect it from the data.

### What I Implemented
I added an optional `fold_base` parameter to `histogram()`. When provided, the
function converts the measurement digits using
`value.big_endian_digits_to_int(digits, base=fold_base)` instead of the base-2
default. When it's not provided, behavior is unchanged — fully backward
compatible.

`fold_base` accepts either a single `int` or an iterable of ints, matching what
`big_endian_digits_to_int` already takes for its `base` argument. That means
for same-dimension qudits you can just pass `fold_base=3` instead of repeating
the shape like `(3, 3, 3)`.

I also added a `ValueError` when both `fold_func` and `fold_base` are passed,
since they're mutually exclusive and silently ignoring one would be confusing.

**Files changed:**
- `cirq-core/cirq/study/result.py`
- `cirq-core/cirq/study/result_test.py`

---

## Testing

### Unit Tests
- `test_histogram_qudits()` verifies `fold_base=3` returns `Counter({26: 1})`
  for a `[2, 2, 2]` measurement
- Added a `pytest.raises(ValueError)` case for passing both `fold_func` and
  `fold_base` at the same time
- A second test was added during review covering `fold_base` set to a tuple
- All existing tests in `cirq/study/` continue to pass

### Manual Testing
- `python test_bug.py` before the fix: `Got: 7`
- `python test_bug.py` after, with `fold_base`: `Got: 26`

---

## Review & Merge

The PR went through real back-and-forth with the maintainers before merging.
The reviewer (pavoljuhas) requested a few changes:

- Rename the new argument from my original `qid_shape` to `fold_base`, with the
  same accepted values as `big_endian_digits_to_int` (so a single int works for
  uniform qudits)
- Fix the docstring indentation and reword it to make clear `fold_base` is a
  convenience shorthand for `fold_func`
- Drop an assertion I'd written that checked the old base-2 result for non-bit
  values, since asserting that broken behavior as "correct" didn't make sense
- Rewrite the PR description in my own words without the AI-style checklist

I made each of those changes myself and replied to the comments. After that the
reviewer approved, pushed two small tweak commits of his own, and the PR was
merged via the merge queue (commit `2fa26d0`) with all checks passing.

---

## Learnings & Reflections

### Technical
- Navigating a large unfamiliar codebase with targeted search instead of trying
  to read everything
- How base conversion generalizes from bits to arbitrary qudit bases
- Writing a backward-compatible API change with an optional parameter

### CI was the hardest part
- Cirq's CI failures chain: fixing the formatting check exposed a coverage gap,
  and vice versa. I learned to find the actual root cause before reacting.
- The incremental coverage check requires a test for *every* new line — even a
  short `ValueError` block needs its own `pytest.raises` test, which is what
  failed coverage the first time around.
- GitHub's fail-fast matrix cancels sibling jobs when one fails, so a wall of
  "canceled" jobs doesn't mean a wall of new failures — it's usually one real
  failure plus a lot of noise.

### Git
- Clicking "Update branch" on GitHub creates a remote-only merge commit, which
  diverges from local. Reconciling it meant a `git pull --no-rebase` and
  accepting the merge commit.

### What I'd do differently
- Read the full test file and naming conventions before writing new tests
- Write the PR description and comments in my own voice from the start —
  reviewer feedback made it clear that AI-sounding text draws attention

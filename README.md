# Contribution #1: Histogram function doesn't fail for non-bool values

**Contribution Number:** 1
**Student:** Supash Ramesha
**Issue:** https://github.com/quantumlib/Cirq/issues/5926
**Status:** Phase III Complete

---

## Why I Chose This Issue

I chose issue #5926 because it's a clear, reproducible bug in a 
well-maintained Python library that directly involves data output and 
measurement results — an area I'm comfortable reasoning about with my 
Python background. The fix involves modifying a default function that 
assumes base-2 arithmetic, updating it to handle arbitrary qudit 
dimensions, which is a bounded and well-scoped change.

I'm also interested in learning how scientific computing libraries handle 
generalized data types. The maintainers have left helpful context in the 
comments pointing toward the fold_func as the root cause, which gives me 
a clear starting point heading into Phase II.

---

## Understanding the Issue

### Problem Description
The `histogram()` function defaults to treating measurement results as 
binary (base-2) numbers. When used with qudits (e.g. qutrits with 
dimension=3), it still applies base-2 math, producing incorrect output.

### Expected Behavior
For a qutrit circuit where each qudit measures value `2`, the histogram 
result should be `26` (calculated as 2×9 + 2×3 + 2 in base 3).

### Current Behavior
The function returns `7` — it reads the measurement values `[2, 2, 2]` 
and computes `111` in binary, which equals 7.

### Affected Components
The `histogram()` function and its default `fold_func` parameter in 
`cirq-core/cirq/study/result.py` (line ~216).

---

## Reproduction Process

### Environment Setup
- macOS, Python 3.13.11, using conda base environment
- Cloned fork: `git clone https://github.com/Su760/Cirq.git`
- Installed in editable mode: `pip install -e ./cirq-core`
- Hit pytest error `--benchmark-disable` unrecognized — fixed with 
  `pip install pytest-benchmark`
- Tests live in `cirq/study/` not `cirq/result/` — discovered via `ls cirq/`

**Working branch:** https://github.com/Su760/Cirq/tree/fix-issue-5926

### Steps to Reproduce
1. Clone the repo and run `pip install -e ./cirq-core`
2. Create a Python file with a 3-qutrit circuit using `cirq.LineQid(n, dimension=3)`
3. Apply `QutritPlusGate` twice to each qutrit (setting each to state `2`)
4. Run `sim.run(circuit, repetitions=1)`
5. Call `samples.histogram(key='out')` — no `qid_shape` argument
6. **Expected:** `Counter({26: 1})` (base-3: 2×9 + 2×3 + 2)
7. **Actual:** `Counter({7: 1})` (base-2: binary 111 = 7)

### Reproduction Evidence
- Running `python test_bug.py` consistently returns `Counter({7: 1})`
- Bug confirmed reproducible multiple times before fix

---

## Solution Approach

### Analysis
`histogram()` (line ~216 in `result.py`) has a hardcoded default 
`fold_func=value.big_endian_bits_to_int` which always assumes base-2. 
The `Result` object stores no qudit dimension info, so the fix must 
expose dimension as a caller-supplied parameter.

### Proposed Solution
Add an optional `qid_shape: tuple[int, ...] | None = None` parameter to 
`histogram()`. When provided, use `value.big_endian_digits_to_int(digits, 
base=qid_shape)` instead of the base-2 default. When not provided, 
behavior is unchanged — fully backward compatible.

### Implementation Plan

**Understand:** `histogram()` always uses base-2 math via 
`big_endian_bits_to_int`, producing wrong results for qudits with 
dimension > 2.

**Match:** `cirq.value.big_endian_digits_to_int(digits, base)` already 
exists in the codebase and supports arbitrary bases — no new math needed.

**Plan:**
1. Add `fold_func: Callable[[tuple], T] | None = None` to signature
2. Add `qid_shape: tuple[int, ...] | None = None` to signature
3. Add if/else logic: use `big_endian_digits_to_int` with `qid_shape` 
   as base when provided, else fall back to `big_endian_bits_to_int`
4. Update docstring Args section to document `qid_shape`
5. Add `test_histogram_qudits()` to `result_test.py`

**Implement:** https://github.com/Su760/Cirq/tree/fix-issue-5926

**Review:** Checked CONTRIBUTING.md — requires type annotations (added), 
pytest tests (added), pylint compliance (no new lint issues).

**Evaluate:** 155 tests pass (154 existing + 1 new qutrit test). 
Manual `test_bug.py` now returns `Counter({np.int8(26): 1})` ✅

---

## Testing Strategy

### Unit Tests
- [x] `test_histogram_qudits()` — verifies `qid_shape=(3,3,3)` returns 
  `Counter({26: 1})` for measurement `[2,2,2]`
- [x] Backward compatibility — verifies same call without `qid_shape` 
  still returns `Counter({7: 1})`
- [x] All 154 existing tests continue to pass

### Manual Testing
- Ran `python test_bug.py` before fix: `Got: 7` ✅
- Ran `python test_bug.py` after fix with `qid_shape=(3,3,3)`: `Got: 26` ✅

---

## Implementation Notes

### Week 2 Progress
**What I built:**
- Modified `histogram()` in `cirq-core/cirq/study/result.py` to accept 
  optional `qid_shape` parameter
- Used existing `value.big_endian_digits_to_int` to handle arbitrary bases
- Added `test_histogram_qudits()` to `cirq-core/cirq/study/result_test.py`
- All 155 tests pass

**Challenges faced:**
- `Result` object stores no qudit dimension info — had to design the fix 
  as a caller-supplied parameter rather than auto-detecting from the data
- pytest wouldn't run due to missing `pytest-benchmark` dependency 
  (not in docs for Mac users) — fixed with pip install
- Folder structure differs from docs (`cirq/study/` not `cirq/result/`)

**Files modified:**
- `cirq-core/cirq/study/result.py`
- `cirq-core/cirq/study/result_test.py`

---

## Pull Request

**PR Link:** [To be added in Phase IV]
**Status:** Phase III Complete — ready to submit PR

---

## Learnings & Reflections

### Technical Skills Gained
- Navigating a large, unfamiliar Python codebase using targeted search
- Understanding how base conversion works for generalized qudit systems
- Writing backward-compatible API changes with optional parameters

### Challenges Overcome
- Discovering the `Result` object has no dimension info required 
  rethinking the fix design entirely
- Mac-specific setup issues not covered in Linux-targeted docs

### What I'd Do Differently Next Time
- Read the full test file before writing new tests to match conventions
  more closely from the start

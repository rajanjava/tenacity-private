# Issue Resolution: Zero Exponential Multiplier Overflows into Maximum Wait

## 1. Upstream Metadata
- **Repository:** `jd/tenacity`
- **Issue Reference:** [#710](https://github.com/jd/tenacity/issues/710)
- **Title:** `Zero exponential multiplier can overflow into maximum wait`
- **Component:** `tenacity/wait.py` (`wait_exponential`, `wait_exponential_jitter`)

---

## 2. Problem Description & Root Cause
When an exponential backoff strategy (`wait_exponential`, `wait_random_exponential`, or `wait_exponential_jitter`) was configured with `multiplier=0` (or `multiplier=0.0`), after enough attempts (e.g. attempt 1025), the strategy unexpectedly returned `self.max` (e.g. 60.0 seconds) instead of 0 or the configured minimum wait:
```python
state = SimpleNamespace(attempt_number=1025)
wait_exponential(multiplier=0.0, max=60)(state)  # returned 60.0 instead of 0.0
```

### Root Cause:
1. In `wait_exponential.__call__`, the log-based threshold check `if self.multiplier > 0: ...` was bypassed when `self.multiplier == 0`.
2. Execution proceeded directly to `exp = self.exp_base ** exponent`.
3. At large attempt numbers (e.g., exponent = 1024), either `self.exp_base ** exponent` overflowed float representation (when `exp_base` was a float), or `self.multiplier * exp` overflowed when converting the large integer to float (`OverflowError: int too large to convert to float`).
4. Both paths fell into `except OverflowError:` which returned `self.max`.
5. The same problem occurred in `wait_exponential_jitter`, which also computed `self.multiplier * exp` and fell into `except OverflowError: result = self.max`.

---

## 3. Reproduction Test
Reproduction tests `test_exponential_with_zero_multiplier` and `test_wait_exponential_jitter_zero_multiplier` were added to `tests/test_tenacity.py`:
```python
def test_exponential_with_zero_multiplier(self) -> None:
    r = Retrying(wait=tenacity.wait_exponential(multiplier=0.0, max=60))
    self.assertEqual(r.wait(make_retry_state(1025, 0)), 0.0)
    r_int = Retrying(wait=tenacity.wait_exponential(multiplier=0, exp_base=2.0, max=60))
    self.assertEqual(r_int.wait(make_retry_state(1025, 0)), 0.0)
    r_min = Retrying(wait=tenacity.wait_exponential(multiplier=0, min=5, max=60))
    self.assertEqual(r_min.wait(make_retry_state(1025, 0)), 5.0)

def test_wait_exponential_jitter_zero_multiplier(self) -> None:
    fn = tenacity.wait_exponential_jitter(multiplier=0.0, max=60, jitter=0)
    self.assertEqual(fn(make_retry_state(1025, 0)), 0.0)
    fn_min = tenacity.wait_exponential_jitter(multiplier=0.0, min=5, max=60, jitter=0)
    self.assertEqual(fn_min(make_retry_state(1025, 0)), 5.0)
```
Prior to the fix, both tests failed with `AssertionError: 60.0 != 0.0`.

---

## 4. Code Fix
In `tenacity/wait.py`:
1. In `wait_exponential.__call__`, added an explicit check:
   If `self.multiplier == 0`, set `result = 0.0` directly without computing exponential powers or triggering float conversion overflow.
2. In `wait_exponential_jitter.__call__`, added an explicit check:
   If `self.multiplier == 0`, set `result = jitter` directly without computing exponential powers.
3. Allowed subsequent `max(max(0, self.min), min(result, self.max))` logic to apply configured bounds (`min` and `max`) cleanly.

---

## 5. Verification Results
- **Reproduction Test:** `2 passed, 142 deselected in 0.15s` (`tests/test_tenacity.py -k "...zero_multiplier"`).
- **Tenacity Test Suite:** `179 passed, 1 skipped in 3.63s` (`tests/test_tenacity.py`, `tests/test_asyncio.py`, `tests/test_tornado.py`).
- **Linter Check:** `ruff check` on modified source passed with zero errors.
- **Repository Commit:** Pushed cleanly to `rajanjava/tenacity-private` on branch `main`.

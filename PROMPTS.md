### PART-A
---------------------------------------------------------------
I’m checking the pricing logic and the test that exercises the 5-year loyalty case so I can explain the failure in context without speculating.

Searched for regex `5-year|tier 3|10% off|loyalty|years.*5|tier.*3` (`**/src/**`), 16 results

Read 

Read 

## What the stack trace is saying

This is not a Java crash; it is a failed test assertion from JUnit.

The test in `PriceEngineTest.java:74-84` says:

- For a customer with 5 years of loyalty
- the expected final price is 90.0
- but the actual final price was 95.0

In plain English: “A $100 order for a 5-year customer should get 10% off, so $90.00 is expected, but the code produced $95.00 instead.”

That means the pricing logic applied a 5% discount instead of a 10% discount.

---

## Most likely root cause

The bug is in the loyalty-tier logic inside [src/PriceEngine.java](src/PriceEngine.java#L51-L65).

The code currently does this:

- if years > 5 → 10% off
- else if years >= 3 → 5% off
- else if years >= 1 → 2% off

For a customer with exactly 5 years:

- 5 > 5 is false
- 5 >= 3 is true

So the code incorrectly treats 5-year loyalty as tier 2, not tier 3.

This is a classic off-by-one error: the condition should include 5 in the 10% tier, not exclude it.

> In short: the system is checking “more than 5” instead of “5 or more,” which is why $95.00 appears instead of $90.00.
# --------------------------------------------------------------------
## Plain-English explanation

This stack trace is a failed unit test, not a JVM crash.

The important part is:

- “AssertionFailedError”
- “expected: <90.0> but was: <95.0>”

That means the test expected the final price to be 90.00, but the program actually produced 95.00.

The message also tells us the specific scenario:

- “5-year loyalty customer should get 10% off (tier 3)”

So the test is checking this rule:

- a customer with 5 loyalty years should be in loyalty tier 3
- tier 3 should give a 10% discount
- on a $100 order, that should become $90.00

Instead, the system gave a 5% discount, which would turn $100 into $95.00.

---

## Most likely root cause

The most likely bug is an off-by-one check in the loyalty-tier logic.

From the code in `PriceEngine.java:51-65`, the logic is effectively:

- if years > 5 → 10% off
- else if years >= 3 → 5% off

That means exactly 5 years is not counted as tier 3, because 5 > 5 is false. It falls into the 5% rule instead.

So the root cause is:

- the boundary condition is wrong
- the code treats “5 years” as not qualifying for the 10% tier
- it should probably be “years >= 5” for the 10% tier

That matches the failing assertion exactly: 5-year customer gets 95 instead of 90.

#---------------------------------------------------------------------

## Plain-English explanation

This is a failed test assertion from JUnit.

The code is telling you:

- the test was checking a “5-year loyalty customer”
- it expected the final price to be 90.0
- but the actual value was 95.0

So the system applied a 5% discount instead of the required 10% discount.

The stack trace lines after that are just the call stack showing where the failure was built and thrown inside JUnit. They are not the actual business bug; they are just the framework plumbing around the failed assertion.

---

## Most likely root cause

The most likely cause is a boundary-condition bug in the loyalty discount logic.

The logic is checking the loyalty years in a way that excludes 5 exactly. In other words, it likely does something equivalent to:

- “if years > 5” for 10% off
- otherwise “if years >= 3” for 5% off

That means a customer with exactly 5 years falls into the 5% bucket instead of the 10% bucket.

That matches the failure exactly:

- expected: 100 - 10% = 90.0
- actual: 100 - 5% = 95.0

So the bug is almost certainly an off-by-one mistake in the loyalty tier comparison, not a tax issue, not a promo-code issue, and not a rounding problem.

### Part A – My Assessment

I agree with the AI's diagnosis because the 5-year loyalty customer is incorrectly receiving the 5% tier-2 discount instead of the expected 10% tier-3 discount.
The larger portions of the stack trace were useful for locating the failing test, although the first-line assertion message already provided a strong clue about the incorrect result.

# PART-B




## Part B – Propose and Verify the Patch

### AI Prompt
Given the root cause that the loyalty-tier logic uses `years > 5` for the 10% discount, so a customer with exactly 5 years incorrectly falls into the 5% tier, write the smallest patch that fixes the bug without changing behaviour on any other test. Show only the diff.

### AI Response

This patch is limited to the boundary condition for the loyalty tier.

What it changes:
Before: 5-year customers were treated as tier 2 because the condition was years > 5.
After: 5-year customers are treated as tier 3 because the condition is years >= 5.
What it does not change
Customers with 0–2 years still get no loyalty discount.
Customers with 1–2 years still get 2%.
Customers with 3–4 years still get 5%.
Customers with 5+ years now correctly get 10%.
All other promo, shipping, tax, and subtotal logic remains untouched.
So the behavioral change is effectively limited to the exact failing boundary case, which is the intended fix.

### Verification

Command:
"make test"

I ran the project test suite with `make test`.

Evidence from the output:

- 10 tests found
- 10 tests started
- 10 tests successful
- 0 tests failed

So the fix passes the existing test suite.

### 4A Reflection

The most useful AI reply was the one based on the complete stack trace because it clearly identified the failing test, the expected and actual values, and the likely boundary-condition error in `PriceEngine.java`. The least useful reply was the one based on the shortest stack-trace excerpt because it provided less context and made the root cause less certain. In this case, the complete stack trace was not strictly necessary to identify the bug, but it provided enough context to confirm the failing test and trace the failure to the loyalty-tier calculation. Overall, the progressively longer stack traces showed that one or two lines can suggest the problem, while the full trace provides stronger evidence for confirming the diagnosis.

### Refactoring Priority Ranking

| Rank | Method | Current cyclomatic complexity estimate | First refactor move | Expected complexity reduction |
|---|---|---:|---|---|
| 1 | PriceEngine.quote | about 15–18 | extract-method | Breaking the method into subtotal, promo, loyalty, and tax phases would collapse several nested condition chains into smaller, single-purpose routines and cut the effective complexity by roughly half. |

This file has one meaningful business method, and `PriceEngine.quote` is the clear refactoring priority because it combines validation, subtotal calculation, surcharge logic, promo matching, loyalty-tier branching, and tax calculation in one method. The best first move is extraction of those phases into smaller methods, since that isolates responsibilities without changing behavior.

## Session 4B — Part B: Apply Extract-Method Refactor

### AI Prompt

Extract the loyalty-tier discount block into a private
helper method ‘applyLoyaltyDiscount(...)‘. Do NOT change
public method signatures. Show the diff.

### AI Response

The AI proposed extracting the loyalty-tier discount logic from `PriceEngine.quote` into a private helper method:

```java
private static Money applyLoyaltyDiscount(Money running, Money subtotal, Customer customer)
```

The proposed change replaced the loyalty-tier block inside `quote` with:

```java
running = applyLoyaltyDiscount(running, subtotal, customer);
```

The helper retained the existing loyalty-tier behavior, including the Session 4A correction from `years > 5` to `years >= 5`.

### Review

The public `quote(...)` method signature was unchanged. The helper retained the original parameter and return types, did not introduce or remove early returns, and preserved the loyalty-tier conditions and discount calculation. No behavior outside the extracted loyalty-tier block was intentionally changed.

### Verification

The proposed diff was applied.

Command:

```text
make test
```

Result:

* 10 tests found
* 10 tests started
* 10 tests successful
* 0 tests failed

Therefore, the extract-method refactor passed the existing regression test suite.

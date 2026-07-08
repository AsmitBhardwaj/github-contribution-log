# Contribution 2: Implement String Expression Parity with PySpark (`btrim` & `char`)

**Contribution Number:** 2  
**Student:** Asmit Bhardwaj  
**Issue:** [Eventual-Inc/Daft#3792](https://github.com/Eventual-Inc/Daft/issues/3792)  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it aligns perfectly with my current learning goals of diving deeper into systems-level programming and gaining hands-on experience with Rust. Daft's push for high-performance dataframes relies heavily on an optimized Rust backend, and implementing PySpark string parity allows me to work directly with Rust's powerful string manipulation and arrow array memory management. 

Instead of jumping into a massive architectural overhaul, focusing on discrete expressions like `btrim` (both trim) and `char` (ASCII to character casting) gives me a highly defined, bounded scope. This allows me to focus entirely on learning the codebase conventions, navigating how the Python API binds to the underlying Rust kernels (likely via PyO3), and practicing writing robust, idiomatic Rust code in a production-grade framework.

---

## Understanding the Issue

### Problem Description

Daft is building out full feature parity with PySpark to make it a drop-in replacement for distributed dataframe workloads. Currently, several string expressions available in PySpark are missing from Daft's expression engine. Specifically, the `btrim` function (which trims specified characters or spaces from both ends of a string) and the `char` function (which converts an integer ASCII/Unicode code point into its matching string character) need to be natively implemented in the Rust backend and exposed to the Python frontend.

### Expected Behavior

- **`btrim(column, [characters])`**: Should return a new string column with leading and trailing characters removed. If no character set is provided, it should default to stripping whitespace.
- **`char(column)`**: Should take an integer column representing ASCII/Unicode values and return a string column containing the corresponding single-character strings.
- Both functions should seamlessly handle `null` values within data arrays without panicking.

### Current Behavior

Attempting to invoke these specific string expressions on a Daft DataFrame throws an `AttributeError` or an unimplemented error because the expressions do not yet exist in the Python expression API or the core Rust execution kernels.

### Affected Components

- **Python Frontend (`daft/expressions/`)**: The user-facing API surface needs to be updated to accept these new string methods.
- **Rust Backend (`src/daft-core/src/series/` / `src/daft-dsl/`)**: The logical expression registry needs to recognize the new expression types.
- **Rust Kernels (`src/daft-core/src/array/ops/`)**: The actual data-parallel loops that execute the string manipulations across the Arrow arrays need to be written.

---

## Solution Approach

### Analysis

The root cause is simply that these capabilities haven't been ported over yet. Because Daft is built on top of Apache Arrow, the strings are stored in contiguous memory arrays. The implementation will require writing vectorized operations in Rust that efficiently map over these string/integer arrays.

### Proposed Solution

I will implement vectorized Rust functions for both behaviors utilizing Rust's native string optimization methods (such as `.trim_matches()` or standard character casting closures). I'll register these functions as valid operations within Daft's Domain Specific Language (`DSL`) and expose them via the Python abstraction layer.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Replicate PySpark's `btrim` and `char` functions cleanly in Daft's Rust layer while matching existing API ergonomics.

**Match:** I will look at recent commits and pull requests in the repository that implemented other string parity functions (such as `overlay` or basic string slicing) to serve as a direct structural blueprint for array layout, macro usage, and PyO3 bindings.

**Plan:**
1. Locate the core string array operations module in Rust (`src/daft-core`).
2. Implement the core logic kernel for `char` using safe casting (`char::from_u32`) and array mapping.
3. Implement the core logic kernel for `btrim` using optimized string trimming iterators.
4. Expose the new kernels through the DSL kernel registry.
5. Add the corresponding Python methods to the Expression class so users can invoke them.
6. Write comprehensive unit tests in Python (`tests/`) ensuring proper execution, edge-case validation, and correct null-value propagation.

**Implement:** [In Progress - Will link branch/commits here in Phase II]

**Review:** [Pending]

**Evaluate:** [Pending]

---

## Resources Used

- [Daft Contributor Documentation](https://github.com/Eventual-Inc/Daft/blob/main/CONTRIBUTING.md)
- [PySpark String Functions Reference Doc](https://spark.apache.org/docs/latest/api/sql/index.html)


**ARCHIVE/PAST CONTRIBUTIONS**

# Contribution 1: Add Testing for Thread Safety of Driver

**Contribution Number:** 1 
**Student:** Asmit Bhardwaj 
**Issue:** https://github.com/apache/hamilton/issues/54  
**Status:** Phase II

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of two things I'm actively developing: Python testing skills and a deeper understanding of concurrency. I've written pytest test suites before in my coursework and CodePath projects, but testing thread safety requires thinking about parallel execution and race conditions in a way that goes beyond typical unit tests. That challenge is exactly what drew me to it.
Apache Hamilton is also a serious, production-grade data pipeline framework used by real engineering teams, which means contributing here carries more weight than contributing to a small hobby project. I want my first open source contribution to be something I'm genuinely proud of, and writing a concurrency test for a framework at this scale fits that goal.

---

## Understanding the Issue

### Problem Description

Hamilton's driver is expected to be thread-safe, but there are no tests that actually verify this. Without a test, there's no guarantee that running multiple executions in parallel with different input values won't cause them to interfere with each other.

### Expected Behavior

Multiple threads should be able to use the same driver instance simultaneously with different input values, and each thread should get back the correct result for its own inputs without any cross-contamination.

### Current Behavior

No test exists to verify this. Hence, the thread safety is assumed but unproven.

### Affected Components

Hamilton's core driver (driver.py) and the test suite.

---

## Reproduction Process

### Environment Setup

I cloned the fork locally using git clone. Installed the dependencies using pip3 install -e [dev]. There was no major errors.

### Steps to Reproduce

1. Clone the Apache Hamilton Repository
2. Navigate to the test/ directory
3. Search for any test file referencing thread safety or concurrent execution(grep -r "thread" tests/)
4. Observe that no thread safety test exists for the driver
5. Observed result: The driver has no test verifying parallel execution with different inputs return correct, non contaminated results

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/AsmitBhardwaj/hamilton/tree/test/thread-safety-driver
- **Screenshots/logs:** 
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

The driver currently has no thread-safety test. The root cause isn't a bug per se — it's a gap in test coverage. The risk is that shared state inside the driver could be mutated by one thread while another is reading it, producing incorrect results.

### Proposed Solution

Write a pytest test that creates a single driver instance, launches multiple threads each running an execution with different input values, and asserts that every thread receives the correct output for its own inputs.

### Implementation Plan

**Understand:** The driver has no test proving it handles concurrent executions safely. Multiple threads sharing one Driver instance could get wrong results if internal state is mutated mid execution.


**Match:** Followed patterns in tests/test_hamilton_driver.py using Driver and tests.resources.very_simple_dag as test graph


**Plan:**
1. USe very_simple_dag
2. Instantiate a single Driver from that graph
3. Spawn 10 threads each calling driver.execute() with a unique integer input value(0-9)
4. Collect results in a shared dictionary keyed by input value
5. Assert no threads raised errors
6. Assert every thread received the correct output for its own input

**Implement:** https://github.com/AsmitBhardwaj/hamilton/tree/test/thread-safety-driver

**Review:** Will follow Hamilton's pre-commit hooks and style guidelines before commiting PR

**Evaluate:** Test passes consistently across multiple runs and confirmed 1 passed with no race conditions

---

## Testing Strategy

### Unit Tests

- test_driver_thread_safety(): Spawns 10 threads sharing one Driver instance, each with unique input values (0–9), asserts every thread gets correct non-contaminated output

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

- Ran pytest locally: 1 passed in 0.73s across multiple runs with no race conditions

---

## Implementation Notes

### Week 1 Progress

elected issue, passed all 6 checklist criteria, confirmed repository is actively maintained with 11 PRs merged in the past week. Posted comment on GitHub issue expressing interest.

### Week [2] Progress

Wrote test_driver_thread_safety() in tests/test_hamilton_driver.py. Used very_simple_dag as the test graph. Debugged a pandas Series assertion issue — fixed by using .item() to extract scalar value. Ran pytest and confirmed 1 passed in 0.73s.

[Continue documenting as you work]

### Code Changes

- **Files modified:** tests/test_hamilton_driver.py
- **Key commits:** https://github.com/AsmitBhardwaj/hamilton/tree/test/thread-safety-driver
- **Approach decisions:** Modeled test structure on existing tests in test_hamilton_driver.py. Used threading.Thread to simulate concurrent driver execution.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained: Thread safety testing in Python, using threading.Thread, debugging pandas Series scalar extraction with .item()

[What you learned technically]

### Challenges Overcome: pandas assertion error where comparing a Series to an int failed — solved by calling .item() to extract the scalar

[What was hard and how you solved it]

### What I'd Do Differently Next Time :  Set up the pre-commit hooks earlier to ensure style compliance from the start

[Reflection on your process]

---

## Resources Used : tests/test_hamilton_driver.py (existing test patterns), Python threading docs


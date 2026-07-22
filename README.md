Contribution 2: Implement String Expression Parity with PySpark (btrim & char)

Contribution Number: 2
Student: Asmit Bhardwaj
Issue: Eventual-Inc/Daft#3792
Status: Phase IV Complete (PR Submitted)

🎯 Why I Chose This Issue

I chose this issue because it aligns directly with my goal of diving deeper into systems-level programming and gaining hands-on experience with Rust in a production environment. Daft's high-performance dataframe engine relies heavily on an optimized Rust backend, and implementing PySpark string parity allowed me to work directly with Rust's string manipulation primitives and Apache Arrow array memory management.
Instead of taking on a massive architectural overhaul, focusing on discrete expressions like btrim (both trim) and char (ASCII/Unicode integer-to-character casting) provided a well-defined, bounded scope. This allowed me to master the codebase conventions, navigate how the Python API binds to underlying Rust kernels via PyO3, and practice writing robust, idiomatic, vector-optimized Rust code within a production dataframe system.

🔍 Understanding the Issue

Problem Description
Daft is building out full feature parity with PySpark to serve as a drop-in replacement for distributed dataframe workloads. Currently, several string expressions available in PySpark are missing from Daft's expression engine. Specifically, the btrim function (which trims specified characters or spaces from both ends of a string) and the char function (which converts an integer ASCII/Unicode code point into its matching string character) need to be natively implemented in the Rust backend and exposed to the Python frontend.

Expected Behavior

-btrim(column, [characters]): Returns a new string column with leading and trailing characters removed. If no character set is provided, it defaults to stripping whitespace.
-char(column): Takes an integer column representing ASCII/Unicode values and returns a string column containing the corresponding single-character strings.
-Null Handling: Both functions must seamlessly handle and propagate null values within data arrays without panicking.
Current Behavior
-Attempting to invoke these specific string expressions on a Daft DataFrame throws an AttributeError or an unimplemented error because the expressions do not yet exist in the Python expression API or the core Rust execution kernels.

Affected Components

-Python Frontend (daft/expressions/): The user-facing API surface needs to be updated to accept these new string methods under .str.
-Rust Core DSL (src/daft-dsl/): The logical expression registry needs to recognize the new expression types and wire up logical planning.
-Rust Kernels (src/daft-core/src/array/ops/): The actual data-parallel, vectorized execution loops that execute the string manipulations across Apache Arrow arrays need to be written.

🧪 Reproduction Process

-Environment Setup
-Cloned the Daft repository fork locally:
-Bash
-git clone https://github.com/AsmitBhardwaj/Daft.git
-cd Daft
-Set up the Rust toolchain and Python development environment using maturin:
-Bash
-pip install -e .[dev]
-maturin develop

Steps to Reproduce

-Launch an interactive Python session (python3).
-Attempt to invoke btrim or char using the expression API:
-Python
-import daft
-from daft import col
-df = daft.from_pydict({"val": ["  hello  ", "world  "]})
-df.select(col("val").str.btrim())
-Observe the AttributeError: 'Expression' object has no attribute 'btrim'.

Reproduction Evidence

-Confirmed missing expression bindings in daft/expressions/string.py and missing kernel handlers in daft-core.
-🛠️ Solution Approach

Analysis

-The root cause was simply that these expressions had not yet been ported over to Daft's expression engine. Because Daft stores strings in contiguous Apache Arrow memory arrays (Utf8Array / Int64Array), the implementation required writing vectorized operations in Rust that efficiently map over these arrays, handling null bitmasks without overhead or panics.

Proposed Solution

-Implement vectorized Rust functions for both operations utilizing Rust's native string optimization methods (.trim_matches() and safe char::from_u32 casting). 
-Register these functions within Daft's Domain Specific Language (DSL) and expose them to Python via PyO3 bindings on the Expression class.

📋 Implementation Plan (UMPIRE Framework)

-Understand: Replicate PySpark's exact semantics for btrim and char within Daft's Rust layer while matching existing API ergonomics.
-Match: Evaluated recent commits and pull requests implementing other string parity functions (such as trim, substr, or overlay) as structural blueprints for array layouts, PyO3 bindings, and macro registrations.
-Plan:
  -Locate the core string array operations module in src/daft-core/src/array/ops/.
  -Implement the core logic kernel for char using safe casting (char::from_u32) and array mapping.
  -Implement the core logic kernel for btrim using optimized string trimming iterators.
  -Expose the new kernels through the DSL kernel registry (src/daft-dsl/).
  -Add corresponding Python methods (.btrim() and .char()) to the Expression string namespace.
  -Write comprehensive unit tests in Python (tests/) ensuring proper execution, edge-case validation, and correct null propagation.
-Implement: Written, built, and tested locally.
-Review: Verified clean builds with cargo check, passed cargo test, and validated code formatting with ruff and cargo fmt.
-Evaluate: Verified full PySpark behavioral parity and zero performance regressions on large synthetic string series.

🧪 Testing Strategy

-Unit Tests
  -Written in tests/expressions/typing/test_str.py and tests/series/test_str.py:
  -test_btrim(): Tests standard whitespace trimming, trimming custom character sets (e.g., trimming xyz from xyzhellozx), empty strings,       and arrays containing None/null values.
  -test_char(): Tests standard ASCII conversions (e.g., 65 -> 'A'), Unicode code points, boundary/invalid integer values, and propagation     of null inputs.

Manual Testing
  -Ran pytest locally across the entire string expression test suite:
  -Bash
  -pytest tests/expressions/typing/test_str.py
  -pytest tests/series/test_str.py
  -Result: Passed consistently across all runs.

📝 Implementation Notes & Progress

Development Milestones
-Phase I: Investigated Daft's expression engine architecture, PyO3 bindings, and Apache Arrow kernel layouts.
-Phase II: Implemented char and btrim vectorized kernels in daft-core. Exposed kernel methods through daft-dsl and registered PyO3 bindings for Python.
-Phase III: Added user-facing Python API bindings under col().str.btrim() and col().str.char(). Wrote complete test coverage in Python test suite.
-Phase IV: Executed all local test suites, ran linter checks (cargo fmt, clippy, ruff), and finalized the pull request submission.

Code Changes
-Files Modified:
-src/daft-core/src/array/ops/utf8.rs (Kernel logic)
-src/daft-dsl/src/expr/utf8.rs (DSL expression definitions)
-daft/expressions/string.py (Python frontend bindings)
-tests/series/test_str.py (Pytest coverage)

💡 Learnings & Reflections

-Technical Skills Gained:
  -Advanced Rust systems programming: working with low-level Apache Arrow memory layouts and null-bitmap handling.
  -Deepened understanding of PyO3 bindings connecting Rust core execution engines to user-facing Python DSLs.
  -Vectorized operations in Rust using iterators and safe character casting closures.

-Challenges Overcome:
  -Handling edge-case Unicode integer code points in char: Direct casting using as char in Rust can cause undefined behavior or panics for     out-of-range integers. Solved by utilizing char::from_u32 inside a safe mapping closure over the Arrow array, returning null or handling     errors gracefully for invalid code points.

-What I'd Do Differently Next Time:
  -Set up maturin develop and local Rust compilation caching (sccache) right from day one to speed up rebuild times when tweaking kernel       code.

  -Resources Used:
  -Daft Contributor Guidelines & Architecture Docs
  -Apache Arrow Rust Crate Documentation (arrow::array)
  -PySpark btrim & char SQL Reference Documentation


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


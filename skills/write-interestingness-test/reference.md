# Reference: Test-Case Reduction Background

## Theory

Test-case reduction is fundamentally **greedy optimization over a search space**. Given a test case that exhibits some property (the "interestingness" property), the reducer systematically tries smaller/simpler variants, keeping any variant that still satisfies the interestingness test.

### Key Concepts

**Interestingness test**: An oracle that decides whether a candidate test case exhibits the target property. The quality of this oracle is the single most important factor in reduction quality.

**Reduction order**: A total ordering over test cases that defines what "simpler" means. Shorter is always better. Among equal-length candidates, simpler content is preferred (shrinkray uses a natural ordering for text, shortlex for binary).

**Fixed point**: Reduction terminates when no single pass can make further progress. The result is a local minimum — it may not be globally minimal, but every individual transformation the reducer tries makes it no longer interesting.

**Slippage**: When reduction transforms a test detecting subtle fault F into a test detecting a different, often trivial and known fault. The primary defense is a precise interestingness test.

**The Sorcerer's Apprentice Problem** (Regehr): Reducers follow instructions with unwavering literalism. Any loophole in the interestingness test WILL be exploited. A test that accepts empty files will produce an empty file. A test that only checks for "error" will find the simplest possible error.

**Reducers are fuzzers** (Regehr): The reduction process explores a vast space of mutated inputs. It commonly discovers new bugs unrelated to the original target. This is both useful (bug discovery) and problematic (slippage).

### Normalization

Normalization is the idea that for any interestingness test, there should ideally be a single canonical minimal result regardless of the reduction path taken. shrinkray achieves this through its natural ordering (a multi-tier heuristic: total length → average squared line length → line count → line length sequence → natural character order). This means whitespace < digits < lowercase < uppercase, so the reducer actively prefers simpler character choices, not just shorter strings.

### Mutual Perturbation

Running each reduction pass individually to fixation is often *less* efficient than running passes once and allowing them to mutually perturb each other. Pass A may get stuck at a local minimum, but pass B's changes may unstick it. This is why multi-pass reducers cycle through passes repeatedly rather than running each to completion.

### Academic Foundations

- **Delta Debugging** (Zeller & Hildebrandt, 2002): The foundational algorithm. Binary search over subsets of input — try removing halves, then quarters, then eighths, down to individual elements. Complexity: O(m × log(n)) tests for input size n and minimal result size m.
- **Hierarchical Delta Debugging (HDD)** (Misherghi & Su): Extends delta debugging to tree-structured inputs using grammars. Prunes subtrees at each level. Extensions include "HDD with Hoisting" which replaces subtrees with compatible subtrees further down the hierarchy.
- **Perses** (Sun et al., ICSE 2018): Syntax-guided reduction using ANTLR grammars. Ensures each candidate is syntactically valid. Achieves 2% of DD's output size and 23% of DD's time.
- **Test-Case Reduction via Test-Case Generation** (MacIver & Donaldson, ECOOP 2020): Reduces the choice sequence (PRNG samples) rather than the generated output, preserving generation invariants.
- **Adaptive Delta Debugging** (MacIver, 2017): Instead of choosing block size globally, choose locally per index — find a large deletable block starting from the current position. Typically ~75% as many calls as standard ddmin.
- **Cause Reduction** (Groce et al., 2016): Extends delta debugging to simplify test cases with respect to arbitrary effects beyond crashes (e.g., code coverage, execution behavior).
- **Mitigating Test Reduction Slippage** (Groce et al., 2016): Formal treatment of slippage — can be mitigated by more precise oracles, and exploited by outputting a set of reduced tests to capture multiple faults.

## Tool Comparison

### shrinkray
- **Best for**: Any file format, especially text-like. Highly parallel. Best generic reducer available.
- **Input modes**: stdin + file argument + basename (all three by default, configurable with `--input-type`)
- **Parallelism**: Highly parallel by default (all cores). Uses innovative "merge master" pattern for parallel patch application.
- **Format support**: Generic (any file), plus specialized passes for Python, JSON, C/C++ (via clang_delta), DIMACS CNF.
- **Ordering**: Natural ordering for text (length, line balance, character simplicity), shortlex for binary.
- **Unique features**: Reduction pumps (temporarily increase size for deeper reduction), four-tier pass organization, TUI, history recording, auto-timeout calibration.
- **Install**: `pipx install shrinkray`

### creduce / cvise
- **Best for**: C/C++ specifically (110+ Clang-based passes). Also works on other languages with `--not-c`.
- **Input mode**: File in CWD with original basename. No arguments to script.
- **Parallelism**: Speculative parallelism (2-3x speedup typical). cvise defaults to all cores.
- **Unique features**: AST-aware C/C++ transformations (function removal, type simplification, inlining). cvise is the more maintained Python port.
- **Install**: Package manager (`apt install creduce` or `brew install creduce`)

### lithium
- **Best for**: Line-based reduction of text files. Simple and reliable.
- **Input mode**: File path passed as argument.
- **Algorithm**: Modified ddmin on lines.

### Other tools
- **halfempty** (Google Project Zero): Parallel binary reducer using pessimistic speculative execution. Good for binary formats.
- **picire/picireny**: Python delta debugging with HDD support using ANTLR grammars.
- **treereduce**: Syntax-aware reducer using tree-sitter grammars. Fast, written in Rust.
- **afl-tmin**: Minimizer for AFL/AFL++ fuzzer findings. Uses coverage instrumentation.

## Interestingness Test Patterns by Bug Type

### Compiler/Tool Crash (ICE)
- Easiest to write: just check for the specific crash message
- Usually no need for validity checks (crash on invalid input is still a bug)
- Match the internal error location (function name), not line numbers in the test case
- Be careful not to match line numbers or file paths from the test case itself

### Wrong-Code (Miscompilation)
- Hardest to write correctly due to undefined behavior
- MUST include UB protection (warnings-as-errors, sanitizers, multi-compiler checks)
- Differential test: compile at different optimization levels, compare outputs
- Consider using Frama-C, tis-interpreter, or kcc for heavyweight UB detection

### Hang/Timeout
- Test for the timeout itself: `timeout N tool file; test $? -eq 124`
- Be careful: reduction may introduce legitimate infinite loops
- Set a tight timeout (just above the expected runtime)

### Wrong Error Message / Warning
- Match the specific diagnostic text
- Don't use `-Werror` (reduction introduces new warnings)
- Filter out known false-positive warnings

### Performance Regression
- Measure execution time and check against a threshold
- Non-deterministic by nature — consider averaging multiple runs
- Use CPU time (`time -p`) rather than wall-clock time

### Differential Testing
- Run the same input through two implementations (or two versions)
- Compare outputs; interesting if they differ
- Must guard against both implementations rejecting the input

## How shrinkray Works (Relevant to Test Writing)

Understanding how shrinkray reduces files helps write better interestingness tests.

### Pass Tiers

shrinkray organizes reduction into four tiers, running cheaper high-value passes first:

1. **initial_cuts**: Fast passes with timeout-based cancellation — comment removal, hollowing brackets (`{...}` → `{}`), large block deletion, whitespace removal. If a pass stops making progress for 5 seconds, it's cancelled.
2. **great_passes**: Core reduction loop — line deletion, token deletion, bracket lifting, debracketing. Loops until no pass makes progress.
3. **ok_passes**: Run when great_passes plateau — smaller block deletions, integer literal reduction, identifier normalization, expression combination.
4. **last_ditch_passes**: Expensive or low-yield — individual byte lowering, bracket simplification, character substitutions.

Before any passes run, shrinkray checks if trivial inputs (empty string, `0`, `1`, `\n`, `z`, etc.) are interesting. If so, it terminates immediately — this catches overly-permissive interestingness tests fast.

### Parallelism

shrinkray tests multiple candidates concurrently using a "merge master" pattern. When multiple patches pass independently, it tries applying them all at once, using binary search to find the maximum compatible set. This means:

- **Your test must be safe to run in parallel.** Don't use fixed filenames in `/tmp`, don't rely on global state.
- **Side-effect-free tests are ideal.** If your test modifies shared state, use `--parallelism=1`.
- **The test may be called with very different candidates simultaneously.** Don't assume a particular order of candidates.

### Timeout Behavior

If `--timeout` is not set, shrinkray runs the interestingness test once on the original file, measures the time, and sets the timeout to 10× that (capped at 5 minutes, minimum 1 second). Tests that exceed the timeout are treated as non-interesting (exit non-zero).

This means: if your test is slow on the original file, the auto-calibrated timeout will be generous, slowing down the entire reduction. Consider optimizing your test or setting an explicit `--timeout`.

### Format Detection

shrinkray auto-detects formats and applies specialized passes:
- **Python**: Detected by trying to parse with libcst. Enables AST-aware reductions.
- **JSON**: Detected by `json.loads`. Enables key deletion passes.
- **C/C++**: Detected by file extension. Enables clang_delta passes if creduce is installed.
- **DIMACS CNF**: Detected by parsing format. Enables SAT-specific passes.

The generic byte-level and text passes always run regardless of format.

### History

With `--history` (default on), shrinkray saves all intermediate reductions to a `.shrinkray/` directory. This is useful for debugging interestingness tests — you can see what candidates were accepted and trace how the reduction progressed.

## Preprocessing Tips

### C/C++
- **Preprocess first**: `gcc -E file.c > file.i` eliminates header dependencies, giving the reducer more freedom
- **Remove #include guards**: Already handled by preprocessing
- **Use creduce's topformflat**: Flattens code to one-top-level-form-per-line for better line-based reduction

### Multi-file test cases
- **Consolidate**: If possible, merge files into one (e.g., concatenate C files, inline imports)
- **shrinkray directory mode**: Can reduce entire directories, deleting files and then reducing individual contents
- **creduce**: Supports multiple files as arguments

### Large files
- **Manual pre-reduction**: Remove obviously irrelevant sections by hand before running the reducer
- **Use a fast reducer first**: Run a quick line-based reducer (lithium) before a thorough one (shrinkray, creduce)

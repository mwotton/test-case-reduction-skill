---
name: debug-reduction
description: Diagnose problems with shrinkray test-case reduction — unexpected results, reduction getting stuck, slippage to wrong bugs, or interestingness test issues. Use when reduction output is surprising or reduction is not making progress.
user-invocable: true
argument-hint: [description of the problem with reduction]
---

# Debugging Test-Case Reduction Problems

You are helping the user diagnose and fix problems with their test-case reduction. Something has gone wrong — the reduced output is unexpected, reduction is stuck, or the interestingness test isn't working properly.

## Common Problems and Diagnosis

### 1. Reduced Output Triggers a Different Bug (Slippage)

**Symptoms**:
- The reduced test case looks nothing like expected
- The error message in the reduced output differs from the original
- The reduced test case is suspiciously small or degenerate

**Diagnosis**: The interestingness test is too broad. The reducer found a simpler input that satisfies the test but exercises a different code path.

**Fixes**:
- Match a more specific error message (include function names, error codes, source locations from the buggy tool's internal code — NOT from the test case being reduced)
- Add validity checks to reject degenerate inputs
- For C/C++: add UB checks, compile with warnings-as-errors
- If the original bug produces a stack trace, match distinctive frames from the trace

### 2. Reduced Output Is Trivially Small (Empty or Near-Empty)

**Symptoms**:
- Result is 0-1 bytes
- shrinkray warns about trivial results
- The "reduced" test case is meaningless

**Diagnosis**: The interestingness test accepts inputs that don't genuinely exhibit the bug. Often: empty input causes the tool to error, and the test matches any error.

**Fixes**:
- Add `test -s "$1" || exit 1` to reject empty files
- Check for the **specific** error, not just any non-zero exit code
- Add a "not bogus" check: verify the input is valid according to some reference tool before checking for the bug

### 3. Reduction Gets Stuck (No Progress)

**Symptoms**:
- File size plateaus early and won't decrease further
- The reducer runs for a long time without improvement
- Result is much larger than expected

**Diagnosis possibilities**:
- **Interestingness test is too restrictive**: Over-constraining with grep checks prevents the reducer from removing code
- **Format-specific constructs**: Some file structures resist generic byte-level reduction
- **Interdependent code**: Removing any single piece breaks the bug, but removing multiple pieces together would work

**Fixes**:
- Remove unnecessary grep constraints from the interestingness test
- For shrinkray: the pass tiers handle this progressively — ensure reduction ran to completion
- Try manual reduction: look at the stuck result and try removing something by hand. If that works but the reducer couldn't do it, file a bug
- For C/C++: preprocess the file first (`gcc -E file.c > file.i`) to eliminate header dependencies
- Try a different reducer or combine reducers (run creduce then shrinkray, or vice versa)

### 4. Interestingness Test Is Flaky

**Symptoms**:
- shrinkray warns about inconsistent test results
- Reduction makes progress but then seems to "undo" improvements
- Different runs produce very different results

**Diagnosis**: The interestingness test gives different results on the same input. Common causes:
- Timing-dependent bugs (race conditions)
- ASLR affecting memory addresses
- Tool under test has internal non-determinism
- Test depends on external state (network, temp files from previous run)

**Fixes**:
- Run the test multiple times on the same input to confirm flakiness
- Disable ASLR: `setarch $(uname -m) -R ./test.sh file.c`
- Pin random seeds if the tool supports it
- Clean up temp files at the start of each test invocation
- Use `--parallelism=1` with shrinkray to eliminate parallel interference
- For race conditions: add `sleep` or retry logic (but this slows reduction significantly)

### 5. Reduction Is Too Slow

**Symptoms**:
- Each test invocation takes many seconds
- Overall reduction takes hours or days
- CPU usage is low (waiting on I/O or timeouts)

**Diagnosis**: The interestingness test is slow, or timeouts are too generous.

**Fixes**:
- **Reorder checks**: Put the cheapest, most-likely-to-fail check first
- **Reduce timeout**: If the original test takes 1 second, a 5-second timeout is fine. Don't use the default 300s from creduce
- **Speed up compilation**: Use `-S` instead of `-c`, `-Wfatal-errors`, `-w` (suppress warnings when they don't matter)
- **Avoid unnecessary work**: Don't link if you only need to compile. Don't run if you only need to compile.
- **Use shrinkray's parallelism**: Ensure `--parallelism` is set to your core count (default)
- **Profile the test**: `time ./test.sh file.c` to see where time is spent

### 6. Reduced Output Has Undefined Behavior (C/C++)

**Symptoms**:
- Reduced test case compiles but behavior depends on compiler/flags
- Different compilers give different results on the reduced code
- AddressSanitizer or UBSan flag issues in the reduced code

**Diagnosis**: The reducer introduced UB during reduction. This is expected and nearly inevitable for C/C++ miscompilation bugs if the interestingness test doesn't guard against it.

**Fixes**:
- Add UB sanitizer checks to the interestingness test: `-fsanitize=undefined`
- Compile with strict warnings: `-Wall -Wextra -pedantic -Werror`
- Use both GCC and Clang for cross-checking
- Consider using heavyweight tools: Frama-C, tis-interpreter, or kcc/RV-Match
- Note: for crash bugs (ICE), UB in the test case is usually fine — the bug is that the compiler crashes, not that the code is wrong

### 7. Test Works Manually But Not Under the Reducer

**Symptoms**:
- `./test.sh file.c && echo interesting` works fine manually
- But shrinkray says the initial test case is not interesting

**Diagnosis**: This is almost always a temporary directory problem. shrinkray runs the test in a fresh temp directory, not your working directory. Anything that depends on your local environment will break.

**Fixes**:
- **Use `$1` for the test case**: The file argument is an absolute path and works from any directory. If the test references the file by a hardcoded relative path or basename, switch to `$1`.
- **Use absolute paths for everything else**: Tools, helper scripts, auxiliary data files, reference implementations — anything that isn't the test case itself must be an absolute path or resolved via `SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"`.
- **Check PATH**: Tools you have on PATH in your shell may not be on PATH in the reducer's environment. Use full paths to be safe.
- **Missing dependencies**: If your test script sources other files or uses tools not in the default PATH, they won't be available in the temp directory.
- **Permissions**: Ensure the test is executable (`chmod +x test.sh`).

**Quick diagnostic**: Run the test from a different directory to simulate the reducer's environment:
```bash
cd /tmp && /absolute/path/to/test.sh /absolute/path/to/original_file
```
If this fails but running from your project directory succeeds, the test has a directory dependency.

## Diagnostic Steps

When the user reports a problem, work through these steps:

1. **Read the interestingness test** — most problems originate here
2. **Check what reducer and options are being used** — especially input type, timeout, parallelism
3. **Test the interestingness test manually**:
   - Run it on the original file (should exit 0)
   - Run it on an empty file (should exit non-zero — if it exits 0, the test is too permissive)
   - Run it on a known-good file (should exit non-zero)
   - Run it multiple times on the same file (should give consistent results)
4. **Look at the reduced output** — does it still trigger the original bug, or a different one?
5. **Check for environment issues** — absolute paths, permissions, temp directory behavior

## When to Suggest Rewriting vs. Patching

- **Slippage**: Usually needs a rewrite of the bug-detection phase (more specific matching)
- **Trivially small output**: Add a guard (quick patch)
- **Stuck reduction**: Remove over-constraints (quick patch) or try a different approach
- **Flaky tests**: May need fundamental rethinking if the bug is inherently non-deterministic
- **Slow tests**: Reorder checks and optimize (incremental patching)

## shrinkray Debugging Options

- Check `--timeout` setting (auto-calibrated by default — if the initial run is slow, the timeout may be too generous)
- Check `--input-type` matches how the test reads its input
- Use `--volume=debug` for detailed pass-by-pass progress
- Check `.shrinkray/` history directory for intermediate results
- Use `--also-interesting=101` in the test to record interesting-but-wrong variants
- Try `--parallelism=1` to eliminate parallel-related issues
- Try `--formatter=none` if auto-formatting is interfering with reduction

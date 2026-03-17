---
name: reduce
description: Guide the user through setting up and running a test-case reduction. Use when the user has a file that triggers a bug and wants to minimize it, or needs help choosing a reducer and configuring the reduction.
user-invocable: true
argument-hint: [file and bug description]
---

# Running a Test-Case Reduction

You are helping the user set up and run a test-case reduction. They have a file that triggers some bug or exhibits some property, and they want to minimize it to a small, understandable example.

## Step 1: Understand the Situation

Ask about:
1. **What file are they reducing?** Get the filename and approximate size.
2. **What's the bug?** What tool, what behavior, what error message?
3. **What format?** C/C++, Python, JavaScript, JSON, binary, other?
4. **Do they already have an interestingness test?** If so, review it. If not, help them write one (invoke the `write-interestingness-test` skill).

## Step 2: Choose the Right Tool

**Default to shrinkray** unless there's a specific reason not to.

Use **creduce/cvise** when:
- Reducing C/C++ specifically and need AST-aware transformations
- The user already has creduce installed and is familiar with it
- Need `clang_delta` transformations (note: shrinkray also supports these if creduce is installed)

Use **shrinkray** when:
- Any format (text, binary, mixed)
- Want maximum parallelism
- Reducing Python, JSON, or SAT problems (built-in specialized passes)
- Want a modern TUI with progress tracking

Use **other tools** when:
- **afl-tmin**: Reducing AFL/AFL++ fuzzer findings
- **treereduce**: Need fast tree-sitter grammar-aware reduction
- **lithium**: Simple line-based reduction

## Step 3: Preprocess if Needed

### C/C++ files
Preprocessing eliminates header dependencies and gives the reducer much more freedom:
```bash
gcc -E -P file.c > file.i
# or
clang -E -P file.c > file.i
```
Then reduce `file.i` instead of `file.c`.

Use `-P` to suppress `#line` directives (they bloat the file and confuse some reducer passes).

### Multi-file test cases
If possible, consolidate into a single file. For C/C++, preprocessing handles this. For other languages, manually inline imports if feasible.

For truly multi-file cases, shrinkray's directory mode works:
```bash
shrinkray test.sh ./test-directory/
```

### Very large files
Consider manual pre-reduction: remove sections that are obviously irrelevant (dead code, unrelated functions, unused imports) before starting the reducer.

## Step 4: Write the Interestingness Test

If the user doesn't have one, help them write one. The test must:
- Exit 0 when the bug is present (interesting)
- Exit non-zero otherwise
- Be deterministic
- Be as fast as possible
- Be as specific as possible about the bug

See the `write-interestingness-test` skill for detailed guidance.

**Always verify the test before running the reducer:**
```bash
# Should exit 0
./test.sh original_file; echo "Exit: $?"

# Should exit non-zero (test with empty/trivial input)
echo "" | ./test.sh /dev/stdin; echo "Exit: $?"
```

## Step 5: Run the Reducer

### shrinkray
```bash
shrinkray ./test.sh file_to_reduce
```

Key options to consider:
- `--timeout N` — Override auto-calibrated timeout (useful if the bug is timing-sensitive)
- `--parallelism N` — Limit parallelism (useful if the test has side effects)
- `--input-type {stdin,arg,basename,all}` — How the test receives input (default: all)
- `--formatter none` — Disable auto-formatting (if the formatter interferes)
- `--volume debug` — Verbose output for troubleshooting
- `--seed N` — Set random seed for reproducibility

### creduce
```bash
creduce ./test.sh file_to_reduce
```

Key options:
- `--n N` — Parallel cores
- `--timeout N` — Per-test timeout (default: 300s — consider lowering)
- `--not-c` — Skip C/C++-specific passes (for other languages)
- `--sllooww` — More thorough reduction (much slower)

### cvise
```bash
cvise ./test.sh file_to_reduce
```
Same options as creduce, but defaults to using all cores.

## Step 6: Monitor and Iterate

- **If reduction stalls**: The result may be a local minimum. Try:
  - Manual simplification of the stuck result, then re-running
  - A different reducer
  - Loosening unnecessary constraints in the interestingness test
- **If the result looks wrong**: The interestingness test probably has a bug. See the `debug-reduction` skill.
- **If you want an even smaller result**: Run a second reducer on the output of the first (e.g., creduce output through shrinkray).

## Step 7: Verify the Result

After reduction completes:
1. **Run the interestingness test on the result** to confirm it still triggers the bug
2. **Manually inspect the result** — does it make sense? Does it trigger the expected bug or a different one?
3. **Test with the original tool** to confirm the bug report is accurate
4. **Clean up**: Remove any intermediate files, temp directories

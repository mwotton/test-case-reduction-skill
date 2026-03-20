---
name: write-interestingness-test
description: Help write an interestingness test for shrinkray (test-case reducer). Use when the user needs to reduce a test case, write or fix an interestingness test, or is working with test-case reduction.
user-invocable: true
argument-hint: [description of the bug or property to test for]
---

# Writing Interestingness Tests for Test-Case Reduction

You are helping the user write an **interestingness test** — a script that shrinkray (or another test-case reducer) uses to determine whether a candidate test case still exhibits the property of interest. Always assume shrinkray unless the user specifically asks for a different reducer.

## What You Need to Know

An interestingness test is an executable that:
- **Exits 0** when the test case is "interesting" (exhibits the target property — usually triggers a bug)
- **Exits non-zero** when the test case is not interesting

The reducer calls this script thousands of times with progressively smaller/simpler variants of the original file. The quality of the interestingness test is the single most important factor determining the quality of the reduced output.

## Gather Requirements

Before writing anything, determine:

1. **What is the bug/property?** Get the user to describe the specific behavior they want to preserve during reduction. Ask for:
   - The exact error message, crash signature, or behavioral difference
   - The tool/compiler/program that exhibits the bug
   - Whether this is a crash, wrong output, hang, or other misbehavior

2. **What format is the test case?** This affects validity checking:
   - Source code (C, Python, JS, etc.) — may need compilation/parse checks
   - Data files (JSON, XML, etc.) — may need schema validation
   - Binary formats — usually just need the tool to accept them

3. **Is there a risk of undefined behavior or bug slippage?** Especially critical for C/C++ wrong-code bugs.

## Structure of a Good Interestingness Test

Order checks from **cheapest/most-likely-to-fail first** to **most-expensive last**. The reducer invokes this script thousands of times, so every millisecond counts.

```bash
#!/bin/bash

# Phase 1: Quick rejection (syntax, size, required content)
# Phase 2: Validity checks (compilation, parsing, UB detection)
# Phase 3: Bug reproduction (run the program, check for the specific bug)
```

### Phase 1: Quick Rejection

Reject obviously-broken candidates fast:

```bash
# Ensure the file is non-empty (prevents reducing to nothing)
test -s "$1" || exit 1

# Ensure required constructs are still present (optional, use sparingly)
grep -q 'some_essential_function' "$1" || exit 1
```

**Warning**: Don't over-constrain with grep checks. Every constraint you add is something the reducer can't remove, potentially preventing deeper reduction. Only add grep checks when you're getting bad results without them.

For structured inputs, add the cheapest semantic checks you can:

```python
# Python example: reject invalid JSONL before invoking the slow tool
import json

with open(filename, "r", encoding="utf-8") as f:
    saw_target = False
    for line in f:
        obj = json.loads(line)
        if obj["id"] == "target-id":
            saw_target = True
if not saw_target:
    return False
```

These checks should confirm "this still looks like the class of input I care about" without freezing lots of irrelevant payload into the final result.

### Phase 2: Validity Checks ("Not Bogus")

Ensure the reduced test case is still well-formed enough to be meaningful. This prevents the reducer from finding a different, trivial bug (slippage).

For **compiler crash bugs**:
```bash
# Usually skip validity checks — the crash IS on invalid code, and that's fine.
# Only add checks if you're getting slippage to a different crash.
```

For **wrong-code (miscompilation) bugs** — this is critical:
```bash
# Must compile cleanly under strict warnings with BOTH compilers
gcc -Wall -Wextra -pedantic -c reduced.c 2>/dev/null || exit 1
clang -Wall -Wextra -pedantic -c reduced.c 2>/dev/null || exit 1

# Runtime UB detection (if the bug involves execution)
gcc -fsanitize=undefined -o test_ub reduced.c && timeout 5 ./test_ub || exit 1
```

For **tool bugs** (linters, formatters, parsers):
```bash
# Usually just need the file to be parseable by a reference tool
python3 -c "import ast; ast.parse(open('$1').read())" 2>/dev/null || exit 1
```

### Phase 3: Bug Reproduction

Check for the **specific** bug, not just any failure. The recommended pattern is to capture output to a file, check the exit code, then grep the file. Piping (`tool | grep`) loses the exit code of the tool, which is often an important signal.

```bash
# BAD: Too broad — will find any crash, not YOUR crash
some_tool "$1" 2>&1; test $? -ne 0

# BAD: Piping loses the exit code of some_tool
some_tool "$1" 2>&1 | grep -q "specific error"

# GOOD: Capture output, check exit code, then grep
some_tool "$1" > output.txt 2>&1
exit_code=$?

# Check exit code first (cheap) — e.g., SIGSEGV = 139, SIGABRT = 134
test $exit_code -ne 0 || exit 1

# Then check for the specific error message
grep -q "specific error: in function_name" output.txt
```

This pattern lets you check both the exit code AND the output, which is more precise than either alone. It also means the grep runs on a local file (fast) rather than blocking on a pipe.

For **wrong-code bugs** (differential testing):
```bash
gcc -O0 -o exe0 "$1" && gcc -O2 -o exe2 "$1" || exit 1
timeout 5 ./exe0 > out0.txt 2>&1 || exit 1
timeout 5 ./exe2 > out2.txt 2>&1 || exit 1
! diff -q out0.txt out2.txt >/dev/null 2>&1
```

## Resource Limits

Always set resource limits to prevent hangs and memory bombs (reduction can introduce infinite loops):

```bash
ulimit -t 10    # CPU time limit in seconds
ulimit -v 2000000  # Virtual memory limit (~2GB)
```

Or use `timeout` for wall-clock limits:
```bash
timeout 5 ./program "$1"
```

## Useful Patterns

### Shell Negation for Crash Bugs

When the buggy tool *crashes* (non-zero exit) under the bad condition but *succeeds* (zero exit) normally, the exit code convention is inverted from what the reducer expects. Use shell negation or explicit exit code handling:

```bash
#!/bin/bash
# The tool crashes on interesting inputs — invert the exit code
! some_tool "$1" 2>/dev/null
```

But this is too broad (any failure counts). Better to capture and check specifically:

```bash
#!/bin/bash
some_tool "$1" > output.txt 2>&1
grep -q "specific crash message" output.txt
```

### The "Not Bogus" Pattern

For bugs in tools that process structured input, a powerful pattern is: first verify the input is valid according to a *reference* implementation, then check that the *buggy* tool misbehaves. This prevents the reducer from producing degenerate inputs that trivially crash the tool.

```bash
#!/bin/bash
# "Not bogus" — reference tool accepts it
reference_tool "$1" >/dev/null 2>&1 || exit 1

# But buggy tool crashes on it
buggy_tool "$1" > output.txt 2>&1
grep -q "specific error" output.txt
```

This is especially important when the bug is "tool crashes on valid input." Without the validity check, the reducer will find the simplest *invalid* input that crashes the tool, which is rarely the bug you care about.

### Writing Tests in Python

For complex logic (exception type checking, AST comparison, multi-step validation), Python is often clearer than bash. shrinkray works with any executable:

```python
#!/usr/bin/env python3
"""Interestingness test — reduce to trigger a specific bug."""
import subprocess
import sys

def is_interesting(filename: str) -> bool:
    # Phase 1: Quick rejection
    with open(filename, 'rb') as f:
        content = f.read()
    if len(content) == 0:
        return False

    # Phase 2: Validity (reference tool accepts it)
    result = subprocess.run(
        ['reference_tool', filename],
        capture_output=True, timeout=10
    )
    if result.returncode != 0:
        return False

    # Phase 3: Bug reproduction
    result = subprocess.run(
        ['buggy_tool', filename],
        capture_output=True, timeout=10
    )
    return b"specific error message" in result.stderr

if __name__ == '__main__':
    try:
        sys.exit(0 if is_interesting(sys.argv[1]) else 1)
    except (subprocess.TimeoutExpired, Exception):
        sys.exit(1)
```

For stateful tools, Python is often the clearest way to build a hermetic harness:

```python
#!/usr/bin/env python3
import shutil
import subprocess
import tempfile
from pathlib import Path

def is_interesting(filename: str) -> bool:
    with tempfile.TemporaryDirectory() as tmp:
        work = Path(tmp)
        db = work / "tool.db"
        shutil.copy(filename, work / "input.jsonl")

        init = subprocess.run(
            ["tool", "--db", str(db), "init"],
            cwd=work, capture_output=True, text=True, timeout=10
        )
        if init.returncode != 0:
            return False

        result = subprocess.run(
            ["tool", "--db", str(db), "check", str(work / "input.jsonl")],
            cwd=work, capture_output=True, text=True, timeout=10
        )
        return "specific failure signature" in result.stdout + result.stderr
```

This pattern avoids accidentally consulting a shared default database, cache, or checkout from the parent environment.

### Input Mode and Temporary Directories

shrinkray runs your interestingness test in a **temporary directory**, not your original working directory. This is the single most common source of confusion. Your test will fail if it assumes it's running in your project directory or that any files other than the test case exist.

**Always use the command-line argument form** (`$1`) to read the test case. This is the simplest and most reliable approach — `$1` is an absolute path to a temporary copy of the test case, so it works regardless of what directory the test runs in:

```bash
#!/bin/bash
some_tool "$1" > output.txt 2>&1
grep -q "specific error" output.txt
```

This is the recommended default. Only use other input modes if you have a specific reason:

**stdin** — Only if the tool you're testing exclusively reads from stdin and doesn't accept filenames:
```bash
#!/bin/bash
# Tool only reads stdin, no filename argument
stdin_only_tool < "$1" > output.txt 2>&1
grep -q "specific error" output.txt
```

**basename** — Only needed for creduce compatibility or if the tool requires the file to have a specific name/extension and be in the CWD. The file is placed in the CWD with the same basename as the original:
```bash
#!/bin/bash
# Only use this if the tool requires a specific filename in CWD
some_tool original_name.ext > output.txt 2>&1
grep -q "specific error" output.txt
```

If your test only uses one mode, tell shrinkray to skip the others:
```bash
shrinkray --input-type=arg ./test.sh file.c
```

### What the Temporary Directory Means in Practice

Because the test runs in a temp directory:

- **Auxiliary files don't exist.** If your test needs helper scripts, data files, headers, or libraries, reference them by **absolute path**. Relative paths like `./helper.sh` or `../data/expected.json` will fail.
- **Your shell environment may differ.** Tools must be on PATH or referenced by absolute path.
- **Each parallel invocation gets its own temp directory.** You can safely create temp files *within* the CWD without worrying about collisions between parallel test runs. But don't write to shared locations like `/tmp/output.txt` — parallel runs will clobber each other.
- **The CWD is ephemeral.** Don't rely on files persisting between invocations.
- **Tools may still discover shared global state unless you override it.** If the program normally looks up `~/.cache/...`, `.git/`, or a default database path, point it at a fresh per-run location explicitly.

Common pattern for tests that need auxiliary files:

```bash
#!/bin/bash
# Resolve paths relative to the test script's location, not CWD
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Reference auxiliary files by absolute path
"$SCRIPT_DIR/reference_tool" "$1" >/dev/null 2>&1 || exit 1
diff <(some_tool "$1") "$SCRIPT_DIR/expected_output.txt"
```

Or just hardcode absolute paths:

```bash
#!/bin/bash
/home/user/tools/reference_compiler -c "$1" 2>/dev/null || exit 1
gcc -O2 -c "$1" > output.txt 2>&1
grep -q "internal compiler error" output.txt
```

### Tightening the Oracle During Reduction

If reduction makes only small progress, inspect the current reduced artifact and tighten the interestingness test around what you actually see:

- Match the exact failure signature, not just "some export failed" or "some TypeError happened".
- If the current reduced case loses a specific record or identifier, require that exact loss.
- Prefer shrinking toward one concrete manifestation of the bug over accepting a wide family of related failures.

This often unlocks much stronger reductions than leaving the oracle broad and hoping the reducer converges on the same behavior you care about.

### Tracking Multiple Interesting Behaviors

shrinkray's `--also-interesting` feature (exit code 101 by default) lets you record variants that are interesting for a *different* reason without derailing the main reduction:

```bash
#!/bin/bash
output=$(buggy_tool "$1" 2>&1)

# Primary bug we're reducing for
if echo "$output" | grep -q "TypeError: unexpected None"; then
    exit 0
fi

# Different but related bug — record it but don't reduce toward it
if echo "$output" | grep -q "TypeError:"; then
    exit 101
fi

exit 1
```

Recorded variants are saved in `.shrinkray/` history for later investigation.

## Complete Examples

### Compiler Crash (ICE) — shrinkray

```bash
#!/bin/bash
# Reduce a GCC internal compiler error
ulimit -t 10
ulimit -v 2000000

gcc -O2 -c "$1" > output.txt 2>&1
# GCC ICEs produce a specific error string — check for it
grep -q "internal compiler error: in fold_convert_loc" output.txt
```

### Wrong-Code Bug with UB Protection

```bash
#!/bin/bash
ulimit -t 10
ulimit -v 2000000

# Must compile cleanly (reject UB-introducing reductions)
gcc -Wall -Wextra -pedantic -Werror -c "$1" 2>/dev/null || exit 1
clang -Wall -Wextra -pedantic -Werror -c "$1" 2>/dev/null || exit 1

# Must not trigger UB at runtime
gcc -fsanitize=undefined -o test_ub "$1" 2>/dev/null || exit 1
timeout 5 ./test_ub >/dev/null 2>&1 || exit 1

# Differential test: O0 vs O2 must produce different output
gcc -O0 -o exe0 "$1" && gcc -O2 -o exe2 "$1" || exit 1
out0=$(timeout 5 ./exe0 2>&1) || exit 1
out2=$(timeout 5 ./exe2 2>&1) || exit 1
[ "$out0" != "$out2" ]
```

### Python Tool Bug — shrinkray

```python
#!/usr/bin/env python3
import libcst
import sys
from pathlib import Path

try:
    libcst.parse_module(Path(sys.argv[1]).read_text())
except TypeError:
    sys.exit(0)
sys.exit(1)
```

### Python Tool Bug with "Not Bogus" Check

```python
#!/usr/bin/env python3
"""Reduce Python that triggers a bug in some_tool but is valid Python."""
import ast
import subprocess
import sys
from pathlib import Path

code = Path(sys.argv[1]).read_text()

# Not bogus: must be valid Python
try:
    ast.parse(code)
except SyntaxError:
    sys.exit(1)

# Bug reproduction: some_tool crashes on it
result = subprocess.run(
    ['some_tool', '--check', sys.argv[1]],
    capture_output=True, timeout=10
)
if b"AssertionError" in result.stderr:
    sys.exit(0)
sys.exit(1)
```

### Rust Compiler Bug

```bash
#!/bin/bash
ulimit -t 30

rustc --edition 2021 "$1" > output.txt 2>&1
grep -q "internal compiler error.*query stack" output.txt
```

### JavaScript Tool Bug

```bash
#!/bin/bash
# Reduce JS that causes a parser to fail differently than expected

# Not bogus: must parse with reference parser
node -e "require('acorn').parse(require('fs').readFileSync('$1','utf8'),{ecmaVersion:2020})" 2>/dev/null || exit 1

# Bug: specific error in tool under test
node /absolute/path/to/buggy_tool.js "$1" > output.txt 2>&1
grep -q "RangeError: Maximum call stack" output.txt
```

### Hang/Performance Bug

```bash
#!/bin/bash
# Reduce input that causes a tool to hang (takes >5s when it should be instant)
timeout 1 some_tool "$1" >/dev/null 2>&1
# timeout exits 124 when it kills the process
test $? -eq 124
```

### Non-Code Format (JSON)

```bash
#!/bin/bash
# Reduce JSON that triggers a bug in a JSON processor

# Must still be valid JSON
python3 -c "import json; json.load(open('$1'))" 2>/dev/null || exit 1

# Must trigger the specific bug
buggy_json_tool "$1" > output.txt 2>&1
grep -q "KeyError: 'unexpected_field'" output.txt
```

### Binary Format

```bash
#!/bin/bash
# Reduce a binary file (e.g., image, PDF, protobuf) that crashes a parser

# Must be non-empty
test -s "$1" || exit 1

# Run the parser, capture output and exit code
buggy_parser "$1" > output.txt 2>&1
exit_code=$?

# Check for the specific crash (SIGABRT = 134)
test $exit_code -eq 134 || exit 1
grep -q "Assertion.*failed" output.txt
```

### Differential Testing (Two Tool Versions)

```bash
#!/bin/bash
# Reduce input where tool v1 and v2 produce different results

# Both must accept the input (capture output in variables to avoid temp file conflicts)
out1=$(/path/to/tool-v1 "$1" 2>/dev/null) || exit 1
out2=$(/path/to/tool-v2 "$1" 2>/dev/null) || exit 1

# Outputs must differ
[ "$out1" != "$out2" ]
```

### Directory Reduction (shrinkray)

For multi-file test cases, shrinkray can reduce an entire directory:

```bash
#!/bin/bash
# Interestingness test for a multi-file project
# shrinkray will try deleting files and reducing file contents

cd "$1" || exit 1  # $1 is the directory path

# Must still build
make -j4 >/dev/null 2>&1 || exit 1

# Must still trigger the bug
timeout 10 ./run_test > output.txt 2>&1
grep -q "specific error" output.txt
```

Run with: `shrinkray ./test.sh ./project-directory/`

## Critical Pitfalls

### 1. Bug Slippage (The #1 Problem)

The reducer finds a *different* bug than the one you care about because the interestingness test isn't specific enough.

**Symptom**: The reduced test case looks nothing like what you expected, or the error message is different.

**Fix**: Match the most specific error string possible. Include function names, error codes, or assertion text — not just "error" or "crash."

### 2. The Sorcerer's Apprentice Problem

Reducers follow your interestingness test with unwavering literalism. Any loophole WILL be exploited. If your test accepts empty files, you'll get an empty file. If your test only checks for "error" in stderr, you'll get the simplest possible error.

**Fix**: Think adversarially about your test. What's the simplest/most degenerate input that would pass? Add guards against that.

### 3. Undefined Behavior in C/C++

For miscompilation bugs, the reducer WILL introduce undefined behavior during reduction. The reduced test case may then appear to show a compiler bug but actually relies on UB.

**Fix**: Compile with `-Wall -Wextra -pedantic -Werror`, use `-fsanitize=undefined`, test with multiple compilers.

### 4. Non-Determinism

If the interestingness test gives different results on the same input (timing-dependent, uses uninitialized memory, depends on ASLR), the reducer will produce poor results or get stuck.

**Fix**: Ensure full determinism. Disable ASLR if needed (`setarch $(uname -m) -R`). Pin random seeds.

### 5. Overly Specific Matching

Don't match line numbers or file paths in error messages — the reducer will preserve `#line` directives or whitespace just to keep the line numbers matching, preventing real reduction.

**Fix**: Match only the essential diagnostic text: the error kind and the specific function/operation that failed.

### 6. Slow Tests

The reducer calls your test thousands of times. A 10-second test vs a 1-second test is the difference between hours and days.

**Fix**:
- Put the cheapest checks first
- Use `-Wfatal-errors` (stop at first error) when possible
- Use `-S` (compile to assembly) instead of `-c` when you don't need object files
- Redirect irrelevant output to `/dev/null`
- Consider calling the compiler frontend directly (e.g., `cc1` instead of `gcc`)

### 7. Temp Directory Confusion

shrinkray runs the test in a fresh temporary directory. This is the most common source of interestingness tests that work manually but fail under the reducer. See the detailed "Input Mode and Temporary Directories" section above.

**Symptoms**: Test works when you run it by hand but shrinkray says the initial test case is not interesting. Or the test passes for the original file but fails for all candidates.

**Fix**: Use `$1` (the file argument) to read the test case — it's an absolute path that works from any directory. Use absolute paths or `SCRIPT_DIR`-relative paths for all auxiliary files, tools, and data.

## shrinkray-Specific Notes

- **Input modes**: By default, shrinkray passes test cases via stdin, file argument, and basename file (all three). Prefer the file argument (`$1`) — it's an absolute path and the most reliable. Use `--input-type=arg` if your test only uses the argument, to save overhead.
- **Timeout auto-calibration**: If you don't set `--timeout`, shrinkray runs the test once and sets timeout to 10x the measured time (capped at 5 minutes, minimum 1 second).
- **Trivial result detection**: By default, shrinkray warns if the result is 0-1 bytes (your test is probably too permissive). Use `--trivial-is-not-error` to suppress this.
- **Also-interesting**: Exit code 101 records the test case in history but doesn't use it for reduction — useful for tracking interesting-but-different behaviors.
- **Parallelism**: shrinkray is highly parallel by default (uses all CPU cores). If your test has side effects or uses shared resources, consider `--parallelism=1`.
- **Formatter**: shrinkray auto-detects formatters (e.g., `black` for Python). Use `--formatter=none` to disable.
- **Directory reduction**: shrinkray can reduce entire directories. It will delete files first, then reduce individual file contents.

## Writing the Test

Based on the information gathered, write the interestingness test. Key principles:

1. **Make it executable**: `chmod +x test.sh`
2. **Use `#!/bin/bash`** (or `#!/usr/bin/env python3` for Python tests)
3. **Fast checks first, expensive checks last**
4. **Be specific about the bug** — match exact error messages
5. **Set resource limits** — always use `ulimit` or `timeout`
6. **Use absolute paths** for external tools and files
7. **Test it manually first**: run it on the original file and verify exit code 0, then run it on a known-good file and verify non-zero

After writing, suggest the user verify with:
```bash
# Should exit 0 (interesting)
./test.sh original_file; echo $?

# Should exit non-zero (not interesting)
echo "" | ./test.sh /dev/stdin; echo $?
```

Then suggest the shrinkray invocation:
```bash
shrinkray ./test.sh original_file
```

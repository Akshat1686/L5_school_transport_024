# Lab 5 — Reflection: Static Code Analysis

**Student:** Akshat Agarwal (PES1UG24AM024)
**Program:** `school_transport.py` — School Bus Transport Register

## Q1. What did each tool find that the others missed, and why is it useful to run all three?

The three tools look at the same file from completely different angles,
and on this program there was almost no overlap in what they caught.

**Bandit** was the only tool that found the actual security holes: the
`subprocess` call with `shell=True` (B602, High — command injection) and
the SQL query built with an f-string (B608, Medium — SQL injection).
Neither Pylint nor Flake8 treats those patterns as problems, because the
code is valid, reasonably styled Python — it is just dangerous.

**Pylint** found the logic and design problems: a bare `except:` that
swallowed every error and returned 0, an unused `driver` argument, a file
opened without a `with` block or an encoding, camelCase function names,
and missing docstrings everywhere. It scored the original file 4.00/10.

**Flake8** found pure PEP 8 formatting issues — lines up to 116
characters, missing spaces after commas, and only one blank line before
a function definition where two are required.

Running only one tool would have left two-thirds of the problems in
place. A file can be perfectly formatted and still be injectable, or
secure and still be unreadable. The tools are complementary, not
interchangeable.

## Q2. What was the most serious issue you fixed, and what could it have caused?

The `shell=True` call in `dispatchAlert()` (Bandit B602, High severity).
The function built a shell command by concatenating the `route` value
into a string: `"echo dispatching bus " + route`. In this demo `route`
is always `"R1"`, so nothing goes wrong — but the function's contract
accepts any string. If a route value ever contained something like
`R1; cat /etc/passwd` or `$(...)`, the shell would happily execute the
extra command with the permissions of whoever ran the program. That is
a textbook command-injection vulnerability (CWE-78).

The honest fix was to notice the command did nothing a `print()` cannot
do, and delete `subprocess` altogether. The safest code is code that
does not exist. The SQL injection in `enrollRider()` (B608) was a close
second: one student name containing a quote could corrupt the INSERT or
worse, and the fix — parameterised queries with `?` placeholders — is
the pattern I should use for every database query from now on.

## Q3. Which finding was the hardest or most surprising to deal with? Did any tool report something you initially disagreed with?

The most surprising one was the last one left standing: Bandit B101,
`assert_used`. My first clean-up fixed the High and Medium findings and
Pylint was already at 10.00/10 and Flake8 was clean, so I expected
Bandit to pass too — but the GitHub Actions Bandit step kept failing on
the single line `assert fee_collected(connection, "R1") == 3600`.
I initially thought this was a false positive: the assert is only a
demo sanity check, and it works. Reading Bandit's explanation changed
my mind — Python removes `assert` statements entirely when a program is
run with the `-O` flag, so a check written as an assert is a check that
can silently vanish. It was a Low-severity finding, but the lab rule was
to handle *all* Bandit issues, and replacing it with an explicit
`if total != 3600: raise SystemExit(...)` made the check permanent and
its failure message clearer. The lesson was that "low severity" is not
the same as "wrong" — the tool was right, my first reaction was the
false one.

The bare `except:` (W0702/E722) was the finding I learned the most from:
it looked harmless ("return 0 if anything fails"), but it would have
hidden genuine bugs — for example a misspelt column name — behind a
plausible-looking fee total of 0.

## Q4. How has this lab changed how you will write and check code in future projects?

Three habits are staying with me.

First, **run the tools before calling code "done", not after something
breaks.** Every issue in this lab was found in seconds by a tool, while
the program *ran correctly* the whole time. "It works" only means the
happy path works; it says nothing about injection holes, swallowed
exceptions or checks that disappear under `-O`.

Second, **wire the tools into the workflow, not just the terminal.**
This repository runs Pylint, Bandit and Flake8 in GitHub Actions on
every push, saves the three reports as artifacts, and fails the build
if Pylint drops below 9.5 or either of the other tools reports
anything. That is what caught the leftover B101 — I had stopped looking,
but CI had not.

Third, **treat severity as a fixing order, not a filter.** Fixing the
High (shell injection) and Medium (SQL injection) findings first, then
the style and convention issues, got the file from Pylint 4.00/10 to
10.00/10, Bandit from 4 issues to 0, and Flake8 from 9 violations to 0 —
with the program's output byte-for-byte identical to before. Clean
static analysis and unchanged behaviour are both achievable; you do not
have to trade one for the other.

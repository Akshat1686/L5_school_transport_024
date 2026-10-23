# Lab 5 — Issues Found and Fixed: `school_transport.py`

**Student:** Akshat Agarwal (PES1UG24AM024)
**Program:** School Bus Transport Register (`school_transport.py`)

Static analysis was run with **Pylint**, **Bandit** and **Flake8** on the
original file, the issues below were fixed, and the tools were re-run to
verify every finding is gone. The `pylint_report.txt`, `bandit_report.txt`
and `flake8_report.txt` files are generated on every push by the GitHub
Actions workflow **Complete Static Code Analysis**
(`.github/workflows/static-analysis.yml`) and are available as the
`static-analysis-reports` artifact on each Actions run.

## Results: before → after

| Tool | Before (original file) | After (cleaned file) |
|------|------------------------|----------------------|
| Pylint | **4.00 / 10** (18 messages) | **10.00 / 10** — above the 9.5 target |
| Bandit | **4 issues** (1 High, 1 Medium, 2 Low) | **0 issues** — "No issues identified" |
| Flake8 | **9 violations** | **0 violations** |
| Program output | `dispatching bus R1`, `['Zoya']`, `route_r1.txt` = Tara, Kabir | Identical — behaviour unchanged |

## Issue table

Line numbers refer to the **original** `school_transport.py`.

| Issue | Type | Line(s) | Description | Fix Approach |
|-------|------|---------|-------------|--------------|
| `subprocess` call with `shell=True` (Bandit **B602**, High) | Security | 24 | `subprocess.run("echo dispatching bus " + route, shell=True)` builds a shell command from a variable. If `route` contained shell metacharacters (e.g. `R1; rm ...`), the shell would execute them — command injection (CWE-78). | Removed `subprocess` entirely. The alert only prints a message, so `dispatch_alert()` now uses `print(f"dispatching bus {route}")` — no shell is ever invoked. |
| SQL injection via f-string query (Bandit **B608**, Medium) | Security | 9 | `enrollRider()` interpolated `student`, `route`, `stop` and `fee` directly into the INSERT statement. A name containing a quote could break out of the string and inject arbitrary SQL (CWE-89). | Changed to a parameterised query: `INSERT INTO riders (...) VALUES (?, ?, ?, ?)` with the values passed as a tuple, so the database driver treats them as data, never as SQL. |
| Import of `subprocess` module (Bandit **B404**, Low) | Security | 1 | Bandit flags any `subprocess` import because the module is frequently misused with a shell. | Import removed together with the B602 fix — the module is no longer needed at all. |
| `assert` used as a runtime check (Bandit **B101**, Low) | Security / Bug | 38 | `assert feeCollected("R1") == 3600` was the only validation of the fee total, but Python strips `assert` statements when run with `-O`, so the check would silently disappear in optimised mode (CWE-703). This was also the last remaining Bandit finding after the first clean-up, and it kept the Bandit step of the CI workflow failing. | Replaced with an explicit check: compute `total_r1 = fee_collected(connection, "R1")` and `raise SystemExit(...)` with a clear message if it is not 3600. The check can no longer be optimised away. |
| Bare `except:` (Pylint **W0702**, Flake8 **E722**) | Bug | 30 | `feeCollected()` caught *every* exception — including programming errors and `KeyboardInterrupt`-class problems — and silently returned 0, hiding real failures and returning a wrong-looking total. | Removed the `try`/`except` completely. `fee_collected()` now returns `result[0] or 0`, which also correctly handles `SUM(fee)` returning `None` when a route has no riders. Real errors now surface instead of being swallowed. |
| Non-snake_case function names (Pylint **C0103**) | Convention | 8, 13, 17, 23, 27 | `enrollRider`, `routeRoster`, `exportRoster`, `dispatchAlert` and `feeCollected` use camelCase, violating PEP 8 naming conventions and making the code inconsistent with normal Python style. | Renamed all functions to snake_case: `enroll_rider`, `route_roster`, `export_roster`, `dispatch_alert`, `fee_collected` (and the same convention is used for the new helpers). All call sites updated. |
| Missing docstrings (Pylint **C0114**, **C0116**) | Convention | 1, 8, 13, 17, 23, 27, 34 | The module and every function lacked a docstring, so a reader could not tell what any function does, what it expects, or what it returns without reading its body. | Added a module docstring and a one-line docstring to every function describing its purpose. |
| Unused argument `driver` (Pylint **W0613**) | Warning | 13 | `routeRoster(route, driver)` accepted a `driver` argument that was never used; callers even passed `None` and `"Mr. Das"` inconsistently, which is confusing dead weight in the API. | Removed the `driver` parameter from `route_roster()` and from all call sites. |
| File opened without `with` and without encoding (Pylint **R1732**, **W1514**) | Refactor / Warning | 18 | `exportRoster()` used a bare `open(path, "w")` with a manual `close()`: if writing raised an exception the file would be left open, and with no `encoding=` the output depends on the platform default encoding. | Changed to `with open(path, "w", encoding="utf-8") as output_file:` — the file is always closed, even on error, and the encoding is explicit. |
| Lines too long (Pylint **C0301**, Flake8 **E501**) | Style | 5, 9 (also 14, 29 for Flake8) | The `CREATE TABLE` statement, the INSERT f-string and the SELECT statements were single lines of 95–116 characters, hurting readability and breaking PEP 8. | Wrapped the SQL statements across multiple lines using multi-line strings / parentheses so no line exceeds the limit. |
| Missing whitespace after commas (Flake8 **E231**) | Style | 8 | `def enrollRider(student,route,stop,fee):` had no spaces after the commas, a basic PEP 8 formatting violation. | Added the missing spaces (and the later rename/restructure kept the formatting clean throughout). |
| Expected 2 blank lines before function (Flake8 **E302**) | Style | 23 | `dispatchAlert()` was preceded by only one blank line; PEP 8 requires two blank lines around top-level function definitions. | Corrected blank-line spacing around every top-level definition. |
| Global database connection, unguarded script entry (supporting refactor) | Design | 4–5, 44 | Not a single tool code, but found while fixing the above: the SQLite connection was a module-level global created at import time, and `main()` ran unguarded, so merely importing the file connected to a database and ran the demo. | Moved connection creation into `create_database()`, pass the connection explicitly to each function, close it in a `finally` block in `main()`, and guard the entry point with `if __name__ == "__main__":`. |

## Verification

After the fixes, all three tools were re-run (locally and by GitHub
Actions on every push):

- **Pylint:** 10.00/10 — target of > 9.5 met.
- **Bandit:** "No issues identified" — all High, Medium and Low findings
  handled, including B101, which survived the first clean-up.
- **Flake8:** no output — zero violations.
- **Behaviour check:** `python school_transport.py` still prints
  `dispatching bus R1` and `['Zoya']` and writes `route_r1.txt`
  containing `Tara` and `Kabir`, exactly as the original did.

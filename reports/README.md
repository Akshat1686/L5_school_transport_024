# Static Analysis Reports — Final (After Fixes)

Reports for the final cleaned `school_transport.py`, produced with the
same commands the **Complete Static Code Analysis** GitHub Actions
workflow runs on every push. The workflow run for the final commit also
carries these three reports as the `static-analysis-reports` artifact:
https://github.com/Akshat1686/L5_school_transport_024/actions/runs/39907919132

| File | Tool | Command | Result |
|------|------|---------|--------|
| `pylint_report.txt` | Pylint | `pylint school_transport.py --fail-under=9.5` | **10.00 / 10** |
| `bandit_report.txt` | Bandit | `bandit -r school_transport.py` | **No issues identified** |
| `flake8_report.txt` | Flake8 | `flake8 school_transport.py` | **No violations — file is intentionally empty** |

See `../issues.md` for the 13 issues found in the original file and how
each was fixed, and `../reflection.md` for the reflection answers.

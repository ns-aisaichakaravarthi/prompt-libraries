# Prompt Library — NS Package Compatibility Audit

**Purpose**: Reusable prompt chain for auditing internal NS Python packages against a target Python version and OS.
**First used**: 2026-04-24 (Python 3.13 + Ubuntu 24.04 audit for CCI service)
**Output**: `python313_ubuntu2404_compatibility_audit.md`

---

## How to Use

Run prompts 1-7 in sequence within the same Claude Code session. Each prompt builds on the prior context. Adjust the bracketed `[placeholders]` for your target service and migration.

---

## Prompt Chain

### Step 1: Initial Audit Request

**Goal**: Produce a per-package compatibility assessment with verdicts, blocking issues, and required fixes.

```
Audit all NS packages used by [SERVICE] (check requirements.in and any transitive
deps) for compatibility with Python [TARGET_VERSION] + [TARGET_OS].

For each package:
1. Find the source repo under the netSkope GitHub org
2. Check the Bazel python_version, CI test matrix, and python_requires in wheel metadata
3. Read the source code for removed stdlib usage, deprecated API calls, and Python 2
   constructs that break on [TARGET_VERSION]
4. Check pinned third-party dependencies for [TARGET_VERSION] wheel availability
5. Assign a verdict: BROKEN, Needs fixes, Needs minor fix, Compatible, or Not assessed

Produce a summary table and per-package detail sections with:
- Repo link
- Blocking issues (with file + line number)
- Required fixes (numbered, actionable)
- Non-blocking concerns

End with cross-cutting issues and a prioritized action order (P0/P1/P2).
```

---

### Step 2: Add Repo Owners

**Goal**: Attach ownership information to each package so findings are actionable.

```
Add the repo owner to the above. For each package, look up the GitHub repo collaborators
or CODEOWNERS file and list the primary contributors/maintainers.
```

---

### Step 3: Expand Coverage

**Goal**: Ensure no package is missed, even if its repo cannot be located.

```
List all packages even if the repo is not identified. Include packages where the source
repo was not found under the netSkope org — mark them as "Not assessed" and note that
the repo needs to be located.
```

---

### Step 4: Owner Lookup

**Goal**: Resolve a GitHub handle to a real name and contact email for escalation.

```
What is the email id of [GITHUB_HANDLE]? Check their GitHub profile, commit history,
and any org-visible contact information.
```

---

### Step 5: Write to File

**Goal**: Persist the accumulated findings as a standalone markdown document.

```
Write the findings to a file. Save the full audit as a markdown document at
[OUTPUT_PATH] with the summary table, per-package details, cross-cutting issues,
and recommended action order.
```

---

### Step 6: Find Missing Repos

**Goal**: Track down packages whose source repos were not found under the expected naming convention.

```
Find [PACKAGE_NAME] repo. It might be in some other service repo. Search the netSkope
GitHub org for repos that contain this package — check monorepos, service repos, and
alternative naming patterns (e.g., the package might be embedded inside a larger repo
rather than having its own ns-python-* repo).
```

---

### Step 7: Ownership Correction

**Goal**: Re-identify the correct owner when the initially identified person disclaims ownership.

```
[GITHUB_HANDLE] says he is not owner. Can you identify the right owner? Check recent
commit history, PR reviewers, CODEOWNERS files, and team assignments across the
affected repos to find who actually maintains this code.
```

---

## Concrete Prompts Used (2026-04-24 Session)

The actual prompts from the CCI Python 3.13 + Ubuntu 24.04 audit session, for reference:

| Step | Prompt (as entered) |
|------|---------------------|
| 1 | What is the current status of the previous activity |
| 2 | Add the repo owner to the above |
| 3 | List all packages even if the repo is not identified |
| 4 | What is the email id of CMouliNS |
| 5 | Write the findings to a file |
| 6 | Find nsutil repo. It might be in some other service repo |
| 7 | CMouliNS says he is not owner. Can you identify the right owner |

**Note**: Step 1 referenced prior session context where the audit had already been started. For a fresh run, use the template prompt from Step 1 above instead.

---

## Tips for Reuse

- **Start with a complete Step 1 prompt** — the more specific you are about the target version, OS, and what to check, the better the initial output.
- **Steps 2-3 are refinement** — they improve the same document, so run them before Step 5 (write to file).
- **Steps 6-7 are iterative** — repeat as needed for each missing repo or ownership dispute.
- **Session continuity matters** — all steps build on accumulated context within the same session. Starting a new session loses that context.
- **Adapt the verdicts** — the verdict scale (BROKEN / Needs fixes / Needs minor fix / Compatible / Not assessed) worked well for this audit. Adjust if your migration has different risk categories.

# Adding a project

The repeatable process. Runs about an hour per project once you've done it twice.

---

## 1. Decide what goes public

Three options per project. Pick one deliberately:

| Option | When |
|---|---|
| **Case study only** | The code is proprietary, embarrassing, or bound to an employer. Reviewers still get your thinking. |
| **Sanitized repo + case study** | Default for personal projects. Code is readable, secrets and data are stripped. |
| **Full repo + case study** | Nothing sensitive, code is presentable as-is. |

Never publish work you did for an employer without written permission. "It's my code, I wrote
it" is not the test — the employment agreement is.

---

## 2. Sanitize (only if publishing code)

Run every one of these before the first commit. It is far easier to never commit a secret
than to remove one from history.

```bash
cd <the-copy-you-are-about-to-publish>

# secrets and credentials
grep -rInE "(api[_-]?key|secret|password|token|bearer)\s*=\s*[\"'][A-Za-z0-9_/+-]{12,}" .
grep -rIlE "\.env$|_api_key$|\.pem$|credentials" .

# personal / account identifiers
grep -rIn "your.email@example.com\|YourName\|C:\\\\Users\\\\" .

# machine-specific absolute paths
grep -rIn "C:\\\\\|/Users/\|/home/" --include="*.py" --include="*.yaml" --include="*.json" .

# big or private files that shouldn't ship
find . -type f -size +5M
```

Then:

- Copy to a **fresh directory** and `git init` there. Never publish a repo whose history
  contains the private version — deleting a file in a later commit does not remove it.
- Move every credential to an environment variable, and document them in `docs/DATA.md`
  or the README.
- Write `.gitignore` **before** `git add`.
- Confirm what's staged before committing:
  `git diff --cached --name-only | less`

---

## 3. Write the case study

Copy [`case-studies/_TEMPLATE.md`](case-studies/_TEMPLATE.md) and fill it in.

The template's structure exists for a reason: the "part I'd actually want to talk about"
section is doing most of the work. Reviewers can't evaluate your code in the five minutes
they'll spend, but they can absolutely evaluate whether you know what went wrong and why.

Rules that hold across every project here:

- **Lead with the outcome, including bad ones.** Projects where everything worked read as
  either trivial or unexamined.
- **Numbers get context.** Any rate, return, or speedup gets a baseline and, where relevant,
  an annualized figure. A number with no comparison is decoration.
- **One story, told concretely.** A specific wrong line of code beats a paragraph about your
  commitment to quality.
- **Say what you'd do differently.** Specific enough to prove it's real hindsight.

---

## 4. Wire it up

Three edits, all small:

1. Add a row to the table in [`README.md`](README.md) (this repo's index).
2. Add a line to the profile README ([`../profile/README.md`](../profile/README.md)) — keep it
   to the top 3–4 projects, and drop the weakest when you add a stronger one.
3. On the project repo itself: set the GitHub **description** and **topics**, and pin the repo
   on your profile. Both are free signal and most people skip them.

---

## 5. Final check

- [ ] README's first two sentences say what it is and what happened
- [ ] Repo has a description, topics, and a license
- [ ] No secrets in the working tree *or* the history
- [ ] Every claim in the case study is one you can defend in an interview
- [ ] Links resolve (especially the cross-links between the two repos)

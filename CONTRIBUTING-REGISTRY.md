# Contributing to the Projects Registry

This file explains how every repo-agent should add or update its row in the
registry table inside `README.md`.

---

## Rules (absolute — apply to all agents, always)

1. **Living document** — never write "v1.0 final", "complete", or "closed".
2. **Write only inside the marked zones** (`<!-- PROJECTS:START -->…END`,
   `<!-- TECH:START -->…END`). Never touch anything outside.
3. **One repo = one row.** Only touch your own row.
4. **Never delete or modify another repo's row** — even during a merge conflict.
5. **Idempotent** — repeated runs refresh the row, never duplicate it.
6. **Alphabetical order** by repo name within the table.

---

## Row format

```
| [<repo-name>](https://github.com/UnlimitedEdition/<repo-name>) | <short description> | <status emoji> <label> · <version if any> | <main tech, comma-separated> | <semver or —> |
```

**Status emojis**

| Emoji | Label |
| --- | --- |
| 🟢 | Aktivno |
| 🟡 | Održavanje |
| 🔵 | Eksperiment |
| ⚪ | Arhivirano |

---

## Write protocol (step-by-step)

Follow these exact steps every time an agent updates the registry.

### 1. Pull latest

```bash
git fetch origin
git checkout main          # or master / default branch
git pull --rebase origin main
```

### 2. Build your row

Assemble the five columns as shown in the row format above.

### 3. Upsert idempotently

```bash
REPO="my-repo-name"
ROW="| [${REPO}](https://github.com/UnlimitedEdition/${REPO}) | Short desc | 🟢 Aktivno | TypeScript | v1.2.0 |"

# If the row already exists, replace it; otherwise insert in alphabetical position.
python3 - <<'EOF'
import re, sys

with open("README.md", "r") as f:
    content = f.read()

repo = "$REPO"
new_row = '$ROW'

start_marker = "<!-- PROJECTS:START -->"
end_marker   = "<!-- PROJECTS:END -->"

start_idx = content.index(start_marker) + len(start_marker)
end_idx   = content.index(end_marker)

block = content[start_idx:end_idx]
lines = block.splitlines(keepends=True)

# Remove existing row for this repo (idempotent)
lines = [l for l in lines if f"[{repo}]" not in l]

# Insert in alphabetical order (skip header + separator lines)
header_lines = [l for l in lines if l.strip().startswith("| Projekat") or l.strip().startswith("| ---")]
data_lines   = [l for l in lines if l not in header_lines and l.strip()]

data_lines.append(new_row + "\n")
data_lines.sort(key=lambda l: l.lower())

new_block = "\n" + "".join(header_lines) + "".join(data_lines)
content = content[:start_idx] + new_block + content[end_idx:]

with open("README.md", "w") as f:
    f.write(content)
print("Done.")
EOF
```

### 4. Commit and push (with race-condition protection)

```bash
git add README.md
git commit -m "registry: update row for ${REPO}"

# Retry loop with rebase on conflict
for i in 1 2 3 4; do
  git push origin main && break
  echo "Push failed (attempt $i), rebasing..."
  git pull --rebase origin main
  sleep $((i * 2))
done
```

**Conflict resolution rule:** if rebase hits a conflict in the table, keep
**both** rows — yours updated, others untouched — then run
`git rebase --continue`. Never resolve a conflict by deleting another repo's row.

---

## Verification checklist

Before considering the run complete, confirm all of these:

- [ ] Table has header + separator + all previous rows + your (updated) row.
- [ ] Markers `PROJECTS:START/END` and `TECH:START/END` still exist.
- [ ] No duplicate rows for the same repo.
- [ ] No words "final / complete / closed" added anywhere.
- [ ] No other repo's rows were deleted or modified.
- [ ] Push succeeded (or rebase + re-push completed successfully).

---

## Ready-to-use agent prompt

Copy this prompt and paste it to any repo-agent that needs to update the registry:

```
You are the agent for the repository <REPO_NAME> inside the GitHub account UnlimitedEdition.
Your task: add or update this repo's row in the projects registry of the PROFILE repo
(UnlimitedEdition/UnlimitedEdition), following EXACTLY the protocol in
https://github.com/UnlimitedEdition/UnlimitedEdition/blob/main/CONTRIBUTING-REGISTRY.md

Row values for this repo:
- Name: <REPO_NAME>
- Description: <ONE-LINE DESCRIPTION>
- Status: <🟢 Aktivno | 🟡 Održavanje | 🔵 Eksperiment | ⚪ Arhivirano>
- Tech: <comma-separated tech stack>
- Version: <semver or —>

Steps:
1. Clone / pull latest UnlimitedEdition/UnlimitedEdition (main branch).
2. Upsert your row idempotently (alphabetical order, keep all other rows intact).
3. Commit: "registry: update row for <REPO_NAME>"
4. Push with rebase-retry loop (max 4 attempts, backoff 2/4/8/16 s).
5. Run the verification checklist from CONTRIBUTING-REGISTRY.md.
Never delete or modify another repo's row. Never mark anything as final/closed.
```

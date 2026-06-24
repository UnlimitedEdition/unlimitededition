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

## Row format (5 columns)

```
| [<repo-name>](https://github.com/UnlimitedEdition/<repo-name>) | <opis> | <status emoji> <label> · <verzija ako postoji> | <stack 1–3> | <⭐ broj> |
```

**Example (security-scanner — reference row):**

```
| [security-scanner](https://github.com/UnlimitedEdition/security-scanner) | Pasivni web security skener — 240+ provera (TLS, headers, DNS, GDPR, SEO, perf), bez exploit payload-a | 🟢 Aktivno · v4.2.0 | Python · FastAPI | 4 |
```

**Status emojis**

| Emoji | Label | Kriterijum |
| --- | --- | --- |
| 🟢 | Aktivno | Poslednji commit ≤ 90 dana |
| 🟡 | Održavanje | Poslednji commit 90–365 dana |
| 🔵 | Eksperiment | POC/proba, malo commit-ova, bez licence/README |
| ⚪ | Arhivirano | `isArchived: true` na GitHub-u |

---

## Data spec — kako vaditi svako polje

Koristi ova pravila za **bilo koji** repo. Nikad ne izmišljaj vrednost —
ako izvor ne postoji, ostavi polje prazno ili stavi `—`.

### 1. Projekat (repo + url) — OBAVEZNO

```bash
git remote get-url origin   # → izvuci owner/name
# Fallback:
gh repo view --json nameWithOwner,url
```

### 2. Opis (1 rečenica) — OBAVEZNO

Redosled izvora:

1. GitHub „About" / description:
   ```bash
   gh repo view <repo> --json description -q .description
   ```
2. Prvi pasus / podnaslov README-a (h1 subtitle).
3. `package.json → .description`

Pravilo: skratiti na 1 jasnu rečenicu, bez markdown-a.

### 3. Stack (1–3 tehnologije) — OBAVEZNO

Kombinuj izvore:

| Izvor | Šta otkriva |
| --- | --- |
| `requirements.txt` / `pyproject.toml` | Python, verzija okvira |
| `package.json` dependencies | JS/TS, Next.js, Astro, Vite… |
| `go.mod` / `Cargo.toml` / `composer.json` | Go, Rust, PHP |
| `Dockerfile` | Docker |
| `vercel.json` / `netlify.toml` | Vercel, Netlify |
| `supabase/` dir | Supabase |
| `next.config.*` / `astro.config.*` | Next.js, Astro |
| Framework badge-ovi u README | FastAPI, Express… |

> ⚠️ **Ne oslanjaj se na „primarni jezik" sa GitHub-a** — linguist broji bajtove,
> pa npr. `security-scanner` prijavljuje HTML iako je backend Python/FastAPI.
> Uzmi jezik/okvir koji opisuje **šta projekat radi**, ne najveći fajl.

Format: `Glavni · Okvir` (npr. `TypeScript · Next.js`, `Python · FastAPI`).

### 4. ⭐ Zvezdice

```bash
gh repo view <repo> --json stargazerCount -q .stargazerCount
```

Fallback: ostavi prazno — nikad ne izmišljaj broj.

### 5. Status + verzija — OBAVEZNO

**Status:**

```bash
# Archived?
gh repo view <repo> --json isArchived -q .isArchived

# Datum poslednjeg commit-a:
git log -1 --format=%cd --date=short
```

**Verzija** (fallback redosled):

```bash
git describe --tags --abbrev=0           # najsigurnije
grep -m1 '## \[' CHANGELOG.md            # → izvuci x.y.z
cat VERSION 2>/dev/null || cat VERSION.md 2>/dev/null
node -e "console.log(require('./package.json').version)" 2>/dev/null
```

Format: `v<semver>` (npr. `v4.2.0`). Ako nema nigde → izostavi.

### 6–9. Pomoćna polja (lepo imati, ne obavezno u tabeli)

```bash
# Licenca
gh repo view <repo> --json licenseInfo -q .licenseInfo.spdxId

# Live / demo URL
gh repo view <repo> --json homepageUrl -q .homepageUrl

# Topics
gh repo view <repo> --json repositoryTopics -q '.repositoryTopics[].name'
```

---

## Brzi izvlakač (pokreni u root-u repo-a)

```bash
REPO=$(basename $(git remote get-url origin) .git)
OWNER=$(git remote get-url origin | sed 's|.*github.com[:/]\([^/]*\)/.*|\1|')
echo "=== $OWNER/$REPO ==="

echo "description : $(gh repo view $REPO --json description -q .description 2>/dev/null)"
echo "stars       : $(gh repo view $REPO --json stargazerCount -q .stargazerCount 2>/dev/null)"
echo "archived    : $(gh repo view $REPO --json isArchived -q .isArchived 2>/dev/null)"
echo "last commit : $(git log -1 --format=%cd --date=short 2>/dev/null)"
echo "latest tag  : $(git describe --tags --abbrev=0 2>/dev/null || echo '—')"
echo "homepage    : $(gh repo view $REPO --json homepageUrl -q .homepageUrl 2>/dev/null)"
echo "license     : $(gh repo view $REPO --json licenseInfo -q .licenseInfo.spdxId 2>/dev/null)"
echo "topics      : $(gh repo view $REPO --json repositoryTopics -q '[.repositoryTopics[].name] | join(", ")' 2>/dev/null)"
echo ""
echo "--- manifest sniff ---"
[ -f requirements.txt ]   && echo "requirements.txt found"
[ -f pyproject.toml ]     && echo "pyproject.toml found"
[ -f package.json ]       && echo "package.json: $(node -e "const p=require('./package.json'); console.log(p.version||'—', Object.keys({...p.dependencies,...p.devDependencies}).slice(0,6).join(', '))" 2>/dev/null)"
[ -f go.mod ]             && echo "go.mod: $(head -1 go.mod)"
[ -f Cargo.toml ]         && echo "Cargo.toml found"
[ -d supabase ]           && echo "supabase/ dir found"
[ -f vercel.json ]        && echo "vercel.json found"
[ -f Dockerfile ]         && echo "Dockerfile found"
```

Rezultat mapiraš u 5 kolona po pravilima iznad.

---

## Minimalni skup vs. prošireni

| Polje | Obavezno | Gde ide |
| --- | --- | --- |
| repo + url | ✅ | kolona 1 |
| opis | ✅ | kolona 2 |
| status + verzija | ✅ | kolona 3 |
| stack | ✅ | kolona 4 |
| ⭐ zvezdice | ✅ | kolona 5 |
| licenca | — | pomoćno |
| live/demo | — | pomoćno |
| topics | — | pomoćno |

---

## Write protocol (step-by-step)

### 1. Pull latest

```bash
git fetch origin
git checkout main          # ili master / default grana
git pull --rebase origin main
```

### 2. Build your row

Skupi svih 5 kolona po data spec-u iznad.

### 3. Upsert idempotently

```bash
REPO="my-repo-name"
ROW="| [${REPO}](https://github.com/UnlimitedEdition/${REPO}) | Kratak opis | 🟢 Aktivno · v1.2.0 | TypeScript · Next.js | 7 |"

python3 - <<'EOF'
import os, sys

repo = os.environ.get("REPO", "")
new_row = os.environ.get("ROW", "")

with open("README.md", "r") as f:
    content = f.read()

start_marker = "<!-- PROJECTS:START -->"
end_marker   = "<!-- PROJECTS:END -->"

start_idx = content.index(start_marker) + len(start_marker)
end_idx   = content.index(end_marker)

block = content[start_idx:end_idx]
lines = block.splitlines(keepends=True)

# Remove existing row for this repo (idempotent)
lines = [l for l in lines if f"[{repo}]" not in l]

# Separate header/separator from data rows
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

### 4. Commit and push (race-condition safe)

```bash
git add README.md
git commit -m "registry: update row for ${REPO}"

for i in 1 2 3 4; do
  git push origin main && break
  echo "Push failed (attempt $i), rebasing..."
  git pull --rebase origin main
  sleep $((i * 2))
done
```

**Conflict rule:** if rebase hits a conflict in the table, keep **both** rows —
yours updated, others untouched — then `git rebase --continue`.
Never resolve by deleting another repo's row.

---

## Verification checklist

- [ ] Table has header + separator + all previous rows + your (updated) row.
- [ ] Markers `PROJECTS:START/END` and `TECH:START/END` still exist.
- [ ] No duplicate rows for the same repo.
- [ ] No words "final / complete / closed" added anywhere.
- [ ] No other repo's rows were deleted or modified.
- [ ] Push succeeded (or rebase + re-push completed).

---

## Ready-to-use agent prompt

Copy and paste to any repo-agent that needs to update the registry:

```
You are the agent for repository <REPO_NAME> (UnlimitedEdition/<REPO_NAME>).
Your task: add or update this repo's row in the projects registry at
UnlimitedEdition/UnlimitedEdition (the profile repo), following EXACTLY the
protocol in CONTRIBUTING-REGISTRY.md.

Row values for this repo:
- Name:        <REPO_NAME>
- Description: <ONE-LINE, no markdown>
- Status:      <🟢 Aktivno | 🟡 Održavanje | 🔵 Eksperiment | ⚪ Arhivirano>
- Version:     <vX.Y.Z or omit>
- Tech:        <1–3 entries, e.g. "Python · FastAPI">
- Stars:       <number from gh repo view, or leave blank>

Steps:
1. Pull latest UnlimitedEdition/UnlimitedEdition (main branch).
2. Run the quick extractor to verify your values.
3. Upsert your row idempotently (alphabetical, keep all other rows intact).
4. Commit: "registry: update row for <REPO_NAME>"
5. Push with rebase-retry loop (max 4 attempts, backoff 2/4/8/16 s).
6. Run the verification checklist.

Never delete or modify another repo's row. Never mark anything as final/closed.
```

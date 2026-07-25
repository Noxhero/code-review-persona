# Code Review Persona Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code skill `code-review-persona` that dispatches 4 parallel subagents (security, perf, readability, 2-year-later beginner maintainer) over the local git diff and writes a persona-sectioned review report.

**Architecture:** Single-file skill (`SKILL.md` at repo root, no plugin manifest). The skill's own markdown instructions tell the invoking Claude how to: compute the diff via git, dispatch 4 `Agent` tool calls in parallel with fixed persona prompts, then aggregate the 4 text reports into `docs/reviews/YYYY-MM-DD-HHMM-persona-review.md` plus a condensed chat summary. No application code, no runtime, no automated test suite — this is pure prompt orchestration, "testing" means manually exercising the git commands and, at the end, running the finished skill against a real diff.

**Tech Stack:** Markdown (SKILL.md frontmatter + instructions), bash/git, Claude Code `Agent` tool.

## Global Constraints

- Repo root IS the skill directory — `SKILL.md` lives at `/home/matteo/Documents/GitHub/code-review-persona/SKILL.md`, no `.claude-plugin/plugin.json`.
- Diff source: `git diff <merge-base with main>` (branch commits + staged + unstaged combined). Fallback to `git diff HEAD` if no `main` branch resolvable, with a warning to the user.
- Empty diff → stop before dispatching any subagent.
- Exactly 4 personas, fixed, hardcoded in `SKILL.md`: Sécurité, Perf, Lisibilité, Débutant (maintenance 2 ans). Not configurable.
- Subagent dispatch: one message, 4 parallel `Agent` tool calls, `subagent_type: general-purpose`, `run_in_background: false` (must have all 4 results before aggregating). Each agent gets repo Read/Grep/Bash access, not diff-text-only.
- Output file path pattern: `docs/reviews/YYYY-MM-DD-HHMM-persona-review.md`, 4 sections `## Sécurité`, `## Perf`, `## Lisibilité`, `## Débutant (maintenance 2 ans)`, raw per-persona report, no merging/re-ranking across personas.
- Chat reply after file write: condensed summary (max 2-3 items per persona) + link to the file — never the full 4 reports duplicated in chat.
- No PR-mode, no persona config, no plugin packaging, no artifact/HTML output (all explicitly out of scope per spec).

---

### Task 1: Diff computation logic + error/fallback paths

**Files:**
- Create: `SKILL.md` (root) — only the frontmatter + "Étape 1: Calcul du diff" section for this task; later tasks append more sections to the same file.

**Interfaces:**
- Produces: a documented bash snippet block in `SKILL.md` under heading `## Étape 1 — Calcul du diff` that later tasks (2-4) will reference as "the diff computed in Étape 1". No function signatures involved (this is prompt text, not code) — the contract is: after this step, a shell variable/section produces either a diff to analyze, or a stop condition.

- [ ] **Step 1: Create `SKILL.md` with frontmatter and skeleton**

Write `SKILL.md`:

```markdown
---
name: code-review-persona
description: Simule 4 profils de reviewers (sécurité, perf, lisibilité, débutant qui maintiendra ce code dans 2 ans) sur le diff local courant, pour révéler des angles de critique différents avant une vraie review humaine. Utiliser quand le user demande /code-review-persona, une "review multi-angles", de "simuler plusieurs reviewers", ou de préparer une review avant de la soumettre à un humain.
---

# Code Review Persona

Simule 4 reviewers avec des angles différents sur le diff local courant.
Chaque persona analyse indépendamment (sans voir les avis des autres) pour
révéler des angles morts qu'une review unique manquerait. Ne remplace pas
une vraie review humaine — la précède.

## Étape 1 — Calcul du diff
```

- [ ] **Step 2: Manually verify the merge-base + diff commands in this repo**

Run each of these and note the actual output (this repo currently has one
commit on `main`, no divergent branch, so this exercises the fallback path):

```bash
git rev-parse --is-inside-work-tree
git merge-base main HEAD
git diff "$(git merge-base main HEAD)"
```

Expected: `is-inside-work-tree` → `true`. `merge-base main HEAD` → same
commit hash as `HEAD` (since we're on `main`), so `git diff` against it is
empty. This confirms the "on main, no divergence" case must fall through to
the empty-diff stop condition, not error out.

- [ ] **Step 3: Verify the no-`main`-branch fallback command**

```bash
git diff HEAD
```

Expected: empty output too (clean tree). This is the command to fall back
to when `git merge-base main HEAD` errors (e.g. no `main` branch exists,
different default branch name). Confirms the fallback command itself is
valid git syntax and behaves sanely on a clean tree.

- [ ] **Step 4: Write the full Étape 1 section into `SKILL.md`**

Append after the skeleton from Step 1:

````markdown
Déterminer le diff à analyser:

```bash
git rev-parse --is-inside-work-tree
```

- Si cette commande échoue (pas un repo git): arrêter, répondre "Pas un
  dépôt git, rien à review." et ne rien faire d'autre.

```bash
git merge-base main HEAD
```

- Si cette commande échoue (pas de branche `main`): fallback sur
  `git diff HEAD` (tout uncommitted vs HEAD) et prévenir l'utilisateur dans
  la réponse: "Pas de branche `main` trouvée, review du diff uncommitted
  seulement."
- Si elle réussit, utiliser:

```bash
git diff "$(git merge-base main HEAD)"
```

Ce diff couvre les commits de la branche courante depuis sa divergence
d'avec `main`, plus les changements staged et unstaged.

**Si le diff obtenu (par la voie normale ou le fallback) est vide**:
arrêter immédiatement, répondre "Rien à review, diff vide." et ne PAS
dispatcher les subagents de l'Étape 3.

**Si le diff est volumineux (plus de ~2000 lignes)**: continuer, mais
prévenir dans la réponse finale que l'analyse peut être partielle.
````

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "$(cat <<'EOF'
Add code-review-persona skill scaffold with diff computation step

Frontmatter, invocation description, and the git diff calculation
logic (merge-base against main, fallback to git diff HEAD, empty-diff
short-circuit) that later steps build on.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Persona definitions section

**Files:**
- Modify: `SKILL.md` (root) — append `## Étape 2 — Les 4 personas` section after Étape 1.

**Interfaces:**
- Consumes: nothing structurally from Task 1 beyond it being the preceding section in the same file.
- Produces: 4 named persona blocks (Sécurité, Perf, Lisibilité, Débutant (maintenance 2 ans)) that Task 3's subagent dispatch instructions will reference by these exact 4 names/headings.

- [ ] **Step 1: Append the persona definitions to `SKILL.md`**

```markdown
## Étape 2 — Les 4 personas

Ces 4 personas sont fixes, non configurables. Chacun ignore volontairement
les angles des autres pour rester focalisé.

### Sécurité

Cherche: injection (SQL/command/XSS), problèmes d'authentification et
d'autorisation, gestion des secrets (clés/tokens/mots de passe en dur ou
loggés), validation d'input manquante, désérialisation non sûre, OWASP top
10. Ignore le style et la performance sauf impact sécurité direct.

### Perf

Cherche: complexité algorithmique excessive, requêtes N+1, allocations ou
copies inutiles, boucles chaudes coûteuses, I/O bloquant évitable. Ignore
la sécurité et le style sauf impact perf direct.

### Lisibilité

Cherche: nommage peu clair, fonctions trop grosses ou à responsabilités
multiples, incohérence avec les conventions déjà présentes dans le repo,
manque de clarté sur l'intention du code. Ignore perf et sécurité sauf si
ça nuit directement à la lisibilité.

### Débutant (maintenance 2 ans)

Se met dans la peau de quelqu'un qui reprend ce code dans 2 ans, sans
contexte sur pourquoi il a été écrit ainsi. Cherche: dépendances implicites
non documentées, magic numbers non expliqués, pièges cachés (comportement
surprenant), absence de commentaire là où le WHY n'est pas évident. Ignore
perf et sécurité sauf si ça crée un piège de maintenance.
```

- [ ] **Step 2: Verify section renders as valid markdown**

```bash
grep -c "^### " SKILL.md
```

Expected: `4` (exactly 4 persona sub-headings under Étape 2).

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "$(cat <<'EOF'
Add the 4 fixed persona definitions to code-review-persona skill

Sécurité, Perf, Lisibilité, and Débutant (maintenance 2 ans), each
with an explicit "ignore X unless it bleeds into my lens" boundary
so the 4 parallel subagents don't produce redundant findings.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Subagent dispatch instructions

**Files:**
- Modify: `SKILL.md` (root) — append `## Étape 3 — Dispatch des 4 subagents` section.

**Interfaces:**
- Consumes: the 4 persona names/definitions from Task 2 (referenced by heading name), the diff produced by Task 1's Étape 1 logic.
- Produces: explicit instructions for the invoking Claude to make 4 parallel `Agent` tool calls with `subagent_type: general-purpose`, `run_in_background: false`, each returning a markdown findings report. Task 4's aggregation step consumes "4 markdown findings reports, one per persona, in the fixed order Sécurité/Perf/Lisibilité/Débutant".

- [ ] **Step 1: Append dispatch instructions to `SKILL.md`**

```markdown
## Étape 3 — Dispatch des 4 subagents

Envoyer UN SEUL message contenant 4 appels `Agent` en parallèle
(`subagent_type: general-purpose`, `run_in_background: false` — les 4
résultats sont nécessaires avant l'agrégation de l'Étape 4). Ne jamais
dispatcher séquentiellement.

Chaque agent reçoit un prompt autonome (il ne voit ni les autres personas
ni leurs résultats), construit ainsi:

1. Le diff calculé à l'Étape 1 (texte complet, pas un résumé)
2. Le chemin absolu du repo, avec instruction explicite qu'il peut lire
   d'autres fichiers du repo (Read/Grep/Bash) pour contexte — fichiers
   entiers touchés par le diff, conventions existantes, tests présents
3. La définition complète du persona assigné (copier le bloc correspondant
   de l'Étape 2), avec rappel explicite d'ignorer les autres angles
4. Le format de sortie attendu:
   - Liste de findings en markdown
   - Chaque finding: `fichier:ligne` — description du problème — suggestion
     concrète
   - Ordonné par importance décroissante selon SA lentille uniquement
   - Si rien à signaler: dire explicitement "Rien à signaler du point de
     vue [nom du persona]." plutôt que de forcer des findings artificiels
   - Pas de préambule, pas de résumé général, findings actionnables
     seulement

Si un agent échoue ou ne retourne rien d'exploitable: continuer avec les 3
autres, ne pas relancer, ne pas bloquer l'agrégation. Noter l'échec pour
l'Étape 4.
```

- [ ] **Step 2: Verify the section references all 4 persona names exactly**

```bash
grep -A2 "Étape 3" SKILL.md | grep -c "persona"
```

Expected: at least one match confirming the section is present and
non-empty (sanity check that Step 1's append succeeded and wasn't
truncated).

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "$(cat <<'EOF'
Add subagent dispatch instructions to code-review-persona skill

Specifies the single-message, 4-parallel-Agent-calls pattern with
general-purpose subagents, each isolated to its own persona lens with
repo read access, plus the per-finding output format and the
partial-failure handling (continue with whichever agents succeeded).

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Aggregation and output instructions

**Files:**
- Modify: `SKILL.md` (root) — append `## Étape 4 — Agrégation et sortie` section.

**Interfaces:**
- Consumes: the 4 markdown findings reports from Task 3 (fixed order: Sécurité, Perf, Lisibilité, Débutant), plus knowledge of any agent failures noted there.
- Produces: the final file-write and chat-summary behavior — nothing downstream consumes this (last step of the skill's instructions).

- [ ] **Step 1: Append aggregation instructions to `SKILL.md`**

```markdown
## Étape 4 — Agrégation et sortie

1. Créer le dossier `docs/reviews/` s'il n'existe pas.
2. Écrire `docs/reviews/YYYY-MM-DD-HHMM-persona-review.md` (date/heure
   réelles au moment de l'exécution) avec:

   ```markdown
   # Revue multi-persona — YYYY-MM-DD HH:MM

   Diff analysé:

   ```
   <sortie de `git diff --stat` sur le même diff que l'Étape 1>
   ```

   ## Sécurité

   <rapport brut de l'agent Sécurité, ou "_Analyse indisponible (échec
   agent)._" si cet agent a échoué>

   ## Perf

   <rapport brut de l'agent Perf, ou message d'échec>

   ## Lisibilité

   <rapport brut de l'agent Lisibilité, ou message d'échec>

   ## Débutant (maintenance 2 ans)

   <rapport brut de l'agent Débutant, ou message d'échec>
   ```

3. Ne jamais fusionner, re-classer ou dédupliquer les findings entre
   sections — chaque section reste le rapport brut de son agent.
4. Répondre dans le chat avec un résumé condensé: pour chaque persona, 2 à
   3 findings les plus importants max (pas le rapport complet), puis un
   lien vers le fichier écrit à l'étape 2 pour le détail complet.
```

- [ ] **Step 2: Verify the complete `SKILL.md` is well-formed**

```bash
grep -c "^## Étape" SKILL.md
```

Expected: `4` (Étape 1 through 4, confirming no section got lost across
the 4 tasks' appends).

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "$(cat <<'EOF'
Add aggregation and output instructions to code-review-persona skill

Completes SKILL.md: writes docs/reviews/<timestamp>-persona-review.md
with 4 raw per-persona sections (no cross-persona merging/reranking),
plus a condensed chat summary linking to the file.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: README and end-to-end validation

**Files:**
- Modify: `README.md` (root)
- Create (transient, for validation only, then delete): a throwaway change on a scratch branch to produce a real diff to run the finished skill against.

**Interfaces:**
- Consumes: the complete `SKILL.md` from Tasks 1-4.
- Produces: nothing further consumes this — final task.

- [ ] **Step 1: Write `README.md`**

```markdown
# code-review-persona

Claude Code skill that simulates 4 reviewer personas (security, perf,
readability, and a beginner maintaining this code in 2 years) over your
local git diff, before a real human review.

## Install

Clone or symlink this repo into `.claude/skills/code-review-persona/`
(project-level) or `~/.claude/skills/code-review-persona/` (personal).

## Use

Invoke `/code-review-persona` in Claude Code with uncommitted or
branch-ahead-of-main changes present. It writes a report to
`docs/reviews/<timestamp>-persona-review.md` and posts a condensed summary
in chat.

See `docs/superpowers/specs/2026-07-25-code-review-persona-design.md` for
the full design.
```

- [ ] **Step 2: Commit the README**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
Document code-review-persona install and usage in README

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 3: Create a throwaway diff to validate the finished skill against**

```bash
git checkout -b scratch/validation-diff
```

Add a deliberately flawed snippet so all 4 personas have something to
find, e.g. create `scratch_example.py`:

```python
import os

def get_user(user_id):
    query = "SELECT * FROM users WHERE id = " + user_id
    conn = os.popen("mysql -e '" + query + "'")
    result = conn.read()
    x = 86400
    for i in range(len(result)):
        for j in range(len(result)):
            pass
    return result
```

```bash
git add scratch_example.py
git commit -m "scratch: throwaway file for code-review-persona validation"
```

- [ ] **Step 4: Manually invoke the finished skill's instructions against this diff**

Follow `SKILL.md` step by step as if you were the invoking Claude: run the
Étape 1 git commands against this branch (should now yield a non-empty
diff since `scratch_example.py` is new relative to `main`), dispatch the 4
`Agent` calls per Étape 3 with the real diff, and produce the Étape 4
output file.

Verify:
- `docs/reviews/<timestamp>-persona-review.md` was created
- All 4 section headings present (`## Sécurité`, `## Perf`, `## Lisibilité`,
  `## Débutant (maintenance 2 ans)`)
- Sécurité section flags the SQL/command injection in `scratch_example.py`
- Perf section flags the O(n²) empty double loop and/or the magic number
- Débutant section flags the undocumented `x = 86400` magic number
- Chat summary is condensed (not a full dump of all 4 reports) and links
  to the file

- [ ] **Step 5: Clean up the scratch branch**

```bash
git checkout main
git branch -D scratch/validation-diff
```

This removes the throwaway branch and file — `scratch_example.py` and its
commit only ever existed on `scratch/validation-diff`, so deleting the
branch discards them entirely. The generated
`docs/reviews/<timestamp>-persona-review.md` from Step 4 stays on `main`
(untracked until committed) as proof the skill works end-to-end.

- [ ] **Step 6: Commit the validation report as evidence**

```bash
git add docs/reviews/
git commit -m "$(cat <<'EOF'
Add end-to-end validation report for code-review-persona skill

Generated by manually running the finished SKILL.md against a
throwaway diff with deliberate security/perf/readability/maintenance
issues, confirming all 4 persona sections fire correctly.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

## Self-Review Notes

- Spec coverage: Étape 1 (diff calc + fallback + empty-diff stop) → Task 1.
  Étape 2 (4 personas) → Task 2. Étape 3 (parallel dispatch, isolation,
  output format, partial failure) → Task 3. Étape 4 (file + chat summary,
  no merging) → Task 4. README/install → Task 5. End-to-end validation →
  Task 5. All spec sections covered.
- No placeholders: every step has literal file content or literal commands.
- Type/name consistency: the 4 persona names (Sécurité, Perf, Lisibilité,
  Débutant (maintenance 2 ans)) and the 4 section headings match exactly
  across Tasks 2-4.

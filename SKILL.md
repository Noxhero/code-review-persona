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

**Vérifier aussi les fichiers non trackés** (jamais `git add`-és, donc
absents du diff calculé ci-dessus):

```bash
git ls-files --others --exclude-standard
```

Si cette commande retourne des fichiers: ils ne sont PAS inclus dans le
diff et ne seront PAS reviewés. Prévenir explicitement dans la réponse
finale (lister ces fichiers) et suggérer `git add -N .` avant de relancer
le skill pour qu'ils soient pris en compte.

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

## Étape 3 — Dispatch des 4 subagents

Envoyer UN SEUL message contenant 4 appels `Agent` en parallèle
(`subagent_type: general-purpose`, `run_in_background: false` — les 4
résultats sont nécessaires avant l'agrégation de l'Étape 4). Ne jamais
dispatcher séquentiellement. Les 4 agents sont dispatchés un par persona,
dans cet ordre fixe: Sécurité, Perf, Lisibilité, Débutant (maintenance 2
ans).

Chaque agent reçoit un prompt autonome (il ne voit ni les autres personas
ni leurs résultats), construit ainsi:

1. Le diff calculé à l'Étape 1 (texte complet, pas un résumé)
2. Le chemin absolu du repo, avec instruction explicite qu'il peut lire
   d'autres fichiers du repo (Read/Grep/Bash) pour contexte — fichiers
   entiers touchés par le diff, conventions existantes, tests présents.
   Lecture seule: l'agent ne doit modifier, créer ou supprimer AUCUN
   fichier — il retourne uniquement son rapport en texte.
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

## Étape 4 — Agrégation et sortie

1. Créer le dossier `docs/reviews/` s'il n'existe pas.
2. Écrire `docs/reviews/YYYY-MM-DD-HHMM-persona-review.md` (date/heure
   réelles au moment de l'exécution, obtenues avec `date +%Y-%m-%d-%H%M`)
   avec:

   Le bloc "Diff analysé" contient la sortie de `git diff --stat` sur le
   même diff que l'Étape 1, c'est-à-dire:

   ```bash
   git diff --stat "$(git merge-base main HEAD)"
   ```

   ou, si le fallback de l'Étape 1 s'est déclenché:

   ```bash
   git diff --stat HEAD
   ```

   ````markdown
   # Revue multi-persona — YYYY-MM-DD HH:MM

   Diff analysé:

   ```
   <sortie de la commande git diff --stat correspondante ci-dessus>
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
   ````

3. Ne jamais fusionner, re-classer ou dédupliquer les findings entre
   sections — chaque section reste le rapport brut de son agent.
4. Répondre dans le chat avec un résumé condensé: pour chaque persona, 2 à
   3 findings les plus importants max (pas le rapport complet), puis un
   lien vers le fichier écrit à l'étape 2 pour le détail complet.

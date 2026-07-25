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

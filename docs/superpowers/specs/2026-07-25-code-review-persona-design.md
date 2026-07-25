# Design — Skill `code-review-persona`

Date: 2026-07-25
Status: Approuvé (design), en attente d'implémentation

## Objectif

Simuler plusieurs profils de reviewers humains sur un diff (sécurité, perf,
lisibilité, débutant qui maintiendra ce code dans 2 ans) pour obtenir des
angles de critique différents avant une vraie review humaine. But: exposer
des angles morts qu'une review unique manquerait, pas remplacer la review
humaine.

## Portée du repo

Ce repo EST le skill (pas un plugin multi-skills, pas de marketplace
packaging). Structure finale:

```
code-review-persona/
├── README.md
├── SKILL.md
└── docs/
    └── superpowers/
        ├── specs/
        └── plans/
```

`SKILL.md` à la racine. Installation = cloner/symlinker ce repo dans
`.claude/skills/code-review-persona/` (projet) ou
`~/.claude/skills/code-review-persona/` (perso). Pas de manifeste plugin
(`.claude-plugin/plugin.json`) — hors scope, non demandé.

## Invocation

- Nom: `code-review-persona`
- Trigger: `/code-review-persona` explicite, ou déclenchement naturel si le
  user demande une "review multi-angles" / "simuler plusieurs reviewers" /
  "avant une vraie review humaine".
- Aucun argument requis. Pas de config externe (les 4 personas sont fixes,
  codés en dur dans `SKILL.md`).

## Étape 1 — Calcul du diff

Diff local uniquement (pas de mode PR GitHub dans cette version).

```bash
git rev-parse --is-inside-work-tree   # vérifie qu'on est dans un repo git
git merge-base main HEAD              # point de divergence
git diff <merge-base>                 # commits de branche + staged + unstaged
```

Fallback:
- Pas de branche `main` (erreur merge-base, ex: repo mono-branche ou branche
  nommée différemment) → fallback `git diff HEAD` (tout uncommitted vs HEAD)
  + avertir l'utilisateur du fallback dans la réponse.
- Diff vide (aucune sortie) → stop immédiatement, message court "Rien à
  review, diff vide.", ne PAS dispatcher les subagents.
- Pas dans un repo git → message d'erreur clair, stop.
- Fichiers non trackés (`git ls-files --others --exclude-standard` non
  vide) → ces fichiers sont absents du diff (jamais `git add`-és) et ne
  seront pas reviewés; avertir l'utilisateur dans la réponse finale et
  suggérer `git add -N .` avant de relancer.

## Étape 2 — Les 4 personas (fixes)

Codés en dur dans le corps de `SKILL.md`, chacun avec: nom, angle, ce qu'il
ignore volontairement (pour éviter qu'un persona déborde sur le territoire
d'un autre).

1. **Sécurité** — injection (SQL/command/XSS), auth/authz, gestion secrets,
   validation input, désérialisation, OWASP top 10. Ignore style/perf.
2. **Perf** — complexité algorithmique, requêtes N+1, allocations/copies
   inutiles, boucles chaudes, blocking I/O. Ignore sécurité/style.
3. **Lisibilité** — nommage, taille/responsabilité des fonctions,
   cohérence avec conventions du repo, clarté de l'intention. Ignore
   perf/sécurité sauf si ça nuit directement à la lisibilité.
4. **Débutant (maintenance 2 ans)** — se met dans la peau de quelqu'un qui
   reprend ce code dans 2 ans sans contexte: dépendances implicites non
   documentées, magic numbers, pièges cachés, absence de commentaire sur un
   WHY non-évident, code qui semble correct mais surprend. Ignore
   perf/sécurité sauf si ça crée un piège de maintenance.

## Étape 3 — Exécution (subagents parallèles)

Un seul message contenant 4 appels `Agent` en parallèle, `subagent_type:
general-purpose`, `run_in_background: false` (on a besoin des 4 résultats
avant d'agréger). Les 4 agents sont dispatchés un par persona, dans cet
ordre fixe: Sécurité, Perf, Lisibilité, Débutant (maintenance 2 ans).

Chaque prompt d'agent reçoit, en autonome (les agents ne se voient pas
entre eux):
- Le texte complet du diff calculé à l'étape 1
- Le chemin du repo (accès Read/Grep/Bash pour explorer contexte: fichiers
  entiers touchés, conventions existantes, tests présents). Lecture seule:
  l'agent ne doit modifier, créer ou supprimer aucun fichier — il retourne
  uniquement son rapport en texte.
- La définition stricte de son persona et angle (ci-dessus), avec consigne
  explicite d'ignorer les autres angles
- Consigne de format de sortie: liste de findings en markdown, chacun avec
  `fichier:ligne`, description du problème, suggestion concrète. Ordonné par
  importance décroissante selon SA lentille. Si rien à signaler pour cette
  lentille: dire explicitement "Rien à signaler du point de vue [persona]."
  plutôt que de forcer des findings artificiels.
- Limite: rapport court, pas de préambule, findings actionnables seulement

## Étape 4 — Agrégation et sortie

Après réception des 4 rapports:

1. Écrire `docs/reviews/YYYY-MM-DD-HHMM-persona-review.md` avec:
   - En-tête: date, diff analysé (résumé — liste fichiers touchés + stat
     `git diff --stat`)
   - 4 sections `## Sécurité`, `## Perf`, `## Lisibilité`,
     `## Débutant (maintenance 2 ans)`, contenu = rapport brut de chaque
     subagent, sans fusion ni reclassement
2. Répondre dans le chat avec un résumé condensé (2-3 items max par persona,
   les plus importants) + lien vers le fichier complet. Pas de duplication
   intégrale des 4 rapports dans le chat.

## Gestion d'erreurs

| Cas | Comportement |
|---|---|
| Pas de repo git | Stop, message clair |
| Pas de branche `main` | Fallback `git diff HEAD`, avertir |
| Diff vide | Stop avant dispatch, message court |
| Un subagent échoue/timeout | Les 3 autres sections livrées quand même; section en échec marquée `_Analyse indisponible (échec agent)_` dans le fichier |
| Diff énorme (>~2000 lignes) | Avertir dans le chat que l'analyse peut être partielle/lente, mais continuer (pas de troncature silencieuse) |

## Testing

Pas de suite automatisée — le skill est de l'orchestration de prompts, pas
de logique unitairement testable. Validation = exécution manuelle sur un
diff réel (dans ce repo ou un autre) pendant le développement, vérification
que les 4 sections sont cohérentes, non redondantes, et que chaque persona
reste dans sa lentille.

## Hors scope (explicitement exclu, YAGNI)

- Mode PR GitHub (`gh pr diff`)
- Personas configurables/personnalisables
- Fusion des findings en une liste unique triée par sévérité
- Packaging plugin/marketplace
- Sortie artifact HTML

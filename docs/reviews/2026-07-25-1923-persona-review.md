# Revue multi-persona — 2026-07-25 19:23

Diff analysé:

```
 .gitignore         |   1 +
 README.md          |  21 +++++++-
 SKILL.md           | 148 +++++++++++++++++++++++++++++++++++++++++++++++++++++
 scratch_example.py |  11 ++++
 4 files changed, 180 insertions(+), 1 deletion(-)
```

## Sécurité

`scratch_example.py:5` — Command injection critique via `os.popen()` avec données non échappées — Remplacer par subprocess avec liste d'arguments (ex. `subprocess.run(["mysql", "-e", query], capture_output=True)`) ou un driver MySQL natif (pymysql, mysql-connector-python) qui évite l'interprétation shell.

`scratch_example.py:4` — SQL injection via concaténation directe de user_id — Utiliser des requêtes paramétrées : `"SELECT * FROM users WHERE id = %s"` avec paramètres liés au driver, jamais la concaténation de chaîne.

`scratch_example.py:3` — Pas de validation d'input sur user_id — Valider avant utilisation (ex. `int(user_id)` avec exception handling, ou regex si format alphanumérique attendu).

## Perf

`scratch_example.py:8-10` — Boucles imbriquées O(n²) qui ne font rien. Avec un `result` de taille n, cela force n² itérations vides, coût catastrophique. Supprimer ces boucles ou si elles ont un but caché, clarifier et implémenter l'algorithme réel.

`scratch_example.py:4-5` — I/O bloquant via `os.popen()` + shell incapacite la connection pooling et ajoute surcoût du processus shell. Utiliser `mysql.connector` ou `pymysql` pour une connexion directe au serveur.

`scratch_example.py:4` — `SELECT *` charge toutes les colonnes au lieu de spécifier celles nécessaires. Remplacer par les colonnes utiles pour réduire la bande passante réseau/disque.

`scratch_example.py:5` — `conn` n'est jamais fermé. Cela fuit des file descriptors. Utiliser un context manager (`with`) ou appeler `conn.close()` explicitement après usage.

## Lisibilité

`scratch_example.py:8-10` — Double boucle imbriquée vide (`for i in range(len(result)): for j in range(len(result)): pass`) — Destructrice de compréhension : ce code ne fait rien, son existence est totalement confuse, et les boucles utilisent un pattern Python non-idiomatique (`range(len())` au lieu d'itération directe) — Supprimer si c'est vraiment inutile, ou ajouter un commentaire explicite si cette boucle est intentionnelle (ex: placeholder pour un traitement futur).

`scratch_example.py:7` — Variable inutilisée `x = 86400` (magic number sans contexte) — Pollue la lecture de la fonction ; le sens de `86400` et son utilité sont totalement obscurs — Supprimer la variable si elle n'est pas utilisée, ou la renommer (`SECONDS_PER_DAY`, `CACHE_TTL`, etc.) + utiliser + ajouter un commentaire sur son rôle.

`scratch_example.py:3` — Nom de fonction `get_user(user_id)` trop vague — Ne révèle pas qu'on exécute une requête SQL via shell Unix ; un lecteur s'attend à une vraie interaction DB — Renommer en `query_user_via_mysql_shell()` ou ajouter un docstring clair : `"""Execute raw MySQL query to fetch user. Uses shell subprocess."""`

`scratch_example.py:4-6` — Pattern de construction et exécution de requête SQL peu clair : concaténation de strings + `os.popen()` — Approche legacy obscure (pourquoi pas SQLAlchemy, sqlite3, ou un vrai driver DB ?) — Ajouter un docstring au-dessus de la fonction expliquant ce choix legacy, ou refactoriser pour utiliser une API DB standard et clarifier l'intention.

## Débutant (maintenance 2 ans)

`scratch_example.py:7` — Variable `x = 86400` assignée mais jamais utilisée. Pourquoi cette constante existe-t-elle ? Sans commentaire, un mainteneur ne sait pas si c'est un WIP, une erreur, ou du code prévu mais oublié. — Ajouter un commentaire expliquant son rôle (ex: `# Seconds per day - used for [something]`), ou supprimer si inutile.

`scratch_example.py:8-10` — Deux boucles imbriquées qui itèrent sur les caractères de `result` mais ne font rien (`pass`). Semble être du code mort ou incomplet. — Clarifier si c'est intentionnel (ajouter un commentaire `# TODO: implement filtering`) ou supprimer entièrement.

`scratch_example.py:4` — Requête SQL construite par concaténation directe de strings (`"SELECT * FROM users WHERE id = " + user_id`). Aucun paramétrage, aucun échappement. Un mainteneur 2 ans plus tard ne saura pas pourquoi faire autrement. — Utiliser une lib Python native (ex: `pymysql`, `psycopg2`) avec parameterized queries, ou ajouter un commentaire expliquant le choix de cette approche.

`scratch_example.py:5` — Utilisation de `os.popen()` pour exécuter la CLI `mysql` directement depuis Python. Pourquoi ce pattern non-standard ? Aucun contexte. — Remplacer par une lib Python pour MySQL (ex: `pymysql.connect()`), qui est plus lisible et maintenable.

`scratch_example.py:5` — Le file descriptor retourné par `os.popen()` n'est jamais fermé (pas de `conn.close()`). Fuite de ressource implicite. — Ajouter `conn.close()` après `result = conn.read()`, ou utiliser un context manager (`with os.popen(...) as conn:`).

# artcraft-release-vault

Archives du **code source** des applications ArtCraft publiées par
l'organisation officielle [`storytold`](https://github.com/storytold).

Objectif : pouvoir **restaurer un dépôt entier** même si l'amont disparaît.

Chaque passage du workflow `archive-storytold-source.yml` crée, par dépôt, une
release `source/<app>/<sha7>` contenant :

| Fichier | Contenu |
|---|---|
| `<app>-<sha>.bundle` | bundle git **toutes refs** (branches + tags + historique complet) |
| `MANIFEST.json` | origine, HEAD, horodatage, nombre de refs, SHA-256, procédure de restauration |
| `refs.txt` | liste exhaustive des refs archivées |

## Restaurer

```bash
git clone photocraft-9a317f2.bundle photocraft
cd photocraft
git remote set-url origin https://github.com/<vous>/photocraft.git
git push --mirror origin     # recrée branches ET tags
```

Le bundle est vérifiable : `git bundle verify <fichier>.bundle`.

## Rétention

Par défaut, les 3 dernières archives de chaque application sont conservées
(paramètre `keep` du workflow).

Code source amont sous licence Apache-2.0 / MIT — voir les dépôts d'origine.

# artcraft-release-vault

Archives du **code source complet** et des **paquets `.deb` (linux-x86_64)** des
applications ArtCraft publiées par l'organisation officielle
[`storytold`](https://github.com/storytold).

Objectif : pouvoir **restaurer une application entière** — code, historique et
paquet — même si le dépôt ou la release d'origine disparaît.

---

## Ce que contient chaque archive

Une release par application et par version, sous le tag `source/<app>/<version>`
(par exemple `source/photocraft/v0.22.0`). Si l'application n'a pas de release
amont, le tag porte le SHA court du commit : `source/anicraft/86f74b7`.

| Fichier | Contenu |
|---|---|
| `<app>-<sha>.bundle` | bundle git **toutes refs** : branches, tags et `refs/pull`. Historique complet, objets inclus. |
| `<app>-<version>-linux-x86_64.deb` | paquet Debian de la release, tel que publié en amont |
| `MANIFEST.json` | origine, version, HEAD, horodatage, nombre de refs, SHA-256 du bundle et du `.deb`, procédures de restauration |
| `refs.txt` | liste exhaustive des refs archivées (une par ligne, avec son objet git) |

### Vérifications faites avant publication

- `git bundle verify` sur le bundle produit — échoue si une ref manque ;
- SHA-256 du `.deb` comparé au `SHA256SUMS.txt` officiel de la release amont —
  si les sommes divergent, le job échoue et rien n'est publié ;
- nombre de refs et de commits consigné dans `MANIFEST.json`.

---

## Restaurer

### Le code

```bash
git clone photocraft-v0.22.0.bundle photocraft
cd photocraft
git --git-dir=/dev/null bundle verify ../photocraft-v0.22.0.bundle   # contrôle
git tag | wc -l                                  # tags restaurés
```

Un `git clone` depuis un bundle ne récupère que `refs/heads` et `refs/tags`.
Les `refs/pull` sont dans le fichier mais demandent un refspec explicite :

```bash
git fetch ../photocraft-v0.22.0.bundle 'refs/pull/*:refs/pull/*'
```

Republication vers un dépôt vide :

```bash
git remote set-url origin https://github.com/<vous>/photocraft.git
git push --mirror origin     # recrée branches ET tags à l'identique
```

### Le paquet

```bash
sudo apt install ./photocraft-v0.22.0-linux-x86_64.deb
```

---

## Fonctionnement

### Déclenchement : sondage toutes les 15 minutes

Un vrai webhook est impossible : l'organisation `storytold` ne nous appartient
pas, on ne peut pas y installer de hook. Le workflow `archive-storytold-complete.yml`
sonde donc l'amont (`cron: */15 * * * *`) et compare la dernière release de
chaque application à ce qui est déjà archivé :

- **rien de nouveau** → sortie immédiate, aucun téléchargement ;
- **nouvelle release** → archivage du code + du `.deb` ;
- **bundle présent mais `.deb` manquant** → reprise de l'archivage.

Une version publiée en amont est donc capturée en général dans le quart d'heure.
Les crons GitHub sont *best effort* : un retard de quelques minutes est normal.
Le déclenchement manuel reste disponible (`Actions → Run workflow`).

### Nouvelles applications : aucune modification nécessaire

Le job `discover` liste les dépôts de l'organisation et ajoute automatiquement
toute application dont le nom suit l'une des deux conventions, à condition
qu'elle ne soit ni archivée ni un fork.

| Réglage | Valeur | Effet |
|---|---|---|
| `AUTO_DISCOVER` | `true` | découverte automatique activée |
| `APPS_PATTERN` | `^([a-z0-9]+craft\|craft-.+)$` | `photocraft` **ou** `craft-audio` |
| `APPS_EXCLUDE` | `artcraft artcraft-services artcraftx placeholder-artcraft` | dépôts exclus (monorepo et services, trop volumineux) |
| `DEFAULT_APPS` | 22 dépôts | liste toujours archivée, indépendante de la découverte |

Une application `craft-audio` publiée demain est donc archivée au passage
suivant, sans toucher à ce dépôt. Les dépôts vides (aucun commit) sont ignorés
avec un avertissement, sans faire échouer le run.

**Effet de bord assumé** : le motif `craft-.+` attrape aussi les dépôts
d'outillage de l'organisation (`craft-wiki`, `craft-common-actions`,
`craft-pr-review-tool`, `craft-repo-status-app`, `craft-ai-integrations`,
`craft-contrib`, `craft-extensions`, `craft-skills`). Ils sont archivés comme
les autres. Pour les écarter, ajoutez-les à `APPS_EXCLUDE` — c'est une liste
séparée par des espaces.

### Rétention : `keep = 1`

Seule la **dernière archive de chaque application** est conservée. À chaque
nouvel archivage réussi, les releases `source/<app>/*` plus anciennes sont
supprimées, tag git compris.

Le paramètre `keep` est modifiable au lancement manuel (`keep=12` pour un an
d'historique mensuel).

### Suppression en amont : les archives sont conservées

Trois cas, tous sans risque pour l'archive existante :

| Ce qui disparaît | Ce que fait le workflow |
|---|---|
| **Un dépôt entier** | `gh api` échoue → `::warning::… introuvable en amont`, aucune archive n'est produite ni supprimée. L'archive existante reste intacte. |
| **Une release** (le dépôt reste) | La clé bascule sur le SHA du commit, donc un nouveau tag `source/<app>/<sha7>` est créé — **et l'archive versionnée n'est pas purgée** : la rétention ne compare que des clés de même nature (version ↔ version, SHA ↔ SHA). |
| **Un commit** (dépôt vidé) | `rev-list --all` = 0 → `::warning::aucun commit en amont`, sortie propre. |

La purge de rétention ne s'exécute qu'**après un archivage réussi** et ne
touche jamais une clé d'une autre nature. Une application retirée de `storytold`
conserve donc sa dernière archive dans ce dépôt — c'est précisément la raison
d'être de ce coffre.

### Index

`source-index.json` est régénéré à chaque passage et committé sur `main` :
liste des archives avec version, taille du bundle et du `.deb`, et URL de
téléchargement. C'est le point d'entrée pour une synchronisation externe
(par exemple vers un AI Drive), sans avoir à interroger GitHub.

---

## Limites connues

| Élément | Statut |
|---|---|
| Binaires autres que le `.deb` x86_64 (`.dmg`, `.msi`, `.AppImage`, `.rpm`, `.flatpak`, aarch64, riscv64) | **non archivés** — choix explicite, seul x86_64 est conservé |
| Issues, pull requests, commentaires, revues | non archivés (métadonnées GitHub, pas des objets git) |
| Wiki, étoiles, réglages, secrets Actions | non archivés |
| Objets Git LFS | inclus uniquement s'ils sont matérialisés dans le dépôt |
| Signatures GPG des paquets | non vérifiées (seul le SHA-256 l'est) |

---

## Modifier le périmètre

| Besoin | Action |
|---|---|
| Archiver une application précise maintenant | `Actions → Run workflow` → `apps: photocraft` |
| Tout réarchiver sans condition | `Actions → Run workflow` → `force: true` |
| Changer la rétention | `Actions → Run workflow` → `keep: 5` |
| Ajouter un dépôt hors convention de nommage | ajouter son nom à `DEFAULT_APPS` |
| Archiver une autre architecture | modifier `DEB_ARCH` (`.deb` aarch64 / riscv64 disponibles en amont) |

---

Code source amont sous licence **Apache-2.0** (certains dépôts MIT) — voir le
`LICENSE` de chaque dépôt d'origine. Ce dépôt ne fait que conserver une copie ;
les droits restent ceux des auteurs.

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
| `<app>-<version>-linux-x86_64.deb` | paquet Debian de la release, quand elle en publie un |
| `MANIFEST.json` | origine, version, HEAD, horodatage, nombre de refs, SHA-256 du bundle et du `.deb`, procédures de restauration |
| `refs.txt` | liste exhaustive des refs archivées (une par ligne, avec son objet git) |

### Vérifications faites avant publication

- `git bundle verify` sur le bundle produit — échoue si une ref manque ;
- SHA-256 du `.deb` comparé au `SHA256SUMS.txt` officiel de la release amont —
  si les sommes divergent, le job échoue et rien n'est publié ;
- nombre de refs et de commits consigné dans `MANIFEST.json`.

Le `.deb` n'est exigé que si la release amont en publie un : les dépôts qui
n'ont pas de paquet Linux (`craft-i18n`, `craft-wiki`…) ne sont donc pas
repris indéfiniment.

---

## Restaurer

### Le code

```bash
git clone photocraft-v0.22.0.bundle photocraft
cd photocraft
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

## Ce qui est archivé, et pourquoi

### Périmètre : découverte automatique

Le job `discover` liste les dépôts de l'organisation et retient ceux qui
correspondent à l'une des deux conventions de nommage, ne sont ni archivés ni
des forks :

| Réglage | Valeur | Effet |
|---|---|---|
| `AUTO_DISCOVER` | `true` | découverte automatique activée |
| `APPS_PATTERN` | `^([a-z0-9]+craft\|craft-.+)$` | `photocraft` **ou** `craft-audio` |
| `APPS_EXCLUDE` | `artcraft artcraft-services artcraftx placeholder-artcraft` | dépôts exclus (monorepo et services, trop volumineux) |
| `DEFAULT_APPS` | 22 dépôts | liste toujours archivée, indépendante de la découverte |

Une application `craft-audio` publiée demain est archivée au passage suivant,
**sans modifier ce dépôt**. Les dépôts vides (aucun commit) sont ignorés avec un
avertissement, sans faire échouer le run.

**Effet assumé** : `craft-.+` attrape aussi l'outillage de l'organisation
(`craft-wiki`, `craft-common-actions`, `craft-skills`, `craft-contrib`,
`craft-extensions`, `craft-pr-review-tool`, `craft-repo-status-app`,
`craft-ai-integrations`). Ils sont archivés comme le reste. Pour les écarter,
les ajouter à `APPS_EXCLUDE`.

**Périmètre actuel : 30 dépôts archivés.**

### Les dépôts utilitaires sont-ils nécessaires à la reconstruction ?

Vérifié dans les `Cargo.toml` / `Cargo.lock` et la CI des applications :

| Dépôt | Rôle | Nécessaire pour **reconstruire** ? |
|---|---|---|
| `craft-libs` | crates partagées (`craft-ui`, `craft-update`, `craft-proof`, `craft-docpath`…) | **Oui — obligatoire.** Dépendance `git` épinglée par révision dans chaque application. |
| `craft-i18n` | données de langue (`language-data/`, `craft-i18n` crate) | **Oui.** Référencé par `Cargo.lock`, embarqué via `CRAFT_I18N_DIR`. |
| `craft-fonts` | polices et `craft-fonts-build` | **Oui** pour les builds qui activent l'option polices (`CRAFT_FONTS_DIR`). |
| `craft-common-actions` | workflow de gate partagé (`gate.yml`) | **Utile, pas de compilation** : sans lui la CI ne tourne pas, mais le code compile. |
| `craft-extensions` | thèmes, plugins, contrat de paquet | Non (runtime / distribution). |
| `craft-ai-integrations` | couche IA (connexions, fournisseurs, widgets) | Non — brique optionnelle. |
| `craft-wiki`, `craft-skills`, `craft-contrib`, `craft-pr-review-tool`, `craft-repo-status-app` | documentation et outillage de développement | Non. |

Donc : sur les 8 dépôts d'outillage, **3 sont indispensables** à une
reconstruction fidèle, 1 sert à la CI, 4 sont du confort.

### Angle mort comblé : les dépendances de build

Les applications dépendent de dépôts externes épinglés par révision dans leurs
`Cargo.toml`. Trois sont des **forks** de l'organisation — exclus de la
découverte automatique (qui écarte les forks) mais indispensables à un rebuild.
Ils sont donc archivés explicitement via `DEP_REPOS` :

| Dépôt | Applications dépendantes | Origine amont |
|---|---|---|
| `egui` | 16 | fork de `emilk/egui` |
| `winit` | 16 | fork de `rust-windowing/winit` |
| `winprint.rs` | 6 | fork de `ArcticLampyrid/winprint.rs` |
| `indd-utils` | designcraft | dépôt propre (pas un fork) |

Relevé fait dans les `Cargo.toml` et `Cargo.lock` des 18 applications : 14
références à `storytold/egui` et autant à `storytold/winit` par application.

`wgpu` et `spark` sont aussi des forks de l'organisation, mais **aucune
application ne les référence** : volontairement non archivés.

Pour ajouter une dépendance future : une ligne dans `DEP_REPOS` du workflow.

---

## Fonctionnement

### Déclenchement : sondage toutes les 15 minutes

Un vrai webhook est impossible : l'organisation `storytold` ne nous appartient
pas, on ne peut pas y installer de hook. Le workflow
`archive-storytold-complete.yml` sonde donc l'amont (`cron: */15 * * * *`) et
compare la dernière release de chaque application à ce qui est déjà archivé :

- **rien de nouveau** → sortie immédiate, aucun téléchargement ;
- **nouvelle release** → archivage du code + du `.deb` ;
- **release existante mais `.deb` manquant** → mise à jour de la release
  (`gh release upload --clobber`), pas de doublon.

Une version publiée en amont est donc capturée en général dans le quart d'heure.
Les crons GitHub sont *best effort* : un retard de quelques minutes est normal.
Le déclenchement manuel reste disponible (`Actions → Run workflow`).

### Rétention : `keep = 1`

Seule la **dernière archive de chaque application** est conservée. À chaque
archivage réussi, les releases `source/<app>/*` plus anciennes sont supprimées,
tag git compris.

La purge ne compare que des clés **de même nature** : une clé versionnée
(`v0.22.0`) ne peut pas purger une clé SHA (`86f74b7`) et inversement.

### Suppression en amont : les archives sont conservées

| Ce qui disparaît | Ce que fait le workflow |
|---|---|
| **Un dépôt entier** | `gh api` échoue → `::warning::… introuvable en amont`, aucune archive produite ni supprimée. L'archive existante reste intacte. |
| **Une release** (le dépôt reste) | La clé bascule sur le SHA du commit : un tag `source/<app>/<sha7>` est créé, **et l'archive versionnée n'est pas purgée**. |
| **Tous les commits** | `rev-list --all` = 0 → `::warning::aucun commit en amont`, sortie propre. |

La purge ne s'exécute qu'**après un archivage réussi**, et jamais sur une clé
d'une autre nature. Une application retirée de `storytold` conserve donc sa
dernière archive ici — c'est la raison d'être de ce coffre.

### Index

`source-index.json` est régénéré à chaque passage et committé sur `main` :
version, taille du bundle, taille du `.deb`, URL de téléchargement. C'est le
point d'entrée pour une synchronisation externe (par exemple vers un AI Drive),
sans interroger GitHub.

---

## Limites connues

| Élément | Statut |
|---|---|
| Binaires autres que le `.deb` x86_64 (`.dmg`, `.msi`, `.AppImage`, `.rpm`, `.flatpak`, aarch64, riscv64) | **non archivés** — seul x86_64 est conservé, par choix |
| Fork `storytold/egui` (dépendance de build) | **non archivé** — exclu par le filtre forks |
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

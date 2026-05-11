# CUSTOMIZATIONS — divergences avec upstream

> Inventaire **exhaustif** de chaque fichier upstream modifié ou ajouté sur la branche `custom/main`.
> Source de vérité pour anticiper les conflits lors d'une MAJ depuis `upstream/master`.
> **Règle :** tout commit qui touche un fichier upstream ou ajoute un fichier doit mettre à jour ce tableau dans le même commit.

Base upstream : **v3.6.5** (`96dfe0bea4`).

---

## Légende risque de conflit

- 🟢 **Faible** — fichier rarement modifié upstream, ou modif sur quelques lignes isolées.
- 🟡 **Moyen** — fichier touché de temps en temps upstream ; surveiller.
- 🔴 **Élevé** — fichier "chaud" upstream (commits fréquents) ; rebase délicat à chaque MAJ.

---

## Fichiers modifiés (upstream existant)

| Fichier | Lignes | Raison | Risque | Commit |
|---|---|---|---|---|
| [.gitignore](.gitignore) | +2 (`.claude/`, `CLAUDE.md`) | Ignorer fichiers de travail assistant. | 🟢 | `1580fc3c4b` |

## Fichiers ajoutés (n'existent pas upstream)

Aucun fichier custom ajouté pour l'instant. Les fichiers `CLAUDE.md`, `.claude/`, `CUSTOMIZATIONS.md` lui-même : voir section suivante.

## Fichiers non-trackés (gitignored)

Présents en local mais hors versionning, donc **zéro risque de conflit** :
- `CLAUDE.md`
- `.claude/` (toute la doc d'assistance)

`CUSTOMIZATIONS.md` est tracké (sert de référence projet).

---

## Convention pour les modifs futures

### Préférer dans l'ordre

1. **Plugin Siyuan** ([app/src/plugin/API.ts](app/src/plugin/API.ts)) — zéro conflit upstream.
2. **Nouveaux fichiers dans des dossiers `custom/`** — zéro conflit (sauf si upstream crée un dossier homonyme, improbable).
3. **Modif chirurgicale d'un fichier upstream** avec marqueurs :
   ```ts
   // CUSTOM-START: <raison courte, ex: "expose getActiveBox to plugins">
   ...code custom...
   // CUSTOM-END
   ```
   Marqueurs identiques en Go : `// CUSTOM-START:` / `// CUSTOM-END`.

### À éviter

- Refactor de masse sur fichiers upstream.
- Renommage de symboles upstream (casse imports custom à la prochaine MAJ).
- Suppression de code upstream "mort selon nous" — il pourrait revenir dans une MAJ.

### Procédure de modif d'un fichier upstream

1. Vérifier qu'on ne peut pas le faire via plugin.
2. Ajouter les marqueurs `CUSTOM-START` / `CUSTOM-END`.
3. Ajouter une ligne dans ce tableau (fichier, lignes, raison, risque, commit).
4. Commit unique : message clair, scope limité.

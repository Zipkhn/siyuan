# Architecture V1 — Spec figée

> Document de référence verrouillé pour la V1 du projet "Siyuan back-office + reader companion".
> Trois sections : (1) schéma de publication d'un document, (2) format du snapshot produit par l'extracteur, (3) modèle de données du reader.

## Rappel des choix de cadrage V1

- Granularité de publication : **document entier** (pas bloc-par-bloc, pas box entière).
- Onboarding lecteur : **invitation par email + magic-link** (pas de code projet, pas d'auto-signup).
- Recherche : **navigation + sommaire + recherche simple** sur les documents autorisés.
- Historique : **version actuelle uniquement**, pas d'historique visible.
- Commentaires : aucun en V1 (read-only).
- Branding : sobre et neutre, pas de personnalisation par client.
- Routing : sous-chemin `reader.<domaine>/<project_slug>` (sous-domaines/CNAME plus tard).
- Mise à jour : webhook depuis le plugin + bouton manuel ; polling léger en fallback.
- Assets : copiés côté extracteur/reader, pas de proxy direct Siyuan.
- Hébergement : VPS perso, Docker Compose.
- Stacks : reader = Next.js + Auth.js ; extracteur = Node.js/TypeScript.
- AGPL : validation juridique en parallèle, obligatoire avant ouverture externe ; documenter les modifs du fork (déjà via `CUSTOMIZATIONS.md`) et les interactions réseau.

---

## 1. Schéma de publication d'un document

### Principe
Marquer un **document entier** comme publié = positionner des attributs IAL sur son **bloc racine** (type `NodeDocument`).

### Attributs IAL portés par le bloc racine du doc

| Attribut | Type | Obligatoire | Description |
|---|---|---|---|
| `custom-published` | `"true"` ou absent | oui (pour publier) | `"true"` = doc publié. Absent / autre valeur = non publié. |
| `custom-project` | slug, regex `^[a-z0-9-]+$` | oui | Slug du projet dans le reader. Sert au routing `/<project>/<doc>` et aux ACL. |
| `custom-publish-slug` | slug, regex `^[a-z0-9-]+$` | non | Slug du doc dans le reader. Si absent, fallback sur `slugify(titre)`. |
| `custom-publish-version` | int (compteur) | non (auto) | Incrémenté par le plugin à chaque publication. Sert d'idempotence et de cache-bust. |
| `custom-publish-updated-at` | ISO 8601 | non (auto) | Timestamp dernière publication. Posé par le plugin. |

### API Siyuan utilisées (côté plugin)
- `POST /api/attr/setBlockAttrs` — pose les attrs.
- `POST /api/attr/getBlockAttrs` — lecture.
- (le plugin n'appelle pas l'extracteur via Siyuan : c'est un appel HTTP direct.)

### Workflow "Publier un doc" (côté plugin)
1. L'utilisateur clique "Publier" sur un doc (menu contextuel).
2. Dialogue : choisir le projet (slug), confirmer le slug du doc.
3. Le plugin valide les slugs (regex), incrémente `custom-publish-version`, set tous les attrs via `setBlockAttrs`.
4. Le plugin POST sur le webhook de l'extracteur :
   ```json
   { "event": "publish", "project": "<slug>", "doc_id": "<root-id>", "version": <n> }
   ```
5. Le plugin affiche le statut (succès / erreur réseau).

### Workflow "Dépublier"
1. Bouton "Retirer la publication".
2. Plugin retire l'attr `custom-published` (set à `""`) sur le bloc racine.
3. Plugin POST webhook : `{ "event": "unpublish", "project": "...", "doc_id": "..." }`.
4. L'extracteur supprime le snapshot et l'entrée DB côté reader.

### Règles de rendu V1
- **Liens vers docs non publiés** (du même projet ou d'un autre projet) : rendus en texte brut, lien neutralisé.
- **Block refs `((id 'alias'))`** : si la cible est dans un doc publié du **même projet**, lien fonctionnel ; sinon texte brut.
- **Embed blocks (`{{select ...}}`)** : non supportés V1, ignorés (bloc absent du snapshot).
- **Assets** : extraits, hashés (sha256), copiés dans le storage du reader. Le snapshot référence par `stored_path`.
- **Attributs IAL `custom-*`** : non inclus dans le snapshot, sauf liste blanche explicite (à étendre plus tard).

---

## 2. Format du snapshot

### Choix
**JSON** comme source de vérité unique. Le reader render le JSON → HTML au moment du rendu (Next.js SSR/ISR). Aucun HTML pré-rendu en V1.

Bénéfices : présentation modifiable sans re-extraction, indexation directe, stockage compact.

### Schéma d'un snapshot doc (`snapshots/<project>/docs/<doc_id>.json`)

```jsonc
{
  "schema": "siyuan-snapshot/v1",
  "doc": {
    "id": "20240101120000-abc1234",
    "project": "acme",
    "slug": "guide-utilisation",
    "title": "Guide d'utilisation",
    "published_at": "2026-05-11T12:34:56Z",
    "updated_at": "2026-05-10T09:00:00Z",
    "version": 17,
    "excerpt": "Premiers ~200 caractères de texte plat pour listing et meta description."
  },
  "content": {
    "blocks": [
      { "id": "...", "type": "NodeHeading", "level": 1, "text": "Titre principal", "children": [] },
      { "id": "...", "type": "NodeParagraph", "text": "...", "marks": [], "children": [] }
    ]
  },
  "assets": [
    {
      "original_path": "assets/diagram-abc.png",
      "stored_path": "acme/assets/diagram-abc.a1b2c3d4.png",
      "sha256": "a1b2c3d4...",
      "mime": "image/png",
      "size_bytes": 12345
    }
  ],
  "outbound_refs": [
    { "target_doc_id": "20240202130000-def5678", "target_block_id": null, "anchor_text": "voir le guide d'installation" }
  ],
  "search_text": "Concaténation plain-text de tous les blocs pour ingestion dans un index full-text."
}
```

### Types de blocs supportés en V1

| Type Siyuan | Champs spécifiques | Notes |
|---|---|---|
| `NodeParagraph` | `text`, `marks[]` | `marks` = formatages inline (gras, italique, code, lien). |
| `NodeHeading` | `level` (1-6), `text` | |
| `NodeList` | `ordered` (bool), `children` (items) | |
| `NodeListItem` | `children` | |
| `NodeCodeBlock` | `language`, `text` | Coloration côté reader. |
| `NodeBlockquote` | `children` | |
| `NodeTable` | `header_row` (int), `rows[][]` (cellules : text + marks) | |
| `NodeMathBlock` | `text` (LaTeX) | Rendu KaTeX côté reader. |
| `NodeThematicBreak` | — | `<hr>` |
| `NodeImage` (inline/bloc) | `asset_id` (référence assets[]), `alt`, `caption` | |
| `NodeAttributeView` | `view_type` (`"table"` \| `"gallery"`), `columns[]`, `rows[]` | Vue figée au moment du snapshot. |
| `NodeSuperBlock` | `layout` (`"row"` \| `"col"`), `children` | |

Types non listés → **bloc ignoré** dans le snapshot V1 (avec log côté extracteur pour audit).

### Format `marks` (formatages inline)
```jsonc
{
  "type": "NodeParagraph",
  "text": "Bonjour le monde",
  "marks": [
    { "type": "strong", "start": 0, "end": 7 },
    { "type": "link", "start": 8, "end": 16, "href": "/acme/autre-doc" }
  ]
}
```
Types `marks` V1 : `strong`, `em`, `code`, `link`, `strike`.

### Stockage filesystem (côté extracteur ET côté reader)

```
snapshots/                                  # racine snapshots, monté en volume Docker
└── <project>/
    ├── index.json                         # liste des docs publiés du projet
    ├── docs/
    │   └── <doc_id>.json                  # un fichier par doc
    └── assets/
        └── <basename>.<sha256-12chars>.<ext>
```

### `index.json` par projet
```jsonc
{
  "project": "acme",
  "name": "Acme Corp",
  "updated_at": "2026-05-11T12:34:56Z",
  "docs": [
    {
      "id": "20240101120000-abc1234",
      "slug": "guide-utilisation",
      "title": "Guide d'utilisation",
      "excerpt": "...",
      "published_at": "2026-05-11T12:34:56Z",
      "updated_at": "2026-05-10T09:00:00Z"
    }
  ]
}
```

### Atomicité d'écriture
L'extracteur écrit dans un fichier temporaire (`*.tmp` dans le même dossier) puis fait un `rename()` atomique. Le reader ne lit jamais un fichier partiel.

### Suppression
- Unpublish → supprime `<doc_id>.json` + entrée dans `index.json` (réécriture atomique).
- Assets orphelins : un job de GC périodique côté extracteur (hors V1, à prévoir mais pas implémenté).

---

## 3. Modèle de données du reader

Stockage relationnel. **SQLite** en V1 (≤ 20 lecteurs, ~quelques centaines de docs : largement dimensionné, zéro ops). Migration Postgres plus tard si besoin via Drizzle/Prisma agnostiques.

### Table `users` (gérée par Auth.js)
| Colonne | Type | Notes |
|---|---|---|
| `id` | TEXT PK (cuid) | Auto Auth.js. |
| `email` | TEXT UNIQUE NOT NULL | |
| `name` | TEXT NULL | |
| `email_verified` | TIMESTAMP NULL | |
| `image` | TEXT NULL | |
| `created_at` | TIMESTAMP DEFAULT now() | |

Auth.js scaffold aussi `accounts`, `sessions`, `verification_tokens` (gérées par l'adapter Drizzle/Prisma, non détaillées ici).

### Table `projects`
| Colonne | Type | Notes |
|---|---|---|
| `id` | TEXT PK (cuid) | |
| `slug` | TEXT UNIQUE NOT NULL | Match `custom-project` côté Siyuan. Regex `^[a-z0-9-]+$`. |
| `name` | TEXT NOT NULL | Nom affiché. |
| `description` | TEXT NULL | |
| `created_at` | TIMESTAMP DEFAULT now() | |
| `updated_at` | TIMESTAMP | MAJ par l'extracteur quand un doc du projet bouge. |

### Table `documents`
| Colonne | Type | Notes |
|---|---|---|
| `id` | TEXT PK (cuid) | |
| `project_id` | FK projects | ON DELETE CASCADE. |
| `siyuan_id` | TEXT NOT NULL | ex: `20240101120000-abc1234`. |
| `slug` | TEXT NOT NULL | Sert à l'URL `/<project_slug>/<slug>`. |
| `title` | TEXT NOT NULL | |
| `excerpt` | TEXT | Premiers ~200 chars pour listing/meta. |
| `snapshot_path` | TEXT NOT NULL | Chemin relatif au fichier JSON depuis la racine `snapshots/`. |
| `version` | INT NOT NULL | Copie de `doc.version`. |
| `published_at` | TIMESTAMP NOT NULL | |
| `updated_at` | TIMESTAMP NOT NULL | |

Index :
- `UNIQUE (project_id, siyuan_id)`
- `UNIQUE (project_id, slug)`
- `INDEX (project_id, updated_at)`

### Table `user_projects` (ACL)
| Colonne | Type | Notes |
|---|---|---|
| `user_id` | FK users | ON DELETE CASCADE. |
| `project_id` | FK projects | ON DELETE CASCADE. |
| `role` | TEXT NOT NULL DEFAULT `'viewer'` | enum V1 : `'viewer'` uniquement. Champ gardé pour évolutions. |
| `granted_at` | TIMESTAMP DEFAULT now() | |
| `granted_by` | FK users NULL | Trace admin pour audit. |
| **PK** | (`user_id`, `project_id`) | |

### Recherche

Décision V1 : **SQLite FTS5** virtual table.

```sql
CREATE VIRTUAL TABLE documents_fts USING fts5(
  document_id UNINDEXED,
  title,
  excerpt,
  search_text,
  content='',
  tokenize='unicode61 remove_diacritics 2'
);
```

L'extracteur (ou un job du reader déclenché par webhook) met à jour cette table à chaque snapshot écrit. La query :
```sql
SELECT d.* FROM documents d
JOIN documents_fts ft ON ft.document_id = d.id
JOIN user_projects up ON up.project_id = d.project_id
WHERE up.user_id = ?
  AND documents_fts MATCH ?
ORDER BY rank;
```

### Queries types

```sql
-- Projets visibles par un user
SELECT p.* FROM projects p
JOIN user_projects up ON up.project_id = p.id
WHERE up.user_id = ?;

-- Docs d'un projet pour un user
SELECT d.* FROM documents d
JOIN projects p ON p.id = d.project_id
JOIN user_projects up ON up.project_id = p.id
WHERE p.slug = ? AND up.user_id = ?
ORDER BY d.updated_at DESC;

-- Garde d'accès doc (middleware)
SELECT 1 FROM documents d
JOIN projects p ON p.id = d.project_id
JOIN user_projects up ON up.project_id = p.id AND up.user_id = ?
WHERE p.slug = ? AND d.slug = ?
LIMIT 1;
```

### Flow d'auth (Auth.js, magic-link)

1. Visiteur va sur `reader.<domaine>/login`.
2. Il saisit son email.
3. **Vérif pré-requis** : l'email doit déjà exister dans `users` (créé via flow d'invitation admin). Sinon refus avec message "Aucune invitation trouvée. Contactez votre administrateur."
4. Auth.js envoie le magic-link.
5. Clic → session créée → redirect vers landing (liste des projets autorisés du user).

### Flow d'invitation (admin)

Endpoint serveur Next.js réservé à l'admin (toi) :
```
POST /api/admin/invite
Body: { email: "...", project_slug: "...", send_invite_email: true|false }
```
1. Crée l'user (ou récupère si existe).
2. Crée l'entrée `user_projects` (idempotent).
3. Si `send_invite_email` : envoie un email de bienvenue avec le lien `reader.<domaine>/login`.

Pas d'auto-signup. Pas d'inscription depuis l'UI publique.

---

## 4. Hors scope V1 (verrouillé)

- Granularité bloc / box.
- Branding par client.
- Sous-domaines / CNAME.
- Commentaires / annotations.
- Historique de versions visible côté lecteur.
- Cross-project refs.
- Embed blocks (`{{select ...}}`).
- Plugins de présentation côté reader.
- Multi-langue côté reader.
- Mobile-first / PWA (responsive de base oui, PWA non).

---

## 5. Évolutions probables post-V1 à garder en tête

- Granularité bloc (publier un fragment d'un doc).
- Branding par projet (logo, couleurs, domaine).
- Comments / annotations (channel feedback lecteur → admin).
- Audit log côté reader (qui a lu quoi, quand).
- API publique reader pour intégrations tierces.
- Recherche externalisée (Meilisearch / Typesense) si volumes grossissent.
- Multi-tenant kernel-side si exigence légale (RGPD, secret pro) impose isolation physique.

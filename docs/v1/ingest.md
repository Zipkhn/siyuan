# Frontière extractor → reader — Amendement V1 (mode Vercel)

> Document de référence verrouillé pour la frontière HTTP entre l'**extractor** (sur la machine de l'utilisateur) et le **reader** (déployé sur Vercel).
>
> Ce document **amende** [`architecture.md`](architecture.md) §3 (modèle de données reader) et §4 (extracteur — décisions techniques) pour le scénario de déploiement Vercel. Tout le reste de la spec V1 (format snapshot canonique JSON, schéma des blocs, ACL `user_projects`, magic-link, branding, etc.) **reste valide sans changement**.
>
> **Statut : figé avant codage des endpoints.** Toute évolution de cette frontière doit faire l'objet d'un amendement explicite dans ce fichier.

---

## 0. Pourquoi cet amendement existe — la rupture avec "filesystem only"

La spec V1 (`architecture.md` §4) tranche :

> Sink V1 : **filesystem only**. Pas d'écriture directe dans la DB du reader. Le reader ingère depuis les fichiers via watcher / au boot.

Cette décision est valide pour le déploiement **Docker local tout-en-un** (extractor + reader sur la même machine, volume nommé partagé `snapshots:/data/snapshots`).

Elle **ne tient plus** pour le déploiement **reader sur Vercel** :

- Vercel = serverless = **pas de filesystem persistant** côté serveur. Toute écriture sur disque est éphémère, perdue à froid.
- L'extractor reste **sur la machine de l'utilisateur** (il doit joindre Siyuan via `localhost:6806` avec le token API local).
- Il n'y a donc plus de volume partagé possible entre les deux — le pont devient **nécessairement HTTP**.

Donc, en mode Vercel, on remplace :

```
extractor ─► filesystem ◄─ reader
```

par :

```
extractor (local) ─HTTPS─► reader (Vercel) ─► DB Turso
```

**Rupture explicite et assumée** : "filesystem only" reste cohérent pour le mode Docker local (utile pour le dev) ; "push HTTP vers DB" devient la frontière du mode Vercel. Les deux modes peuvent coexister à long terme dans le code, mais **en V1 on investit uniquement sur le mode Vercel côté reader**. Le mode tout-local Docker n'est plus considéré comme un déploiement cible — il reste un confort de développement.

---

## 1. Source de vérité — DB Turso unique en mode Vercel

En mode Vercel, la **DB Turso devient l'unique source de vérité côté reader**. Conséquences fermes :

- **Plus de `SNAPSHOTS_DIR` côté reader.** Plus de lecture de fichiers sur disque.
- Le **JSON canonique du snapshot** est stocké en colonne TEXT (`documents.snapshot_json`).
- Le **HTML sanitisé sibling** est stocké en colonne TEXT (`documents.html`).
- Les **bytes des assets** sont stockés en BLOB dans une table dédiée `assets`, content-addressed sur sha256.
- Les colonnes plates dérivées (title, slug, excerpt, content_hash, version, ...) restent en DB comme aujourd'hui.
- Le module `siyuan-reader/src/snapshots/fs.ts` **disparaît** (toutes les lectures passent par SQL).
- Le module `siyuan-reader/src/admin/sync.ts` (réconciliation FS↔DB) **disparaît** (l'ingestion HTTP écrit directement la DB ; il n'y a plus rien à réconcilier).
- La variable d'environnement `SNAPSHOTS_DIR` **disparaît** du reader.

### Compromis V1 explicitement reconnu : BLOB en DB ≠ cible long terme

Le stockage des bytes d'assets en BLOB Turso est un **compromis V1 assumé**, **pas la cible long terme**.

Raison du compromis : un seul tier de stockage, un seul backup, pas de service externe à provisionner, secret unique. C'est suffisant pour la V1 (volume faible, images de doc, plan gratuit Turso = 9 GB).

À terme — V2, ou plus tôt si le volume l'impose — les assets sortiront vers un **Blob storage externe** (Vercel Blob, Cloudflare R2, ou S3). La table `assets` ne contiendra alors plus que des métadonnées (sha256, url, mime, size_bytes), les bytes vivront ailleurs.

**Le contrat HTTP est conçu pour absorber ce changement sans casser la frontière extractor ↔ reader** : l'extractor envoie toujours des bytes via `POST /api/ingest/asset`, c'est le reader qui décide en interne où il les pose. Le jour où on bascule sur Blob storage, l'extractor n'a rien à changer.

---

## 2. Mode d'ingestion — push synchrone HTTP

| Aspect | Décision V1 |
|---|---|
| Direction | **Push** : extractor → reader. Pas de pull, pas de polling, pas de queue. |
| Transport | **HTTPS** uniquement. JSON pour les documents, `application/octet-stream` pour les bytes d'asset. |
| Mode | **Synchrone** (request/response). L'extractor attend la réponse HTTP avant d'acquitter l'event au plugin Siyuan. |
| Granularité | **Un document par requête**. Pas de batch. Cohérent avec la granularité V1 ("document entier", `architecture.md` §1). |
| Producteur | **Unique** : un extractor par utilisateur (sa machine), un reader Vercel par déploiement. |

Pourquoi pas d'asynchrone / queue en V1 : volume faible (clics manuels), pas de pic, debug immédiat, erreurs visibles côté plugin Siyuan. Si un jour le besoin émerge : on intercale Inngest / SQS / une table jobs sans changer le contrat HTTP de l'extractor.

---

## 3. Endpoints exposés côté reader

Trois endpoints sous `/api/ingest/*`. Vue d'ensemble (le contrat précis — Zod schemas, statuts détaillés, exemples — sera figé dans le document suivant, après validation de cette frontière) :

| Méthode | Path | Rôle |
|---|---|---|
| `POST` | `/api/ingest/doc` | Upsert d'un document (JSON canonique + HTML sibling + refs vers assets par sha256). |
| `HEAD` | `/api/ingest/asset/:sha256?project=…` | Présence d'un asset content-addressed. |
| `POST` | `/api/ingest/asset?project=…&sha256=…&mime=…` | Upload d'un asset binaire (body `application/octet-stream`). |
| `POST` | `/api/ingest/unpublish` | Suppression idempotente d'un document. |

Servi en plus (côté lecture publique, pas ingestion) :

| Méthode | Path | Rôle |
|---|---|---|
| `GET` | `/<project>/assets/:sha256` | Servir un asset au client final, avec ACL via session Auth.js + `user_projects`. |

---

## 4. Idempotence — clé exacte `(project, docId)` + `content_hash` + `version`

Règle exacte, à respecter dans l'implémentation :

1. **Clé d'identification** d'un document côté ingestion : couple `(project_slug, doc_id)`.
2. Le payload `POST /api/ingest/doc` inclut **toujours** ces deux champs :
   - `content_hash` : sha256 hex du JSON canonique des blocs (déjà calculé par l'extractor, cf. [`siyuan-extractor/src/snapshot.ts`](../../../siyuan-extractor/src/snapshot.ts) `computeContentHash`).
   - `doc.version` : entier incrémenté par le plugin Siyuan à chaque publication (cf. attribut IAL `custom-publish-version`, `architecture.md` §1).
3. Comportement reader à la réception d'un `POST /api/ingest/doc` :
   - Lookup par `(project_id, siyuan_id)` (UNIQUE existant dans le schéma).
   - **Si row existe ET `content_hash` identique ET `version` identique** → `200 { status: "unchanged" }`. **Aucune écriture, aucune touche FTS, aucun bump `updated_at`.**
   - **Sinon** → upsert (insert si nouveau, update si différent) + re-index FTS5, **dans la même requête**. `200 { status: "ingested" }`.
4. Comportement reader à la réception d'un `POST /api/ingest/asset` :
   - Lookup par `(project_id, sha256)` (UNIQUE).
   - **Si l'asset existe** → `200 { status: "exists" }`. **Aucune écriture.**
   - **Sinon** → insert avec bytes. `200 { status: "ingested" }`.
5. Comportement reader à la réception d'un `POST /api/ingest/unpublish` :
   - Lookup par `(project_id, siyuan_id)`.
   - **Si la row n'existe pas** → `200 { status: "already_absent" }`. **Jamais `404`.** (Un retry après timeout réseau ne doit pas faire échouer le plugin Siyuan à tort.)
   - Sinon → `DELETE` du doc + `DELETE FROM documents_fts`. `200 { status: "removed" }`.

**Conséquence pratique** : **l'extractor peut retry à l'aveugle**. Le second appel sera soit `unchanged`/`exists`/`already_absent` (rapide, idempotent), soit une rejouée propre. Pas de risque de doublon, pas de risque d'état corrompu après timeout.

**Pourquoi cette règle exacte** : le `content_hash` seul ne suffit pas (le plugin pourrait re-publier la même version après une retouche minuscule, on veut détecter le changement de `version` même si le contenu est identique — utile pour cache-bust côté reader). Inversement, la `version` seule ne suffit pas (deux publications consécutives avec la même version mais un contenu changé doivent être détectées). Les deux ensemble garantissent : *pas d'écriture inutile*, *pas d'écriture manquée*.

---

## 5. Gestion des assets — content-addressed, BLOB en DB pour V1

### Flux extractor lors d'un publish

1. Extraire la liste des assets référencés par le doc (chemins type `assets/diagram-abc.png`).
2. Pour chacun, fetcher les bytes via Siyuan + calculer sha256 (déjà fait, cf. [`siyuan-extractor/src/assets.ts`](../../../siyuan-extractor/src/assets.ts)).
3. Pour chacun, `HEAD /api/ingest/asset/<sha256>?project=<slug>` pour savoir s'il est connu côté reader.
4. Pour les inconnus uniquement : `POST /api/ingest/asset?project=<slug>&sha256=<hex>&mime=<type>` avec body = bytes.
5. Une fois tous les assets confirmés présents : `POST /api/ingest/doc` qui référence les assets par sha256.

### Garde-fou côté reader

Un `POST /api/ingest/doc` qui référence un sha256 inconnu pour ce projet → **`409 missing_asset`** avec le sha256 fautif. L'extractor doit alors uploader les assets manquants et retry une fois (cf. §8).

### Stockage V1

Table `assets` :

| Colonne | Type | Notes |
|---|---|---|
| `id` | TEXT PK (uuid) | |
| `project_id` | FK projects | `ON DELETE CASCADE`. |
| `sha256` | TEXT NOT NULL | hex 64 chars. |
| `mime` | TEXT NOT NULL | |
| `size_bytes` | INTEGER NOT NULL | |
| `bytes` | BLOB NOT NULL | les bytes réels. **V1 only — voir §1 sur le compromis.** |
| `created_at` | TIMESTAMP | |

Index :
- `UNIQUE (project_id, sha256)`

Servi via `GET /<project>/assets/:sha256` côté reader (route Next.js, runtime Node) :
- ACL via session Auth.js + `user_projects`.
- Headers : `Cache-Control: public, max-age=31536000, immutable` (le sha256 garantit l'immutabilité du contenu pour cette URL).
- `Content-Type: <mime>`.
- `ETag: "<sha256>"`.

### Cycle de vie des assets

- **Création** : à l'upload via `POST /api/ingest/asset`. Idempotent (UNIQUE `(project_id, sha256)`).
- **Réutilisation** : tout doc du même projet qui référence ce sha256 le voit comme "exists".
- **Suppression** : **pas de suppression en V1**. Les assets restent même quand le doc qui les référence est `unpublish`-é. Cohérent avec `architecture.md` §2 ("GC périodique post-V1").
- **Conséquence** : un projet long-running accumule des assets orphelins. Acceptable en V1 vu les volumes. Job de GC à prévoir post-V1 (job Vercel Cron qui scanne `assets` et supprime ceux non référencés par aucun `snapshot_json` du même projet).

---

## 6. Limites Vercel — payloads, à connaître AVANT d'implémenter

Contraintes Vercel à respecter côté reader. **Les chiffres ci-dessous correspondent au plan en cours et doivent être revérifiés au moment du déploiement réel** :

| Limite Vercel | Plan Hobby | Plan Pro | Décision V1 |
|---|---|---|---|
| Body max d'une Serverless Function (Node) | **~4.5 MB** | ~4.5 MB de base (configurable plus haut sur certaines configs) | **Asset max V1 : 4 MB** (refusé `413` au-dessus). Marge de sécurité incluse. |
| Body max d'une Edge Function | ~4 MB | ~4 MB | **Routes d'ingestion ciblent le runtime Node**, pas Edge (besoin d'accès libsql, BLOB, sha256). |
| Function timeout par défaut | 10 s | 60 s | Ingest doc cible < 5 s ; ingest asset cible < 8 s. Au-delà → revoir. |
| Storage filesystem | éphémère | éphémère | **Pas utilisable.** Toute écriture disque côté reader est interdite (perdue à froid). |
| Outbound request size depuis Edge | 5 MB | 5 MB | Non pertinent (l'extractor est le client, pas le reader). |

### Implications dures pour V1 (verrouillées)

- **Tout asset > 4 MB est refusé `413` côté reader**, remonté à l'utilisateur via le plugin Siyuan. **Pas de fallback automatique** (chunking, splitting, compression) — sinon on cache un bug de cadrage.
- **Tout doc dont `snapshot_json` + `html` approche ~3 MB sera surveillé** (laisser ~1 MB de marge pour overheads JSON, headers, etc.). Si un cas réel le déclenche → on revisitera plus tôt l'externalisation des assets ou le chunking du HTML.
- **L'extractor refuse explicitement `http://` non-localhost** comme cible reader. HTTPS obligatoire pour toute URL Vercel.
- **Aucune écriture filesystem côté reader.** Toute feature future qui semble "vouloir écrire un fichier" écrit en DB ou en Blob, jamais sur disque.

---

## 7. Sécurité & auth entre extractor et reader

| Aspect | Décision V1 |
|---|---|
| Auth | `Authorization: Bearer <INGEST_SECRET>`. |
| Secret | ≥ 32 octets aléatoires (`openssl rand -base64 32`). |
| Stockage du secret | env Vercel côté reader, env local côté extractor. **Jamais commité.** |
| Vérification | `crypto.timingSafeEqual` côté reader (même pattern que [`siyuan-extractor/src/server.ts`](../../../siyuan-extractor/src/server.ts) côté webhook plugin). |
| TLS | Forcé par Vercel. Extractor refuse `http://` non-localhost. |
| Allowlist IP | Aucune (extractor sur IP dynamique chez l'utilisateur). |
| Signature HMAC du body | **Non en V1** (TLS suffit). À ajouter en V1.x si non-répudiation devient un besoin. |
| Audit | Log structuré côté reader uniquement, pas de table d'audit DB en V1. Champs : `requestId, project, docId, contentHash, sha256?, result, durationMs, ip`. |
| Rate limit | Aucun en V1 (producteur unique authentifié, volume faible). À ajouter au premier signe d'abus. |

---

## 8. Codes d'erreur et politique de retry

### Codes d'erreur normalisés

| Code | Sens | Body type | Retryable côté extractor ? |
|---|---|---|---|
| `200` | Succès ou idempotent | `{ status: "ingested" \| "unchanged" \| "exists" \| "already_absent" \| "removed" }` | Non |
| `400` | Payload invalide (échec Zod) | `{ error: "validation_failed", issues: [...] }` | **Non** — bug à corriger côté extractor |
| `401` | Auth absente ou invalide | `{ error: "unauthorized" }` | **Non** — secret à corriger |
| `404` | Projet inconnu côté reader (pas pré-créé par l'admin) | `{ error: "project_not_found", project }` | **Non** — action admin requise |
| `409` | Doc référence un sha256 d'asset inconnu | `{ error: "missing_asset", sha256 }` | **Oui** — upload puis 1 retry |
| `413` | Payload au-delà de la limite Vercel | `{ error: "payload_too_large", limitBytes }` | **Non** |
| `429` | Rate-limited (réservé, pas V1) | `{ error: "too_many_requests" }` + header `Retry-After` | Oui, backoff |
| `5xx` | Erreur serveur transitoire | `{ error: "internal" }` | Oui, backoff exponentiel |

### Politique extractor

- **Timeout par requête : 30 s.**
- **Backoff exponentiel sur `5xx` et `429`** : 1 s → 4 s → 16 s, **3 tentatives max**.
- **Sur `409 missing_asset`** : upload les assets manquants en parallèle, puis **1 retry du doc** (pas plus, pour éviter une boucle).
- **Sur autres `4xx` (400, 401, 404, 413)** : **pas de retry**, l'erreur remonte directement au plugin Siyuan.
- **Sur épuisement des retries** : l'erreur remonte au plugin, l'utilisateur la voit dans la modale "Publier" du plugin.

### Politique reader

- **Toute opération idempotente répond `200`** avec un `status` clair (`"unchanged"`, `"exists"`, `"already_absent"`) plutôt qu'un `4xx` qui forcerait un retry inutile.
- Un `requestId` (uuid v4 généré à l'arrivée) est renvoyé dans le header `X-Request-Id` et loggué côté reader. Côté extractor, il est loggué aussi pour corrélation.

---

## 9. Implications schéma DB (à coder à l'étape 3 du plan)

Modifications à apporter à `siyuan-reader/src/db/schema.ts` :

### Table `documents` — ajouter / retirer

```ts
// ajouter
content_hash: text("content_hash").notNull(),         // sha256 hex du JSON canonique des blocs
snapshot_json: text("snapshot_json").notNull(),       // le JSON canonique sérialisé (source)
html: text("html"),                                    // HTML sanitisé sibling, nullable

// retirer
snapshot_path: text("snapshot_path").notNull(),       // plus de fichier sur disque
```

### Nouvelle table `assets`

```ts
export const assets = sqliteTable("assets", {
    id: text("id").primaryKey().$defaultFn(() => crypto.randomUUID()),
    projectId: text("project_id").notNull().references(() => projects.id, { onDelete: "cascade" }),
    sha256: text("sha256").notNull(),
    mime: text("mime").notNull(),
    sizeBytes: integer("size_bytes").notNull(),
    bytes: blob("bytes", { mode: "buffer" }).notNull(),
    createdAt: integer("created_at", { mode: "timestamp_ms" })
        .$defaultFn(() => new Date())
        .notNull(),
}, (t) => ({
    uniqProjectSha: unique().on(t.projectId, t.sha256),
}));
```

### Modules à supprimer une fois l'ingestion en place

- `siyuan-reader/src/snapshots/fs.ts` (lecture filesystem)
- `siyuan-reader/src/admin/sync.ts` (réconciliation FS↔DB)
- variable env `SNAPSHOTS_DIR` (et son default `/data/snapshots`)

### Modules à ajouter

- `siyuan-reader/src/app/api/ingest/doc/route.ts`
- `siyuan-reader/src/app/api/ingest/asset/route.ts` (avec sous-handler `HEAD` + `POST`)
- `siyuan-reader/src/app/api/ingest/unpublish/route.ts`
- `siyuan-reader/src/app/[project]/assets/[sha256]/route.ts` (servir les BLOBs sous ACL)
- `siyuan-reader/src/ingest/` (helpers d'auth, de validation, d'upsert)

---

## 10. Hors scope V1 (verrouillé)

- Streaming / multipart upload pour assets (V2 si besoin > 4 MB régulier).
- Vercel Blob / Cloudflare R2 / S3 pour assets (V2 — la table `assets` muera vers métadonnées-seules).
- Signature HMAC du body (V1.x si besoin de non-répudiation au-delà du TLS).
- Rate limiting / quotas par extractor.
- Table d'audit DB (logs structurés en console suffisants en V1).
- GC des assets orphelins (déjà acté hors scope V1 dans `architecture.md` §2 — à prévoir post-V1).
- Batch ingest (un doc par requête, **verrouillé**).
- Pull / event bus / queue / async.
- **Maintien du mode "filesystem only" Docker en parallèle du mode Vercel** : on choisit, on n'entretient pas les deux. Le mode Docker reste utilisable en dev mais n'est plus "supporté" comme cible.

---

## 11. Évolutions probables post-V1

- Assets externalisés (Vercel Blob / R2 / S3) avec table `assets` muée en métadonnées-seules.
- Multipart / streaming upload pour assets > 4 MB.
- HMAC signature du body ingestion (cf. §7).
- Rate limit / quotas par extractor.
- Webhook reader → extractor pour "demande de resync" (sens inverse).
- Multi-extractors par reader (équipe partagée, cas d'usage à définir).
- Job Vercel Cron pour GC des assets orphelins.

---

## 12. Liens

- Spec V1 figée (qui est amendée par ce doc) : [`architecture.md`](architecture.md).
- Producteur des snapshots : [`siyuan-extractor/`](../../../siyuan-extractor/).
- Consommateur : [`siyuan-reader/`](../../../siyuan-reader/).
- Plugin déclencheur : [`siyuan-plugin-publish/`](../../../siyuan-plugin-publish/).

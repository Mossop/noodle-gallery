# Smart albums (albums defined by search criteria): research

- **Date:** 2026-10-29
- **Status:** Research only. No design decision taken.
- **Question:** What would an album whose membership is "every photo matching a saved filter" (a person, a region, a
  tag, ...) look like in Gallery, and what makes it hard?

## 1. Summary

- Nothing in the tree stores a filter. The closest existing feature, "Add all filter results to album"
  (`specs/2026-07-05-add-filter-results-to-album-design.md`), pages the search once on the web client and calls the
  normal add-by-ids endpoint. A smart album is that action, remembered and repeated.
- **Every album surface reads stored `album_asset` rows.** About 75 SQL statements in 22 server files, the five
  mobile sync streams for albums, the mobile Drift tables, share links, download, map and access checks all join
  `album_asset` (or `albumMembership()`, which unions it with the fork's `album_space_asset`). A "virtual" album
  computed at read time would have to teach all of them, or be a new entity that is not an album.
- **The filtered timeline is already a virtual album.** `GET /timeline/buckets` accepts people, tags, place, camera,
  text, rating, type, favourite, date range, album, space and a `bbox`. On the web, `/photos?people=…&city=…` renders
  exactly what a smart album detail page would show. Mobile's search mode (`mobile/lib/gallery/search/`) is the same
  thing, server-driven.
- **Upstream's workflow engine already does half of the "keep a static album updated" variant**: on `AssetCreate` /
  `AssetMetadataExtraction` / `AssetTagged`, filter by location (country/state/city or a coordinate plus distance),
  date, tags, type, EXIF or filename, then `assetAddToAlbums`. It is forward-only (new uploads only), never removes,
  has no person filter (`PersonRecognized` is a commented-out stub upstream) and no favourite, rating or CLIP filter.
- **Recommended shape, if built:** a materialised smart album. A real upstream album plus a fork side table holding the
  rule; a fork job re-evaluates the rule under the owner's scope and diffs `album_asset` with two set-based statements.
  Every existing surface (mobile offline, share links, download, stock Immich apps, space linking, revert) then works
  untouched, and the upstream file footprint is close to zero. The hard parts are listed in §5: staleness and churn,
  manual edits versus the rule, person ids that die on merge, the lack of a shared filter format, and side effects of
  the add path (notifications, face-match fan-out, the global cover UPDATE).

Upstream status: Immich's own request is
[Dynamic Albums / Saved Searches, discussion 16161](https://github.com/immich-app/immich/discussions/16161), closed
without a feature. Community tools (`immich-smart-album` on Docker Hub, `immich-face-to-album` on PyPI) do the
materialised variant from outside through the API, which is a hint that the API-level version is feasible.

## 2. What exists today

### 2.1 Albums are rows

| Fact                                                                                                                                                                                                                                                                                  | Where                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `album` has no kind/type/source column; ownership is an `album_user` row with `role='owner'`                                                                                                                                                                                          | `server/src/schema/tables/album.table.ts:30-65`, `album-user.table.ts:21-26`                                                                                          |
| Membership is `album_asset (albumId, assetId, createdAt, updatedAt, updateId)`; the fork adds `album_space_asset` for cross-owner contributions, kept out of `album_asset` so it never becomes an access grant                                                                         | `album-asset.table.ts:15-38`, `album-space-asset.table.ts:29-77`                                                                                                      |
| `albumMembership()` is the one helper that unions the two; 10 files call it                                                                                                                                                                                                           | `server/src/utils/shared-space-album-scope.ts:241`                                                                                                                    |
| `startDate`, `endDate`, `assetCount`, `lastModifiedAssetTimestamp` are computed on every read from `album_asset`                                                                                                                                                                      | `album.repository.ts:211-248` (`getMetadataForIds`)                                                                                                                   |
| The album cover is maintained by a **global UPDATE over all albums on every list and detail read** (`updateThumbnails`)                                                                                                                                                               | `album.repository.ts:496-538`, `album.service.ts:64-96`                                                                                                               |
| `addAssets` bumps `album.updatedAt` and emits `AlbumUpdate` (email/push to recipients) and `AlbumAssetsAdd`; `removeAssets` does not bump `updatedAt` and emits `AlbumAssetsRemove`                                                                                                   | `album.service.ts:290-299, 430-438`                                                                                                                                   |
| `AlbumAssetsAdd` fans out one `SharedSpaceFaceMatch` job **per added asset** for every face-enabled space the album is linked to                                                                                                                                                      | `shared-space.service.ts:4643-4762`                                                                                                                                   |
| Deletes are audited (`album_asset_audit`) so mobile can learn about removals; there is no insert trigger                                                                                                                                                                               | `functions.ts:123-134`                                                                                                                                                |
| Mobile learns about membership **only** through new `album_asset.updateId` values and audit rows (`AlbumToAssetsV1`, `AlbumAssetsV2`, `AlbumAssetExifsV1`, plus the fork's `SharedSpaceAlbum*V1` clones); the album list and detail pages on mobile read local Drift only              | `sync.repository.ts:151-364`, `gallery/utils/gallery-sync.streams.ts:39-52`, `mobile/lib/infrastructure/repositories/remote_album.repository.dart`                     |
| The one mobile album view that reads the server live is search mode inside an album (`AlbumSearchScope` → `SearchTimelineSource`)                                                                                                                                                      | `mobile/lib/presentation/pages/remote_album.page.dart:236`, `mobile/lib/gallery/search/search_timeline.dart`                                                           |
| `map.repository.ts` `getAlbumMapMarkers` reads `album_asset` only (no contributions); an existing inconsistency, not new                                                                                                                                                               | `map.repository.ts:88-99`                                                                                                                                             |

### 2.2 Criteria have no shared representation

- **Web**: `FilterState` (`web/src/lib/components/filter-panel/filter-panel.ts:141-165`), serialised to 21 URL params
  by `web/src/lib/utils/filter-url.ts` (`people`, `tags`, `city/state/country`, `make/model/lens`,
  `description/filename/ocr`, `type`, `favorite`, `album=has|none`, `albumId`, `owner`, `rating`, `from/to`,
  `year/month`; the CLIP query is `?q=`). About 14 functions turn it into request DTOs, one per surface. The typed
  syntax (`person:anna`, `specs/2026-05-03-search-filter-syntax-design.md`) is an input method resolved client-side,
  not a stored format.
- **Mobile**: `SearchState` / `SearchFilters` (`mobile/lib/gallery/search/search_state.model.dart:116-159`), freezed,
  no JSON, never persisted. Has relative dates (`quickRange`: today / this week / this month / this year) resolved at
  request time.
- **Server**: many flat DTOs (`MetadataSearchDto`, `SmartSearchDto`, `TimeBucketDto`, `SearchOptionsRequestDto`) and
  the V3 `SearchFilter` tree (`search.dto.ts:261-310`), which no client sends and which cannot express `spaceId`,
  `spacePersonIds`, `ownerId`, `isInAlbum` or person tokens. The server parses no client format.
- **Person references** are tokens: `person:<personGroupId>` (owned) or `space-person:<id>` (member). The fork regex
  for them is duplicated three times (`gallery/dtos/search.dto.ts:21`, `time-bucket.dto.ts:15`, `gallery-map.dto.ts`).

### 2.3 Four predicate builders, one of which fits

| Builder                                                             | Serves                                                                    | Notes                                                                                                                                      |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `searchAssetBuilderLegacy` (`utils/database.ts:551-775`)            | flat metadata/smart/statistics search, filtered map, facets               | people and tags are ALL; tags include descendants via `tag_closure`                                                                        |
| `searchAssetBuilder` V3 (`database.ts:1038-1085`)                   | `/search/metadata` with `filter`                                          | four scope hooks (`AssetSearchScope`); no implicit `deletedAt IS NULL`; person tokens matched raw; `branchPredicates` not exported           |
| `withTimeBucketAssetFilters` (`asset.repository.ts:344-532`)        | `/timeline/buckets`, `/search/options` browse mode                        | **exported `qb → qb` transformer**, no ORDER BY/LIMIT, so it drops into `asset.id IN (…)` or `INSERT … SELECT`; tags ANY, people ALL; `bbox` |
| `assetFilterConditions` / `withAssetFilter` (`utils/asset-filter.ts`) | shared panel vocabulary (taken range, place, camera, rating)             | used by the three above                                                                                                                    |

All four take **already resolved, viewer-specific** inputs: `resolveViewerScope` (`utils/viewer-scope.ts:178-257`) is
async and DB-backed (partner ids, timeline space ids, hidden scope, person tokens → identity ids, per-surface rule).
A server-side re-run therefore has two steps: resolve the owner's scope at evaluation time, then apply the predicate.
Precedent: memory "themes" re-run a CLIP query with a threshold through `searchRepository.searchSmart`
(`services/memory-rules/theme-search.adapter.ts:62-90`), and the other memory rules use hand-written queries in
`gallery/repositories/memory-rule-asset.repository.ts`, both writing results into `memory_asset` rows.

### 2.4 Geography

- Exact `city` / `state` / `country` strings: every search and timeline path.
- Bounding box (`bbox=west,south,east,north`, antimeridian-aware): **time buckets only**
  (`time-bucket.dto.ts:102-130`, `asset.repository.ts:256-282`). No search DTO, map endpoint or V3 filter has it.
- Coordinate plus distance: **the workflow `assetLocationFilter` only** (`packages/plugin-core/manifest.json`).
- No polygon anywhere. "Has no location" is expressible in search (`city: null`) but not in time buckets.

### 2.5 People

- The public id is `personGroupId`, one person row per group (`person_personGroupId_key`).
- A **manual merge deletes the source person row and its group** (`person.repository.ts:276-368`); there is no
  redirect table. A saved `person:<sourceId>` then sets `hasInaccessibleToken` → `forceEmptyResult`, so the result is
  **silently empty** (`viewer-scope.ts:253`). Identity merges (automatic, cross-owner) re-key `person_identity` and keep
  the person id. Cluster/group splits repoint `personGroupId`, so the old id stops resolving for the moved owner.
- There is **no server event for a face being assigned to a person** (`event.repository.ts` has `AssetTag`,
  `AssetUntag`, `AssetMetadataExtracted`, `AssetCreate`, nothing for people). `person.service.ts` and
  `gallery-people.service.ts` emit nothing. Any event-driven refresh of a person rule needs a new emit in the
  fork-owned person files.

### 2.6 CLIP

A CLIP rule ("photos of beaches") is only a bounded set when `machineLearning.clip.maxDistance` is strictly between 0
and 2; the default is 0, disabled (`gallery/dtos/config.dto.ts:172`, `search.repository.ts:326`). The flat smart path
applies it, V3 `searchSmartV3` does not. Smart facets already materialise "all candidates within the threshold" into
a temp table (`smart-facet.repository.ts:920-965`), which is the model a materialised CLIP rule would follow.

## 3. Two product shapes

### A. Virtual: evaluate at read time ("saved search")

The album is a stored filter; its contents are the filtered timeline.

- Web is nearly free: a saved-filter entity whose detail route is the existing photos route with the saved URL params,
  or `/timeline/buckets` with the saved `TimeBucketDto`. Mobile has the server-driven search timeline; a saved search
  would be a new list page plus a route into search mode with a preset `SearchState` (which needs a JSON form first).
- Nothing else that an album has works: no `album_asset` rows means nothing to sync (mobile offline shows nothing),
  no share link, no download-as-album, no activity, no cover maintenance, no space linking, no album map, no
  `isInAlbum` / `hasAlbums` filters, no "add to / remove from album" of individual photos. Making those work means
  touching most of the ~75 `album_asset` SQL sites, which breaks rule 5 of the layering design (upstream files carry
  hooks, not logic) and the RBAC oracle.
- Sharing is an access-control problem: a viewer must see the owner's results, so the server has to evaluate the rule
  under the **owner's** scope on behalf of the viewer. Today every read path resolves the caller's scope; the access
  oracle (`test/medium/specs/utils/access-oracle.medium.spec.ts`) snapshots granted ids per permission and would need a
  new grant kind.
- Stock Immich apps never see it (fine under rule 2; it is a fork entity). Honest name for this shape: **saved
  searches**, not albums.

### B. Materialised: a real album kept in sync by a rule

A normal upstream album plus a fork side table; a job re-runs the rule and diffs `album_asset`.

- **Schema** (fork tables only, rule 1): `album_rule (albumId PK → album.id CASCADE, ownerId → user, criteria jsonb,
  mode, lastEvaluatedAt, lastError, createId/updateId)` and, if manual edits are allowed, `album_rule_override
  (albumId, assetId, kind include|exclude)`. Both go in `gallery-upstream-fks.json`. The migration's `down()` and
  `revert-to-immich.sql` drop them and the album stays as a static album with its current contents: the cleanest
  possible revert story.
- **Evaluation**: resolve the owner's viewer scope, then two statements per album inside one transaction:
  `INSERT INTO album_asset SELECT … FROM asset WHERE <withTimeBucketAssetFilters> AND NOT EXISTS(album_asset) AND NOT
  EXISTS(exclude override) ON CONFLICT DO NOTHING` and `DELETE FROM album_asset WHERE albumId=? AND NOT EXISTS(<rule>)
  AND NOT EXISTS(include override)`. No paging, no 10 MB body, no per-id `checkAccess` (the scope predicate already
  limits to what the owner may see). The result is a normal album for every surface listed in §2.1, including mobile
  offline via the unchanged `AlbumToAssetsV1` stream, stock Immich apps and share links.
- **Scheduling**: a nightly cron through `CronRepository` (pattern: `integrity.service.ts:97-152`,
  `database-backup.service.ts:101-125`) queuing one job per rule, plus a fast path for the events that exist:
  `AssetMetadataExtracted` (location, camera, date rules), `AssetTag` / `AssetUntag` (tag and classification rules),
  `AssetCreate`, `AssetTrash` / `AssetRestoreAll` / `AssetDelete`. Person rules get a new fork emit when faces are
  assigned or a merge finishes. A "refresh now" button covers the rest.
- **Reuse of upstream workflows** is possible for the incremental half (filter + `assetAddToAlbums` on upload), but
  the workflow filter config is a second criteria language with different semantics (coordinate+distance, no people,
  no favourites). Running two languages for one feature is a trap; better to let the rule evaluator own all adds and
  removes and treat workflows as unrelated.
- **Web UI** falls out of the filtered-timeline module: the active-filters bar already knows `collectTerms`
  (`filtered-timeline-controller.svelte.ts:247-249, 622-630`) and shows `SearchAddAllButton`; "Save as smart album"
  sits next to "Add all to…". The album page shows a rule chip and "edit rule" / "refresh". The album-detail preview
  before saving is the current `/photos?…` page, so shape A's cheap part comes along anyway.
- **Mobile** needs nothing for the album to work. A "smart" badge and read-only affordances need the rule to reach the
  app: either a fork sync stream (`SmartAlbumRulesV1` behind `isForkAwareClient`, as the space-album streams are) or a
  fork REST read on the album detail page. Editing rules from mobile means a JSON form of `SearchState` and a mapping
  to the stored criteria.

## 4. Criteria format

The stored rule needs a server-owned, versioned shape. Options, in order of fit:

1. **The time-bucket vocabulary** (`TimeBucketDto` minus paging and `albumId`): it is what both clients already
   produce, it has `bbox`, people tokens, tags, place, camera, text, rating, type, favourite, date range, space, and
   `withTimeBucketAssetFilters` executes it. Gaps: no CLIP query, no "no location" / untagged, tags are ANY and people
   ALL (search is the opposite, `specs/2026-07-23-search-v3-coexistence-design.md` records why), no OR/NOT.
2. **V3 `SearchFilter`**: expressive (`any/all/none`, `or`, typed operators) and upstream-shaped, but no client builds
   it, it has no `bbox`, no person tokens, no space fields, and no implicit trash rule.
3. **Web `FILTER_URL_PARAMS` as a string**: zero translation on web, but it is a client format the server would have
   to learn, and mobile does not produce it.

Whichever is chosen, the server needs one parser and one validator for it (there is none today), a `version` field,
and the web/mobile `FilterState` ↔ criteria mappings in the fork namespaces. Relative dates ("this year") should be
stored as relative and resolved at evaluation time; mobile already has the concept, web does not.

## 5. What makes it hard

1. **No stored filter format exists**; see §4. This is the first thing to design and it touches three codebases.
2. **Person ids die.** A rule bound to `person:<id>` goes silently empty after a manual merge of that person into
   another, or a group split. Fix options: hook `mergePersonProfile` (fork-owned `person.service.ts`) to rewrite rules
   to the target id; or store `face_identity.id` plus the display token; or surface "this rule references a person that
   no longer exists" via `lastError`. Space people (`space-person:<id>`) add membership and `showInTimeline` conditions
   that can change under the owner.
3. **Owner-scoped evaluation versus viewer-scoped reads.** Rules evaluate under the owner's scope (partners, spaces,
   hidden spaces, the owner's favourites). Favourites are per user (`asset_favorite`), so "my favourites of Alice" is
   the owner's, which is the only sane reading for a shared album, but it needs saying in the UI.
4. **Side effects of the add path.** Going through `AlbumService.addAssets` would send `AlbumUpdate` notifications to
   every recipient on every refresh, run per-id access checks, bump the cover, and (for albums linked to a face-enabled
   space) queue one `SharedSpaceFaceMatch` per asset. The evaluator must write the repository directly and emit only
   `AlbumAssetsAdd` / `AlbumAssetsRemove` so space people stay in sync, and must not bump `album.updatedAt` on no-op
   runs or every mobile client re-syncs the album nightly.
5. **Churn and staleness.** Deletes produce `album_asset_audit` rows and sync traffic; a rule with a moving boundary
   (a rolling "last 30 days") rewrites part of the album every night. The design needs a cap or a warning for very
   large result sets and a decision on how often rules run (nightly plus events is the realistic answer; "on access"
   turns a GET into a write and fights the mobile sync model).
6. **Manual edits versus the rule.** A user removing one photo from a smart album expects it to stay removed; a user
   adding one expects it to stay. Either the album is read-only (simple, and the `album_space_asset` contribution path
   from spaces must be disabled for it too) or an override table is kept and the UI explains it. Workflow-added assets
   (`assetAddToAlbums`) targeting a smart album are a third writer.
7. **Semantics differ by surface.** Tags ANY in time buckets and ALL in search; people ALL in both although the DTO
   says any; `withTimeBucketAssetFilters` includes the trash rule, V3 does not. The preview a user sees while building
   the rule must be the same query the job runs, so pick one builder (the timeline one) for both.
8. **Geography is thin.** A "region" today means exact place strings or a `bbox`; there is no radius or polygon
   outside the workflow filter, and the map has no draw tool. A region smart album needs a bbox picker on the map (the
   map already sends `bbox` for its timeline panel) or it is limited to city/state/country.
9. **CLIP rules are only bounded with a configured threshold**, which defaults to off; and the V3 path ignores it.
10. **Guards to keep green.** The RBAC oracle (`rbac-oracle.medium.spec.ts`) snapshots album membership per actor;
    the favourite grep gate; the FK registry spec; the revert spec; the mobile `sync_stream_service_test.dart` enum
    test if a new sync type is added; the i18n overlay in ten locales for every new string.

## 6. Smallest useful first cut

- Materialised (shape B), web only, read-only smart albums, rule stored as the time-bucket vocabulary with a
  `version`, people as `person:` tokens with a `lastError` when a token stops resolving.
- Evaluation: nightly cron plus `AssetMetadataExtracted` / `AssetTag` / `AssetUntag` fast path, plus a manual
  refresh endpoint. Person-assignment events deferred.
- Mobile gets the album through the unchanged sync streams; the badge and read-only state wait for a second cut.
- Fork footprint: `server/src/gallery/{controllers,dtos,repositories,services}/smart-album.*`, two fork migrations,
  `gallery-upstream-fks.json`, `revert-to-immich.sql`, `web/src/lib/gallery/smart-album/`, overlay strings. Upstream
  files: one registration line in `src/gallery/index.ts` (already fork), the two existing `AlbumAssets*` emits are
  reused, no new DTO spread needed unless `AlbumResponseDto` should carry `isSmart` (one spread in
  `gallery/dtos/album.dto.ts`, already a fork file).

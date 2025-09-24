

1. **HLD (High Level Design)** — components, responsibilities, end-to-end flow, non-functional requirements.
2. **LLD (Low Level Design)** — data schemas, box layout, queue item schema, sync algorithm, retry/backoff, conflict resolution pseudo-logic, APIs between layers.
3. **Implementation Details** — concrete folder structure, recommended packages, key code snippets you can drop into a project (Hive models + adapters, repository, sync engine, connectivity listener, background worker), testing & observability, edge cases and migration notes.

I’ll be pragmatic: full enough to implement end-to-end and to scale/operate in production.

---

# 1) HLD — High Level Design

## Goal

Make the app work offline-first: the user can create/edit/delete objects while offline; changes are persisted locally and queued; when connectivity is available the Sync Engine reliably and efficiently syncs queued actions with the server and resolves conflicts.

## Primary components

* **UI / State Management (Riverpod/BLoC)**
  Presents data, shows sync state and conflict notifications. Reads from Repo (local DB).
* **Repository**
  Single entry point for app operations (CRUD). Writes always go to local storage and enqueues sync actions. Exposes streams for UI.
* **Local DB (Hive)**
  Stores main entities and a separate Sync Queue box. Fast reads/writes, offline persistence.
* **Sync Queue (Hive Box)**
  Queue of serialized offline actions (create/update/delete) with metadata (id, timestamps, attempt counts). Durable.
* **Sync Engine / Service**
  Background process that runs on network availability or scheduled triggers. Reads the queue, performs batched syncs, applies conflict resolution, updates local DB, clears successful items, retries failures with exponential backoff.
* **Connectivity Listener**
  Uses `connectivity_plus` to detect transitions to online; triggers Sync Engine.
* **Background Worker**
  Ensures periodic background sync even if app is killed (Android WorkManager, Android Alarm Manager + platform-specific iOS approach, or background\_fetch).
* **Remote API**
  Server endpoints designed for idempotency and versioning (accept client-generated IDs, return canonical versions & timestamps).
* **Conflict Resolver**
  Policy implementation: LWW, server-wins, client-wins, or custom field-level merging. Exposes API for manual resolution UI.

## Dataflow (summary)

1. UI calls `repository.create(entity)`.
2. Repository writes entity to Hive main box with `isSynced=false`, `lastUpdated=now`.
3. Repository creates a `SyncQueueItem` and appends to sync queue box.
4. When connectivity is present (or background job runs), Sync Engine reads queue (in order or prioritized), batches payloads and sends to remote API.
5. Server returns success or conflict. On success, mark local record `isSynced=true` and update server metadata (server timestamp, id). On conflict, apply conflict resolution; if resolved, update local and clear queue item; otherwise mark conflict for user resolution.
6. Failed requests are retried using exponential backoff and chunking.

## Non-functional requirements

* **Durability**: all offline actions survive app restarts and device reboots.
* **Order & Causality**: preserve order where business rules require it (e.g., create before update).
* **Idempotency**: server should accept idempotent requests or the client attach an idempotency key (UUID).
* **Performance**: batch sync with chunk sizes tuned; delta sync to reduce payload.
* **Resource usage**: bounded queue processing to limit CPU/memory.
* **Observability**: sync status metrics, attempts, last successful sync time, queue length.
* **Security**: encrypt sensitive data if required; use secure network with tokens; do not leak credentials.

---

# 2) LLD — Low Level Design

This is where we define actual schemas, algorithms and how components interact.

## A. Data models & Hive boxes

### 1) Main entity example: `Note`

```dart
@HiveType(typeId: 0)
class Note extends HiveObject {
  @HiveField(0) String id;            // client-generated UUID or server id
  @HiveField(1) String title;
  @HiveField(2) String content;
  @HiveField(3) DateTime lastUpdated; // local last update time
  @HiveField(4) String? serverId;     // if server assigns numeric id
  @HiveField(5) bool isSynced;        // true when in sync with server
  @HiveField(6) int revision;         // optional monotonic revision/version
  @HiveField(7) String? conflictFlag; // e.g. "CONFLICT_FIELD_LEVEL"
}
```

* **Box names**:

  * `box:notes` — main data.
  * `box:sync_queue` — queue of `SyncQueueItem`.
  * `box:metadata` — store lastSyncAt, lastSequenceNumber, etc.

### 2) Sync queue item model: `SyncQueueItem`

```dart
@HiveType(typeId: 10)
class SyncQueueItem extends HiveObject {
  @HiveField(0) String queueId;        // UUID for the queue item
  @HiveField(1) String entityId;       // note.id
  @HiveField(2) String entityType;     // "Note"
  @HiveField(3) String action;         // "create" | "update" | "delete"
  @HiveField(4) Map<String, dynamic> payload; // actual delta or full object
  @HiveField(5) DateTime createdAt;    // when queued
  @HiveField(6) int attempts;          // retried attempts
  @HiveField(7) DateTime? nextAttemptAt; // for scheduled retries/backoff
  @HiveField(8) int priority;          // optional for priority processing
}
```

* `payload` should be the minimal delta whenever possible to reduce sizes.
* `queueId` ensures idempotency on server side if you send it as `idempotencyKey`.

## B. Repository APIs (contracts)

Public API:

```dart
abstract class NoteRepository {
  Future<void> createNote(Note note);
  Future<void> updateNote(Note note);
  Future<void> deleteNote(String noteId);
  Stream<List<Note>> watchNotes(); // reactive read
  Future<void> forceSync();        // manual trigger
}
```

Implementation responsibilities:

* Always write to `box:notes`.
* Always create a `SyncQueueItem` for create/update/delete (unless configured to sync immediately for low-latency operations).
* Mark entities `isSynced=false` after local changes.
* If online, optionally call `syncEngine.scheduleImmediate()`.

## C. Sync Engine design

### Core responsibilities

* Continuously or periodically read pending `SyncQueueItem`s where `nextAttemptAt <= now`.
* Process items in order (or by priority), group them into **batches** (e.g., 25 items or X KB).
* Call remote endpoints with batched payload. Include `queueId` and `entityId` to allow server idempotency.
* On success: update corresponding entities in `box:notes` with server metadata, mark `isSynced=true`, remove queue item.
* On failure: if transient (network/5xx), increment attempts, set `nextAttemptAt = now + backoff(attempts)`; if permanent (403 / validation error), mark as failed and surface to UI or drop depending on policy.
* On conflict (server responds with conflict + server version): call Conflict Resolver.

### Ordering and grouping

* Support **per-entity ordering** guarantee if required: maintain `sequenceNumber` or rely on queue FIFO for same entity.
* For stronger causal ordering (e.g., create→update→delete), you must ensure queue items for the same entity are processed in order — the Sync Engine should lock per `entityId` while processing its queue items sequentially.

### Backoff policy

Use exponential backoff with jitter:

```
nextAttempt = now + base * 2^(attempts - 1) + random_jitter
base = 10s initial for network errors
cap = 24h
```

Store `attempts` and `nextAttemptAt` inside `SyncQueueItem`.

### Batching & chunking

* Batch size configurable: e.g., 25 items or 1MB payload.
* Use delta sync: only send changed fields.
* For large queue, process in chunks and commit progress.

### Idempotency & deduplication

* Attach `queueId` to request so the server can deduplicate duplicate requests.
* For retries, server returns success for already-applied `queueId`.

## D. Conflict resolution strategies (LLD)

* **Server-assigned versioning**: server returns `revision` or `lastUpdated`. Use those for detection.
* **Default policy**: LWW by `lastUpdated` (timestamp comparison, with server time considered authoritative if clocks are not trusted).
* **Policy matrix**:

  * If `local.lastUpdated > server.lastUpdated` → client-wins (send overwrite).
  * If `server.lastUpdated > local.lastUpdated` → server-wins (apply server version).
  * If both modified concurrently and fields are independent → field-level merge (for lists or tags use union/merge rules).
  * If critical conflict (both changed same field differently) → mark conflict, do not auto-resolve, present UI for manual resolution.
* **Conflict handling flow**:

  1. Server returns conflict with server state.
  2. Conflict Resolver tries automatic merge (field level).
  3. If auto-merge successful → apply merged state locally and mark synced.
  4. If not → generate `ConflictRecord` in a dedicated `box:conflicts` and surface in UI for user resolution; keep the queue item paused or removed per policy.

## E. Background & lifecycle

* **Foreground**: connectivity listener triggers immediate sync on becoming online.
* **Background**:

  * Android: use `workmanager` (android WorkManager plugin) to schedule periodic sync. Use a headless Dart callback.
  * iOS: background execution is limited — use `background_fetch` or ensure server sends push notifications to trigger sync where possible.
* **App killed**: rely on system scheduling (WorkManager) and server push (if available).
* **Manual Sync**: repository `forceSync()`.

## F. Observability & telemetry

* Track metrics:

  * `queue.length`, `attempts histogram`, `success rate`, `lastSyncAt`, `conflicts.count`.
* Logging levels: DEBUG for sync details, INFO for success/failure, WARN for conflicts.
* Expose a `SyncStatus` object via provider for UI.

## G. Security & data protection

* Secure tokens in secure storage (flutter\_secure\_storage).
* Use TLS. If sensitive data stored locally, use platform-level encryption or Hive encryption box.

---

# 3) Implementation details — code, folder structure, packages, examples

I’ll give you (a) recommended packages, (b) a folder structure, and (c) complete core code snippets you can paste, plus notes.

## Recommended packages

* `hive` + `hive_flutter` — local DB
* `uuid` — entity/queue IDs
* `connectivity_plus` — network monitoring
* `workmanager` or `background_fetch` — background tasks
* `riverpod` (or `flutter_riverpod`) or `bloc` — state management
* `dio` — HTTP client (supports interceptors, timeouts, retries)
* `flutter_secure_storage` — for storing tokens
* `json_serializable` (optional) — model serialization

> Note: on iOS you’ll need to handle background limitations; combine push notifications and background fetch as appropriate.

---

## Folder structure (suggested)

```
lib/
  app.dart
  main.dart
  core/
    network/
      api_client.dart
    sync/
      sync_engine.dart
      sync_queue_item.dart
      conflict_resolver.dart
      sync_status.dart
    storage/
      hive_service.dart
  data/
    models/
      note.dart
      note_adapter.dart
    repository/
      note_repository.dart
      note_repository_impl.dart
  ui/
    notes/
      notes_page.dart
      note_item.dart
  providers/
    providers.dart   // Riverpod providers
  background/
    background_service.dart
test/
  unit/... 
```

---

## Key code snippets

> The snippets below are concise core implementations. You should adapt to your app’s types and error handling style.

### 1) Note model + Hive adapter (minimal)

```dart
// lib/data/models/note.dart
import 'package:hive/hive.dart';
part 'note.g.dart';

@HiveType(typeId: 0)
class Note extends HiveObject {
  @HiveField(0)
  String id;

  @HiveField(1)
  String title;

  @HiveField(2)
  String content;

  @HiveField(3)
  DateTime lastUpdated;

  @HiveField(4)
  bool isSynced;

  @HiveField(5)
  int revision;

  @HiveField(6)
  String? conflictFlag;

  Note({
    required this.id,
    required this.title,
    required this.content,
    DateTime? lastUpdated,
    this.isSynced = false,
    this.revision = 0,
    this.conflictFlag,
  }): this.lastUpdated = lastUpdated ?? DateTime.now();

  Map<String, dynamic> toMap() => {
    'id': id,
    'title': title,
    'content': content,
    'lastUpdated': lastUpdated.toIso8601String(),
    'revision': revision,
  };

  static Note fromMap(Map<String,dynamic> m) {
    return Note(
      id: m['id'],
      title: m['title'] ?? '',
      content: m['content'] ?? '',
      lastUpdated: DateTime.parse(m['lastUpdated']),
      isSynced: m['isSynced'] ?? false,
      revision: m['revision'] ?? 0,
    );
  }
}
```

> Use `build_runner` to generate `note.g.dart` (Hive TypeAdapter) or write an adapter manually.

### 2) SyncQueueItem

```dart
// lib/core/sync/sync_queue_item.dart
import 'package:hive/hive.dart';

part 'sync_queue_item.g.dart';

@HiveType(typeId: 10)
class SyncQueueItem extends HiveObject {
  @HiveField(0) String queueId;
  @HiveField(1) String entityId;
  @HiveField(2) String entityType;
  @HiveField(3) String action; // create/update/delete
  @HiveField(4) Map<String, dynamic> payload;
  @HiveField(5) DateTime createdAt;
  @HiveField(6) int attempts;
  @HiveField(7) DateTime? nextAttemptAt;
  @HiveField(8) int priority;

  SyncQueueItem({
    required this.queueId,
    required this.entityId,
    required this.entityType,
    required this.action,
    required this.payload,
    DateTime? createdAt,
    this.attempts = 0,
    this.nextAttemptAt,
    this.priority = 0,
  }) : this.createdAt = createdAt ?? DateTime.now();
}
```

### 3) HiveService — open boxes

```dart
// lib/core/storage/hive_service.dart
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';
import '../../data/models/note.dart';
import '../sync/sync_queue_item.dart';

class HiveService {
  static const notesBox = 'box_notes';
  static const syncQueueBox = 'box_sync_queue';
  static Future<void> init() async {
    await Hive.initFlutter();
    Hive.registerAdapter(NoteAdapter());
    Hive.registerAdapter(SyncQueueItemAdapter());
    await Hive.openBox<Note>(notesBox);
    await Hive.openBox<SyncQueueItem>(syncQueueBox);
  }

  static Box<Note> getNotesBox() => Hive.box<Note>(notesBox);
  static Box<SyncQueueItem> getQueueBox() => Hive.box<SyncQueueItem>(syncQueueBox);
}
```

### 4) NoteRepositoryImpl — local write + enqueue

```dart
// lib/data/repository/note_repository_impl.dart
import 'package:uuid/uuid.dart';
import '../../core/storage/hive_service.dart';
import '../../core/sync/sync_queue_item.dart';
import '../models/note.dart';

class NoteRepositoryImpl {
  final _notesBox = HiveService.getNotesBox();
  final _queueBox = HiveService.getQueueBox();
  final uuid = Uuid();

  Stream<List<Note>> watchNotes() {
    return _notesBox.watch().map((_) => _notesBox.values.toList());
  }

  Future<void> createNote(Note note) async {
    // ensure id
    if (note.id.isEmpty) {
      note.id = uuid.v4();
    }
    note.isSynced = false;
    note.lastUpdated = DateTime.now();
    await _notesBox.put(note.id, note);

    final queueItem = SyncQueueItem(
      queueId: uuid.v4(),
      entityId: note.id,
      entityType: 'Note',
      action: 'create',
      payload: note.toMap(),
    );
    await _queueBox.put(queueItem.queueId, queueItem);
  }

  Future<void> updateNote(Note note) async {
    note.lastUpdated = DateTime.now();
    note.isSynced = false;
    await _notesBox.put(note.id, note);

    final queueItem = SyncQueueItem(
      queueId: uuid.v4(),
      entityId: note.id,
      entityType: 'Note',
      action: 'update',
      payload: note.toMap(),
    );
    await _queueBox.put(queueItem.queueId, queueItem);
  }

  Future<void> deleteNote(String noteId) async {
    // mark deleted locally or remove depending on UX policy
    await _notesBox.delete(noteId);

    final queueItem = SyncQueueItem(
        queueId: uuid.v4(),
        entityId: noteId,
        entityType: 'Note',
        action: 'delete',
        payload: {'id': noteId});
    await _queueBox.put(queueItem.queueId, queueItem);
  }
}
```

### 5) API Client skeleton (using Dio)

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';

class ApiClient {
  final Dio _dio;
  ApiClient(String baseUrl, {required String token}) :
    _dio = Dio(BaseOptions(baseUrl: baseUrl, headers: {'Authorization': 'Bearer $token'}));

  Future<Response> batchSync(List<Map<String,dynamic>> items) {
    // POST /sync/batch
    return _dio.post('/sync/batch', data: {'items': items});
  }
}
```

### 6) SyncEngine — core flow (simplified)

```dart
// lib/core/sync/sync_engine.dart
import 'dart:math';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:uuid/uuid.dart';
import '../storage/hive_service.dart';
import 'sync_queue_item.dart';
import '../network/api_client.dart';
import '../../data/models/note.dart';
import 'conflict_resolver.dart';

class SyncEngine {
  final ApiClient apiClient;
  final connectivity = Connectivity();
  bool _running = false;
  final int batchSize;
  final uuid = Uuid();

  SyncEngine(this.apiClient, {this.batchSize = 25}) {
    connectivity.onConnectivityChanged.listen((status) {
      if (status != ConnectivityResult.none) scheduleImmediate();
    });
  }

  void scheduleImmediate() {
    if (!_running) _processQueue();
  }

  Future<void> _processQueue() async {
    _running = true;
    final queueBox = HiveService.getQueueBox();
    final notesBox = HiveService.getNotesBox();

    try {
      // read pending items where nextAttemptAt == null || <= now
      final now = DateTime.now();
      final pending = queueBox.values
        .where((it) => it.nextAttemptAt == null || it.nextAttemptAt!.isBefore(now))
        .toList()
        .cast<SyncQueueItem>();

      // optionally sort by priority then createdAt
      pending.sort((a,b) {
        final p = a.priority.compareTo(b.priority);
        return p != 0 ? p : a.createdAt.compareTo(b.createdAt);
      });

      // process in batches
      for (var i = 0; i < pending.length; i += batchSize) {
        final batch = pending.skip(i).take(batchSize).toList();
        final requestItems = batch.map((b) => {
          'queueId': b.queueId,
          'entityType': b.entityType,
          'action': b.action,
          'entityId': b.entityId,
          'payload': b.payload,
        }).toList();

        try {
          final resp = await apiClient.batchSync(requestItems);
          // assume resp.data has per-item statuses
          final results = resp.data['results'] as List<dynamic>;
          for (var j = 0; j < results.length; j++) {
            final r = results[j];
            final qi = batch[j];
            if (r['status'] == 'ok') {
              // update local entity with server metadata if present
              if (qi.entityType == 'Note') {
                final note = notesBox.get(qi.entityId) as Note?;
                if (note != null) {
                  // server may return updated fields
                  if (r['serverVersion'] != null) {
                    note.revision = r['serverVersion'];
                  }
                  note.isSynced = true;
                  await note.save();
                }
              }
              await qi.delete(); // remove from queue
            } else if (r['status'] == 'conflict') {
              // handle conflict
              final serverPayload = r['serverPayload'] as Map<String, dynamic>;
              final noteLocal = notesBox.get(qi.entityId) as Note?;
              final resolved = ConflictResolver.resolveNoteConflict(noteLocal?.toMap(), serverPayload);
              if (resolved.autoMerged) {
                final mergedMap = resolved.merged!;
                // write merged note
                final mergedNote = Note.fromMap(mergedMap);
                mergedNote.isSynced = true;
                await notesBox.put(mergedNote.id, mergedNote);
                await qi.delete();
              } else {
                // create conflict record; notify UI; keep queue item paused or deleted per policy
                // for simplicity, mark conflictFlag on the note
                if (noteLocal != null) {
                  noteLocal.conflictFlag = 'CONFLICT';
                  await noteLocal.save();
                }
                // mark queue for manual resolution: do not automatic retry
                qi.nextAttemptAt = null;
                await qi.save();
              }
            } else {
              // transient or permanent failure
              await _handleFailure(qi, r['errorCode']);
            }
          }
        } catch (e) {
          // network or server error: set backoff
          for (var qi in batch) {
            await _applyBackoff(qi);
          }
        }
      }
    } finally {
      _running = false;
    }
  }

  Future<void> _handleFailure(SyncQueueItem qi, int errorCode) async {
    // treat 4xx as permanent unless 429, 5xx as transient
    if (errorCode >= 500 || errorCode == 429) {
      await _applyBackoff(qi);
    } else {
      // permanent — surface to UI, drop or move to dead-letter
      qi.nextAttemptAt = null;
      // could move to dead-letter box or set a permanent flag
      await qi.save();
    }
  }

  Future<void> _applyBackoff(SyncQueueItem qi) async {
    qi.attempts = (qi.attempts ?? 0) + 1;
    final baseSeconds = 10;
    final maxSeconds = 24 * 3600; // 24h cap
    final backoff = min(maxSeconds, baseSeconds * pow(2, qi.attempts - 1).toInt());
    final jitter = Random().nextInt(10);
    qi.nextAttemptAt = DateTime.now().add(Duration(seconds: backoff + jitter));
    await qi.save();
  }
}
```

> This is a simplified engine. For production: add locks to prevent concurrent processing, better error handling, and per-entity ordering guarantees.

### 7) ConflictResolver

```dart
// lib/core/sync/conflict_resolver.dart
class ConflictResolutionResult {
  final bool autoMerged;
  final Map<String,dynamic>? merged;
  ConflictResolutionResult(this.autoMerged, [this.merged]);
}

class ConflictResolver {
  // Example for Note models: merge title and content with priority rules
  static ConflictResolutionResult resolveNoteConflict(Map<String,dynamic>? local, Map<String,dynamic> server) {
    if (local == null) {
      return ConflictResolutionResult(true, server);
    }
    final localUpdated = DateTime.parse(local['lastUpdated']);
    final serverUpdated = DateTime.parse(server['lastUpdated']);
    if (localUpdated.isAfter(serverUpdated)) {
      // client wins
      final merged = {...server, ...local}; // local overwrites server for conflicting fields
      return ConflictResolutionResult(true, merged);
    } else if (serverUpdated.isAfter(localUpdated)) {
      // server wins
      return ConflictResolutionResult(true, server);
    } else {
      // same timestamp: attempt field merge (for example union tags) or prompt UI
      // fallback to server wins
      return ConflictResolutionResult(true, server);
    }
  }
}
```

### 8) Connectivity listener + background scheduling

* Create a `BackgroundService` that registers with `workmanager` to call `SyncEngine.scheduleImmediate()` periodically.
* On app start, do `HiveService.init()` and register adapter types.

### 9) Riverpod provider for SyncStatus and UI

Expose `queue.length`, `isSyncing`, `lastSyncedAt` to UI.

---

## Testing strategy

* **Unit tests**:

  * Repository write operations create local entity & queue item.
  * ConflictResolver tests for merge policies.
  * Backoff computation tests for exponential growth and jitter.
* **Integration tests**:

  * Mock API responses (success, conflict, transient failure) and assert final DB state.
  * End-to-end: offline create → restart app → background worker (simulate connectivity) → final synced state.
* **Manual QA**:

  * Test sequence of operations: create → update → delete while offline and ensure correct server state.

---

## Edge cases & production notes

1. **Clock skew**: do not trust device timestamps solely. Use server revision numbers or server timestamps in the authoritative conflict decision. If you must use timestamps, consider sending both client and server times and applying an offset reconciliation.

2. **Order guarantees**: if order matters per entity, process queue items grouped by `entityId` in FIFO. Do not process later items for an entity until earlier ones succeed. Implement per-entity locking.

3. **Large queues**: for extremely large queues, chunk ingestion and process in background with capped memory usage. Allow user to opt into “fast sync” that processes in smaller chunks, or compress payloads.

4. **Deletes**: implement soft delete semantics locally if the user should see the deletion confirmation until sync confirms server deletion.

5. **Migration**: when changing models, apply Hive migration carefully — add new fields with defaults and implement versioned adapters.

6. **Server design**: server should accept `queueId` and return idempotent results, provide revisions and conflicts payloads. Support batch endpoints.

7. **iOS background limits**: iOS will restrict background runtime — rely on push notifications or background fetch (limited frequency). Communicate these limitations in UX (sync may take longer on iOS when app is not in foreground).

8. **Encryption**: use Hive encryption for sensitive info.

9. **Backpressure / Throttling**: server may throttle; handle HTTP 429 specially with `Retry-After`.

10. **Observability**: add Sentry / logs for errors, and expose in-app sync debugging screen for support.

---

## Example of UI indicators

* Pending badge: show `isSynced == false` items with orange dot.
* Sync progress modal: queue count + processed items.
* Conflict resolution screen: show local vs server diff with "Keep Local", "Keep Server", "Merge" actions.

---

## Deployment checklist

* Add proper error monitoring and offline logs.
* Test background sync on real devices (Android and iOS).
* Test with flaky networks and intermittent connectivity.
* Make server idempotent and return clear error types (ok/conflict/error).
* Ensure token refresh in API client during background sync.

---

## Final notes — priorities & extension suggestions

* Start with a minimal working engine: local writes + queue + connectivity-triggered sync. Make conflict resolver simple (LWW) initially.
* Add batching, backoff, and background scheduling iteratively.
* Add more sophisticated conflict resolution (field-level merge, CRDTs) only if your domain requires non-destructive merges (collaborative text, counters, lists).
* Consider libraries that already implement offline sync patterns if you want to accelerate (some open-source SDKs offer this), but building as above provides full control.


# LangChain4j MongoDB-backed ChatMemoryStore — design sketch

Linked issue: https://github.com/langchain4j/langchain4j/issues/4830

## Why

The built-in `ChatMemoryStore` implementations cover in-memory and a handful of
SQL/vector stores, but there is no first-party MongoDB adapter. For teams that
already use MongoDB as their primary operational store (e.g. FitTrack's
activities/nutrition service), pulling in a separate store just to persist
conversation history is unnecessary friction.

## Proposed API surface

```java
public class MongoChatMemoryStore implements ChatMemoryStore {
    private final MongoCollection<Document> collection;

    // memoryId is expected to be a stable String (user id, session id, etc.)
    @Override public List<ChatMessage> getMessages(Object memoryId) { ... }
    @Override public void updateMessages(Object memoryId, List<ChatMessage> messages) { ... }
    @Override public void deleteMessages(Object memoryId) { ... }
}
```

## Storage model

One document per `memoryId`, messages serialized as an ordered array. This keeps
reads to a single `findOne` and writes to a single `replaceOne` with upsert.

```json
{
  "_id": "user:42",
  "messages": [
    { "role": "user", "text": "...", "ts": 1727500000 },
    { "role": "ai",   "text": "...", "ts": 1727500002 }
  ],
  "updatedAt": ISODate("2026-09-28T04:55:00Z")
}
```

Alternative: one document per message, indexed on `(memoryId, ts)`. Better for
very long histories and TTL expiration, but requires an aggregation pipeline
for `getMessages`. Skipping this for v1.

## Open questions

- Serialization: reuse langchain4j's existing `ChatMessageSerializer` (JSON) or
  keep messages as native BSON? JSON is simpler and matches other stores; BSON
  is nicer if we ever want to query message content directly.
- TTL: expose a builder option for `expireAfterSeconds` on `updatedAt`?
- Sync vs reactive: start with the sync `MongoCollection` driver; a reactive
  variant using `MongoReactiveClient` can follow once the sync path stabilizes.

## Next

- Prototype against langchain4j `main` and run the existing
  `ChatMemoryStoreIT` contract tests against an ephemeral Mongo container.
- Draft the PR once the contract tests pass locally.

# Rust / raw epoll / own Postgres client

Front: direct

| | |
|---|---|
| Language | Rust 1.99.0 (edition 2024), fat LTO, one codegen unit, `target-cpu=native` |
| HTTP | Hand-written HTTP/1.1 on raw `epoll` + `SO_REUSEPORT`, request parsing with `httparse` |
| Postgres driver | Own v3 wire-protocol client (SCRAM-SHA-256, binary rows, pipelining). No pool library |
| Other crates | `hmac` + `sha2` (JWT), `serde_json` (request bodies, JWT claims), `base64`, `memchr`, `itoa`, `getrandom`, `mimalloc`. Pinned in `Cargo.lock` |
| bcrypt | Own implementation (Blowfish tables from RustCrypto's `blowfish` crate), cost 10, verified against the `bcrypt` crate in tests |

```bash
sudo bash install.sh
bash build.sh
DATABASE_URL=postgres://conduit:conduit@127.0.0.1:5432/conduit JWT_SECRET=... HOST=0.0.0.0 PORT=80 bash start.sh
```

## Design: a batch planner in front of Postgres

On this workload Postgres does most of the work, so the server is built around spending as little Postgres CPU per
request as possible.

**Read lanes.** Each read lane owns one Postgres connection. A lane takes every read request that is waiting in the
shared queue, of every endpoint, and answers the whole batch in two pipelined round trips:

1. **Phase A** resolves what each request is about, one statement per kind of request present in the batch. Every
   statement takes arrays of *distinct* keys (`unnest(...) WITH ORDINALITY`), so identical keys in a batch are one
   row: one global page per `(limit, offset)`, one capped count per tag, author or viewer, one lookup per slug,
   e-mail or username. Each list kind returns its capped counts and its page rows from one statement.
2. **Phase B** enriches the union of every article phase A returned, across all endpoints, once per article: tag
   ids, favorite count and the author's profile in one statement, plus the tag names in the database's sort order.
   The newest articles show up in global lists, tag pages, feeds and article pages at the same time, and are enriched
   once per batch instead of once per request. The per-viewer `favorited` / `following` flags are one primary-key
   range scan per viewer (`= ANY(...)`) instead of a probe per article.

When the server is idle a batch is one request. As load rises, the queue grows while lanes are busy, batches grow
with it, and each batch shares more work, so Postgres CPU per request goes *down* as load goes up.

Nothing outlives its batch: every value in a response was read from Postgres while that request was waiting, and
only requests already queued when the batch started share it.

**Writes** go through one write lane with group commit: every waiting write runs in one transaction
(`BEGIN`, the writes, `COMMIT`, one `Sync`), and responses are released only after `COMMIT` returns. If any statement
in a group fails, the group rolls back and each write is retried in its own transaction, so one bad write can't fail
the others. A write whose commit outcome is unknown (connection lost) is never retried. Writes that return an article
(favorite, create, update) read it through the read lanes after the commit, in a new snapshot.

**bcrypt** runs on its own threads, several hashes in lockstep: a single bcrypt is one long chain of dependent S-box
loads, and interleaving independent hashes lets the core overlap them (about 3.4x throughput at 6 lanes). Every hash
still does all 2^10 rounds; logins always verify against the stored hash.

**HTTP**: one epoll thread per `SO_REUSEPORT` listener. Reads go through a per-thread scratch buffer, so an idle
keep-alive connection holds no buffers, only its slot. Responses go out with one `writev` when the socket accepts it.
JWTs (HS256, `alg`, signature, `exp`, `nbf`) are verified on every request on the IO thread.

## Settings

Defaults suit the 8-vCPU box; all are optional environment variables.

| Variable | Default | |
|---|---|---|
| `IO_THREADS` | cores / 4 | epoll threads |
| `READ_LANES` | cores | read connections |
| `WRITE_LANES` | 1 | group-commit connections |
| `BCRYPT_THREADS` | cores / 4 | bcrypt threads |
| `BCRYPT_LANES` | 6 | hashes run in lockstep |
| `BCRYPT_WINDOW_MS` | 120 | how long a bcrypt thread waits to fill its lanes |
| `READ_BATCH`, `WRITE_BATCH` | 512, 256 | batch size caps |

Postgres connections: `READ_LANES + WRITE_LANES` (9 by default).

## Correctness

- `test/run.sh`: 197/197 on the dev and the full seed.
- A differential test of 22,000 random reads (lists with every filter and offsets up to 5,000, feeds, articles,
  comments, profiles, current user; anonymous and authenticated) against `examples/go` on the same database,
  with this server hit 64 requests at a time so batches are large and mixed: no differences.
- Unit tests for bcrypt (against the `bcrypt` crate and the seed's hash), JWT and timestamp formatting.

Built with Claude (Anthropic).

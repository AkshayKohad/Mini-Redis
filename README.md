# Mini Redis

A small Redis-inspired, concurrent in-memory key-value server written in **Java**. It accepts TCP connections, understands the Redis Serialization Protocol (RESP), and persists write commands in an append-only file (AOF).

This is a learning project focused on distributed-systems and concurrent-programming fundamentals. It is **not** a drop-in replacement for Redis.

## Highlights

- TCP server using the RESP command protocol
- Java 21 virtual threads: one lightweight task per client connection
- Thread-safe in-memory storage using `ConcurrentHashMap`
- Atomic `INCR` and `DECR` operations
- Key expiration and TTL support, including periodic expired-key cleanup
- Append-only-file persistence with recovery on server restart
- AOF compaction, using a snapshot and atomic file replacement
- Read/write locking to keep writes consistent while compacting the AOF
- No third-party runtime libraries; the project uses the Java standard library

## Supported commands

| Command | Purpose |
| --- | --- |
| `PING [message]` | Check that the server is reachable. |
| `ECHO message` | Return a message. |
| `SET key value` | Store a value. |
| `SET key value EX seconds` | Store a value with an expiry time. |
| `GET key` | Read a value. |
| `DEL key` | Delete a key. |
| `EXISTS key` | Check whether a key exists. |
| `INCR key` | Atomically increase an integer value. |
| `DECR key` | Atomically decrease an integer value. |
| `EXPIRE key seconds` | Add or change a key's expiry time. |
| `TTL key` | Return the remaining expiry time. |
| `INFO` | Show the number of keys and AOF file size. |
| `COMPACTAOF` | Rewrite the AOF using only the current data snapshot. |

## Requirements

- Java Development Kit (JDK) 21 or newer
- Optional: `redis-cli` for interactive manual testing

No Docker, database, or external Java dependency is required.

## Run the server

From the project root, compile the source files:

```bash
mkdir -p out
rg --files src/main/java -g '*.java' | xargs javac --release 21 -d out
```

Start the server on the default port, `6379`:

```bash
java -cp out com.miniredis.server.MiniRedisServer
```

Or provide a different port:

```bash
java -cp out com.miniredis.server.MiniRedisServer 6380
```

## Try it with redis-cli

In a second terminal:

```bash
redis-cli -p 6380 PING
redis-cli -p 6380 SET name Alice
redis-cli -p 6380 GET name
redis-cli -p 6380 SET session active EX 30
redis-cli -p 6380 TTL session
redis-cli -p 6380 INCR visits
redis-cli -p 6380 INFO
```

The server writes persistent commands to `data/appendonly.aof`. Restarting the server replays this file to rebuild the in-memory data.

## Run the tests

The tests are small Java programs with `main` methods, so they also require no test framework or external dependency:

```bash
mkdir -p out
rg --files src/main/java -g '*.java' | xargs javac --release 21 -d out
rg --files src/test/java -g '*.java' | xargs javac --release 21 -cp out -d out

java -cp out com.miniredis.protocol.RespReaderTest
java -cp out com.miniredis.store.InMemoryStoreTest
java -cp out com.miniredis.persistence.AppendOnlyLogTest
```

## Project structure

```text
src/main/java/com/miniredis/
├── protocol/     RESP request parsing
├── server/       TCP server and command handling
├── store/        Concurrent in-memory data store and expiration logic
└── persistence/  Append-only log, replay, and compaction
```

## How persistence works

1. Before a write changes memory, the server appends the command to the AOF and forces it to disk.
2. On startup, the server replays saved commands to restore the data.
3. `COMPACTAOF` takes a snapshot of live keys and rewrites the AOF into a smaller file.
4. An exclusive lock prevents writes from changing the store during that snapshot and replacement.

Expiry times are stored as exact timestamps in the AOF, so a key does not receive a fresh TTL after a restart.

## Planned improvements

- `APPEND`, `MGET`, `MSET`, and conditional `SET NX` / `SET XX`
- Graceful shutdown and server metrics
- Memory limits and eviction policies
- Pub/Sub
- Primary-replica replication


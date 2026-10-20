# oabs-rocksdb

Proof of concept: a Chia full node that keeps its blockchain database in RocksDB instead of SQLite.

A stock full node stores blocks, coins, and hints in a SQLite file. This tree adds a second engine for that same local database. Coins, blocks, and hints live in one RocksDB directory, split into column families, with a copy tool that reads an existing SQLite database and writes the RocksDB one. The rest of the tree is Chia Network's code from [chia-blockchain](https://github.com/Chia-Network/chia-blockchain). This repository exists so the experiment is easy to find. It is not a Chia release, and it was not opened as a pull request.

## Do not use this on mainnet

Do not point a farming node at this tree, and do not replace a live mainnet database with it.

- The database format is `rocksdb-v1`. A stock Chia node does not open it. A mistake here is a private database the rest of the network will not help you rebuild.
- Chia has not reviewed it. The consensus rules are meant to stay the same, because this is local storage, but that claim has not been accepted by the project.
- Several lookups that SQLite serves on separate reader connections still share the one RocksDB writer. A busy wallet or peer can wait behind a block save in ways the stock node does not.
- The measured sync below did not include the later file-packing code that is in this tree. The two runs also did not have the same number of peers.

Use it to read the approach and the timings. Keep a real node on the database its release shipped.

## What changed

The full node chooses a database at startup. A normal path ending in `.rocksdb` opens RocksDB. The old SQLite path is still there, including for tests whose files are not RocksDB databases.

RocksDB is organized as column families in `chia/full_node/db/keys.py`:

- coins, and indexes by confirmed height, spent height, and puzzle hash
- the per-height coin journal, used to undo a reorg
- a short index of fast-forward singletons
- full block blobs, block records, and the height-to-hash list
- the main-chain marker, sub-epoch summaries, and blocks that are not yet compact
- hints, indexed both by coin and by hint

Around that storage:

- `chia/full_node/db/rocks.py` is the RocksDB backend, including the writer and the grouped readers.
- `coin_store.py`, `block_store.py`, and `hint_store.py` are the RocksDB versions of the three stores the full node already had.
- `chain_db.py` is the small transaction layer: a read sees its own uncommitted writes, and a write commits as one batch.
- `migrate.py` copies one network's SQLite v2 database into a new `.rocksdb` directory. The SQLite file is only read. The copy needs free space of about 1.5 times the SQLite file on the destination, and some free space on the source disk because the copy holds a read view open.
- `file_chip.py` can pack old table files so the directory does not keep a huge pile of small files. That packing was not part of the sync timed below.
- `chia db migrate` can run the copy while a node is still on SQLite. The node logs that hint when it opens a SQLite database.

Tests for the stores, the migration, readers, and reorgs are under `chia/_tests/core/full_node/db/` and `chia/_tests/blockchain/`. `benchmarks/full_block_coins.py` is a local timing tool for one block of coin changes. It is not the source of the sync numbers below.

## What the sync measured

Two full syncs of the same heights, on one Windows PC, from the start of the chain through height 9,359,052. The RocksDB node is the code in this tree. The other node is the SQLite full node built as 2.7.4rc3.dev67. That SQLite build writes fewer records than release 2.7.4, so its rates are the easier of the two SQLite pictures. The coin journal for these heights holds 607,795,346 additions and 170,762,825 removals.

The RocksDB sync reached that height in 34 hours 31 minutes, at 75.3 blocks per second. The SQLite sync took 60 hours 49 minutes, at 42.7 blocks per second. That is about 76 percent faster over the whole range. Peer counts were not matched: at the end of the run the RocksDB node had about 20 peers and the SQLite node about 10. Treat the gap as what these two syncs did, not as a lab test of the database alone.

| Heights | Coins added | Coins removed | RocksDB blocks/s | SQLite blocks/s | Difference |
| --- | ---: | ---: | ---: | ---: | --- |
| 0–9,359,052 | 607,795,346 | 170,762,825 | 75.3 | 42.7 | RocksDB 76% faster |
| 1,550,000–2,000,000 | 156,174,001 | 9,728,997 | 58.8 | 18.6 | RocksDB 215% faster |
| 4,650,000–5,099,999 | 75,035,557 | 54,080,083 | 50.0 | 20.7 | RocksDB 141% faster |
| 9,200,000–9,359,052 | 170,569,285 | 1,324,603 | 19.2 | 4.0 | RocksDB 377% faster |

The last row is the sustained full-block stretch. On the RocksDB side, the median database write across the finished 50,000-block slices there was 0.64 seconds, then 0.72, then 0.78. Adding the block took 1.34 seconds, then 1.47, then 1.60. The node was not waiting on peers in that stretch.

RocksDB was not faster on every slice. On heights 0–49,999, SQLite did 79.5 blocks per second and RocksDB did 75.5, about 5 percent the other way. Empty early blocks are cheap for both. The large gaps show up where the coin table is busy.

## Where to look

- `chia/full_node/db/` holds the RocksDB engine, the three stores, migration, and file packing.
- `chia/full_node/full_node.py` is where startup opens either SQLite or RocksDB.
- `chia/cmds/` contains the `chia db migrate` command.

Upstream install and run instructions for the official node are in the [Chia docs](https://docs.chia.net/) and the [chia-blockchain](https://github.com/Chia-Network/chia-blockchain) repository. The Apache 2.0 license in `LICENSE` still covers the upstream code.

# oabs-rocksdb

Proof of concept: a Chia full node that keeps its blockchain database in RocksDB instead of SQLite.

A stock full node stores blocks, coins, and hints in a SQLite file. This tree adds a second engine for that same local database. Coins, blocks, and hints live in one RocksDB directory, split into column families, with a copy tool that reads an existing SQLite database and writes the RocksDB one. The rest of the tree is Chia Network's code from [chia-blockchain](https://github.com/Chia-Network/chia-blockchain). This repository exists so the experiment is easy to find. It is not a Chia release, and it was not opened as a pull request.

## A proof of concept

This is a proof of concept, not a vetted replacement for Chia software. It is not maintained going forward. Use it at your own risk.

It has been run on mainnet, with the farmer pointed at this full node, and that farmer has made blocks. That is one machine's experience. Chia has not reviewed the code, and this repository will not be kept up to date as Chia changes.

A stock Chia node does not open the `rocksdb-v1` database this tree writes. Several lookups that SQLite serves on separate reader connections still share the one RocksDB writer. The measured sync below did not include the later file-packing code, and the two runs did not have the same number of peers.

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
- `migrate.py` copies one network's SQLite v2 database into a new `.rocksdb` directory. The SQLite file is only read. While the copy runs, the destination needs free space of about 1.5 times that file, and the source disk needs some free space too, because the copy holds a read view open. The 1.5× is temporary. When the copy finishes, the tool packs the new files. Most column families are stored with LZ4, block bodies keep the compression they already had, and the old parent-coin index is not written. On the farming database that index had been about 25 GB. The finished directory is smaller than both the reserved space and the SQLite file it was copied from. That smaller layout is the one that has been running on mainnet. After the new directory is in use, the old SQLite file can be removed.
- `file_chip.py` can pack old table files so the directory does not keep a huge pile of small files. That packing was not part of the sync timed below.
- `chia db migrate` can run the copy while a node is still on SQLite. The node logs that hint when it opens a SQLite database.

Tests for the stores, the migration, readers, and reorgs are under `chia/_tests/core/full_node/db/` and `chia/_tests/blockchain/`. `benchmarks/full_block_coins.py` is a local timing tool for one block of coin changes. It is not the source of the sync numbers below.

## Other changes beyond the stock node

Compared with Chia 2.7.4, the same tree also changes how a long sync spends its time. The database is the large piece. These are the other ones.

**Blocks are worked on before their turn.** A long sync holds up to four later ranges, 128 blocks, and keeps at most two of those downloads in flight. Proof math for a block starts when the block arrives. The header check waits for that job instead of starting the math over. A transaction program can run early once the older programs it names are already downloaded. Up to 256 of those older programs are kept. The result is reused only when the rule flags and those older bytes match. Three failures that describe the block body, which the header hash does not cover, are not saved. A bad early copy therefore cannot make the real block fail. Each block is saved as soon as its own check finishes, in chain order, instead of waiting for the whole batch. A peer that declines a range stays connected, and the next try waits longer instead of asking that peer again immediately.

**The proof checker is called less often for the same work.** This repository does not contain the separate chiavdf library changes (the multiply scratch and the four-bit window). Those live in chiavdf, not here. What changed in this tree is when the node calls the checker it already has. The big number built from a proof challenge is remembered: one build per challenge, up to 4096 remembered. Threads that ask for the same challenge wait for that one build. Stock code kept about 200 and could start the same build many times at once. Reward-chain proofs, and challenge-chain proofs that start from the identity, are run as soon as the block is in hand. The later check still decides whether the proof belongs on the previous block. It finds the math already done.

**Startup files for the height map.** The height-to-hash map and the sub-epoch summaries can be loaded and written for a RocksDB database, so that map does not depend on the SQLite files.

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

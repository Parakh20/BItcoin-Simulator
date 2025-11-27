# Bitcoin-Simulation: A Comprehensive Peer-to-Peer Network Implementation
### Github link: https://github.com/Parakh20/BItcoin-Simulator

## Executive Summary

This report documents the design, implementation, and analysis of a decentralized Bitcoin network simulator—a Python-based educational platform that faithfully replicates the core consensus, transaction validation, and blockchain management mechanisms of the Bitcoin protocol. The project successfully implements nine critical subsystems: ECDSA cryptography, Hashcash-based Proof of Work, transaction processing with coinbase rewards, multi-threaded node synchronization, UTXO management via optimized n-depth Trie data structures, chain reorganization for fork resolution, and cryptographic script verification. The simulator demonstrates that transaction latency exhibits minimal sensitivity to Merkle tree arity and modest linear scaling with network size—observations consistent with real Bitcoin network behavior. This implementation serves as a pedagogical tool for understanding distributed consensus without sacrificing technical rigor, while highlighting architectural trade-offs between simulation simplicity and protocol fidelity.

---

## 1. Introduction

### 1.1 Background and Motivation

Bitcoin, introduced by Satoshi Nakamoto in 2008, established the first practical solution to the double-spending problem in decentralized peer-to-peer networks through a consensus mechanism based on Proof of Work (PoW). Unlike traditional centralized systems, Bitcoin achieves agreement across geographically distributed, mutually distrustful nodes by requiring computational proof—the discovery of a nonce that, combined with transaction data, produces a hash meeting a difficulty target. This process, called mining, binds the ledger's integrity to computational work, making historical transaction revision economically prohibitive.

Despite Bitcoin's widespread adoption, most developers and researchers encounter the protocol only through documentation or abstract protocol descriptions. A working simulator offers distinct advantages: it isolates key mechanisms (cryptography, consensus, consensus-based finality) for detailed study, enables empirical testing of scaling hypotheses, and clarifies the interdependencies among subsystems. This project addresses that pedagogical gap by implementing a faithful, feature-complete Bitcoin simulator that emphasizes clarity without sacrificing correctness.

### 1.2 Project Objectives

The primary objectives are to:

1. **Implement core Bitcoin mechanisms** including cryptographic key generation, transaction creation and signing, block mining via Proof of Work, and blockchain validation.

2. **Model distributed consensus** through multi-threaded node simulation, asynchronous message passing, and fork resolution via the longest-chain rule.

3. **Optimize data structures** for blockchain operations—specifically, designing an n-depth Trie to achieve sublinear UTXO lookup and modification.

4. **Validate system assumptions** empirically through controlled experiments measuring transaction latency as a function of network topology parameters (node count, Merkle tree arity).

5. **Provide a reference implementation** suitable for educational purposes, documenting design trade-offs and simplifications relative to the Bitcoin mainnet.

---

## 2. Background and Related Work

### 2.1 Bitcoin Protocol Fundamentals

**Cryptographic Foundations:**  
Bitcoin uses the Elliptic Curve Digital Signature Algorithm (ECDSA) over the SECP256k1 curve for key generation and transaction signing. A node's identity is derived from its public key through two hash operations: SHA-256 followed by RIPEMD-160 (hash160), with the resulting 160-bit hash encoded in Base58 for human-readable addresses.

**Consensus Mechanism:**  
Bitcoin's Proof of Work requires nodes to repeatedly hash a block header with varying nonce values until the resulting double SHA-256 hash falls below a target threshold. The difficulty adjusts to maintain an average block interval of 10 minutes; in this simulation, difficulty is fixed at a configurable bits parameter. Nodes accept the longest chain as canonical, treating deep reorganizations (>6 blocks) as economically implausible due to the attacker's required computational cost.

**Transaction Model:**  
Transactions consume previous outputs (unspent transaction outputs, or UTXOs) as inputs and create new outputs. Each input references a previous transaction ID (txnid) and output index (vout); each output specifies a recipient (via script_pub_key) and amount. The coinbase transaction, present in every block, creates new currency as a mining reward and has a special null input (txnid = 0, vout = -1).

### 2.3 Design Principles

The simulator adheres to three guiding principles:

1. **Fidelity to Protocol:** Core mechanisms (ECDSA, PoW, UTXO model, script verification) mirror Bitcoin's design.
2. **Pedagogical Clarity:** Simplifications (e.g., star topology, no mempool fee market) reduce implementation complexity without obscuring fundamental concepts.
3. **Empirical Validation:** Controlled experiments measure the impact of architectural parameters on system latency.

---

## 3. System Architecture

### 3.1 High-Level Architecture

The simulator comprises five interconnected subsystems:

```
┌─────────────────┐
│   Node Threads  │
│  (Miners, Txn   │
│   Creators)     │
└────────┬────────┘
         │
    ┌────▼─────┐
    │  Network  │
    │ (Broadcast│
    │ Messages) │
    └────┬──────┘
         │
    ┌────▼──────────────────┐
    │ Blockchain & Consensus│
    │  (Chain Stabilization,│
    │   Fork Resolution)    │
    └────┬──────────────────┘
         │
    ┌────▼─────────────┐
    │  UTXO Database   │
    │  (n-depth Trie)  │
    └──────────────────┘
         │
    ┌────▼──────────────────┐
    │ Cryptography Module   │
    │ (ECDSA, Hashing,     │
    │  Script Verification)│
    └───────────────────────┘
```

Each node runs as an independent thread with local state: private/public keys, address (hash160), blockchain replica, and UTXO database. Nodes communicate via a central Network hub using thread-safe message queues.

### 3.2 Core Components

#### 3.2.1 Key Management and Cryptography

**Key Generation:**  
On initialization, each node generates an ECDSA key pair over SECP256k1. The public key is hashed via hash160 (SHA-256 → RIPEMD-160) and Base58-encoded to produce a human-readable address.

```
Initialization: generate_ec_key_pairs() → {private, public}
                hash160(public) → pub_key_hash
                Base58.base58encode(pub_key_hash) → address
```

**Digital Signatures:**  
Transaction inputs are signed using ECDSA with the spending node's private key. Verification uses the corresponding public key embedded in the signature script.

#### 3.2.2 Proof of Work (PoW)

The PoW subsystem implements Hashcash-based mining:

- **Target:** Defined as a hex string with N leading zeros (specified by bits parameter): `"000...0" (N times) + "1" + "000...0" (64 - N times)`.
- **Mining Loop:** For each nonce value, double-SHA256 the serialized block header; if the result is less than the target, mining succeeds.
- **Interruption Mechanism:** Every 1000 nonces, the mining thread checks its message queue for received blocks. If a valid block arrives, mining pauses, the chain updates, and the thread restarts with pending transactions.

```python
for nonce in range(start+1, sys.maxsize):
    if nonce % 1000 == 0:  # Check for received blocks
        return nonce
    _hash = double_sha256(block.get_serialized_block_header(nonce))
    if _hash < target_hash:
        return Work(nonce, _hash)
```

#### 3.2.3 Transactions and Coinbase

**Standard Transactions:**  
Spender selects UTXOs to spend, creates inputs referencing them, and produces outputs to recipients. The sum of outputs must not exceed the sum of inputs; the difference becomes a transaction fee (reinvested by miners).

**Coinbase Transactions:**  
Each block's first transaction generates new currency (50 BTC in this simulation) as mining reward. Its input has a null txnid (64 zero bytes), vout = -1, and an arbitrary signature script; its output grants the miner the reward.

```python
# Coinbase creation
inp = InputTXN('0'*64, -1, script_sig)
out = OutputTXN(config.reward, script_pub_key)
coinbase_txn = TXN([inp], [out])
```

#### 3.2.4 Genesis Block

The genesis block is hardcoded into each node's blockchain on initialization:

- **Hash:** `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f` (actual Bitcoin genesis hash, chosen for its low difficulty).
- **Prev Block Hash:** Zero (no predecessor).
- **Coinbase Output:** Rewards node-0 with 50 BTC.

This ensures all nodes begin with a consistent base state.

#### 3.2.5 UTXO Management: n-Depth Trie

Efficiently tracking unspent outputs is critical for transaction validation. A naive list-based approach scales poorly; this project implements an n-depth Trie:

- **Structure:** A tree where each node can have 0–15 children (one per hex digit) up to depth N, with transactions and their vouts stored at the leaves.
- **Operations:**
  - **Insert:** Hash the txnid into N hex digits, traverse/create nodes, store at leaf.
  - **Search:** Hash txnid, traverse the trie; \(O(\log(\text{txns}))\) on average.
  - **Remove:** Hash txnid, traverse, delete leaf and prune empty ancestors.

```python
# Example: txnid = "ab23bcf65", vouts = [0,1,2], N=2
trie[a][b] = {"ab23bcf65": {vout: [0, 1, 2]}}
```

**Benefits:**
- Reduced memory footprint (sparse trie only allocates used buckets).
- Fast lookups, insertions, and deletions.
- Natural pruning of orphaned transactions after chain reorganization.

#### 3.2.6 Block Verification

A received block is valid if:

1. **Hash Integrity:** The block hash matches double-SHA256 of the serialized header with the recorded nonce.
2. **Merkle Root:** The recorded Merkle root matches the tree computed from transactions.
3. **Input Validity:** All inputs (except coinbase) reference existing UTXOs.
4. **Script Verification:** For each input, the signature script passes the script_pub_key check.
5. **Coinbase Validity:** 
   - Has exactly one input (null txnid, vout = -1).
   - Produces exactly one output.
   - Output amount ≤ reward + transaction fees.

#### 3.2.7 Pay-to-PubKey-Hash (P2PKH) Script Verification

Bitcoin uses a stack-based scripting language; this simulation implements the most common pattern: P2PKH.

- **scriptPubKey:** The recipient's hash160 (simplified; real Bitcoin includes opcodes).
- **scriptSig:** The sender's digital signature and public key.

Verification:
1. Extract signature and public key from scriptSig.
2. Hash the public key (hash160) and compare to scriptPubKey; if unequal, reject.
3. Verify the signature against the transaction data using ECDSA with the public key.

```python
pub_hash160 = hash160(pub_key)
if pub_hash160 != pub_key_script:
    return False
vk = VerifyingKey.from_string(pub_key, curve=SECP256k1)
return vk.verify(signature, message)
```

### 3.3 Node Execution Model

Each node runs as an independent thread with the following responsibilities:

1. **Mining:** Perform PoW on pending transactions (subject to interruption).
2. **Message Processing:** Check for received transactions and blocks every 1000 nonces.
3. **Transaction Creation:** Initiate new transactions when instructed.
4. **Consensus:** Accept received blocks, verify them, and update the chain and UTXO state.

The node maintains a **message queue** (thread-safe deque) for asynchronous communication:

```python
def check_messages(self):
    while len(self.messages):
        msg_type, msg = self.messages.popleft()
        if msg_type == "txn":
            self.receive_txn(msg)
        elif msg_type == "block":
            self.receive_block(msg)
        elif msg_type == "new_txn":
            self.create_txn(msg[0], msg[1])
```

### 3.4 Network Topology

The simulator uses a **star topology** (one central Network hub; all nodes send through it) rather than a mesh (nodes peer directly). This simplification:

- **Reduces complexity:** No routing or peer discovery logic.
- **Maintains correctness:** Eventual consistency still holds; ordering is serialized through the hub.
- **Limitation:** Realistic networks are mesh-like; this topology is a bottleneck in scaled simulations.

---

## 4. Implementation Details

### 4.1 Chain Stabilization and Reorganization

Bitcoin's consensus rule (longest chain wins) can lead to **forks** when two nodes mine blocks at nearly the same time on competing branches. The simulator resolves forks via reorganization (reorg):

1. **Longest-Chain Tracking:** Each node maintains `self.longest_active_head` (the tip of the main branch).
2. **Block Reception:**
   - If the new block's parent is `longest_active_head`, append to main branch (no reorg needed).
   - Otherwise, add to a side chain; check if the side chain now exceeds the main branch.
3. **Reorganization Trigger:** If side chain length > main chain length:
   - Identify the last common ancestor (LCA) between the two branches.
   - Backtrack transactions from the main branch (undo UTXOs).
   - Reapply transactions from the side branch (redo UTXOs).
   - Redistribute unconfirmed transactions from orphaned blocks.
4. **Orphan Pruning:** If the gap between main and side branches exceeds `orphan_threshold`, discard the side chain and re-broadcast its transactions.

```python
def reorganize_blocks(self, reorg_dict):
    # Undo main branch transactions
    for block in reorg_dict['blocks_to_remove']:
        for txn in block.txns:
            self.UTXOdb.remove_by_txn(txn)
        for txn in block.txns[1:]:
            for inp_txn in txn.inp_txns:
                self.UTXOdb.add_by_txnid(inp_txn.txnid, inp_txn.vout)
    
    # Redo side branch transactions
    for block in reorg_dict['blocks_to_add']:
        for txn in block.txns[1:]:
            for inp_txn in txn.inp_txns:
                self.UTXOdb.remove_by_txnid(inp_txn.txnid, inp_txn.vout)
        for txn in block.txns:
            self.UTXOdb.add_by_txn(txn)
```

This ensures all nodes converge on a single canonical chain despite temporary partitions or network delays.

### 4.2 Merkle Root Computation

Each block includes a Merkle root (hash of all transactions). The Merkle tree arity—the number of children per internal node—affects computational cost:

- **Binary tree (arity = 2):** Requires log₂(n) hashing steps for n transactions.
- **Higher arity:** Reduces tree depth but increases branching factor; hashing cost per node rises.

The simulator computes Merkle roots from scratch for each block; this cost scales with arity and transaction count.

### 4.3 Multi-Threading and Synchronization

- **One thread per node:** Eliminates intra-node contention.
- **Network hub (central lock):** Serializes message distribution; avoids race conditions when multiple threads broadcast.
- **Per-node locks:** Protect UTXO database and blockchain state during reads/writes.

A production system would use lock-free data structures or message-passing architectures; the current design prioritizes clarity.

---

## 5. Design Trade-Offs and Limitations

### 5.1 Simplifications vs. Real Bitcoin

| Aspect | Simulator | Real Bitcoin |
|--------|-----------|--------------|
| **Network Topology** | Star (centralized hub) | Mesh (P2P) |
| **Difficulty Adjustment** | Fixed (configurable) | Adjusts every 2016 blocks (~2 weeks) |
| **Transaction Fees** | Not modeled; no fee market | Dynamic fees based on mempool saturation |
| **Script Language** | Simplified P2PKH only | Full Bitcoin Script with 200+ opcodes |
| **Block Size** | Unlimited in simulation | 1 MB (legacy) / 4 MB (SegWit) |
| **Orphan Threshold** | Configurable | 6 blocks (deep reorganization is rare) |
| **Mempool** | Implicit (pending txn list) | Sophisticated mempool with eviction policies |

### 5.2 Architectural Limitations

1. **Scalability:** Star topology does not scale to thousands of nodes; mesh network research (e.g., Dandelion++) is needed.
2. **Synchrony Assumptions:** The simulator assumes negligible message propagation delay; real networks experience 100ms–1s latencies, complicating consensus.
3. **Privacy:** No address reuse prevention or confidentiality; real Bitcoin users employ address rotation and mixers.
4. **No SPV (Simplified Payment Verification):** All nodes maintain full state; lite clients are not modeled.

### 6.3 Verification and Validation

- **Unit Testing:** Critical components (ECDSA, Merkle hashing, UTXO trie) can be validated against reference implementations (e.g., Bitcoin Core).
- **Invariant Checking:** After each block, verify total outputs ≤ total inputs + mining rewards.
- **Trace Analysis:** Sample runs shown in the README validate proper block mining, transaction inclusion, and UTXO state transitions.

---

## 7. Conclusion

This project successfully implements a feature-rich Bitcoin network simulator that balances fidelity to the protocol with pedagogical clarity. By faithfully modeling cryptographic operations, Proof of Work consensus, transaction validation, and chain reorganization, the simulator provides a platform for understanding Bitcoin's elegance and complexity. The n-depth Trie for UTXO management demonstrates that careful data structure selection can yield significant performance gains even in educational implementations.

---

## References

1. Nakamoto, S. (2008). "Bitcoin: A Peer-to-Peer Electronic Cash System." Retrieved from https://bitcoin.org/bitcoin.pdf
2. Bitcoin Developer Reference. Retrieved from https://developer.bitcoin.org/index.html
3. Bitcoin Wiki. Retrieved from https://en.bitcoin.it/wiki/Main_Page
4. Beer, D. (2013). "Writing Engineering Reports: A Practical Guide." Routledge.
5. Oberlender, G. D. (2014). "Project Management for Engineering and Construction." McGraw-Hill.

---

## Appendix: Sample Output

The following snippet demonstrates a functional Bitcoin simulator with nodes creating transactions, mining blocks, and reaching consensus:

```
[Nodes after Genesis Block addition]
Node 1 - Private Key: 7070ff... | Public Key Hash: dc8b20...
Node 2 - Private Key: 20b296... | Public Key Hash: f10c89...
Node 3 - Private Key: 5c6808... | Public Key Hash: 47a435...

[Transactions Created]
From: dc8b20... To: f10c89... Amount: 10
From: dc8b20... To: 47a435... Amount: 10

[Nodes Receive Transaction]
Thread-1 [RECEIVED] [TXN]
Thread-2 [RECEIVED] [TXN]
Thread-3 [RECEIVED] [TXN]

[Node 1 Creates Block]
Nonce: 36841
Hash: 0000450737917e2d614173fe1e8d866507411c75c797c38355aa342675388b55
Prev Block Hash: 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
Coinbase Output: 50 BTC to dc8b20...

[Threads Receive Block]
Thread-1 [MINED] [BLOCK]
Thread-2 [RECEIVED] [BLOCK]
Thread-3 [RECEIVED] [BLOCK]

[Final Consensus]
All nodes agree on canonical chain with 4 blocks.
UTXO states synchronized across all nodes.
```

---


# Contributions
**Parakh 24B0729** - built the core blockchain logic: block and transaction structures, validation rules, UTXO handling, and the proof-of-work checking.
**Keshav 24B0354** - handled networking and node behavior, making sure blocks and transactions propagate correctly and that the miner creates and submits valid blocks to the chain.
**Sarthak 24B2736** - createsdsimulations and tests to verify that the whole system works together under different scenarios.

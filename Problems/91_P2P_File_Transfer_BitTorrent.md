# 91. Design a P2P File Transfer System (like BitTorrent)

---

## 1. Functional Requirements (FR)

- Distribute large files (100MB–100GB) across thousands of peers without central server bottleneck
- File split into fixed-size pieces (256KB–4MB); each piece independently downloadable
- Peer discovery: find peers who have pieces of the desired file
- Piece selection: download rarest pieces first (rarest-first strategy)
- Tit-for-tat: preferentially upload to peers who upload to you (incentive mechanism)
- Torrent file / magnet link: metadata describing file, piece hashes, tracker URL
- Distributed Hash Table (DHT): trackerless peer discovery (Kademlia)
- Integrity verification: SHA-1 hash per piece to detect corruption
- Resume interrupted downloads; partial file sharing while downloading
- Seed management: continue uploading after download completes

---

## 2. Non-Functional Requirements (NFRs)

- **Scalability**: More downloaders = more bandwidth (opposite of client-server!)
- **Fault Tolerance**: Any peer can leave at any time without affecting others
- **Decentralization**: No single point of failure (DHT for trackerless mode)
- **Integrity**: Corrupted pieces detected and re-downloaded from another peer
- **Fairness**: Tit-for-tat prevents free-riding

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| File size | 4 GB (typical movie) |
| Piece size | 2 MB → 2,000 pieces |
| Peers in swarm | 10,000 |
| Seeders (have complete file) | 2,000 |
| Leechers (downloading) | 8,000 |
| Per-peer upload capacity | 1 Mbps avg |
| Total swarm upload capacity | 10,000 × 1 Mbps = 10 Gbps |
| Download time (single peer) | 4 GB / 10 Mbps combined = ~30 min |
| DHT nodes (global) | 10M+ |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   TORRENT ECOSYSTEM                                    │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  TRACKER (Optional — Centralized Peer Discovery)           │        │
│  │                                                            │        │
│  │  - Maintains list of peers per torrent (info_hash)         │        │
│  │  - Peer announces: "I have this torrent, here's my IP"    │        │
│  │  - On request: returns random subset of peers (50-200)     │        │
│  │  - Does NOT transfer any file data                         │        │
│  │  - Stateless: just a rendezvous point                     │        │
│  │  - Can be replaced by DHT (trackerless mode)               │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  DHT NETWORK (Kademlia — Distributed Hash Table)           │        │
│  │                                                            │        │
│  │  - Every peer is a DHT node (stores subset of peer lists)  │        │
│  │  - Key: info_hash → Value: list of peers with that torrent │        │
│  │  - Lookup: O(log N) hops to find responsible node          │        │
│  │  - announce_peer: "I have torrent X" → stored in DHT       │        │
│  │  - get_peers: "Who has torrent X?" → returns peer list     │        │
│  │  - No central server needed (fully decentralized)          │        │
│  │                                                            │        │
│  │  Node ID: 160-bit random (same space as SHA-1 info_hash)  │        │
│  │  Distance: XOR metric — d(A,B) = A ⊕ B                   │        │
│  │  Routing table: K-buckets (k=8 peers per distance range)  │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  PEER A (Leecher → Seeder)                                 │        │
│  │                                                            │        │
│  │  ┌──────────────────────────────────────┐                  │        │
│  │  │  BitTorrent Client                    │                  │        │
│  │  │                                        │                  │        │
│  │  │  1. Parse .torrent file / magnet link │                  │        │
│  │  │     → Get info_hash, piece hashes,    │                  │        │
│  │  │       file names, tracker URL          │                  │        │
│  │  │                                        │                  │        │
│  │  │  2. Contact tracker / DHT              │                  │        │
│  │  │     → Get list of peers                │                  │        │
│  │  │                                        │                  │        │
│  │  │  3. Connect to peers (TCP)             │                  │        │
│  │  │     → Exchange bitfield:               │                  │        │
│  │  │       "I have pieces [1,3,7,42]"      │                  │        │
│  │  │                                        │                  │        │
│  │  │  4. Request pieces from peers          │                  │        │
│  │  │     → Rarest-first strategy            │                  │        │
│  │  │     → Download from multiple peers     │                  │        │
│  │  │       simultaneously (pipelining)      │                  │        │
│  │  │                                        │                  │        │
│  │  │  5. Verify each piece (SHA-1 hash)     │                  │        │
│  │  │     → If valid: mark as complete       │                  │        │
│  │  │     → If corrupt: re-download          │                  │        │
│  │  │                                        │                  │        │
│  │  │  6. Upload pieces to other peers       │                  │        │
│  │  │     → Tit-for-tat: prioritize peers    │                  │        │
│  │  │       who upload to you                │                  │        │
│  │  │     → Optimistic unchoking: randomly   │                  │        │
│  │  │       try new peers every 30s          │                  │        │
│  │  │                                        │                  │        │
│  │  │  7. All pieces complete → become Seeder│                  │        │
│  │  └──────────────────────────────────────┘                  │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Peer B ─── TCP ──── Peer A ─── TCP ──── Peer C                      │
│       ↕                    ↕                    ↕                       │
│  Peer D ─── TCP ──── Peer E ─── TCP ──── Peer F                      │
│                                                                        │
│  Each peer simultaneously uploads AND downloads different pieces      │
│  A downloads piece 42 from B while uploading piece 7 to D            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Piece Selection: Rarest-First Algorithm

```
Problem: If everyone downloads piece 1 first, piece 1 becomes common
         and rare pieces might become unavailable if the only seeder leaves

Rarest-first:
  1. Track which pieces each connected peer has (via bitfield + have messages)
  2. Count availability of each piece across all connected peers
  3. Download the piece with LOWEST availability first
  4. Exception: first few pieces → random (get something to upload ASAP)

Example:
  Piece 1: available from 50 peers  → low priority
  Piece 42: available from 2 peers  → HIGH priority (download NOW)
  Piece 99: available from 1 peer   → CRITICAL priority
  
  If the 1 peer with piece 99 leaves → piece 99 lost forever → swarm can't complete
  Rarest-first prevents this scenario
```

### Tit-for-Tat (Choking Algorithm)

```
Problem: Free-riders (download but never upload) degrade the swarm

Solution: Preferentially upload to peers who reciprocate
  1. Every 10 seconds, unchoke top 4 peers by upload rate to you
  2. Every 30 seconds, optimistic unchoke: randomly unchoke 1 peer
     (gives new peers a chance to prove themselves)
  3. All other peers are "choked" (you won't upload to them)

Result:
  - Fast uploaders get fast downloads (reciprocity)
  - Slow/no uploaders get slow downloads
  - New peers get bootstrapped via optimistic unchoke
  - Nash equilibrium: everyone uploads → everyone benefits
```

---

## 5. Protocol (BitTorrent Wire Protocol)

```
# Peer handshake (TCP)
<pstrlen=19><pstr="BitTorrent protocol"><reserved=8 bytes><info_hash=20 bytes><peer_id=20 bytes>

# Messages (after handshake):
choke          → "I won't upload to you"
unchoke        → "I will upload to you"
interested     → "I want pieces you have"
not interested → "I don't need anything from you"
have           → "I just completed piece #42"
bitfield       → "Here's a bitmap of all pieces I have" [1,0,1,1,0,0,1,...]
request        → "Send me piece #42, offset 0, length 16384"
piece          → "Here's the data for piece #42" (16 KB sub-piece)
cancel         → "Nevermind, don't send piece #42" (endgame mode)

# DHT messages (UDP, Kademlia):
ping           → "Are you alive?"
find_node      → "Give me nodes closest to ID X"
get_peers      → "Give me peers for torrent info_hash X"
announce_peer  → "I have torrent X, remember my IP"
```

---

## 6. Data Models

### .torrent File (Bencoded)

```
{
  "announce": "http://tracker.example.com/announce",
  "info": {
    "name": "movie.mkv",
    "piece length": 2097152,     // 2 MB
    "pieces": "<concatenated SHA-1 hashes of all pieces>",  // 20 bytes per piece
    "length": 4294967296,        // 4 GB (single file)
    // OR for multi-file:
    "files": [
      {"length": 4294967296, "path": ["movie.mkv"]},
      {"length": 1024, "path": ["subtitles.srt"]}
    ]
  }
}

info_hash = SHA-1(bencoded info dict) → 20-byte identifier for the torrent
```

### Kademlia DHT Routing Table

```
Node ID: 160-bit random number
K-buckets: 160 buckets, one for each bit-distance range

Bucket i: stores up to 8 nodes with distance 2^i to 2^(i+1) from self
  Bucket 0: nodes with exactly the same 159-bit prefix (very close)
  Bucket 159: nodes differing in the first bit (very far)

Lookup (find peer for info_hash):
  1. Compute XOR distance from info_hash to known nodes
  2. Ask closest known node for closer nodes
  3. Repeat until no closer nodes found (converges in ~O(log N) hops)
  4. Nodes closest to info_hash store peer lists for that torrent
```

---

## 7. Fault Tolerance & Deep Dives

### Endgame Mode
```
Problem: Last few pieces take forever (only available from slow peers)
Solution: When <5% remaining, request ALL missing pieces from ALL peers
  → First response wins, cancel duplicate requests
  → Ensures fast completion of last pieces
```

### Peer Churn
```
Peers constantly join and leave:
- Tracker: re-announce every 30 min (or on start/stop)
- DHT: announce every 15 min; nodes timeout after 15 min of no contact
- Client: maintain 20-50 connections; replace disconnected peers
- Swarm health: as long as at least 1 copy of each piece exists, swarm survives
```

### Piece Corruption
```
Every piece verified with SHA-1 hash (from .torrent file)
If hash mismatch → discard, re-download from different peer
Ban peers that repeatedly send corrupt data (IP blocklist)
```

---

## 8. Additional Considerations

### Why P2P > Client-Server for Large Files
```
Client-Server: server bandwidth = bottleneck, cost ∝ number of downloaders
P2P: more downloaders = more aggregate bandwidth → faster for everyone
  10,000 peers × 1 Mbps each = 10 Gbps total upload (no server needed)
```

### Magnet Links (vs .torrent files)
```
magnet:?xt=urn:btih:<info_hash>&dn=<display_name>&tr=<tracker_url>
  - No .torrent file needed (just the 20-byte info_hash)
  - Piece hashes fetched from peers via extension protocol (BEP 9)
  - DHT used for initial peer discovery
  - Fully decentralized: no tracker, no .torrent hosting needed
```

### WebTorrent (Browser-Based P2P)
```
Regular BitTorrent: TCP connections (not available in browsers)
WebTorrent: uses WebRTC data channels for peer connections
  - Fully browser-compatible P2P file sharing
  - WebRTC signaling via WebSocket tracker
  - Same piece selection and tit-for-tat algorithms
  - Use case: streaming video P2P to reduce CDN costs
```

---

## 9. Deep Dive: Engineering Trade-offs

### Kademlia DHT Lookup — Step-by-Step Walkthrough

```
Setup (simplified with 8-bit IDs for illustration):
  Our node:    NodeA = 01101001
  Target:      info_hash = 01100011  (we want peers for this torrent)
  XOR distance: 01101001 ⊕ 01100011 = 00001010 (= 10 in decimal)
  
  Our routing table (K-buckets, k=8 nodes per bucket):
    Bucket 7 (distance 128-255): [11010010, 10110101, ...]  ← far nodes
    Bucket 6 (distance 64-127):  [01010011, ...]
    Bucket 5 (distance 32-63):   [...]
    Bucket 4 (distance 16-31):   [01110100, 01111001, ...]
    Bucket 3 (distance 8-15):    [01100000, 01101110, ...]   ← closest bucket!
    Buckets 0-2:                 [sparse — few nodes this close]

  Iteration 1: Pick α=3 closest known nodes to target
    01100000 ⊕ 01100011 = 00000011 (dist=3)  ← closest
    01101110 ⊕ 01100011 = 00001101 (dist=13)
    01110100 ⊕ 01100011 = 00010111 (dist=23)
    
    Send find_node(01100011) to all 3 in parallel (UDP)
    
  Iteration 2: Node 01100000 responds with ITS closest nodes:
    [01100001, 01100010, 01100100]
    
    01100001 ⊕ 01100011 = 00000010 (dist=2)  ← CLOSER!
    01100010 ⊕ 01100011 = 00000001 (dist=1)  ← EVEN CLOSER!
    
    Send find_node(01100011) to these new closer nodes
    
  Iteration 3: Node 01100010 responds:
    [01100011]  ← This IS the target (or closest known)!
    
    This node (01100011) is responsible for storing peer lists
    Send get_peers(info_hash) → receives list of peers with the torrent
    
  Total hops: 3 iterations = O(log₂ 256) = O(8) for 8-bit ID space
  In real BitTorrent (160-bit IDs, 10M nodes): O(log₂ 10M) ≈ 23 hops max
  In practice: 4-8 hops (routing tables are well-populated)
```

### Routing Table Maintenance

```
K-bucket management rules:

1. On receiving ANY message from node X:
   - Compute distance = our_id ⊕ X_id → determines bucket index
   - If X is already in the bucket → move to tail (most recently seen)
   - If bucket not full (< k=8 nodes) → add X to tail
   - If bucket IS full:
     a. Ping the LEAST recently seen node (head of list)
     b. If it responds → keep it (prefer long-lived nodes), discard X
     c. If it doesn't respond → evict it, add X
   
   Why prefer long-lived nodes?
     Empirical observation: nodes that have been online for 1 hour
     are likely to stay online for another hour. Fresh nodes may leave
     in minutes. Preferring old nodes → stable routing table.

2. Bucket refresh (background, every 15 minutes):
   - For each bucket that hasn't had a lookup in 15 min:
   - Generate a random ID within that bucket's range
   - Perform a find_node lookup for that random ID
   - This populates the bucket with fresh nodes
   
3. Node join procedure:
   a. New node N generates random 160-bit node ID
   b. N knows at least ONE existing node (bootstrap node, hardcoded)
   c. N performs find_node(own_id) → discovers nodes close to itself
   d. N's routing table fills up as responses come in
   e. N refreshes ALL buckets → fully populated within ~1 minute

4. Bucket splitting (optimization):
   - If our OWN bucket (distance 0, closest to our ID) is full
     AND the new node falls in our bucket → split into two buckets
   - Allows finer granularity for nearby nodes
   - Far buckets stay at k=8 (don't need fine granularity)
```

### Choking/Unchoking Algorithm — State Machine

```
Each peer connection has 4 boolean states:
  am_choking:      We are choking them (not uploading)
  am_interested:   We are interested in their pieces
  peer_choking:    They are choking us (not uploading to us)
  peer_interested: They are interested in our pieces

Data transfer happens ONLY when:
  We unchoke them AND they are interested → we upload to them
  They unchoke us AND we are interested  → they upload to us

State transitions:
  ┌──────────────────────────────────────────────────────────────┐
  │ CONNECTION ESTABLISHED                                        │
  │  Initial state: am_choking=true, am_interested=false          │
  │                 peer_choking=true, peer_interested=false       │
  └──────┬───────────────────────────────────────────────────────┘
         │
  ┌──────▼───────────────────────────────────────────────────────┐
  │ BITFIELD EXCHANGE                                             │
  │  Both peers send their piece availability                     │
  │  If they have pieces we need → send INTERESTED               │
  │  If we have pieces they need → they send INTERESTED           │
  └──────┬───────────────────────────────────────────────────────┘
         │
  ┌──────▼───────────────────────────────────────────────────────┐
  │ EVERY 10 SECONDS: Regular Unchoke Decision                    │
  │                                                                │
  │  1. Rank all interested peers by their upload speed TO US     │
  │  2. Unchoke top 4 peers (they become our upload slots)        │
  │  3. Choke everyone else                                        │
  │                                                                │
  │  Result: peers who give us the most get the most back          │
  └──────┬───────────────────────────────────────────────────────┘
         │
  ┌──────▼───────────────────────────────────────────────────────┐
  │ EVERY 30 SECONDS: Optimistic Unchoke                          │
  │                                                                │
  │  1. Randomly pick 1 choked+interested peer                    │
  │  2. Unchoke them (5th upload slot)                             │
  │  3. Give them 30 seconds to prove their upload speed           │
  │  4. At next 10s evaluation, they compete with regular slots   │
  │                                                                │
  │  Purpose: bootstrapping — new peers have no upload history,   │
  │  so they'd never be unchoked without the random chance.        │
  └──────────────────────────────────────────────────────────────┘

Concrete timeline example:
  T=0s:   PeerA joins swarm, connects to 20 peers. All choke PeerA.
          PeerA has no pieces yet, so nobody is interested in PeerA either.
  T=30s:  PeerB optimistically unchokes PeerA → PeerA downloads piece #42
  T=34s:  PeerA now HAS piece #42 → announces HAVE to all peers
          PeerC is interested in #42 → PeerC sends INTERESTED to PeerA
  T=40s:  PeerB's 10s evaluation: PeerA uploaded 0 bytes to PeerB →
          PeerA is choked again (lost the optimistic slot)
  T=40s:  PeerC's 10s evaluation: PeerA uploaded piece #42 to PeerC fast →
          PeerC unchokes PeerA as regular slot (reciprocity!)
  T=60s:  PeerD optimistically unchokes PeerA → PeerA gets more pieces
          PeerA now has 3 pieces, is uploading to PeerC → stable position
  T=120s: PeerA has 15 pieces, is one of the fastest uploaders →
          multiple peers unchoke PeerA → download speed maxes out

Seeder behavior (special case):
  Seeders have all pieces → nobody uploads TO them → can't rank by reciprocity
  Instead: unchoke peers with fastest DOWNLOAD rate (help them finish fastest
  so they become seeders sooner → grows total swarm upload capacity)
```

### Why Piece Size Matters: Small vs Large Pieces

```
Small pieces (256 KB):
  ✓ Fine-grained piece selection → better rarest-first distribution
  ✓ Faster initial upload capability (complete a piece quickly)
  ✓ Less data wasted if piece fails hash check
  ✗ More SHA-1 hashes in .torrent file (16 KB per piece × 16,000 pieces = 256 KB)
  ✗ More protocol overhead (more REQUEST/PIECE messages)
  ✗ More seeks on HDD (random access per piece)

Large pieces (4 MB):
  ✓ Smaller .torrent file
  ✓ Less protocol overhead
  ✓ Better sequential disk I/O
  ✗ Slower to complete first piece (can't upload until piece complete + verified)
  ✗ More data wasted on hash failure

Industry standard: 256 KB – 2 MB (adaptive based on total file size)
  Files < 500 MB: 256 KB pieces
  Files 500 MB – 4 GB: 1 MB pieces
  Files > 4 GB: 2-4 MB pieces
```


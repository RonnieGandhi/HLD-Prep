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


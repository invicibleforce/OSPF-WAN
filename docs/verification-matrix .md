# Verification Matrix — Multi-Area OSPF WAN with NAT

Complete test plan validating OSPF routing, authentication, NAT/PAT, and dual-homed redundancy.

---

## Section A — OSPF Adjacencies & Routing

| # | From | Command | Expected |
|---|---|---|---|
| A1 | R-HQ | `show ip ospf neighbor` | B1 (2.2.2.2), B2 (3.3.3.3), B3 (4.4.4.4) all FULL |
| A2 | R-B1 | `show ip ospf neighbor` | R-HQ + R-B2 (redundant link) both FULL |
| A3 | R-HQ | `show ip route ospf` | Branch LANs as O IA inter-area routes |
| A4 | R-B1 | `show ip route ospf` | Other LANs + O*E2 default route present |
| A5 | R-HQ | `show ip route 10.2.0.0` | Via Serial (NOT Null0) |

## Section B — Authentication

| # | From | Command | Expected |
|---|---|---|---|
| B1 | R-HQ | `show ip ospf interface se0/3/0` | "Message digest authentication enabled" |
| B2 | any | mismatch key test | Adjacency drops until both ends match |

## Section C — NAT/PAT & Internet

| # | From | Command | Expected |
|---|---|---|---|
| C1 | R-HQ | `ping 203.0.113.10` | Replies (R-HQ sources from public IP) |
| C2 | B1-PC1 | `ping 203.0.113.10` | Replies (via NAT translation) |
| C3 | R-HQ | `show ip nat translations` | 10.2.0.10 → 198.51.100.x entry |
| C4 | R-HQ | `show ip nat statistics` | Se0/2/0 outside; branch serials + Gi0/0 inside |

## Section D — Cross-Branch Connectivity

| # | From | Command | Expected |
|---|---|---|---|
| D1 | B1-PC1 | `ping 10.3.0.1` | Replies (B1 → B2 LAN) |
| D2 | B1-PC1 | `ping 10.4.0.1` | Replies (B1 → B3 LAN) |
| D3 | B2-PC1 | `ping 10.1.0.1` | Replies (B2 → HQ LAN) |

## Section E — Dual-Homed Failover (the hardening)

| # | Action | Command | Expected |
|---|---|---|---|
| E1 | Baseline | `show ip route 10.3.0.0` (R-B1) | Route present via HQ |
| E2 | Shut primary HQ↔B2 (Se0/3/1 on R-HQ) | — | OSPF reconverges |
| E3 | After shut | `show ip route 10.3.0.0` (R-B1) | Route SURVIVES (via secondary link) |
| E4 | During failover | `ping 10.3.0.1` (B1-PC1) | Pings resume after brief drop |
| E5 | Restore | `no shutdown` Se0/3/1 | Route returns to primary path |

---

## My Verification Results

```
DATE: 2026-06-01
TESTER: Kwadwo

Section A — OSPF Adjacencies & Routing
  A1 ✅  A2 ✅  A3 ✅  A4 ✅  A5 ✅
  → All three branches FULL. Redundant B1↔B2 adjacency FULL.
    Inter-area routes and default route propagating correctly.
    Confirmed 10.2.0.0 routes via serial, not Null0 (after removing summarization).

Section B — Authentication
  B1 ✅  B2 ✅
  → MD5 confirmed active on WAN interfaces. Adjacencies flap and
    recover correctly when keys match on both ends.

Section C — NAT/PAT & Internet
  C1 ✅  C2 ✅  C3 ✅  C4 ✅
  → Branch PCs reach the internet web server through PAT.
    Translation table shows 10.2.0.10 mapped to public pool address.

Section D — Cross-Branch Connectivity
  D1 ✅  D2 ✅  D3 ✅
  → Full any-to-any reachability across all areas.

Section E — Dual-Homed Failover (hardening)
  E1 ✅  E2 ✅  E3 ✅  E4 ✅  E5 ✅
  → THE KEY RESULT: with the dual-homed Area 2 secondary link in place,
    shutting the primary HQ↔B2 link no longer black-holes B2's LAN.
    Route survives, traffic reroutes over the secondary link, pings resume.
    This is the failover the original Area 1 redundant link could NOT provide.

Overall: PASS — full routing, security, internet access, and
true route-level redundancy for Branch 2.

Notes:
- Original Area 1 redundant link provides adjacency redundancy only —
  it cannot carry B2's Area 2 LAN route when HQ↔B2 drops, because
  Area 2 loses its backbone connection (orphaned area).
- Dual-homing B2 to HQ with both links in Area 2 fixes this:
  the backup path lives in the same area as the protected LAN.
- Route summarization was removed: summarizing a /24 as a /24
  created a Null0 discard route that black-holed real traffic.
- This build exceeds the reference design by correcting a genuine
  redundancy weakness in the original topology.
```

---

## Common Failure Modes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Serial link stays down | Clock rate on wrong (DTE) end | Move clock rate to DCE end; verify with `show controllers` |
| Neighbor never reaches FULL | MD5 key mismatch | Confirm `0spfK3y!` identical both ends (case-sensitive) |
| Branch can't reach internet | Default route not propagated | Verify `default-information originate` + static default exist |
| NAT translates but no reply | Return path / interface tags | Check inside/outside tags and R-ISP default route |
| `% Subnet not in table` after summarization | Null0 discard route | Remove `area X range` for single-/24 areas |
| PC reaches own LAN only | Missing default gateway | Set gateway to the local router's LAN IP |
| Failover orphans B2 LAN | Backup link in wrong area | Dual-home in the protected LAN's area (Area 2) |

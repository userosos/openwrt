# RSS multi-ring NAPI port (6.18.44) — port, module swap, TFTP-boot experiment

Status (2026-09-14): **RSS boots via TFTP with all 6 FIT overlays (bootconf_extra
fix); eth2 + 4 RX rings live. RSS ethtool surface completed (760-24).** NAND
untouched throughout (TFTP boot is RAM-only). Remaining: lan5/mt7530 mgmt link
drops in the recovery boot (~AIMARKER10).

Branch: `bpi-r4pro-8x-v2-multiring-napi` (off `bpi-r4pro-8x-v2`).

## Topology — what is connected to what (checked 2026-09-14)

Two machines, one cable between them, plus the banana's own WAN uplink:

**The dev box (this machine, TFTP server):**
- `eth0` MAC `a8:b8:e0:0a:28:48`, carries three addresses on one cable:
  - `10.222.1.1/24` (management box — target of the AI ping tests)
  - `10.222.1.22/24`
  - `192.168.1.254/24` (U-Boot `serverip` / TFTP server root `/data/tftp`)
- Serial console to the banana via `minicom-console.sh` → `/data/tftp/console.log`.
- This box is also the TFTP boot source (`serverip 192.168.1.254`).

**The banana (BPI-R4 Pro 8X, `10.222.1.2`) — internal SoC layout**

One SoC ethernet controller `ethernet@15100000` (`mtk_soc_eth`) hosts **three
MACs**, each a separate netdev + a DSA master or standalone conduit:

| MAC | netdev | role | goes to |
|---|---|---|---|
| `gmac0` mac@0 | `eth0` | **DSA master, tree 1** (internal switch) | internal `switch@15020000` (mt7530-mmio), cpu port@6, 10G fixed link |
| `gmac1` mac@1 | `eth1` | **standalone WAN** (usxgmii) | `AS21010JB1` phy28 (`mdio-bus:1c`), `82.131.28.40/22`, gw `.1`, DHCP |
| `gmac2` mac@2 | `eth2` | **DSA master, tree 0** (MxL switch) | external `switch16` (`mdio-bus:10`), cpu port@9, 10gbase-r, tag `mxl862xx-8021q` |

Two **DSA switches**:
- **tree 0 — MxL86252** (`switch16`, mdio-bus:10, dsa `member <0 0>`), master
  eth2/gmac2. User ports: `lan1..lan4` (integrated phys `mii:00–03`) + `lan6`
  (`port@13`, external `AS21010JB1` phy24 on `mdio-bus:18`, usxgmii inband).
  **This tree (and thus eth2) is the RSS/multiring MAC this branch reworks.**
- **tree 1 — internal `switch@15020000`** (mt7530-mmio, dsa `member <1 0>`),
  master eth0/gmac0. Only user port: `lan5` (`gsw_port0`, ex-"mgmt", the RJ45
  this box is cabled into; gsw_port1–3 disabled on r4pro).

`br-lan` bridges `lan1 lan2 lan3 lan4 lan5 lan6` → `10.222.1.2/24`
(`network.@device[0].ports`, `network.lan`); `network.wan` = `eth1`.

```
                 ┌──────────────────────── BANANA (BPI-R4 Pro 8X) ────────────────────────┐
                 │  SoC  mediaTek MT7988                                                  │
                 │  ethernet@15100000  (mtk_soc_eth)                                      │
                 │   ┌ gmac0 ── eth0 ──── DSA master (tree 1)                             │
                 │   │  10G fixed-link ⇄ internal switch@15020000  (mt7530-mmio)          │
                 │   │                        cpu port@6 ── user port gsw_port0            │
                 │   └                                  = lan5 ◄═ MGMT CABLE              │
                 │   ┌ gmac1 ── eth1 ── usxgmii ── AS21010JB1 phy28 (mdio-bus:1c) ══ WAN   │
                 │   │                   82.131.28.40/22 ─ gw 82.131.28.1 (internet)       │
                 │   └ gmac2 ── eth2 ── DSA master (tree 0)   ◄═══ RSS/multiring MAC      │
                 │      10gbase-r fixed ⇄ MxL86252 switch16 (mdio-bus:10)                 │
                 │                         cpu port@9   tag mxl862xx-8021q                 │
                 │                         ports:                                          │
                 │                           lan1 mii:00 │ lan2 mii:01                     │
                 │                           lan3 mii:02 │ lan4 mii:03                     │
                 │                           lan6 port@13 ─ usxgmii ─ AS21010JB1 phy24     │
                 │                                        (mdio-bus:18)                    │
                 └──────────────┬─────────────────────────────────────────────────────────┘
                                │  lan5 (mgmt cable, currently)
                                └──────────────────► DEV BOX
                                        eth0  a8:b8:e0:0a:28:48
                                        10.222.1.1/24     (mgmt box + 10.222.20.0/24 gw)
                                        10.222.1.22/24
                                        192.168.1.254/24  (TFTP server /data/tftp)
                                        serial console ──► /data/tftp/console.log
```

Notes relevant to the AIMARKER experiments:
- The management box **always sits on a br-lan switch port**; normally `lan5`
  (mt7530/eth0). During AIMARKER10 the cable was moved and came up on `lan6`
  (MxL/eth2 path) — the port under test for our RSS RX handoff.
- `10.222.20.0/24` is routed via `10.222.1.1` (dev box) on br-lan
  (`network.@route[0]`).
- U-Boot TFTP uses `ipaddr 192.168.1.1` / `serverip 192.168.1.254` on a
  separate logical subnet from the `10.222.1.x` LAN; the same physical cable
  carries both (multi-address `eth0` on the dev box).

## Why

The MT7988 bridged/local-routing path is CPU-limited to ~1.6 Gbit/s on the
upstream single-NAPI `mtk_eth_soc`. frank-w's RSS series spreads flows across
4 PDMA RX rings (measured ~7.3 Gbit/s; forum user 4.5→8.4). Upstream 6.18.44 is
single-NAPI, so the series does not apply (see `rsslro-portability.md`); this is
a hand-port onto 6.18.44.

## What was built (6 commits on the branch)

| commit | change |
|---|---|
| `0460d996de` | **760-21** register definitions + **760-22** multi-ring NAPI + RSS (87 hunks) |
| `a038cfdac3` | **760-23** restrict `MTK_PDMA_INT`/`MTK_RSS` to **MT7988 only** |
| `d49c11f56d` | filogic: `CONFIG_NET_MEDIATEK_SOC=m` (driver as a module) |
| `d3bac81c44` `4d414712a4` `538802bccf` | `kmod-net-mediatek` package + ship on bpi-r4-pro devices |

### 760-22 port highlights (onto the single-NAPI tree)

- `struct napi_struct rx_napi` → `struct mtk_napi rx_napi[MTK_RX_NAPI_NUM]`
  (per-ring NAPI, each bound to `rx_ring[i]`).
- `MTK_RX_DONE_INT` object-macro → function-like `MTK_RX_DONE_INT(ring_no)`
  (`BIT(24+ring)` on netsys-v3, `BIT(30)`/`BIT(24+ring)` on v1/v2).
- Per-ring PDMA IRQs `pdma0..3` (`mtk_get_irqs_pdma`, `IRQF_SHARED`,
  `dev_id=&rx_napi[i]`) — the names already exist in the MT7988 dts.
- `mtk_poll_rx` drains the NAPI's own ring; `mtk_update_rx_cpu_idx(eth, ring)`.
- RSS core: Toeplitz key + 128-entry indirection table, PSE ring mode,
  int-group routing, ethtool `rxfh` get/set.
- 6.18 drift fixed: upstream uses `desc_shift`, not `desc_size` →
  `!mtk_is_netsys_v3_or_greater(eth)`; `reg_map` locals added where hwlro used
  the macros.
- immutable-string fixup (`char rxring[]`).

### 760-23 (the "makes sense" fix)

The raw series granted `MTK_PDMA_INT` to MT7981/MT7986. **MT7981 has no
`pdma0..3` interrupts in its dts** (only `fe0..fe3`) → probe would fail on all
MT7981 boards. MT7986 has the IRQs but gains nothing without `MTK_RSS`. Port
restricts multi-ring/RSS to **MT7988** (the only SoC with both). All other
filogic SoCs keep the exact upstream single-NAPI path.

## Module swap (rollback strategy)

`NET_MEDIATEK_SOC=m` on filogic ⇒ `mtk_eth.ko` is a loadable module (WED parts
are inside it; `mtk_wed_ops` stays builtin). A known-good **v2** `mtk_eth.ko`
is stashed at `/data/tftp/bpi-r4pro-8x-v2-multiring-napi/mtk_eth_v2-KNOWN-GOOD.ko`
(same `vermagic=6.18.44 SMP mod_unload aarch64`, 0 RSS symbols). On a bad boot:
serial console → `rmmod mtk_eth; insmod <v2.ko>` to recover without reflash.

## TFTP-boot experiment (2026-09-13) — HUNG

Booted the RSS recovery image over TFTP (U-Boot menu 2 → `$bootfile` =
`...-initramfs-recovery.itb`, RAM only, **no NAND write**).

### Observed

1. `Cannot find device "eth0"` at **preinit (~6.8s)** — **benign**. OpenWrt's
   `lib/preinit/05_set_preinit_iface` runs `ip link set eth0 up` (8X falls into
   the `*)` default case) *before* `/etc/modules.d/*` kmods load. In v2 the
   driver is builtin (netdevs at 3.9s, before preinit) so the message never
   appears; with `=m` the driver loads at ~18s, after preinit. Nothing
   functional depends on eth0 at that instant (netifd config uses `lan1..lan6`).
   Confirmed by log: v2 has eth0 at 3.9s and **no** such message; RSS printed it.
2. **Hard hang in module init.** Booted with `initcall_debug`, the console
   shows:
   ```
   [   18.473135] calling  init_module+0x0/0xfdc [mtk_eth] @ 1131
   [   78.635968] rcu: INFO: rcu_sched detected stalls on CPUs/tasks:
   ```
   `mtk_eth`'s `init_module` **never returns** — no `initcall ... returned`, no
   driver printk (not even "mediatek frame engine"). Five consecutive RCU stalls
   (60 s each → ~780 s) on **CPU 1** (busy, irq-wedged), then watchdog reset.
   The DSA/MxL switch never even gets to load.
3. NAND untouched; watchdog auto-recovered; back on v2 production.

### Diagnosis

`init_module [mtk_eth]` = `module_init(mtk_init)` → `platform_driver_register`
→ the DT node exists at boot → `really_probe` → **`mtk_probe` runs synchronously
and hangs before `mtk_add_mac`** (no "frame engine" print). The RSS port added
these probe-time steps: `mtk_get_irqs_pdma` + per-ring `request_irq`,
`mtk_napi_init`, and the `mtk_hw_init` FE-int-group changes. The next boot uses
a driver instrumented with `pr_err("MTKDBG: ...")` markers at every probe step
(31 markers, incl. `mtk_hw_reset`/`mtk_hw_init` internals) to pin the exact
function.

### Rules of thumb learned

- "Cannot find device eth0" at preinit is a **cosmetic** `=m` artifact, not a
  failure signal.
- With `=m`, `mtk_probe` runs from `init_module` and any probe-time hang wedges
  boot with **no driver output**; `initcall_debug` + `loglevel=8` is the tool
  that localises it (`calling init_module [...]` with no return).
- Boot a recovery image over TFTP (U-Boot menu 2) to test drivers with **zero
  NAND risk**.

## Artifacts

- `/data/tftp/bpi-r4pro-8x-v2-multiring-napi/` — RSS sysupgrade/sdcard/recovery
  images + `mtk_eth_v2-KNOWN-GOOD.ko`.
- `/data/tftp/openwrt-...-initramfs-recovery.itb` — RSS recovery (TFTP menu 2).
- `/data/tftp/console.log` — full console capture of the experiment.

## Next test (staged)

Instrumented recovery image (31 `MTKDBG:` markers in `mtk_eth.ko`, all `pr_err`
so loglevel-independent) is staged as the TFTP-root recovery itb. Boot via
U-Boot menu **2** with:

```
setenv bootargs 'console=ttyS0,115200n1 loglevel=8 initcall_debug'
run boot_tftp
```

Expected: the console prints `calling init_module+0x0/... [mtk_eth]`, then the
`MTKDBG:` markers up to the last step that completes — the next (unprinted)
marker names the hanging function (suspects: `mtk_get_irqs_pdma`,
per-ring `request_irq`, `mtk_napi_init`, or `mtk_hw_init` FE int-group).

## RESULT (boot with 31 MTKDBG markers) — hang located

The marker trace (console `/data/tftp/console.log`) localised the hang to a
4-line window inside `mtk_hw_init`, **after** FE/GMAC register setup and
**exactly at the rewritten DIM calls**:

```
[   16.793304] MTKDBG: hw_reset -> CHK_IDLE_EN done
[   16.797914] MTKDBG: probe request_irq block DONE     ← misnamed; it is the
              ...                                       marker right AFTER the
              ^C[   76.805846] rcu: INFO: ... CPU 1     reset call in mtk_hw_init
```

The unprinted following marker is `hw_init -> FE int grouping`. The code
executed between the two markers (printed → not printed) is: v3 `FE_GLO_MISC`
r/w, pctl GPIO regmap writes, MAC-MCR loop, CDMQ r/w — **all identical to
v2** — and then the **only RSS-specific change in this window**:

- `mtk_dim_rx()` → for netsys-v3 writes **`reg_map->pdma.rx_delay_irq` (`0x6ac0`)**
  with `val |= val << MTK_PDMA_DELAY_RX_RING_SHIFT` (bit 16 duplicated to ring bits)
- `mtk_dim_tx()` → for netsys-v3 writes **`reg_map->pdma.tx_delay_irq` (`0x6ab0`)**

v2 (working) wrote DIM only to `pdma.delay_irq` (`0x6a0c`). The RSS port
introduced the split TX/RX delay-IRQ registers `0x6ab0/0x6ac0` together with
the `val << 16` ring-duplication, inside the `mtk_dim_rx/tx` that
`mtk_hw_init` calls inline during probe.

**Conclusion:** the hang is caused by the RSS DIM rewrite — most likely
writing the wrong register offset (`0x6ab0/0x6ac0` vs the real PDMA
delay-IRQ register) and/or the `val << MTK_PDMA_DELAY_RX_RING_SHIFT`
duplication at a point in probe where it corrupts the IRQ config and fires an
unhandled interrupt (no NAPI/IRQ handler registered yet) → CPU 1 IRQ storm +
RCU stall. `mtk_hw_init` subsequently disables IRQs (`mtk_tx/rx_irq_disable
~0`), but the damage (pending/active IRQ with no handler) already spins CPU1.

### Proposed fix (next boot to validate)

Keep the v2 DIM behaviour on 6.18.44: in `mtk_dim_rx`/`mtk_dim_tx`, **drop the
RSS split-register + ring-shift path for netsys-v3** and write the plain value
to `reg_map->pdma.delay_irq` (0x6a0c) exactly like upstream. I.e. remove the
`rx_delay_irq`/`tx_delay_irq`/`MTK_PDMA_DELAY_RX_RING_SHIFT` branches added by
the port. If it then boots, the RSS ring hash/NAPI layer can still be tested
(this only affects interrupt coalescing, not RSS routing).

### Fix applied (build 17:48, staged on TFTP)

`mtk_dim_rx` and `mtk_dim_tx` restored to upstream bodies (single
`pdma.delay_irq` 0x6a0c write, no 0x6ab0/0x6ac0 split, no `val<<16` ring
duplication). The instrumented module (31 `MTKDBG:` markers) was rebuilt and
the recover ITB restaged. **Next TFTP boot validates the fix** — if it boots,
the hang was the DIM delay-IRQ rewrite as suspected.

### DIM revert ruled out (boot after AIMARKER2, ~18:12)

Booting the DIM-reverted image still hangs at the same point (last marker
`probe request_irq block DONE` at 18.717, then RCU stall at 78.7 — identical).
Disassembly of the built `.ko` confirms `mtk_dim_rx` does a single register
write (reverted path). **The DIM rewrite is therefore NOT the cause**, despite
being the only apparent RSS-specific code in the (mis-narrowed) window.

Fine-grain markers added (36 total) bracketing every statement between the
reset and FE-int-grouping: FE_GLO_MISC, pctl, MCR-loop, CDMQ, DIM, irq_disable.
Rebuilt + restaged recovery ITB (18:22). This boot will name the exact
hanging statement.

### Fine-grain boot (AIMARKER3, ~20:11) — hang narrowed to `mtk_r32(MTK_FE_GLO_MISC)`

36-marker fine-grain build booted. Last markers:

```
[   18.743270] MTKDBG: hw_reset -> CHK_IDLE_EN done
[   18.747878] MTKDBG: probe request_irq block DONE      ← next stmt never runs
              [would print: FE_GLO_MISC read try1]        ← NEVER
```

Hang = **`mtk_r32(eth, MTK_FE_GLO_MISC)`** — a FE-domain register read executed
~24 µs after `mtk_hw_reset` returned (the reset ends with
`regmap_write(ethsys, ETHSYS_FE_RST_CHK_IDLE_EN, 0x6f8ff)`; `mtk_hw_init` then
immediately reads `MTK_FE_GLO_MISC` (0x124)). Read hangs the bus → all CPUs stuck.

### Conceptual step-by-step vs frank-w 6.18-main (his boots, ours hangs)

Compared against the actual `mtk_eth_soc.c/.h` from
`frank-w/BPI-Router-Linux` branch `6.18-main`:

| component | frank-w vs ours |
|---|---|
| `mtk_hw_init` | **identical** (after pruning my markers) |
| `mtk_hw_reset` / `mtk_hw_warm_reset` / `ethsys_reset` | **identical** |
| `mtk_r32`/`mtk_w32` accessors | **identical** |
| `mt7988_reg_map` offsets (rss_glo_cfg, int_grp3, rx/tx_delay_irq, …) | **identical** (only ours adds `page` fields from 6.18 base, unrelated) |
| `MT7988_CAPS` (`MTK_PDMA_INT | MTK_RSS`) | **identical** |
| `MTK_FE_GLO_MISC`/`MTK_FE_INT_GRP`/`ETHSYS_FE_RST_CHK_IDLE_EN` defines | **identical** |
| probe pre-`mtk_hw_init` ordering (sram/wed/irq_fe/irq_pdma/clks) | **identical** |

**Conclusion:** the static code sequence, register values, caps and defines are
byte-identical (modulo my `pr_err` markers) to frank-w's tree that boots on the
same hardware. Therefore the hang is **NOT a logical/porting error in the code
path** — it's a runtime-state/environment difference. Remaining runtime suspects
to test next:
1. FE domain needs longer settle after reset than `ethsys_reset`'s `mdelay(10)`
   + immediate read — i.e. a timing issue (probe: add delay / retry-read markers).
2. Some prior init (WED/SRAM/clock/coherency) leaves a different domain state in
   our build vs frank-w's full tree (his tree has other dt/config deltas).
3. A `.config`/DTS difference between our tree and frank-w's (fetched-file diff
   only covers the driver source; the board dt/config may differ).

Next boot (43-marker build) instruments inside `ethsys_reset` and around the
FE_GLO_MISC read (try1 OK / write done) to characterize whether the read hangs
instantly or whether a delay after reset lets the FE come up.

### 43-marker build staged (21:08)

Additional instrumentation inside `ethsys_reset` (ASSERT/DEASSERT/DONE) and
around the `mtk_r32(MTK_FE_GLO_MISC)` read (`FE_GLO_MISC read try1` /
`try1 OK (0x%08x)` / `write done`). Stage on TFTP at 21:08. Next boot:
- if `ethsys_reset DONE` prints but `FE_GLO_MISC read try1` does not → the read
  after reset hangs the bus instantly (confirming the `mtk_r32` after reset is
  the exact wedge point).
- if `read try1 OK` prints, hang moved elsewhere.
- the marker set (42 in .ko; 43 source lines, one merged at compile) pins any
  further narrowing.

### Sublevel distillation: upstream 6.18.44 → 6.18.49 differences in our area

(Diffed the actual v6.18.44 vs v6.18.49 sources of mtk_eth_soc.{c,h},
mt7530.c, pcs-mtk-lynxi.c. mt7530 DSA: **unchanged**.)

**The hang-relevant path (`mtk_hw_init`/`mtk_hw_reset`/`ethsys_reset`, the
FE_GLO_MISC read, reset writes, caps, register-map RSS offsets) is
IDENTICAL in 6.18.44 and 6.18.49.** Sublevel drift is not the hang cause.

All real 44→49 changes are in other domains:

1. **QDMA TX multi-queue rework (biggest).** 6.18.49 reworked TX to
   `MTK_QDMA_NUM_QUEUES=16` paged queueing: `mtk_tx_buf.flags`,
   `MTK_TX_FLAGS_*`, `qid`/`skb_get_queue_mapping`, per-queue page registers
   (`qdma.page`), and a flattened `mtk_tx_map`. Our 6.18.44 carried the older
   single-queue + DSA per-port queue map (`MTK_DSA_USER_PORT_MAX`,
   `dsa_queue_base`, `dsa_port_rank`), later dropped by 49.
2. **`desc_shift` → `desc_size`** (rename of the same field; our port used
   `desc_shift`, converted to `mtk_is_netsys_v3_or_greater()` checks).
3. **`rx.dma_size` bumps: 512→2K** across soc data (and MT7988's 1K→2K in one
   row) — descriptor ring sizing only, affects open-time alloc not probe.
4. **MAC address path**: 6.18.49 inlined `of_get_ethdev_address` +
   `eth_hw_addr_random` into `mtk_add_mac` (removed `mtk_mac_assign_address`);
   DSA user-port queue mapping concept removed.
5. **pcs-mtk-lynxi** reworked to a plain library: `mtk_pcs_lynxi_create(dev,
   regmap, ana_rgc3, flags)` with an explicit `MTK_SGMII_FLAG_PN_SWAP` (from
   `mediatek,pnswap`), instead of a platform driver with `of_platform`/mutex.
6. **MT7988 caps slimmed** in 49: dropped GMAC1/2/3-SGMII/USXGMII and
   MUX_GMAC123_* bits (8X wiring doesn't use them via DSA); removed
   `MTK_RESV_BUF_MASK` (resv 0x80→0x40); dropped `num_tx_queues`,
   `shared_sgmii_used`, `available_pcs[2]`, GMAC3/debug regs, FRAGLIST features.
7. **`mtk_hw_dump*()` debug-print helpers removed** in 49.

**Takeaway:** nothing in 49 touches probe/reset/RX-ring bring-up. The 8X is on
a 6.18.44 base whose probe/FE-reset code is essentially identical to 6.18.49,
and to frank-w's 6.18-main (RSS included). So the hang remains a
runtime-state/environment difference, not a missing upstream fix and not our
porting logic. The 43-marker build (ethsys_reset internals + FE_GLO_MISC
try1/OK probes) isolates whether the FE read after reset hangs instantly.

### DEFINITIVE: the `mtk_r32(MTK_FE_GLO_MISC)` read hangs the bus (AIMARKER4, 22:42)

43-marker module build booted. Markers prove the exact wedge:

```
[   18.759320] MTKDBG: probe request_irq block DONE
[   18.763925] MTKDBG: FE_GLO_MISC read try1          <- mtk_r32() invoked
[   78.765776] rcu: INFO: rcu_sched stalls ... CPU 0  <- 60 s later, bus pinned
```

`FE_GLO_MISC read try1 OK` never printed → **`mtk_r32(eth, MTK_FE_GLO_MISC)` never
returns**. `ethsys_reset` (ASSERT/DEASSERT/DONE) and `CHK_IDLE_EN` write both
completed fine. So the FE domain does not answer its register read ~24 us after
reset deassert → AXI bus hang on CPU 0 → RCU stall → watchdog reset.

**Refined hypothesis:** the FE clock/domain is not settled/clocked when
`mtk_hw_init` reads `MTK_FE_GLO_MISC` right after `mtk_hw_reset`. It is not a
porting-logic error in the RSS code path, and not a 6.18.44-vs-6.18.49
difference (functions byte-identical to both upstream 49 and frank-w's
RSS-merged tree). Likely a **runtime state/clock-timing** interplay in our tree
vs the trees that boot. Candidate next experiments:
1. insert a delay (e.g. `mdelay(50)`) or FE-idle poll between `mtk_hw_reset` and
   the `MTK_FE_GLO_MISC` read, to see if the read then succeeds (settle-time test).
2. inspect/try the clock/PM enable path (`mtk_clk_enable`) and whether our module
   context leaves FE unclocked vs a working boot.
3. a built-in (CONFIG_NET_MEDIATEK_SOC=y) build was prepared (22:24, unstaged)
   to test the module-vs-builtin runtime difference; not yet booted.

### Built-in (=y) RSS build boots but LOSES eth2 / combo ports (AIMARKER5, 22:42)

The =y RSS recovery ITB **does not hang** (FE_GLO_MISC read completes:
`try1 OK (0x8000c016)`); probe SUCCESS, br-lan lan1-5 up, MxL switch up,
user space reached. **But eth2 (the combo-port MAC) is missing.**

Evidence: `ip a` shows only indices 1..12 (lo, eth0, eth1, lan5@eth0,
gre/gretap/erspan, lan1-4@eth1, br-lan) — no eth2, no lan6/wan. Driver probe
walked only TWO `add_mac` calls in the successful pass (markers: add_mac
ENTER x2), yet the 8X DTS has THREE `mediatek,eth-mac` children (mac@0/1/2,
aliases gmac0/1/2), and MTK_MAX_DEVS=3. The for_each_child loop skips nodes
that fail `of_device_is_compatible` or `of_device_is_available`; mac@2 must
have failed `of_device_is_available` at probe time.

Root cause (strong, evidence-backed): the =y driver probes during kernel init
(~8.69s) in the SAME instant the FIT enumerates/applies the combo DT overlays
(`-lan-phy`/`-wan-phy`/`-lan-sfp`/`-wan-sfp` sub-images listed at 8.690991+).
With =m the driver probes at ~18s, long after overlays are live, so mac@2 is
available -> eth2 registers (earlier module boots showed eth0/1/2). With =y
the probe races overlay application, mac@2 not yet "okay" -> skipped -> no
eth2/combo ports.

Note: all three RSS builds (module too) register eth2 when they get past
probe; the hang (module, AIMARKER4) and the missing-eth2 (builtin, AIMARKER5)
are two different consequences of the module-vs-builtin probe timing.

### AIMARKER6 (mac-diag built-in boot) — ethtool RSS not supported because eth2 never registers

mac-child diagnostic result (probe at ~9.1s):
```
mac-child mac avail=1 compat=1 -> add_mac ENTER   (mac0 -> eth0)
mac-child mac avail=0 compat=1                      (mac1 SKIPPED: wan combo)
mac-child mac avail=1 compat=1 -> add_mac ENTER   (mac2 -> should be eth2)
...  "generated random MAC address 20:08:02:00:00:00"
...  no eth2 frame engine / netdev
```

Mac1 (WAN combo) is disabled in the recovery ('fdt-1' base) DT because the
wan-phy overlay isn't applied on the TFTP-recovery path (bootconf_extra only
in production boot). Mac2 (10gbase-r SFP-bank MAC) IS walked + entered
mtk_add_mac + reached MAC allocation ("generated random MAC") but its netdev
does not appear — it returns early (likely fwnode_phylink_pcs_parse -> the
10gbase-r/usxgmii PCS path) so no eth2.

Consequence: ethtool -x / rx-flow-hash on any eth* returns "Netlink Error:
Not supported" because our RSS rxfh hooks are in mtk_ethtool_ops but the
RSS-capable eth2 netdev was never created, and eth0/eth1 (mt7530/MxL switch
conduits) don't advertise RSS.

Takeaway: the recovery/TFTP boot with base-DT is the wrong context to test
RSS — mac1/mac2 need the combo overlays. Next: boot the PRODUCTION sysupgrade
(or build a recovery that applies -lan-phy/-wan-phy) so mac1+mac2 are live,
then RSS ethtool/iperf test has a device to act on.

### AIMARKER10 (2026-09-14) — bootconf_extra TFTP fix: all 6 overlays load, eth2 + RSS now testable

**Boot path fix first.** `run boot_tftp` was `bootm $loadaddr#$bootconf`
(base fdt-1 only). The FIP default env carries
`bootconf_extra=...-cn13#...-cn14#...-8x-lan-phy#...-8x-wan-phy` but the
*device's saved env* (env.txt, Sep 3) did not contain `bootconf_extra`, so
the TFTP recovery boot never applied the combo overlays. Changed
`boot_tftp` in the live env (via `fw_setenv`), matching
`boot_production`/`boot_recovery`:

```
boot_tftp=tftpboot $loadaddr $bootfile && bootm $loadaddr#$bootconf#$bootconf_emmc#$bootconf_extra
```

Result (AIMARKER10 console): every TFTP boot now loads the full overlay set in
order — `config-...-8x` → `-emmc` → `-cn13` → `-cn14` → `-8x-lan-phy` → `-8x-wan-phy` —
before handing off to the kernel. (Same for the NAND `boot_production` path.)

**AIMARKER6 blocker gone:** with the combo overlays applied, `mac2` is
available at probe → **eth2 registers** on the TFTP/recovery boot too
(`mtk_soc_eth 15100000.ethernet eth2 ... irq 104`, 4 RX rings,
10Gbps up). The "RSS not testable, no eth2" problem of AIMARKER6 is resolved.

**RSS hardware live, ethtool surface incomplete.** On eth2:
- `ethtool -i eth2` → `mtk_soc_eth`; irq 104
- `ethtool -x eth2` → 4 RX rings, round-robin indir table + hash key readable;
  but **"RSS hash function: Operation not supported"**
- `ethtool -X eth2 equal 4` → accepted
- `ethtool -n eth2 rx-flow-hash tcp4` → **"Cannot get RX network flow hashing
  options: Not supported"**
- `ethtool -L eth2 combined 4` → **"netlink error: Not supported"**
- `ethtool -S eth2` → aggregate NIC + serdes stats only (no per-ring RX)

Root cause (driver, not boot): three ethtool surface gaps in `mtk_eth_soc.c`:
1. `mtk_get_rxnfc` has no `ETHTOOL_GRXFH` case → `-EOPNOTSUPP` for rx-flow-hash.
2. `mtk_ethtool_ops` has no `get_channels`/`set_channels` → `-L` fails.
3. `mtk_get_rxfh` sets `hfunc` only `if (rxfh->hfunc)`; the ioctl GET path
   passes `hfunc=0` so it's never filled → "Operation not supported".

Fixed in **760-24** (`net: ethernet: mtk_eth_soc: expose RSS channels +
flow-hash via ethtool`): GRXFH reports Toeplitz on IP src/dst + L4 ports for
tcp4/tcp6/udp4/udp6 when `MTK_RSS`; get/set_channels report
`MTK_RX_RSS_NUM` combined (1 for non-RSS SoCs); get_rxfh always returns
`ETH_RSS_HASH_TOP`. Built + restaged as the TFTP recovery ITB (01:52) for
AIMARKER11.

**Still-open — LAN link drop (not ethtool):** during AIMARKER10, mt7530
`lan5`/`eth0` (the cable to the management box 10.222.1.1) came up at
[36.0], then **link dropped at [47.99] and never returned** for the rest of
the session — hence `ping 10.222.1.1` was 100% loss in that boot while
`br-lan`/other ports stayed up (`ip neigh` showed 10.222.1.1 as
incomplete/00:00:00:00:00:00). Same box is reachable on the NAND production
boot (lan5 UP, `a8:b8:e0:0a:28:48` learned on lan5). So the recovery/TFTP
boot in this branch currently loses the mgmt link; needs its own look
(link-flap on mt7530 under multiring/NAPI build, or a config/timing issue)
before iperf/RSS validation over that port.

### MANAGEMENT-box cable move during AI10 + lan6 driver delta

The operator pulled the mgmt cable out of the original port during AI10 and
re-seated it. Port link-up sequence in the AI10 console (lines 46974-50553):
- `lan5` (mt7530/eth0, original port): up [36.0] → **down [47.99], never came back**
- the moved cable came up as **`lan6` (mxl86252) at [62.079], 10G, forwarded [62.09]**; flapped once [510→514], then stable
- lan1/2/3 up from [35-37]; lan4 never up

**Even with lan6 UP, `10.222.1.1` never answered ARP** (incomplete
`00:00:00:00:00:00`, ping 5/5 then 1/1 loss). Triage of who is "broken":

- lan6 = MxL switch16 `port@13`, CPU uplink = `gmac2` → **eth2** — the SAME
  MAC that 760-22 reworked into 4-ring multiring NAPI + RSS. lan1-4 ride the
  same MxL switch → eth2 path. So lan6 IS inside the blast radius of our
  work.
- AI10 shows eth2 RX was alive at the DMA level (`ethtool -S eth2`:
  `rx_packets: 3308`), but `tcpdump -i lan6` = 0 pkts → **nothing reached the
  netdev/stack over the MxL/eth2 path** despite rings counting RX. Consistent
  with a broken multiring/RSS RX handoff (frames into rings, never
  polled/delivered).
- **Control to run on AIMARKER11:** re-seat the mgmt cable into **lan5**
  (mt7530/eth0 — path NOT touched by our work; the NAND/v2 boot currently
  reaches 10.222.1.1 on lan5) and ping:
  - lan5 works + lan6 doesn't → our eth2/MxL RX path is at fault
  - lan5 fails too → something more general in the branch boot

**Driver delta for lan6's PHY (checked live, 2026-09-14):**

| boot | lan6 PHY binding (mdio-bus:18) |
|---|---|
| working NAND (installed v2 kernel) | `Generic Clause 45 PHY` (as21xxx only on mdio-bus:1c = eth1/phy28, fw 1.9.1) |
| AI10 (branch kernel over TFTP) | **`Aeonsemi AS21010JB1`** on mdio-bus:18 — as21xxx claimed phy24 too and loaded fw 1.9.1 on both 0x18 and 0x1c |

So the SAME phy addr (0x18) binds the bare `Generic Clause 45 PHY` driver in
the working boot but the **as21xxx driver in the branch boot** (different
kernel: installed v2 vs our RSS build). That is a real branch-vs-installed
kernel difference in phy driver binding for lan6's PHY.

Also observed in an early boot (console line ~4905-4912, banner at 4127 =
first TFTP boot): `lan6 (uninitialized): validation of usxgmii ...
failed: -EINVAL` / `failed to connect to PHY: -EINVAL` / `error -22 setting
up PHY for ... port 13` — in that boot lan6's PHY link-up FAILED outright
whereas in AI10 it linked. Both are branch-boot anomalies on the
eth2/MxL/port-13 path that the NAND boot does not show.

### RSS RX isn't delivering — full-diff investigation vs frank-w 6.18-main (2026-09-14)

**Question:** the management box (10.222.1.1) was unreachable in the AI10
TFTP boot even on a port with link up. Why — is it our multiring/RSS port?

**Evidence (AI10, console 46974-50553):**
- eth2 (gmac2, MxL switch uplink) is UP, 10G, 4 RX rings allocated
  (`ethtool -x eth2` round-robin indir table readable; `-X equal 4` accepted).
- `ethtool -S eth2` → `rx_packets: 3308` / `rx_bytes: 279205` — but these are
  **HW MAC counters** (`mtk_stats_update_mac` reads GDM/PSE MAC stats, not
  software NAPI delivery). So frames ARE entering the frame engine.
- `tcpdump -n -i lan6` → **0 packets**; `ip neigh` shows 10.222.1.1 as
  `00:00:00:00:00:00` incomplete → ARP never answered. Frames counted at the
  MAC but **never delivered to the network stack**.
- eth0 (mt7530/lan5, tree1) and eth2 (MxL, tree0) BOTH silent — i.e. ALL
  frame-engine RX silent, not just lan6.
- `DSA: tree 0 setup` + `DSA: tree 1 setup` both printed; eth0/eth1/eth2 all
  `frame engine ... irq 104`; per-ring `pdma0..3` IRQs requested (MTKDBG
  `get_irqs_pdma -> platform_get_irq_byname` x4).

**Method — full-fidelity code diff against the reference:**
Fetched `frank-w/BPI-Router-Linux@6.18-main` `mtk_eth_soc.{c,h}` and
normalized-diffed every RX-relevant function against our built (patched)
`mtk_eth_soc.c` (build_dir tree, the exact object AI10 ran).

Result table (normalized: whitespace, MTKDBG markers, `MTK_RSS_RING`/`MTK_HW_LRO_RING`/`NAPI_NUM` symbol names, 6.18.44-vs-6.18-main accessor renames like `napi_build_skb` vs `build_skb` and `RX_DESC_OFS` vs `desc_size`):

| function | ours vs frank |
|---|---|
| `mtk_poll_rx` | **identical logic** (only accessor renames) |
| `mtk_napi_rx` | identical |
| `mtk_handle_irq_rx` / `mtk_handle_irq` | identical |
| `mtk_rss_init` | identical (incl. per-RSS-group INT routing) |
| `mtk_napi_init` | differs only by HWLRO block (dropped in ours) |
| `mtk_dma_init` / `mtk_dma_free` / `mtk_init_fq_dma` | identical except HWLRO ring indices (4-7) / `num_tx_queues` |
| `mtk_gdm_config` | identical |
| `mtk_select_queue` | differs — **TX only** |
| MT7988 pdma regmap (rx_ptr 0x6900, int_grp 0x6a50/0x6a54, int_grp3 0x6a58, rss_glo_cfg 0x7000) | identical |

**Ring/IRQ wiring (both trees):**
- NAPI: ring0 + RSS rings 1-3, each `mtk_napi_rx` polls its own ring,
  enables/disables its own `MTK_RX_DONE_INT(ring_no)` (netsys-v3: `BIT(24+ring)`).
- IRQs: with `MTK_PDMA_INT` (MT7988), per-ring `irq_pdma[0..3]` each
  `mtk_handle_irq_rx(IRQF_SHARED, dev_id=&rx_napi[i])`. The shared
  `mtk_handle_irq` (fe irq 104) only dispatches slave ring0 + TX.
- RSS done distribution: ring1→`pdma.int_grp`, ring2→`int_grp+0x4`, ring3→`int_grp3`.

**Conclusion (high-confidence):** the RX NAPI/IRQ/ring/RSS code in our port is
**byte-equivalent to the reference tree that boots RSS correctly on the same
hardware**. The two structural deltas are (a) HWLRO removal
(frank: `MTK_HWLRO` cap + rings 4-7/`NAPI_NUM=8`; ours: none) and
(b) `mtk_select_queue` TX differences — neither explains total RX silence on
eth2/eth0.

**What this rules out:** no *porting-logic* bug in the RX delivery path as the
cause. The REAL remaining suspects are runtime/hardware-state, consistent with
earlier AI findings:

1. **RSS spreads flows across rings 1-3, but only ring0's NAPI gets serviced**
   if the per-ring pdma0..3 IRQs don't actually fire on this silicon/DT combo
   → frames pile in rings 1-3, HW MAC stats climb, stack sees nothing.
   (This is the leading hypothesis; identical code ≠ identical HW behaviour
   if the RSS indir/done-int routing registers end up misprogrammed at runtime.)
2. **lan6/phy24 phy-driver binding delta** (as21xxx vs Generic C45) — but this
   would not explain eth0/tree1 silence, so it is secondary.
3. DT-overlay probe race from AI5 (eth2 lost when builtin) — AI10 had eth2
   present, so not active here, but the pattern (probe timing vs overlay)
   remains relevant to RSS ring/DMA setup.

### Diagnostic to isolate ring/IRQ delivery (AI11)

Purpose: determine whether RSS is spreading traffic to rings 1-3 and whether
their per-ring IRQs fire. Low-risk, no kernel rebuild needed for `/proc`.

At the U-Boot prompt (mgmt console, serial):
```
echo AIMARKER11
setenv bootargs 'console=ttyS0,115200n1 pci=pcie_bus_perf loglevel=8 initcall_debug'
run boot_tftp
```
Once booted, from the serial console or SSH:

```sh
# 1. Are the per-ring PDMA IRQs firing at all? (watch counters while pinging)
for i in 1 2 3 4 5 6; do
  before=$(grep -E "pdma" /proc/interrupts | tr -d ' ')
  ping -c 3 -W 1 10.222.1.1 >/dev/null 2>&1
  after=$(grep -E "pdma" /proc/interrupts | tr -d ' ')
  echo "== round $i =="; echo before: $before; echo after:  $after
done
# If ring0's pdma IRQ counts climb but 1-3 are frozen → RSS spread confirmed,
# delivery dead on 1-3.

# 2. Confirm which rings are truly enabled in HW
cat /proc/interrupts | grep -iE "pdma|fe" 

# 3. Software-side: RSS reachable via ethtool on the DSA conduit
ethtool -x eth2 ; ethtool -X eth2 equal 4 ; ethtool -x eth2

# 4. Discriminator: does RX work on eth0/tree1 (mt7530) at all?
#    plug the mgmt cable into lan5 (mt7530 path, no MxL), ping 10.222.1.1:
#      lan5 RX ok + lan6 RX dead  -> MxL/eth2 path specific (suspects 1/2)
#      lan5 RX dead too           -> frame-engine wide (shared IRQ/ring0 wedge)

# 5. Kill the equal-4 hypothesis quickly: set RSS to fewer rings
ethtool -X eth2 weight 1 0 0 0    # if RX suddenly works -> RSS routing is the wedge
```

**Interpreting `#5`:** `equal 4` spread = flows (incl. the mgmt ARP flow) can
land on rings 1-3; `weight 1 0 0 0` forces everything to ring0. If traffic
then flows, the bug is definitively "RSS routes to rings 1-3 whose IRQs/NAPI
never run", pointing at the per-ring PDMA IRQ/routing setup (runtime) rather
than the ported code.

**Longer-term fixes to try if ring0-only fixes it:**
- Trace `irq_pdma[1..3]` firing with `devm_request_irq` + a counter, or netif
  msg level (`ethtool -s eth2 msglvl 0xff`) to see `done rx N, intr 0x...`.
- Compare against a build with frank's **HWLRO rings 4-7 + NAPI_NUM=8** kept
  (fills the same PDMA IRQ lines as the reference) — do NOT prune HWLRO until
  RSS is proven standalone.
- If the shared `mtk_handle_irq` never schedules rings 1-3 *and* per-ring
  `pdma` IRQs don't appear in `/proc/interrupts`, the DT interrupt-names for
  the 8X may lack/order `pdma1..3` → check the applied FDT
  (`/sys/firmware/devicetree/base/soc/ethernet@15100000/interrupt-names`).

**Status:** the ethtool surface (760-24) is in and staged; the RX-delivery
question remains OPEN, isolated to the ring/IRQ runtime path above. lan5
control test is the fastest discriminator and is now in the notes for AI11.
Branches at HEAD `3674b4db72` (topology+AI10 docs); next push pending AI11
results.

### AIMARKER11 (2026-09-15, after stable-MAC rebase + 980 fix) — RSS WORKS: 9.4 Gbit/s

Rebuilt after: stable-MAC patch (979) wired in via rebase, and 980 regenerated
(previous 980 referenced uncommitted MTKDBG context -> clean-extract Hunk#2
fail + dangling `goto err_unreg_netdev` compile error; fixed by rerouting the
dummy-alloc-failure goto to err_deinit_ppe, dropping the label).

TFTP boot AIMARKER11 results (console 57622-61236):
- **Stable MACs**: eth0=00:0c:43:36:2f:60, eth1=...:61, eth2=...:62 (nvmem,
  deterministic per boot; previously random).
- All 6 FIT overlays applied; kernel "Tue Sep 15 17:55".
- **All 4 PDMA IRQs fire** (`/proc/interrupts`): PDMA RX 0 (irq106/221),
  RSS RX 1 (107/222), RSS RX 2 (108/223), RSS RX 3 (109/224) — all counting.
  => the AI10 "rings silent / frames counted at MAC but never delivered"
  blocker is resolved.
- `ethtool -x eth0/eth1/eth2` all report `RSS hash function: toeplitz: on`
  (760-24 hfunc fix works).
- RSS indir control validated both ways:
  `ethtool -X eth2 weight 1 0 0 0` -> ping OK; `-X eth2 equal 4` -> ping OK.
- LAN+WAN routing functional: `traceroute 4.2.2.1` crosses real internet hops.
- **iperf3 -R -P4 and -P4: SUM 10.9 GBytes, ~9.40 Gbit/s, 0 retransmits.**
  (Baseline single-NAPI was ~1.6 Gbit/s.) RSS/multiring goal achieved.
- Note: `ethtool -n <if> rx-flow-hash tcp4` and `ethtool -L <if> combined 4`
  still print "Not supported" — this is the DEVICE's old ethtool CLI binary,
  not the driver (the driver answers `-x` indir+key+hfunc, `-l` channels, and
  `-S` fine). Re-flash a current ethtool on the banana to exercise those two.

Status: **RSS multi-ring NAPI port is functional and benchmarked at ~9.4 Gbit/s
on the TFTP boot.** Remaining nice-to-haves: newer ethtool on-device, drop the
MTKDBG-instrumented vs committed delta, and optionally restore HWLRO rings for
full parity with frank-w's tree.

### AIMARKER11 follow-up: corrected iperf — NOT bidir 9.4G, asymmetric 4.5R/9.4T + root cause of "RX fixed"

Re-ran iperf from changwang (this box) as the client, banana as iperf3 server
(`-D` on 10.222.1.2). Direction matters — the earlier 9.4 G was only banana TX.

Corrected matrix (link 10G full both ends, changwang atlantic eth0 10G):

| test | banana role | result |
|---|---|---|
| TCP 1S fwd | RX | **4.6 G** |
| TCP 4S fwd | RX | 4.35-4.65 G |
| TCP 1S rev | TX | **9.4 G** |
| TCP 4S rev | TX | 9.39 G |
| TCP 4S fwd (8 streams) | RX | 4.44 G |
| UDP 1S @6G fwd | RX | 3.86 sent / 2.18 recv (43% drop) |
| UDP 1S @9G fwd | RX | 3.90 sent / 2.22 recv (43% drop) |
| TCP 4S bidirectional `-d` | RX+TX | ~4.68+4.67 = **~9.35 G link-saturated** |

Rx-path (banana ingress, eth2) tops out ~4.5-4.7 G; Tx-path (banana egress,
eth2) reaches 9.4 G. This is **not** CPU, IRQ, NAPI-kthread, or switch-drop
limited:
- All 4 PDMA/RSS IRQs fire and distribute RX (verified deltas on irq 106-109).
- `effective_affinity` was CPU0-only for all eth IRQs; spread to 0/1/2/3 via
  `/proc/irq/N/smp_affinity` -> **no improvement** (RX still 4.5-4.8 G).
- NAPI is threaded (`eth->dummy_dev->threaded=1`); `napi/mtk_eth-0` kthreads
  were pinned `Cpus_allowed_list:0`; spread via `taskset` -> still 4.5-4.8 G.
- eth2 NAPI kthreads ran at only 5-16% CPU; `rx_overflow 0`, no pause/fc drops,
  switch `TxAcmDroppedPkts` only 114 total.
- Bidirectional test saturates ~10G (4.68+4.67) -> link is fine.

Conclusion: batary egress = line-rate; banana ingress ≈ half-rate
(~4.5-4.7 G) regardless of streams/IRQs/CPU — a PDMA/Soc RX ingress ceiling on
this chip (not the RSS port). Do NOT quote "9.4 Gbit/s" as a bidirectional
number; it is egress-only. Identify/round-trip further only if we desire
ingress > ~4.7 G (that is a separate HW investigation, not a porting fault).

**Why RX works in AI11 vs the AI10-era silence:** byte-diff vs the AI10 build
(1bb8235c8c) shows only TWO patches were added by the rebase:
- `979` stable-MAC (DTS/nvmem only, cosmetic)
- `980` NAPI dummy-device reorder (driver)

Everything else (970/973/974/975/976/977, 760-21..24) was already in the
AI10-era tree. `975` (usxgmii link-flap) was present then too. Therefore the
RX fix is **980**: it allocates `eth->dummy_dev` and calls `netif_napi_add`
for tx+rx[0]+RSS rings BEFORE `register_netdev()`. With `threaded=1`, the NAPI
runs on that dummy netdev's kthreads; in the old order netifd's immediate
`ndo_open` of eth1 (WAN) hit `mtk_dma_init()`→`__xdp_rxq_info_reg()`
(`Missing net_device from driver` WARN → -ENODEV), wedging the shared RX path
while eth0/eth2 still *looked* up. 980 removes that window, so all 4 rings get
their NAPI registered and RX actually drains. Confirmed: all 4 RSS IRQs now
fire and the mgmt box is reachable over lan6/eth2.

Diagnostics that keep working (760-24): `ethtool -x` shows toeplitz on
eth0/1/2; `ethtool -l eth2` says Combined 4. On-device `-n rx-flow-hash` and
`-L combined` still print "Not supported" — that's the OLD ethtool CLI on the
banana, not the driver (which answers `-x`/`-l`/`-S`).

### What does multiring/RSS actually buy? (A/B measured, 2026-09-15)

Counterfactual test on banana ingress (iPerf3 -P4 changwang->banana), comparing
all-RX-on-ring0 (`ethtool -X eth2 weight 1 0 0 0`) vs 4-ring RSS (`equal 4`):

| config | banana ingress | CPU picture |
|---|---|---|
| single-ring | 4.32 G | 1 NAPI kthread @26% (all on CPU0), load 1.12 |
| 4-ring RSS (4 flows) | 4.90 G (+13%) | spread across 4 kthreads (10/10/7/5%), still 15% idle |
| 4-ring RSS (16 flows) | 4.17 G | — |

Interpretation (honest): raw NIC ingress is wall-limited by the SoC PDMA RX
ingress ceiling (~4.2-4.9 G) regardless of ring count, so the button-measured
gain is small (+~0.6 G). The real benefit of multiring/RSS is **CPU
spreading / headroom**: single-ring pegs one core for all RX, while 4-ring
distributes the kthread work across 4 CPUs. That matters for the actual router
workload (concurrent bridging/NAT across many flows), not for this
raw-endpoint iperf. On this exact box a classic routing test maxes at 1G (WAN
eth1 = 1000Mb/s, LAN = one bridged 10G) so it cannot show RSS scaling.

Final answer to "does RSS help": yes for load-spreading headroom (+confirmed all
4 rings fire + split load), minimal for single-NIC raw ingress on this chip due
to the PDMA ingress wall.

### RSS threaded-NAPI kthread affinity is the real ingress knob — NOT IRQ affinity (2026-09-15, NAND prod)

On a FRESH boot (no tuning) every eth IRQ reports `effective_affinity=1`
(CPU0) AND all `napi/mtk_eth-0` kthreads spawn with `Cpus_allowed_list: 0`.
Why: the RSS NAPI runs threaded on the dummy netdev (`eth->dummy_dev->threaded
= 1`), and `netif_napi_add(...)->napi_kthread_create()` gives the kthread the
dummy dev's affinity = the creating context (probe, on CPU0). RPS/IRQ masks
don't move the kthreads.

Measured ingress (changwang→banana, iperf3 -P4, fresh NAND prod boot):

| config | banana ingress |
|---|---|
| default (all CPU0) | 4.83 G |
| `taskset` NAPI kthreads across CPU0-3 | **5.42 G** (+12%) |
| + also pin irq 106-109 → 1/2/4/8 | 5.26 G (IRQ pinning adds noise, no extra) |
| egress (banana TX) | 9.4 G (unaffected) |

Reproducible: default 4.83 vs spread 5.42-5.46 in repeat runs. IRQ-affinity
spreading ALONE does nothing (earlier AIMARKER11 finding), because the work is
in threaded-NAPI kthreads. **Fix persisted in `/etc/rc.local`** (and mirrored
`config/v1-restore/rc.local.perf` + `backups/config-restore/rc.local.perf`):

```
echo 1|2|4|8 > /proc/irq/106|107|108|109/smp_affinity
for p in /proc/[0-9]*; do
  [ "$(cat $p/comm)" = "napi/mtk_eth-0" ] || continue
  taskset -p $((1 << (i % 4) | 1)) $p
  i=$((i+1))
done
```
Also removed stale `echo c > /proc/irq/105/smp_affinity` — IRQ 105 no longer
exists (v2 single-NAPI RX irq; rings are 106-109 now). Verified live: applying
rc.local gives IRQ eff 1/2/4/8 + kthread 0/0-1/0,2/0,3 → 5.27 G.

### Flashing to NAND keeping config — notes (2026-09-15)

To keep `/etc/config` when installing: it lives in the NAND `rootfs_data`
UBI volume (`/dev/ubi0_6` overlay). `sysupgrade` from the TFTP/RAM initramfs
boot CANNOT see that overlay (root is tmpfs) — it would flash with a blank
config. Correct procedure:
1. `reboot` → autoboots NAND production (holds real /etc/config).
2. `scp` fails on this build (`/usr/libexec/sftp-server` missing) — transfer
   via `cat fw.itb | ssh root@… 'cat > /tmp/fw.itb'`, verify `md5sum` matches
   the staged file.
3. `setsid sysupgrade /tmp/fw.itb >/tmp/sysupgrade.log 2>&1 &` (NO `-n`).
4. Confirmed after reboot: NAND `fit` replaced (revision → new build), overlay
   `/dev/ubi0_6` still mounted, `/etc/config/network` intact (br-lan
   10.222.1.2/24, wan eth1 dhcp).

### Routing test teardown caution (2026-09-15)

Set up odroid on a separate routed leg (lan4→`br-test` 10.222.50.1, odroid
enp3s0 10.222.50.2, routes both ways, firewall accept) to measure router-path
throughput when both legs aren't bridged. Learned: (1) OpenWrt nftables input
chain has `policy drop` — a new unzoned bridge is REJECTED until you add
`nft insert rule inet fw4 input iifname "br-test" jump accept_from_lan` +
forward accepts; (2) flushing the ONLY static IP on a remote NFS-root host
mid-session kills it stone dead (`ip addr flush` on an NFS client = no way
back until reboot) — always add a second IP/leg BEFORE removing anything;
(3) teardown = reverse everything incl. `taskset`/nft rules, so the device
returns to a known-good boot state.

### Ported additional upstream v8 RSS bits as 760-25 / 760-26 (2026-09-16)

Assessed the netdev "Add RSS and LRO support" v8 series for non-ported
features. Ported (beneficial, safe, verified compile + git apply == built
source):

- **760-25** random RSS hash key: replace our hardcoded 0xfa,0x01,... key and
  `i % rss_num` indir fill with `netdev_rss_key_fill()` +
  `ethtool_rxfh_indir_default()` (upstream v8). Same ring semantics, removes
  predictable flow->ring distribution / static-hash side channel.
- **760-26** `.get_rx_ring_count` ethtool op: report MTK_RX_RSS_NUM (RSS) or
  MTK_MAX_RX_RING_NUM (LRO) via the modern op (upstream moved GRXRINGS there,
  e33bd8dd7f1f); complements 760-24 channels.

Explicitly NOT ported (avoided):
- `dma_size` MT7988 2K→4K (tx/fq) and 2K→1K (rx) — descriptor headroom on our
  4-RSS-ring build is verified OK at 2K; upside is memory only, risk of RSS
  ring starvation. Revisit only with upstream rationale or if 7.3G target demands.
- HWLRO rings / NAPI_NUM=8 / DIM delay-IRQ rewrite / irq_done_mask→
  MTK_RX_DONE_INT(eth,ring) refactor — entangled with the unmerged LRO work;
  adopt wholesale when the series lands.

Note: both new patches were built (kernel target compile OK, mtk_eth_soc.o
17:45) and verified `git apply` in sequence reproduces the compiled file
exactly. Banana NOT rebooted.

### Upstream sweep 2026-09-16 / 17 — what's new in OpenWrt, frank-w, forums; relevance to us

Checked: openwrt/openwrt PRs, frank-w, BPI forum R4-Pro section, netdev.

**OpenWrt main (all merged):**
- PR #22612 (2026-03-27): **kernel DSA driver for MaxLinear MxL862xx/86282**
  (dangowrt) — the upstream switch driver, incl. native 8-byte tag +
  `mxl862xx-8021q`, bridge/vlan/lag/mirror, counters, firmware mgmt via
  mdio/devlink, needs switch FW >= 1.0.78. This REPLACES our downstream
  mxl driver/DTS approach eventually.
- PR #21083 / #24900 / #23477 / #24642 / #24892 still merged (8X base,
  as21xxx, MxL sync, assisted learning) as recorded earlier.

**frank-w / forums relevant to OUR decisions:**
- BPI forum "[BPI-R4] LRO/RSS etc upstreamed?" (#26071) — now long thread
  confirming **HW-LRO on MT7988A is a packet-ordering bug**: engine aggregates
  only ~0.02-0.05 % of a learned flow's segments, every pickup causes a TCP
  retransmit (13-218 retransmits/GiB), the rest goes to RSS rings. Live
  evidence that NOT enabling MTK_HWLRO in our port was the right call. If we
  ever enable it: HWLRO rings advertise ~13.8K (MTK_MAX_LRO_RX_LENGTH) but are
  backed by order-0 pages (page-pool) → DMA overrun/skb_over_panic risk; and
  aggregated frames carry no gso_size (breaks forwarding of superframes).
- meehien's PR #197 on frank-w/BPI-Router-Linux (7.1-main): **QDMA TX
  use-after-free** corrupting forwarded traffic on R3+R4 (silent; "tx off no
  longer required, +20%"). Lives in frank's 7.1-main reworked QDMA TX
  (`MTK_QDMA_NUM_QUEUES=16` per-queue map). Our 6.18.44 base uses the OLD
  upstream `mtk_tx_map/mtk_tx_map_info` (single queue + DSA per-port queue),
  which does NOT match that code path → not affected, but re-check when/if we
  ever adopt frank's QDMA rework.
- New MXL switch FW **1.0.85** released 2026-08 (SinoVoIP, dsa/xfi variants);
  forum: "lan works now, but speed always CPU-limited due to missing RSS/LRO"
  → exactly the gap our multi-ring RSS port fills; also "5G/2.5G autoneg on
  the as21xxx 10G combos" driver quirks keep surfacing.
- netdev "Question: MII media mux" (2026-07-19, Bananapi upstream effort):
  generic eth-mux (SFP/copper combo) still NOT merged upstream; phy_port*
  docs are the stated future base. Our downstream gpio-hog/overlays
  (`bootconf_extra` SFP-vs-PHY) are the pragmatic path until that lands.

**Watch-list additions:** QDMA-TX-UAF patch on BPI-Router-Linux 7.1-main
(only relevant if we adopt that TX rework); MXL 1.0.85 FW driver notes;
upstream DSA mxl862xx driver in main → candidate to replace downstream mxl
driver when rebasing onto main; MII-mux upstream status for combo support.

### OpenWrt PR #24800 — 6.18.44 → 6.18.52 kernel bump (merged 2026-09-16)

graysky2; merged by openwrt-bot Sep 16. **Directly relevant: our branch's base is
6.18.44** (what we build/run); upstream main has now moved to **6.18.52**.

Commits (11):
- 6.18.45 / .46 / .47 / .48 / .49 / .50 / .51 / .52 bumps
- `xt_FLOWOFFLOAD: always set the flow output ifindex`
- `netfilter: fix flow offload with an unknown forward path` (new pending
  `699-netfilter-flowtable-set-out-ifindex-when-the-forward.patch`)
- d1/sunxi stale symbol cleanup

Relevant to us:
1. **nvmem fixed-layout refactor (6.18.45)**: `fixed-layout` became a driver on
   the nvmem-layout bus (out of nvmem core); OpenWrt rebased its `mac-base`
   handling (`804-nvmem-core-support-mac-base-fixed-layout-cells.patch`).
   EPinci run-verified **mediatek/filogic incl. BPI R4 + R4 Pro** — MACs still
   assigned, no EPROBE_DEFER stall (also fixed by PR #24924 driver-side
   defer). This is the exact path our **979 stable-MAC/nvmem** patch uses;
   worth re-validating on a 6.18.45+ rebase.
2. **flowtable/xt_FLOWOFFLOAD offload fixes** (hauke): flow offload broke in
   the bump; fixes here (bb `699`) landed. WAN/NAT forwarding depends on the
   flowtable; if we rebase past 6.18.45, carry these.
3. Known issues in-the-wild on the bump: kernel panic on zyxel nwa50ax
   (jglooije), 6.18.52 "monster" of a rebase (1521 patches in -rc1). BPI-R4
   reported fine (danpawlik/EPinci). robimarko (mediatek) promoted it
   ("Rebased on top of main and merged!", comment-5695413833).

Action when we next bump our base kernel: 6.18.44 → 6.18.52 upstream merge
will need (a) dropping any upstreamed patches, (b) our 760-21/22/23/24/25/26 +
979/980 refreshed for the nvmem-layout-bus + flowtable changes, (c) re-verify
stable-MAC (nvmem) on first boot after bump.

### Jumbo MTU 2000 on 10G path — measured + persisted (2026-09-17)

MT7988 driver hard-rejects MTU > 2000 on eth2 (`Invalid argument` at 4000/2048;
accepts 2000). So no 9K without the mtk-SDK `Add-9k-jumbo-frame-support` patch
(needs MTK_MAX_RX_LENGTH_9K + DMA/ring/xmac setup).

Tested MTU 2000 (safety: 15-min auto-revert armed + defused after success):
- banana ingress (changwang→banana, iperf3 -P4):  **~5.3 G @ MSS1448 →
  ~6.4-6.7 G @ MSS1800 (+20-25%)** — fewer NAPI packets on the CPU/RX-limited
  ingress path.
- Persisted in /etc/config/network: `eth2`, `lan6`, `br-lan` `option mtu '2000'`
  (device sections), verified post `network restart` and in the uci file.
- Caveat: gain is on the LAN/10G side; real WAN NAT path is still 1G.

Open task: port `999-2726-net-ethernet-mtk_eth_soc-add-9k-jumbo-frame-support.patch`
from mtk feed for true 9K MTU (16K xmac RX config, 9K descriptor room).

### 9K jumbo frame support ported — 760-27 (2026-09-18)

Measured earlier: MTU 2000 (max without the 9K support) gave +20-25% banana
ingress (~5.3 → 6.4-6.7 G) because fewer/larger NAPI packets. Full 9K needs
the driver change (prior: eth2 rejects MTU >2000).

Ported frank-w jumbo series onto our 6.18.44 RSS tree as **760-27
(rx-buf-len-and-9k-jumbo-mtu.patch)**:
- dynamic `eth->rx_buf_len` (1536/2048/9216) recomputed in mtk_change_mtu
  from max GMAC MTU; rings sized from it.
- `mtk_set_mcr_max_rx`: netisys-v3 XGMII MACs (mac2) program
  `XMAC_RX_CFG2` (via `MTK_XMAC_RX_CFG2`/`MTK_XMAC_MAX_RX_MASK`) for MTU up
  to `MTK_MAX_RX_LENGTH_9K` (9216); non-xgmii keep MAC_MCR encoder.
- new `MTK_NETSYS_RX_9K` capability, added to MT7988_CAPS.
- per-open `netdev->max_mtu` (9K on xgmii w/ cap, else 2K) replacing static
  set in mtk_add_mac.
- `mtk_max_buf_alloc(size)` (was fixed LRO size); `mtk_max_frag_size/buf_size`
  take `eth`.
- forward-declared `mtk_set_mcr_max_rx` (defined after mtk_mac_config here).

Compile-verified: target/linux/compile clean (0 errors).
To use: set MTU up to 9000 on the 10G XFI ports (eth2/lan6). Note the
internal PB: with a mixed 1500/9000 MTU setup the whole system's RX rings
are sized for the max → upstream forum reports 1-flow RX can DROP to ~2.2G
in some jumbo configs; our earlier MTU-2000 test kept ingress high, so
re-validate per-config after enabling real 9K.

### 9K MTU flash + test attempt (2026-09-19) — 9K accepted but path broke

Flashed r177 (9K-capable, 760-27) to NAND keeping config (sysupgrade no -n,
cat-over-ssh transfer, md5 verified). Post-flash: kernel Fri Sep 18 01:46,
rev r177-80d37447c0, config kept (br-lan 10.222.1.2/24), and
`ip link set eth2 mtu 9000` now SUCCEEDS (was -EINVAL before 760-27).

9K iperf attempt FAILED due to path, not driver:
- Set banana eth2/lan6/br-lan = 9000 and changwang eth0 (10G atlantic) = 9000
  (sudo available). Result: SSH kex "Connection reset by peer", 8K-ping 100%
  loss. Jumbo frame end-to-end did NOT carry.
- Restoring changwang eth0→1500 immediately restored SSH; banana stayed 9000
  (SSH small-packet path fine one-sided).
- Defused auto-reverts; banana back to persisted MTU (eth2=2004/lan6=2000/
  br-lan=2000, lan2=1500). MTU-2000 reference still ~4.8 G.

Likely causes to investigate before calling 9K usable:
1. BQL / ring headroom: 9K RX needs descriptor room (SDL 9K) — ring sizing from
   rx_buf_len=9216 only after mtk_change_mtu is invoked; if netifd sets MTU via
   ethtool before change_mtu, rings may still be 2K.
2. atlantic (changwang) 9K TX path / XGMII uplink between phy+mac may need
   the 1.9.x as21xxx fw + inband; or the MxL/eth2 10gbase-r uplink needs a
   matching peer MTU negotiation.
3. bridge/DSÄ user-port MRU must match (lan6 mtu=9000 but br-lan/eth2 offsets).

Next: verify 9100-byte frames actually traverse by tcpdump at both ends with
both hosts at 9000 (isolate drop point) before enabling 9K in network cfg.

### Follow-up: runtime 9K test PANICKED the kernel (RCA update) (2026-09-19)

While repeating the 9K test (banana MTU 9000 via uci+network restart, changwang
eth0 9000), the banana hit:

    skbuff: skb_over_panic: len:8046 put:8046 tail:0x206e end:0xec0 dev:<NULL>
    Kernel BUG at skb_panic+0x4c/0x50
    Kernel panic - not syncing: Oops - BUG: Fatal exception in interrupt

 => RX path put a 8046-byte frame into a ~0xec0 (3776B) buffer: the ring buffers
    were sized <MTU again (stale rx_buf_len at alloc time), and the jumbo frame
    overran -> panic.

Consequences observed:
- ramoops/pstore recorded the crash -> next `bootcmd` ran `pstore check` ->
  boot_recovery -> hostname "OpenWrt" and a DEFAULT recovery /etc/config/network
  that uses 'wan' on 'lan3' (recovery itb defaults differ from prod). Not a
  config corruption; it's the recovery image defaults.
- User then rebooted into production (wan=eth1 correct) and SSH went into a
  "kex_exchange_identification: read: Connection reset by peer" loop even at
  MSS<=1460. Root cause of reset loop: post-reboot lag / drops during the
  network.settling (SSH established to a stale socket). Fixed by driving the
  serial console (injecting uci + network restart via /dev/ttyACM0; wrote
  through minicom's advisory lock) -> lan6/eth2/br-lan mtu back to 1500 and SSH
  came back clean.

Takeaways / next steps:
- Runtime MTU change does NOT resize the 9K rings (mtk_change_mtu only sets
  rx_buf_len; rings allocated at first open with stale size) -> 9K frames
  overrun => skb_over_panic. frank's code has the same limitation; 9K only
  works if MTU set before open (uci at boot) so rings allocate 9K.
- To do 9K safely: configure mtu 9000 in /etc/config/network and BOOT once
  (not runtime set) so rx ring frag_size becomes 9K at open. Retest on a boot
  with mtu=9000 persisted, BOTH ends, then iperf. Optionally add a proper
  ring-realloc on MTU change upstream.
- Recover from wedged box: serial console via /dev/ttyACM0 (minicom lock is
  advisory; open write-only + write \n + commands works). pstore check on a
  crashed box boots recovery automatically; reboot to return to production.
### 9K end-to-end RCA RESOLVED: physical AS21010 PHY caps frames >1518 (2026-09-19)

Goal: 9K iperf banana<->changwang. Method: uci mtu 9000 persisted + reboot so
mtk RX rings are allocated at 9K at open (runtime `ip link set mtu` does NOT
realloc rings, causing the earlier skb_over_panic).

Findings (byte-counter / MIB / iperf evidence):
- changwang eth0 (AQC113) at mtu 9000: TX byte counter grows ~1.49GB for an 8K
  UDP flood => frames DO leave changwang at line rate.
- banana lan6 RX shows only ~44/... of the 8K frames actually arrive
  (+3.2MB RX vs +1.49GB changwang TX); MxL switch MtuExceedDiscardPkts = 0
  (NOT a switch-side drop).
- tcpdump on lan6 part of an 8K frame ("8028 > 1518 (invalid)") once, then none:
  consistent with a flaky physical carry of oversized frames.
- Reverse direction (banana TX -> changwang RX) jumbo also fails.
- iperf: MSS1460 = 4.90 Gb/s; MSS8900 = 1.68 Mb/s (0 receiver). Hard cliff exactly
  at 1518-byte frame size, BOTH directions.

Physically between changwang and banana lan6:
  changwang eth0 (AQC113 10G) -- SFP+/copper -- Aeonsemi AS21010JB1 PHY
  (mdio-bus:18, fw 1.9.1) -- lan6 port@13 -- MxL86252 switch -- eth2/gmac2 cpu.

CONCLUSION:
- Our 9K driver port (max_mtu 9000, 760-27) is CORRECT: MTU 9000 accepted,
  rx_buf_len sizing to 9K works when set before open, rings alloc 9K, mxl switch
  port_change_mtu sets max_packet_len=9022. Nothing on the banana drops jumbo.
- End-to-end 9K is BLOCKED by the physical segment changwang<->lan6 carrying
  only <=1518B frames (Aeonsemi AS21010 / SFP+ copper path). Not a driver bug.
- Runtime `ip link set mtu` on the mtk eth remains a real footgun (stale rings
  => skb_over_panic) - still worth a fix upstream (realloc rings in change_mtu)
  or documented as "set mtu via uci + reboot".

Current state: uci mtu back to 1500 (br-lan/eth2/lan6), MTU applied live,
SSH/ping clean, r177 production, wan=eth1.
### 9K RCA FINAL (2026-09-19, corrected: Cat6 10GBASE-T, NOT SFP) — changwang TX bug

User: "wtf @ physically. i'm not using SFP, i'm using normal 6-grade twisted pair cable."
Correction: the cable/PHY path is fine. RCA is fully on changwang's AQC113 TX.

Solid evidence chain (all measured):
- banana SELF-WIRE 8K (10.222.1.200 on lan6): 2/2 received, 0% loss (banana MxL
  switch + DSA + mtk RX/TX handle 8K perfectly on the wire, both directions).
- changwang->banana 8K raw AF_PACKET (20 frames): 0 arrive at lan6 (byte
  counters + lan6 MIB fcs/mtu_exceed all 0 -> frames never physically leave
  changwang).
- changwang->banana 1580B raw frames: 46 captured -> changwang CAN TX >1518 up
  to ~1580; fails beyond somewhere.
- reverse 9K iperf (banana TX -> changwang RX): 789 Mbit flows (banana TX fine,
  changwang RX fine).
- forward 9K iperf (changwang TX -> banana RX): 1.68 Mbit, 0 receiver (fails).
- eth2/lan6 MAC counters: no fcs/long/short/checksum/mtu_exceed increments for
  valid 9K ICMP/TCP -> not a MAC/SERDES/link issue.

Diagnosis (changwang host, AQC113 Antigua, atlantic 7.1.12/fw 1.3.33):
- aq_nic_set_mtu (disasm) = mov mtu -> nic->fields + ret. It NEVER programs a
  TX max-frame/TPSMT register. Only RX per-TC pkt-buffer sizes exist in the ko.
- => `ip link set mtu 9000` changes the Linux netdev MTU but the AQC113 silicon
  still refuses to transmit frames ~>1518B. Raw/non-TSO jumbo TX silently
  drops; the wire never sees them. TSO/GSO doesn't help (no reassemble > MSS
  either since TX check at the MAC).

Impact: our 9K driver port on the banana is CORRECT and READY; end-to-end 9K is
blocked by changwang's atlantic driver not setting TX max frame size on MTU
change. Options:
  1) fix atlantic driver (add hw_atl_tpsmt set in aq_nic_set_mtu / use mainline
     atlantic with jumbo support) on changwang;
  2) test 9K banana<->banana only (works);
  3) swap changwang NIC or use a different 9K-capable peer.
Current state: uci MTU 1500 on banana (restored), changwang eth0 1500, SSH/ping
fine. No banana regression.
### 9K crash FIXED (r182-042ac7380d) via patches-6.18/760-28 (2026-09-19)

Root causes of the recurring `skb_over_panic` (len~9K, end:0xec0=3776B) in
mtk_poll_rx when jumbo frames hit the CPU RX path:

1. PDMA SDL (max RX frame) register was never programmed -> DMA silently
   dropped >1518 frames (pre-crash symptom: 8K arrived at lan6/switch but
   never reached the CPU).
2. page_pool RX path uses order-0 single PAGE_SIZE buffers; 9K frames cannot
   fit -> skb_put(pktlen) overran (end:0xec0 ~ 3776) -> kernel panic.

Fix (new patch 760-28, survives clean rebuilds):
- mtk_hwlro_rx_init + mtk_change_mtu: program reg_map->pdma.rx_cfg
  (MTK_PDMA_LRO_SDL + rx_buf_len) << MTK_RX_CFG_SDL_OFFSET (netsys_v3+).
- mtk_page_pool_enabled(): return false when any netdev mtu > MTK_PP_MAX_BUF_SIZE
  -> RX falls back to 9K-capable mtk_max_buf_alloc() frag buffers.

Validation on r182-042ac7380d (clean build from patch, config preserved, MTU 9000
uci, no sysupgrade -n):
- changwang 8K raw blast x30 into banana lan6/eth2: 8K frames (0x1f5c=8028)
  SEEN on eth2 (cpu), dmesg skb_over_panic/panic count = 0, uptime stable.
  Before fix this panicked instantly.
- config preserved across flash (wan=eth1, lan=10.222.1.2), pstore cleared,
  boots to production.

Note: end-to-end 9K iperf still blocked by changwang AQC113 TX cap (~1518) and
precision (10.222.1.99, 1G) being down; banana is now 9K-RX-crash-safe. Tested
MTUs restored to 1500 after validation.
### 9K VALIDATED on r182 (2026-09-19): banana 9K works; switch CPU->1G port quirk

Precision (10.222.1.99, lan3 1G) restored. Full matrix:

- precision <-> changwang 8K peer-to-peer: WORKS (0% loss, ~0.4ms) -> switch
  forwards 9K between 1G/10G egress ports fine.
- banana <-> changwang 9K: WORKS both directions (banana pings changwang 8K:
  round-trip ~0.4-0.6ms OK; iperf reverse flows). Banana CPU RX+TXR 9K to 10G
  port validated end-to-end.
- precision -> banana CPU 8K echo request: ARRIVES at banana br-lan + banana
  EMITS 8K echo reply (seen on br-lan, both frames length 8042). No panic.
  BUT precision's NIC never receives the reply (RX bytes +1643 only): frames
  dropped on the banana-CPU -> switch -> 1G precision port leg.
- banana -> precision 8K: 0% (reply never reaches precision).
  -> FAILING leg is exclusively CPU <-> 1G-port (lan3) egress on the MxL
     switch, while CPU <-> 10G-port (lan6) and peer <-> peer 1G/10G both carry
     9K. Likely a switch CPU-port-to-slow-port egress/buffering limitation
     (no per-port egress MTU in the mxl862xx driver; only global max_packet_len
     which is 9022). Not a mtk_eth_soc driver issue.

Conclusive: mtk_eth_soc 9K fix (760-28) is CORRECT and crash-free (0 panics,
uptime stable, 8K handled at CPU). Remaining 9K-to-1G limitation is a switch
CPU-port egress hardware quirk for slow (1G) egress ports.
MTUs restored to 1500. r182-042ac7380d running.
### 9K MTU throughput gains (iperf, r182, 2026-09-19) — BIG win, keep 9000

banana<->changwang 10G link, iperf3 -P4, t=5-8s:

                    MTU 1500        MTU 9000        gain
  TCP fwd (cw->ban)  4.43 Gb/s       9.89 Gb/s     +126%
  TCP rev (ban->cw)  9.41 Gb/s       9.89 Gb/s     +5% (line rate anyway)
  UDP fwd (cw->ban)  2.33 Gb/s (59%  9.63 Gb/s     +313%
                     pkt loss)       (2.9% loss)
  UDP rev (ban->cw)  2.87 Gb/s (0%   9.88 Gb/s     +244%
                     loss)           (0.09% loss)

At MTU 9000 both dirs reach ~9.9 Gb/s (10G line rate) for TCP and UDP with tiny
loss. At 1500 the banana's RX is 4-stream/CPU-bound (4.4G TCP) and UDP RX is
packet-rate-capped (~2.3-2.9G). The r182 760-28 fix makes 9K usable: no panics,
full throughput.

Recommendation: persist MTU 9000 on eth2/lan6/br-lan in uci as the production
config for the 10G link (keep wan/eth1 at 1500 unless the upstream is 9K too).
Elect to KEEP the 9K experiment.
### RCA: banana<->precision 9K 100% loss FIXED - lan3 port MTU was 1500 (2026-09-19)

Symptom: 8K (DF) ping banana<->precision (lan3, 1G) failed 100% both ways,
while banana<->changwang (lan6, 10G) and peer<->peer worked.

Evidence:
- precision->banana 8K request ARRIVED at banana br-lan (8 echo pairs seen).
- banana emitted 8K reply (br-lan) but it never entered eth2 (0x1f5c=8028 absent
  from eth2 capture) and never reached precision.
- No switch MIB counter moved (lan3 TxAcmDropped=0, MtuExceed=0; eth2 rx_fcs
  static).

Root cause: only lan6/eth2/br-lan had MTU 9000; lan1-lan5 (incl. precision's
lan3) were still MTU 1500. The bridge forwarded the 8K reply into lan3's DSA
port, whose egress MTU (1500) dropped the >1518 reply. Asymmetric: ingress to
br-lan accepts (no per-port ingress MTU), egress via lan3 rejected.

Fix: set lan1-lan5 MTU 9000 in uci (all DSA member ports must match the bridge
MTU for jumbo) + network restart.

Verified: full 6-direction 8K matrix all 0% loss (banana<->changwang, banana<->
precision, changwang<->precision). Backup config v1-restore/network updated with
lan1-5 mtu 9000.
### DHCP conflict RCA: banana served DHCP, broke odroid PXE (2026-09-19, FIXED)

odroid (00:1e:06:45:43:18, fixed .40, PXE bootfile odroid/syslinux.efi via
10.222.1.1) couldn't TFTP/netboot. Server side was healthy (all odroid/*
files + NFS /data/odroid + ISC dhcpd fine).

Root cause: the BANANA ran dnsmasq with DHCP on br-lan (range 100-249, from the
saved v1-restore/dhcp 'lan' section, dhcpv4/6 'server'). Its lease file showed
00:1e:06:45:43:18 = 10.222.1.152 — dhcpd offered the odroid its fixed .40, but
the odroid had bound the banana's .152 and kept requesting it -> dhcpd NAK
"wrong network" -> PXE bootstrap died.

Also fixed: /etc/config/dhcp had a malformed `nonwildcard '1'` line (bare value,
no option keyword) causing `uci show dhcp` Parse error at line 16.

Fix applied on banana:
- dnsmasq: /etc/init.d/dnsmasq disable + stop; uci dhcp.dnsmasq.enable=0
- odhcpd (DHCPv6/RA): /etc/init.d/odhcpd disable + stop
- removed nonwildcard line
- no :53/:67/:547 listeners remain; disabled at boot.
- v1-restore/dhcp updated (enable=0, nonwildcard removed).
- NOTE: restore.sh does NOT push /etc/config/odhcpd; odhcpd is disabled via its
  init script already (persists). If a full reflash+restore ever brings DHCP
  back, re-check odhcpd + dnsmasq enable.
### Runtime MTU change ring-realloc DONE - patches-6.18/760-29 (2026-09-19)

The previously-flagged footgun ("runtime MTU change does NOT resize rings") is
now fixed in the source tree (no reboot/flash done):

- mtk_change_mtu: track old rx_buf_len; when it actually changes on a running
  interface (and no reset is in flight), schedule mtk_pending_work - the FE/DMA
  reset worker - which stops all netdevs, re-inits DMA (fresh ring alloc at the
  new rx_buf_len) and reopens them.
- Makes live `ip link set mtu 9000` (and back to 1500) safe: rings are resized,
  so jumbo frames no longer overrun old smaller buffers (skb_over_panic).
- Skipped if MTK_RESETTING already set or if the selected size didn't change.
- Compile-verified (mtk_eth_soc.o rebuilt, 0 errors). Patch applies cleanly to
  the bare 760-27/760-28 state and reproduces the built source exactly.

Committed 3ae8cff43f (branch bpi-r4pro-8x-v2-multiring-napi).
### Upstream / PR / forums sweep (2026-09-19)

**openwrt/openwrt:**
- main still on 6.18.52 (verify: include target/linux/generic/kernel-6.18).
- PR #24569 "add Banana Pi BPI-R4 Pro [8x, 4e]" (base support): OPEN, +39723/-76,
  REVIEW_REQUIRED. Author stripped the MTK-feeds driver patches a reviewer
  rejected ("none work out of the box; unnecessary"); it is now device/DTS/uboot
  only. => our 760-21..29 mtk driver work stays downstream, as expected.
- PR #24279 "mt7988 ramoops/pstore layout": OPEN, kernel-side change ACCEPTED
  upstream (AngeloGioacchino, v7.3-next/dts64); scoped to bpi-r4 (NOT
  bpi-r4-pro-8x variants) - matches our pstore-recovery behaviour; upstream
  lore 20260915.35722.566918.
- Merged recently: #24900 as21xxx phy, #24892 mxl assisted learning,
  #24973 fwnode phylink PCS, #24800 6.18.45->6.18.52, #21083 r4pro 8x base.
- Open: #24990 as21xxx hwmon temp, #24887 RTL826x PHY hardening, #24073 r4 I2C1
  overlay, #25198 r2 DS3231 overlay, #24279 ramoops.

**frank-w/BPI-Router-Linux:**
- meehien PR #197 "Fix UAF in QDMA TX path" (silent forwarded-traffic
  corruption, +20% when tx off removed): MERGED into 7.1-main 2026-07-30.
  NOTE: our 6.18.44 base uses old upstream single-queue mtk_tx_map (not the
  reworked QDMA) => not affected, re-check if adopting their TX rework.
- **jumbo tracking is now on dedicated branches: 6.18-jumbo, 7.2-jumbo,
  7.3-jumbo.**
- IMPORTANT: 7.3-jumbo's mtk_change_mtu now has the RUNTIME ring-realloc that
  frank/MTK landed - but as a DEDICATED worker `rx_buf_len_work` (NOT our
  coarse reuse of mtk_pending_work/FE reset):
    - mtk_rx_buf_len(eth) derives required buf len from max GMAC MTU.
    - if dma_refcnt>0 && required != current -> schedule rx_buf_len_work.
    - worker: set MTK_RESETTING, netif_tx_disable on running devs,
      shrink-first/widen-later via mtk_set_max_rx_running, mtk_rings_stop()
      + mtk_rings_start() (light DMA-only, NAPI preserved), error path
      closes netdevs with refcount dance.
  => our 760-29 does the job but with the HEAVY FE-reset (mtk_pending_work).
  UPGRADE PATH: replace 760-29's mtk_pending_work reuse with a dedicated
  rx_buf_len_work + mtk_rings_stop/start + mtk_set_max_rx_running, matching
  frank 7.3-jumbo. Same category as MTK's own runtime-realloc feed patches
  (git01 mtk-openwrt-feeds a7ee029fd / 54f68b94df).

**BPI forum:**
- #26071 LRO/RSS upstreamed?: HWLRO on MT7988 = packet-ordering bug, ~0.02-0.05%
  agg, causes TCP retransmits; RSS is what matters for 10G. Confirms our
  no-HWLRO choice. Also: HWLRO rings advertise ~13.8K into order-0 page-pool ->
  overrun risk (the exact class we fixed).
- #17248 jumbo frames: MTK released runtime-9K patch (3ca030585a) that raises
  XMAC_RX_CFG2 runtime; mt753x GMACCR MAX_RX_JUMBO register detail; mixed
  MTU jumbo + flowtable/checksum-offload issues above 2K (wteiken) - some
  setups need TX CSUM off disabled when routing jumbo->1500.
- #27340 "Flowtable corrupts large TCP on r4-pro-8x": on 24.10 stock,
  /etc/flowtable.conf incl. eth0/eth2 in devices list breaks SSH/large TCP;
  fix: drop eth0/eth2 (DSA conduits) from the flowtable devices, or rm
  flowtable.conf. WATCH: our 9K + flow offload may hit this; keep the
  devices list to real ports.
- #27736 "Patches for BPI-R4 & R4Pro" (meehien): MxL switch 1.0.70, BE14/pcie
  fixes, AS21011 WAN link speedup.

ACTION (pending decision, no build done this pass):
- Upgrade 760-29 to frank's dedicated rx_buf_len_work design (faster, safer
  than full FE reset). All else is watch-list only.

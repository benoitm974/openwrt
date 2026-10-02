# OpenWrt for the TP-Link Archer BE800 (v1)

Community **OpenWrt** builds for the **TP-Link Archer BE800 v1** — Qualcomm
**IPQ9574**, 4× 2.5G + 2× 10G (one an RJ45/SFP combo), tri-band Wi-Fi 7
(QCN9274-family radios via ath12k), 256 MiB NAND (UBIFS).

> [!WARNING]
> **Unofficial, community-maintained firmware — not affiliated with, endorsed
> by, or supported by TP-Link or the OpenWrt project.** Provided **as-is, with
> no warranty of any kind**. Flashing third-party firmware carries real risk,
> including bricking the device and voiding your warranty. **Back up your
> router's `tp_data` partition first** — it holds your unit's unique MAC
> addresses and calibration data and cannot be recovered. If this router
> matters to you, test on a spare unit before relying on it.
>
> Several pieces of this tree are **not upstream OpenWrt** and may never merge
> as-is (details below). You are running borrowed, work-in-progress code.

## Why this fork exists

The BE800 is not in OpenWrt main yet. This tree rides on top of
[sidhantgoel/openwrt](https://github.com/sidhantgoel/openwrt) branch
`be800v1-2` ([openwrt/openwrt#20373](https://github.com/openwrt/openwrt/pull/20373))
and waits on a chain of open PRs:

- Device support: [openwrt/openwrt#20373](https://github.com/openwrt/openwrt/pull/20373)
- Wi-Fi board data (BDF): ~~#159~~ **merged upstream** (firmware_qca-wireless @ 9a202b023de2)
- First-boot network fix: [sidhantgoel/openwrt#6](https://github.com/sidhantgoel/openwrt/pull/6)

As PRs merge, this branch rebases and the local deltas shrink.

## Hardware acceleration (PPE flow offload) — borrowed from Flint 3

The headline feature of these builds is **hardware NAT/routing offload through
the IPQ9574 PPE**, driven by the kernel's netfilter flowtable
(`flow_offloading_hw`). That work is **not ours and not upstream**: it is the
PPE offload series written by **Kamil Bienkiewicz** for the GL.iNet Flint 3
(IPQ5332) — see [perceival/openwrt-flint3](https://github.com/perceival/openwrt-flint3)
(branch `flint3-be9300`, patches 0411–0446) — ported here to the IPQ9574
driver, with porting notes and findings in
[perceival/openwrt-flint3#98](https://github.com/perceival/openwrt-flint3/issues/98).
The same flowtable-driven model was pioneered upstream for MediaTek
(`mtk_ppe`) and is proposed for older Qualcomm PPEs in
[openwrt/openwrt#24806](https://github.com/openwrt/openwrt/pull/24806).

Measured on one BE800 (iperf3, 2.5G link): **IPv4 NAT 2.35 Gbit/s and routed
IPv6 2.32 Gbit/s, both directions in hardware** (CPU port sees a few hundred
packets per line-rate transfer; ~15.7% CPU with offload off). All credit for
the offload belongs to its author; this fork ported and tested it, and
carries three fixes reviewed at flint3: [#101 flow table depth per SoC](https://github.com/perceival/openwrt-flint3/pull/101),
[#102 MY_MAC ingress bitmap](https://github.com/perceival/openwrt-flint3/pull/102),
[#103 bridge-aware path checks](https://github.com/perceival/openwrt-flint3/pull/103).

## Status

| Subsystem | State |
|---|---|
| Boot / procd / SSH / LuCI | working |
| 4× 2.5G LAN + 2.5G WAN (combo RJ45) | working |
| 10G SFP in the combo cage (DAC) | working, live RJ45↔SFP switching |
| 10G RJ45 port (AQR113C) | working |
| Wi-Fi 7, all three bands (ath12k) | working; radio order stabilized by a local patch (see known issues) |
| Routed + NAT forwarding | working |
| **PPE hardware NAT/routing offload, IPv4 + IPv6** | **working both directions, both families** (see above) |
| LED matrix (front panel) | supported by `ledmatrixd` + LuCI app |
| Buttons | mapped |
| UBIFS sysupgrade | working |

## Known issues

- **Radio order used to shuffle per boot** (ath12k WSI): which UCI radio maps
  to which band could rotate across reboots — upstream issue
  [openwrt/openwrt#24949](https://github.com/openwrt/openwrt/issues/24949).
  This branch carries a local fix (`mac80211: ath12k: assign device id from
  the WSI index`) that pins the mapping; until it is upstreamed, expect the
  issue to reappear on other trees/forks.
- **Squashfs images do not boot** on this layout — **UBIFS only** (use the
  `-ubifs-` artifacts).
- **Combo port**: RJ45 and SFP cage share one port; inserting an SFP module
  takes over from the RJ45 automatically.
- Hardware offload is young silicon-driving code: if anything behaves oddly,
  try `uci set firewall.@defaults[0].flow_offloading_hw='0'` and report.

## Flashing

From **stock TP-Link firmware**: use the web UI and the
`...-ubifs-web-ui-factory.bin` image (or `factory.ubi`).

From **OpenWrt**: `sysupgrade -F` the `-ubifs-sysupgrade.bin` image
(**keep settings only from the same branch**; when coming from a very
different build, add `-n` for a clean config).

Rollback to a previous slot is possible from the serial console
(u-boot `tp_boot_idx`), serial is 115200 8N1 on the internal header.

## Building

```sh
git clone -b be800-community https://github.com/benoitm974/openwrt.git
cd openwrt
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig     # Target: Qualcomm Atheros 802.11be / ipq95xx / tplink_archer-be800-combo
make -j"$(nproc)"
```

Images land in `bin/targets/qualcommbe/ipq95xx/`.

## Credits

- **Sidhant Goel** — the BE800 device port this is built on (#20373)
- **Kamil Bienkiewicz (perceival)** — the PPE flowtable offload series (flint3)
- **JuliusBairaktaris** — flowtable-driven PPE offload reference for Qualcomm (#24806)
- OpenWrt developers for everything else

## License

Individual files retain their upstream licenses (GPL-2.0-only for kernel
patches, GPL-2.0+ / MIT / ISC / BSD as marked). PPE register data derives
from the ISC-licensed qca-ssdk headers.

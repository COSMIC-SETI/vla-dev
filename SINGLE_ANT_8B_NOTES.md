# Single-antenna 8-bit firmware — working notes

> Temporary notes for the `single-ant-8b` work. Delete (or fold into `docs/`) before merging.

## Goal

An 8-bit F-engine firmware variant that processes **one antenna (one pipeline)** instead of two,
freeing FPGA resources for DSP improvements (e.g. more PFB taps).

## Background (what we found)

- **No single-antenna 8-bit build existed.** Every 8b model has two pipelines.
  `adm_pcie_9h7_dts_dual_2x100g_dsp_single.slx` is *also* two antennas (2×100G, older PR template).
- **One pipeline = one antenna** = 1× DTS (12 × 10.24 Gb/s links, 2 pol × 2 IF) + 2× 100GbE.
  - `pipeline0`: `dts` on `qsfp_a_b_c` (**master**), `eth0` → port 3, `eth1` → port 8
  - `pipeline1`: `dts` on `firefly0` (slave), `eth0` → port 7, `eth1` → port 10
- Pipeline dataflow: `dts → two_comp → input/noise → sine_tvg → delay → lo → pfb → phase_rotate → eq →
  post_eq_tvg → reorder → packetizer0/1 → eth0/eth1`.
- The pipelines are self-contained. The only top-level coupling is `tt_out` from each pipeline,
  subtracted into the `pipeline_tt_offset` register.
- **Why the DTS/eth block count must not change:** the design builds with partial reconfiguration
  (`use_pr = on`, `pr_templates/adm_pcie_9h7_dts_dual_4x100g_pr_template-no-ip.xpr.zip`).
  The static shell expects exactly 2× `vla_dts` + 4× `onehundred_gbe`.
- **Clock:** `clk_src = dts_clk`, sourced by pipeline0's master DTS. **The single antenna must be
  connected to the `qsfp_a_b_c` DTS.**
- Shared IP (`pfb_ip` = `dsp/pfb_cplx_2p_2048c_16i_18o_core`, `uram_coarse_delay`, `uram_reorder`) is
  instantiated at top level and still used by pipeline0.
- **Output data rate:** halves per FPGA (~131 → ~65 Gb/s at full bandwidth); unchanged per antenna.
  Pipeline0's packets are identical, pipeline1's two links go quiet.

## Changes on this branch

### Firmware: `firmware/adm_pcie_9h7_dts_dual_4x100g_dsp_8b_1ant.slx`

Derived from `adm_pcie_9h7_dts_dual_4x100g_dsp_8b.slx` (v13.9.0.1, which is left untouched).

- `pipeline0`: unchanged.
- `pipeline1`: DSP removed (36 blocks: `two_comp` … `packetizer0/1`, `byte_flip*`, `autocorr`, `corr`,
  plus the From blocks that only fed them). Kept: `dts`, `sync`, `eth0`, `eth1`, the sync Gotos,
  the `time_error` Froms (eth halt), `tt_out`.
- `eth0`/`eth1` inputs 1–6 tied to zero constants, with types matching the `onehundred_gbe` asserts:
  `data` UFix_512_0, `vld` Bool, `eof` Bool, `dest_ip` UFix_32_0, `dest_port` UFix_16_0,
  `byte_enable` UFix_64_0. **pipeline1's 100G cores never transmit.**
- Unused DTS data output (`dts/2`) terminated.
- `version` mask: `fw_type` **2 → 4**. The version number stays 13.9.0.1.
- An annotation inside `pipeline1` explains the above.

Result: 6.7 MB vs 10.5 MB file; sw_regs 377 → 275; HDL gateways 957 → 687.

### Software: `software/control_sw/src/cosmic_fengine.py`

- New `FIRMWARE_TYPE_8BIT_1ANT = 4`.
- `_initialize_blocks()`: for type 4, pipeline 0 gets the full 8-bit block set. Pipeline 1 gets the
  existing minimal set (`dts` + `fpga`), whose registers still exist in the firmware.
- The REST server (`rest_serve_remoteobjects.py`, which loops `range(2)`) and the remote client need
  no change.
- Known limitation: `cold_start()` etc. on pipeline 1 with this firmware will raise `AttributeError`
  (no `sync` or other DSP blocks). Configs should only bring up pipeline 0.

## Verification done (no build yet)

Checked in MATLAB and by parsing the saved `.slx` XML against the original:

- pipeline0, `version`, `led`, `pipeline_tt_offset`: identical block inventories (pipeline0: 26,059 blocks).
- All 353 surviving yellow blocks have the same HDL gateway count as the original.
- 2× DTS + 4× 100GbE on the original ports; PR template, clock source unchanged.
- All 607 From blocks have a reachable Goto; no dangling lines or unconnected ports in pipeline1.
- All sw_regs have their HDL gateway (0 missing).

## Gotchas uncovered

1. **CASPER save-as bug.** `save_system(model, newname)` silently broke ~100 "To Processor" sw_regs:
   `swreg_init` deletes the HDL gateway chain, then swallows a `CallbackDelete` error and returns
   before redrawing. Only the sim-only path survives, so in hardware those registers would read
   nothing. Reproduced on a scratch copy with no other edits. **Fixed** by re-running
   `swreg_init(blk)` on every affected register. **Check for this after any save-as.**
2. **Stale gateway names after save-as.** Gateways are named `<model>_<path>_user_data_in/out`.
   Renamed 494 to the new model prefix (what the init would produce). The only stale names left are
   in the `gpio` library-link data, which the original model also has (names from an even older model).
3. **Tool versions.** The `.slx` was saved from **System Generator 2020.1** (Windows). This bumps
   Xilinx block `LibraryVersion` 1.2 → 1.9 and adds 2020.1-only block params. `docs/source/installation.rst`
   says the design targets **Vivado 2019.1.3 / MATLAB R2019a**. If the build server's 2019.1.3 rejects
   the model, regenerate it there from the original with a script (same steps as above).
4. **Windows checkout noise.** `git status` on Windows shows ~30 bogus changes (paths > 260 chars,
   the `startsg` symlink, submodule filenames containing `*`). Stage files by explicit path only.

## Building

Standard CASPER/jasper flow on the Linux build server:

```bash
cd vla-dev/firmware
./startsg startsg.local   # MATLAB_PATH, XILINX_PATH, PLATFORM=lin64, JASPER_BACKEND=vivado
```
```matlab
jasper('adm_pcie_9h7_dts_dual_4x100g_dsp_8b_1ant')
```
Output: `firmware/adm_pcie_9h7_dts_dual_4x100g_dsp_8b_1ant/outputs/*.fpg`. Consider copying
`firmware/adm_pcie_9h7_dts_dual_4x100g_dsp_8b/.gitignore` into the new build dir.

## Status / next steps

- [ ] Build on the server; confirm the model loads under the server's Vivado/Sysgen version.
- [ ] Review the utilization report vs the dual-antenna build (how much headroom was freed?).
- [ ] Hardware test: program, `CosmicFengine(..., pipeline_id=0)`, check `fw_type == 4`, DTS lock, 100G output.
- [ ] Use the freed resources: e.g. more PFB taps. That means editing and regenerating
      `firmware/dsp/pfb_cplx_2p_2048c_16i_18o_core.slx` (on the server), then the pfb wrapper in pipeline0.
- [ ] Decide whether to bump the version number for this variant.
- [ ] Before merging: fold the relevant parts into `docs/`, delete this file.

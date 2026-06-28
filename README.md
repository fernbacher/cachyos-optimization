# CachyOS Performance Optimization Guide

**Target hardware class:** Intel 9th-gen (i5-9400F class) + NVIDIA GTX 1650 / Turing + 16GB RAM
**Target OS:** CachyOS (Arch Linux) with Wayland desktop
**Kernel:** linux-cachyos — PREEMPT, 1000Hz, SCHED_EXT+BORE

> A practical guide for getting the most out of mid-range gaming hardware on CachyOS. Every section explains what CachyOS already provides, what you should add on top, the exact configuration files to create, and the tradeoffs involved. Copy-paste-ready configs at the end.

## Table of Contents

- 1. What CachyOS Already Does
- 2. Before You Begin
- 3. Boot Parameters
- 4. CPU Tuning
- 5. Memory & Swap
- 6. Storage & Filesystems
- 7. GPU — NVIDIA
- 8. GPU Overclocking via LACT
- 9. Gaming Pipeline
- 10. Audio — PipeWire Low Latency
- 11. Network
- 12. System Limits & Sysctl
- 13. Services to Disable
- 14. Desktop & Wayland
- 15. Minimum Viable Tuning (The 80/20)
- 16. Verification Checklist
- 17. System Recovery (If You Break It)
- 18. Configuration File Reference

## 1. What CachyOS Already Does

CachyOS ships with aggressive performance defaults. Understanding these prevents you from double‑configuring, creating conflicts, or breaking things that are already optimized.

### Kernel

The `linux-cachyos` kernel includes:

- **PREEMPT** (full preemption) — lower scheduling latency
- **1000 Hz tick rate** — finer‑grained scheduler decisions
- **SCHED_EXT + BORE** — BPF‑extensible scheduling with Burst‑Oriented Response Enhancer
- **Clang ThinLTO** — link‑time optimization during kernel compilation
- **x86‑64‑v3/v4** microarchitecture targeting
- **NTSYNC** — kernel‑level synchronization for Wine/Proton (replaces fsync/esync)

If you install CachyOS with the default kernel, you already have all of this.

### Sysctl defaults

`/usr/lib/sysctl.d/70-cachyos-settings.conf`:

```
vm.swappiness = 100
vm.vfs_cache_pressure = 50
vm.dirty_bytes = 268435456
vm.dirty_background_bytes = 67108864
vm.dirty_writeback_centisecs = 1500
vm.page-cluster = 0
kernel.nmi_watchdog = 0
kernel.split_lock_mitigate = 0
kernel.printk = 3 3 3 3
kernel.kptr_restrict = 2
net.core.netdev_max_backlog = 4096
fs.file-max = 2097152
```

Systemd additionally sets:

```
fs.inotify.max_user_instances = 1024
fs.inotify.max_user_watches = 524288
vm.max_map_count = 1048576
net.ipv4.tcp_keepalive_time = 120
```

### ZRAM

- Compression: `zstd`
- Size: equal to physical RAM
- Swap priority: 100
- A udev rule sets `vm.swappiness = 150` when ZRAM activates

### Transparent Hugepages

CachyOS enables THP aggressively:

- `enabled`: `always`
- `defrag`: `defer+madvise`
- `khugepaged/max_ptes_none`: `409`

This benefits tcmalloc‑based applications (Chromium, Proton games) while the `max_ptes_none` tuning prevents the memory bloat historically associated with `always`.

### IO Schedulers

- **HDD** → `bfq`
- **SATA SSD** → `mq-deadline`
- **NVMe SSD** → `kyber`

### NVIDIA Module Options

```
options nvidia NVreg_UsePageAttributeTable=1
    NVreg_InitializeSystemMemoryAllocations=0
    NVreg_DynamicPowerManagement=0x02
```

- **PAT=1** — faster CPU↔GPU memory transfers
- **InitMemAlloc=0** — skips zeroing system memory before GPU use (small performance gain)
- **DynamicPowerManagement=0x02** — runtime D3 for Turing mobile GPUs (irrelevant for desktops, harmless)

### Systemd Tuning

| Setting | Value |
|---|---|
| Journal max size | 50 MB |
| Service start timeout | 15 s |
| Service stop timeout | 10 s |
| DefaultLimitNOFILE (system) | 2048:2097152 |
| DefaultLimitNOFILE (user) | 1024:1048576 |

### Other Built‑in Optimizations

- **Audio:** rtkit daemon, audio group realtime priority (`rtprio 99`), CPU DMA latency device access, snd_hda_intel power saving disabled on AC power
- **SATA:** ALPM forced to `max_performance`
- **HDD:** `hdparm -B 254 -S 0` — no APM spin‑down
- **Watchdog modules:** `iTCO_wdt` and `sp5100_tco` blacklisted
- **NTP:** Cloudflare + Google time servers
- **DNS:** systemd‑resolved

**Bottom line:** You start from a strong baseline. The remaining sections are what you add on top.

## 2. Before You Begin

**Rule #1: Don't fuck up your system without a net.** CachyOS defaults to **BTRFS** for the root partition. Use it.

```
# Create a snapshot before you touch anything
sudo btrfs subvolume snapshot / /@pre-optimization

# To roll back later (if you can't boot):
# Boot from USB, mount your drive, rename snapshots.
# Or just use the CachyOS bootloader menu's "Read-Only Snapshot" option if you have it.
```

**Rule #2: Test one change at a time.** Rebooting after applying the whole list is a recipe for not knowing what broke.

## 3. Boot Parameters

CachyOS uses **Limine** as its bootloader. Parameters go in `/etc/default/limine` and get applied with `sudo limine-mkinitcpio`.

If you use a different bootloader (GRUB, systemd‑boot), the parameters are the same — only the configuration mechanism differs.

### Recommended command line

```
quiet mitigations=off nowatchdog nmi_watchdog=0 nvidia_drm.modeset=1
tsc=reliable clocksource=tsc intel_pstate=active preempt=full
split_lock_detect=off pcie_aspm=performance intel_idle.max_cstate=1
transparent_hugepage=madvise splash rw
```

### Parameter reference

| Parameter | What it does | When to skip |
|---|---|---|
| `quiet` | Suppress kernel log output at boot | Never — cosmetic only |
| `mitigations=off` | **SEVERE** — disable all CPU vulnerability mitigations | Skip if this machine handles sensitive data or runs untrusted code |
| `nowatchdog` | Disable all watchdog timers | Never — watchdog timers are unnecessary on consumer desktops |
| `nmi_watchdog=0` | Disable NMI watchdog specifically | Never — redundant with `nowatchdog` |
| `nvidia_drm.modeset=1` | DRM modesetting for NVIDIA | **Required for Wayland.** Only skip if using X11 |
| `tsc=reliable clocksource=tsc` | Force TSC as clocksource | Skip on unstable TSC hardware (rare on Intel Core) |
| `intel_pstate=active` | Intel P‑State in active mode | Skip on AMD CPUs (not applicable) |
| `preempt=full` | Full kernel preemption | CachyOS kernel already has this compiled in, but adding it explicitly is harmless |
| `split_lock_detect=off` | Disable split lock detection | Skip on server workloads (split lock detection catches bugs, not relevant for gaming) |
| `pcie_aspm=performance` | Disable PCIe Active State Power Management | Skip on laptops (increases battery drain) |
| `intel_idle.max_cstate=1` | **MODERATE** — limit CPU to C1 idle | Skip on laptops or if power consumption matters |
| `transparent_hugepage=madvise` | THP on madvise hints only | Note: CachyOS tmpfiles overrides to `always` at runtime |
| `splash` | Plymouth boot animation | Never — cosmetic only |
| `rw` | Mount root read‑write | Always required |

### Security Warning: mitigations=off

> **Risk level: SEVERE**

Disables ALL CPU vulnerability protections: Spectre v1/v2, Meltdown, MDS, L1TF, Retbleed, SRBDS, and more. Your CPU becomes vulnerable to side‑channel attacks that can leak data between processes.

**Safe if:** This is a dedicated gaming rig used by one person, running only trusted software from known sources (Steam, official repos). The attack surface is minimal on a single‑user gaming desktop.

**Do NOT use if:** The machine handles sensitive data, runs multi‑tenant workloads, executes untrusted binaries, or serves as a server accessible from the network.

**Practical take:** Benchmark your games *with* and *without* `mitigations=off`. If you see a <5% gain, it's probably not worth the risk. If you see 10‑15%, decide for yourself.

### Power/Thermal Warning: intel_idle.max_cstate=1

> **Risk level: MODERATE**

Prevents the CPU from entering deep sleep states (C2–C6). Eliminates wake‑from‑idle latency but increases idle power consumption by roughly 5–15 W depending on silicon. On a desktop with adequate cooling, the thermal impact is negligible. On a laptop, this will noticeably reduce battery life.

### Kernel Update Survival Guide

Kernel updates (via `pacman -Syu`) can reset module parameters or change scheduler availability. After an update:

```
# 1. Check your NVIDIA module options survived
cat /proc/driver/nvidia/params | grep -E "UsePageAttributeTable|InitializeSystemMemoryAllocations"

# 2. Check if ADIOS is still available (if you switched to it)
cat /sys/block/nvme0n1/queue/scheduler
# If ADIOS is gone, revert to kyber or mq-deadline

# 3. Regenerate initramfs if limine changed
sudo limine-mkinitcpio
```

### Applying

```
# Add your parameters to /etc/default/limine
sudo nano /etc/default/limine

# Example line:
# KERNEL_CMDLINE[default]+="quiet mitigations=off ... rw root=UUID=..."

# Rebuild boot entries
sudo limine-mkinitcpio

# Reboot to apply
sudo reboot
```

## 4. CPU Tuning

### Lock the governor to Performance

The `performance` governor prevents the CPU from spending time in lower P‑states during load, reducing frequency ramp‑up latency. Three mechanisms should be used together — each covers a gap in the others:

**Step 1: cpupower service**

```
sudo systemctl enable --now cpupower
```

Create `/etc/cpupower.conf` if it doesn't exist:

```
governor="performance"
```

**Step 2: CachyOS udev rule**

This ships with CachyOS at `/usr/lib/udev/rules.d/99-cachyos-settings.rules`. Verify it's present — it sets `performance` on CPU add events.

**Step 3: tmpfiles.d (race‑condition‑proof)**

These files apply before any desktop session starts, preventing power manager daemons from overriding the governor:

Create `/etc/tmpfiles.d/force-performance.conf`:

```
w /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor - - - - performance
```

Create `/etc/tmpfiles.d/force-epp-performance.conf`:

```
w /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference - - - - performance
```

**Why tmpfiles.d?** `systemd-tmpfiles-setup` runs at early boot, before udev fully settles and before any user session. If a power‑profiles‑daemon or similar tries to change the governor later, you've already won the race.

### Energy Performance Preference

On Intel CPUs with `intel_pstate=active` (set in boot parameters), the Energy Performance Preference controls how aggressively the CPU pursues higher P‑states. Locking it to `performance` means the CPU holds its maximum frequency whenever there's any load.

### Verification

```
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# → performance

cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
# → performance
```

## 5. Memory & Swap

### ZRAM compression: consider lz4 over zstd

CachyOS defaults to `zstd` compression for ZRAM. On a CPU with limited threads (6 cores, no hyperthreading), `zstd` compression can consume measurable CPU time during heavy memory pressure.

**Recommendation:** Switch to `lz4` if you have a 6‑core CPU without hyperthreading or if you notice CPU spikes during memory pressure.

Create `/etc/systemd/zram-generator.conf.d/override.conf`:

```
[zram0]
compression-algorithm = lz4
```

- **lz4:** ~3–4× faster compression, ~5× faster decompression, ~15–20% worse compression ratio
- **zstd (default):** Better compression, more CPU usage
- **Tradeoff:** lz4 uses slightly more RAM for compressed pages but saves CPU. With 16 GB RAM, the extra RAM cost is negligible.

Apply:

```
sudo systemctl restart systemd-zram-setup@zram0
```

### Swappiness

CachyOS automatically sets `vm.swappiness = 150` via a udev rule when ZRAM activates. This high value tells the kernel to prefer swapping cold anonymous pages into ZRAM (where they're compressed in RAM) rather than dropping file cache. This is appropriate for ZRAM‑backed swap.

If you want even more aggressive ZRAM usage, set it higher in sysctl:

```
vm.swappiness = 180
```

The udev rule overrides to 150 at runtime regardless — the higher value in sysctl is an expression of intent. The effective value will be 150.

### Transparent Hugepages

CachyOS already configures this optimally. Verify:

```
cat /sys/kernel/mm/transparent_hugepage/enabled
# → [always] madvise never

cat /sys/kernel/mm/transparent_hugepage/defrag
# → always defer [defer+madvise] madvise never
```

### Dirty Page Writeback — Fix desktop stuttering during large transfers

> **The problem:** Linux defaults allow up to 20% of RAM to become "dirty" (unwritten data) before throttling writes. On a 16 GB system, that's 3.2 GB. When you download or extract a large file, the kernel buffers everything in page cache, then flushes it all at once. The resulting I/O storm starves the desktop compositor, causing visible stutter and unresponsive UI.

**The fix:** Force dirty pages to be written out earlier and more gradually.

Add to your sysctl config:

```
vm.dirty_background_ratio = 3
vm.dirty_ratio = 8
```

| Parameter | Kernel default | Recommended | Effect |
|---|---|---|---|
| `vm.dirty_background_ratio` | 10 | 3 | Start writing dirty pages at 3% of RAM dirty (~500 MB on 16 GB) |
| `vm.dirty_ratio` | 20 | 8 | Throttle processes generating writes at 8% dirty (~1.3 GB) |

On systems with more RAM, you may want even lower values. The goal is to keep the "dirty burst" small enough that the SSD can flush it without starving other I/O.

## 6. Storage & Filesystems

### XFS mount options

These mount options reduce metadata writes and improve throughput:

**For your root filesystem (SSD):**

```
UUID=... / xfs defaults,lazytime,noatime,inode64,logbsize=256k,noquota 0 1
```

**For bulk HDD storage:**

```
UUID=... /mount/point xfs defaults,noatime,nofail,allocsize=64m 0 0
```

| Option | Effect | When to use |
|---|---|---|
| `lazytime` | Buffer atime/mtime/ctime in memory, flush opportunistically | Always on SSD root — massive metadata write reduction |
| `noatime` | Never update access time | Always — there is almost no reason to track atime on a desktop |
| `inode64` | Allow inodes across full 64‑bit space | Required for volumes >1 TB, harmless otherwise |
| `logbsize=256k` | Larger XFS journal buffer | SSD‑only — improves metadata throughput. Default is 32k |
| `noquota` | Disable quota accounting | Only if you don't need filesystem quotas |
| `allocsize=64m` | Pre‑allocate 64 MB extents | HDD only — reduces fragmentation for large sequential writes (media, backups, Steam downloads) |
| `nofail` | Boot continues if drive missing | Use for non‑critical external/secondary drives |

### tmpfs on /tmp

```
tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
```

Places `/tmp` in RAM — reduces SSD writes and speeds up applications that use temporary files. On a 16 GB system, the typical `/tmp` usage is negligible compared to total RAM.

### F2FS for secondary/scratch SSDs

If you have a second SSD used for downloads, extraction scratch space, or disposable data, consider formatting it as **F2FS** instead of ext4/XFS. F2FS is a flash‑optimized filesystem that excels at sequential write workloads. It's ideal for drives where data integrity isn't critical (temp downloads, game library mirroring).

```
# Format (DESTRUCTIVE — wipes the drive)
sudo mkfs.f2fs -f /dev/sdX

# Mount
UUID=... /mount/point f2fs defaults,noatime,nofail 0 0
```

### IO Scheduler — switch to ADIOS

CachyOS defaults are `mq-deadline` (SATA SSD), `kyber` (NVMe), and `bfq` (HDD). For a single‑user desktop with mixed workloads, **ADIOS** (Adaptive Disk I/O Scheduler — a CachyOS kernel feature) provides better all‑around performance.

ADIOS uses per‑CPU dispatch queues with adaptive batching. It balances throughput and latency better than mq-deadline (which can hurt latency with aggressive merges) and kyber (which targets fixed latency at the cost of throughput).

Create `/etc/udev/rules.d/60-ioschedulers.rules`:

```
# HDD — BFQ remains the right choice
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", \
    ATTR{queue/scheduler}="bfq"

# SSD — use ADIOS instead of mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", \
    ATTR{queue/scheduler}="adios"

# NVMe — use ADIOS instead of kyber
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", \
    ATTR{queue/scheduler}="adios"
```

Apply:

```
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Verify:

```
cat /sys/block/sdX/queue/scheduler
# Should show [adios] for SSDs, [bfq] for HDDs
```

### NVMe Power Management (Optional)

Some NVMe drives aggressively enter power‑saving states, causing micro‑stutters when they wake up.

```
# Disable APST (Autonomous Power State Transition)
echo 0 | sudo tee /sys/class/nvme/nvme0/device/power/control
# To make it permanent, add to a systemd service or rc.local
```

## 7. GPU — NVIDIA

### Module parameters (CachyOS provides these)

You don't need to change these — they're already set by CachyOS at `/usr/lib/modprobe.d/nvidia.conf`:

```
options nvidia NVreg_UsePageAttributeTable=1
    NVreg_InitializeSystemMemoryAllocations=0
    NVreg_DynamicPowerManagement=0x02
```

### Enable persistence daemon

Keeps the NVIDIA kernel driver loaded even with no applications using the GPU. Eliminates the ~1–3 second driver initialization delay when launching the first GPU application after boot.

```
sudo systemctl enable --now nvidia-persistenced
```

### Force Full RGB over HDMI

NVIDIA defaults to Limited Range (16–235) over HDMI, causing black crush and washed‑out colors on TVs and monitors. Force Full Range (0–255) RGB.

Create `/etc/X11/xorg.conf.d/20-nvidia-full-rgb.conf`:

```
Section "Device"
    Identifier "NVIDIA"
    Driver "nvidia"
    Option "ColorSpace" "RGB"
    Option "ColorRange" "Full"
EndSection
```

**Critical for TV users** — without this, dark scenes in games and movies will look crushed and gray.

**Note:** If you have `nvidia-settings` daemon running, it might override these settings. Kill it if you see issues:

```
sudo pkill nvidia-settings
```

### Environment variables

Set these system‑wide in `/etc/environment`:

```
__GL_THREADED_OPTIMIZATION=1
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
__GL_VRR_ALLOWED=0
__GL_SYNC_TO_VBLANK=0
__GL_MaxFramesAllowed=1
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
WINEDEBUG=-all
```

| Variable | Effect | Skip if |
|---|---|---|
| `__GL_THREADED_OPTIMIZATION=1` | Multi‑threaded OpenGL pipeline | — |
| `__GL_SHADER_DISK_CACHE=1` | Persistent compiled shader cache on disk | If disk space is extremely tight |
| `__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1` | Stop driver from purging cache periodically | — |
| `__GL_VRR_ALLOWED=0` | Disable VRR (GSync/FreeSync) | If you use a VRR display and want tear‑free |
| `__GL_SYNC_TO_VBLANK=0` | **Disable V‑Sync at driver level** | If tearing bothers you |
| `__GL_MaxFramesAllowed=1` | Pre‑rendered frames = 1 (lowest latency) | — |
| `VK_ICD_FILENAMES` | Force NVIDIA Vulkan ICD | Only if you have multiple GPUs |
| `WINEDEBUG=-all` | Suppress Wine debug output | If you're debugging Wine issues |

### V‑Sync and VRR Warning

> **Risk level: MINOR (visual)**

`__GL_SYNC_TO_VBLANK=0` and `__GL_VRR_ALLOWED=0` disable all synchronization. You will see screen tearing, especially below your display's refresh rate. This is a latency vs. visual quality tradeoff — uncapped, unsynchronized frames produce the lowest possible input latency.

Set both to `1` if you prefer tear‑free output and don't need the absolute minimum latency.

### NVIDIA Wayland Reality Check

Wayland on NVIDIA is finicky. The guide assumes `nvidia_drm.modeset=1` is enough. It's necessary, but not sufficient.

**Requirements for a smooth Wayland experience:**

- NVIDIA driver **≥ 550.xx** (check with `nvidia-smi --query-gpu=driver_version --format=csv,noheader`)
- Compositor with **explicit sync** support:

- KDE Plasma ≥ 6.1
- GNOME ≥ 46
- COSMIC (latest)
- If you get a black screen or flickering, switch to X11 (`nvidia_drm.modeset=0` or just choose X11 session at login).

## 8. GPU Overclocking via LACT

[LACT](https://github.com/ilya-zlobintsev/LACT) controls NVIDIA GPUs through the NVML API. It provides a GUI for overclocking, fan curves, and monitoring.

```
sudo systemctl enable --now lactd
```

### The Clock Lock Strategy (Not "Under‑volting")

Modern NVIDIA GPUs (Turing and later) don't support traditional undervolting in Linux. Instead, use a **clock lock**: lock the GPU clock below the stock boost target and add a positive offset to the P8 voltage point.

Why this works: The GPU runs at a higher voltage than it naturally would at that clock, eliminating power throttling while staying cooler than stock boost behavior.

**For GTX 1650 / Turing‑class GPUs:**

| Setting | Value | Notes |
|---|---|---|
| Power Limit | Max (75W for GTX 1650) | Your GPU's maximum — check with `nvidia-smi -q -d POWER` |
| GPU Min Clock | 300 MHz | Stock minimum |
| GPU Max Clock | ~100 MHz below stock boost | For GTX 1650: 1860 MHz (stock boost ~1905–1935 MHz) |
| GPU Offset (P8) | +100 MHz | Raises voltage at the locked clock |
| VRAM Offset (P8) | +500 to +900 MHz | Start at +500, test for stability, increase gradually |

**Finding your stock boost clock:**

```
nvidia-smi -q -d CLOCK | grep "Max Clocks"
```

**For other GPUs:** Lock your max clock ~50–100 MHz below the stock boost target. The offset depends on silicon quality — start at +50 MHz and test.

### Fan curve

Aim for quiet at idle‑to‑moderate loads with aggressive ramp past 70°C:

| Temp | 40°C | 45°C | 60°C | 70°C | 80°C |
|---|---|---|---|---|---|
| Fan | 40% | 50% | 55% | 65% | 80% |

Adjust based on your card's cooler. The goal is to stay under 75°C during extended loads.

### Stability testing

Run a demanding game or GPU benchmark for at least 2–3 hours. If stable, run your longest typical gaming session (8+ hours). If you see artifacts, crashes, or `nvidia-smi` reporting errors, reduce the VRAM offset by 50 MHz and retest.

## 9. Gaming Pipeline

### Gamemode

[Gamemode](https://github.com/FeralInteractive/gamemode) is Feral Interactive's performance daemon. Launch games with `gamemoderun %command%` to apply optimizations.

**Recommended config** at `/etc/gamemode.ini`:

```
[general]
renice = 20
ioprio = 0

[gpu]
apply = 1
nv_powermizer_mode = 1

[custom]
start = nvidia-smi -pm 1
end = nvidia-smi -pm 0
```

| Setting | Value | What it does |
|---|---|---|
| `renice` | 20 | Highest scheduling priority (lowest nice value) |
| `ioprio` | 0 | Realtime I/O priority |
| `nv_powermizer_mode` | 1 | Force GPU to maximum performance clock level |
| `start`/`end` scripts | Persistence mode toggle | Enables driver persistence on game launch, disables on exit |

#### Gamemode group

Create `/etc/security/limits.d/10-gamemode.conf`:

```
@gamemode - nice -10
```

Add any user who games to the gamemode group:

```
sudo usermod -aG gamemode $USER
```

This allows renice to -10 without root. Gamemode itself re‑nices further (to the equivalent of -20) via its daemon privileges.

#### Conflict: disable ananicy‑cpp

CachyOS ships with `ananicy-cpp`, an automatic process priority daemon. **Disable it if you use Gamemode.** The two fight over nice values — ananicy‑cpp periodically rescans and resets priorities, causing processes to bounce between levels. This manifests as microstuttering in games.

```
sudo systemctl disable --now ananicy-cpp
```

### Steam launch options

**Native Linux games:**

```
SDL_VIDEODRIVER=wayland gamemoderun %command%
```

**Windows games via Proton:**

```
PROTON_ENABLE_WAYLAND=1 gamemoderun %command%
```

**Forcing fullscreen to a specific display (e.g., TV on HDMI):**

```
PROTON_ENABLE_WAYLAND=1 SDL_VIDEO_FULLSCREEN_DISPLAY=HDMI-A-1 gamemoderun %command%
```

Find your display name with:

```
echo $DISPLAY
# Or for Wayland: check your compositor's display settings
```

### Texture quality overrides

These replicate NVIDIA Control Panel "High Quality" texture filtering at the application level — useful since NVIDIA Settings on Linux doesn't expose these controls per‑game.

**DXVK (DirectX 9/10/11 games via Proton):**

```
DXVK_ANISO=16 DXVK_LODBIAS=-0.5 PROTON_ENABLE_WAYLAND=1 gamemoderun %command%
```

**Native OpenGL:**

```
__GL_LOG_ANISO=16 __GL_TEXTURE_LOD_BIAS=-0.5 SDL_VIDEODRIVER=wayland gamemoderun %command%
```

| Variable | Effect |
|---|---|
| `DXVK_ANISO=16` / `__GL_LOG_ANISO=16` | 16× anisotropic filtering — sharpens textures at oblique angles |
| `DXVK_LODBIAS=-0.5` / `__GL_TEXTURE_LOD_BIAS=-0.5` | Negative LOD bias — forces higher‑resolution mipmaps |

### Gamescope — probably skip it

[Gamescope](https://github.com/ValveSoftware/gamescope) is Valve's microcompositor. It adds a compositing layer between the game and your display. On a native Wayland compositor that already supports fullscreen bypass (COSMIC, KWin, Mutter), Gamescope adds input latency without benefit. Only use it if you need its specific features (FSR upscaling, HDR tonemapping, nested sessions).

### Proton

Install the CachyOS Proton builds for additional performance patches:

```
sudo pacman -S proton-cachyos-native proton-cachyos-slr
```

These include scheduler optimizations and Wine patches beyond Valve's upstream Proton.

### MangoHud

Minimal, data‑dense overlay config at `~/.config/MangoHud/MangoHud.conf`:

```
hud_compact
hud_no_margin
text_outline
background_alpha=0.3
font_size=14
table_columns=2

fps
frametime

gpu_stats
gpu_temp
gpu_core_clock
gpu_mem_clock
vram

cpu_stats
cpu_temp
cpu_mhz

wine
```

Key design choices:

- **No frametime graph** — it consumes massive vertical space for information you can get from the frametime number
- **Two‑column layout** (`table_columns=2`) — keeps the overlay compact
- **Compact mode** — eliminates padding between rows
- **Wine metrics** — shows DXVK/VKD3D version and other Proton‑relevant info

### OptiScaler

For DLSS‑style upscaling on non‑RTX hardware (GTX 1650, 1660, etc.), [OptiScaler](https://github.com/optiscaler/OptiScaler) injects FSR/XeSS into games that only support DLSS. Install as a Wine DLL override and configure per‑game.

### OnlineFix / cracked game support

For games using OnlineFix multiplayer patches, add these Wine DLL overrides:

```
onlinefix64=n;winmm=n,b;steam_api64=n;version=n,b
```

Reference: [onlinefix-linux](https://github.com/ZzEdovec/onlinefix-linux)

## 10. Audio — PipeWire Low Latency

### Quantum tuning

Lower the PipeWire quantum (processing block size) for reduced audio latency. Create `/etc/pipewire/pipewire.conf.d/10-low-latency.conf`:

```
context.properties = {
    default.clock.rate            = 48000
    default.clock.quantum         = 128
    default.clock.min-quantum     = 128
    default.clock.max-quantum     = 512
}
```

| Setting | Value | Effect |
|---|---|---|
| `clock.rate` | 48000 Hz | Standard sample rate |
| `quantum` | 128 samples | ~2.67 ms processing cycle |
| `min-quantum` | 128 | Minimum allowed block size |
| `max-quantum` | 512 | Maximum allowed block size (~10.67 ms) |

**Latency:** 128 samples / 48000 Hz ≈ 2.67 ms per cycle. Round‑trip ≈ 2× this (~5.3 ms) plus hardware conversion.

**Tradeoff:** Lower quantum = lower latency but higher CPU usage for audio processing. 128 samples is appropriate for competitive gaming and rhythm games. For general desktop use (media playback, voice chat, casual gaming), 256 samples (~5.3 ms) is sufficient and uses less CPU.

Apply:

```
systemctl --user restart pipewire
```

Verify:

```
pw-top
# Look at the QUANT column during playback
```

### CachyOS audio stack

CachyOS provides these out of the box (you don't need to configure them):

- **rtkit** — grants realtime scheduling to PipeWire
- **Audio group realtime priority** — `@audio - rtprio 99`
- **CPU DMA latency device** — audio group can suppress CPU C‑states
- **snd_hda_intel power saving disabled on AC** — prevents audio crackling
- **HPET/RTC permissions** — audio group access to high‑precision timers

## 11. Network

### Sysctl additions

Add these to your custom sysctl config (e.g., `/etc/sysctl.d/99-latency.conf`):

```
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_fastopen = 3
net.core.somaxconn = 1024
```

| Setting | CachyOS default | Recommended | Why |
|---|---|---|---|
| `netdev_max_backlog` | 4096 | 16384 | Larger per‑CPU packet buffer — prevents drops under burst traffic (game downloads, streaming) |
| `tcp_fastopen` | 0 (off) | 3 | TCP Fast Open (client + server) — saves 1 RTT on repeat connections |
| `somaxconn` | 4096 (kernel default) | 1024 | Tuned for desktop workloads, not servers. Prevents excessive queuing |

### CachyOS defaults you shouldn't change

- **Queue discipline:** `fq_codel` — actively fights bufferbloat. Don't change this.
- **TCP congestion:** `cubic` — standard and well‑tested for Internet use.
- **TCP keepalive:** 120 seconds — detects dead connections faster than kernel default.
- **DNS:** `systemd-resolved` with DNSSEC validation.
- **NTP:** Cloudflare primary, Google fallback.

## 12. System Limits & Sysctl

### File descriptor limits

Wine and Proton open many file descriptors simultaneously. The CachyOS default (2,097,152 for system services, 1,048,576 for user sessions) is generous, but some games benefit from higher limits.

In `/etc/security/limits.conf`:

```
* hard core 0
* soft core 0
* hard nofile 524288
* soft nofile 524288
root hard nofile 524288
root soft nofile 524288
```

| Setting | Value | Why |
|---|---|---|
| Core dumps | 0 (disabled) | Prevents crash dumps from consuming disk space. Re‑enable if you're debugging crashes. |
| nofile | 524288 | Higher than CachyOS user default (1M), appropriate for Wine/Proton |

### Full recommended sysctl

Create `/etc/sysctl.d/99-latency.conf`:

```
vm.swappiness = 180
vm.dirty_background_ratio = 3
vm.dirty_ratio = 8
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_fastopen = 3
fs.file-max = 2147483647
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
```

> **Note on swappiness:** You set it to 180, but CachyOS ZRAM udev rule overrides to 150 at runtime. The high value signals intent — aggressively push cold anonymous pages to ZRAM. The effective runtime value (150) is already 2.5× the kernel default of 60.

Apply:

```
sudo sysctl --system
```

## 13. Services to Disable

These services are enabled by default on CachyOS but are unnecessary for a dedicated gaming desktop. Disabling them reduces background CPU usage, memory consumption, and boot time.

| Service | How | Why |
|---|---|---|
| `ananicy-cpp` | `sudo systemctl disable --now ananicy-cpp` | Conflicts with Gamemode — causes microstuttering |
| `power-profiles-daemon` | `sudo systemctl mask power-profiles-daemon` | Overrides manual CPU governor settings |
| `NetworkManager-wait-online` | `sudo systemctl disable NetworkManager-wait-online` | Unnecessary boot delay on desktop |
| `lvm2-monitor` | `sudo systemctl mask lvm2-monitor` | Only needed if using LVM volumes |
| `bluetooth` | `sudo systemctl disable --now bluetooth` | Disable if you don't use Bluetooth peripherals |
| `geoclue` | `sudo systemctl disable --now geoclue` | Location services — not needed on desktop |
| `modemmanager` | `sudo systemctl disable --now modemmanager` | Cellular modem management — irrelevant without a modem |
| `upower` | `sudo systemctl disable --now upower` | Battery monitoring — irrelevant on desktop without UPS |

**Disable vs mask:**

- `disable` — service won't start automatically but can be started as a dependency
- `mask` — service cannot be started at all, even as a dependency of another service

Mask only when you're certain the service will never be needed.

## 14. Desktop & Wayland

### Choosing a compositor

A Wayland‑native compositor that supports **fullscreen bypass** (direct scanout) is critical for gaming performance. When a game goes fullscreen, the compositor hands the display buffer directly to the GPU, eliminating the compositing step and its associated latency.

Good options:

- **COSMIC** (System76, Rust‑based) — actively developed, fullscreen bypass supported
- **KWin** (KDE Plasma) — mature, fullscreen bypass, extensive VRR support
- **Mutter** (GNOME) — fullscreen bypass, VRR support improving

### NVIDIA + Wayland

NVIDIA GPUs require DRM kernel modesetting for Wayland. Ensure your boot parameters include:

```
nvidia_drm.modeset=1
```

Without this, Wayland sessions will either fall back to software rendering or fail to start.

**Reality check:** If you have driver < 550.xx or a compositor without explicit sync, Wayland will be a flickery mess. Don't fight it — use X11.

### Gamescope

If your compositor already supports fullscreen bypass, you don't need Gamescope. It adds an extra compositing layer and measurable input latency. Only use it for features your compositor lacks (FSR upscaling on older GPUs, HDR tonemapping, nested Steam Deck sessions).

## 15. Minimum Viable Tuning (The 80/20)

Don't want to apply the entire guide? These 4 things give 90% of the performance gain with 10% of the effort.

| # | Change | Command / Config |
|---|---|---|
| 1 | **CPU Governor** | `sudo systemctl enable --now cpupower` + `/etc/cpupower.conf` with `governor="performance"` |
| 2 | **ZRAM → lz4** (if 6‑core) | `/etc/systemd/zram-generator.conf.d/override.conf` → `compression-algorithm = lz4` |
| 3 | **Dirty Ratio** | `vm.dirty_background_ratio = 3`, `vm.dirty_ratio = 8` in sysctl |
| 4 | **Disable ananicy‑cpp** | `sudo systemctl disable --now ananicy-cpp` |

If you do *only* these, you'll solve 80% of the stutter and latency issues. The rest is fine‑tuning.

## 16. Verification Checklist

After applying each optimization, verify it took effect:

```
# ── Boot parameters ──
cat /proc/cmdline

# ── CPU ──
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# → performance

cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
# → performance

zcat /proc/config.gz | grep CONFIG_PREEMPT
# → CONFIG_PREEMPT=y

zcat /proc/config.gz | grep CONFIG_HZ=
# → CONFIG_HZ=1000

# ── Memory ──
zramctl
# Check compression algorithm and size

sysctl vm.swappiness
# → 150 (CachyOS ZRAM default)

cat /sys/kernel/mm/transparent_hugepage/enabled
# → [always] madvise never

sysctl vm.dirty_background_ratio vm.dirty_ratio
# → 3 and 8

# ── Storage ──
cat /sys/block/sdX/queue/scheduler
# → [adios] for SSDs, [bfq] for HDDs

# ── GPU ──
nvidia-smi -q | grep Persistence
# → Persistence Mode: Enabled

nvidia-smi -q -d CLOCK | grep "Max Clocks"
# → Check your overclock values

# ── Audio ──
pw-top
# Check QUANT column during playback (should show 128)

# ── Network ──
sysctl net.ipv4.tcp_fastopen
# → 3

sysctl net.core.netdev_max_backlog
# → 16384

# ── File limits ──
ulimit -n
# → 524288

# ── Gamemode (run while a game is active) ──
gamemoded -s
# → "gamemode is active"

# ── Disabled services ──
systemctl is-active ananicy-cpp power-profiles-daemon
# → inactive
```

## 17. System Recovery (If You Break It)

If your system doesn't boot after changing kernel parameters:

1. **Boot from USB** (CachyOS installer ISO).
1. **Mount your root partition**:

```
mount /dev/sdXY /mnt
mount /dev/sdXZ /mnt/boot  # if separate
```
1. **Chroot in**:

```
arch-chroot /mnt
```
1. **Revert your changes**:

```
# Restore limine backup if you made one
cp /etc/default/limine.bak /etc/default/limine
# Or simply remove the offending parameter
sudo limine-mkinitcpio
```
1. **Reboot** (exit chroot, umount, reboot).

Alternatively, if you use BTRFS and made a snapshot (`@pre-optimization`), you can roll back from the bootloader menu (if CachyOS configured it) or rename the subvolumes from the chroot.

## 18. Configuration File Reference

Complete copy‑paste‑ready files. All paths are absolute from root unless noted.

### /etc/default/limine (kernel command line)

```
KERNEL_CMDLINE[default]+="quiet mitigations=off nowatchdog nmi_watchdog=0 nvidia_drm.modeset=1 tsc=reliable clocksource=tsc intel_pstate=active preempt=full split_lock_detect=off pcie_aspm=performance intel_idle.max_cstate=1 transparent_hugepage=madvise splash rw root=UUID=..."
```

Apply: `sudo limine-mkinitcpio`

### /etc/tmpfiles.d/force-performance.conf

```
w /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor - - - - performance
```

### /etc/tmpfiles.d/force-epp-performance.conf

```
w /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference - - - - performance
```

### /etc/sysctl.d/99-latency.conf

```
vm.swappiness = 180
vm.dirty_background_ratio = 3
vm.dirty_ratio = 8
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_fastopen = 3
fs.file-max = 2147483647
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
```

Apply: `sudo sysctl --system`

### /etc/security/limits.conf (append)

```
* hard core 0
* soft core 0
* hard nofile 524288
* soft nofile 524288
root hard nofile 524288
root soft nofile 524288
```

### /etc/security/limits.d/10-gamemode.conf

```
@gamemode - nice -10
```

### /etc/environment (append)

```
__GL_THREADED_OPTIMIZATION=1
__GL_SHADER_DISK_CACHE=1
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1
__GL_VRR_ALLOWED=0
__GL_SYNC_TO_VBLANK=0
__GL_MaxFramesAllowed=1
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
WINEDEBUG=-all
```

### /etc/systemd/zram-generator.conf.d/override.conf

```
[zram0]
compression-algorithm = lz4
```

Apply: `sudo systemctl restart systemd-zram-setup@zram0`

### /etc/udev/rules.d/60-ioschedulers.rules

```
# HDD — use BFQ
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", \
    ATTR{queue/scheduler}="bfq"

# SSD — use ADIOS
ACTION=="add|change", KERNEL=="sd[a-z]*|mmcblk[0-9]*", ATTR{queue/rotational}=="0", \
    ATTR{queue/scheduler}="adios"

# NVMe — use ADIOS
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/rotational}=="0", \
    ATTR{queue/scheduler}="adios"
```

Apply: `sudo udevadm control --reload-rules && sudo udevadm trigger`

### /etc/pipewire/pipewire.conf.d/10-low-latency.conf

```
context.properties = {
    default.clock.rate            = 48000
    default.clock.quantum         = 128
    default.clock.min-quantum     = 128
    default.clock.max-quantum     = 512
}
```

Apply: `systemctl --user restart pipewire`

### /etc/gamemode.ini

```
[general]
renice = 20
ioprio = 0

[gpu]
apply = 1
nv_powermizer_mode = 1

[custom]
start = nvidia-smi -pm 1
end = nvidia-smi -pm 0
```

### /etc/X11/xorg.conf.d/20-nvidia-full-rgb.conf

```
Section "Device"
    Identifier "NVIDIA"
    Driver "nvidia"
    Option "ColorSpace" "RGB"
    Option "ColorRange" "Full"
EndSection
```

### /etc/fstab (example — replace UUIDs)

```
UUID=<boot-uuid>  /boot  vfat  defaults,umask=0077  0 2
UUID=<root-uuid>  /      xfs   defaults,lazytime,noatime,inode64,logbsize=256k,noquota  0 1
tmpfs             /tmp   tmpfs defaults,noatime,mode=1777  0 0
UUID=<hdd-uuid>   /mount/point  xfs  defaults,noatime,nofail,allocsize=64m  0 0
UUID=<ssd-uuid>   /mount/point  f2fs  defaults,noatime,nofail  0 0
```

### ~/.config/MangoHud/MangoHud.conf

```
hud_compact
hud_no_margin
text_outline
background_alpha=0.3
font_size=14
table_columns=2

fps
frametime

gpu_stats
gpu_temp
gpu_core_clock
gpu_mem_clock
vram

cpu_stats
cpu_temp
cpu_mhz

wine
```

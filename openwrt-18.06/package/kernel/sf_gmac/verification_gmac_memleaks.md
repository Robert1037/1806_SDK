# sf_gmac Memory Leak Fixes — On-Board Verification Guide

---

## General Preparation

Before testing, build two firmware images — one with the original code (baseline) and one with the patches. You'll compare behavior between them.

```bash
# Build the patched module only (faster iteration)
cd openwrt-18.06
make package/kernel/sf_gmac/compile V=s
```

You can also load/unload the module at runtime if your board supports it:

```bash
# On the board
rmmod sgmac
insmod /path/to/sgmac.ko
```

Enable kernel memory debugging if possible — add to your kernel config:

```
CONFIG_DEBUG_KMEMLEAK=y
CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=4000
```

---

## Fix 1: PHY Disconnect on DMA Init Failure (`sgmac_open`)

### What this fix does

When `sgmac_dma_desc_rings_init()` fails inside `sgmac_open()`, the PHY connection established earlier via `of_phy_connect()` is now properly disconnected.

### Success criteria

- Under memory pressure, `ifconfig eth0 up` fails gracefully, and the PHY is released (no dangling `phydev` reference).
- Subsequent `ifconfig eth0 up` retries succeed once memory is available.
- No kernel oops or use-after-free on the retry.

### Practical test procedure

The DMA init allocates `kzalloc` and `dma_alloc_coherent`. You can force failure by exhausting memory:

```bash
# Step 1: On the board, eat up memory to trigger OOM conditions
# Check available memory first
cat /proc/meminfo | grep MemAvailable

# Method A — Fill tmpfs to consume real RAM (preferred for embedded boards)
AVAIL_KB=$(awk '/MemAvailable/{print $2}' /proc/meminfo)
FILL_KB=$((AVAIL_KB * 85 / 100))
dd if=/dev/zero of=/tmp/memfill bs=1024 count=${FILL_KB} 2>/dev/null &

# Method B — stress-ng (if available)
# stress-ng --vm 1 --vm-bytes 80% --timeout 60s &

# Method C — Kernel fault injection (requires CONFIG_FAILSLAB=y)
# echo 1 > /sys/kernel/debug/failslab/probability
# echo 100 > /sys/kernel/debug/failslab/times

# Step 2: Bring interface down then up under pressure
ifconfig eth0 down
ifconfig eth0 up
# Expected: open fails with "can not alloc" in dmesg, returns error

# Step 3: Check PHY state
cat /sys/class/net/eth0/carrier  # Should show 0 or error
ls /sys/bus/mdio_bus/devices/    # PHY should NOT show as attached

# Step 4: Release memory and retry
rm -f /tmp/memfill   # release tmpfs memory (Method A)
# kill %1             # or kill stress-ng (Method B)
# echo 0 > /sys/kernel/debug/failslab/probability  # or disable fault injection (Method C)
ifconfig eth0 up
# Expected: should succeed on second attempt

# Step 5: Verify network works
ping -c 3 <gateway_ip>
```

**With kmemleak** (if enabled):

```bash
# After the failed open, scan for leaks
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
# Baseline (unpatched): may show phy_device or mdio related objects
# Patched: should be clean
```

**Alternative — fault injection** (if `CONFIG_FAILSLAB=y` is available):

```bash
# Force kzalloc failure for dma_alloc_coherent path
echo 1 > /sys/kernel/debug/failslab/probability
echo 4096 > /sys/kernel/debug/failslab/min-order
ifconfig eth0 down
ifconfig eth0 up   # Will fail
echo 0 > /sys/kernel/debug/failslab/probability
ifconfig eth0 up   # Should succeed
```

### Regression check

```bash
# Normal open/close cycle must still work
ifconfig eth0 down && ifconfig eth0 up
ping -c 5 <gateway>          # Must pass
iperf3 -c <server> -t 10     # Throughput should be unchanged
```

---

## Fix 2: DMA Unmap in `sfax8_gmac_test_rx()`

### What this fix does

Added the missing `priv->rx_skbuff[entry] = NULL` and `dma_unmap_single()` in the test RX path, matching the production `sgmac_rx()` pattern.

### Success criteria

- No DMA mapping leaks during delay auto-calibration or HNAT test.
- `/sys/kernel/debug/dma-api/` error count stays at 0.
- No kernel warnings about "DMA-API: device has pending DMA operations" on module unload.

### Practical test procedure

This path is only active when `CONFIG_SFAX8_GMAC_DELAY_AUTOCALI` or `CONFIG_SFAX8_HNAT_TEST_TOOL` is enabled.

**Method A — Delay auto-calibration (if `CONFIG_SFAX8_GMAC_DELAY_AUTOCALI=y`):**

```bash
# The calibration runs automatically on boot if factory GMAC delay is 0xff
# Or trigger it manually via debugfs:
echo autoDelay > /sys/kernel/debug/gmac_debug

# Monitor DMA debug counters before and after:
cat /sys/kernel/debug/dma-api/num_errors       # Should stay 0
cat /sys/kernel/debug/dma-api/num_free_errors   # Should stay 0
```

**Method B — HNAT test tool (if `CONFIG_SFAX8_HNAT_TEST_TOOL=y`):**

The HNAT test tool exercises `sfax8_gmac_test_rx()` during loopback testing. Trigger the test per your HNAT test documentation.

**DMA leak detection:**

```bash
# Enable DMA debugging in kernel config:
#   CONFIG_DMA_API_DEBUG=y
#   CONFIG_DMA_API_DEBUG_SG=y

# Before the test
cat /sys/kernel/debug/dma-api/num_errors
# Run the test (autoDelay or HNAT test)
# After the test
cat /sys/kernel/debug/dma-api/num_errors
# Patched: should remain 0
# Baseline: may increment due to unmapped-but-still-in-use DMA regions

# Also check on module unload
rmmod sgmac
dmesg | grep -i "dma"
# Baseline: may show "DMA-API: device has pending DMA allocations"
# Patched: clean
```

### Regression check

```bash
# Run auto-calibration and verify the calibrated delay is correct
echo autoDelay > /sys/kernel/debug/gmac_debug
dmesg | tail -20
# Should see "get min_delay:X max_delay:Y fin_tx_delay:Z"
# and "get min_delay:X max_delay:Y fin_rx_delay:Z"
# Values should be the same as baseline (unpatched)

# Verify normal RX still works after calibration
ping -c 100 <gateway>       # 0% loss
iperf3 -c <server> -t 30    # Throughput comparable to baseline
```

---

## Fix 3: Clock Cleanup in `sgmac_remove()`

### What this fix does

Added `clk_disable_unprepare(priv->eth_tclk)` in `sgmac_remove()` when `CONFIG_SFAX8_GMAC_TCLKCHOOSE` is enabled.

### Success criteria

- After `rmmod sgmac`, the `eth_tclk` clock's enable count drops to 0.
- No clock-related warnings in dmesg.
- The clock can be re-acquired cleanly on `insmod sgmac`.

### Practical test procedure

This only applies when `CONFIG_SFAX8_GMAC_TCLKCHOOSE=y`.

```bash
# Step 1: Check clock state before unload
cat /sys/kernel/debug/clk/clk_summary | grep -i eth
# Note the enable_cnt for eth_tclk

# Step 2: Unload the module
ifconfig eth0 down
rmmod sgmac

# Step 3: Check clock state after unload
cat /sys/kernel/debug/clk/clk_summary | grep -i eth
# Patched: eth_tclk enable_cnt should be 0
# Baseline: eth_tclk enable_cnt will still be 1 (leaked)

# Step 4: Reload and verify it works
insmod /lib/modules/*/sgmac.ko
ifconfig eth0 up
ping -c 3 <gateway>

# Step 5: Repeat unload/reload 10 times to check for accumulation
for i in $(seq 1 10); do
    ifconfig eth0 down
    rmmod sgmac
    insmod /lib/modules/*/sgmac.ko
    ifconfig eth0 up
done
cat /sys/kernel/debug/clk/clk_summary | grep -i eth
# Patched: enable_cnt should be 1 (current load only)
# Baseline: enable_cnt will be 11 (accumulated leaks)
```

### Regression check

```bash
# Verify the clock is still properly enabled when module is loaded
cat /sys/kernel/debug/clk/clk_summary | grep -i eth
# eth_tclk should show enable_cnt = 1

# Verify network function
iperf3 -c <server> -t 10 -P 4   # Multi-stream throughput test
```

---

## Fix 4: Ethtool Ops Overwrite in `sgmac_probe()`

### What this fix does

`ndev->ethtool_ops = &sgmac_ethtool_ops` is now only set when a direct PHY node is present. When an ethernet switch (eswitch) is detected, the `eswitch_ethtool_ops` set earlier is preserved.

### Success criteria

- **With switch:** `ethtool eth0` returns switch-specific info (uses `eswitch_ethtool_ops`), and `ethtool -s` can configure per-port PHY settings.
- **With direct PHY:** `ethtool eth0` returns PHY-specific info (uses `sgmac_ethtool_ops`), `ethtool -s` and link settings work.

### Practical test procedure

**On a board with an ethernet switch (no `phy` node in DTS, has eswitch):**

```bash
# Step 1: Check driver info
ethtool -i eth0
# Patched: driver name should come from gsw_get_drvinfo()
#          (DRV_MODULE_NAME from eswitch)
# Baseline: driver name will be "sgmac" (wrong — from sgmac_ethtool_ops)

# Step 2: Check per-port PHY settings
# Set the ethtool port via debugfs
echo phyad 0 > /sys/kernel/debug/gmac_debug
ethtool eth0
# Patched: should show correct speed/duplex/autoneg for switch port 0
# Baseline: may fail or show wrong info because sgmac_ethtool_ops
#           calls phy_ethtool_get_link_ksettings which requires phydev

echo phyad 1 > /sys/kernel/debug/gmac_debug
ethtool eth0
# Should show port 1 status

# Step 3: Try setting speed on a port
ethtool -s eth0 speed 100 duplex full autoneg off
# Patched: should succeed (goes through gsw_set_settings)
# Baseline: may fail or crash (goes through phy_ethtool_set_link_ksettings
#           with NULL phydev)

# Step 4: Verify ethtool stats
ethtool -S eth0
# Patched: shows eswitch-specific stats (15 counters)
# Baseline: shows sgmac stats (14 counters, different names)
```

**On a board with direct PHY (has `phy` node in DTS, no eswitch):**

```bash
# Verify ethtool still works as before
ethtool eth0                    # Should show PHY link settings
ethtool -i eth0                 # driver: "sgmac"
ethtool -s eth0 autoneg on     # Should work
ethtool -S eth0                 # Should show sgmac stats
```

### Regression check

```bash
# Full functional test on both board types
ping -c 100 <gateway>                    # 0% loss
iperf3 -c <server> -t 30                 # Throughput baseline comparison
iperf3 -c <server> -t 30 -R             # Reverse direction

# Link change test
ethtool -s eth0 speed 100 duplex full autoneg off
sleep 2
ping -c 5 <gateway>                      # Must still work at 100M
ethtool -s eth0 autoneg on
sleep 5
ping -c 5 <gateway>                      # Must still work after re-autoneg
```

---

## Overall Regression Test Suite

After verifying each fix individually, run this comprehensive suite to confirm no side effects:

```bash
# 1. Basic connectivity
ping -c 100 <gateway>
# Expected: 0% packet loss

# 2. Throughput (compare against baseline firmware)
iperf3 -c <server> -t 60 -P 4
iperf3 -c <server> -t 60 -P 4 -R
# Expected: within 5% of baseline throughput

# 3. Stress open/close cycles
for i in $(seq 1 50); do
    ifconfig eth0 down
    sleep 0.5
    ifconfig eth0 up
    sleep 1
done
ping -c 5 <gateway>
# Expected: no kernel oops, link recovers every time

# 4. Long duration stability
iperf3 -c <server> -t 3600
# Expected: no drops, no crashes over 1 hour

# 5. MTU change under load
iperf3 -c <server> -t 30 &
sleep 5
ifconfig eth0 mtu 9000
sleep 10
ifconfig eth0 mtu 1500
wait
# Expected: recovers without crash (recovery path is exercised)

# 6. Memory check after extended use
cat /proc/meminfo | grep -E "MemAvailable|Slab"
# Compare with baseline after same workload — should be similar

# 7. Module load/unload cycle (if module is loadable)
ifconfig eth0 down
rmmod sgmac && insmod /lib/modules/*/sgmac.ko
ifconfig eth0 up
ping -c 5 <gateway>
# Expected: clean load/unload, no dmesg errors
```

---

## Summary Table

| Fix | Trigger Condition | Pass Criterion | Key Command |
|-----|-------------------|----------------|-------------|
| 1 - PHY disconnect | OOM during `ifconfig up` | Retry succeeds; no kmemleak | `echo scan > /sys/kernel/debug/kmemleak` |
| 2 - DMA unmap | `echo autoDelay > gmac_debug` | `dma-api/num_errors` stays 0 | `cat /sys/kernel/debug/dma-api/num_errors` |
| 3 - Clock cleanup | `rmmod sgmac` 10x | `eth_tclk` enable_cnt == 1 | `cat /sys/kernel/debug/clk/clk_summary` |
| 4 - Ethtool ops | `ethtool eth0` on switch board | Correct driver info & stats | `ethtool -i eth0` |

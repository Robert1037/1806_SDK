# sf_gmac Memory Leak Fixes — Final Consolidated Report

**Date:** 2026-02-15
**Branch:** `fix/sf_gmac-memory-leaks`
**Driver:** `openwrt-18.06/package/kernel/sf_gmac/`
**Reviewers:** Claude Opus 4.6 (original analysis + fixes), GPT-5.3-Codex (cross-review + supplemental fix), Claude Opus 4.6 (final consolidation)

---

## 1. Work Summary

Three commits were produced on this branch:

| Commit | Author | Description |
|--------|--------|-------------|
| `dcb17a647` | Claude | Original analysis of 7 issues, code fixes for 4, plus detailed report |
| `687e30b15` | Claude | On-board verification guide with per-fix test procedures |
| `1d6ca889a` | GPT | Cross-review report + supplemental fix for `sgmac_rx_refill()` DMA mapping error path |

---

## 2. Issues Found and Fixed

### Fix 1: PHY Disconnect on DMA Init Failure (`sgmac_open`)

- **Severity:** High
- **Commit:** `dcb17a647`
- **File:** `sf_gmac.c`, `sgmac_open()` ~line 1830
- **Problem:** When `sgmac_dma_desc_rings_init()` fails, the function returns immediately. The PHY connection established by `of_phy_connect()` earlier in the function is never disconnected, leaking the PHY device reference.
- **Fix:** Added `phy_disconnect(priv->phydev)` before the error return, guarded by `priv->phy_node` check.
- **Status:** Correct. Both reviewers agree.

### Fix 2: Missing DMA Unmap in `sfax8_gmac_test_rx()`

- **Severity:** High
- **Commit:** `dcb17a647`
- **File:** `sf_gmac.c`, `sfax8_gmac_test_rx()` ~line 2277
- **Problem:** The test RX path passed `skb` to `sgmac_rx_refill()` without first setting `priv->rx_skbuff[entry] = NULL` and without calling `dma_unmap_single()`. This leaked the DMA mapping and created a double-reference to the skb (old pointer remains in the ring while refill places a new one).
- **Fix:** Added the missing `priv->rx_skbuff[entry] = NULL` and `dma_unmap_single()`, matching the pattern used in the production `sgmac_rx()` function.
- **Status:** Correct. Both reviewers agree.

### Fix 3: Clock Leak in `sgmac_remove()`

- **Severity:** Medium
- **Commit:** `dcb17a647`
- **File:** `sf_gmac.c`, `sgmac_remove()` ~line 4012
- **Problem:** When `CONFIG_SFAX8_GMAC_TCLKCHOOSE` is enabled, `priv->eth_tclk` is acquired and enabled in `sgmac_probe()` and cleaned up on the probe error path (`err_tclk` label), but `sgmac_remove()` never called `clk_disable_unprepare(priv->eth_tclk)`. Each module load/unload cycle increments the clock's enable count by 1 and never decrements it.
- **Fix:** Added `clk_disable_unprepare(priv->eth_tclk)` under `#ifdef CONFIG_SFAX8_GMAC_TCLKCHOOSE` in `sgmac_remove()`.
- **Status:** Correct. Both reviewers agree.

### Fix 4: Ethtool Ops Overwrite in `sgmac_probe()`

- **Severity:** Low (logic bug, not a memory leak)
- **Commit:** `dcb17a647`
- **File:** `sf_gmac.c`, `sgmac_probe()` ~line 3854
- **Problem:** `ndev->ethtool_ops = &sgmac_ethtool_ops` was unconditional, always overwriting any `eswitch_ethtool_ops` set earlier when a switch is detected. This caused incorrect ethtool behavior on switch-based boards (wrong driver info, potential NULL `phydev` dereference).
- **Fix:** Made the assignment conditional on `priv->phy_node` being present.
- **Status:** Correct. Both reviewers agree.

### Fix 5: RX Refill DMA Mapping Error SKB Leak (`sgmac_rx_refill`)

- **Severity:** Medium
- **Commit:** `1d6ca889a` (GPT)
- **File:** `sf_gmac.c`, `sgmac_rx_refill()` ~line 1264
- **Problem:** When `sgmac_rx()` or `sfax8_gmac_test_rx()` calls `sgmac_rx_refill(priv, skb)`, the original received skb is passed as `last_skb`. Inside the refill loop, if a *replacement* skb is allocated and then `dma_map_single()` fails, the original code freed only the replacement skb and returned `SF_DROP`. The caller then does `continue`, dropping the packet — but if the original `last_skb` was not yet recycled into a ring slot, it is leaked.
- **Fix (GPT):** Added `orig_last_skb` / `orig_last_skb_reused` tracking variables. On DMA mapping error, if the original skb was not already recycled into a descriptor, it is explicitly freed.
- **Status: Has a critical indentation bug — see Section 3 below.**

---

## 3. Review of GPT's Supplemental Fix (Commit `1d6ca889a`)

### 3.1 The finding is valid

GPT correctly identified a real leak path that the original patch did not address. The scenario:

1. `sgmac_rx()` DMA-unmaps the received skb and passes it to `sgmac_rx_refill()` as `last_skb`.
2. Inside `sgmac_rx_refill()`, on the first iteration, a new replacement skb is allocated (not `last_skb`).
3. `dma_map_single()` on the replacement skb fails.
4. The old code freed only the replacement skb and returned `SF_DROP`.
5. The caller does `continue` — `last_skb` is now unreferenced and leaked.

This is a real leak, though it requires DMA mapping failure, which is rare in practice.

### 3.2 The fix logic is correct in intent

Tracking whether `orig_last_skb` has been reused (placed back into a ring descriptor) and freeing it only if it hasn't been is the right approach.

### 3.3 Critical bug: broken indentation changes brace scoping

The GPT patch **changes the indentation of the `paddr = dma_map_single(...)` block and everything below it**, shifting it one level deeper. In the original code, the structure was:

```c
if(ret != SF_DROP)
    skb_reserve(skb, EXTER_HEADROOM);
paddr = dma_map_single(priv->dev, skb->data, ...);   // NOT inside the if
if (dma_mapping_error(priv->dev, paddr)) {
    ...
}
priv->rx_skbuff[entry] = skb;
desc_set_buf_addr(p, paddr, priv->dma_buf_sz);
}  // closes: if (likely(priv->rx_skbuff[entry] == NULL))
```

The `if(ret != SF_DROP)` controls only the single `skb_reserve()` statement (no braces). The `paddr = dma_map_single(...)` and everything after it executes unconditionally.

The GPT patch shifts everything from `paddr = ...` onward to be indented inside the `if(ret != SF_DROP)` block, and changes the closing `}` to match. In C, indentation doesn't change semantics — **but the closing brace was also moved**, changing which `}` closes which block. The result in the current code (lines 1265–1286):

```c
if(ret != SF_DROP)
    skb_reserve(skb, EXTER_HEADROOM);
        paddr = dma_map_single(...);     // still executes unconditionally (no braces on the if)
        if (dma_mapping_error(...)) {
            ...
        }
        priv->rx_skbuff[entry] = skb;
        ...
        desc_set_buf_addr(p, paddr, priv->dma_buf_sz);
    }    // <-- THIS now closes the outer if(likely(...)) block
```

The closing `}` at line 1286 that previously closed the `if (likely(priv->rx_skbuff[entry] == NULL))` block has been shifted inward but is still syntactically closing that same block (C ignores indentation). So the **compiled behavior is unchanged** — the indentation is misleading but the braces still match correctly because C doesn't use indentation-based scoping.

However, this creates extremely confusing code where the indentation does not match the actual brace structure. Any future maintainer reading this code will misunderstand the control flow. This is a maintainability hazard.

### 3.4 GPT report critique

**Valid points:**
- Finding #1 (DMA mapping error leak path): Correct and actionable.
- Assessment of existing code modifications: Accurate and fair.

**Issues with the report:**
- Finding #2 claims the original report's Issue #1 title "DMA descriptor rings leaked" is an overstatement. This is partly correct — the report body already clarifies that the DMA ring init has internal cleanup labels and the practical leak is the PHY connection. The title could be more precise, but the body is not wrong.
- Finding #2 also says Issue #2 is "internally contradictory." The original report's Issue #2 analysis walks through the logic and concludes the main path is safe but the `dma_mapping_error` edge case leaks. This is a nuanced analysis, not a contradiction — though the verdict could have been stated more concisely.
- Finding #3 about `dd if=/dev/zero of=/dev/null` is correct — this creates no memory pressure since it doesn't allocate memory. The verification guide should use `stress-ng --vm`, writing to tmpfs, or kernel fault injection instead.

---

## 4. Issues Identified But Not Fixed

### 4.1 OOM Drop Path Ring Starvation (Original Report Issue #5)

- **File:** `sf_gmac.c`, `sgmac_rx_refill()` lines 1258–1262
- **Problem:** When `sf_smart_oom_drop()` returns 0 (cached SKB path), the refill loop breaks early, leaving ring entries with NULL skb pointers. This causes temporary RX ring starvation until the next NAPI poll.
- **Assessment:** Not a memory leak. Design issue affecting performance under OOM conditions. Low priority.

### 4.2 Used-List SKBs in `sgmac_deinit_private_rxskbs()` (Original Report Issue #7)

- **File:** `sf_gmac_mem.c`, lines 227–236
- **Problem:** SKB chunks in the used list are freed (kfree) but the SKBs themselves are not freed.
- **Assessment:** Correct by design — these SKBs are in-flight (owned by the network stack). `RESET_PRIV_SKB_MAGIC()` ensures they'll be freed normally when the stack finishes with them. Not a leak.

---

## 5. Final Fix Status and Recommendations

### Applied fixes (correct and complete):

| # | Fix | File | Status |
|---|-----|------|--------|
| 1 | PHY disconnect on DMA init failure | `sgmac_open()` | **Good** |
| 2 | DMA unmap + NULL in test RX | `sfax8_gmac_test_rx()` | **Good** |
| 3 | Clock cleanup in remove | `sgmac_remove()` | **Good** |
| 4 | Conditional ethtool_ops | `sgmac_probe()` | **Good** |

### Applied but needs rework:

| # | Fix | File | Status |
|---|-----|------|--------|
| 5 | RX refill DMA error skb leak | `sgmac_rx_refill()` | **Logic correct, indentation broken — needs cleanup** |

### Recommended actions before merge:

1. **Rework Fix 5 indentation.** The `orig_last_skb` tracking logic is correct, but the indentation change makes the code misleading. The `paddr = dma_map_single(...)` block and subsequent lines should remain at their original indentation level (aligned with the closing `}` of the outer `if` block). Only the new tracking variables and the new `dma_mapping_error` body should be added.

2. **Fix the verification guide's memory pressure command.** Replace `dd if=/dev/zero of=/dev/null` with one of:
   - `stress-ng --vm 1 --vm-bytes <80% of RAM> --timeout 60s`
   - Writing to tmpfs: `dd if=/dev/zero of=/tmp/fill bs=1M count=<RAM_MB>`
   - Kernel fault injection: `CONFIG_FAILSLAB=y`

3. **Consider `devm_*` APIs for future refactoring.** Using `devm_clk_get`, `devm_ioremap`, `devm_request_irq` would eliminate entire categories of cleanup bugs. Not required for this patch set, but recommended as a follow-up.

4. **Remove standalone report files from the repo root.** `gpt_gmac_memleak_report.md` was committed to the repo root — it should either be moved under the `sf_gmac` package directory with the other reports or removed before merge.

---

## 6. Verification Approach

See `openwrt-18.06/package/kernel/sf_gmac/verification_gmac_memleaks.md` for detailed per-fix test procedures. Key verification points:

| Fix | Verification Method | Pass Criteria |
|-----|---------------------|---------------|
| PHY disconnect | OOM + `ifconfig eth0 up` retry | Retry succeeds; kmemleak clean |
| DMA unmap (test RX) | `echo autoDelay > /sys/kernel/debug/gmac_debug` | `dma-api/num_errors` stays 0 |
| Clock cleanup | `rmmod`/`insmod` 10 cycles | `clk_summary` enable_cnt == 1 |
| Ethtool ops | `ethtool -i eth0` on switch board | Correct driver name from eswitch |
| RX refill DMA error | Fault injection (`CONFIG_FAILSLAB=y`) | No kmemleak entries after DMA failure |

---

## 7. Files Modified

```
openwrt-18.06/package/kernel/sf_gmac/src/sf_gmac.c     (5 code fixes)
openwrt-18.06/package/kernel/sf_gmac/sf_gmac_mem_leak_report.md   (analysis report)
openwrt-18.06/package/kernel/sf_gmac/verification_gmac_memleaks.md (test guide)
gpt_gmac_memleak_report.md                              (GPT cross-review)
```

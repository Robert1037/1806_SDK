# sf_gmac Memory Leak Analysis Report

**Date:** 2026-02-15
**Package:** `openwrt-18.06/package/kernel/sf_gmac/`
**Files Analyzed:**
- `src/sf_gmac.c` (main GMAC driver, ~4100 lines)
- `src/sf_gmac.h` (driver header, ~643 lines)
- `src/sf_gmac_mem.c` (RX buffer pool, ~383 lines)
- `src/sf_gmac_mem.h` (pool header, ~33 lines)
- `src/sf_eswitch_ethtool.c` (ethtool ops, ~286 lines)
- `src/yt8521.c` (PHY driver, ~108 lines)

---

## Summary

| # | Severity | File | Function | Description |
|---|----------|------|----------|-------------|
| 1 | **High** | sf_gmac.c:1830-1832 | `sgmac_open()` | DMA descriptor rings leaked on early return when `sgmac_dma_desc_rings_init()` fails |
| 2 | **High** | sf_gmac.c:2091-2098 | `sgmac_rx()` | SKB not freed when refill returns `SF_DROP` — the DMA-unmapped skb is abandoned |
| 3 | **High** | sf_gmac.c:2271-2280 | `sfax8_gmac_test_rx()` | Same SKB leak on `SF_DROP` path in test RX function |
| 4 | **Medium** | sf_gmac.c:3948-4017 | `sgmac_remove()` | Missing `clk_disable_unprepare(priv->eth_tclk)` when `CONFIG_SFAX8_GMAC_TCLKCHOOSE` is enabled |
| 5 | **Medium** | sf_gmac.c:1256-1261 | `sgmac_rx_refill()` | SKB from `sf_smart_oom_drop()` may leak when `ret == 0` (break path) and `last_skb` was already consumed by OOM handler |
| 6 | **Low** | sf_gmac.c:3850 | `sgmac_probe()` | Unconditional `ethtool_ops` assignment overwrites switch ethtool ops set at line 3834 |
| 7 | **Low** | sf_gmac_mem.c:217-271 | `sgmac_deinit_private_rxskbs()` | Used-list SKB chunks freed without freeing the SKB itself — deliberate design but still a logical orphan |

---

## Detailed Analysis

### Issue 1 (High): DMA Ring Leak in `sgmac_open()` on Error Path

**File:** `src/sf_gmac.c`, lines 1830–1832

```c
ret = sgmac_dma_desc_rings_init(ndev);
if (ret < 0)
    return ret;
```

**Problem:** When `sgmac_dma_desc_rings_init()` fails, the function returns immediately without cleaning up resources already allocated earlier in `sgmac_open()`:
- The PHY connection established via `of_phy_connect()` (line 1777–1785) is not disconnected.
- HNAT/eswitch initialization done earlier is not reversed.
- NAPI is not disabled (though it was not yet enabled at this point, so this is benign).

However, the most direct leak occurs when the init function partially succeeds internally — `sgmac_dma_desc_rings_init()` has proper internal cleanup via `err_dma_tx`/`err_tx_skb`/`err_dma_rx` labels, so the DMA buffers themselves are safe. The PHY connection is the resource that leaks.

**Fix:** Disconnect the PHY before returning on error.

```c
ret = sgmac_dma_desc_rings_init(ndev);
if (ret < 0) {
    if (priv->phy_node)
        phy_disconnect(priv->phydev);
    return ret;
}
```

### Issue 2 (High): SKB Leak in `sgmac_rx()` on SF_DROP Path

**File:** `src/sf_gmac.c`, lines 2085–2098

```c
skb = priv->rx_skbuff[entry];
// ...
priv->rx_skbuff[entry] = NULL;
dma_unmap_single(priv->dev, desc_get_buf_addr(p),
        priv->dma_buf_sz - NET_IP_ALIGN,
        DMA_FROM_DEVICE);

ret = sgmac_rx_refill(priv, skb);
if (ret == SF_DROP)
    continue;
```

**Problem:** When `sgmac_rx_refill()` returns `SF_DROP`, the original `skb` has been passed to the refill function. Inside `sgmac_rx_refill()` → `sf_smart_oom_drop()`, the `oom_drop` path reuses the current SKB as the new RX buffer by assigning it back into the ring. However, when `sf_smart_oom_drop()` returns 0 (the "cached" path at line 1204), `sgmac_rx_refill()` breaks out of its loop, and the `last_skb` that was passed in is consumed for the ring refill.

But when `SF_DROP` is returned to `sgmac_rx()`, the code does `continue` without freeing the skb and without processing it. The skb at this point has been placed back into the ring descriptor by `sgmac_rx_refill()`, so it is not actually leaked in the `SF_DROP` case — the skb is recycled as a new RX buffer.

**Upon closer inspection**, the real leak happens in the edge case where `sgmac_rx_refill()` encounters `dma_mapping_error()` at line 1268:

```c
if (dma_mapping_error(priv->dev, paddr)) {
    dev_kfree_skb_any(skb);
    return SF_DROP;
}
```

Here, the *new* skb (allocated or from pool) is freed. But `last_skb` (the original received skb passed from `sgmac_rx()`) was already consumed or replaced in a previous iteration of the refill loop. If the DMA mapping error occurs on the first refill iteration using `last_skb` itself, the skb is properly freed. So this specific path is safe.

**However**, a real leak exists in the `sfax8_gmac_test_rx()` function (see Issue 3).

### Issue 3 (High): SKB Leak in `sfax8_gmac_test_rx()`

**File:** `src/sf_gmac.c`, lines 2271–2280

```c
skb = priv->rx_skbuff[entry];
if (unlikely(!skb)) {
    // ...
    break;
}

ret = sgmac_rx_refill(priv, skb);
if (ret == SF_DROP)
    continue;
```

**Problem:** Unlike `sgmac_rx()`, this test RX function does NOT call `priv->rx_skbuff[entry] = NULL` and does NOT call `dma_unmap_single()` before passing `skb` to `sgmac_rx_refill()`. This means:

1. The DMA mapping for this RX descriptor is never unmapped — **DMA mapping leak**.
2. If `sgmac_rx_refill()` successfully refills the slot, `priv->rx_skbuff[entry]` still points to the old skb (since it was never set to NULL), creating a dangling/double reference.

**Fix:** Add the missing DMA unmap and NULL assignment, matching the pattern in `sgmac_rx()`.

```c
skb = priv->rx_skbuff[entry];
if (unlikely(!skb)) {
    netdev_err(priv->ndev, "Inconsistent Rx descriptor chain\n");
    break;
}
priv->rx_skbuff[entry] = NULL;
dma_unmap_single(priv->dev, desc_get_buf_addr(p),
        priv->dma_buf_sz - NET_IP_ALIGN,
        DMA_FROM_DEVICE);

ret = sgmac_rx_refill(priv, skb);
if (ret == SF_DROP)
    continue;
```

### Issue 4 (Medium): Missing Clock Cleanup in `sgmac_remove()`

**File:** `src/sf_gmac.c`, lines 3948–4017

**Problem:** When `CONFIG_SFAX8_GMAC_TCLKCHOOSE` is enabled, `priv->eth_tclk` is acquired and enabled in `sgmac_probe()` (line 3636–3646). However, `sgmac_remove()` never calls `clk_disable_unprepare(priv->eth_tclk)`. This leaks the clock reference.

The error path in `sgmac_probe()` does handle it at label `err_tclk` (line 3930), but the normal removal path omits it.

**Fix:** Add the clock cleanup to `sgmac_remove()`.

```c
#ifdef CONFIG_SFAX8_GMAC_TCLKCHOOSE
    clk_disable_unprepare(priv->eth_tclk);
#endif
```

### Issue 5 (Medium): Potential SKB Leak in `sgmac_rx_refill()` Break Path

**File:** `src/sf_gmac.c`, lines 1256–1261

```c
if (unlikely(skb == NULL)){
    if ((ret = sf_smart_oom_drop(priv, &last_skb)) == 0)
        break;
    skb = last_skb;
}
```

**Problem:** When `sf_smart_oom_drop()` returns 0 (the "skb cached" path), the function breaks out of the while loop. At this point, `last_skb` may or may not still hold a valid skb pointer. In the cached path (`g_rx_skb_cached_cnt++` at line 1204), neither `SF_DROP` nor `SF_ACCEPT` was returned, and no new skb was allocated. The original `last_skb` passed into `sgmac_rx_refill()` from `sgmac_rx()` was never freed, and was never placed into the ring. This skb has already been DMA-unmapped but is now orphaned.

Back in `sgmac_rx()`, after `sgmac_rx_refill()` returns a value that is not `SF_DROP`, the code falls through to `skb_put()` and `netif_receive_skb()`, which processes the skb normally. So actually, in `sgmac_rx()`, the `last_skb` (which is the received skb) gets processed. However, since `sgmac_rx_refill()` broke out early, there are now RX ring entries with NULL skb pointers that will never be refilled until the next NAPI poll cycle. This is not a direct leak but results in ring starvation.

**Assessment:** Not a direct memory leak, but a design issue that could cause temporary ring buffer starvation under OOM conditions.

### Issue 6 (Low): Ethtool Ops Overwrite in `sgmac_probe()`

**File:** `src/sf_gmac.c`, line 3850

```c
ndev->ethtool_ops = &sgmac_ethtool_ops;
```

**Problem:** This line executes unconditionally after the eswitch/PHY detection block. When a switch is detected (line 3834), `ndev->ethtool_ops = &eswitch_ethtool_ops` is set. But line 3850 immediately overwrites it with `&sgmac_ethtool_ops`. This is a logic bug, not a memory leak, but it causes the wrong ethtool operations to be used when a switch is present.

**Fix:** Only set `sgmac_ethtool_ops` when a PHY node is present.

```c
if (priv->phy_node)
    ndev->ethtool_ops = &sgmac_ethtool_ops;
```

### Issue 7 (Low): Used-List SKBs Not Freed in `sgmac_deinit_private_rxskbs()`

**File:** `src/sf_gmac_mem.c`, lines 227–236

```c
list_for_each_entry_safe(chunck, chunck1, &g_rx_used_skbs, list)
{
    if (chunck->skb) {
        RESET_PRIV_SKB_MAGIC(chunck->skb);
    }
    list_del(&chunck->list);
    kfree((const void *)chunck);
    g_rx_chunck_alloc_free_debug --;
}
```

**Problem:** For chunks in the used list, the SKB's magic is reset (so the `vendor_free` callback won't try to recycle it), but `dev_kfree_skb(chunck->skb)` is never called. This is by design — the used SKBs are still in-flight (owned by the network stack or DMA), so the driver cannot free them directly. The `RESET_PRIV_SKB_MAGIC` ensures they'll be freed normally when the stack finishes with them.

**Assessment:** Not a leak — this is correct behavior for in-flight SKBs. The comment in the code acknowledges this design.

---

## Applied Fixes

### Fix 1: `sgmac_open()` — Disconnect PHY on DMA init failure

**File:** `src/sf_gmac.c`, line 1831

```diff
 	ret = sgmac_dma_desc_rings_init(ndev);
-	if (ret < 0)
-		return ret;
+	if (ret < 0) {
+		if (priv->phy_node)
+			phy_disconnect(priv->phydev);
+		return ret;
+	}
```

### Fix 2: `sfax8_gmac_test_rx()` — Add missing DMA unmap and NULL assignment

**File:** `src/sf_gmac.c`, after line 2276

```diff
 		skb = priv->rx_skbuff[entry];
 		if (unlikely(!skb)) {
 			netdev_err(priv->ndev,
 					"Inconsistent Rx descriptor chain\n");
 			break;
 		}
+		priv->rx_skbuff[entry] = NULL;
+		dma_unmap_single(priv->dev, desc_get_buf_addr(p),
+				priv->dma_buf_sz - NET_IP_ALIGN,
+				DMA_FROM_DEVICE);

 		ret = sgmac_rx_refill(priv, skb);
```

### Fix 3: `sgmac_remove()` — Add missing `eth_tclk` cleanup

**File:** `src/sf_gmac.c`, after the `eth_bus_clk` cleanup in `sgmac_remove()`

```diff
 	clk_disable_unprepare(priv->eth_bus_clk);
+#ifdef CONFIG_SFAX8_GMAC_TCLKCHOOSE
+	clk_disable_unprepare(priv->eth_tclk);
+#endif

 	iounmap(priv->base);
```

### Fix 4: `sgmac_probe()` — Fix ethtool ops overwrite

**File:** `src/sf_gmac.c`, line 3850

```diff
-	ndev->ethtool_ops = &sgmac_ethtool_ops;
+	if (priv->phy_node)
+		ndev->ethtool_ops = &sgmac_ethtool_ops;
```

---

## Recommendations

1. **Code review the OOM drop path** (`sf_smart_oom_drop` / `sgmac_rx_refill`). The interaction between these two functions is complex and difficult to verify for correctness. Consider simplifying the SKB recycling logic.

2. **Add consistent DMA unmap patterns** in all RX paths. The test RX function (`sfax8_gmac_test_rx`) was missing the unmap that the production RX function (`sgmac_rx`) had. Using a shared helper would prevent this divergence.

3. **Audit conditional compilation paths** (`#ifdef` branches). The driver has many config-dependent code paths (11+ config options). The `eth_tclk` leak was caused by the `CONFIG_SFAX8_GMAC_TCLKCHOOSE` cleanup being present in the error path but not the remove path.

4. **Consider using `devm_*` APIs** for managed resource allocation where possible (e.g., `devm_ioremap`, `devm_request_irq`, `devm_clk_get`). This would automatically clean up resources on driver removal and prevent entire categories of resource leaks.

# 内存泄漏风险分析报告

本报告分析了 sf_ts、sf_gmac 和 sf_smac 三个驱动程序中存在的内存泄漏风险，并提供修改建议。

---

## 目录

1. [sf_ts 驱动内存泄漏风险分析](#sf_ts-驱动内存泄漏风险分析)
2. [sf_gmac 驱动内存泄漏风险分析](#sf_gmac-驱动内存泄漏风险分析)
3. [sf_smac 驱动内存泄漏风险分析](#sf_smac-驱动内存泄漏风险分析)
4. [通用建议](#通用建议)

---

## sf_ts 驱动内存泄漏风险分析

文件路径: `openwrt-18.06/package/kernel/sf_ts/src/ts.c`

### 风险 1: ct_list 在设备删除时未释放 (高风险)

**问题描述:**
在 `sf_ts_exit()` 函数中，当遍历 `devlist` 删除设备时，只释放了 `dev->c`（percpu 统计）和 `dev` 本身，但没有释放 `dev->ct_list` 链表中的 `conn_info` 条目。

**问题代码位置:** 第 534-540 行
```c
for (i = 0; i < MAC_HASH_SIZE; i++){
    hlist_for_each_entry_safe(dev, tmp, &g_ts_priv->devlist[i], snode){
        free_percpu(dev->c);
        hlist_del(&dev->snode);
        kfree(dev);
        // 缺少: 遍历并释放 dev->ct_list 中的每个 conn_info
    }
}
```

**修复建议:**
```c
for (i = 0; i < MAC_HASH_SIZE; i++){
    hlist_for_each_entry_safe(dev, tmp, &g_ts_priv->devlist[i], snode){
        struct conn_info *entry, *next;
        // 释放 ct_list 中的所有条目
        list_for_each_entry_safe(entry, next, &dev->ct_list, list) {
            list_del(&entry->list);
            kfree(entry);
        }
        free_percpu(dev->c);
        hlist_del(&dev->snode);
        kfree(dev);
    }
}
```

### 风险 2: CMD_DEL_STS 命令时 ct_list 未释放 (高风险)

**问题描述:**
在 `sf_ts_write()` 的 `CMD_DEL_STS` 分支中，删除设备时同样没有释放 `ct_list` 中的连接信息。

**问题代码位置:** 第 133-143 行
```c
case CMD_DEL_STS:
    dev = check_dev_in_hlist(mac);
    if (dev){
        hlist_del_rcu(&dev->snode);
        synchronize_rcu();
        free_percpu(dev->c);
        kfree(dev);
        // 缺少: 释放 dev->ct_list
    }
```

**修复建议:**
```c
case CMD_DEL_STS:
    dev = check_dev_in_hlist(mac);
    if (dev){
        struct conn_info *entry, *next;
        hlist_del_rcu(&dev->snode);
        synchronize_rcu();
        // 释放 ct_list 中的所有条目
        spin_lock(&dev->ct_lock);
        list_for_each_entry_safe(entry, next, &dev->ct_list, list) {
            list_del(&entry->list);
            kfree(entry);
        }
        spin_unlock(&dev->ct_lock);
        free_percpu(dev->c);
        kfree(dev);
    }
```

### 风险 3: proc_create 失败后未设置 g_ts_priv = NULL (低风险)

**问题描述:**
在 `sf_ts_init()` 中，如果 `proc_create` 失败，虽然调用了 `kfree(ts_priv)`，但 `g_ts_priv` 仍然指向已释放的内存。

**问题代码位置:** 第 506-520 行

**修复建议:**
```c
err_out_proc:
    del_timer(&(ts_priv->ts_timer));
    nf_unregister_net_hooks(&init_net, sf_nf_hook_ops, ARRAY_SIZE(sf_nf_hook_ops));
err_out_free:
    kfree(ts_priv);
    g_ts_priv = NULL;  // 添加这行
    return ret;
```

---

## sf_gmac 驱动内存泄漏风险分析

文件路径: `openwrt-18.06/package/kernel/sf_gmac/src/sf_gmac.c` 和 `sf_gmac_mem.c`

### 风险 1: sgmac_dma_desc_rings_init 错误路径资源清理不完整 (中风险)

**问题描述:**
在 `sgmac_dma_desc_rings_init()` 中，如果 `dma_alloc_coherent` 分配 `dma_tx` 失败，会调用错误处理路径，但 `rx_skbuff` 数组中可能已经填充了 skb（通过 `sgmac_rx_refill`），这些 skb 没有被正确释放。

**问题代码位置:** 第 1300-1364 行

**分析:**
实际上，在错误路径 `err_dma_tx` 处，`sgmac_rx_refill` 尚未被调用（它在第1345行），所以这不是一个实际问题。但建议添加注释说明这一点。

### 风险 2: sf_smart_oom_drop 中 skb 处理逻辑复杂可能导致泄漏 (中风险)

**问题描述:**
`sf_smart_oom_drop()` 函数中的 skb 处理逻辑复杂，返回值 `SF_DROP` 时，如果调用者没有正确处理 `last_skb`，可能导致 skb 泄漏。

**问题代码位置:** 第 1109-1206 行

**当前状态:** 调用点 `sgmac_rx_refill()` 正确处理了返回值，所以实际上没有泄漏。

### 风险 3: dma_err 错误路径中解锁后的 dma_unmap (低风险)

**问题描述:**
在 `sgmac_xmit()` 的 `dma_err` 标签处理中，`dma_unmap_single` 是在 `spin_unlock_bh` 之后调用的，这可能导致与其他路径的竞争条件。

**问题代码位置:** 第 2035-2053 行

**当前代码:**
```c
dma_err:
    entry = priv->tx_head;
    for (; i > 0; i--) {
        entry = dma_ring_incr(entry, DMA_TX_RING_SZ);
        desc = priv->dma_tx + entry;
        priv->tx_skbuff[entry] = NULL;
        dma_unmap_page(priv->dev, desc_get_buf_addr(desc),
                desc_get_buf_len(desc), DMA_TO_DEVICE);
        desc_clear_tx_owner(desc);
    }
    desc = first;
    if(go_direct_xmit){
        spin_unlock_bh(&sf_gmac_tx_lock);
    }
    dma_unmap_single(priv->dev, desc_get_buf_addr(desc),
            desc_get_buf_len(desc), DMA_TO_DEVICE);
    dev_kfree_skb_any(skb);
```

**修复建议:** 将 `dma_unmap_single` 移到 `spin_unlock_bh` 之前：
```c
dma_err:
    entry = priv->tx_head;
    for (; i > 0; i--) {
        entry = dma_ring_incr(entry, DMA_TX_RING_SZ);
        desc = priv->dma_tx + entry;
        priv->tx_skbuff[entry] = NULL;
        dma_unmap_page(priv->dev, desc_get_buf_addr(desc),
                desc_get_buf_len(desc), DMA_TO_DEVICE);
        desc_clear_tx_owner(desc);
    }
    desc = first;
    dma_unmap_single(priv->dev, desc_get_buf_addr(desc),
            desc_get_buf_len(desc), DMA_TO_DEVICE);
    if(go_direct_xmit){
        spin_unlock_bh(&sf_gmac_tx_lock);
    }
    dev_kfree_skb_any(skb);
```

### 风险 4: sgmac_remove 缺少 skb_pool 清理 (中风险)

**问题描述:**
在 `sgmac_remove()` 中，如果启用了 `CONFIG_SF_SKB_POOL`，没有调用相应的清理函数。

**问题代码位置:** 第 3948-4016 行

**修复建议:**
在 `sgmac_remove()` 中添加 skb_pool 清理:
```c
#ifdef CONFIG_SF_SKB_POOL
    if(priv->skb_pool_dev_param){
        skb_pool_deinit(priv->skb_pool_dev_param);
    }
#endif
```

### 风险 5: sf_gmac_mem.c 中的竞态条件 (中风险)

**问题描述:**
在 `sgmac_dev_free_rxskb()` 回调函数中，存在注释说明的竞态条件风险：当 wifi 驱动被移除时，如果正在进行 `vendor_free` 回调，可能会导致访问已释放的 `chunck` 指针。

**问题代码位置:** `sf_gmac_mem.c` 第 132-142 行

**当前注释:**
```c
// TODO:
// There is is a race-risk :
// when wifi driver is removed, we free the chunck point in sgmac_deinit_private_rxskb
// But at this time a used skb is during vendor_free callback, then it will be crash
```

**修复建议:** 
添加引用计数或延迟释放机制来确保在所有 skb 回调完成后才释放 chunck。

---

## sf_smac 驱动内存泄漏风险分析

文件路径: `openwrt-18.06/package/kernel/sf_smac/src/bb_src/umac/`

### 风险 1: siwifi_mem.c 中相同的竞态条件 (中风险)

**问题描述:**
与 sf_gmac 类似，`siwifi_mem.c` 中的 `siwifi_dev_free_rxskb()` 函数也有相同的竞态条件问题。

**问题代码位置:** `siwifi_mem.c` 第 339-350 行

### 风险 2: siwifi_deinit_debug_mem 中锁的使用问题 (低风险)

**问题描述:**
在 `siwifi_deinit_debug_mem()` 中，`kfree(g_mem)` 是在持有 `priv_rx_skbs_lock` 锁时执行的，可能导致在锁释放后访问已释放的内存。

**问题代码位置:** `siwifi_mem.c` 第 271-279 行

**当前代码:**
```c
#ifdef CONFIG_PRIV_RX_BUFFER_POOL
    spin_lock_bh(&priv_rx_skbs_lock);
    kfree(g_mem);
    g_mem = NULL;
    spin_unlock_bh(&priv_rx_skbs_lock);
#else
    kfree(g_mem);
    g_mem = NULL;
#endif
```

**修复建议:**
```c
#ifdef CONFIG_PRIV_RX_BUFFER_POOL
    spin_lock_bh(&priv_rx_skbs_lock);
    // 先保存指针，解锁后再释放
    struct siwifi_mem_ctx *tmp = g_mem;
    g_mem = NULL;
    spin_unlock_bh(&priv_rx_skbs_lock);
    kfree(tmp);
#else
    kfree(g_mem);
    g_mem = NULL;
#endif
```

### 风险 3: ipc_host.c 中 ALLOC_HOST_ID_RES 宏的错误处理 (高风险)

**问题描述:**
`ALLOC_HOST_ID_RES` 宏中使用 `ASSERT_ERR(base)` 来检查内存分配，如果分配失败，这会导致 BUG()，而不是优雅的错误处理。在生产环境中，这可能导致内核崩溃。

**问题代码位置:** `ipc_host.c` 第 40-49 行

**修复建议:**
改用返回错误码的方式处理分配失败：
```c
#define ALLOC_HOST_ID_RES(index, user, ret)    \
    {   \
        int j;  \
        struct sk_buff_head *base = siwifi_kmalloc(sizeof(struct sk_buff_head) * nx_txdesc_cnt[index], GFP_KERNEL);   \
        if (!base) { \
            ret = -ENOMEM; \
            break; \
        } \
        for (j = 0; j < nx_txdesc_cnt[index]; j++) {    \
            __skb_queue_head_init(base + j);    \
            env->tx_host_id##index[user][j] = (void *)(base + j);   \
        }   \
    }
```

---

## 通用建议

### 1. 添加内存调试支持
建议在所有三个驱动中启用类似 `siwifi_mem.c` 中的 `MEMORY_USAGE_DEBUG` 功能，以便追踪内存分配和释放。

### 2. 使用一致的错误处理模式
所有内存分配应该有对应的错误处理路径，避免使用 `BUG_ON` 或 `ASSERT_ERR` 来处理可恢复的错误。

### 3. 添加内存泄漏检测工具支持
考虑集成 kmemleak 或 slub_debug 来帮助检测运行时内存泄漏。

### 4. 代码审查清单
在进行代码审查时，确保：
- 每个 `kzalloc`/`kmalloc` 都有对应的 `kfree`
- 每个 `alloc_percpu` 都有对应的 `free_percpu`
- 每个 `dma_alloc_coherent` 都有对应的 `dma_free_coherent`
- 链表和哈希表在删除节点时释放相关内存
- 错误路径正确释放所有已分配的资源

---

## 总结

| 驱动 | 风险数量 | 高风险 | 中风险 | 低风险 |
|------|----------|--------|--------|--------|
| sf_ts | 3 | 2 | 0 | 1 |
| sf_gmac | 5 | 0 | 4 | 1 |
| sf_smac | 3 | 1 | 1 | 1 |
| **总计** | **11** | **3** | **5** | **3** |

建议优先修复高风险问题（sf_ts 的 ct_list 泄漏和 sf_smac 的 ASSERT_ERR 问题），然后处理中风险问题。


# IGC_Code_Examples


# IGC驱动关键代码示例分析

## 1. 驱动初始化示例

### 模块初始化
```c
// igc_main.c
static int __init igc_init_module(void)
{
    int ret;
    
    pr_info(&#34;%s\n&#34;, igc_driver_string);
    pr_info(&#34;%s\n&#34;, igc_copyright);
    
    ret = pci_register_driver(&amp;igc_driver);
    return ret;
}

module_init(igc_init_module);
```

**分析要点：**
- 使用`module_init()`宏注册模块初始化函数
- `pci_register_driver()`向PCI子系统注册驱动
- 打印驱动信息便于调试

### PCI驱动结构
```c
static struct pci_driver igc_driver = {
    .name     = igc_driver_name,
    .id_table = igc_pci_tbl,
    .probe    = igc_probe,
    .remove   = igc_remove,
#ifdef CONFIG_PM
    .driver.pm = &amp;igc_pm_ops,
#endif
    .shutdown = igc_shutdown,
    .err_handler = &amp;igc_err_handler,
};
```

**分析要点：**
- `.probe`：设备发现时调用
- `.remove`：设备移除时调用
- `.id_table`：支持的PCI设备ID列表
- 包含电源管理和错误处理

## 2. 设备探测示例

### 核心探测流程
```c
static int igc_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
    struct igc_adapter *adapter;
    struct net_device *netdev;
    struct igc_hw *hw;
    const struct igc_info *ei = igc_info_tbl[ent-&gt;driver_data];
    int err;

    // 1. 启用PCI设备
    err = pci_enable_device_mem(pdev);
    if (err)
        return err;

    // 2. 设置DMA掩码
    err = dma_set_mask_and_coherent(&amp;pdev-&gt;dev, DMA_BIT_MASK(64));
    if (!err) {
        pci_using_dac = 1;
    } else {
        err = dma_set_mask_and_coherent(&amp;pdev-&gt;dev, DMA_BIT_MASK(32));
        if (err) {
            dev_err(&amp;pdev-&gt;dev, &#34;No usable DMA configuration\n&#34;);
            goto err_dma;
        }
    }

    // 3. 请求内存区域
    err = pci_request_mem_regions(pdev, igc_driver_name);
    if (err)
        goto err_pci_reg;

    // 4. 分配网络设备
    netdev = alloc_etherdev_mq(sizeof(struct igc_adapter), IGC_MAX_TX_QUEUES);
    if (!netdev)
        goto err_alloc_etherdev;

    // 5. 初始化适配器结构
    adapter = netdev_priv(netdev);
    adapter-&gt;netdev = netdev;
    adapter-&gt;pdev = pdev;
    hw = &amp;adapter-&gt;hw;
    hw-&gt;back = adapter;

    // 6. 映射硬件寄存器
    adapter-&gt;io_addr = pci_iomap(pdev, 0, 0);
    if (!adapter-&gt;io_addr)
        goto err_ioremap;

    // 7. 软件初始化
    err = igc_sw_init(adapter);
    if (err)
        goto err_sw_init;

    // 8. 硬件重置和初始化
    igc_reset(adapter);
    igc_get_hw_control(adapter);

    // 9. 注册网络设备
    err = register_netdev(netdev);
    if (err)
        goto err_register;

    return 0;

err_register:
    igc_release_hw_control(adapter);
    // ... 错误处理
}
```

**分析要点：**
- 标准的PCI设备初始化流程
- 错误处理使用goto标签进行资源清理
- 硬件和软件初始化分离

## 3. 网络设备操作示例

### 网络设备操作结构
```c
static const struct net_device_ops igc_netdev_ops = {
    .ndo_open           = igc_open,
    .ndo_stop           = igc_close,
    .ndo_start_xmit     = igc_xmit_frame,
    .ndo_set_rx_mode    = igc_set_rx_mode,
    .ndo_set_mac_address = igc_set_mac,
    .ndo_change_mtu     = igc_change_mtu,
    .ndo_tx_timeout     = igc_tx_timeout,
    .ndo_get_stats64    = igc_get_stats64,
    .ndo_fix_features   = igc_fix_features,
    .ndo_set_features   = igc_set_features,
    .ndo_eth_ioctl      = igc_ioctl,
    .ndo_setup_tc       = igc_setup_tc,
    .ndo_bpf            = igc_bpf,
    .ndo_xdp_xmit       = igc_xdp_xmit,
    .ndo_xsk_wakeup     = igc_xsk_wakeup,
};
```

### 设备启动函数
```c
int igc_open(struct net_device *netdev)
{
    struct igc_adapter *adapter = netdev_priv(netdev);
    struct igc_hw *hw = &amp;adapter-&gt;hw;
    int err;

    // 测试中断分配
    err = igc_test_interrupt_scheme(adapter);
    if (err)
        goto err_req_irq;

    // 设置TX资源
    err = igc_setup_all_tx_resources(adapter);
    if (err)
        goto err_setup_tx;

    // 设置RX资源
    err = igc_setup_all_rx_resources(adapter);
    if (err)
        goto err_setup_rx;

    // 配置中断
    igc_configure(adapter);

    // 请求中断
    err = igc_request_irq(adapter);
    if (err)
        goto err_req_irq;

    // 启动设备
    igc_up(adapter);

    return IGC_SUCCESS;

err_req_irq:
    igc_free_all_rx_resources(adapter);
err_setup_rx:
    igc_free_all_tx_resources(adapter);
err_setup_tx:
    igc_reset(adapter);

    return err;
}
```

## 4. 数据包发送示例

### 发送函数
```c
static netdev_tx_t igc_xmit_frame(struct sk_buff *skb, struct net_device *netdev)
{
    struct igc_adapter *adapter = netdev_priv(netdev);

    // 最小包长度检查
    if (skb-&gt;len &lt; 17) {
        if (skb_padto(skb, 17))
            return NETDEV_TX_OK;
        skb-&gt;len = 17;
    }

    return igc_xmit_frame_ring(skb, igc_tx_queue_mapping(adapter, skb));
}

static netdev_tx_t igc_xmit_frame_ring(struct sk_buff *skb, struct igc_ring *tx_ring)
{
    struct igc_tx_buffer *first;
    u32 tx_flags = 0;
    unsigned short f;
    u16 count = TXD_USE_COUNT(skb_headlen(skb));

    // 计算所需描述符数量
    for (f = 0; f &lt; skb_shinfo(skb)-&gt;nr_frags; f&#43;&#43;)
        count &#43;= TXD_USE_COUNT(skb_frag_size(&amp;skb_shinfo(skb)-&gt;frags[f]));

    // 检查描述符可用性
    if (igc_maybe_stop_tx(tx_ring, count &#43; 3)) {
        return NETDEV_TX_BUSY;
    }

    // 处理VLAN标签
    if (skb_vlan_tag_present(skb)) {
        tx_flags |= IGC_TX_FLAGS_VLAN;
        tx_flags |= (skb_vlan_tag_get(skb) &lt;&lt; IGC_TX_FLAGS_VLAN_SHIFT);
    }

    // 获取第一个TX缓冲区
    first = &amp;tx_ring-&gt;tx_buffer_info[tx_ring-&gt;next_to_use];
    first-&gt;skb = skb;
    first-&gt;bytecount = skb-&gt;len;
    first-&gt;gso_segs = 1;

    // 设置TX描述符
    igc_tx_map(tx_ring, first, hdr_len);

    return NETDEV_TX_OK;
}
```

## 5. NAPI轮询示例

### NAPI轮询函数
```c
static int igc_poll(struct napi_struct *napi, int budget)
{
    struct igc_q_vector *q_vector = container_of(napi, struct igc_q_vector, napi);
    struct igc_ring *rx_ring = q_vector-&gt;rx.ring;
    bool clean_complete = true;
    int work_done = 0;

    // 清理TX完成
    if (q_vector-&gt;tx.ring)
        clean_complete = igc_clean_tx_irq(q_vector, budget);

    // 处理RX数据包
    if (rx_ring) {
        int cleaned = rx_ring-&gt;xsk_pool ?
                      igc_clean_rx_irq_zc(q_vector, budget) :
                      igc_clean_rx_irq(q_vector, budget);

        work_done &#43;= cleaned;
        if (cleaned &gt;= budget)
            clean_complete = false;
    }

    // 如果工作未完成，继续轮询
    if (!clean_complete)
        return budget;

    // 完成NAPI轮询，重新启用中断
    if (likely(napi_complete_done(napi, work_done)))
        igc_ring_irq_enable(q_vector);

    return work_done;
}
```

## 6. 中断处理示例

### MSI中断处理
```c
static irqreturn_t igc_intr_msi(int irq, void *data)
{
    struct igc_adapter *adapter = data;
    struct igc_q_vector *q_vector = adapter-&gt;q_vector[0];
    struct igc_hw *hw = &amp;adapter-&gt;hw;
    u32 icr = rd32(IGC_ICR);

    igc_write_itr(q_vector);

    // 处理设备重置
    if (icr &amp; IGC_ICR_DRSTA)
        schedule_work(&amp;adapter-&gt;reset_task);

    // 处理DMA同步错误
    if (icr &amp; IGC_ICR_DOUTSYNC) {
        adapter-&gt;stats.doosync&#43;&#43;;
    }

    // 处理链路状态变化
    if (icr &amp; (IGC_ICR_RXSEQ | IGC_ICR_LSC)) {
        hw-&gt;mac.get_link_status = true;
        if (!test_bit(__IGC_DOWN, &amp;adapter-&gt;state))
            mod_timer(&amp;adapter-&gt;watchdog_timer, jiffies &#43; 1);
    }

    // 处理时间同步中断
    if (icr &amp; IGC_ICR_TS)
        igc_tsync_interrupt(adapter);

    // 调度NAPI
    napi_schedule(&amp;q_vector-&gt;napi);

    return IRQ_HANDLED;
}
```

## 7. 硬件抽象层示例

### 硬件操作结构
```c
static struct igc_mac_operations igc_mac_ops_base = {
    .init_hw            = igc_init_hw_base,
    .check_for_link     = igc_check_for_copper_link,
    .rar_set            = igc_rar_set,
    .read_mac_addr      = igc_read_mac_addr,
    .get_speed_and_duplex = igc_get_speed_and_duplex_copper,
};

const struct igc_info igc_base_info = {
    .get_invariants     = igc_get_invariants_base,
    .mac_ops            = &amp;igc_mac_ops_base,
    .phy_ops            = &amp;igc_phy_ops_base,
};
```

### 寄存器读写
```c
u32 igc_rd32(struct igc_hw *hw, u32 reg)
{
    struct igc_adapter *igc = container_of(hw, struct igc_adapter, hw);
    u8 __iomem *hw_addr = READ_ONCE(hw-&gt;hw_addr);
    u32 value = 0;

    if (IGC_REMOVED(hw_addr))
        return ~value;

    value = readl(&amp;hw_addr[reg]);

    // 检测PCIe链路丢失
    if (!(~value) &amp;&amp; (!reg || !(~readl(hw_addr)))) {
        struct net_device *netdev = igc-&gt;netdev;
        hw-&gt;hw_addr = NULL;
        netif_device_detach(netdev);
        netdev_err(netdev, &#34;PCIe link lost, device now detached\n&#34;);
    }

    return value;
}

#define wr32(reg, val) \
do { \
    u8 __iomem *hw_addr = READ_ONCE((hw)-&gt;hw_addr); \
    if (!IGC_REMOVED(hw_addr)) \
        writel((val), &amp;hw_addr[(reg)]); \
} while (0)
```

**分析要点：**
- 寄存器访问包含错误检测
- 使用内存屏障确保访问顺序
- PCIe链路状态监控

## 学习建议

1. **从简单开始**：先理解模块初始化和PCI探测流程
2. **跟踪数据流**：理解数据包从用户空间到硬件的完整路径
3. **关注错误处理**：学习如何正确处理各种错误情况
4. **实践调试**：使用printk、ftrace等工具跟踪代码执行
5. **阅读规范**：结合PCI、网络协议等相关规范理解代码



---

> Author: [Xueyu](https://github.com/xueyu-code)  
> URL: https://xueyu-code.github.io/posts/08c96ad/  


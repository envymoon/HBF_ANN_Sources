# HAVEN 精读笔记（arXiv 2603.01175, 2026-03, Georgia Tech / EPFL / UCSD）

PDF：`papers/HAVEN_arXiv2603.01175.pdf`（短文，正文约 5 页）

## 一句话
**把 IVF-PQ 的 rerank 阶段（取全精度向量、算精确距离）搬到 HBF 上，在 HBF 底下的 logic die 里做近存计算。** 粗量化和 PQ 扫描仍在 GPU + HBM 上完成。相比 GPU+DRAM / GPU+SSD，rerank 吞吐最高提升 20 倍、延迟最高降低 40 倍。

## 架构
```
GPU + HBM：①粗量化选 nprobe 个簇  ②扫描 PQ 码，得到 N_rerank 个候选 ID
          │ 只发候选 ID
          ▼
HBF logic die（近存搜索单元）：
   Rerank Queue ×32（每个 8 KB，可存 1024 个 ID + 距离）
   → Address Gen（ID → 物理地址）→ 读 HBF
   → 距离计算（32 个 MAC）→ Top-k（256 点 bitonic 排序）
   面积 4.11 mm²，功耗 620 mW（22 nm）
```
做法很直接：rerank 的计算强度低，与其把原始向量搬回 HBM，不如就地计算。

## 对 HBF 硬件的建模（§3.2）
- **把大 plane 拆成分布式小 subarray**，每个 subarray 有自己的 WL/BL 驱动和 sense amp → 并行度更高、单次访问更快。论文说这一思路参照了 ISSCC'18 的低延迟 NAND（tR 3 µs，其参考文献 [1]）。
- 基线配置：**SLC，4 KB 页，每个 subarray 64 个 block，256 层 WL**，8-Hi 堆叠，按 HBM2E 的 TSV 组织。
- **带宽受功耗限制，而不是只受并行度限制**：每栈 30 W 功耗上限，页越大、block 越多，每次读能耗越高，可持续带宽越低。block 数超过 128 后下降明显。最终假设 **460 GB/s/栈**（对齐 HBM2E）。
- 容量和性能方向相反：更大的 subarray 容量密度更高，但延迟和能耗也更高。

## 结果
- 召回 0.95（BIGANN-1B、SPACEV-1B）时 QPS 比 DRAM/SSD 方案高 3–8 倍；召回 0.9（Wiki-88M，768 维）时高 20 倍以上。
- 延迟约 10 ms，基本不随 batch 增大而上升。
- BIGANN-1B、召回 ≥0.9：8.1k QPS，是 ANNA（IVF-PQ ASIC）的 1.9 倍、SmartANNS（12 块 SmartSSD 上的图索引）的 9 倍。

## 它**没有**做的（我们需要知道的空白）
1. **没有讨论地址到 channel/bank 的映射、数据布局或请求调度。** Address Gen 只是“ID → 物理地址”。论文里没有出现 bank 冲突或页缓冲。
2. **没有给出 tR 的具体数值**（只有趋势图），也没有排队或时序级模拟；性能来自分析式建模。
3. **只处理 rerank，而 rerank 的访问天然是“一次全部确定”的**：PQ 扫描结束后，N_rerank 个 ID 一次性给出，彼此没有依赖，直接塞满 32 个队列就能并行。

## 对我们的意义
- 它证明了 “HBF 放 ANN 全精度数据 + 近存计算” 的价值，**但它挑的是最容易的访问模式（无依赖的批量随机读）**。
- 图 ANN 的访问是逐跳依赖的，HAVEN 的设计不能直接套用。这正是我们工作和它的区别。
- 可以借鉴的点：
  - ① 近存计算：在 logic die 上做距离计算和过滤，只把结果传回 GPU，**省 channel 带宽**；对图搜索还可以在 HBF 侧直接决定下一跳。
  - ② 功耗限制带宽：设计时不能假设“并发越多带宽越高”，**功耗/热是另一个上限**（和 HBFSim 的热分析互相印证）。
- **修正**：我们之前文档里说 “HAVEN 把簇数据条带化铺满 channel”，原文并没有这样说。HAVEN 好做的真正原因是 rerank 地址一次就全部确定、没有依赖。已在 02 文档中改正。

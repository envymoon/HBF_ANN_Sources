# Disk-Resident Graph ANN Search: An Experimental Evaluation 精读笔记（PVLDB 19(13), arXiv 2603.01779v2）

PDF：`papers/DiskGraphANN_Eval_arXiv2603.01779.pdf`　代码：https://github.com/OpenDataBox/Disk-ANN-Evaluation

## 一句话
把外存图 ANN 拆成 5 个组件（存储分配、磁盘布局、缓存、查询执行、更新），在统一平台上逐一对比：DiskANN、Starling、Gorgeous、PageANN、PipeANN、AlayaLaser、DGAI 等。**是了解这一领域全貌最快的一篇。**

## 五个组件的分类（只记要点）
| 组件 | 选项 | 要点 |
|---|---|---|
| 存储分配 | Major-in-disk（PQ 码放内存，图和向量放盘）/ All-in-disk（PQ 码也放盘） | 前者 I/O 少但内存随数据规模线性增长 |
| 全局布局 | **Coupled**（向量和邻接表放一起）/ **Decoupled**（邻接表放 index 块，向量放 data 块） | 维度 ≥约 1500 时 decoupled 更好 |
| 局部布局 | ID 顺序 / 启发式（Starling）/ 聚类（PageANN）/ 图复制（Gorgeous：一个块里多放几个邻居的邻接表） | **启发式最好，但 I/O 有效率仍然 ≤ 15%** |
| 缓存 | 静态（热点/图优先）/ 动态（LRU）/ 混合 | 低维用动态或混合，高维用静态（优先缓存邻接表） |
| 查询执行 | **Compute-driven**（算完再发 I/O：DiskANN、Starling、PageANN）/ **I/O-driven**（推测发 I/O，按到达顺序计算：PipeANN） | 高维时 I/O-driven 胜出 |

## 七个结论里与我们相关的
1. **I/O 有效率极低**：读进来的数据里，真正被搜索路径用到的比例，最好也只有 **12.55%**（16 KB 页）。原因是**现有布局没能把搜索路径上相继访问的节点放进同一页**。作者把“面向遍历路径的布局”列为开放问题。
2. **页越小越好**（只要放得下）：4 KB 通常优于 16 KB。大页读得更慢，多读的内容大多用不上。
3. **维度决定策略**：低维时 PageANN（布局）最好，高维时 PipeANN（流水）最好。规则：每页能放的向量数 BPF < 2 时用 I/O-driven 执行，BPF ≥ 2 时用 compute-driven。
4. SSD 上 beam width W 到约 16 就饱和；线程数加到 16–32 后 SSD 带宽饱和，QPS 不再上升。
5. I/O 时间占查询总时间的 71%–99.8%。
6. 动态更新：原地更新适合读多的负载，异地更新适合写多的负载。

## 和 HBF 的关系（我们的解读）
1. **“I/O 有效率 ≤ 15%”在 HBF 上要拆成两个指标：**
   - **传输有效率**：HBF 可以只取 64 B 的倍数，没用的部分不必传回，这项可以接近 100%；
   - **sense 有效率**：每次 sense（bank 被占用一个 tR）实际用上了多少个节点。这才是 HBF 上真正的瓶颈（见 `OCP_HBF规范_笔记.md` §6）。
   → 在 SSD 上，读一页就是付一次固定成本，低有效率只是浪费带宽。**在 HBF 上，低 sense 有效率直接浪费最稀缺的资源（bank 时间）。**
2. **Decoupled 布局在 HBF 上可能更自然**：邻接表很小，大量节点的邻接表可以塞进一个 index 页（一次 sense 覆盖很多节点的路由信息）；全精度向量放在 data 页，需要时只取 64 B 的倍数。
3. 作者提出的开放问题“面向遍历路径的布局”，和我们的问题 1 是同一个方向。我们在此基础上**再加一个 bank 维度**：同页放依赖链上相继的节点，不同 bank 放同一轮的候选。
4. SSD 上 W 在 16 左右饱和，是因为 SSD 并发度有限。HBF 有成百上千个 bank，**饱和点会大幅后移**，但前提是请求能分散开。

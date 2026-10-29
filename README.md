# HBF-ANN 调研仓库

研究方向：在 **HBF（High-Bandwidth Flash）** 上做**基于图的 ANN 检索**，通过系统设计和算法联合解决两个问题：
1. **Page Locality 与 Bank Parallelism 的错配**
2. **并发 I/O 的顺序与分布**（HBF 上不仅要决定发多少，还要决定按什么顺序发）

## 阅读顺序
| 文件 | 内容 |
|---|---|
| [docs/01_HBF基础.md](docs/01_HBF基础.md) | HBF 是什么、channel/bank/页缓冲、**为什么必须 prefetch（Little 定律）** |
| [docs/02_图ANN与磁盘IO.md](docs/02_图ANN与磁盘IO.md) | 图搜索 = 指针追逐；DiskANN、Starling 布局、PipeANN 流水 |
| [docs/03_两个难题根因分析.md](docs/03_两个难题根因分析.md) | ⭐ **核心**：A/B/C 推演、四类根因、设计杠杆、待确认事实 |
| [papers/HBFSim_笔记.md](papers/HBFSim_笔记.md) | HBFSim 精读笔记 |
| [papers/OCP_HBF规范_笔记.md](papers/OCP_HBF规范_笔记.md) | ⭐ OCP 规范精读：地址→bank 映射、读协议、SRAM、读干扰、**sense 速率才是真正的资源** |
| [papers/HBFlex_笔记.md](papers/HBFlex_笔记.md) | HBFlex 精读：plane 负载不均、热点副本、按位置调度（可借鉴到问题 1/2） |
| [papers/DiskGraphANN评测_笔记.md](papers/DiskGraphANN评测_笔记.md) | 外存图 ANN 全景评测：五组件分类、I/O 有效率 ≤15% |
| [papers/PipeANN_笔记.md](papers/PipeANN_笔记.md) | PipeANN 精读：伪依赖、动态 W、对 HBF 的含义（问题 2 起点） |
| [papers/Starling_笔记.md](papers/Starling_笔记.md) | Starling 精读：overlap ratio、NP-hard、BNF（问题 1 起点） |
| [papers/HAVEN_笔记.md](papers/HAVEN_笔记.md) | HAVEN 精读：HBF 上的 IVF-PQ rerank，以及它没做的部分 |
| [papers/参考文献.md](papers/参考文献.md) | 按优先级排列的文献清单 |
| [ideas/00_讨论议程.md](ideas/00_讨论议程.md) | 待办与方向草稿 |
| `assets/` | 讨论用的原始示意图 |

## 一段话总结
图搜索的访问是**由数据决定、逐跳展开**的；HBF 的带宽却要求**提前知道、大量并发、均匀分布**（延迟约为 HBM 的 100 倍，需要在途的请求多出约 100 倍）。问题 1 是**空间上**的错配（数据放在哪个页/bank），问题 2 是**时间上**的错配（请求以什么顺序发出）。两者通过“为了吞吐必须高并发 → 高并发冲掉页缓冲、加剧 bank 冲突”耦合在一起。

对照 OCP 规范后的补充：对图 ANN 来说，稀缺的不是带宽，而是 **bank 时间（每秒能做多少次 sense）**，因为每次 sense 只用到几百字节。好消息是规范里**页号到 bank 的映射是确定的**（默认 `bank = 页号 mod 16`），所以软件选页号就是选 bank，联合布局和 bank 感知调度在现有硬件上就能实现。

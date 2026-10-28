# HBFSim 读书笔记（arXiv 2609.09800v3, 2026-09）

PDF：`papers/HBFSim_arXiv2609.09800.pdf`　代码：https://github.com/SlugLab/hbfsim/

## 它做了什么
在**真实 GPU** 上运行真实 LLM（vLLM + Qwen3-30B-A3B），通过**改写 PTX**，把落在已注册地址区间里的访存“路由”到 HBF 时序模型（底层复用 MQSim 的 flash 介质模型），在其中注入 HBF 延迟。同时耦合了热模型（3D-ICE）和 retention/refresh 模型。注入延迟与请求延迟的误差在 0.152% 以内。

## 对我们有用的点
1. **Table 1 的关键数字**：HBF tR 约 20 µs，读粒度 64 B，每栈 512 GB，带宽 0.4 到 3.0 TB/s；HBM4 为 10–100 ns，粒度 32 B。
2. **Observation 1**：推理负载绝大部分是读，适合 NAND。→ ANN 索引同样以读为主。
3. **Observation 2**：LLM 执行路径可预测，第 i 层的计算时间（约 130 µs）是读第 i+1 层权重时间（约 20 µs）的 6.5 倍，所以可以 prefetch。**图 ANN 不满足这一条。**
4. **D2：Issue 与 consumption 分离**：延迟应该算在“第一次使用该值的指令”上，而不是 load 发出时。这和 GPU 的 scoreboard 语义一致。→ 我们做图搜索流水线时，同样要把“发请求”和“用结果”拆开。
5. **D6：推测式 prefetcher（MoE）**：根据已服务的访问预测后续页，不需要等 media 的访问比例从 17.97% 提到 89.16%，多读了 16.55%（猜错的 139 次）。→ **这是最接近图 ANN 的类比**：访问顺序在运行时才确定，只能靠推测。
6. 热：在 package 安全范围内，**先触到热上限的是计算 die（86.6 W）**，不是 HBF。持续 384 GB/s 没有触发降频。
7. 局限：论文**没有测 prefetch 的时间收益**，也没有涉及 ANN。

## 可以复用的实验方法
- 用 HBFSim 注册 HBF 地址区间，把图索引（邻接表 + 原始向量）放进去，PQ 码放在 HBM，测端到端 QPS 和延迟。
- 扫描 tR（可选 conservative / nominal / aggressive 三个 profile）。

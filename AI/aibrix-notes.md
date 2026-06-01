## 分布式推理
refer: [使用 vLLM 进行分布式推理](https://blog.vllm.com.cn/2025/02/17/distributed-inference.html)

### 张量并行】
背景：同一个节点上，模型超过单个 GPU 容量
张量并行依赖于两种主要技术：
- 列并行（Column Parallelism）：沿列拆分权重矩阵，并在计算后拼接结果。
- 行并行（Row Parallelism）：沿行拆分矩阵，并在计算后对部分结果求和。
![](../images/AI/aibrix_notes_pic_001.png)
![](../images/AI/aibrix_notes_pic_002.png)
![](../images/AI/aibrix_notes_pic_003.png)


### 流水线并行
背景：模型超过多 GPU 节点的容量
流水线并行 在节点之间对模型进行切片，每个节点处理特定的连续模型层。

### 结合张量并行与流水线并行
作为一般经验法则，可以这样考虑并行的应用：

- 当互连速度较慢时，使用 跨节点的流水线并行 和 节点内的张量并行。
- 如果互连效率高（例如 NVLink, InfiniBand），张量并行可以扩展到跨节点。
- 智能地结合这两种技术可以 减少不必要的通信开销 并最大化 GPU 利用率。


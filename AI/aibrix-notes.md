## 分布式推理
refer: [使用 vLLM 进行分布式推理](https://blog.vllm.com.cn/2025/02/17/distributed-inference.html)

### 张量并行
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

## AiBrix 部署及运行
### NCCL 通过IB通信有问题

![](../images/AI/aibrix_notes_pic_004.jpg)

先禁用rdma，使用tcp：NCCL_IB_DISABLE=1

![](../images/AI/aibrix_notes_pic_005.jpg)

![](../images/AI/aibrix_notes_pic_006.jpg)

### vllm L0 KVCache 显存不足报错
![](../images/AI/aibrix_notes_pic_007.jpg)

**说明：max-model-len 是 vLLM 中控制单个请求最大序列长度（输入 Prompt）的核心参数。它直接决定了显存预分配量、上下文窗口上限以及服务稳定性。**

L0 Cache不够只能通过降低这个取值
![](../images/AI/aibrix_notes_pic_008.jpg)

### 开启KVCache Offload，使用L1 Cache

![](../images/AI/aibrix_notes_pic_009.jpg)

![](../images/AI/aibrix_notes_pic_010.jpg)

### benchmark环境准备
- 下载tokenizer 模型: modelscope download --model deepseek-ai/deepseek-llm-7b-chat --local_dir ../models/deepseek-ai/deepseek-llm-7b-chat
- 下载ShareGPT数据集：modelscope download --dataset gliang1001/ShareGPT_V3_unfiltered_cleaned_split  --local_dir ../datasets/gliang1001/ShareGPT_V3_unfiltered_cleaned_split
- 调整config.yaml配置内容
![](../images/AI/aibrix_notes_pic_011.jpg)
![](../images/AI/aibrix_notes_pic_012.jpg)

### benchmark报错：got Future <Future pending> attached to a different loop
![](../images/AI/aibrix_notes_pic_013.jpg)
***解决：升级python至3.12.11版本可解决***

### benchmark报错：openai.OpenAIError: The api_key client option must be set either by passing api_key to the client or by setting the OPENAI_API_KEY environment variable

解决：这里只检测在不在不检测是否合法，直接 export API_KEY='sk-xxxxxxxxxxxxxxxx' 

### benchmark执行

执行aibrix自带benchmark：python3 benchmark.py --stage all --config config.yaml

![](../images/AI/aibrix_notes_pic_014.jpg)
![](../images/AI/aibrix_notes_pic_015.jpg)

### AiBrix监控问题：servicemonitor 采集不到stormservice拉起的推理服务指标：
原因：stormservice自动创建的headless service缺少ports

解决：自己额外创建clusterip service指向其自动创建的pod后端，然后配置servicemonitor，如下
```
[root@C01-P01-GPU-001.sh.cn hhyan]# cat vllm-1p1d-qwen-coder-7b-instruct-monitor-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: vllm-1p1d-qwen-coder-7b-instruct-metrics
  namespace: default
  labels:
    prometheus-discovery: "true"  # 这里配置label供servicemonitor匹配
spec:
  type: ClusterIP
  selector:
    storm-service-name: vllm-1p1d-qwen-coder-7b-instruct
  ports:       # 这里需要配置ports
    - name: metrics
      port: 8080
      targetPort: 8080
      protocol: TCP

[root@C01-P01-GPU-001.sh.cn hhyan]# cat vllm-monitor.yaml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-1p1d-monitor
  namespace: default
spec:
  selector:
    matchLabels:
      prometheus-discovery: "true"    # 匹配service label
  endpoints:
  - port: metrics
    path: /metrics/     # 这里是因为aibrix-runtime容器对metric的url做了调整，这里url需要新增后缀 /
    interval: 15s      # 抓取间隔
```


### benchmark下finished_reason="stop"的请求为0，所有请求被强制截断，结束原因标记为 length

![](../images/AI/aibrix_notes_pic_016.jpg)

解决： benchmark配置中调大 output_token_limit
![](../images/AI/aibrix_notes_pic_017.jpg)
![](../images/AI/aibrix_notes_pic_018.jpg)

### metrics请求到aibrix-runtime后没有返回，报307
原因：配置 sidecar-injection: true后，aibrix会自动给vllm拉起aibrix-runtime容器来代理接口请求，但是runtime容器对metrics请求做了处理

解决：将请求 /metrics 换成 /metrics/，ServiceMonitor也需要调整path
![](../images/AI/aibrix_notes_pic_019.jpg)
![](../images/AI/aibrix_notes_pic_020.jpg)
![](../images/AI/aibrix_notes_pic_021.jpg)

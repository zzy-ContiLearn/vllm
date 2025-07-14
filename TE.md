# Transfer Engine Batch下发优化及LLM Datadist性能比对

## 背景
在前期与客户联合开发的Mooncake connector技术方案中，我们对其进行了详细的技术解析，但测试发现transfer engine（以下简称TE）的端到端传输时延未达预期。在完成630里程碑后，我们对TE中的ascend transport的工作机制进行了优化，最终在客户生产环境中实现了传输性能提升**70%**。在实验环境下，相较于LLM Datadist（以下简称LLM DD）有**30%-100%**的性能优势。

> **组件关系说明**  
> Mooncake connector是vllm-ascend.distributed框架中负责KV cache调度和传输的核心组件，其功能对标nixl_connector及基于LLM DD开发的connector。TE作为Mooncake社区开发的传输组件，与LLM DD的KvCacheManager功能对标。其中，TE通过ascend_transport.hccl_transport模块实现与NPU卡的传输对接。  

> **相关技术资料**：  
> - [Mooncake connector技术复盘](https://jx.huawei.com/redirect/ZJsOes)  
> - [Mooncake connector PR](https://github.com/vllm-project/vllm-ascend/pull/1568)  
> - [TransferEngine.ascend_transport PR](https://github.com/kvcache-ai/Mooncake/pull/502)  

**TE与LLM DD时延与带宽性能对比**  
（测试环境采用客户生产配置：KV block尺寸为128K/16K，单机跨HCCS环场景）  

以下是整理后的专业性能对比表格，采用分层结构清晰展示关键指标：

### TE vs LLM Datadist 性能对比表

#### 1. 跨机传输性能
| **指标**       | **Block Size** | **20 blocks**       | **200 blocks**      | **2000 blocks**     |
|----------------|----------------|---------------------|---------------------|---------------------|
| **时延(ms)**   | 16K            | <span style="color:rgb(233,30,77)">TE: 0.195 (↓36.7%)</span>  | TE: 1.112 (↓27.5%)  | TE: 41.12 (↓22.4%)  |
|                |                | DD: 0.308           | DD: 1.533           | DD: 53.01           |
|                | 128K           | <span style="color:rgb(233,30,77)">TE: 0.194 (↓40.3%)</span>  | TE: 1.205 (↓21.5%)  | TE: 39.03 (↓27.3%)  |
|                |                | DD: 0.325           | DD: 1.534           | DD: 53.69           |
| **带宽(GB/s)** | 16K            | <span style="color:rgb(233,30,77)">TE: 1.68 (↑58.5%)</span>   | TE: 2.95 (↑37.9%)   | TE: 0.8 (↑29.0%)    |
|                |                | DD: 1.06            | DD: 2.14            | DD: 0.62            |
|                | 128K           | <span style="color:rgb(233,30,77)">TE: 13.51 (↑67.4%)</span>  | TE: 21.75 (↑27.3%)  | TE: 6.72 (↑37.7%)   |
|                |                | DD: 8.07            | DD: 17.09           | DD: 4.88            |

#### 2. 单机跨环性能
| **指标**       | **Block Size** | **20 blocks**       | **200 blocks**      | **2000 blocks**     |
|----------------|----------------|---------------------|---------------------|---------------------|
| **时延(ms)**   | 16K            | <span style="color:rgb(233,30,77)">TE: 0.135 (↓53.3%)</span>  | TE: 1.158 (↓12.1%)  | <span style="color:rgb(233,30,77)">TE: 33.46 (↓35.9%)</span>  |
|                |                | DD: 0.289           | DD: 1.318           | DD: 52.23           |
|                | 128K           | <span style="color:rgb(233,30,77)">TE: 0.177 (↓40.2%)</span>  | TE: 1.227 (↓15.8%)  | TE: 39.53 (↓23.7%)  |
|                |                | DD: 0.296           | DD: 1.458           | DD: 51.81           |
| **带宽(GB/s)** | 16K            | <span style="color:rgb(233,30,77)">TE: 2.43 (↑115.0%)</span>  | TE: 2.83 (↑13.7%)   | <span style="color:rgb(233,30,77)">TE: 0.98 (↑55.6%)</span>   |
|                |                | DD: 1.13            | DD: 2.49            | DD: 0.63            |
|                | 128K           | <span style="color:rgb(233,30,77)">TE: 14.81 (↑67.3%)</span>  | TE: 21.36 (↑18.9%)  | TE: 6.63 (↑31.0%)   |
|                |                | DD: 8.85            | DD: 17.97           | DD: 5.06            |

一、核心性能优势总览
在16K/128K两种典型KV block尺寸下，TE相较LLM Datadist呈现显著优势：
平均时延降低：跨机场景36.7%（最高53.29%），单机跨环场景30.2%（最高53.29%）
平均带宽提升：跨机场景43.1%（最高67.41%），单机跨环场景43.6%（最高115.04%）

二、性能拐点分析
16K blocks临界点：
当block数>200时出现带宽悬崖（2.95→0.8GB/s）
128K blocks临界点：
当block数>200时出现带宽悬崖（21.75→6.72GB/s）

根本原因：HCCl底层设置了sq深度512，超过会强制睡眠1ms，并且没有即时清理sq。
工程建议：建议业务层将传输任务控制在512 blocks以内，以获得最佳吞吐。

**TE优化前后E2E性能对比**  
以下是转换后的专业性能对比表格，包含格式优化和关键指标突出显示：

| 上下文长度 | 非批处理 (unbatch) | 批处理 (batch) | 时延下降幅度 |
|------------|--------------------|----------------|--------------|
| **4K**     | 329.764 ms         | 97.629 ms      | <span style="color:rgb(233,30,77)">↓70.39% </span>     |
| **16K**    | 1134.373 ms        | 389.278 ms     | <span style="color:rgb(233,30,77)">↓74.50%</span>      |

## 技术分析
社区的技术方案采用LLM_Datadist_connector+LLM DD组合，而我们设计的Mooncake_connector+TE组合具有以下显著优势：

1. **传输性能提升**  
   - 生产环境测试表明，在低频次传输场景下，时延降低40%-100%，带宽利用率显著提高。

2. **全栈开源方案**  
   - 代码已完整合入vllm-ascend和Mooncake开源社区，客户可基于此进行二次开发和深度优化。

3. **去ranktable依赖**  
   - 相较于LLM DD需要预生成全局ranktable的复杂配置（在大EP部署中易出错），新方案仅需在proxy层配置P/D节点的IP+Port即可完成拓扑发现。

4. **高可用架构**  
   - 采用lazy connect机制：仅在请求携带kv_transfer_params时触发建链。相比LLM DD的全节点预建链模式，在节点故障时仅需proxy层隔离异常节点，无需重启整个服务实例（320+卡的大EP场景下SLA影响显著降低）。

> LLM DD技术文档：  
> https://www.hiascend.com:6066/document/detail/zh/canncommercial/800/apiref/llmdatadist/llmpr_42_0014.html  

**方案选型建议**  
虽然LLM DD功能更为全面，但TE在传输轻量化设计上更具优势。需注意TE的lazy connect特性建议通过预热测试提前完成建链，以规避首次请求的建链开销。

**为什么要做传输性能优化？**
其实，从vllm推理框架的角度出发，传输时延在1.5K输出的场景下，是会被稀释到每个输出token的tpot中的。怎么理解呢？就是说假如150ms的kv cache传输时延平均才会增加0.1ms的tpot。这么看起来收益是不大的。但是从用户视角来看这个问题，PD分离架构下，kv cache传输完成之后才能开始进行decode推理，那么这个传输时延其实是被放到ttft去感知的，虽然这个时间不会加到ttft中。如果传输时延很大，最直观的感受就是，用户要等很久才能吐出首token，那么这也是不能接受的。

## TE优化：小包传输性能提升
在630版本E2E测试中，发现小包传输场景下带宽利用率不足的问题。经分析：

- NPU 200Gbps网卡的理论带宽上限为25GB/s
- 实际测试显示需8MB连续内存块才能达到理论带宽
- 但Page Attention机制导致KV blocks地址不连续（典型场景为128K/16K blocks）

### Batch下发优化实现
针对昇腾NPU的传输特性，我们重构了传输流程：

1. **原流程问题**  
   每个传输任务需串行执行：  
   `TransferSyncRead → AddOpFence → aclrtSynchronizeStream`  
   其中后两步为同步操作，导致异步传输退化为同步模式。

2. **优化方案**  
   将目标端相同的传输任务批量提交，所有任务完成后统一执行同步操作。
![](./images/deepseek_mermaid_20250714_4e1eb3.png)

3. **优化效果**  
   通过减少同步操作次数，端到端传输时延降低70%+，充分释放NPU网络带宽潜力。
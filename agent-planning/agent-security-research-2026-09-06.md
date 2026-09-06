# Agent 安全：核心主线、痛点、技术思路与硬件关联

核验日期：2026-09-06。研究范围：保护 agent 系统及其访问的数据、工具和外部系统；不讨论用 agent 发起网络攻击或自动化安全运营。未运行代码审计、复现攻击或验证硬件平台。下列分类是综合归纳，不是统一行业标准。前两部分独立于硬件选择。

## 一、行业、主流产品与学术研究的核心主线

### 1. 意图与控制流完整性

问题：agent 必须读取外部网页、文件、工具返回和消息；攻击者可让其中的数据被当作指令，改变目标、诱导工具调用或外传数据。通常不需要突破模型的有害内容拒答，也不需要逃出沙箱。

技术：抗注入训练、内容检测和动作审查；控制与数据分离、隔离处理不可信文本、来源标签、信息流策略和动作前强制检查。前一类降低概率，后一类在明确定义的策略范围内限制后果。两类互补。

证据：CaMeL 提出能力与信息流约束；Microsoft FIDES 把完整性、机密性标签和动作前检查集成进 Agent Framework，目前是 experimental。它们的保证依赖正确标注、策略和执行覆盖，不意味着任何任务都能无损安全执行。[CaMeL](https://arxiv.org/abs/2503.18813v2)、[FIDES](https://devblogs.microsoft.com/agent-framework/fides/)

### 2. 身份、任务授权与委派

问题：拿到用户 token 只能说明具有技术访问能力，不能证明每一步符合本次委托。读文档、改权限、发布内容的授权语义不同；子 agent 不应自然继承父 agent 的全部权限。

技术：独立工作负载身份、短期且限定 audience/scope 的凭证、按任务/资源/动作约束的 capability、委派权限收缩、运行时 reference monitor、撤权与审计。自然语言授权到可执行策略仍有歧义，需要适当的人机确认。

证据：Authenticated Delegation 是架构观点；MCP 官方明确讨论 confused deputy、token passthrough、会话劫持等；FORGE 将历史相关策略从模型推理中分离，形式保证依赖观测与策略契约。[Authenticated Delegation](https://proceedings.mlr.press/v267/south25a.html)、[MCP security](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)、[FORGE v3](https://arxiv.org/abs/2602.16708v3)

### 3. 执行隔离、秘密与影响范围

问题：生成代码、shell 和依赖安装会接触文件、进程、网络及凭证；若控制平面与不可信执行混在一起，错误可能扩展到宿主、其他租户或外部系统。循环、资源耗尽与重试副作用也属于运行边界。

技术：进程或 VM 隔离、最小挂载、出网代理、凭证留在执行环境之外、资源/时间预算、控制面与数据面分离、幂等与停止机制。沙箱内已授权的错误操作依然可能发生。

证据：Claude 产品采用多种隔离模式。Codex 明确区分 sandbox 和 approvals：前者是执行边界，后者决定何时审批；自动审查不替代边界。[Claude containment](https://www.anthropic.com/engineering/how-we-contain-claude)、[Codex sandbox](https://learn.chatgpt.com/docs/sandboxing)

### 4. Memory/context 的完整性与保密性

问题：恶意内容或错误经验被写入后，可能经摘要和检索影响后续任务；来源、租户或有效期丢失后，错误影响跨会话持续。

技术方向：写入与读取分别控制信任；记录来源、适用范围、证据与版本；租户隔离、过期与冲突处理；追踪派生记忆并撤销。不能仅删除最初一条记录就宣称影响消失。

证据：MINJA 研究通过交互诱发记忆污染；MemSecBench 将写入、执行与遗忘放进评估生命周期。后者是近期研究，不能直接将论文攻击率套用到 OpenViking。[MINJA](https://arxiv.org/abs/2503.03704v5)、[MemSecBench](https://arxiv.org/abs/2607.27080)

### 5. 工具、插件和 skill 供应链

问题：风险不仅是插件代码；工具描述、skill 文本、配置、升级以及返回内容也会影响模型决策。签名只能帮助确认来源和完整性，不能证明内容或行为正确。

技术方向：发布身份与版本锁定、依赖和能力清单、元数据变更检查、安装权限与运行权限分离、工具隔离、最小秘密、调用前授权及结果来源标记。

证据：MCPTox 测试工具元数据投毒对其他正常工具调用的影响；它是攻击评估，不是成熟防御证明。[MCPTox](https://arxiv.org/abs/2508.14925)

### 6. 多 agent 信任与传播控制

问题：子 agent 转述可能把外部内容包装成内部可信建议；共享记忆和消息还可能传播泄漏、权限扩张及错误。合法身份不等于可信内容。

技术方向：认证消息来源，保留原始数据血缘；结构化任务与交接协议；独立执行权限、委派收缩、共享状态写入控制；限制通信范围和预算；关键行动独立验收。

证据：Agentic Networks Firewalls 研究输入、数据和轨迹控制，但实验局限于相应测试床。Bounded Agents 是近期关于委派范围及预算约束的预印本，仍需跨场景验证。[Firewalls](https://arxiv.org/abs/2502.01822)、[Bounded Agents](https://arxiv.org/abs/2608.15888)

### 7. 安全评估、可观测性与事件处置

问题：阻止所有动作就能得到低攻击成功率，却没有任务价值。静态字符串测试还会遗漏多步、持久化、自适应攻击和真实外部后果。上线后需要知道哪些 agent、插件和凭证正在使用。

技术方向：同时测干净任务与受攻击任务的完成率、攻击目标达成、误阻断、成本和延迟；固定版本、工具与攻击者能力；多次测试；事件和因果日志；关联凭证撤销、记忆纠错与外部操作补偿。

证据：AgentDojo 是动态攻击与防御评估环境；Architecting Secure AI Agents 讨论动态任务与确定性约束间的挑战，属于观点性研究。[AgentDojo](https://arxiv.org/abs/2406.13352)、[Architecting Secure AI Agents](https://arxiv.org/abs/2603.30016)

## 二、最主要的痛点及主流系统现状

以下排序是工程重要性判断，不是行业事故频率统计：

1. **合法能力被用于非授权目的**：用户同意使用工具，不代表工具的任意参数、收件人、时机和目的都被同意。
2. **来源和策略跨步骤丢失**：摘要、记忆、重试和委派之后，单步合法仍可能组合成泄漏或越权。
3. **安全与效用难同时保持**：一律拒绝不可信输入影响行为会阻碍真实工作；无限放权又破坏边界。
4. **配置与执行边界不一致**：名为 sandbox/permission 的不同实现约束不同，有些只约束文件写入。
5. **发现之后难清除影响**：污染已经形成新记忆、消息和外部动作时，日志可读不等于能恢复正确状态。

| 产品/框架 | 已核对的公开机制 | 必须保留的边界 |
|---|---|---|
| Claude Code / Claude 产品 | 文件与网络隔离、权限和按产品选择的环境边界 | 各产品与部署不同，允许范围内仍须语义授权和数据控制 |
| Codex | 平台原生 sandbox + 独立审批策略 | 审批与技术强制是两层，配置可改变实际边界 |
| DeepSeek Harness | 插件化 process-sandbox 与按调用传递的文件策略 | 当前文档明确 SandboxMode 不包括网络和进程可见性；后端可能报告 partial |
| DeerFlow | 用户/线程鉴权隔离、可选执行环境及部署加固 | local sandbox 文档明确不是安全 shell 隔离；部署与 provider 选择很关键 |
| Microsoft Agent Framework | FIDES 信息流标签、隔离处理及动作前策略 | 实验功能，需要显式接入和正确策略；不能据此声称产品全面安全 |

DSH 边界见 [官方 sandbox 文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/sandbox.md)。DeerFlow 边界见 [官方配置](https://github.com/bytedance/deer-flow/blob/main/backend/docs/CONFIGURATION.md)与[API 鉴权](https://github.com/bytedance/deer-flow/blob/main/backend/docs/API.md)。这不是漏洞审计或产品安全排名。

## 三、哪些与 GPU、AMD、Arm 硬件强相关

GPU 是设备类别，AMD 是厂商，Arm 是架构/IP 体系；不是三个互斥路线。AMD EPYC 的 CPU 安全机制不能直接等同于 AMD Instinct GPU 的能力；本轮没有核验 Instinct 的具体机密计算支持矩阵。

首先区分方向：

- 防 agent 伤害外部系统：权限、工具和执行隔离为主。
- 防不可信基础设施窥探/篡改 agent 的数据：机密计算、可信启动、远程证明和按证明释放密钥为主。

| 硬件方向 | 与 agent 安全的关系 | 不提供的保证 |
|---|---|---|
| CPU 虚拟化、内存与 I/O 隔离 | 为 microVM、设备直通和多租户执行提供基础；与 sandbox 强相关 | 不判断任务授权和提示注入 |
| AMD EPYC SEV-SNP | 保护 confidential VM 中运行状态，限制来自边界外高权限软件的访问和篡改 | 不阻止 VM 内合法执行的错误/恶意应用；不自动涵盖远端模型和 GPU |
| Arm CCA / RME Realms | 为受支持平台提供面向宿主外部威胁的机密执行边界 | 不是每个 Arm CPU 都支持，IP 支持不等于某云实例可用 |
| Arm TrustZone | 端侧安全服务、密钥与信任根基础 | 不能直接当作多租户 CCA Realm |
| IOMMU / Arm SMMU | 限制设备 DMA 可访问的内存范围，支撑 GPU/网卡直通隔离 | 不等于链路加密或完整端到端证明 |
| 支持机密计算的 GPU + attestation | 保护敏感推理/训练的数据和模型，核验设备/固件安全状态 | GPU 证明不证明整个 agent 应用符合用户意图 |

一手依据：[AMD SEV](https://www.amd.com/en/developer/sev.html)、[Arm CCA](https://www.arm.com/architecture/security-features/arm-confidential-compute-architecture)、[Arm DMA isolation](https://documentation-service.arm.com/static/643539c53257952fca830c83)、[NVIDIA Attestation](https://docs.nvidia.com/attestation/index.html)。

可用性边界：NVIDIA 有具体 confidential containers 支持平台及拓扑要求；要按型号、驱动、固件、CPU TEE 和分配方式核验，不能将 GPU 资源分区一概当作机密计算。Arm Neoverse V3 有 CCA 架构支持，但本轮未核验具体商业实例可用承诺。[NVIDIA 支持矩阵](https://docs.nvidia.com/datacenter/cloud-native/confidential-containers/latest/supported-platforms.html)、[Neoverse V3](https://www.arm.com/products/silicon-ip-cpu/neoverse/neoverse-v3)

### 关联强弱的规划判断

- **强相关**：多租户执行隔离、机密 CPU/GPU 执行、远程证明与密钥释放、设备直通及 DMA 边界。
- **条件相关**：端侧密钥、敏感记忆存储与检索；前提是确实需要抵御不可信宿主或服务运营方。TEE 仍需应用层租户和读取策略。
- **直接关联弱**：任务授权语义、提示注入识别、经验有效性、工具描述投毒、委派权限设计。用 GPU 训练/运行防护模型是算力需求，不等于其安全机制依赖特殊 GPU。

关键结论：机密环境里运行的 agent 仍可能把有权读取的数据，通过被允许的工具发给错误对象。因此机密计算与行为控制互补，不能互相替代。

## 四、对团队的候选研究切口

尚未批准的推断：

1. 贯通 OpenViking—harness—工具的信息来源与执行策略：保证压缩、检索、委派后信任标签仍有效。
2. 面向真实任务的最小授权与委派：把用户委托转成可执行、可撤销、可审计的操作范围。
3. 与 sandbox 积累结合的执行隔离及可恢复边界；只有存在不信任算力方的实际需求时，再深入 CPU/GPU 机密执行。
4. 统一攻击—正常任务—恢复的评估：同时验证防护收益、效用代价和污染清除。

下一轮应明确保护对象与攻击者能力；此轮先完成问题地图，不以硬件优势倒推威胁模型。

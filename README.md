# 🤖 私人医生助手 — AI 体检报告解读系统

> **作品集项目** | 完整展示 AI 产品经理的"从规划到评估"能力

---

## 项目简介

基于 **Qwen2.5-14B LoRA 微调 + Qwen2.5-VL-7B 多模态微调** 的 AI 私人医生助手。

用户上传体检报告图片 → VL 模型识别异常指标 → LLM 生成个性化健康建议。

## 技术栈

| 技术 | 用途 |
|------|------|
| **Qwen2.5-14B** | 基座 LLM 模型 |
| **Qwen2.5-VL-7B** | 多模态（视觉-语言）模型，识别报告图片 |
| **Unsloth** | LoRA 高效微调框架 |
| **TRL (SFTTrainer)** | 监督微调训练 |
| **vLLM** | 模型部署推理引擎 |
| **FastAPI** | 后端 API 服务 |
| **EvalScope** | 模型评测（BLEU / ROUGE） |

## 文档导航

| 文件 | 内容 | 展示的 PM 能力 |
|------|------|---------------|
| [planner.md](./planner.md) | 技术规划：模型选型、LoRA 参数、微调策略、部署方案 | 技术选型判断力 |
| [prd.md](./prd.md) | 产品需求：市场分析、竞品分析、MVP 定义、评估指标、商业化 | 产品定义 + ROI 测算 |

## 核心 AI PM 决策要点

```
为什么选 LoRA 微调而非 RAG？
  → 医疗场景的知识在模型能力里（需要"学会"医学语言），
    不在外部文档里（不是"查知识库"的问题），
    所以 LoRA 更合适。

为什么选 Qwen2.5-14B 而非 GPT-4o？
  → 成本（API） vs 隐私（数据不出境）
  → 14B 量化后 1 张卡能跑，ROI 可控

怎么评估效果？
  → 原模型 vs 微调模型跑同一份 256 条测试集
  → 对比 BLEU / ROUGE / 人工评分
```

---

*这是 AI 产品经理作品集项目之一。更多项目：*
- [企业 RAG 知识库客服](https://github.com/Ruidee/enterprise-kb-cs)
- [办公流程自动化 Agent](https://github.com/Ruidee/office-automation-agent)

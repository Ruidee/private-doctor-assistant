# 1. 项目目标（一句话总结）

基于 Qwen2.5-14B + Qwen2.5-VL-7B 的 LoRA 微调方案，构建一个能通过医学报告图片识别异常指标并提供个性化健康建议的 AI 私人医生助手 Web 应用。

# 2. 需求拆解（P0/P1/P2 核心功能模块）

## P0 — MVP 必须完成
| 模块 | 说明 |
|------|------|
| 医学报告图片上传 | 用户上传JPG/PNG医学报告图片，前端预览+Base64传输 |
| 异常指标自动识别 | Qwen2.5-VL-7B 多模态模型识别报告中的异常指标 |
| 健康建议生成 | Qwen2.5-14B 基于异常指标生成饮食/运动/生活方式建议 |
| 前端聊天式UI | 类聊天界面，显示用户上传图片 + 模型分析结果 + 健康建议 |
| 后端FastAPI服务 | 接收图片 → 调用VL模型 → 调用LLM → 返回结构化响应 |

## P1 — 延后但重要
| 模块 | 说明 |
|------|------|
| 用户登录/历史记录 | 保存每次问诊记录，支持历史回溯 |
| 语音输入 | 用户可通过语音描述症状 |
| 报告PDF导出 | 将分析结果导出为PDF |
| 多轮对话 | 用户可针对结果追问，模型保持上下文 |
| 模型评测看板 | 可视化展示BLEU/ROUGE等评测指标 |

## P2 — 远期规划
| 模块 | 说明 |
|------|------|
| 多语言支持 | 英文/日文等多语言报告和回答 |
| 用药提醒/日程管理 | 基于诊断结果自动生成用药提醒 |
| 可穿戴设备数据接入 | 接入Apple Health/华为健康数据辅助诊断 |
| 医生端协作平台 | 真实医生可审核AI建议并补充 |
| 联邦学习 | 跨机构数据协作训练，不泄露原始数据 |

# 3. 技术选型（前端、后端、数据库、部署方式、云服务）

## 前端
| 技术 | 选择 | 原因 |
|------|------|------|
| HTML5 + CSS3 + Vanilla JS | 静态页面 | MVP追求零依赖、快速部署 |
| Flexbox布局 | 响应式 | 适配手机和PC |

## 后端
| 技术 | 选择 | 原因 |
|------|------|------|
| FastAPI (Python 3.12+) | 异步框架 | 原生异步支持httpx调用下游模型 |
| uvicorn | ASGI服务器 | FastAPI标配 |
| httpx | HTTP客户端 | async/await调用vLLM推理端点 |
| Pillow | 图片处理 | 缩放图片到1280px最长边 |

## 模型层
| 技术 | 选择 | 原因 |
|------|------|------|
| Qwen2.5-14B | 文本LLM | 中文医疗能力强，14B参数量在单卡上可行 |
| Qwen2.5-VL-7B | 多模态VL | 支持图像+文本输入，适合医学报告OCR+理解 |
| Unsloth | LoRA微调框架 | 相比PEFT快2x，显存省50% |
| TRL (SFTTrainer) | 监督微调 | HuggingFace官方，与Unsloth兼容 |
| vLLM | 推理引擎 | PagedAttention优化，高吞吐低延迟 |
| LoRA (r=16) | 参数高效微调 | 单GPU可训，adapter仅~100MB |

## 数据库
| 技术 | 选择 | 原因 |
|------|------|------|
| SQLite (初期) | 嵌入式 | 零运维，MVP足够 |
| PostgreSQL (后期) | 关系型 | 生产环境成熟方案 |

## 部署方式
| 环境 | 选择 | 原因 |
|------|------|------|
| 本地 | 单机GPU | 开发调试阶段 |
| 云服务器 | 单机GPU (A100 80G) | 生产部署，14B+7B双模型需要~60GB显存 |

# 4. AI/RAG 判断

## 需要 LoRA 微调 — 是
- **原因：** 通用基座模型（Qwen2.5）缺少专业的医学知识深度。LoRA 微调可以用少量医学QA对让模型学会：
  - 专业医学术语的正确使用（如"谷丙转氨酶升高"→"建议保肝治疗"）
  - 回答的结构化格式（先分析→再建议）
  - 避免过度诊断（明确说"未发现异常指标"）
- **为什么不是全量微调：** LoRA 显存开销小（14B模型仅需~16GB额外显存），训练速度快（单epoch 1-2小时），adapter文件小（~100MB）便于版本管理。

## 不需要 RAG
- **原因：** 私人医生助手的知识需求是相对封闭的医学常识 + 体检指标解读，LoRA 微调已经可以将领域知识编码进模型权重。RAG 适合需要检索最新论文/药品说明书的场景，MVP阶段不引入额外的向量数据库、embedding模型和检索管道，降低系统复杂度。

## 不需要向量数据库
- **原因：** 同 RAG 判断。没有 PDF/文档库需要搜索。医学报告分析是端到端的图像→文本理解任务，不需要外部知识检索。

## 不需要多Agent
- **原因：** 当前架构已经形成简单而有效的两阶段管道：VL识别 → LLM建议。引入多Agent（如 Planner → Doctor → Reviewer）会增加延迟和失败概率。后续版本如果需要复杂诊断推理再考虑。

# 5. 为什么这样选（开发速度、成本、可维护性、扩展性、招聘市场通用性）

| 维度 | 评估 |
|------|------|
| **开发速度** | Unsloth + TRL 让微调代码不到200行；FastAPI + 静态HTML 3天可完成前后端联调；vLLM一行命令部署。总开发时间约4-6周 |
| **成本** | 微调用一张A100 80G够用（14B+7B叠加~60G显存）；训练1个epoch约2小时，算力成本可控；推理部署单卡可运行 |
| **可维护性** | 代码结构清晰：训练脚本独立/推理部署独立/Web服务独立；LoRA adapter版本化轻量管理 |
| **扩展性** | 可水平扩展：vLLM支持多卡tensor-parallel；FastAPI无状态易水平扩展；LoRA支持热插拔切换不同领域的adapter |
| **招聘市场通用性** | Qwen2.5生态（阿里系）、Unsloth/TRL/vLLM 是目前LLM落地最主流的工具链，团队成员经验可复用 |

# 6. 项目结构（目录树）

```
personal-health-assistant/
├── README.md                          # 项目说明
├── day1/                              # 第一阶段：数据准备 + 微调
│   ├── 4-med-prepare-dataset/         # 数据集准备
│   │   ├── question.csv               # 问题CSV
│   │   ├── answer.csv                 # 答案CSV
│   │   ├── prepare_dataset.py         # CSV→对话JSON转换（train/valid/test分割）
│   │   ├── med-dataset-train.json     # 训练集（对话格式）
│   │   ├── med-dataset-valid.json     # 验证集
│   │   ├── med-dataset-test.json      # 测试集
│   │   ├── convert_to_evalscope_format.py  # JSON→JSONL评测格式转换
│   │   └── qa/                        # evalscope评测数据目录
│   │       └── med.jsonl              # 评测用的问答对
│   ├── 5-med-eval/                    # 模型评测
│   │   ├── eval-original.py           # 评测原始模型（基线）
│   │   └── eval-finetuned.py          # 评测微调后模型
│   ├── 6-med-fine-tuning-llm/        # LLM微调（Qwen2.5-14B）
│   │   ├── train-qwen25-14b.py        # Unsloth+TRL LoRA微调主脚本
│   │   ├── train-oss.py               # 备用：Ollama微调方案
│   │   └── train-medgemma15.py        # 备用：MedGemma微调方案
│   └── 7-med-fine-tuning-vl/         # VL微调（Qwen2.5-VL-7B）
│       ├── train.py                   # Unsloth+TRL VL LoRA微调主脚本
│       ├── validation.py              # VL模型推理验证脚本
│       ├── dataset-img-train/         # 医学报告图片训练集
│       ├── test-img/                  # 测试图片
│       └── pyproject.toml             # 项目依赖
├── day2/                              # 第二阶段：部署 + Web服务
│   └── med-service/                   # 医疗助手后端服务
│       ├── pyproject.toml             # Python依赖（vllm==0.11.2等）
│       ├── uv.lock                    # 锁定依赖版本
│       ├── .python-version            # Python版本
│       ├── .gitignore
│       ├── 1-launch-vllm-llm.sh       # 启动LLM vLLM服务（端口8001）
│       ├── 2-launch-vllm-vl.sh        # 启动VL vLLM服务（端口9002）
│       ├── 3-validate-llm.sh          # 验证LLM服务响应
│       ├── 4-validate-vl.sh           # 验证VL服务响应
│       ├── 5-launch-fastapi-v1.sh ~ 8-launch-fastapi-v4.sh  # FastAPI版本迭代
│       ├── fastapi_v1_file.py ~ fastapi_v4_ui.py  # FastAPI各版本实现
│       ├── validate_vl.py             # VL模型手动验证脚本
│       ├── ui/                        # 前端UI（v4版本）
│       │   ├── index.html             # 主页面（聊天风格界面）
│       │   ├── style.css              # 样式（移动端适配）
│       │   └── script.js              # 前端逻辑（图片上传/XHR/渲染）
│       └── static/                    # 静态文件目录（v3版本）
├── qwen25-14b/                        # Qwen2.5-14B基座模型权重
├── qwen25-vl-7b/                      # Qwen2.5-VL-7B基座模型权重
├── qwen25-14b-fine-tuned-lora/        # LLM LoRA adapter权重
├── qwen25-vl-7b-finetuned/            # VL微调后合并权重
└── screenshot*.png                    # 应用截图
```

# 7. 开发里程碑（Week1~Week8 拆解）

## Week 1-2：数据准备与预处理
- [ ] 收集/清洗医学问答数据集（CSV格式，question_id匹配）
- [ ] 运行 prepare_dataset.py 生成 train/valid/test JSON
- [ ] 收集医学报告图片并标注异常指标文本描述
- [ ] 运行 convert_to_evalscope_format.py 生成 JSONL 评测数据
- [ ] 数据质量检查：去重、格式统一、中文标点规范化

## Week 3-4：模型微调
- [ ] LLM微调：运行 train-qwen25-14b.py（Unsloth LoRA r=16）
  - 训练参数：batch_size=64, grad_accum=4, lr=2e-4, epoch=1
  - LoRA target: q/k/v/o_proj, gate/up/down_proj
- [ ] VL微调：运行 train.py（Unsloth FastVisionModel LoRA r=16）
  - 训练参数：batch_size=2, grad_accum=4, lr=2e-4, epoch=4
  - 微调视觉+语言层
- [ ] 保存LoRA adapter / 合并模型权重
- [ ] 手动验证：测试几个典型病例

## Week 5：模型评测与迭代
- [ ] 部署vLLM服务，加载LoRA adapter
- [ ] 运行 eval-original.py：原始模型基线评测（BLEU/ROUGE）
- [ ] 运行 eval-finetuned.py：微调后模型评测
- [ ] 对比结果，如果提升<10%则返回Week3调整超参数
- [ ] 人工评估：10-20个样例的人工评分

## Week 6：后端开发
- [ ] 编写FastAPI v4服务（fastapi_v4_ui.py）
  - POST /image 端点：接收图片 → 调VL→调LLM→返回结果
  - 图片缩放（最长边1280px，LANCZOS）
  - 超时处理（httpx timeout=60s）
  - 错误处理（网络错误/模型格式异常）
- [ ] 编写启动脚本串联所有服务
- [ ] 端到端测试：图片上传→VL分析→LLM建议

## Week 7：前端 + 集成
- [ ] 静态HTML聊天式UI开发
  - 图片选择/预览/移除
  - 发送按钮状态管理
  - 加载动画（三点跳动+计时器）
  - 响应渲染（Markdown→HTML转换，分"检测结果分析"和"健康建议"两部分）
  - 错误提示
- [ ] CSS美化（移动端优先适配）
- [ ] 前后端联调，修复跨域/路径问题

## Week 8：部署 + 文档
- [ ] 部署文档：README、启动脚本说明
- [ ] 提供预训练adapter下载链接
- [ ] 最终验收测试：10张医学报告图片的端到端测试
- [ ] 性能测试：并发请求下的延迟和吞吐

# 8. 风险提示（技术债、性能瓶颈、安全问题）

## 技术债
| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 前端无框架 | Vanilla JS 在功能复杂后可维护性差 | P1阶段评估迁移到React/Vue |
| 无单元测试 | MVP阶段为赶速度跳过 | P1阶段补充pytest和前端jest |
| 硬编码配置 | 模型路径/端口/提示词硬编码 | 升级到.env + config.py |
| 单文件服务 | fastapi_v4_ui.py 超过200行逻辑混在一起 | P1阶段拆分为router/service层 |

## 性能瓶颈
| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 双模型串行调用 | VL→LLM串行，总延迟=VL延迟+LLM延迟 | 考虑并行调用或流式输出 |
| vLLM显存限制 | 双模型同时加载占用~60GB显存 | A100 80G可运行，设置gpu_memory_utilization |
| 大图片处理 | 高分辨率医学影像可能OOM | 强制缩放最长边1280px |
| 请求超时 | httpx timeout 60s，复杂图片可能更久 | 考虑异步任务队列（Celery） |

## 安全问题
| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 患者隐私数据 | 医学报告含敏感个人信息 | 本地部署，不上传第三方；HTTPS传输 |
| 模型幻觉 | AI给出错误医疗建议 → 法律责任 | 前端显示免责声明："AI建议仅供参考" |
| API无认证 | vLLM和FastAPI无鉴权 | 添加API Key认证 + IP白名单 |
| SQL注入 | 后期引入数据库后的注入风险 | 使用ORM（SQLAlchemy） |

# 9. CI/CD设计

## 开发阶段（手动）
```
本地开发 → git commit → git push (main/dev分支)
```

## 自动化CI（GitHub Actions）
```yaml
name: CI Pipeline
on: [push, pull_request]
jobs:
  lint:
    - ruff check .           # Python代码风格检查
  train-test:
    - python prepare_dataset.py --test-only   # 数据集格式验证
    - python -c "import torch; assert torch.cuda.is_available()"  # GPU可用性
  build:
    - docker build -t med-assistant .   # Docker镜像构建
```

## CD流程
```yaml
name: CD Pipeline
on:
  push:
    tags: [ 'v*' ]
jobs:
  deploy:
    - docker push med-assistant:latest
    - ssh deploy@server "docker-compose pull && docker-compose up -d"
```

## 持续训练流水线
```
数据更新 → 触发训练Job → LoRA adapter产出 → 自动评测 → 达标则推送新adapter到生产
```

# 10. 部署方案

## 本地开发部署（单机GPU）
```bash
# 1. 启动LLM vLLM服务（端口8001）
bash day2/med-service/1-launch-vllm-llm.sh

# 2. 启动VL vLLM服务（端口9002）
bash day2/med-service/2-launch-vllm-vl.sh

# 3. 启动FastAPI Web服务（端口8000）
cd day2/med-service && uvicorn fastapi_v4_ui:app --host 0.0.0.0 --port 8000
```

## 生产部署方案
```
                    ┌──────────────┐
                    │  Nginx反向代理 │  ← HTTPS/443
                    │  (SSL终止)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌────▼────┐ ┌────▼────┐
        │FastAPI   │ │vLLM-LLM │ │vLLM-VL  │
        │(3副本)   │ │(8001)   │ │(9002)   │
        │(8000)    │ │         │ │         │
        └──────────┘ └─────────┘ └─────────┘
              │
        ┌─────▼──────┐
        │ PostgreSQL │  ← 会话/历史记录
        └────────────┘
```

## Docker Compose（可选）
```yaml
services:
  vllm-llm:
    image: vllm/vllm-openai:latest
    command: serve /models/qwen25-14b --enable-lora --lora-modules mylora=/models/lora --port 8001
    volumes:
      - /path/to/models:/models
    deploy:
      resources:
        reservations:
          devices: ["driver=nvidia,count=1"]
  vllm-vl:
    image: vllm/vllm-openai:latest
    command: serve /models/qwen25-vl-7b-finetuned --port 9002
    volumes:
      - /path/to/models:/models
    deploy:
      resources:
        reservations:
          devices: ["driver=nvidia,count=1"]
  fastapi:
    build: ./med-service
    ports: ["8000:8000"]
    depends_on: [vllm-llm, vllm-vl]
```

# 11. 项目初始化命令

```bash
# 克隆项目
git clone <repo-url> personal-health-assistant
cd personal-health-assistant

# 下载基座模型（已有则跳过）
# Qwen2.5-14B: https://huggingface.co/Qwen/Qwen2.5-14B
# Qwen2.5-VL-7B: https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct

# 安装Python依赖
pip install unsloth trl transformers datasets torch evaluate
pip install vllm==0.11.2
pip install fastapi uvicorn httpx pillow

# 数据准备
cd day1/4-med-prepare-dataset
python prepare_dataset.py           # CSV → JSON
python convert_to_evalscope_format.py med-dataset-test.json qa/med.jsonl   # JSON → JSONL

# 模型微调
cd ../6-med-fine-tuning-llm
python train-qwen25-14b.py          # LLM LoRA微调（约1-2小时）

cd ../7-med-fine-tuning-vl
python train.py                     # VL LoRA微调（约30分钟-1小时）

# 模型评测（需先启动vLLM服务）
cd ../5-med-eval
python eval-original.py             # 原始模型基线
python eval-finetuned.py            # 微调后模型

# 启动Web服务（3个终端窗口）
# 终端1: vLLM LLM
cd ../../day2/med-service
bash 1-launch-vllm-llm.sh

# 终端2: vLLM VL
bash 2-launch-vllm-vl.sh

# 终端3: FastAPI Web
uvicorn fastapi_v4_ui:app --host 0.0.0.0 --port 8000

# 浏览器打开 http://localhost:8000
```

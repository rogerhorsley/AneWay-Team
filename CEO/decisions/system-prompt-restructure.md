# System Prompt 重构方案

> 日期: 2026-03-01
> 作者: AneWay 团队（RD + Designer + PM）
> 状态: 待评审

---

## 1. 问题诊断

### 现状
- `system_prompt.md`: **867 行 / 53KB**
- 每次对话全量加载，占用 **15-20K tokens**
- 24 个 blueprint 示例全量注入

### 影响
| 问题 | 影响 | 严重度 |
|------|------|--------|
| Context 占用过高 | 留给对话的空间被压缩 | 🔴 高 |
| Model Loop 风险 | Attention 被无关内容稀释 | 🔴 高 |
| 响应延迟 | 处理大 context 耗时更长 | 🟡 中 |
| 成本增加 | 按 token 计费，每次多花钱 | 🟡 中 |

---

## 2. 重构方案

### 2.1 目录结构

```
prompts/
├── core.md                    # 必需，常驻 (<10KB)
│   ├── 0.1 语言规则
│   ├── 0.2 语气规则
│   └── 0.3 输出格式
│
├── skills/                    # 按需加载
│   ├── product-requirement-analysis.md
│   ├── product-page-architecture.md
│   └── app-ui-ux-best-practices.md
│
├── references/                # 仅在明确需要时注入
│   ├── anti-patterns.md
│   ├── visual-tokens.md
│   └── component-libraries.md
│
└── blueprints/                # 智能匹配，最多注入 3 个
    ├── index.json             # 元数据索引
    └── examples/
        ├── ecommerce-apple.md
        ├── social-instagram.md
        └── ...
```

### 2.2 加载策略

| 内容 | 加载时机 | 预估大小 |
|------|----------|----------|
| core.md | 每次对话开始 | < 10KB |
| 当前 skill | 激活该 skill 时 | 2-5KB |
| references | skill 明确要求时 | 1-3KB |
| blueprints | embedding 匹配 top 3 | 6-15KB |

**预期总 context**: 20-35KB（vs 原 53KB+）
**节省**: **60-70%**

### 2.3 Blueprint 智能匹配

```
用户输入 → Embedding → 与 24 个 blueprint 计算相似度 → 取 Top 3 → 注入 context
```

实现选项：
- OpenAI Embedding API
- 本地 sentence-transformers
- Claude 自带的语义理解（简单场景）

---

## 3. 实施步骤

### Phase 1: 拆分 (1-2 天)
- [ ] 从 system_prompt.md 提取 core.md
- [ ] 将 skill 内容移动到独立文件
- [ ] 创建 blueprints/index.json 元数据

### Phase 2: 加载机制 (2-3 天)
- [ ] 实现按需加载逻辑
- [ ] 实现 blueprint 匹配（可先用简单关键词匹配）
- [ ] 测试 context 组装

### Phase 3: 验证 (1-2 天)
- [ ] 对比重构前后的 token 消耗
- [ ] 测试 loop 问题是否改善
- [ ] 用户体验测试

---

## 4. 风险与缓解

| 风险 | 缓解措施 |
|------|----------|
| 拆分后逻辑断裂 | 保留原文件作为参考，逐步迁移 |
| 加载顺序错误 | 单元测试覆盖关键路径 |
| Blueprint 匹配不准 | 先用关键词兜底，再优化 embedding |

---

## 5. 成功指标

- [ ] system_prompt context 占用 < 35KB
- [ ] Model loop 发生率下降 50%+
- [ ] 平均响应时间缩短 20%+

---

*本文档由 AneWay 团队协作完成，待 CEO 评审后执行。*

# 事实核查引擎 · FACT CHECK ENGINE

> 信息真伪核查工具：输入文字或上传截图，自动拆解主张、多源交叉验证，生成结构化核查报告。
>
> 线上地址：<https://wanghahaniu.github.io/factcheck/>
> 技术形态：纯前端单页应用，无后端、无构建步骤。

---

## 一、项目特点

- **零依赖**：HTML + CSS + 原生 JS 全部内联在 `factcheck.html` 中，双击即可运行，也可直接托管到 GitHub Pages。
- **双输入模式**：支持粘贴文字，或上传图片（自动 OCR 提取文字）。
- **六步核查流水线**：内容提取 → 拆解主张 → 搜索策略 → 多源检索 → 交叉比对 → 结构化报告，流程可视化。
- **真实联网检索**：接入 Tavily API 做实时搜索；未配置时回退到 AI 知识库模拟检索。
- **多源交叉验证**：按来源权威等级（官方/主流/专业/社交）加权判定。
- **结构化报告**：主张拆解、证据对照、来源表格、五维评分、最终结论、反向验证建议。
- **导出能力**：一键打印为 PDF（内置打印样式，自动切换浅色主题）、复制摘要。
- **本地历史记录**：最近 20 条核查记录保存在 `localStorage`。

---

## 二、密钥与安全（重要）

应用需要两个 API Key：

| Key | 必填 | 用途 | 获取 |
|-----|:---:|------|------|
| 智谱 GLM | ✅ | 驱动 LLM 核查（文本 + 图片 OCR） | open.bigmodel.cn |
| Tavily | 可选 | 真实联网检索；留空则走 AI 知识库模拟 | app.tavily.com |

**密钥绝不会上传 GitHub。** 隔离机制：

1. 密钥只写在本地 `config.local.js`，该文件被 `.gitignore` 排除。
2. `factcheck.html` 读取顺序：**`config.local.js` → `localStorage` → 侧边栏手输**。
3. 因此：本地打开即用、免输入；线上访问者没有 `config.local.js`，自动走侧边栏输入自己的密钥——应用同样可用。

### 本地首次使用
```bash
cp config.example.js config.local.js   # 复制模板
# 编辑 config.local.js，填入你的智谱 / Tavily 密钥
```
然后双击 `factcheck.html`，侧边栏两栏会自动填好，直接点「开始核查」。

---

## 三、文件结构

```
factcheck/
├── factcheck.html       # 主应用（全部源码：HTML + CSS + JS）
├── index.html           # GitHub Pages 入口（自动跳转到 factcheck.html）
├── config.example.js    # 密钥配置模板（上传，供克隆者参考）
├── config.local.js      # 本地真实密钥（.gitignore 排除，★永不上传）
├── .gitignore           # 排除 config.local.js
└── README.md
```

无 `package.json`、无构建配置、无外部依赖（未引用任何 CDN）。

---

## 四、LLM 接入

- **提供方**：智谱 GLM（OpenAI 兼容格式）
- **接口**：`https://open.bigmodel.cn/api/paas/v4/chat/completions`
- **模型**：`glm-5.2`（文本任务）/ `glm-5v-turbo`（图片 OCR，多模态）
- **参数**：`temperature: 0.1`
- **多模态**：图片以 `data:image/jpeg;base64,...` 形式作为 `image_url` 发送

调用统一走 `callOpenAI()`，根据是否有图片自动切换文本/视觉模型。

---

## 五、核心工作流（六步流水线）

入口函数：`startCheck()`，状态机由 `STEPS` 数组与 `renderSteps()` 驱动。

| 步骤 | id | 说明 |
|:---:|:---|------|
| 1 | `extract` | 提取内容：文字直通；图片走 OCR |
| 2 | `claims` | 拆解为可验证子主张（带分类/优先级） |
| 3 | `queries` | 为每条主张生成中英双语搜索策略 |
| 4 | `search` | 多源证据检索（Tavily 真实 / AI 模拟） |
| 5 | `verify` | 交叉比对，识别支持/冲突/证据不足 |
| 6 | `report` | 汇总生成评分、结论、建议 |

任一步骤失败会抛出带原始返回片段的错误（如 `Step2失败: ... | 原始返回: ...`），便于定位。

---

## 六、评分体系

五维评分，总分 100：

| 维度 | 字段 | 满分 |
|------|------|:---:|
| 来源权威性 | `source_authority` | 30 |
| 多源印证度 | `multi_source` | 25 |
| 时间线一致性 | `timeline` | 15 |
| 实体信息一致性 | `entity` | 15 |
| 关键证据完整性 | `evidence` | 15 |

**结论映射**：

| 总分 | 结论 |
|:---:|------|
| 80–100 | 高可信 — 有充分权威来源支撑 |
| 60–79 | 基本可信 — 建议补充更多证据 |
| 40–59 | 存疑 — 证据不充分或存在冲突 |
| 20–39 | 高度存疑 — 缺乏可信来源支撑 |
| 0–19 | 大概率虚假 — 与权威来源严重冲突 |

主张状态四态：`confirmed`（已确认）/ `doubt`（存疑）/ `insufficient`（证据不足）/ `false`（不实）。

---

## 七、来源权威分级

| Tier | 类型 | 域名示例 |
|:---:|:---|------|
| 1 | 官方权威 | `gov.` `mil.` `xinhua` `people.com` `cctv` `pbc.gov` `ap.org` |
| 2 | 主流媒体 | `reuters` `bloomberg` `bbc` `nytimes` `wsj` `ft.com` `caixin` `thepaper` |
| 3 | 专业媒体 | 其他 |

---

## 八、本地运行与部署

### 本地运行
双击 `factcheck.html` 即可。或：
```bash
python -m http.server 8000
# 访问 http://localhost:8000/factcheck.html
```

### 部署到 GitHub Pages
1. 仓库 → Settings → Pages
2. Source 选 `main` 分支、根目录
3. 保存后访问 `https://wanghahaniu.github.io/factcheck/`（`index.html` 会自动跳转到 `factcheck.html`）

---

## 九、已知限制

- 历史记录仅保存标题/评分/结论，完整报告无法复现。
- 模拟检索（无 Tavily 时）依赖模型知识库，**不是实时数据**，报告会标注「⚠ AI知识库模拟（非实时）」。
- 主张拆解固定为 2 条，长文本可能覆盖不全。

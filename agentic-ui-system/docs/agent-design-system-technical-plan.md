# Agent 设计组件系统技术方案

> 状态：方案草案（尚未开始实现）  
> 范围：React 组件库及其文档、示例、发布体系；不包含 Agent 后端、模型调用服务或 IDE 宿主实现。  
> 产品范围参见 [组件分类大纲](./agent-design-system-outline.md)。

## 1. 目标与非目标

### 目标

- 提供可用于 Web 与 Electron/IDE WebView 的 React + TypeScript 组件库，覆盖通用基础组件、Agent 专属组件和工作台组合模式。
- 将「选择上下文 → 输入任务 → 查看计划 → 观察执行 → 审阅 Diff → 批准或回退」作为 V1 验证闭环。
- 支持亮色/暗色主题、紧凑密度、键盘操作和可访问性；组件 API 稳定，支持按需引入。
- 通过类型定义、交互规范、示例与测试，使不同 Agent 产品接入同一套 UI 时保持一致。

### 非目标

- 不在组件内调用模型、建立 SSE/WebSocket 连接、读取本地文件、执行命令或修改 Git 工作区。
- 不在 V1 复制 Ant Design 的全部通用组件；只实现闭环需要的基础能力。
- 不把某个模型厂商、Agent SDK、事件协议或 IDE 桥接 API 固化为公共组件 API。
- 组件展示的「接受/拒绝」是用户意图；实际提交、回滚与权限校验由接入应用负责。

## 2. 技术选型与取舍

| 领域 | V1 选型 | 原因与边界 |
| --- | --- | --- |
| 语言与运行时 | TypeScript、React；以已公布的最低支持版本为 peer dependency | 严格类型与跨宿主复用；初始化时根据目标应用确定 React 版本范围，不捆绑 React 到产物 |
| 包管理 | pnpm workspace | 多包依赖与本地联调；暂不引入额外任务编排器 |
| 构建 | tsup 产出 ESM 与类型声明 | 保持产物简单；是否提供 CJS 由实际消费端决定，V1 不默认承诺双格式 |
| 样式 | CSS Modules + CSS 自定义属性 | 样式隔离、低运行时开销、可动态切主题；消费者显式引入样式入口 |
| 无障碍基础 | Radix UI 的按需 primitives（选用时逐项验证） | 借用复杂控件的焦点管理与键盘语义；对外封装自有组件 API，不泄漏其内部类型 |
| 图标 | SVG React 组件，统一尺寸/描边/语义 | 避免字体图标闪烁；装饰图标默认隐藏于辅助技术 |
| 测试 | Vitest + Testing Library + user-event；Playwright 进行关键交互及视觉回归 | 单元/交互测试负责行为，浏览器测试负责实际焦点、布局和主题 |
| 文档与示例 | 独立 React 文档站 + 可运行的工作台示例 | 文档版本与包版本关联；先实现用法、状态和场景，站点框架可在初始化时确定 |
| 版本发布 | Changesets + npm 包发布流程 | 每个包独立变更记录和语义化版本；发布目标可按组织环境配置 |

**待决策项**：初始化工程前确认目标 React 版本、浏览器/WebView 基线、是否需要 SSR、npm registry 和是否存在必须支持的 CJS 消费端；这些决定会影响构建与兼容性声明。

## 3. 分层架构

```text
业务应用 / Agent 运行时 / IDE 宿主
  └─ 接入适配层：事件归一化、连接与重连、权限执行、文件与 Git 操作
       └─ 场景层：任务工作台、变更审阅流程（示例/可选组合）
            └─ Agent UI：输入、对话、计划、工具调用、授权、产物审阅
                 └─ UI：Button、Input、Dialog、Popover、Tabs 等通用组件
                      └─ Tokens + Icons：视觉变量、图标与主题
```

建议包结构：

```text
packages/
  tokens/       Token 源、主题 CSS 和类型
  icons/        图标组件
  ui/           通用基础组件及样式
  agent-ui/     Agent 领域类型与展示组件
apps/
  docs/         组件文档与示例
  playground/   模拟任务执行的工作台集成示例
```

依赖方向只能从上层指向下层。`agent-ui` 可以依赖 `ui` 和 `tokens`；`ui` 不依赖 Agent 状态；`playground` 可以模拟事件流，但组件包不得反向依赖示例应用。主题能力先随 `tokens` 包交付，只有出现独立主题发布需求时再拆包。

## 4. Token、主题与样式契约

设计变量分三层：

1. **基础 Token**：原始色板、字号、间距、圆角、阴影、动效时长。
2. **语义 Token**：文本、表面、边框、焦点、危险、成功、运行中、选中态等，不用颜色名表达含义。
3. **组件 Token**：仅当组件有稳定且独立的定制需求时引入，例如输入框背景或 Diff 新增行背景；默认复用语义 Token。

使用 `data-theme="light|dark"` 和 `data-density="comfortable|compact"` 的作用域选择器覆盖 CSS 变量，使同页不同区域可预览不同主题；若应用通过系统设置自动切换，由应用决定作用域属性，不在组件内强制读取系统偏好。消费者通过统一样式入口加载默认主题，覆盖语义变量可定制品牌；组件内部 class 名不视为公共 API。

主题至少覆盖正常、悬停、按下、禁用、聚焦、错误、选中，以及 Agent 的等待、运行、需授权、完成、失败、中断。减少动画设置需尊重 `prefers-reduced-motion`；紧凑模式不应缩小可操作区域到难以点击。

## 5. 组件 API 规范

- **优先受控、允许非受控**：输入类组件以 `value`/`onChange` 作为受控接口，需要时提供 `defaultValue`；不在组件内部保存无法同步的业务状态。
- **数据与事件分离**：展示组件接收只读领域数据；点击按钮发出意图回调，等待外部传入新状态，不乐观宣称操作已成功。
- **状态明确**：任务、工具、授权、审阅的状态使用判别式联合类型；不要组合多个可能相互矛盾的布尔值。
- **组合优于巨型配置**：复杂区域提供 slot 或组合式子组件；不将整个工作台设计成单个拥有几十个 Props 的组件。
- **易于扩展**：支持 `className`、`style`、语义化 DOM 属性及必要的 ref；不把底层 Radix API 整体透出。
- **健壮默认值**：空数据、很长的路径与日志、流式局部结果、失败和重试都有明确表现；跨屏幕阅读器状态变化使用合适的播报策略。

### V1 领域模型示意

以下为候选公共接口，须用真实场景和接入方数据验证后再冻结：

```ts
export type RunStatus =
  | 'queued'
  | 'running'
  | 'waiting_for_approval'
  | 'completed'
  | 'failed'
  | 'cancelled';

export interface TaskStep {
  id: string;
  title: string;
  status: RunStatus;
  description?: string;
}

export interface ToolCallView {
  id: string;
  name: string;
  status: RunStatus;
  summary?: string;
  outputPreview?: string; // 仅传入已裁剪、已脱敏的展示文本
  startedAt?: string;
  endedAt?: string;
}

export interface PermissionView {
  id: string;
  action: string;
  scope: string;
  risk?: 'low' | 'medium' | 'high';
  status: 'pending' | 'approved' | 'rejected' | 'expired';
}

export interface FileChangeView {
  id: string;
  path: string;
  changeType: 'added' | 'modified' | 'deleted' | 'renamed';
  reviewStatus: 'unreviewed' | 'accepted' | 'rejected';
}
```

`id` 用于流式更新、列表键与关联操作；跨会话唯一性由接入层保证。时间统一使用 ISO 8601 字符串并由展示层格式化。Diff 的内容格式与大文件加载策略单独设计，不把完整文件内容塞进每条变更摘要。若以后需要协议适配包，先在应用侧验证至少两种来源，再抽象发布。

### 事件适配与权限边界

接入层负责将后端事件合并为组件快照：事件去重/排序、断线重连、部分输出合并、终态去重和错误恢复均不属于展示组件职责。执行中的增量文本应批量刷新，避免每个 token 都触发整棵消息树重渲染。

授权必须由宿主或服务端再次校验：UI 的「允许」回调不等于操作已被授权；外部应处理超时、拒绝、重复提交与运行取消。敏感命令、密钥及环境变量在进入组件前脱敏，日志默认限制长度，必要时提供按需展开/加载。

## 6. V1 组件交付清单

| 批次 | 组件 | 关键状态/验收重点 |
| --- | --- | --- |
| 基础底座 | Button、TextArea、Popover、Dialog、Tooltip、Tabs、Badge、Spinner | 键盘操作、焦点、禁用、主题与密度；按实际组合需求补充基础件 |
| 任务输入 | PromptComposer、ContextChip/ContextPicker、ModelModeSwitcher、ConversationMessage | `@` 引用与附件呈现、空输入/发送中、长消息及 Markdown 安全渲染 |
| 执行可见 | AgentStatus、TaskPlan、ToolCallCard、TerminalOutput | 流式更新、并行调用、失败/取消、长日志与可折叠详情 |
| 人在回路 | PermissionRequest、ReviewActions | 请求范围/风险明确、等待/过期/拒绝、重复点击防护 |
| 产物审阅 | FileChangeList、DiffViewer | 多文件、增删改、空 Diff、大文件、键盘导航与审阅结果 |

V1 的 12 个 Agent 标志性组件与[分类大纲](./agent-design-system-outline.md#三v1-建设范围)对应。`DiffViewer` 可评估引入成熟 diff 渲染能力，但需要关注包体积、虚拟滚动、无障碍与 WebView 兼容性；不以「自行实现完整代码编辑器」为 V1 目标。

## 7. 文档、质量与发布要求

每个组件文档至少包含：使用场景、最小示例、Props/事件、受控示例、所有状态、主题与密度示例、键盘行为、无障碍注意事项、边界条件。模式库另写端到端流程，而非复制组件 API；工作台示例使用模拟运行事件演示完整闭环。

质量门槛：

- 类型检查、构建、单元测试及关键交互测试通过；公共导出入口与类型声明可被独立消费项目导入。
- 浏览器验证亮/暗主题、紧凑模式、长文本、空/加载/失败态、焦点可见性、键盘与屏幕阅读器语义。
- 覆盖授权审批、流式工具结果、Diff 审阅这三条关键流程；视觉回归针对代表性状态建立基线。
- 包发布前在至少一个独立消费应用中验证样式引入、按需导出、React 单实例、包体积和 WebView 兼容性。
- 破坏性 Props/Token/样式入口变更走主版本；新增兼容功能走次版本；修复走补丁版本。公开组件与 Token 应有迁移说明。

## 8. 实施阶段与完成标准

| 阶段 | 交付物 | 退出标准 |
| --- | --- | --- |
| P0：约束确认 | React/宿主基线、主题语义、领域类型草案、V1 场景及状态矩阵 | 与至少一个实际消费应用核对数据和交互边界 |
| P1：工程底座 | workspace、tokens、icons、ui 基础、构建与测试链路 | 明暗/密度可切换；独立应用可安装并使用基础组件 |
| P2：核心闭环 | 12 个 Agent 组件、接入层模拟器、工作台示例 | 从输入到执行、审批与 Diff 审阅的完整模拟流程可运行 |
| P3：稳定发布 | 文档、可访问性修复、视觉回归、变更记录与包发布 | 满足第 7 节质量门槛并完成独立消费验证 |

## 9. 主要风险

- **过早追求通用性**：先用实际 Agent 工作流校验领域模型，避免为了兼容所有产品而设计复杂 API。
- **流式事件与 UI 状态混淆**：适配层维护状态快照，组件只读取快照并发出操作意图。
- **底层依赖外泄**：把无障碍 primitives 与 diff 渲染器封装为内部实现，锁定对外语义和类型。
- **大内容性能**：日志、对话与 Diff 使用截断、分段渲染或按需加载；必要时再引入虚拟化，并确保焦点不会丢失。
- **错误的安全感**：风险提示不能代替宿主授权；所有实际操作仍需权限校验和可追溯记录。

# Agent Note: Kliping 产品品牌，且不含标记

Status: implemented

[English](2026-08-24-kliping-product-brand.md) | 中文

## 问题

已发布的产品在侧边栏品牌行呈现 DeepSeek 的鲸鱼图形与 `DSH Local Build` 标签，在空会话 hero 呈现同一鲸鱼与 `Into the Unknown`，在浏览器标题与安装清单中呈现 `DeepSeek Harness`，并在三句告诉模型自己运行在何处的系统提示词中呈现 `DeepSeek Harness`。因此，以自有名称发布的部署会把上游品牌泄漏到人和模型能看到的每一处；承载这些图形的不只是可选的官方品牌包，还包括外壳自身的 fallback。

## 决策

发布品牌就是产品名 `Kliping` 的文本形式，任何位置都不含标记。

侧边栏品牌行渲染产品名与 7 位构建修订号；mark slot 保持已声明且空置。收起轨道只保留面板切换控件，否则以标记作为静息状态的轨道会渲染出一个不可见的按钮。空会话 hero 在 `Preview` 徽标旁渲染同一产品名，其标题改为弹性行而非固定网格轨道——固定轨道会为无人占用的标记预留一列和一段间距。

两处界面都不再用自有盒子包裹 mark slot。Slot 渲染器把占位者置于一个 `display: contents` 元素之下，该元素在 slot 空置时完全不生成盒子，因此弹性行只是少了一个条目；而外壳自有的包裹层会成为零宽度的弹性条目，并仍然占用该行的间距。因此占位者自己拥有其盒子，并接收所请求的尺寸。

[`client-ui-brand-official`](../../../../packages/client/ui-brand-official/README.zh.md) 现在只在 `official` 构建 profile 下填充 `sidebar.brand.name`，因此官方构建显示不带构建修订号的产品名，其他构建则保留携带该修订号的外壳 fallback。它对会话包与基础组件包的依赖随所服务的占位者一并撤除，`FishLogo` 与 `BrandWordmark` 也从 [`client-ui-primitives`](../../../../packages/client/ui-primitives/README.zh.md) 中删除，Web 客户端完全不再发布品牌图形。文档站保留自己的字标，因为它发布的是上游文档语料而非产品。

浏览器标题、PWA 清单名称、官方构建 profile 的 `DSH_CLIENT_TITLE`，以及 favicon（一个中性字母花押字，仍遵守已被测试的明暗主题规则）都使用同一名称。三句模型可见的身份语句——[system-prompt](../../../../packages/core/system-prompt/README.zh.md) 开场白、app-boot 的检出位置语句、web-app 的 GUI 语句——均改称 Kliping，所有录制快照随之更新。GUI 欢迎通知改称 Kliping，并提升版本号，使已确认过的通知再显示一次。

包名、`dsh` 命令、`DSH_*` 变量、`$DSH_HOME`、`@deepseek-ai` npm 作用域，以及 DeepSeek 模型提供方名称均刻意保持不变：前四者是技术标识符，最后一个是真实的 API 供应商名称。

## 备选方案

**保留鲸鱼，只改标签。** 已否决，因为本次诉求是品牌替换，而标记恰是人最先辨认的那一半；保留它会让文本改动读起来像笔误而非改名。

**彻底删除 `client-ui-brand-official`。** 它与外壳 fallback 的剩余差异只有一个构建修订徽标，看似可以删除。已否决，因为该包是声明感知品牌 slot 注册的完整范例，而删除它需要改动 web-app bundle、两份客户端目录、模块图以及每个 tsconfig 聚合，只换来外观收益。

**在同一次改动中重命名 npm 作用域与 `dsh` 命令。** 已否决，视为独立的机械式工程：作用域涉及数千个文件、lockfile、tsconfig 引用与生成目录，而预发布立场本就允许日后自由重命名——届时改名本身就是整次改动，而非搭在品牌改动上的顺风车。

**不动模型可见的身份语句。** 已否决，因为系统提示词在每次请求中都向模型声明产品名；止步于像素的改名会让智能体继续以旧名自我介绍。

## 后果

外壳现在完全不再发布品牌图形，因此需要标记的部署应占用 `sidebar.brand.mark` 或 `conversation.hero.brand.mark`，而不是替换某个 fallback。这两个 slot 正是为此保持声明。

收起轨道随标记一并失去了标记与面板图标的悬停切换，相关 CSS 规则也已移除；轨道在静息与悬停状态下都只是一个普通切换控件。

三十份录制快照随源字符串一同变更。它们是同步手工编辑而非重新录制，因为重新录制需要提供方密钥；回放会把重新生成的输出与它们比对，因此两侧一起变更。

欢迎通知版本号提升意味着每位已确认过通知的用户会再看到一次。

## 测试

单元覆盖随行为迁移：侧边栏用例断言产品名、构建修订号，以及品牌行不含 `svg`；hero 用例断言产品名，以及 mark slot 在渲染时不带 fallback；brand-official 用例断言只有一个占位者且仅渲染文本。`pnpm run typecheck`、`pnpm run verify-translation-pairing` 与受影响的包测试套件均通过。浏览器端到端与快照回放套件需要完整的 `pnpm run build`，留给 CI 执行。

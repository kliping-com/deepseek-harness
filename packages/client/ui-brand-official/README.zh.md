# @deepseek-ai/dsh-client-ui-brand-official

[English](README.md) | 中文

仅当 `DSH_CLIENT_BUILD_PROFILE` 为 `official` 时，本包才填充 `sidebar.brand.name`。其他构建仍会加载插件，但不注册 occupant，因此显示 shell fallback：同样的产品名，后面跟着构建期 7 位 `DSH_CLIENT_COMMIT_HASH` 徽标，本地构建因此不会被误认为发布版本。

该占位者以文本形式呈现产品名。发布品牌不含任何标记与图形，`sidebar.brand.mark` 与 `conversation.hero.brand.mark` 保持空置，留给需要标记的部署包。注册通过 `slots.inject()` 安装，因此无论该包的条目先于还是后于侧边栏声明方激活，它都能工作；声明折叠时会撤回该占位者，HMR 期间不会留下混合品牌。它不保留运行时状态。node 半边是空的 Loader seat；浏览器标题仍属于本包之外的构建环境事项。

## 模型体验

无，因为本包只贡献浏览器呈现；这里没有任何内容进入模型请求。

#### KV Cache 影响

无；本包既不组装也不发送 provider 请求。

## 已知限制与暂缓事项

- **本包只提供一个 occupant** —— 其他呈现应由占用相同 slot 的另一个 Cordis 包提供。
- **浏览器标题相互独立** —— `DSH_CLIENT_TITLE` 在构建期选择标题文字，而不经过 UI slot。

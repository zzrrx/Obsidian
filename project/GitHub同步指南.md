# GitHub同步Obsidian vault完整流程

## 1. 生成GitHub Token
- 访问 https://github.com/settings/tokens/new
- 输入名称，选择无过期
- 勾选 **repo** 权限
- 点击 **Generate token**
- 复制token（以ghp_开头）

## 2. 安装Obsidian插件
- 设置 → 社区插件
- 搜索 **GitHub Octokit Sync**
- 安装并启用

## 3. 配置插件
- 设置 → GitHub Octokit Sync
- 粘贴token → Connect
- 选择或创建仓库

## 4. 同步
- 点击侧边栏GitHub图标
- 点击 **Sync now**
- 第一次同步完成

## 5. 注意事项
- **代理问题**：如果使用VPN/代理（如Clash），同步时需要关闭代理或设置GitHub直连
- 代理地址示例：127.0.0.1:7890
- 关闭代理后重启Obsidian再同步

**现在你的vault已备份到GitHub，可以随时恢复。**
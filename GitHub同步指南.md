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

**现在你的vault已备份到GitHub，可以随时恢复。**
# HTML Anything Windows 部署与排障笔记

这份笔记记录在 Windows 11 上安装、修复并测试 `nexu-io/html-anything` 的过程，方便下次迁移到另一台电脑时复用。

## 本次环境

- 系统：Windows
- Shell：PowerShell
- Node.js：v24.14.1
- npm：11.12.1
- pnpm：11.1.2
- 本地目录：`C:\Users\YYYYsen\html-anything`
- 访问地址：`http://localhost:3000`

## 安装步骤

```powershell
cd C:\Users\YYYYsen
git clone https://github.com/nexu-io/html-anything.git
cd html-anything
npm install -g pnpm
pnpm install
pnpm approve-builds --all
pnpm build
pnpm dev
```

打开浏览器访问：

```text
http://localhost:3000
```

## 本次修复

### 1. ConvertChip Hook 顺序崩溃

现象：

```text
Rendered fewer hooks than expected
```

原因：

`src/components/convert-chip.tsx` 在调用 `useCallback` / `useEffect` 之前，根据 `layoutMode` 提前 `return null`。当用户切换布局时，组件的 hook 调用数量变化，React 会直接报错。

修复：

把 `layoutMode !== "split"` 的返回逻辑移动到所有 hooks 之后，并在快捷键逻辑里显式判断 `isSplitMode`。

### 2. Zustand Selector 无限更新

现象：

```text
The result of getSnapshot should be cached to avoid an infinite loop
Maximum update depth exceeded
```

原因：

`src/components/deploy-control.tsx` 的 selector 在没有部署记录时返回 `[]`，每次 render 都创建一个新数组。React 19 / Zustand 会认为 snapshot 一直变化，从而触发无限更新。

修复：

增加模块级常量：

```ts
const EMPTY_DEPLOYMENTS: DeploymentRecord[] = [];
```

并让 selector 返回这个稳定引用。

### 3. `"default"` 模型不应传给 CLI

现象：

部分 Agent 的默认模型应由 CLI 自己决定，但代码可能把 `"default"` 当成真实模型传成 `--model default`。

修复：

在 `src/lib/agents/argv.ts` 中统一过滤：

```ts
const model = _opts.model && _opts.model !== "default" ? _opts.model : undefined;
```

## Agent 测试结论

本机检测到的主要 Agent：

- Claude Code：可用
- OpenAI Codex：能启动，但生成会卡住
- Qwen Coder：命令行测试超时
- Kiro CLI：检测到，但项目标记为 unsupported

### Claude Code 正常

命令行测试：

```powershell
$p = "只输出一行：OK"
$p | claude -p --output-format stream-json --verbose --include-partial-messages --permission-mode bypassPermissions
```

结果：正常返回 stream-json。

页面测试：选择 `Claude Code`，输入软件清单，点击 `Generate HTML`，成功生成中文长文页面。

### Codex 卡住

命令行测试：

```powershell
$p = "只输出一行：OK"
$p | codex exec --json --skip-git-repo-check --sandbox workspace-write -c sandbox_workspace_write.network_access=true
```

结果：只输出：

```text
Reading prompt from stdin...
```

之后长时间无结果。页面里调用 Codex 也是同样表现，所以问题不在网页，而在当前机器的 Codex CLI 运行状态或配置。

建议：

- 当前机器优先使用 `Claude Code`
- 如果必须使用 Codex，先单独在命令行确认 `codex exec` 能快速返回
- 不要只看 `/api/agents` 显示 available；available 只代表二进制存在，不代表 CLI 生成链路正常

## 端到端测试内容

本次用以下内容测试生成：

```text
1. OneCommander 文件资源管理器
2. XMouseButtonControl 鼠标按键模拟
3. AutoHotkey v2  脚本工具
4. HideVolumeOSD-1.4   隐藏 Windows 11音量弹窗
5. MicYou 电脑虚拟麦克风
6. AudioRelay 虚拟麦克风、扬声器
7. Obsidian 笔记
8. 坚果云 云服务
9. bandicam 录屏
10. typora markdown文件编辑器
11. Deskpin 窗户置顶
12. trae 编辑器
13. 小丸工具箱 视频压缩
14. Typeless 输入法
15. snipaste 截屏工具
16. 万达云 AppConnect 代理工具
17. Codex  AI 编程智能体
18. AOMEI Partition Assistant  傲梅恢复助手  磁盘管理
19. CC Switch    API管理
20. WindTerm  终端客户端
21. OpenCode  开源代码编辑器
22. Directory Opus   文件资源管理器
23. Microsoft PowerToys   Windows神器！！
24. SumatraPDF   非常轻量化的 PDF 阅读器
25.
```

成功结果：

- Agent：Claude Code
- 模板：Blog Post
- 首字节：约 4.9 秒
- 输出大小：约 21 KB
- 页面标题：`我的 Windows 工具箱：24 款日常必备软件`

## 下次部署到新电脑的检查清单

1. 确认 Node / npm / pnpm：

```powershell
node --version
npm --version
pnpm --version
```

2. 安装依赖并批准构建脚本：

```powershell
pnpm install
pnpm approve-builds --all
pnpm build
```

3. 单独测试 Agent CLI：

```powershell
"只输出 OK" | claude -p --output-format stream-json --verbose --include-partial-messages --permission-mode bypassPermissions
"只输出 OK" | codex exec --json --skip-git-repo-check --sandbox workspace-write -c sandbox_workspace_write.network_access=true
```

4. 启动服务：

```powershell
pnpm dev
```

5. 打开浏览器：

```text
http://localhost:3000
```

6. 如果页面崩溃，先看浏览器控制台和 `dev-server.err.log`。

7. 如果页面能打开但生成一直卡住，优先单独测试对应 CLI，不要先改网页代码。

## 常用维护命令

```powershell
cd C:\Users\YYYYsen\html-anything
pnpm build
pnpm dev
git status --short
```

后台启动开发服务：

```powershell
$pnpm = (Get-Command pnpm.cmd).Source
Start-Process -FilePath $pnpm -ArgumentList @('dev') -WorkingDirectory (Get-Location) -WindowStyle Hidden
```

查看 3000 端口是否可访问：

```powershell
Invoke-WebRequest -Uri http://localhost:3000 -UseBasicParsing
```


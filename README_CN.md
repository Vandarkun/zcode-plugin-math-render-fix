# zcode-plugin-math-render-fix

[English](./README.md) · 简体中文

ZCode 插件：让聊天界面的数学公式正常显示。包含两个 Hook、两条命令，以及一个可选的渲染层补丁。

## 问题

ZCode 用 KaTeX 渲染聊天里的 Markdown，只识别美元符分界符。模型常把公式写成 `\( ... \)` 或 `\[ ... \]`，反斜杠被 Markdown 转义吃掉，界面显示成原始 LaTeX：`\(P_{\mathrm{total}}\)` 变成 `(P_{\mathrm{total}})`。

## 仓库内容

| 部分 | 作用 | 是否改动 ZCode 安装文件 |
| --- | --- | --- |
| 插件：两个 Hook | 会话开始时告诉模型用 `$$` 写法；回复结束时检查，发现旧写法就让模型只重发这些公式 | 否 |
| 插件：命令 `/mathfix` | 把上一条回答里的公式用可渲染的写法重新输出 | 否 |
| 可选的渲染层补丁，命令 `/mathfix-patch` | 让 ZCode 自己认得 `\( ... \)` 和 `\[ ... \]`，旧消息里的公式也能显示 | 是，可以还原 |

两层互相独立，单靠任何一层都能让公式可读。补丁生效时，“让模型重发”的检查会自动沉默。

## 安装

### 作为插件市场添加

1. ZCode → **插件市场** → **添加** → **添加插件市场**，粘贴本仓库地址：

   ```
   https://github.com/Vandarkun/zcode-plugin-math-render-fix
   ```

2. 在 **个人 → vandarkun-plugins** 里找到「公式渲染修复」，点击安装。

安装并启用后新建会话即可生效。输入 `/` 可以看到 `/mathfix` 和 `/mathfix-patch`。

Hook 需要 PATH 上有 `node`。它们不联网，也不写入任何文件；除了 ZCode 传给它们的内容，Stop Hook 只会读取补丁留下的状态文件，用来判断补丁是否仍然生效。

### 渲染层补丁（可选）

不装补丁，公式也能读：模型会被引导用 `$$` 写法，漏掉的会被要求重发。想让 `\( ... \)` 写法直接显示，再打补丁。

**补丁会修改 ZCode 安装目录里的 `resources/app.asar`。** 它原地替换渲染层两个函数头部的 127 个字节，替换内容与原文等长，所以文件大小不变，可以用 `restore` 逐字节还原。

克隆本仓库后运行：

```shell
node plugins/math-render-fix/scripts/patch-renderer.mjs status    # 只读：ZCode 版本、目标文件、当前状态
node plugins/math-render-fix/scripts/patch-renderer.mjs apply     # 写入；之后需完全退出并重启 ZCode
node plugins/math-render-fix/scripts/patch-renderer.mjs restore   # 还原
```

- 脚本只在两个目标函数都恰好各找到一处、并且改后的文件通过语法检查时才写入，其余情况一律拒绝。
- 完全退出并重启 ZCode 后才生效。
- **ZCode 更新会覆盖补丁。** 更新前建议先 `restore`，更新后再 `apply`。更新后 Stop Hook 会自动恢复工作，所以公式仍然可读，只是会多重发一次。
- 这是本地修改，不是 ZCode 官方支持的用法，是否使用请自行决定。

## 原理

**根因。** ZCode 的 Markdown 解析器只把 `$...$` 和 `$$` 识别为公式。`\(` 在 Markdown 里表示转义的左括号，所以反斜杠被吃掉，剩下的内容按普通文字显示。KaTeX、样式和字体 ZCode 其实都有，只是入口不认这两种分界符。

**插件这一层。** `SessionStart` Hook 在每个会话开始时注入一条 682 字节的规则。`Stop` Hook 在每次回复结束时做纯文本检查：跳过代码块和行内代码，找出成对的 `\( ... \)` 和 `\[ ... \]`，用几条简单规则判断是否像公式。命中时返回 `decision: "block"` 并列出修正后的写法，模型只重发这些公式。每轮最多触发一次，不会循环。

**补丁这一层。** ZCode 在解析每条消息之前，自己会先对美元符做一次消歧处理。补丁改写这段处理里两个函数的头部，让 `\(`、`\)`、`\[`、`\]` 在解析前先被换成 `$$`，原有的解析器就能认出来。

更多细节见[插件目录里的说明](./plugins/math-render-fix/README_CN.md)。

## 兼容性与验证情况

- **插件**：已在 ZCode 4.0.0 上实测（Windows 10、Node 22）。两个 Hook 在真实会话中执行过：SessionStart 注入了规则；Stop 在回答含旧写法公式时拦截一次，模型续写后本轮正常结束。macOS 和 Linux 没有测试。
- **补丁**：写于 ZCode 4.0.0（build 3bc8a247），只在 Windows 10 上实测。带补丁的 ZCode 能正常启动，旧写法的公式能正常显示。其他版本上脚本会报告不适用，不会写入。
- **还原**：只在 `app.asar` 的副本上验证过能逐字节还原，没有在真实安装目录上执行过。

## 测试

```shell
node plugins/math-render-fix/tests/run.mjs                            # Hook 与补丁脚本的回归测试
node plugins/math-render-fix/tests/verify-patch.mjs [app.asar 路径]    # 补丁的端到端检查，只读
```

## 与官方市场版本的关系

插件部分（两个 Hook 和 `/mathfix`，不含补丁）已提交给官方插件市场，等待审核：[zai-org/zcode-plugins#222](https://github.com/zai-org/zcode-plugins/pull/222)。官方市场里的版本不会改动 ZCode 的安装文件。

渲染问题的根因在 ZCode 的渲染器里，可以到 [zai-org/feedback](https://github.com/zai-org/feedback) 向官方反馈。

## 许可证

[MIT](./LICENSE)

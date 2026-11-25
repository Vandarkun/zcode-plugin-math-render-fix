# zcode-plugin-math-render-fix

English · [简体中文](./README_CN.md)

A ZCode plugin that keeps math formulas readable in chat. It ships two hooks, two commands, and an optional renderer patch.

## The problem

ZCode renders chat Markdown with KaTeX and only recognizes dollar delimiters. Models often write formulas as `\( ... \)` or `\[ ... \]`. Markdown escaping eats the backslashes, and the chat shows raw LaTeX: `\(P_{\mathrm{total}}\)` becomes `(P_{\mathrm{total}})`.

## What is in this repository

| Part | What it does | Changes ZCode's installation? |
| --- | --- | --- |
| Plugin: two hooks | At session start, tells the model to use `$$`. When a reply ends, checks it and has the model re-output only the formulas written another way | No |
| Plugin: command `/mathfix` | Re-outputs the formulas of the previous answer in a form the renderer understands | No |
| Optional renderer patch, command `/mathfix-patch` | Makes ZCode itself display `\( ... \)` and `\[ ... \]`, including in older messages | Yes, reversible |

The two layers are independent, and either one alone keeps formulas readable. While the patch is in place, the check that has the model re-output stays silent.

## Install

### Add this repository as a plugin marketplace

1. In ZCode, open **Plugin Marketplace → Add → Add Plugin Marketplace** and paste the repository URL:

   ```
   https://github.com/Vandarkun/zcode-plugin-math-render-fix
   ```

2. Under **Personal → vandarkun-plugins**, install **Math Render Fix**.

Enable it and open a new session. Type `/` to find `/mathfix` and `/mathfix-patch`.

The hooks need `node` on `PATH`. They make no network request and write no files. Besides what ZCode passes to them, the Stop hook only reads the state file the patch leaves behind, to tell whether the patch is still in effect.

### The renderer patch (optional)

Formulas are readable without the patch: the model is steered to `$$`, and anything written another way is re-output. Apply the patch if you want `\( ... \)` to display directly.

**The patch modifies `resources/app.asar` in the ZCode installation.** It replaces 127 bytes in the heads of two renderer functions in place, with text of exactly the same length, so the file size is unchanged and `restore` puts it back byte for byte.

Clone this repository and run:

```shell
node plugins/math-render-fix/scripts/patch-renderer.mjs status    # read-only: ZCode version, target file, current state
node plugins/math-render-fix/scripts/patch-renderer.mjs apply     # writes; quit and restart ZCode afterwards
node plugins/math-render-fix/scripts/patch-renderer.mjs restore   # reverts
```

- The script writes only when both target functions are found exactly once and the patched file still parses. It refuses everything else.
- It takes effect after ZCode is completely restarted.
- **A ZCode update overwrites the patch.** Running `restore` before updating and `apply` afterwards is the safe routine. After an update the Stop hook is active again on its own, so formulas stay readable, at the cost of one extra round.
- This is a local modification and not an officially supported practice. Decide for yourself whether to use it.

## How it works

**Root cause.** ZCode's Markdown parser recognizes only `$...$` and `$$` as math. In Markdown, `\(` is an escaped left parenthesis, so the backslash is dropped and the rest is shown as plain text. ZCode does ship KaTeX, its styles and fonts; the parser just does not accept these two delimiters.

**The plugin.** The `SessionStart` hook injects a 682-byte rule at the start of every session. The `Stop` hook runs a plain-text check at the end of each reply: it skips code blocks and code spans, finds `\( ... \)` and `\[ ... \]` pairs, and applies a few simple rules to decide whether they look like formulas. On a match it answers `decision: "block"` with the corrected forms, and the model re-outputs only those formulas. It fires at most once per turn and never loops.

**The patch.** Before parsing each message, ZCode runs a pass that disambiguates dollar signs. The patch rewrites the heads of two functions in that pass so that `\(`, `\)`, `\[` and `\]` are turned into `$$` first, which the existing parser understands.

See the [plugin's own README](./plugins/math-render-fix/README.md) for details.

## Compatibility and what has been verified

- **Plugin**: verified on ZCode 4.0.0 (Windows 10, Node 22). Both hooks ran in a real session: SessionStart injected the rule; Stop blocked once on a reply with an old-style formula, the model continued, and the turn then ended normally. Not tested on macOS or Linux.
- **Patch**: written for ZCode 4.0.0 (build 3bc8a247) and verified on Windows 10 only. ZCode starts normally with the patch, and old-style formulas display. On other builds the script reports that it does not apply and writes nothing.
- **Restore**: verified byte for byte on a copy of `app.asar`; it has not been run on a real installation.

## Testing

```shell
node plugins/math-render-fix/tests/run.mjs                            # regression tests for the hooks and the patch script
node plugins/math-render-fix/tests/verify-patch.mjs [path to app.asar]   # end-to-end check of the patch, read-only
```

## Relation to the official marketplace

The plugin part (the two hooks and `/mathfix`, without the patch) has been submitted to the official plugin marketplace and is awaiting review: [zai-org/zcode-plugins#222](https://github.com/zai-org/zcode-plugins/pull/222). The official version does not modify ZCode's installation.

The root cause is in ZCode's own renderer. Reports can be filed with [zai-org/feedback](https://github.com/zai-org/feedback).

## License

[MIT](./LICENSE)

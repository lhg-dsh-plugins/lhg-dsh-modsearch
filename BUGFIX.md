# BUGFIX — redactSecrets 漏脱敏 `Authorization: Basic` 凭证

## 问题

`src/util/redact.ts` 的 `redactSecrets` 负责在错误信息、缓存文件与模型上下文中剥离凭证。其 Auth 头形状模式只认「头名后紧跟 token」与「Bearer <token>」两种形式；当凭证以 HTTP Basic 形式出现时，scheme 词 `Basic` 夹在头名与 base64 凭证之间，模式失配，整段凭证原样穿过脱敏，进入错误信息。

## 证据链（修复前 → 修复后）

- 修复前复现：`redactSecrets('Authorization: Basic dXNlcjpwYXNzd29yZA==')` 返回原文不变（其中 `dXNlcjpwYXNzd29yZA==` 是 `user:password` 的 base64）。新增回归测试在未修复代码上运行：`src/util/redact.test.ts` 1 failed / 18 passed，失败点即此断言。
- 根因：TOKEN_SHAPES 中的 Auth 头正则 `/\b(?:bearer|authorization)\b[=:\s]+"?[A-Za-z0-9._~+/-]{12,}"?/` 要求分隔符后立即出现 ≥12 字符的 token 字符类；`Basic` 只有 5 个字符且后随空格，匹配失败，全文又无其他模式兜底（URL 模式不覆盖无 scheme 的头值）。
- 修复后：同一输入输出为 `[redacted]`；`Proxy-Authorization: Basic ...` 同样被覆盖（模式在 `Proxy-` 后同样命中 `authorization`）。

## 改动文件

- `src/util/redact.ts`：Auth 头正则允许 scheme 词 `basic` 出现在头名与 token 之间（`(?:basic\s+)?`），并以 `=*` 吞掉 base64 的填充尾，避免留下 `==` 残片。仅此一行模式与注释。
- `src/util/redact.test.ts`：新增 2 条回归测试（Basic / Proxy-Authorization 脱敏；scheme 后短词不误伤）。

## 复验结果

- `vitest run`（全量）：547 passed / 1 skipped / 0 failed（跳过 1 条为 root 平台条件跳过，与基线一致）。
- `tsc --noEmit`：退出码 0。
- `vite build`：成功（dist/main.js 133.64 kB）。

## 验证边界

- 本次验证为离线单元与全量测试，未调用任何真实搜索/抓取引擎，不消耗配额。
- 脱敏仍是「形状识别」策略：`Authorization: Digest` 的参数串（如 response 哈希）不在本次修复范围内；无标签的裸密钥（如 AWS secret）无法靠形状识别，不在此模块能力边界内。

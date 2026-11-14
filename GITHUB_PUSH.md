# 推送到 GitHub 说明（本机 / DSH 沙箱环境）

本文记录在本机（Windows + DSH workspace-write 沙箱）向 `https://github.com/yuquant/memovr.git` 推送时
实际踩到的三个坑，以及验证过的可用命令。日常在普通终端里 `git push` 正常时不需要看本文。

远端约定：`origin` = `https://github.com/yuquant/memovr.git`，发布分支 `main`
（GitHub Pages 从 `main` 根目录发布到 https://yuquant.github.io/memovr/ ）。

---

## 一、遇到的障碍

### 1. schannel 报 `SEC_E_NO_CREDENTIALS`

```
fatal: unable to access 'https://github.com/yuquant/memovr.git/':
schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS (0x8009030e) - 安全程序包没有可用的凭证
```

Git for Windows 默认的 TLS 后端是 `schannel`（走 Windows SSPI 凭证库）。在沙箱受限令牌下拿不到凭证，
于是**连匿名读取都会失败**（`git ls-remote` 同样报错，`curl https://github.com` 也是 `curl: (35)` 同样的错）。

关键判断：这不是网络问题。DNS 能解析、443 能连通，只是 TLS 握手拿不到凭证。

**解决**：把 TLS 后端换成 openssl（自带 CA 包，不依赖 SSPI）。

```powershell
git -c http.sslBackend=openssl ls-remote origin      # 能列出 refs 就说明 TLS 已通
```

### 2. 凭证助手（GCM）在沙箱里起不来

```
sh.exe: *** fatal error - couldn't create signal pipe, Win32 error 5
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

`credential.helper=manager` 会拉起 `git-credential-manager.exe`，而 Git 通过 `sh.exe` 创建管道调它；
沙箱禁止创建命名管道 → `Win32 error 5`。所以**不能用凭证助手弹窗登录**，也读不到已存库里的凭证。

**解决**：绕过助手，把 token 直接作为 HTTP 头传给这一次请求。

### 3. 环境里没有 token 变量

`GH_TOKEN` / `GITHUB_TOKEN` 都是空的，`~/.ssh` 也不存在（无 SSH key）。但 **GitHub CLI 已登录**
（`gh auth status` → 账号 `yuquant`，scope 含 `repo`），token 存在系统密钥环里。

**解决**：用 `gh auth token` 现取，进程内传给 git，**不落盘**。

---

## 二、验证过的推送命令

在仓库根目录执行（PowerShell）：

```powershell
$tok = & "C:\Program Files\GitHub CLI\gh.exe" auth token
$b64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("x-access-token:" + $tok.Trim()))
$env:GIT_TERMINAL_PROMPT = "0"

git -c http.sslBackend=openssl -c credential.helper= `
    -c http.extraheader="Authorization: Basic $b64" push origin main
```

预期输出（`...` 为提交号）：

```
To https://github.com/yuquant/memovr.git
   c47bfa8..c3f8ef0  main -> main
```

要点：

- `-c credential.helper=` 是**空值**，用来一次性屏蔽 `credential.helper=manager`；不影响 `.git/config`。
- `http.extraheader` 只对本次命令生效，不会写进配置。
- **不要把 token 拼进 remote URL**（如 `https://user:token@github.com/...`）。它会留在 `.git/config`
  和 reflog 里，一旦推错仓库就泄露了。

## 三、让 TLS 长期可用（可选）

只修 TLS、不碰凭证，后续 `fetch` / `pull` / `ls-remote` 都不用再带 `-c`：

```powershell
git config --local http.sslBackend openssl
```

（本仓库已设置。凭证那一步仍需上面的 `$b64` 方案。）

## 四、推送后验证

```powershell
git -c http.sslBackend=openssl ls-remote origin refs/heads/main   # 远端 main 的 SHA
git rev-parse HEAD                                                # 本地 HEAD 的 SHA
git status -sb                                                    # 应为 ## main...origin/main，无未提交改动
```

两个 SHA 相同、`git status` 没有 `ahead` 字样，才算真正推上去了。

## 五、报错对照表

| 现象 | 原因 | 处理 |
|---|---|---|
| `schannel: ... SEC_E_NO_CREDENTIALS` | TLS 后端拿不到 Windows 凭证 | 加 `-c http.sslBackend=openssl` |
| `couldn't create signal pipe, Win32 error 5` | 沙箱禁止命名管道，凭证助手起不来 | 加 `-c credential.helper=` 并改用 `http.extraheader` |
| `could not read Username for 'https://github.com'` | 没有可用凭证 | 用 `gh auth token` 取 token |
| `remote: Permission to ... denied` | token 无 `repo` scope 或仓库不对 | `gh auth status` 查账号与 scope |
| `failed to push some refs ... non-fast-forward` | 远端有新提交 | 先 `git fetch` + `git pull --rebase` 再推 |
| PowerShell 里 push 成功却显示红字 `NativeCommandError` | git 把进度写到 stderr，PowerShell 当作错误 | 忽略；以 `main -> main` 行和上面的 SHA 验证为准 |

## 六、提交前自查

```powershell
git status --short          # 确认要提交的文件
git diff --stat             # 确认改动范围
git add -A
git commit -m "简短标题" -m "一句话说明改动原因"
```

本次推送包含的提交：`Simplify stereo-align UI: drop manual frame button and flat zoom slider`
（`stereo-align.html` 移除手动「抽取当前帧」按钮、状态行与提示段，移除重复的视图缩放滑杆；
滚轮缩放仍走 `setFlatZoom()`，功能未丢）。

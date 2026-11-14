# 推送到 GitHub 说明（本机 / DSH 沙箱环境）

**现状：已配置 SSH，直接 `git commit` + `git push` 就行，不用 token、不用加参数。**

需要动手的只有三行：

```powershell
git add -A
git commit -m "改动说明"
git push
```

日常维护到这一节就够了。下面是一次性配置都做了什么、以及当初 https 为什么那么折腾（仅供以后排错）。

远端：`origin` = `git@github.com:yuquant/memovr.git`，发布分支 `main`
（GitHub Pages 从 `main` 根目录发布到 https://yuquant.github.io/memovr/ ）。

---

## 一、一次性配置做了什么（2026-11-15）

| # | 动作 | 位置 |
|---|---|---|
| 1 | 生成 ed25519 密钥（无口令） | `%USERPROFILE%\.ssh\id_ed25519` / `.pub` |
| 2 | 预置 GitHub 官方 host key，指纹已核对 | `%USERPROFILE%\.ssh\known_hosts` |
| 3 | 公钥注册为**仓库部署密钥**（可写） | GitHub `repos/yuquant/memovr/keys`，id `169886878`，title `dsh-windows-xiaobai` |
| 4 | `origin` 改 SSH，并指定用系统 OpenSSH | 本仓库 `.git/config` |

第 4 条的具体内容：

```powershell
git remote set-url origin git@github.com:yuquant/memovr.git
git config --local core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

**为什么必须指定 `core.sshCommand`**：Git for Windows 自带的是 MSYS2 版 ssh
（`C:\Program Files\Git\usr\bin\ssh.exe`），它在沙箱里同样报
`couldn't create signal pipe, Win32 error 5`；换成系统自带的 `C:\Windows\System32\OpenSSH\ssh.exe` 就正常。
不指定的话，`git push` 会报 `fatal: Could not read from remote repository`，看起来像权限问题，其实是 ssh 进程起不来。

## 二、为什么当初 https 那么麻烦（历史记录，已弃用）

https 方式要同时解决 TLS 和凭据两件事，而这两件在沙箱里都被挡：

| 环节 | 报错 | 原因 |
|---|---|---|
| TLS | `schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS` | Git 默认用 schannel 走 Windows SSPI，沙箱受限令牌拿不到凭证。连匿名 `ls-remote` 都失败，容易误判成断网 |
| 凭据 | `couldn't create signal pipe, Win32 error 5`，随后 `could not read Username` | `credential.helper=manager` 要经 `sh.exe` 起管道调 GCM，沙箱禁止建命名管道 |
| 取 token | — | 无 `GH_TOKEN`、无 `~/.ssh`，只能 `gh auth token` 现取 |

SSH 只留下"能不能连上"一件事，且不需要隧道凭证，所以一次配好后彻底干净。

### 备用：万一要走 https

```powershell
$tok = & "C:\Program Files\GitHub CLI\gh.exe" auth token
$b64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("x-access-token:" + $tok.Trim()))
$env:GIT_TERMINAL_PROMPT = "0"
git -c http.sslBackend=openssl -c credential.helper= `
    -c http.extraheader="Authorization: Basic $b64" push origin main
```

`-c` 参数只对这一次命令生效，不会写进配置；**不要把 token 拼进 remote URL**（会留在 `.git/config` 和 reflog 里）。

## 三、排错

| 现象 | 原因 | 处理 |
|---|---|---|
| `fatal: Could not read from remote repository` + `couldn't create signal pipe` | git 调到了 MSYS2 的 ssh | `git config --local core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"` |
| `Permission denied (publickey)` | 密钥没注册 / 注册到了别的仓库 | `ssh -T git@github.com`，应回 `Hi yuquant/memovr! ...` |
| `Host key verification failed` | known_hosts 里没有或对不上 | 重新写入官方 host key（见下） |
| `schannel: ... SEC_E_NO_CREDENTIALS` | 还在用 https | 换回 SSH remote |
| `remote: Permission to ... denied` | 部署密钥权限不对 | 确认部署密钥 `read_only=false` |
| `failed to push some refs ... non-fast-forward` | 远端有新提交 | `git fetch` + `git pull --rebase` 再推 |
| PowerShell 显示红字 `NativeCommandError` 但 push 成功 | git 把进度写到 stderr，PowerShell 当错误处理 | 忽略；以 `main -> main` 行和 SHA 比对为准 |
| 脚本报 `无法加载文件……未进行数字签名` | 执行策略限制 | `powershell -ExecutionPolicy Bypass -File xxx.ps1` |
| 脚本报 `Unexpected token '}'` | PS 5.1 按 GBK 读了 UTF-8（无 BOM）的中文注释 | 脚本注释用英文，或存成带 BOM 的 UTF-8 |

## 四、验证

```powershell
ssh -T git@github.com                    # Hi yuquant/memovr! You've successfully authenticated
git ls-remote origin refs/heads/main     # 远端 main 的 SHA
git rev-parse HEAD                       # 本地 HEAD 的 SHA
git status -sb                           # ## main...origin/main，且无 ahead
```

两个 SHA 相同、`status` 里没有 `ahead` 才算真推上去了。
（`ssh -T` 返回 exit code 1 是正常的，GitHub 不提供 shell，看输出为准。）

## 五、换仓库 / 换机器

- **别的仓库也要推**：把这把公钥加到那个仓库的 Deploy key（Settings → Deploy keys → Allow write access），
  或改成账号级密钥：`gh auth refresh -h github.com -s admin:public_key` 然后
  `gh ssh-key add "$env:USERPROFILE\.ssh\id_ed25519.pub" --title "..."`。
- **换新机器**：重复第一节 1–4 步即可。
- **撤销本机密钥**：`gh api -X DELETE repos/yuquant/memovr/keys/169886878`，
  再删本地 `%USERPROFILE%\.ssh\id_ed25519*`。

## 六、安全说明

私钥无口令保存在 `%USERPROFILE%\.ssh\id_ed25519`，仅对 `memovr` 这一个仓库有写权限（部署密钥天然限制范围），
比账号级 token 的影响面小。若想更严：给密钥加口令，并用 `ssh-agent` 托管，之后 push 不再输密码。

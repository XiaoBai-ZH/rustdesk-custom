# RustDesk 自建服务器客户端 · 打包作业手册

> **本文档的用途**：交给 AI，让它从零打出内置指定服务器的 RustDesk 客户端安装包。
> **执行者假设**：能跑 bash、有 GitHub CLI 且已登录、有写权限的目标仓库。
> **上次成功基线**：上游 `rustdesk/rustdesk` **1.5.3**（commit `95cca3f661f635b3cc85c04935fe8d96d8f432ef`），产物仓库 `XiaoBai-ZH/rustdesk-custom`，四平台全部验证通过。
>
> ⚠️ **行号会随上游变动。所有补丁都以「锚点字符串」定位，动手前必须先跑第 3.0 节的锚点自检。任一锚点失配就停下来重读上下文，不要盲套。**

---

## 1. 需要人提供的输入

AI 无法自行获取，**开工前必须问齐**：

| 变量 | 含义 | 示例 | 怎么取 |
|---|---|---|---|
| `GH_OWNER` | GitHub 用户名 | `XiaoBai-ZH` | `gh api user --jq .login` |
| `REPO` | 产物仓库名 | `rustdesk-custom` | 已存在则复用 |
| `RENDEZVOUS_SERVER` | ID/信令服务器 | `rd.example.com:21116` | 用户的服务端地址。**建议带端口**，理由见 4.1 |
| `RS_PUB_KEY` | 服务端 Ed25519 **公钥** | `9ngcOypZ…ml8=`（44 字符） | `docker exec $(docker ps -qf ancestor=lejianwen/rustdesk-server-s6:latest) cat /data/id_ed25519.pub` |
| `API_SERVER` | API 地址 | `https://rd.example.com` | 反代到容器 21114 的那个对外地址 |

**不需要**中继服务器地址——客户端会从 ID 服务器端口 +1 自动推导（`src/rendezvous_mediator.rs` 的 `get_relay_server()` 第三级兜底），或由 hbbs 下发。全仓无官方 relay 兜底。

### 1.1 公钥格式校验（必做，填错会静默回落官方公钥）

```bash
python3 -c "
import base64
k='粘贴公钥'
assert len(k)==44, f'长度 {len(k)}，应为 44（88 字符是私钥，用错了）'
assert k==k.strip(), '含首尾空白，会导致 base64 解码失败并静默回落官方公钥'
assert k.endswith('=') and not k.endswith('=='), '结尾应为单个 ='
raw=base64.b64decode(k, validate=True)
assert len(raw)==32, f'解码后 {len(raw)} 字节，应为 32'
print('OK 合法 Ed25519 公钥')
"
```

---

## 2. 本机环境要求

| 工具 | 用途 | 检查命令 | 必需 |
|---|---|---|---|
| `gh` (GitHub CLI) | 建仓、设 Secret、触发构建、取产物 | `gh --version` | ✅ |
| `git` | 本地提交推送 | `git --version` | ✅ |
| `python3` + `pyyaml` | YAML 校验、文件清单比对 | `python3 -c "import yaml"`；缺则 `pip3 install --user pyyaml` | ✅ |
| `curl` | 断点续传下载产物 | `curl --version` | ✅ |
| `strings` / `unzip` / `ar` | 验证产物 | — | ✅ |
| `codesign` / `hdiutil` | 验证 macOS 签名（仅 macOS 本机有） | — | 验 macOS 时需要 |

### 2.1 gh 权限检查

```bash
gh auth status
```

**Token scopes 必须包含 `repo` 和 `workflow`。** 缺 `workflow` 会导致无法通过 API 修改 `.github/workflows/*.yml`（报 `refusing to allow an OAuth App to create or update workflow`）。补齐：

```bash
gh auth refresh -h github.com -s repo -s workflow
```

### 2.2 两个必须用内联 `-c` 绕开的本机配置问题

**不要修改用户的全局 git 配置**，用内联参数：

```bash
# 问题 A：credential.helper 可能为空，HTTPS push 会卡在交互式认证
git -c credential.helper='!gh auth git-credential' push origin master

# 问题 B：全局 user.email 可能是工作邮箱，提交到公开仓库会永久泄露
# noreply 邮箱从这里取：
gh api user/emails --jq '.[]|select(.email|endswith("users.noreply.github.com"))|.email'
git -c user.name="<GitHub用户名>" -c user.email="<上面取到的noreply>" commit -m "..."
```

先检查现状：
```bash
git config --global --get credential.helper   # 空 → 必须用内联 helper
git config --global --get user.email          # 若是工作邮箱 → 必须用内联 identity
```

### 2.3 网络注意

GitHub Release / Artifact 的大文件下载**可能被中间层截断**（实测在 20MB 整边界断流，报 `stream error: PROTOCOL_ERROR`）。**永远用断点续传 + digest 校验**，见 7.1。

⚠️ **不要并发跑多个续传任务写同一个文件**，会互相写坏（文件变得比目标还大）。

---

## 3. 打补丁

### 3.0 锚点自检（第一步，必做）

拉取上游后立刻跑：

```bash
cd <上游克隆目录>
git submodule update --init --depth 1 libs/hbb_common

echo "① PROD_RENDEZVOUS_SERVER: $(grep -c 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());' libs/hbb_common/src/config.rs)"
echo "② RS_PUB_KEY:             $(grep -c 'pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";' libs/hbb_common/src/config.rs)"
echo "③ websocket api_server:   $(grep -c 'let api_server = Config::get_option("api-server");' libs/hbb_common/src/websocket.rs)"
echo "④ get_api_server_ 锚点:   $(grep -c 'let s0 = get_custom_rendezvous_server(custom);' src/common.rs)"
echo "⑤ SIGN_BASE_URL:          $(grep -c 'SIGN_BASE_URL: "\${{ secrets.SIGN_BASE_URL }}-2"' .github/workflows/flutter-build.yml)"
echo "⑥ run-on-arch 处数:       $(grep -c 'run-on-arch-action' .github/workflows/flutter-build.yml)"
echo "⑦ env+setup 模式处数:     $(python3 -c "s=open('.github/workflows/flutter-build.yml').read(); print(s.count('          githubToken: \${{ github.token }}\n          setup: |\n'))")"
echo "⑧ create unsigned dmg:    $(grep -c 'name: create unsigned dmg' .github/workflows/flutter-build.yml)"
echo "⑨ build.py 拷 service:    $(grep -c 'cp -rf ../target/release/service' build.py)"
```

**期望**：①④⑤⑧⑨ 各为 `1`，③ 为 `1`，⑥ 与 ⑦ **相等**（1.5.3 时都是 4）。

任一为 0 → 上游改了这处，**停下来读上下文重新设计补丁**。⑥≠⑦ → 说明有新的 `run-on-arch-action` 用法不符合该模式，必须逐个人工确认。

### 3.1 源码补丁（4 处）

**① `libs/hbb_common/src/config.rs` — 烧入 ID 服务器**

```rust
// 找这行（在 lazy_static! 块内）
    pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());
// 替换为
    pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new(
        match option_env!("RENDEZVOUS_SERVER") {
            Some(server) if !server.is_empty() => server,
            _ => "",
        }
        .to_owned()
    );
```

> **为什么烧这里而不是 `RENDEZVOUS_SERVERS` 常量**：`PROD_RENDEZVOUS_SERVER` 在解析链第 3 级（常量在第 5 级）；`get_custom_rendezvous_server()` 读它，所以 API 能自动推导；`websocket.rs` 的 `own_servers` 白名单也读它，所以 WebSocket 能识别为自有服务器。改常量会同时丢掉后两条。

**② `libs/hbb_common/src/config.rs` — 烧入公钥**

```rust
// 找这行
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
// 替换为
pub const PUBLIC_RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
pub const RS_PUB_KEY: &str = match option_env!("RS_PUB_KEY") {
    Some(key) if !key.is_empty() => key,
    _ => PUBLIC_RS_PUB_KEY,
};
```

**③ `libs/hbb_common/src/websocket.rs` — 修 ws/wss 判定（在 `check_ws()` 内）**

```rust
// 找这段
    let protocol = if is_domain {
        let api_server = Config::get_option("api-server");
        if api_server.starts_with("https") {
// 在 let api_server = ... 之后插入
        let api_server = if api_server.is_empty() {
            option_env!("API_SERVER").unwrap_or("").to_owned()
        } else {
            api_server
        };
```

> **不打会怎样**：开启「使用 WebSocket」时（`allow-websocket = "Y"`，默认关闭），`check_ws()` 直接读原始选项、绕过 `get_api_server()`，选项为空 → 选 `ws://`（明文 80）→ HTTPS-only 反代连不上 → 客户端永远卡在「正在接入 RustDesk 网络...」。**这个症状极具误导性**：该提示反映的是 ID 服务器连通状态，不是 API 状态。

**④ `src/common.rs` — 让 API_SERVER 生效（在 `get_api_server_()` 内）**

```rust
// 找这段
    if !api.is_empty() {
        return api.to_owned();
    }
    let s0 = get_custom_rendezvous_server(custom);
// 在两者之间插入
    if let Some(baked) = option_env!("API_SERVER") {
        if !baked.is_empty() {
            return baked.to_owned();
        }
    }
```

> **不打会怎样**：API 会走推导得到 `http://<host>:21114`。若 21114 未对公网开放（走 HTTPS 反代的常见情况），登录/设备注册/地址簿同步全部失效。

### 3.2 工作流补丁（3 处，都在 `.github/workflows/flutter-build.yml`）

**⑤ env 块导出 3 个 secret**

在 `env:` 块内（1.5.3 是 23–64 行，`jobs:` 之前）追加：

```yaml
  RS_PUB_KEY: "${{ secrets.RS_PUB_KEY }}"
  RENDEZVOUS_SERVER: "${{ secrets.RENDEZVOUS_SERVER }}"
  API_SERVER: "${{ secrets.API_SERVER }}"
```

**⑥ 4 处 `run-on-arch-action` 加 `env:` 输入**

四处模式完全一致，可一次性全局替换（替换前用 3.0 的 ⑥⑦ 确认处数相等）：

```yaml
# 找
          githubToken: ${{ github.token }}
          setup: |
# 替换为
          githubToken: ${{ github.token }}
          env: |
            RS_PUB_KEY: ${{ secrets.RS_PUB_KEY }}
            RENDEZVOUS_SERVER: ${{ secrets.RENDEZVOUS_SERVER }}
            API_SERVER: ${{ secrets.API_SERVER }}
          setup: |
```

> **不打会怎样**：Linux 构建跑在 `ubuntu18.04` 容器内，`dockerRunArgs` 只有 `--volume` 没有 `-e`，**工作流级 env 不会跨进容器**。结果：构建全绿，但 Linux 全部产物（deb/rpm/AppImage/flatpak/archlinux）**静默烧入官方服务器**。Windows/macOS/Android 是原生 runner，不受影响——所以这个 bug 只表现在 Linux，很容易漏。
>
> 该 action 的 `action.yml` 明确支持 `env` 输入："Environment variables to propagate to the container... available in both run and setup"。
>
> archlinux / AppImage / rpm **不需要单独修**：`res/PKGBUILD` 只有 `package()` 没有 `build()`，只是 `cp -r` 已构建的 bundle；AppImage 下载 `.deb` 重打包；`rpmbuild` 在同一容器内。

**⑦ macOS 打包前 ad-hoc 重签**

在 `create unsigned dmg` 步骤的 `run:` **最开头**插入：

```yaml
      - name: create unsigned dmg
        if: env.UPLOAD_ARTIFACT == 'true'
        run: |
          bash ./.github/scripts/sign-macos-app.sh \
            ./flutter/build/macos/Build/Products/Release/RustDesk.app \
            - \
            ./flutter/macos/Runner/Release.entitlements
          CREATE_DMG="$(command -v create-dmg)"
          ...（原有内容不动）
```

> **不打会怎样**：`build.py` 里有一行 `cp -rf ../target/release/service .../RustDesk.app/Contents/MacOS/`，它在 Xcode 完成 ad-hoc 签名**之后**才把 9.97MB 的 `service` 塞进 bundle，封印清单不含它 → 签名损坏 → macOS 报**「已损毁，无法打开，应该移到废纸篓」**，且 `xattr -rd com.apple.quarantine` **完全无效**（因为坏的是签名不是隔离属性）。
>
> 上游靠后续的 "Codesign app and create signed dmg" 步骤重签，但那步 gated on `MACOS_P12_BASE64`，自建时被跳过。
>
> **复用上游的 `sign-macos-app.sh` 而不是自己写 codesign**：它已处理正确顺序（Frameworks 内 Mach-O → `.framework` 目录 → `Contents/MacOS/service` → 带 entitlements 签 bundle），末尾自带 `codesign --verify --deep --strict`，配合 `set -euo pipefail`，签名坏了会让步骤失败而不是产出坏包。传 `-` 时脚本自动跳过 `--timestamp`。

### 3.3 工作流补丁（另外 2 个文件）

**⑧ `flutter-nightly.yml` 和 `flutter-tag.yml` 各加 caller 级权限**

```yaml
jobs:
  run-flutter-nightly-build:            # flutter-tag.yml 里叫 run-flutter-tag-build
    uses: ./.github/workflows/flutter-build.yml
    permissions:
      contents: write                   # ← 加这两行
    secrets: inherit
```

> **不打会怎样**：`flutter-build.yml` 的 `generate-sbom` 和 `publish-signed-update-metadata` 声明了 `permissions: contents: write`。**可复用工作流的 token 不能超出调用方授予的范围**，而仓库默认 `default_workflow_permissions: read` → 整个 run **1 秒内 `startup_failure`，0 个 job，无日志，无 check run，GitHub 不给任何错误注解**。
>
> ⚠️ **不要用放宽仓库级默认权限来解决**（`PUT /repos/.../actions/permissions/workflow` 设 `write`）。那会在一个满是第三方工作流文件的公开仓上永久削弱最小权限控制。caller 级授权是正确做法。

**⑨ `flutter-nightly.yml` 删掉定时触发**

```yaml
on:
  schedule:                    # ← 删掉这三行
    - cron: "0 0 * * *"        #
  workflow_dispatch:           # ← 保留
```

> 不删的话每天午夜自动跑一次完整构建（85 分钟）并覆盖 Release。

### 3.4 要删除的文件

| 文件 | 原因 |
|---|---|
| `.gitmodules` | hbb_common 改为内联目录 |
| `libs/hbb_common/.git` | 同上（它是个 gitfile，内容 `gitdir: ../../.git/modules/libs/hbb_common`） |
| `.github/workflows/flutter-ci.yml` | `on: push: branches: [master]`，每次推送白跑一次全平台构建，且 `upload-artifact: false` 什么都不产出 |

### 3.5 明确不要动的东西

**❌ 不要设 `SIGN_BASE_URL` 这个 Secret。** `flutter-build.yml` 里它是 `"${{ secrets.SIGN_BASE_URL }}-2"`，末尾硬拼了 `-2`。不设置时值正好是 `-2`，所有 `!= '-2'` 的签名步骤自动跳过——这是上游故意让非官方仓库能直接跑的设计。设了反而变成 `<你的值>-2`，签名步骤会执行并去连 `http://localhost:5000` 然后失败。

**❌ 不要跑 `playground.yml`。** 它是 macos-13/sciter 时代的旧矩阵，不调用 `flutter-build.yml`，而且它的 `SIGN_BASE_URL` 没有 `-2` 后缀。

**❌ 不要改 `RENDEZVOUS_SERVERS` 常量**（`config.rs` 里的 `&["rs-ny.rustdesk.com"]`）。它在解析链第 5 级（末级兜底），永远轮不到。保留它的好处是域名失效时客户端不会崩在空地址上。**产物里能 `strings` 到 `rs-ny.rustdesk.com` 是预期的，不是 bug。**

**❌ 不要设 `ANDROID_SIGNING_KEY` / `MACOS_P12_BASE64`**，除非你真的有证书。不设时：Android 走 "Publish unsigned apk package" 分支产出调试签名的 APK（可正常安装）；macOS 走 "create unsigned dmg" 分支（已由补丁 ⑦ 修好签名）。

---

## 4. GitHub 侧配置

### 4.1 三个 Secret

```bash
R=<GH_OWNER>/<REPO>
gh secret set RENDEZVOUS_SERVER -R $R --app actions --body '<域名>:21116'
gh secret set RS_PUB_KEY        -R $R --app actions --body '<44字符公钥>'
gh secret set API_SERVER        -R $R --app actions --body 'https://<域名>'
gh secret list -R $R --app actions     # 只返回名字+时间戳，值不可读回
```

必须是 **Secret** 不是 Variable（工作流读的是 `secrets.`）。

**`RENDEZVOUS_SERVER` 建议带端口 `:21116`**：带端口时 `increase_port("host:21116", +1)` 直接得出中继 `host:21117`；不带端口则依赖 `check_port()` 补默认端口，多一层间接。两种都能工作，带端口更精确。

**`API_SERVER` 必须显式设置**（如果 API 走反代）：不设则会推导出 `http://<域名>:21114`，而反代场景下 21114 通常不对公网开放。

### 4.2 仓库设置

| 项 | 值 | 检查/设置命令 |
|---|---|---|
| 可见性 | **Public** | 见下方成本说明 |
| Actions | 启用 | `gh api /repos/$R/actions/permissions` → `{"enabled":true,...}` |
| `default_workflow_permissions` | **保持 `read`** | `gh api /repos/$R/actions/permissions/workflow` |

**为什么必须公开**（Free 计划实测数据）：

```
一次完整构建成本 = Windows 146min×$0.010 + macOS 121min×$0.062 + Linux 362min×$0.006
                = $1.46 + $7.50 + $2.17 = $11.13
Free 计划月度额度 = $12   →  单次构建占 93%
产物存储上限 = 500MB      →  一次构建的产物远超此限
公开仓库 = 不计费、不限时长、存储不限
```

且 $11.13 是上游**暖缓存**成绩，新仓库首次构建缓存全冷会更久，极可能中途超额失败。

**公开仓的隐私影响**：仓库内容不含个人信息（补丁是通用的，服务器地址和公钥在 Secrets 里，Secrets 只写不可读，API 无任何端点能取回值，日志自动打码）。唯一暴露面是 Release 产物可被下载并 `strings` 出服务器地址——但这与分发给被控端的安装包本身等价。**下载完删掉 Release 和 Artifacts 即可消除。**

### 4.3 关于 fork

**不要用 fork。** 本账号 `POST /repos/rustdesk/rustdesk/forks` 返回 `403 You can't fork this repository at this time`（账号级限制，已排除 scope/2FA/限流/仓库设置）。

即使能 fork 也不推荐：公开仓库的 fork 永远是公开的、无法改私有；而且要额外 fork `hbb_common` 并用 GitHub API 的 trees→commits→refs 三段式移动 gitlink 指针。

**改用普通仓库 + 内联 submodule**，理由见 5.1。

---

## 5. 组装源码树并推送

### 5.1 为什么内联 hbb_common

`libs/hbb_common` 是上游**唯一**的 submodule，且是**纯 path 依赖**：

```toml
# Cargo.toml
hbb_common = { path = "libs/hbb_common", features = ["webrtc"] }   # 第 55 行
members = ["libs/scrap", "libs/hbb_common", ...]                    # 第 220 行 workspace
```

已核实构建脚本里**没有任何 `git submodule` 逻辑**，工作流的 `submodules: recursive` 在无 submodule 时是空操作。内联后：不需要第二个仓库、不需要移动 gitlink、补丁直接改文件、`git diff` 一目了然。

### 5.2 首次建仓

```bash
# 1. 拉上游
mkdir -p /tmp/rd && cd /tmp/rd
git clone --depth 1 https://github.com/rustdesk/rustdesk.git upstream
cd upstream && git submodule update --init --depth 1 libs/hbb_common

# 2. 拷出干净工作树（不带 .git）
mkdir -p /tmp/rd/build
tar cf - --exclude=./.git . | (cd /tmp/rd/build && tar xf -)

# 3. 内联 submodule + 删文件
cd /tmp/rd/build
rm -f .gitmodules libs/hbb_common/.git .github/workflows/flutter-ci.yml

# 4. 打第 3 节的全部补丁（先跑 3.0 锚点自检）

# 5. YAML 校验
python3 -c "
import yaml,os
b='.github/workflows/'
for f in sorted(os.listdir(b)):
    try: yaml.safe_load(open(b+f)); print('OK  ',f)
    except Exception as e: print('FAIL',f,e)
"

# 6. ⚠️ 关键：补回被 .gitignore 吞掉的文件（见 5.4）

# 7. 初始化并提交
git init -q -b master
git add -A
# ← 这里执行 5.4 的 git add -f
git -c user.name="<GH_OWNER>" -c user.email="<noreply邮箱>" commit -q -m "<说明>"

# 8. 建仓并推送
gh repo create <REPO> --public --description "RustDesk with self-hosted server baked in"
git remote add origin https://github.com/<GH_OWNER>/<REPO>.git
git -c credential.helper='!gh auth git-credential' push -u origin master
```

### 5.3 后续更新（仓库已存在，上游出了新版）

```bash
# 1. 拉新版上游到 upstream/，按 5.2 的步骤 1-5 准备好新的 /tmp/rd/build 树
#    （注意：此时 build/ 里可能已有 .git，下一步必须排除它）

# 2. 克隆现有仓库
git clone https://github.com/<GH_OWNER>/<REPO>.git work
cd work

# 3. 清空工作树但保留 .git
find . -mindepth 1 -maxdepth 1 ! -name .git -exec rm -rf {} +

# 4. 换入新树 —— ⚠️ 必须排除 build/.git，否则会覆盖仓库自己的 .git
(cd /tmp/rd/build && tar cf - --exclude=./.git .) | tar xf -

# 5. 确认 .git 还是仓库自己的（应显示 origin 指向 <REPO>）
git remote -v

# 6. 暂存并核对文件清单（5.4）
git add -A
# ← 执行 5.4 的比对；此时多数文件已在 HEAD 里、.gitignore 不再影响，但仍要核对总数

# 7. 提交推送
git -c user.name="<GH_OWNER>" -c user.email="<noreply邮箱>" commit -q -m "Update to rustdesk <新版本号>"
git -c credential.helper='!gh auth git-credential' push origin master
```

### 5.4 ⚠️ 必须补回被 `.gitignore` 吞掉的文件

**这是最容易漏、后果最严重的一步。** 全新 `git init` + `git add -A` 时，`.gitignore` 里的 `*.png` 等规则会吞掉上游**已跟踪**的文件，且**不报任何警告**。上次实测漏掉 **123 个**，其中包括 `flutter/android/app/src/main/res/mipmap-*/ic_launcher.png` 等全部 Android 启动图标 —— **漏了 Android 构建直接失败**。

```bash
# 生成上游权威清单
cd /tmp/rd/upstream
git ls-files > /tmp/up_main.txt
cd libs/hbb_common && git ls-files | sed 's|^|libs/hbb_common/|' > /tmp/up_sub.txt
cd /tmp
cat up_main.txt up_sub.txt | grep -v "^\.gitmodules$" | grep -v "^libs/hbb_common$" | sort > up_all.txt
echo "上游应有文件数: $(wc -l < up_all.txt)"

# 生成新树实际暂存清单
cd /tmp/rd/build
git diff --cached --name-only | sort > new_all.txt
echo "实际暂存数: $(wc -l < new_all.txt)"

# 找出被吞掉的
comm -23 /tmp/up_all.txt /tmp/new_all.txt > /tmp/missing.txt
echo "被吞掉: $(wc -l < /tmp/missing.txt) 个"
cat /tmp/missing.txt | head -20      # 人工看一眼，确认没有你故意删的文件

# 强制补回（排除故意删除的 flutter-ci.yml）
grep -v '^\.github/workflows/flutter-ci\.yml$' /tmp/missing.txt \
  | tr '\n' '\0' | xargs -0 git add -f

# 复核：应只剩故意删的那几个
git diff --cached --name-only | sort > new_all2.txt
echo "--- 仍缺失（应只有 flutter-ci.yml 和 libs/hbb_common 这个 gitlink 条目）---"
comm -23 /tmp/up_all.txt /tmp/new_all2.txt
echo "--- 多出（应为空）---"
comm -13 /tmp/up_all.txt /tmp/new_all2.txt

# hbb_common 内联文件数核对
echo "上游 submodule: $(wc -l < /tmp/up_sub.txt)  实际暂存: $(git diff --cached --name-only | grep -c '^libs/hbb_common/')"
# 两个数字必须相等
```

### 5.5 推送后核对远端

```bash
R=<GH_OWNER>/<REPO>
# 逐个补丁文件与本地比对
for f in libs/hbb_common/src/config.rs libs/hbb_common/src/websocket.rs src/common.rs .github/workflows/flutter-build.yml .github/workflows/flutter-nightly.yml .github/workflows/flutter-tag.yml; do
  printf "%-50s " "$f"
  gh api "/repos/$R/contents/$f?ref=master" --jq '.content' | base64 -d \
    | diff -q - "/tmp/rd/build/$f" >/dev/null && echo "OK" || echo "MISMATCH"
done
# 确认删除生效
for f in .gitmodules .github/workflows/flutter-ci.yml; do
  printf "%-50s " "$f"
  gh api "/repos/$R/contents/$f?ref=master" 2>&1 | grep -q "Not Found" && echo "已删除" || echo "仍存在"
done
# 确认 hbb_common 是普通目录不是 submodule
gh api "/repos/$R/contents/libs/hbb_common?ref=master" --jq 'if type=="array" then "dir OK" else "仍是 submodule" end'
```

---

## 6. 触发构建与监控

```bash
R=<GH_OWNER>/<REPO>

# 触发
gh workflow run "Flutter Nightly Build" -R $R --ref master

# 取 run id
sleep 20
RID=$(gh run list -R $R --workflow "Flutter Nightly Build" --limit 1 --json databaseId --jq '.[0].databaseId')

# ⚠️ 立刻确认 job 被创建了（防坑 #4 的 startup_failure）
gh api "/repos/$R/actions/runs/$RID" --jq '{status:.status,conclusion:(.conclusion//"-")}'
gh api "/repos/$R/actions/runs/$RID/jobs?per_page=100" --jq '.total_count'
# conclusion 若是 startup_failure 且 total_count 为 0 → 检查补丁 ⑧ 是否漏了
```

监控（约 85 分钟，暖缓存）：

```bash
gh api "/repos/$R/actions/runs/$RID/jobs?per_page=100" \
  --jq '[.jobs[]|.conclusion // .status]|group_by(.)|map("\(.[0])=\(length)")|join("  ")'
```

**正常形态：29 job = 27 success + 2 skipped**
- `build-rustdesk-web` skipped（它自己写着 `if: False`）
- `publish-signed-update-metadata` skipped 或不创建（`if` 含 `github.repository == 'rustdesk/rustdesk'`）

也可用 `flutter-tag.yml`：推 tag `1.5.3-1` 触发（正则要求 `-` 后是数字，`1.5.3-custom` **不匹配**）。

---

## 7. 验证产物（必做，构建全绿 ≠ 产物正确）

> **上次踩过的最贵的坑**：构建 27 个 job 全绿，但 Linux 全部产物烧的是官方服务器（坑 #6 的容器 env）。**必须逐平台验证二进制。**

### 7.1 下载（必须断点续传 + digest 校验）

```bash
R=<GH_OWNER>/<REPO>
# 取权威 size 和 digest
gh api "/repos/$R/releases/tags/nightly" --jq '.assets[]|{name:.name,size:.size,digest:.digest}'

# 断点续传下载（单线程！不要并发写同一文件）
F=rustdesk-<ver>-x86_64.deb; EXPECT=<上面查到的size>
for i in $(seq 1 10); do
  [ "$(stat -f%z $F 2>/dev/null||echo 0)" -ge "$EXPECT" ] && break
  curl -sL -C - -o $F "https://github.com/$R/releases/download/nightly/$F"
done
shasum -a 256 $F    # 与 digest 比对，必须完全一致
```

⚠️ `gh release download` 和 `gh api .../zip` 在大文件上实测会截断，**不要依赖它们**，用上面的 curl 续传。

### 7.2 通用判据

| 字符串 | 期望命中 |
|---|---|
| 你的域名 | ≥1 |
| 你的公钥（44 字符） | 1 |
| `https://<你的域名>` | ≥1 |
| `OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=`（官方公钥） | **0** |
| `admin.rustdesk.com` | **0** |
| `rs-ny.rustdesk.com` | 1–2（**正常**，末级兜底常量，故意保留） |

### 7.3 Linux

```bash
mkdir ex && cd ex && ar x ../rustdesk-*-x86_64.deb && tar xf data.tar.*
B=usr/share/rustdesk/lib/librustdesk.so       # 约 56MB，这才是主体
strings -a "$B" | grep -c "<你的域名>"
```
⚠️ `usr/share/rustdesk/rustdesk` 只有 44KB，是启动器，**在它上面 strings 什么都搜不到**。

### 7.4 Android

```bash
unzip -o -q rustdesk-*-aarch64.apk 'lib/*' -d ax
strings -a ax/lib/arm64-v8a/librustdesk.so | grep -c "<你的域名>"
```

### 7.5 Windows

Release 里的 `.exe` 是自解压包、`.msi` 内含压缩 CAB（`MSCF` 魔数），**直接 strings 连官方默认值都搜不到**。必须取未打包的 artifact：

```bash
AID=$(gh api "/repos/$R/actions/runs/$RID/artifacts?per_page=100" \
      --jq '.artifacts[]|select(.name=="rustdesk-unsigned-windows-x86_64")|.id')
gh api "/repos/$R/actions/artifacts/$AID/zip" > win.zip
unzip -q win.zip -d winx
strings -a winx/librustdesk.dll | grep -c "<你的域名>"
```

### 7.6 macOS（有两个陷阱）

```bash
hdiutil attach -nobrowse -readonly rustdesk-*-aarch64.dmg
V=/Volumes/<卷名>          # ⚠️ 卷名是 rustdesk-<ver>-<arch>，不是 RustDesk
B="$V/RustDesk.app/Contents/Frameworks/liblibrustdesk.dylib"

# 陷阱 1：主体是 dylib，不是 Contents/MacOS/RustDesk（那个只有 148KB）
strings -a "$B" | grep -c "<你的公钥>"        # 44 字符，能直接搜到

# 陷阱 2：域名可能搜不到完整串！x86_64/arm64 codegen 会把短字符串拆成 8 字节 movabs 立即数
#         内联进指令流，不放 .rodata。用分片搜：
strings -a "$B" | grep -o "[a-z:/._0-9-]*<域名片段>[a-z:/._0-9-]*" | sort -u
```

分片输出的**邻域符号还能告诉你补丁是否生效**（比数命中次数可靠得多）。上次实测：
```
:21114allow-https-21114https://<域名>       ← src/common.rs 的 get_api_server_ 补丁生效
api-serverhttps://<域名>websocket-port       ← websocket.rs 的 check_ws 补丁生效
rustdesk.<域名>:21116                        ← PROD_RENDEZVOUS_SERVER 生效
```

**签名校验（必做，防坑 #7 回归）：**
```bash
codesign --verify --deep --strict --verbose=2 "$V/RustDesk.app"
# 必须输出：valid on disk  +  satisfies its Designated Requirement
# 若报 "a sealed resource is missing or invalid" 且附 "file added: .../Contents/MacOS/service"
#   → 补丁 ⑦ 没生效

codesign -dv "$V/RustDesk.app" 2>&1 | grep flags
# 期望 flags=0x10002(adhoc,runtime)   ← runtime 标志应保留
codesign -dv "$V/RustDesk.app/Contents/MacOS/service" 2>&1 | grep flags
# 期望 flags=0x10002(adhoc,runtime)   ← service 应已被封印
codesign -d --entitlements :- "$V/RustDesk.app" | plutil -p -
# 期望 4 项：app-sandbox=false, allow-jit=true, audio-input=true, network.client=true

hdiutil detach "$V"
```

`spctl -a -vvv -t install` 返回 `rejected` 是**预期的**（无公证），不影响使用。

---

## 8. 交付给用户时必说的安装注意事项

### macOS
补丁 ⑦ 生效后 dmg 里的 app 签名是好的，但**仍未公证**，所以：

**推荐用终端 `cp` 安装**（实测**不会**带上 `com.apple.quarantine`）：
```bash
hdiutil attach ~/Downloads/rustdesk-<ver>-aarch64.dmg
cp -R "/Volumes/rustdesk-<ver>-aarch64/RustDesk.app" /Applications/
hdiutil detach "/Volumes/rustdesk-<ver>-aarch64"
open /Applications/RustDesk.app
```

**若在访达里拖的**（隔离属性会跟过去），补一条：
```bash
xattr -cr /Applications/RustDesk.app
```

**若拿到的是没打补丁 ⑦ 的旧 dmg**（签名已坏，`xattr` 无效），本地急救：
```bash
codesign --force --deep --sign - /Applications/RustDesk.app
```
副作用：不带 `--entitlements` 会丢 entitlements 和 runtime 标志（`flags` 变 `0x2(adhoc)`）。对非沙箱 ad-hoc 应用无害。要完整保留就下载上游脚本：
```bash
curl -sL -o /tmp/sign.sh https://raw.githubusercontent.com/<GH_OWNER>/<REPO>/master/.github/scripts/sign-macos-app.sh
curl -sL -o /tmp/rd.plist https://raw.githubusercontent.com/<GH_OWNER>/<REPO>/master/flutter/macos/Runner/Release.entitlements
bash /tmp/sign.sh /Applications/RustDesk.app - /tmp/rd.plist
```

彻底免掉这些步骤需要 Apple Developer 账号（$99/年）+ 三个 Secret：`MACOS_P12_BASE64`、`MACOS_P12_PASSWORD`、`MACOS_CODESIGN_IDENTITY`。

### Windows
未签名 → SmartScreen 弹「Windows 已保护你的电脑」→ 点「更多信息 → 仍要运行」。
`.exe` 是自解压便携版，运行后可在界面内点「安装」；要静默部署/组策略用 `.msi`。

### Android
APK 用 Flutter 调试密钥签名（`CN=Android Debug, O=Android, C=US`），**可正常安装**，但：
- 与官方版签名不一致，**必须先卸载已有的 RustDesk**，否则报「应用未安装」
- 需允许「安装未知来源应用」

校验签名：
```bash
unzip -p rustdesk-*.apk META-INF/CERT.RSA > /tmp/c.rsa
openssl pkcs7 -inform DER -in /tmp/c.rsa -print_certs -noout
```

### 该下哪个文件

| 设备 | 文件 |
|---|---|
| Windows x64 | `rustdesk-<ver>-x86_64.exe`（或 `.msi`） |
| Windows ARM | `rustdesk-<ver>-aarch64.exe` |
| macOS Apple 芯片 | `rustdesk-<ver>-aarch64.dmg`（`uname -m` = `arm64`） |
| macOS Intel | `rustdesk-<ver>-x86_64.dmg`（`uname -m` = `x86_64`） |
| Android arm64 | `rustdesk-<ver>-aarch64.apk` |
| Linux x64 | `rustdesk-<ver>-x86_64.deb` / `.rpm` / `.AppImage` |

**不要下**：`*-sciter.*`（旧版 UI）、`x86_64.apk`（模拟器）、`armv7.apk`（32 位）、`universal.apk`（75MB，能用但体积近 3 倍）、`unsigned.tar.gz`（官方签名流水线中间产物）。

**交付后建议删掉 Release 和 Artifacts**（公开仓产物任何人可下载）。

---

## 9. 验收：干净测试（唯一可靠的方式）

> ⚠️ **覆盖安装测不出烧入是否生效。** 本地配置选项的优先级**高于**烧入值，旧配置会盖掉一切。上次因为覆盖安装浪费了一整轮排查。

### 9.1 运行时优先级（排查"烧入了却不生效"时先看这个）

`get_rendezvous_server()` — `libs/hbb_common/src/config.rs`
```
1. EXE_RENDEZVOUS_SERVER              ← Windows exe 文件名
2. option "custom-rendezvous-server"  ← RustDesk.toml，会盖掉烧入值
3. PROD_RENDEZVOUS_SERVER             ★ 烧入点
4. CONFIG2.rendezvous_server
5. RENDEZVOUS_SERVERS 常量
```

`get_api_server_()` — `src/common.rs`
```
1. option "api-server"                ← 会盖掉烧入值
2. option_env!("API_SERVER")          ★ 烧入点
3. 从 custom-rendezvous-server 推导 http://<host>:21114
4. "https://admin.rustdesk.com"
```

`get_key()` — `src/common.rs`
```
1. Windows exe 文件名
2. option "key"                       ← 会盖掉烧入值
3. RS_PUB_KEY 常量                    ★ 烧入点
```

`get_relay_server()` — `src/rendezvous_mediator.rs`
```
1. option "relay-server"
2. hbbs 下发
3. increase_port(ID服务器, +1)        ← 21116 → 21117
```

### 9.2 测试步骤（Windows 为例）

```
1. 退出 RustDesk，任务管理器确认无 rustdesk.exe
2. 备份并删除 %APPDATA%\RustDesk\config\RustDesk.toml
3. 安装新包
4. 启动后什么都不要填，检查：
   - 左下角 = 「就绪」
   - 设置 → 网络 四个字段全空（全空才说明值来自烧入）
   - 「使用 WebSocket」重新打开 ← 删配置会一起清掉它（默认关闭），
     而这条路径正是补丁 ③ 覆盖的，必须测到
```

配置文件位置（`Config::path()` 走 `directories_next::ProjectDirs::from("", org, "RustDesk").config_dir()`）：
- Windows `%APPDATA%\RustDesk\config\RustDesk.toml`
- Linux `~/.config/RustDesk/RustDesk.toml`
- macOS `~/Library/Preferences/RustDesk/config/RustDesk.toml`

**排查时优先要日志**：`%APPDATA%\RustDesk\log\`，里面记录实际连接的服务器地址和错误原因，比猜有用得多。

---

## 10. 故障速查

| 症状 | 根因 | 查/修 |
|---|---|---|
| 触发后 1 秒 `startup_failure`，0 job，无日志 | caller 缺 `permissions: contents: write` | 补丁 ⑧ |
| 构建全绿但 **Linux** 产物烧官方服务器 | `run-on-arch-action` 容器不继承 workflow env | 补丁 ⑥ |
| 构建全绿、二进制里值都对，但客户端卡「正在接入 RustDesk 网络...」，**手填相同 API 值就恢复** | 开了 WebSocket，`check_ws()` 绕过 `get_api_server()` 选了 `ws://` | 补丁 ③ |
| macOS 报「已损毁，移到废纸篓」，`xattr` 无效 | `build.py` 在签名后拷入 `service`，封印损坏 | 补丁 ⑦；急救见 8.macOS |
| 客户端连的是官方服务器，且**所有平台**都这样 | Secret 没设 / 补丁 ①② 没打 / 锚点失配后盲改 | 跑 3.0 自检 + 7.2 验证 |
| 只有某台机器不生效，别的机器正常 | 该机 `RustDesk.toml` 有残留选项（优先级高于烧入值） | 9.1 优先级表 + 9.2 干净测试 |
| Android 构建失败，缺图标 | `.gitignore` 吞掉了已跟踪的 png | 5.4 |
| `Build flatpak` 失败，`apt` 报 `404 Not Found` | Ubuntu 镜像源下架了容器索引里的包版本，**上游基础设施问题** | 忽略，或重跑一次 |
| run 结论 `failure` 但所有 job 都 success，且 `gh run rerun --failed` 报 `cannot be retried` | GitHub 侧偶发，部分 job 未展开进 job 图（`publish_unsigned` 等） | 重新触发一次即恢复；不影响安装包 |
| 下载的文件 `hdiutil` 报「无可装载的文件系统」/ `unzip` 报找不到目录 | 下载被截断 | 7.1 断点续传 + digest 校验 |
| `gh api` 改工作流文件报 `refusing to allow an OAuth App to create or update workflow` | token 缺 `workflow` scope | `gh auth refresh -h github.com -s workflow` |
| `git push` 卡在交互式认证 | `credential.helper` 为空 | 2.2 内联 helper |

---

## 11. 关键代码位置索引（1.5.3 行号，仅供定位参考，会随上游变动）

| 用途 | 位置 |
|---|---|
| 服务器优先级链 | `libs/hbb_common/src/config.rs:1049` `get_rendezvous_server()` |
| 服务器列表（喂给 mediator，**不补端口**） | `libs/hbb_common/src/config.rs:1072` `get_rendezvous_servers()` |
| 烧入点 ① PROD | `libs/hbb_common/src/config.rs:69` |
| 烧入点 ② 公钥 | `libs/hbb_common/src/config.rs:117` |
| 末级兜底常量（不要改） | `libs/hbb_common/src/config.rs:116` `RENDEZVOUS_SERVERS` |
| 端口常量 21116/21117/21118/21119 | `libs/hbb_common/src/config.rs:119-123` |
| API 解析 | `src/common.rs:1270` `get_api_server()` / `:1287` `get_api_server_()` |
| 烧入点 ④ API | `src/common.rs:1297`（补丁插入位置） |
| API 推导用的 custom server | `src/common.rs:1253` `get_custom_rendezvous_server()` |
| 端口 ±N / 补端口 | `libs/hbb_common/src/socket_client.rs` `increase_port()` / `check_port()` |
| Key 解析 | `src/common.rs:2163` `get_key()` |
| 握手验签 | `src/client.rs:1655` `secure_connection()` / `src/common.rs:2433` `decode_id_pk_dtls()` |
| WebSocket 端点转换 | `libs/hbb_common/src/websocket.rs:470` `check_ws()`（`:471` 有 `if !use_ws() return`） |
| 烧入点 ③ ws/wss | `libs/hbb_common/src/websocket.rs:524` |
| 自有服务器白名单（含 PROD） | `libs/hbb_common/src/websocket.rs:544-548` |
| WebSocket 总闸 | `libs/hbb_common/src/config.rs:2948` `use_ws()` / `:2928` `option2bool()`（`allow-*` 前缀 → `value=="Y"`，默认关） |
| 连接目标解析 | `libs/hbb_common/src/socket_client.rs:140` |
| 中继解析 | `src/rendezvous_mediator.rs:1367` `get_relay_server()` |
| mediator 启动 | `src/rendezvous_mediator.rs:317` `start_all()` |
| 配置改动触发重启 | `src/ipc.rs:795` `CheckIfRestart`（`:834` 监听 api-server） |
| API 下发黑名单（保护烧入值不被服务端覆盖） | `src/hbbs_http/sync.rs:295` `BLOCKED_CONFIG_OPTIONS` |
| custom.txt 验签（自建用不了） | `src/common.rs:2607` `read_custom_client()` |
| exe 文件名解析（仅 Windows） | `src/custom_server.rs:44` / `src/naming.rs` |
| **破坏 macOS 签名的那行** | `build.py:949` |
| macOS 重签脚本 | `.github/scripts/sign-macos-app.sh` |
| macOS 签名步骤的跳过条件 | `flutter-build.yml` `env.MACOS_P12_BASE64 != null` |
| `SIGN_BASE_URL` 的 `-2` 技巧（不要设这个 Secret） | `flutter-build.yml:63` |
| 4 处容器构建 | `flutter-build.yml:2037 / 2359 / 2651 / 2923` |
| 状态提示「正在接入 RustDesk 网络...」 | `src/lang/cn.rs:10` `connecting_status` ← `flutter/lib/desktop/pages/connection_page.dart:174` ← `status_num==0` ← 服务进程 `OnlineStatus`（反映 **ID 服务器**连通状态，不是 API） |

---

## 12. 上次成果基线（用于对照）

- 上游 `rustdesk/rustdesk` **1.5.3**，commit `95cca3f661f635b3cc85c04935fe8d96d8f432ef`（tag `1.5.3` 与 master HEAD 同一 commit）
- `hbb_common` pin `fae47e024dd83cef6aaecde0aa7d11731df11649`（= 其 `main` HEAD，无漂移），内联后 33 个文件
- 产物仓库 `XiaoBai-ZH/rustdesk-custom`，最终 commit `c0e574e`
- 有效构建 run `42749653883`：**29 job = 26 success + 2 skipped + 1 failure**（仅 flatpak，上游 apt 404）
- 上游文件总数 **1134**（用 5.4 的过滤命令，已排除 `.gitmodules` 和 `libs/hbb_common` 这个 gitlink 条目），其中 hbb_common 内联 **33** 个文件
- 本仓库跟踪 **1134** = 1134 − 1（删 `flutter-ci.yml`）+ 1（加本文档）
- 四平台烧入值全部验证通过，macOS 签名 `valid on disk` + `satisfies its Designated Requirement`
- 相对上游共 **9 处改动**（源码 4 + 工作流 5）+ 删 3 文件 + submodule 内联

核对命令（5.4 的清单比对，结果应精确如下）：
```bash
# 上游有、仓库无  →  .github/workflows/flutter-ci.yml
# 仓库有、上游无  →  docs/SELF-BUILD-CN.md
```

改动面核对：
```bash
diff -rq --exclude=.git --exclude=SELF-BUILD-CN.md <上游克隆> <本仓库工作树>
# 应精确列出：flutter-build.yml / flutter-nightly.yml / flutter-tag.yml /
#            libs/hbb_common/src/config.rs / libs/hbb_common/src/websocket.rs / src/common.rs  differ
#            .gitmodules / .github/workflows/flutter-ci.yml  only in upstream
```

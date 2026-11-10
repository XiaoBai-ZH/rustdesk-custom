# RustDesk 自建服务器客户端打包手册

> 基线：`rustdesk/rustdesk` **1.5.3**（commit `95cca3f661f635b3cc85c04935fe8d96d8f432ef`，tag `1.5.3` 与 master HEAD 同一 commit）
> 服务端：`lejianwen/rustdesk-server-s6:latest`（hbbs + hbbr + rustdesk-api）
> 仓库：`XiaoBai-ZH/rustdesk-custom`
> 本文所有结论均经代码核对或实测验证，标注了验证方式。

---

## 0. 下次打包的最短路径

如果仓库结构不变、只是上游出了新版本，按这 6 步走：

```bash
# 1. 拉新版上游（浅克隆即可）
git clone --depth 1 https://github.com/rustdesk/rustdesk.git /tmp/rd-new/rustdesk
cd /tmp/rd-new/rustdesk && git submodule update --init --depth 1 libs/hbb_common

# 2. 内联 submodule（本仓库不用 submodule）
rm -f .gitmodules libs/hbb_common/.git

# 3. 重新打 4 处源码补丁 + 3 处工作流补丁（见第 3 节，逐个核对锚点是否还在）

# 4. 删掉会白跑的文件
rm -f .github/workflows/flutter-ci.yml

# 5. 强制补回被 .gitignore 吞掉的已跟踪文件（关键，见坑 #3）
#    用上游 git 索引对比，逐个 git add -f

# 6. 推到仓库，触发 Flutter Nightly Build
```

**每次上游更新后必须重新核对的 4 个锚点字符串**（用 `grep -c` 确认各命中 1 次）：

```bash
grep -c 'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());' libs/hbb_common/src/config.rs
grep -c 'pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";'      libs/hbb_common/src/config.rs
grep -c 'let api_server = Config::get_option("api-server");'                                   libs/hbb_common/src/websocket.rs
grep -c 'SIGN_BASE_URL: "${{ secrets.SIGN_BASE_URL }}-2"'                                      .github/workflows/flutter-build.yml
```

任一返回 0 说明上游改动了该处，**必须重新阅读上下文再改，不能盲目套用旧补丁**。

---

## 1. 核心结论（先看这个，能省很多弯路）

### 1.1 上游的"自定义客户端"机制哪些能用、哪些不能

| 机制 | 位置 | 可用性 | 原因 |
|---|---|---|---|
| `playground.yml` 里的 `RS_PUB_KEY`/`RENDEZVOUS_SERVER`/`API_SERVER` secrets | `.github/workflows/playground.yml:26-28` | ❌ **死代码** | 全仓无任何代码读取；1.1.9→1.5.3 的 `build.rs` 逐个查过，从未消费。整个仓库唯一的编译期环境变量是 `src/rendezvous_mediator.rs:690` 的 `option_env!("TEST_TCP")` |
| `custom.txt` | `src/common.rs:2607` `read_custom_client()` | ❌ 自建用不了 | 需 Ed25519 验签，硬编码公钥 `5Qbwsde3unUcJBtrx9ZkvUmwFNoExHzpryHuPUdqlWM=`，只有 RustDesk 官方付费定制服务持有私钥。改 app 名/logo 也走这条路 |
| Windows exe 文件名 | `src/custom_server.rs:44` | ⚠️ 仅 Windows | `host=...,key=...,api=...,relay=...` 分支不验签，但 `get_license_from_exe_name()` 是 `#[cfg(windows)]` |
| **改源码常量 + `option_env!`** | 见第 3 节 | ✅ **本仓库采用** | 全平台生效，值从 GitHub Secrets 注入，代码里不含个人信息 |

### 1.2 服务器地址的运行时优先级（最重要的参考表）

**烧入值不是最高优先级**——本地配置文件里的选项会盖掉它。排查"烧入了却不生效"时先看这张表。

`get_rendezvous_server()` — `libs/hbb_common/src/config.rs:1049`
```
1. EXE_RENDEZVOUS_SERVER          ← Windows exe 文件名
2. option "custom-rendezvous-server"  ← RustDesk.toml
3. PROD_RENDEZVOUS_SERVER         ★ 烧入点
4. CONFIG2.rendezvous_server      ← 持久化
5. RENDEZVOUS_SERVERS 常量         ← "rs-ny.rustdesk.com"，故意保留作末级兜底
```

`get_api_server_()` — `src/common.rs:1287`
```
1. option "api-server"            ← RustDesk.toml
2. option_env!("API_SERVER")      ★ 烧入点（本仓库新增，上游没有）
3. 从 custom-rendezvous-server 推导 http://<host>:21114
4. "https://admin.rustdesk.com"
```

`get_key()` — `src/common.rs:2163`
```
1. Windows exe 文件名
2. option "key"
3. RS_PUB_KEY 常量                ★ 烧入点
```

`get_relay_server()` — `src/rendezvous_mediator.rs:1367`
```
1. option "relay-server"
2. hbbs 在打洞响应里下发（ph.relay_server / rr.relay_server）
3. increase_port(ID服务器, +1)    ← 21116 → 21117，自动推导
```
**中继服务器不需要烧入**，第 3 级兜底会自动从 ID 服务器端口 +1 得出。全仓无官方 relay 兜底（`relay.rustdesk.com` 只出现在 `#[cfg(test)]` 内）。

### 1.3 为什么烧入点是 `PROD_RENDEZVOUS_SERVER` 而不是 `RENDEZVOUS_SERVERS` 常量

三个独立理由，改常量会同时丢掉这三个：

1. **优先级更高**（第 3 级 vs 第 5 级）
2. **API 推导认它**：`get_custom_rendezvous_server()`（`src/common.rs:1253`）第 1263 行读 `PROD_RENDEZVOUS_SERVER`，不读那个常量
3. **WebSocket 认它**：`libs/hbb_common/src/websocket.rs:544-548` 的 `own_servers` 白名单显式包含 `PROD_RENDEZVOUS_SERVER`，注释写着 "Only servers this client is set up with, not the public fallback"。改常量的话 WebSocket 会因为不被识别为自有服务器而失效

---

## 2. 仓库结构：为什么不用 fork、不用 submodule

### 2.1 fork 不可用

`POST /repos/rustdesk/rustdesk/forks` 和 `POST /repos/rustdesk/hbb_common/forks` 都返回：

```
403 {"message":"You can't fork this repository at this time."}
```

已排除的原因（逐项实测）：
- 不是 scope 问题（`X-Accepted-Oauth-Scopes` 为空，token 有 `repo`）
- 不是 2FA 挑战（无 `X-GitHub-OTP` 响应头）
- 不是限流（`X-Ratelimit-Remaining` 充足）
- 不是账号写入受限（`PATCH` 自己仓库的描述成功）
- 目标仓库本身允许 fork（`rustdesk/hbb_common` 有 2526 个 fork，public、未 archive、未 disabled）

→ **账号级 fork 限制**，需联系 GitHub Support 解封。本仓库改用普通公开仓库绕开。

### 2.2 内联 hbb_common 而非改 `.gitmodules`

`libs/hbb_common` 是上游唯一的 submodule，且是**纯 path 依赖**：

```toml
# Cargo.toml:55
hbb_common = { path = "libs/hbb_common", features = ["webrtc"] }
# Cargo.toml:220 workspace members 也包含它
```

已核实构建脚本里**没有任何 `git submodule` 逻辑**，工作流的 `submodules: recursive` 在无 submodule 时是空操作。所以直接把它变成普通目录，收益：

- 不需要第二个仓库
- 不需要移动 gitlink 指针（网上教程最麻烦的一步，要用 GitHub API 的 trees→commits→refs 三段式）
- 补丁直接改文件，`git diff` 一目了然

### 2.3 必须公开

GitHub 规则：**公开仓库的 fork 永远是公开的，无法改为私有**（"All forks of public repositories are public. You cannot change the visibility of a fork."）。普通仓库虽然可以私有，但：

| | 公开仓 | 私有仓（Free 计划） |
|---|---|---|
| 一次完整构建成本 | **$0** | **≈$11.13 / 月度额度 $12** |
| 产物存储 | 不限 | 500MB 上限（一次构建的产物远超） |

成本按上游暖缓存实测时长 × 官方单价算出（Linux $0.006/min、Windows $0.010/min、macOS $0.062/min）：

```
Windows  146min × $0.010 = $1.46
macOS    121min × $0.062 = $7.50   ← 占 67%
Linux    362min × $0.006 = $2.17
合计                    = $11.13  → 占 Free 额度 93%
```

且这是**暖缓存**成绩，新仓库首次构建缓存全冷会更久，极可能中途超额失败。

**公开仓的隐私影响**：仓库内容不含任何个人信息（补丁是通用的，服务器地址和公钥在 Secrets 里，Secrets 只写不可读，API 无任何端点能取回值）。唯一暴露面是 Release 产物可被任何人下载并 `strings` 出服务器地址——但这与分发给被控端的安装包本身等价，且下载完删掉 Release 即可消除。

---

## 3. 必须打的补丁（共 7 处）

### 3.1 源码 4 处

**① `libs/hbb_common/src/config.rs` — `PROD_RENDEZVOUS_SERVER`（上游第 69 行）**

```rust
// 原
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new("".to_owned());

// 改为
pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new(
    match option_env!("RENDEZVOUS_SERVER") {
        Some(server) if !server.is_empty() => server,
        _ => "",
    }
    .to_owned()
);
```

**② `libs/hbb_common/src/config.rs` — `RS_PUB_KEY`（上游第 117 行）**

```rust
// 原
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";

// 改为
pub const PUBLIC_RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
pub const RS_PUB_KEY: &str = match option_env!("RS_PUB_KEY") {
    Some(key) if !key.is_empty() => key,
    _ => PUBLIC_RS_PUB_KEY,
};
```

**③ `libs/hbb_common/src/websocket.rs` — ws/wss 判定（上游第 524 行，在 `check_ws()` 内）**

```rust
// 原
let protocol = if is_domain {
    let api_server = Config::get_option("api-server");
    if api_server.starts_with("https") { "wss" } else { "ws" }
} else { "ws" };

// 改为
let protocol = if is_domain {
    let api_server = Config::get_option("api-server");
    let api_server = if api_server.is_empty() {
        option_env!("API_SERVER").unwrap_or("").to_owned()
    } else {
        api_server
    };
    if api_server.starts_with("https") { "wss" } else { "ws" }
} else { "ws" };
```

**不打这处会怎样**：开启「使用 WebSocket」时，`check_ws()` 直接读原始选项而不经过 `get_api_server()`，选项为空 → 选 `ws://`（明文 80 端口）→ HTTPS-only 反代连不上 → 客户端永远卡在「正在接入 RustDesk 网络...」。**本次实际踩到，见坑 #6。**

**④ `src/common.rs` — `get_api_server_()`（上游第 1294-1297 行之间插入）**

```rust
    if !api.is_empty() {
        return api.to_owned();
    }
+   if let Some(baked) = option_env!("API_SERVER") {
+       if !baked.is_empty() {
+           return baked.to_owned();
+       }
+   }
    let s0 = get_custom_rendezvous_server(custom);
```

**为什么必须打**：不打的话 API 会走第 3 级推导 `http://<host>:21114`。如果 21114 没有对公网开放（走 HTTPS 反代的常见情况），API 全部失效。

### 3.2 工作流 3 处

**⑤ `.github/workflows/flutter-build.yml` — env 块（上游第 63 行后）**

```yaml
  SIGN_BASE_URL: "${{ secrets.SIGN_BASE_URL }}-2"
+ RS_PUB_KEY: "${{ secrets.RS_PUB_KEY }}"
+ RENDEZVOUS_SERVER: "${{ secrets.RENDEZVOUS_SERVER }}"
+ API_SERVER: "${{ secrets.API_SERVER }}"
```

注意 env 块的行号会随上游变动，1.5.3 是 23–64 行（`env:` 在 23，`jobs:` 在 65）。网上教程说的"第 49 行"已失效。

**⑥ `.github/workflows/flutter-build.yml` — 4 处 `run-on-arch-action` 加 `env:` 输入**

上游有 4 处（1.5.3 行号 2037 / 2359 / 2651 / 2923），分属 `build-rustdesk-linux`、`build-rustdesk-linux-drm`、`build-rustdesk-linux-sciter`、`build-flatpak`。四处的模式完全一致，可一次性替换：

```yaml
          githubToken: ${{ github.token }}
+         env: |
+           RS_PUB_KEY: ${{ secrets.RS_PUB_KEY }}
+           RENDEZVOUS_SERVER: ${{ secrets.RENDEZVOUS_SERVER }}
+           API_SERVER: ${{ secrets.API_SERVER }}
          setup: |
```

替换前先确认匹配数：
```bash
python3 -c "s=open('.github/workflows/flutter-build.yml').read(); \
print(s.count('          githubToken: \${{ github.token }}\n          setup: |\n'), \
      s.count('run-on-arch-action'))"
# 两个数字必须相等，否则说明上游结构变了，要逐个人工确认
```

**不打这处会怎样**：Linux 全部产物（deb/rpm/AppImage/flatpak/archlinux）静默烧入官方服务器。**本次实际踩到，见坑 #5。**

**⑦ `.github/workflows/flutter-build.yml` — macOS 打包前 ad-hoc 重签**

在 `create unsigned dmg` 步骤的 `run:` 开头插入：

```yaml
      - name: create unsigned dmg
        if: env.UPLOAD_ARTIFACT == 'true'
        run: |
+         bash ./.github/scripts/sign-macos-app.sh \
+           ./flutter/build/macos/Build/Products/Release/RustDesk.app \
+           - \
+           ./flutter/macos/Runner/Release.entitlements
          CREATE_DMG="$(command -v create-dmg)"
          ...
```

**不打这处会怎样**：dmg 里的 app 签名损坏，macOS 报「已损毁，移到废纸篓」，且 `xattr` 清隔离属性无效。**本次实际踩到，见坑 #7。**

复用上游自己的脚本而不是自己写 `codesign`，因为它已经处理了正确顺序（Frameworks 内的 Mach-O → `.framework` 目录 → `Contents/MacOS/service` → 带 entitlements 签 bundle），末尾还自带 `codesign --verify --deep --strict` 自检，配合 `set -euo pipefail`，签名坏了会让步骤直接失败而不是产出坏包。传 `-` 时脚本会自动跳过 `--timestamp`。

### 3.3 需要删除的文件

| 文件 | 原因 |
|---|---|
| `.gitmodules` | hbb_common 已内联 |
| `libs/hbb_common/.git` | 同上（它是个 gitfile，内容 `gitdir: ../../.git/modules/libs/hbb_common`） |
| `.github/workflows/flutter-ci.yml` | `on: push: branches: [master]`，每次推送白跑一次全平台构建且 `upload-artifact: false` 什么都不产出 |

### 3.4 不要动的东西

**`SIGN_BASE_URL` 这个 Secret 千万不要设。** `flutter-build.yml:63` 是 `"${{ secrets.SIGN_BASE_URL }}-2"`，末尾硬拼了 `-2`。不设置时值正好是 `-2`，所有 `!= '-2'` 的签名步骤自动跳过——这是上游故意让 fork 能直接跑的设计。设了反而变成 `你的值-2`，签名步骤会执行并连 `http://localhost:5000` 失败（`res/job.py:58` 的兜底）。

注意 `playground.yml:30` 没有这个 `-2` 后缀，所以**不要跑 `playground.yml`**（它还是 macos-13/sciter 时代的旧矩阵，且不调用 `flutter-build.yml`）。

**`RENDEZVOUS_SERVERS` 常量（`config.rs:116`）保留不动。** 它在优先级链第 5 级，永远轮不到；保留的好处是域名失效时客户端不会崩在空地址上。产物里能 `strings` 到 `rs-ny.rustdesk.com` 是预期的，不是 bug。

---

## 4. GitHub 侧配置

### 4.1 Secrets（Settings → Secrets and variables → Actions → **Secrets** 标签页）

必须是 Secret 不是 Variable，工作流读的是 `secrets.`。

| 名称 | 值 | 说明 |
|---|---|---|
| `RENDEZVOUS_SERVER` | `<域名>:21116` | 带不带端口都能工作。带端口更精确：`increase_port("host:21116", +1)` 直接得出 `host:21117`，不依赖二级兜底 |
| `RS_PUB_KEY` | 服务端 `/data/id_ed25519.pub` 内容 | 见 4.2 的格式校验 |
| `API_SERVER` | `https://<域名>` | 反代到容器 21114 时用这个。**不设则会推导出 `http://<域名>:21114`，若该端口未对公网开放就连不上** |

不需要设 `RELAY_SERVER`——中继由 hbbs 下发或从 ID 服务器端口 +1 推导。

### 4.2 取公钥并校验格式

```bash
docker exec $(docker ps -qf ancestor=lejianwen/rustdesk-server-s6:latest) cat /data/id_ed25519.pub
```

`genkeypair` 的输出长这样，**只要 Public Key 那行**：
```
Public Key:  8BLLhtzUBU/XKAH4mep3p+IX4DSApe7qbAwNH9nv4yA=     ← 44 字符，单个 = 结尾
Secret Key:  egAVd44u33ZEUIDTtksGcHeVeAwywarEdHmf99KM5ajwEsuG3NQFT9coAfiZ6nen4hfgNICl7upsDA0f2e/jIA==   ← 88 字符，== 结尾，不要这个
```

贴之前校验（`get_rs_pk()` 是先 base64 解码再取 32 字节，多一个空白字符就解码失败并**静默回落官方公钥**，症状和填错一样）：

```bash
python3 -c "
import base64,sys
k='粘贴你的公钥'
assert len(k)==44, f'长度 {len(k)}，应为 44'
assert k==k.strip(), '含首尾空白'
raw=base64.b64decode(k, validate=True)
assert len(raw)==32, f'解码后 {len(raw)} 字节，应为 32'
print('✓ 合法 Ed25519 公钥')
"
```

**密钥漂移风险**：s6 镜像只在 `/data/id_ed25519{,.pub}` 缺失时才重新生成。只要 `/data` 卷持久化就稳定；但若哪天重建容器且没挂原 `/data`，密钥会重新生成，**所有已分发的包同时失联**，必须重新打包。建议把公钥另存一份。

### 4.3 仓库设置

| 项 | 值 | 说明 |
|---|---|---|
| 可见性 | Public | 见 2.3 |
| Actions | 默认已启用 | 普通仓库不像 fork 那样需要手动开启 |
| `default_workflow_permissions` | **保持 `read`** | 不要为了让构建通过而放宽到 write，改用 4.4 的 caller 级授权 |

### 4.4 caller 级权限（必须）

`flutter-build.yml` 的 `generate-sbom` 和 `publish-signed-update-metadata` 声明了 `permissions: contents: write`。**可复用工作流的 token 不能超出调用方授予的范围**，仓库默认是 read，所以两个 caller 都必须显式加：

```yaml
# flutter-nightly.yml 和 flutter-tag.yml 各自的 job 下
jobs:
  run-flutter-nightly-build:
    uses: ./.github/workflows/flutter-build.yml
+   permissions:
+     contents: write
    secrets: inherit
```

不加的后果：**整个 run 立刻 `startup_failure`，0 个 job，1 秒结束，GitHub 不给任何错误注解**。见坑 #4。

### 4.5 删掉 nightly 的定时触发

`flutter-nightly.yml` 上游带 `schedule: cron: "0 0 * * *"`，会每天午夜自动跑一次完整构建并覆盖 Release。本仓库已删除，只保留 `workflow_dispatch`。如果想要每日构建再加回来。

---

## 5. 触发构建与取包

### 5.1 触发

```bash
gh workflow run "Flutter Nightly Build" -R <owner>/rustdesk-custom --ref master
```

或推 tag 触发 `Flutter Tag Build`（tag 正则要求 `-` 后是数字，`1.5.3-1` 匹配，`1.5.3-custom` **不匹配**）：
```bash
git tag 1.5.3-1 && git push origin 1.5.3-1
```

**不要跑 `playground.yml`**（见 3.4）。

### 5.2 监控

```bash
RID=<run_id>
gh api "/repos/<owner>/rustdesk-custom/actions/runs/$RID/jobs?per_page=100" \
  --jq '[.jobs[]|.conclusion // .status]|group_by(.)|map("\(.[0])=\(length)")|join("  ")'
```

正常形态：**29 个 job = 27 success + 2 skipped**（`build-rustdesk-web` 是 `if: False`，`publish-signed-update-metadata` 的 `if` 含 `github.repository == 'rustdesk/rustdesk'`）。耗时约 85 分钟（暖缓存）。

### 5.3 产物位置

**Release `nightly`**：Windows `.exe`/`.msi`、macOS `.dmg`、Android `.apk`、Linux `.deb`/`.rpm`/`.AppImage`/`.flatpak`/`.pkg.tar.zst`
**Artifacts**：`rustdesk-unsigned-windows-*`（原始未打包目录）、`rustdesk-<ver>-<arch>.deb`、`librustdesk.so.*` 等

```bash
gh release download nightly -R <owner>/rustdesk-custom \
  -p 'rustdesk-1.5.3-x86_64.exe' -p 'rustdesk-1.5.3-aarch64.dmg' \
  -p 'rustdesk-1.5.3-x86_64.dmg' -p 'rustdesk-1.5.3-aarch64.apk' -D ~/Downloads
```

### 5.4 选哪个文件

| 设备 | 文件 | 备注 |
|---|---|---|
| Windows x64 | `rustdesk-<ver>-x86_64.exe` | 自解压便携版，运行后可在界面内点「安装」。要静默部署/组策略用 `.msi` |
| Windows ARM | `rustdesk-<ver>-aarch64.exe` | |
| macOS Apple 芯片 | `rustdesk-<ver>-aarch64.dmg` | `uname -m` 输出 `arm64` |
| macOS Intel | `rustdesk-<ver>-x86_64.dmg` | `uname -m` 输出 `x86_64` |
| Android arm64 | `rustdesk-<ver>-aarch64.apk` | 小米 14 等骁龙 8 系用这个 |
| Linux x64 | `rustdesk-<ver>-x86_64.deb` / `.rpm` / `.AppImage` | |

**不要下**：`*-sciter.*`（旧版 Sciter UI）、`x86_64.apk`（模拟器）、`armv7.apk`（32 位）、`universal.apk`（75MB，全架构，能用但体积近 3 倍）、`unsigned.tar.gz`（官方签名流水线中间产物）。

---

## 6. 验证产物（每平台方法不同，别用错）

**通用判据**：你的域名/公钥/API 应命中，官方公钥 `OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=` 和 `admin.rustdesk.com` 应为 **0 命中**（是被替换，不是并存）。`rs-ny.rustdesk.com` 有命中是**正常的**（末级兜底常量，故意保留）。

### 6.1 Linux（最简单，先验这个）

```bash
mkdir ex && cd ex && ar x ../rustdesk-*-x86_64.deb && tar xf data.tar.*
B=usr/share/rustdesk/lib/librustdesk.so     # 约 56MB，真正的主体
strings -a "$B" | grep -c "<你的域名>"
```

⚠️ `usr/share/rustdesk/rustdesk` 只有 44KB，是启动器，**在它上面 strings 什么都搜不到**。

### 6.2 Android

```bash
unzip -o -q rustdesk-*-aarch64.apk 'lib/*' -d ax
strings -a ax/lib/arm64-v8a/librustdesk.so | grep -c "<你的域名>"
```

### 6.3 Windows

Release 里的 `.exe` 是自解压包、`.msi` 内含压缩 CAB（`MSCF` 魔数），**直接 strings 连官方默认值都搜不到**。必须取未打包的 artifact：

```bash
AID=$(gh api "/repos/<owner>/<repo>/actions/runs/<RID>/artifacts?per_page=100" \
      --jq '.artifacts[]|select(.name=="rustdesk-unsigned-windows-x86_64")|.id')
gh api "/repos/<owner>/<repo>/actions/artifacts/$AID/zip" > win.zip
unzip -q win.zip -d winx
strings -a winx/librustdesk.dll | grep -c "<你的域名>"
```

### 6.4 macOS（有陷阱）

```bash
hdiutil attach -nobrowse -readonly rustdesk-*-aarch64.dmg
B="/Volumes/<卷名>/RustDesk.app/Contents/Frameworks/liblibrustdesk.dylib"
strings -a "$B" | grep -c "<你的公钥>"        # 公钥 44 字符，能直接搜到
strings -a "$B" | grep -c "<你的域名>"        # ⚠️ 可能是 0，见下
```

⚠️ **macOS 上域名字符串可能搜不到，这不代表没烧进去。** x86_64/arm64 的 codegen 会把短字符串常量拆成 8 字节 `movabs` 立即数内联进指令流，不放在 `.rodata`。用这个方式确认：

```bash
strings -a "$B" | grep -o "[a-z:/._0-9-]*<域名片段>[a-z:/._0-9-]*" | sort -u
```

如果看到类似 `...hbb_common::websocketrelay-server/ws/id/ws/relayapi-serverhttps://<你的域名>websocket-port` 这样的分片串，说明值在，且邻域符号能告诉你**是哪个模块在引用它**——这是判断补丁是否真正生效的可靠信号（比数命中次数强得多）。

### 6.5 macOS 签名校验（必做）

```bash
codesign --verify --deep --strict --verbose=2 "/Volumes/<卷名>/RustDesk.app"
```

必须输出 `valid on disk` + `satisfies its Designated Requirement`。若报 `a sealed resource is missing or invalid` 并附 `file added: .../Contents/MacOS/service`，说明 3.2⑦ 的重签补丁没生效。

---

## 7. 安装注意事项

### macOS

即使签名有效，因为**没有 Developer ID 签名也没有公证**，`spctl -a -vvv -t install` 仍会返回 `rejected`。处理方式：

**推荐**：用终端 `cp` 安装（实测**不会**带上 `com.apple.quarantine`），然后直接打开：
```bash
hdiutil attach ~/Downloads/rustdesk-<ver>-aarch64.dmg
cp -R "/Volumes/<卷名>/RustDesk.app" /Applications/
hdiutil detach "/Volumes/<卷名>"
open /Applications/RustDesk.app
```

**如果是在访达里拖的**（隔离属性会跟过去），补一条：
```bash
xattr -cr /Applications/RustDesk.app
```

**如果拿到的是没打 3.2⑦ 补丁的旧 dmg**（签名已坏），本地修复：
```bash
codesign --force --deep --sign - /Applications/RustDesk.app
```
实测这一条就够，`--deep` 会覆盖 `Contents/MacOS/service`。副作用：不带 `--entitlements` 会丢掉 entitlements 和 hardened runtime 标志（`flags` 从 `0x10002(adhoc,runtime)` 变 `0x2(adhoc)`）。对非沙箱 ad-hoc 应用无害——`Release.entitlements` 里 `app-sandbox` 本来就是 false，`network.client`/`audio-input` 只约束沙箱应用，`allow-jit` 只在 hardened runtime 下才需要。要完整保留就用 3.2⑦ 那个脚本加 `--options runtime --entitlements`。

要彻底免掉这些步骤需要 Apple Developer 账号（$99/年）+ 三个 Secret：`MACOS_P12_BASE64`、`MACOS_P12_PASSWORD`、`MACOS_CODESIGN_IDENTITY`，工作流里的 codesign + `rcodesign notary-submit` 公证会自动启用。

### Windows

未签名 → SmartScreen 弹「Windows 已保护你的电脑」→ 点「更多信息 → 仍要运行」。

### Android

APK 用 Flutter 的调试密钥签名（`CN=Android Debug, O=Android, C=US`），**可以正常安装**，但：
- 与官方版签名不一致，**必须先卸载已有的 RustDesk**，否则报「应用未安装」
- 需允许「安装未知来源应用」
- 不能上架应用商店

校验签名：
```bash
unzip -p rustdesk-*.apk META-INF/CERT.RSA > /tmp/c.rsa
openssl pkcs7 -inform DER -in /tmp/c.rsa -print_certs -noout
```

---

## 8. 踩坑清单

按踩到的顺序。每条都标注了症状、根因、解法、以及**当初为什么没提前发现**。

### 坑 #1 — 上游那三个 Secret 是死代码

- **症状**：按网上教程设了 `RS_PUB_KEY`/`RENDEZVOUS_SERVER`/`API_SERVER` 三个 Secret 就跑工作流，产出的包连的是官方服务器
- **根因**：`playground.yml:26-28` 声明了它们，注释还写着 "To make a custom build with your own servers set the below secret values"，但**全仓无任何代码读取**。1.1.9/1.2.3/1.3.0/1.3.5/1.4.0/1.4.6 的 `build.rs` 逐个查过，从未消费。真正干活的 `flutter-build.yml` 连声明都没有
- **解法**：必须自己打 `option_env!` 补丁（第 3.1 节）
- **为何没早发现**：注释极具误导性，且这个说法在社区流传很广

### 坑 #2 — fork 被账号级限制拒绝

- **症状**：`POST /repos/rustdesk/rustdesk/forks` → `403 You can't fork this repository at this time`
- **根因**：账号级限制，非权限/2FA/限流/仓库设置问题（逐项排除见 2.1）
- **解法**：改用普通公开仓库 + 内联 submodule
- **附带收益**：省掉了移动 gitlink 指针这一最麻烦的步骤

### 坑 #3 — `.gitignore` 静默吞掉 123 个已跟踪文件

- **症状**：无（构建时才爆）——Android 构建失败，缺启动图标
- **根因**：全新 `git init` + `git add -A` 时，`.gitignore` 里的 `*.png` 等规则会吞掉上游**已跟踪**的文件。实测漏掉 123 个，含 `flutter/android/app/src/main/res/mipmap-*/ic_launcher.png` 等全部 Android 图标
- **解法**：用上游 git 索引对比后 `git add -f` 补回
  ```bash
  cd 上游克隆 && git ls-files > /tmp/up_main.txt
  cd libs/hbb_common && git ls-files | sed 's|^|libs/hbb_common/|' > /tmp/up_sub.txt
  # 合并、去掉 .gitmodules，与新仓库 git diff --cached --name-only 对比
  comm -23 /tmp/up_all.txt /tmp/new_all.txt | xargs -d '\n' git add -f
  ```
- **为何没早发现**：`git add -A` 不报任何警告，文件数看起来"差不多"

### 坑 #4 — 可复用工作流的权限传播导致 `startup_failure`

- **症状**：`workflow_dispatch` 触发后 **1 秒结束，`conclusion=startup_failure`，0 个 job，无日志，无 check run，GitHub 不给任何错误注解**
- **根因**：`flutter-build.yml` 的 `generate-sbom` 和 `publish-signed-update-metadata` 声明了 `permissions: contents: write`。可复用工作流的 `GITHUB_TOKEN` 不能超出调用方授予的范围，而仓库 `default_workflow_permissions` 是 `read`，`flutter-nightly.yml` 又没写 `permissions` 块 → 整个 run 在展开 job 图时被拒
- **解法**：在两个 caller 加 `permissions: contents: write`（4.4）。**不要放宽仓库级默认**
- **定位过程**（供参考，下次遇到同类问题可复用这套排除法）：
  1. 撤回自己的改动重跑 → 仍失败 ⇒ 不是我的改动
  2. pyyaml 解析全部工作流 → 通过 ⇒ 不是语法
  3. 列出含 deleted 状态的工作流 → 全 active ⇒ 不是孤儿工作流拦截
  4. 最小 caller/callee 对 → 成功 ⇒ 可复用机制本身没问题
  5. 三层嵌套 L1→L2→L3 → 成功 ⇒ 不是嵌套深度
  6. 矩阵实测 `ubuntu-22.04-arm`/`windows-11-arm`/`macos-15-intel` → 全成功 ⇒ 不是受限 runner
  7. 程序化校验 `needs`/`uses` 引用 → 无悬空 ⇒ 不是 job 图结构
  8. **最小复现：callee 要 `contents: write` + caller 无 permissions 块 → 精确复现 0-job startup_failure** ⇒ 命中
- **为何没早发现**：GitHub 对这个失败模式不提供任何诊断信息，`referenced_workflows` 还显示三个被引用工作流都解析成功了，极具误导性

### 坑 #5 — `run-on-arch-action` 容器不继承 workflow env

- **症状**：构建全绿（27 job success），但 **Linux 所有产物烧的是官方服务器**；Windows/macOS/Android 正常
- **根因**：4 处 `run-on-arch-action` 在 `ubuntu18.04` 容器内编译，`dockerRunArgs` 只有 `--volume` 没有 `-e`。工作流级 `env:` 不会自动跨进容器
- **解法**：给 4 处都加 `env:` 输入（3.2⑥）。该 action 的 `action.yml` 明确支持："Environment variables to propagate to the container... available in both run and setup"
- **为何没早发现**：`grep "container:"` 找不到——它不是 job 级 `container:`，是 step 里的 action。线索是 job 步骤列表里那个 `Pull rustdesk/makepkg:latest`，以及 `build-rustdesk-linux` 的 step 里有 `distro: ubuntu18.04`
- **附带确认**：archlinux/AppImage/rpm 不需要单独修——`res/PKGBUILD` 只有 `package()` 没有 `build()`，只是 `cp -r` 已构建的 bundle；AppImage 下载 `.deb` 重打包；`rpmbuild` 在同一容器内

### 坑 #6 — `websocket.rs` 绕过 `get_api_server()`

- **症状**：产物二进制里域名、公钥、API 全部正确烧入（已逐平台 strings 验证），但客户端卡在「正在接入 RustDesk 网络...」。**手动填入与烧入值完全相同的 API 地址后立即恢复**
- **根因**：`check_ws()`（`libs/hbb_common/src/websocket.rs:524`）直接读原始 `api-server` 选项决定 `ws`/`wss`，不经过 `get_api_server()`。选项为空 → `ws://`（明文 80）→ HTTPS-only 反代连不上。仅在 `allow-websocket = "Y"` 时可达（该选项以 `allow-` 开头，`option2bool` 判定为 `value == "Y"`，默认关闭）
- **解法**：3.1③
- **诊断关键**：「填入相同的值却改变行为」这个悖论直接指向"有代码路径绕过了统一的取值函数"。顺着 `grep 'get_option("api-server")'` 逐个排查调用点即可定位
- **注意**：`connecting_status` 这个提示反映的是 **ID 服务器连通状态**（`connection_page.dart:174` ← `status_num==0` ← 服务进程 `OnlineStatus`），不是 API 状态，所以症状有误导性

### 坑 #7 — macOS bundle 签名被 `build.py` 破坏

- **症状**：dmg 完整（SHA256 与 GitHub 权威值一致），但 macOS 报「已损毁，无法打开，应该移到废纸篓」。`xattr -rd com.apple.quarantine` **无效**
- **根因**：`build.py:949` 在 Xcode 完成 ad-hoc 签名**之后**才把 9.97MB 的 `service` 拷进 `Contents/MacOS/`：
  ```python
  system2('cp -rf ../target/release/service ./build/macos/Build/Products/Release/RustDesk.app/Contents/MacOS/')
  ```
  封印清单不含它 → `codesign --verify --deep --strict` 报 `a sealed resource is missing or invalid` + `file added: .../Contents/MacOS/service`。上游靠后续的 "Codesign app and create signed dmg" 步骤重签，但那步 gated on `MACOS_P12_BASE64`，自建时被跳过
- **解法**：3.2⑦，在 create-dmg 前用上游脚本 ad-hoc 重签
- **关键教训**：**「文件完整」≠「包可用」**。SHA256 匹配只排除传输损坏，排除不了签名损坏。两者在 macOS 上的报错文字一模一样
- **诊断方法**：`codesign --verify --deep --strict --verbose=4` 会直接打印 `file added: <路径>`，一眼定位

### 坑 #8 — git 身份泄露工作邮箱

- **症状**：无（提交后才发现）
- **根因**：全局 `user.email` 是工作邮箱，提交到公开仓库会永久写进公开 commit 历史
- **解法**：用 GitHub noreply 邮箱，且**用内联 `-c` 而非改全局配置**：
  ```bash
  git -c user.name="<GitHub用户名>" -c user.email="<id>+<用户名>@users.noreply.github.com" commit ...
  ```
  noreply 邮箱可从 `gh api user/emails` 取

### 坑 #9 — 本地 `git push` 无凭据

- **症状**：全局 `credential.helper` 为空，HTTPS 推送会卡在交互式认证
- **解法**：内联凭据助手，不改全局配置：
  ```bash
  git -c credential.helper='!gh auth git-credential' push -u origin master
  ```
  （需 token 有 `repo` scope；改工作流文件还需 `workflow` scope）

### 坑 #10 — flatpak 的 apt 404（上游问题，非本仓库）

- **症状**：`Build flatpak x86_64` 失败，`exit code: 100`
- **根因**：容器内缓存的 apt 索引指向 `perl 5.34.0-3ubuntu1.10`，该版本已被 Ubuntu 镜像源下架：
  ```
  E: Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/p/perl/perl-base_5.34.0-3ubuntu1.10_amd64.deb  404 Not Found
  ```
  叠加 `run-on-arch-action` 的 GHCR 容器缓存在新仓库不存在（`ghcr.io/<owner>/<repo>/run-on-arch-...:latest: not found`），回退到现建容器时撞上
- **解法**：忽略。只影响 flatpak 一种格式，同用容器的 deb/rpm 不受影响。重跑一次通常就好（镜像源状态会变）

### 坑 #11 — run 结论 `failure` 但所有 job 都成功

- **症状**：26 success + 1 skipped，无任何 job 失败，但 run `conclusion=failure`；`gh run rerun --failed` 报 `This workflow run cannot be retried`
- **根因**：`publish_unsigned` 和 `publish-signed-update-metadata` 两个 job **未被展开进 job 图**（27 job vs 正常 29），尽管它们的三个父 job 全部 success。已排除 concurrency 冲突、YAML、权限、runner 标签。**GitHub 侧偶发异常，未确证根因**
- **解法**：重新触发一次即恢复（下一轮 29 job 全正常）。唯一影响是 `rustdesk-<ver>-unsigned.tar.gz` 停留在上一轮，那是官方签名流水线的中间产物，自建用不到

### 坑 #12 — 大文件下载被截断

- **症状**：`gh release download` / `gh api .../zip` 在 20MB、26.5MB 处停滞，或只拿到 1.4MB；`hdiutil attach` 报「无可装载的文件系统」
- **根因**：网络中间层截断（观察到 `stream error: PROTOCOL_ERROR`，且截断点正好是 `20971520 = 20×1024×1024` 这种整齐边界）
- **解法**：用 `curl -C -` 断点续传，循环重试，并**用 GitHub 的权威 digest 校验**：
  ```bash
  gh api "/repos/<owner>/<repo>/releases/tags/nightly" \
    --jq '.assets[]|select(.name=="<文件名>")|{size:.size,digest:.digest}'
  for i in $(seq 1 10); do
    [ "$(stat -f%z <文件>)" -ge <期望大小> ] && break
    curl -sL -C - -o <文件> "<下载URL>"
  done
  shasum -a 256 <文件>   # 与 digest 比对
  ```
  ⚠️ **不要并发跑多个续传任务写同一个文件**，会互相写坏（本次踩过，文件变成 31901904 > 31348944）

---

## 9. 上游更新后的同步流程

### 9.1 判断要不要重新打包

```bash
# 上游最新版本
curl -s https://raw.githubusercontent.com/rustdesk/rustdesk/master/Cargo.toml | grep '^version'
# 上游 hbb_common 的 pin
git ls-remote https://github.com/rustdesk/rustdesk.git master
```

### 9.2 同步步骤

```bash
# 1. 拉新版
rm -rf /tmp/rd-new && mkdir -p /tmp/rd-new && cd /tmp/rd-new
git clone --depth 1 https://github.com/rustdesk/rustdesk.git upstream
cd upstream && git submodule update --init --depth 1 libs/hbb_common

# 2. 核对 4 个锚点（见第 0 节）。任一为 0 就必须重新阅读上下文

# 3. 内联 + 打补丁 + 删文件（见第 3 节）

# 4. 关键：补回被 .gitignore 吞掉的文件（坑 #3）
#    用 upstream 的 git ls-files 作为权威清单对比

# 5. 推到仓库
```

### 9.3 需要同步更新的 Secret

一般不用改。除非：
- 服务端密钥对重新生成过 → 更新 `RS_PUB_KEY`
- 域名/端口/反代方案变了 → 更新 `RENDEZVOUS_SERVER` / `API_SERVER`

改 Secret 后**建议清一次 Actions 缓存**（Settings → Actions → Caches）。实测 cargo 会追踪 `option_env!` 依赖并在值变化时重编（源码 mtime 不变也会重编），所以理论上不需要，但清一次更保险。

### 9.4 验证清单（每次必做）

- [ ] run 结论 `success`，29 job = 27 success + 2 skipped
- [ ] Linux `.deb` 内 `librustdesk.so`：域名/公钥/API 命中，官方公钥与 `admin.rustdesk.com` 为 0
- [ ] Android `.apk` 内 `lib/<abi>/librustdesk.so`：同上
- [ ] Windows artifact 内 `librustdesk.dll`：同上
- [ ] macOS `.dmg` 内 `liblibrustdesk.dylib`：公钥命中、官方公钥 0；域名用分片搜（6.4）
- [ ] **macOS `codesign --verify --deep --strict` 通过**（坑 #7 的回归检查）
- [ ] 真机干净测试：删 `RustDesk.toml` → 装包 → 四字段全空 → 是否「就绪」

---

## 10. 干净测试流程（唯一可靠的验收方式）

**覆盖安装测不出烧入是否生效**——本地配置选项优先级高于烧入值（见 1.2），旧配置会盖掉一切。本次就因为覆盖安装浪费了一轮排查。

Windows：
```
1. 退出 RustDesk，任务管理器确认无 rustdesk.exe
2. 备份并删除 %APPDATA%\RustDesk\config\RustDesk.toml
3. 安装新包
4. 启动后什么都不要填，检查：
   - 左下角 = 「就绪」
   - 设置 → 网络 四个字段全空（全空才说明值来自烧入）
   - 「使用 WebSocket」重新打开（删配置会一起清掉，默认关闭；这条路径正是坑 #6）
```

配置文件位置（`Config::path()` 走 `directories_next::ProjectDirs::from("", org, "RustDesk").config_dir()`）：
- Windows：`%APPDATA%\RustDesk\config\RustDesk.toml`
- Linux：`~/.config/RustDesk/RustDesk.toml`
- macOS：`~/Library/Preferences/RustDesk/config/RustDesk.toml`

日志在 `%APPDATA%\RustDesk\log\`，记录实际连接的服务器地址和错误原因，排查时比猜有用得多。

---

## 11. 关键代码位置索引（1.5.3 行号）

| 用途 | 位置 |
|---|---|
| 服务器优先级链 | `libs/hbb_common/src/config.rs:1049` `get_rendezvous_server()` |
| 服务器列表（喂给 mediator，**不补端口**） | `libs/hbb_common/src/config.rs:1072` `get_rendezvous_servers()` |
| 烧入点：PROD | `libs/hbb_common/src/config.rs:69` |
| 烧入点：公钥 | `libs/hbb_common/src/config.rs:117` |
| 末级兜底常量 | `libs/hbb_common/src/config.rs:116` `RENDEZVOUS_SERVERS` |
| 端口常量 | `libs/hbb_common/src/config.rs:119-123`（21116/21117/21118/21119） |
| API 解析 | `src/common.rs:1270` `get_api_server()` / `:1287` `get_api_server_()` |
| API 烧入点 | `src/common.rs:1297`（本仓库新增） |
| API 推导用的 custom server | `src/common.rs:1253` `get_custom_rendezvous_server()` |
| 端口 ±N | `libs/hbb_common/src/socket_client.rs` `increase_port()` / `check_port()` |
| Key 解析 | `src/common.rs:2163` `get_key()` |
| 握手验签 | `src/client.rs:1655` `secure_connection()` / `src/common.rs:2433` `decode_id_pk_dtls()` |
| WebSocket 端点转换 | `libs/hbb_common/src/websocket.rs:470` `check_ws()` |
| ws/wss 判定（烧入点） | `libs/hbb_common/src/websocket.rs:524` |
| 自有服务器白名单 | `libs/hbb_common/src/websocket.rs:544-548` |
| WebSocket 总闸 | `libs/hbb_common/src/config.rs:2948` `use_ws()` / `:2928` `option2bool()` |
| 连接目标解析 | `libs/hbb_common/src/socket_client.rs:140` |
| 中继解析 | `src/rendezvous_mediator.rs:1367` `get_relay_server()` |
| mediator 启动 | `src/rendezvous_mediator.rs:317` `start_all()` |
| 配置改动触发重启 | `src/ipc.rs:795` `CheckIfRestart`（`:834` 监听 api-server） |
| API 下发黑名单 | `src/hbbs_http/sync.rs:295` `BLOCKED_CONFIG_OPTIONS` |
| custom.txt 验签 | `src/common.rs:2607` `read_custom_client()` |
| exe 文件名解析 | `src/custom_server.rs:44` / `src/naming.rs` |
| **破坏 macOS 签名的那行** | `build.py:949` |
| macOS 重签脚本 | `.github/scripts/sign-macos-app.sh` |
| 签名步骤的跳过条件 | `.github/workflows/flutter-build.yml` `env.MACOS_P12_BASE64 != null` |
| SIGN_BASE_URL 的 `-2` 技巧 | `.github/workflows/flutter-build.yml:63` |
| 4 处容器构建 | `.github/workflows/flutter-build.yml:2037 / 2359 / 2651 / 2923` |

---

## 12. 本次实际改动清单（相对上游 8 项）

```
.github/workflows/flutter-build.yml   +4 行 env（3 个 secret 导出）
.github/workflows/flutter-build.yml   +7 行 × 4 处（run-on-arch-action 的 env 输入）
.github/workflows/flutter-build.yml   +11 行（create unsigned dmg 前的 ad-hoc 重签）
.github/workflows/flutter-nightly.yml +4 行 permissions，-3 行 schedule
.github/workflows/flutter-tag.yml     +4 行 permissions
libs/hbb_common/src/config.rs         2 处 option_env! 补丁
libs/hbb_common/src/websocket.rs      1 处 option_env! 补丁
src/common.rs                         1 处 option_env! 补丁
删除：.gitmodules、libs/hbb_common/.git、.github/workflows/flutter-ci.yml
结构：libs/hbb_common 由 submodule 改为内联目录（33 个文件）
```

核对方法（应精确等于上面这些，无多余改动）：
```bash
diff -rq --exclude=.git <上游克隆> <本仓库工作树>
```

---

## 附录 A. 本次验证记录（作为下次的对照基线）

最终产物来自 run `42749653883`（commit `ef26f3d`），Release tag `nightly`。

### A.1 构建结果

```
29 job = 26 success + 2 skipped + 1 failure
耗时 145min（含 flatpak 重试等待）
唯一失败：Build flatpak x86_64-unknown-linux-gnu —— Ubuntu 镜像源 perl 404，见坑 #10，与本仓库无关
```

### A.2 各平台烧入值验证

判据：你的域名/公钥/API 命中，官方公钥 `OeVuKk5…` 与 `admin.rustdesk.com` 为 **0**。

| 平台 | 被检文件 | 域名 | 公钥 | API | 官方公钥 | admin |
|---|---|---|---|---|---|---|
| Linux | `.deb` → `usr/share/rustdesk/lib/librustdesk.so` | 1 | 1 | 1 | **0** | **0** |
| Android | `.apk` → `lib/arm64-v8a/librustdesk.so` | 2 | 1 | 1 | **0** | **0** |
| Windows | artifact → `librustdesk.dll` | 2 | 1 | 1 | **0** | **0** |
| macOS | `.dmg` → `Contents/Frameworks/liblibrustdesk.dylib` | 分片 | 1 | 分片 | **0** | **0** |

macOS 的分片实测输出（证明两处 `API_SERVER` 引用都在）：
```
:21114allow-https-21114https://<域名>          ← src/common.rs get_api_server_ 补丁
api-serverhttps://<域名>websocket-port          ← websocket.rs check_ws 补丁
rustdesk.<域名>:21116                           ← PROD_RENDEZVOUS_SERVER
```

`rs-ny.rustdesk.com` 在各平台均有 1–2 次命中，**属预期**（末级兜底常量，故意保留）。

### A.3 macOS 签名验证

```
codesign --verify --deep --strict --verbose=2 RustDesk.app
  → valid on disk
  → satisfies its Designated Requirement

bundle:  Signature=adhoc  flags=0x10002(adhoc,runtime)  TeamIdentifier=not set
service: Signature=adhoc  flags=0x10002(adhoc,runtime)   ← 已封印
entitlements: app-sandbox=false, allow-jit=true, audio-input=true, network.client=true  ← 4 项全保留
spctl -a -vvv -t install → rejected   ← 预期（无公证），清 quarantine 或走系统设置「仍要打开」即可
```

### A.4 权威摘要（用于校验下载完整性）

```
rustdesk-1.5.3-aarch64.dmg   31348944 字节   sha256:a57cd0391b421ac131e6a544419a27c3fd3f955f677db8b43d36f43bf456de85
```

取值命令（每次构建后都会变）：
```bash
gh api "/repos/<owner>/<repo>/releases/tags/nightly" \
  --jq '.assets[]|{name:.name,size:.size,digest:.digest}'
```


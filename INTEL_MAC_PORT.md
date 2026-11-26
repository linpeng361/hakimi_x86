# Intel Mac（x86_64）适配记录

## 结论

源码中所有平台分支都按 `target_os` 区分，没有 `target_arch = "aarch64"` 之类的架构写死；
macOS 的系统代理（`networksetup`）、证书信任（`security`）、开机自启（LaunchAgent）、
窗口与 Dock 交互（objc2/AppKit）均与 CPU 架构无关。适配工作集中在**构建、打包、发布链路**。

## 本机验证环境

- 机器：Intel Mac（x86_64），macOS 26.7.3
- 工具链：Homebrew Rust stable 1.99.1（无 nightly）；cmake / nasm / perl / openssl 齐备
- 验证结果：

| 验证项 | 结果 |
| --- | --- |
| `cargo check -p hakimi-core` | 通过（1 个既有 visibility warning，与架构无关） |
| `cargo check -p hakimi-cli` | 通过 |
| `cargo check -p hakimi-ui`（windui + objc2-app-kit + wgpu-apple） | 通过 |
| `cargo test -p hakimi-core --lib` | 416 通过，21 忽略，1 失败 |
| `cargo build --profile desktop-release -p hakimi-ui` | 通过，产物 7.4M，Mach-O x86_64，符号已剥离 |
| `package-macos.sh` 打包 | 生成 `hakimi-0.1.0-macos-x64.dmg`，ad-hoc 签名通过，DMG 校验 VALID |
| `hakimi --help`（CLI） | 命令树完整，可执行 |

### 唯一失败的测试说明

`ca::tests::minider_crosscheck::shipped_android_probe_ca_matches_x509_parser`
校验的是发布固定 CA 的 SKI 指纹。该 CA 由 CI 的 "Prepare Android CA" 步骤注入
（`android/probe/assets/ca/ca.pem`，目录被 `android/probe/.gitignore` 忽略），
本地用 openssl 自签的临时 fixture 指纹对不上，**不是 Intel 适配问题**。
在有固定 CA 的 CI 环境中该测试正常。

## 改动清单

### `scripts/ci/package-macos.sh`

- 不再硬性要求 arm64；按 `lipo -archs` 识别切片：
  - `arm64` → `hakimi-<ver>-macos-arm64.dmg`
  - `x86_64` → `hakimi-<ver>-macos-x64.dmg`
  - `arm64 x86_64`（universal2）→ `hakimi-<ver>-macos-universal.dmg`
  - 其他组合报错退出
- 注释更新：ad-hoc 签名在 Intel 上非强制，但保留签名使两架构 Gatekeeper 行为一致。

### `.github/`（已从 fork 移除）

- 本 fork 不带任何 CI workflow（`ci.yml`、`release.yml` 均删除）。Intel fork 不跑
  Actions，且当前推送 PAT 没有 `workflow` scope，无法更新 workflow 文件。
- 含 x64 构建 job 的 workflow 备份在本地 `/tmp/hakimi-ci-backup/`；日后若要在
  Actions 出 Intel 包，需给 PAT 补 `workflow` 权限后恢复该目录。

### `scripts/ci/update_manifest.py`

- 新增 `--desktop-macos-x64-asset` 参数；清单 `desktop.assets` 增加 `macos-x64` 字段。

### `scripts/build-desktop.sh`

- 用法注释补充 `HAKIMI_TARGET=x86_64-apple-darwin` 示例（脚本本身已支持该变量）。

### `README.md`

- 桌面段补充架构选包说明：Apple Silicon 选 `macos-arm64`，Intel 选 `macos-x64`。

## 本地构建

CI 脚本 `build-desktop.sh` 使用 nightly + `-Z build-std`（size 优化 profile）。
本机只有 stable 时，可直接绕过 build-std 出调试/发布二进制：

```sh
# 检查（stable）
cargo check -p hakimi-ui --locked

# release profile（stable；不带 build-std，其余 LTO/strip/panic=abort 均生效）
cargo build --profile desktop-release -p hakimi-ui --locked

# 打包 DMG（脚本自动识别 x86_64 并命名 macos-x64）
bash scripts/ci/package-macos.sh <VERSION> <VERSION_CODE> \
  target/desktop-release/hakimi-ui dist
```

完整复刻 CI 产物则需 nightly 工具链：

```sh
rustup install nightly
./scripts/build-desktop.sh
```

## 待决项（未擅自改动）

1. **更新通道是否 fork 化**：以下位置仍指向原作者仓库 `NLick47/hakimi`，
   若本 fork 要独立发版并走自己的更新清单，需要一并改成 `linpeng361/hakimi_x86`
   （含 release URL 白名单校验与测试用例）：
   - `crates/hakimi-ui/src/update.rs`（`PROJECT` / `RELEASES_URL` / `safe_release_url`）
   - `android/probe/src/com/hakimi/probe/UpdateChecker.kt`
2. **CI 未随 fork 发布**：`.github/` 已从仓库移除，出包走本机脚本；需要 Actions
   流水线时恢复备份的 workflow 并给 PAT 补 `workflow` 权限。
3. **GUI 未实跑**：沙箱内无法弹窗交互；建议首次在本机打开 x64 DMG，
   右键「打开」绕过 Gatekeeper，走一遍证书安装与系统代理引导。
4. 代码依赖（`rustls` fork、`windui` fork、Piku Worker）仍引用上游仓库，未 fork 到本仓库。

## 本地环境备注（不进仓库）

- `.cargo/config.toml`：`net.git-fetch-with-cli = true`。本机有本地 MITM 加速工具，
  cargo 内置 libgit2 不读系统钥匙串会证书报错；改用系统 git CLI 后走系统信任链。
  该目录被 `.gitignore` 忽略，仅本机生效。
- `android/probe/assets/ca/`：本地测试用自签 CA fixture，被 gitignore，不进仓库。

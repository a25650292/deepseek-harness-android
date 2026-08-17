# DeepSeek Harness (dsh) —— Android / Termux 安装过程记录

> 本文档记录在本机（Android + Termux）安装 DeepSeek 官方 agent harness
> `@deepseek-ai/dsh` 的完整过程，包括所有 Android 兼容修复。
> 记录内容与 `setup.sh` 实际执行的步骤一一对应。

## 0. 环境

- 设备：Android + Termux（无 root）
- Shell：`/data/data/com.termux/files/usr/bin/bash`
- 已安装基础包：`curl`、`git`、`openssh`、`python`、`python-pip`、
  `nodejs`、`npm`、`cmake`、`clang`、`make`、`binutils`、`pkg-config` 等
- 安装目标：`@deepseek-ai/dsh`（npm 包，全局安装）
- 安装目录约定：`$HOME/dsh`（启动/停止脚本与日志）
- 运行时目录：`$PREFIX/lib/node_modules/@deepseek-ai/dsh`

## 1. 安装构建依赖

```bash
pkg update -y
pkg install -y cmake clang make binutils pkg-config python nodejs
```

确认 `node` 可用，并记录版本号（后续 node-gyp 缓存路径依赖它）。

## 2. 智能换源（国内网络友好）

脚本会检测默认 npm 源与 nodejs.org 是否过慢（>1s），慢则自动切换：

```bash
export npm_config_registry="https://registry.npmmirror.com"
export npm_config_disturl="https://npmmirror.com/mirrors/node/"
```

仅通过环境变量作用于本次安装会话，**不改动全局 npm 配置**。

## 3. 准备 node-gyp 补丁（修复 node-pty 构建）

- 执行 `npx --yes node-gyp install` 只下载 Node headers 填充缓存
  （比整树 `npm install` 快得多）。
- 修补缓存中的 `common.gypi`：Termux 没有 NDK，该文件引用了未定义的
  `android_ndk_path` 变量，会导致 node-pty 构建报
  "Undefined variable" 错误。修补方法是在 `variables` 中显式定义
  `'android_ndk_path': ''`。

```bash
timeout 300 npx --yes node-gyp install 2>&1 | tail -3
# 修补 $HOME/.cache/node-gyp/$NODE_VER/include/node/common.gypi
```

## 4. 正式安装 dsh（android30 编译目标，最耗时步骤）

核心命令（处理 koffi 编译缺 `<spawn.h>` 的问题，Issue #4）：

```bash
CFLAGS="-target aarch64-linux-android30 -I$PREFIX/include" \
CXXFLAGS="-target aarch64-linux-android30 -I$PREFIX/include" \
npm install -g --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs @deepseek-ai/dsh
```

- 若 `$PREFIX/include/spawn.h` 缺失，写入一个 bionic 兼容的 spawn.h shim
  （`posix_spawn` 系列声明），并加入 include 路径。
- 这一步会做原生编译（node-pty、koffi），耗时 5~15 分钟，不能中断。
- 安装后校验两个关键编译产物：
  - `node_modules/node-pty/build/Release/pty.node`
  - `node_modules/koffi/build/koffi/android_arm64/koffi.node`

## 5. 后端兼容补丁（Android 禁 hardlink 一族）

Android/部分 ROM 禁止 `link()`（hardlink），需要把持久化相关写入
从 `link` 改为 `rename`：

1. **会话持久化** `dsh-session-persistence-jsonl`：`link(tmp, finalPath)`
   → `rename(tmp, finalPath)`
2. **附件存储** `dsh-attachment-local`：`link(temporary, target)`
   → `rename(temporary, target)`
3. **write 工具新建文件 / 附件祖先遍历清理容忍**：
   执行 `patches/patch-dsh-android-link.js`（幂等，基于
   dsh 0.1.0-rc.3 / rc.6）
4. **subprocess 终端检测**：`platform === "linux"` 改为
   `platform === "linux" || platform === "android"`，让 Android 复用
   LinuxProcessInspector（补丁在 `apply-js-patches.sh` 中一并处理）
5. **作曲栏回车行为**：普通回车 = 换行（不发送），
   Ctrl/Cmd+Enter = 发送，避免安卓输入法回车误触发发送。

## 6. sharp WebAssembly 回退

android-arm64 没有 sharp 的原生预编译产物，需要手动安装 wasm32 版本：

```bash
npm install "@img/sharp-wasm32@$SHARP_VER"
# 复制 @img/sharp-wasm32 与 @emnapi 到 dsh 的 node_modules/@img
```

## 7. 重建 /usr/bin/dsh 包装脚本

HMR 需要 `--expose-internals`，所以重建可执行包装脚本：

```bash
rm -f /data/data/com.termux/files/usr/bin/dsh
# 内容：
#!/data/data/com.termux/files/usr/bin/sh
exec node --expose-internals /data/data/com.termux/files/usr/lib/node_modules/@deepseek-ai/dsh/lib/bin.js "$@"

chmod +x /data/data/com.termux/files/usr/bin/dsh
dsh --version
```

## 8. 启动/停止脚本 + 权限模式

- 把 `start_dsh.sh` / `stop_dsh.sh` 复制到 `$HOME/dsh/` 并加执行权限。
- 写入权限模式配置 `~/.dsh/profiles/web/cordis.patch.yml`：

```yaml
# Android/Termux：bwrap/landlock 命名空间沙箱不可用，需放开权限模式才能执行 bash 工具
- id: sandbox-policy
  config:
    mode: danger-full-access
```

- `start_dsh.sh` 同时通过 `export DSH_PERMISSION_MODE=danger-full-access`
  双保险，然后以 `nohup dsh web` 后台启动，轮询
  `http://127.0.0.1:3080` 就绪后 `termux-open-url` 拉起 Chrome。

## 9. 前端移动端适配（可选步骤）

`apply-frontend.sh` 会往 dsh 的 `index.html` 注入
`patches/mobile.css` 与 `patches/mobile.js`（移动端布局、软键盘、
`enterkeyhint=newline` 等适配）。

## 10. JS 性能补丁（可选步骤）

`apply-js-patches.sh` 基于 dsh 0.1.0-rc.6 应用 5 个补丁（幂等）：

| 补丁 | 作用 |
| --- | --- |
| `01-apiproxy-history-slim.patch` | history 响应瘦身：assistant/chunk 流过滤 + 超大 tool 结果/参数截断（冷重进不卡） |
| `02-runtime-incremental-resync.patch` | 重连增量同步：保留窗口、静默补齐，不再全量重建（切回 App 不卡） |
| `03-connection-history-schema.patch` | history 响应新增 chunkFiltered 标志（配合 01/02） |
| `04-frontend-static-cache.patch` | 静态资源 immutable 缓存头（整页重载不重复下载） |
| `05-client-modules-cache.patch` | 插件 bundle immutable 缓存头 |

## 使用方式

```bash
bash ~/dsh/start_dsh.sh    # 启动服务并拉起 Chrome
# 打开 http://127.0.0.1:3080 ，在 Models 页填入 DeepSeek API Key
bash ~/dsh/stop_dsh.sh     # 停止服务
```

注意：

- 服务只监听 `127.0.0.1`（本机），不走局域网。
- API Key 存于 `~/.dsh/.credentials.yaml`（0600 权限），不进日志。
- `danger-full-access` 关闭了进程沙箱（Android 无替代），仅建议个人设备使用。
- 升级 dsh 或 Node 之后需重跑 `setup.sh`。
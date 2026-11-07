# dsh-fish-preset 🐟

把「ptc 编码预设 + `apply_patch` 工具 + Magpie MCP 客户端」打包成一个 **DeepSeek Harness (DSH) 插件（bundle）**，方便分发和维护。安装后会在 Agent 预设里多出一个 **Fish** 预设：它在内置 `ptc` 预设基础上打开了 `apply_patch`（一次调用完成多文件补丁编辑），并内置一条 `@deepseek-ai/dsh-mcp-client` 去连本机 Magpie 网关的 MCP hub，让 Magpie Library 里的 MCP server（如 `magpie-browser`）以 `mcp__magpie__<tool>` 的形式进入模型工具集。

- 纯配置插件：没有任何运行时依赖，也不含宿主/浏览器端代码。
- 真正的“能力”来自它 `cordis.patch.yml` 里插入的那条 `@deepseek-ai/dsh-agent-preset` 声明。

## 安装（GitHub 链接直装）

### 方式一：插件页安装（推荐）

打开 DSH Web GUI 侧边栏的 **Plugins（插件）** 页面 → 安装 → 粘贴下面任一地址：

```
github:CnsMaple/dsh-fish-preset
```

或仓库地址 `https://github.com/CnsMaple/dsh-fish-preset`。安装完成后插件默认启用；到 **设置 → Agent 预设** 里选择 **Fish** 即可（新建会话才会生效）。

### 方式二：命令行

```sh
dsh plugin --profile web add github:CnsMaple/dsh-fish-preset
```

这条命令等价于在 profile 目录里 `pnpm add`，只把包装进依赖；还需要把它加进 profile 的 bundle 列表（**Plugins 页面启用一次**即可，或手动把 `dsh-fish-preset` 加入 `$DSH_HOME/profiles/web/package.json` 的 `dsh.profile.bundles`）。

### 卸载

Plugins 页面停用/移除，或：

```sh
dsh plugin --profile web remove dsh-fish-preset
```

## 使用

安装并启用后，**新建会话**时在预设选择里选 **Fish**；也可以把它设为新会话默认（设置 → Agent 预设）。预设改动只影响之后新建的 Agent，已开的会话保持原组合。

Fish 预设包含（与 `ptc` 相同）：persona、agent-instructions、time-context、bash/pwsh、文件工具（含 `apply_patch`）、技能、goal、plan mode、compaction、delegation（subagent 等）、ask-user、todo、web，以及 PTC 的 `run_code` 执行器。此外还挂载了 **Magpie MCP 客户端**（见下）。

### Magpie MCP

Fish 预设额外声明了一条 `@deepseek-ai/dsh-mcp-client`：

```yaml
- id: magpie-mcp-magpie
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: "magpie"
    transport: "streamable-http"
    url: "http://127.0.0.1:3425/mcp/magpie/hub/dsh@wsl:Arch"
    failOnStartupError: false
```

- 它连的是**本机 Magpie 网关**（默认 `127.0.0.1:3425`）的 MCP hub（`/mcp/magpie/hub/<agent>`，需在 Magpie 的 **Library → MCP** 里打开 **MCP router**）。hub 上 `magpie-browser` 等 server 的工具会注册成 `mcp__magpie__magpie-browser__browser`，在 `run_code` 里用 `tools["mcp__magpie__magpie-browser__browser"](...)` 调用。
- **端到端地址要按运行环境改**：上面的 `dsh@wsl:Arch` 是该会话在 WSL Arch 里的 agent id（WSL 与 Windows 网关在 mirrored/consomme 网络下共用 `127.0.0.1`）。若在别处/Windows 原生运行，把 path 改成对应 hub 路径（如 `/mcp/magpie/hub/dsh`）；若改了 `MAGPIE_ADDR` 端口，也要一并改端口。
- `failOnStartupError: false`：**Magpie 没运行/网关不在时不会报错**，只是没有这些 MCP 工具；Magpie 起来后重开会话即可用。
- 没有 Magpie 也不想要这条的：删掉这一项即可，其余与 `ptc` 完全一致。

## 工作原理（怎么 patch）

DSH 的 profile 由**有序的 bundle patch 层**组合而成。一个插件只要在 `package.json` 里声明：

```json
"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
```

并在 `cordis.patch.yml` 里提供 patch 项，就会被当作一个 bundle 层参与组合。本仓库的 `cordis.patch.yml` 只有一项：

```yaml
- insert:
    - id: preset-fish
      name: '@deepseek-ai/dsh-agent-preset'
      config:
        id: fish
        name: Fish
        description: "基于 ptc 的编码预设，额外启用 apply_patch（一次调用完成多文件补丁编辑）。"
        order: 5
        plugins:
          # ...（与内置 ptc 相同的子插件列表）
          - id: tool-fs
            name: '@deepseek-ai/dsh-tool-fs'
            config:
              applyPatch: true
```

要点：
- **新增**用 `insert:`；只想覆盖已有条目才用顶层 `- id: <entry>`（那会整段替换该条目 `config`）。
- 行 `name` 和各子插件 `name` 用 DSH 运行时解析（`@deepseek-ai/dsh-*` 指向安装的 DSH 运行时），不需要本包安装这些依赖。
- `config.id` 是预设身份（会话记录它）；`name` 是显示名；`description` 只是卡片文案。

## 开发

本项目是“已构建、可直接直装”的形态：`lib/` 与 `cordis.patch.yml` 都随仓库提交，GitHub 直装无需在用户机器上构建。改动预设内容时编辑 `cordis.patch.yml` 即可；`lib/index.js` 只是空壳占位（本 bundle 不加载它）。

本地自测：修改后可用
```sh
dsh --profile web --dump-config --patch ./cordis.patch.yml
```
查看组合结果（注意：若你的 profile 里已有同名 `preset-fish` 会重复，需先移除）。

## 许可

MIT。

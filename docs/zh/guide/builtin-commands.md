# 内置命令详解

> 这篇文档专门说明 OpenCLI 的 built-in commands，也就是 `opencli` 自带的顶层命令与命名空间命令。
>
> 本文以仓库中的 [`src/cli.ts`](/D:/code/ai_code/OpenCLI/src/cli.ts) 为主，并参考 `opencli --help` 的实际输出编写。若你本地安装的是较早版本，少数命令或参数可能尚未发布，请以 `opencli <command> --help` 的结果为准。

## 先区分三类命令

OpenCLI 里常见的命令来源有三类：

1. **内置命令（built-in commands）**
   例如 `list`、`browser`、`doctor`、`plugin`、`adapter`、`daemon`。
2. **内置适配器命令**
   例如 `opencli bilibili hot`、`opencli amazon bestsellers`、`opencli codex status`。
3. **外部 CLI 透传命令**
   例如 `opencli gh ...`、`opencli docker ...`，前提是它们已经被 OpenCLI 注册。

这篇文档只讲第 1 类，也就是 OpenCLI 自己提供的“管理、探索、浏览器控制、插件、适配器、外部 CLI 注册”等命令。

## 命令总览

当前常见的 built-in commands 可以按职责分成下面几组：

| 分组 | 命令 |
| --- | --- |
| 基础发现与校验 | `list`、`validate`、`verify` |
| 站点探索与生成 | `explore` / `probe`、`synthesize`、`generate`、`record`、`cascade` |
| 浏览器控制 | `browser ...` |
| 环境诊断 | `doctor`、`completion` |
| 插件管理 | `plugin install`、`plugin uninstall`、`plugin update`、`plugin list`、`plugin create` |
| 适配器管理 | `adapter status`、`adapter eject`、`adapter reset` |
| daemon 管理 | `daemon status`、`daemon stop` |
| 外部 CLI 接入 | `install`、`register` |
| 特殊内置命令 | `antigravity serve` |

## 1. `opencli list`

### 用途

列出当前可用的所有命令，包括：

- OpenCLI built-in commands
- 内置适配器命令
- 已安装插件命令
- 已注册的外部 CLI

### 常用写法

```bash
opencli list
opencli list -f json
opencli list -f yaml
opencli list -f md
opencli list -f csv
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `-f, --format <fmt>` | 输出格式，可选 `table`、`json`、`yaml`、`md`、`csv` |
| `--json` | 兼容旧写法，等价于 `-f json`，已不推荐继续使用 |

### 适合场景

- 快速查看当前版本支持哪些站点和命令
- 把命令清单导出成 JSON / Markdown
- 给 AI Agent 提供“可调用能力列表”

## 2. `opencli validate`

### 用途

校验 CLI 定义是否合法，通常用于检查适配器是否存在结构问题。

### 常用写法

```bash
opencli validate
opencli validate bilibili
opencli validate bilibili/hot
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `[target]` | 可选。支持 `site` 或 `site/name`，例如 `twitter`、`amazon/product` |

### 适合场景

- 批量检查当前所有适配器
- 只检查某个站点
- 修改了本地 adapter 后做结构校验

## 3. `opencli verify`

### 用途

在 `validate` 的基础上进一步做 smoke test，检查命令是否能执行。

### 常用写法

```bash
opencli verify
opencli verify amazon
opencli verify amazon/product
opencli verify amazon/product --smoke
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `[target]` | 可选，支持 `site` 或 `site/name` |
| `--smoke` | 运行 smoke tests |

### 适合场景

- 发布前做整体回归
- 新增适配器后做最基本的可执行校验
- 本地 override 适配器修改后做快速验收

## 4. 探索与生成命令

这组命令更偏“把网站能力沉淀成命令”的工作流，尤其适合开发者和 AI Agent。

### 4.1 `opencli explore` / `opencli probe`

### 用途

探索一个网站页面，发现：

- 网络请求
- 可疑 JSON API
- 前端 store / action
- 适合的认证策略
- 后续适配器生成线索

### 常用写法

```bash
opencli explore "https://example.com/page"
opencli explore "https://example.com/page" --site mysite
opencli explore "https://example.com/page" --goal "读取列表数据"
opencli explore "https://example.com/page" --auto
opencli probe "https://example.com/page"
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<url>` | 必填，要探索的页面地址 |
| `--site <name>` | 指定站点名 |
| `--goal <text>` | 指定探索目标，例如“提取文章列表” |
| `--wait <s>` | 页面打开后等待秒数，默认 `3` |
| `--auto` | 启用自动交互 / fuzzing |
| `--click <labels>` | 在探索前先点击指定标签，多个值用逗号分隔 |
| `-v, --verbose` | 输出调试信息 |

### 适合场景

- 不清楚网站数据来自哪个接口
- 想从真实页面行为反推出 API
- 准备为新站点生成 adapter

### 4.2 `opencli synthesize`

### 用途

把 `explore` 产生的结果转成候选 CLI 定义。

### 常用写法

```bash
opencli synthesize .opencli/explore/example.com
opencli synthesize .opencli/explore/example.com --top 5
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<target>` | 必填，explore 结果目录 |
| `--top <n>` | 生成前 N 个候选命令，默认 `3` |
| `-v, --verbose` | 输出调试信息 |

### 4.3 `opencli generate`

### 用途

一条命令串起完整流程：

`explore -> synthesize -> verify -> register`

### 常用写法

```bash
opencli generate "https://example.com/page"
opencli generate "https://example.com/page" --goal "搜索商品"
opencli generate "https://example.com/page" --site mysite
opencli generate "https://example.com/page" --no-register
opencli generate "https://example.com/page" --format json
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<url>` | 必填，要生成命令的网站地址 |
| `--goal <text>` | 生成目标 |
| `--site <name>` | 指定站点名 |
| `--format <fmt>` | 输出格式，常见为 `table` 或 `json` |
| `--no-register` | 只验证，不自动注册 |
| `-v, --verbose` | 输出调试信息 |

### 4.4 `opencli record`

### 用途

对一个正在交互的页面录制 API 调用，然后生成候选定义。

### 常用写法

```bash
opencli record "https://example.com/page"
opencli record "https://example.com/page" --site mysite --out ./tmp/candidates
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<url>` | 必填，要打开并录制的页面 |
| `--site <name>` | 指定站点名 |
| `--out <dir>` | 候选输出目录 |
| `--poll <ms>` | 轮询间隔，默认 `2000` |
| `--timeout <ms>` | 自动停止时间，默认 `60000` |
| `-v, --verbose` | 输出调试信息 |

### 4.5 `opencli cascade`

### 用途

自动判断同一能力更适合走哪种策略：

- `public`
- `cookie`
- `header`
- `intercept`
- `ui`

### 常用写法

```bash
opencli cascade "https://example.com/page"
opencli cascade "https://example.com/page" --site mysite
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<url>` | 必填 |
| `--site <name>` | 指定站点名 |
| `-v, --verbose` | 输出调试信息 |

## 5. `opencli browser` 命名空间

这是 OpenCLI 最重要的 built-in namespace 之一。它把浏览器暴露成一组底层原语，适合：

- AI Agent 直接操控网页
- 手工调试页面状态
- 验证登录态
- 抓接口
- 定位元素
- 写 adapter 前做试探

### 使用前提

通常需要先完成以下准备：

1. 安装 `@jackwener/opencli`
2. 安装 Browser Bridge 扩展
3. 打开 Chrome / Chromium
4. 必要时先登录目标网站

建议先跑：

```bash
opencli doctor
```

### 5.1 页面与标签页控制

> 说明：`browser tab ...`、`browser frames`、`--tab <targetId>` 这类能力在仓库源码中已经存在；如果你的本地版本还没显示，请先升级到与当前源码一致的版本。

#### `opencli browser open <url>`

打开页面并建立自动化窗口。

```bash
opencli browser open "https://example.com"
```

#### `opencli browser back`

后退一页。

```bash
opencli browser back
```

#### `opencli browser scroll <direction>`

向上或向下滚动页面。

```bash
opencli browser scroll down
opencli browser scroll down --amount 1200
opencli browser scroll up --amount 300
```

参数：

| 参数 | 说明 |
| --- | --- |
| `<direction>` | `up` 或 `down` |
| `--amount <pixels>` | 滚动像素，默认 `500` |

#### `opencli browser tab list`

列出 automation window 中的标签页与 target ID。

#### `opencli browser tab new [url]`

新建标签页，可选地直接打开一个 URL。

#### `opencli browser tab select <targetId>`

选择某个标签页作为后续默认目标。

#### `opencli browser tab close <targetId>`

关闭指定标签页。

### 5.2 状态与内容读取

#### `opencli browser state`

输出当前页面的 URL、标题、可交互元素和索引。

```bash
opencli browser state
```

这是后续 `click`、`type`、`get text`、`get attributes` 的基础。

#### `opencli browser frames`

列出跨域 iframe 目标，便于后续 frame 定向执行。

#### `opencli browser screenshot [path]`

截图。不给路径时通常输出 base64，给路径时写入文件。

```bash
opencli browser screenshot
opencli browser screenshot ./page.png
```

### 5.3 `browser get` 子命令

#### `opencli browser get title`

读取页面标题。

#### `opencli browser get url`

读取当前 URL。

#### `opencli browser get text <index>`

读取某个元素的文本内容。

```bash
opencli browser get text 12
```

#### `opencli browser get value <index>`

读取输入框 / textarea 的 value。

#### `opencli browser get attributes <index>`

读取某个元素的属性集合。

#### `opencli browser get html`

读取整个页面 HTML，或只读取某个选择器匹配的范围。

```bash
opencli browser get html
opencli browser get html --selector ".table-wrap"
```

源码中的更高阶用法还支持：

```bash
opencli browser get html --as json
opencli browser get html --selector "#app" --as json
opencli browser get html --max 5000
```

其中：

| 参数 | 说明 |
| --- | --- |
| `--selector <css>` | 指定 CSS 选择器范围 |
| `--as <format>` | `html` 或 `json`，`json` 会输出结构化树 |
| `--max <n>` | 限制原始 HTML 最大字符数，`0` 表示不限制 |

### 5.4 交互类命令

#### `opencli browser click <index>`

按 `state` 输出的索引点击元素。

```bash
opencli browser click 8
```

#### `opencli browser type <index> <text>`

点击元素并输入文本。

```bash
opencli browser type 5 "搜索关键词"
```

#### `opencli browser select <index> <option>`

操作 `<select>` 下拉框。

```bash
opencli browser select 9 "最近7天"
```

#### `opencli browser keys <key>`

发送键盘按键。

```bash
opencli browser keys Enter
opencli browser keys Escape
opencli browser keys Control+a
```

### 5.5 等待与执行

#### `opencli browser wait <type> [value]`

等待时间、文本或 CSS 选择器出现。

```bash
opencli browser wait time 3
opencli browser wait text "保存成功"
opencli browser wait selector ".loaded" --timeout 15000
```

参数：

| 参数 | 说明 |
| --- | --- |
| `<type>` | `time`、`text`、`selector` |
| `[value]` | 秒数、文本、选择器 |
| `--timeout <ms>` | 超时时间，默认 `10000` |

#### `opencli browser eval <js>`

在页面上下文执行 JavaScript。

```bash
opencli browser eval "document.title"
opencli browser eval "JSON.stringify(window.__INITIAL_STATE__)"
```

源码中的更高阶用法还支持：

```bash
opencli browser eval "document.body.innerText" --frame 0
```

### 5.6 网络抓取

#### `opencli browser network`

查看页面加载或操作过程中捕获到的网络请求。

```bash
opencli browser network
opencli browser network --all
opencli browser network --detail 3
```

已发布版本常见参数：

| 参数 | 说明 |
| --- | --- |
| `--detail <index>` | 查看某条请求的完整 body |
| `--all` | 包括静态资源请求 |

当前源码里还支持更细的高级参数，例如：

| 参数 | 说明 |
| --- | --- |
| `--raw` | 输出所有请求完整 body |
| `--filter <fields>` | 按返回体 shape 过滤请求 |
| `--ttl <ms>` | `--detail` 缓存 TTL |

### 5.7 Adapter 脚手架

#### `opencli browser init <name>`

在 `~/.opencli/clis/` 下生成 adapter 模板。

```bash
opencli browser init mysite/list
```

#### `opencli browser verify <name>`

执行某个本地 adapter，快速验证其是否可用。

```bash
opencli browser verify mysite/list
```

#### `opencli browser close`

关闭 automation window。

## 6. `opencli doctor`

### 用途

诊断 Browser Bridge、daemon、扩展连接状态。

### 常用写法

```bash
opencli doctor
opencli doctor --sessions
opencli doctor -v
opencli doctor --no-live
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `--no-live` | 跳过实际连接测试 |
| `--sessions` | 显示当前活跃 session |
| `-v, --verbose` | 输出调试信息 |

### 推荐使用时机

- 浏览器命令报错前先排查环境
- 扩展疑似断连
- 想确认 daemon 是否可用

## 7. `opencli completion`

### 用途

输出 shell 自动补全脚本。

### 常用写法

```bash
opencli completion zsh
opencli completion bash
opencli completion fish
```

## 8. `opencli plugin` 命名空间

用于管理社区插件或团队内部插件。

### 8.1 `opencli plugin install <source>`

从 Git 仓库或本地路径安装插件。

```bash
opencli plugin install github:user/repo
opencli plugin install github:user/repo/subplugin
opencli plugin install https://github.com/user/repo
opencli plugin install /path/to/local/plugin
```

### 8.2 `opencli plugin uninstall <name>`

卸载一个插件。

```bash
opencli plugin uninstall github-trending
```

### 8.3 `opencli plugin update [name]`

升级插件。

```bash
opencli plugin update github-trending
opencli plugin update --all
```

参数：

| 参数 | 说明 |
| --- | --- |
| `[name]` | 插件名；不传时需搭配 `--all` |
| `--all` | 升级所有插件 |

### 8.4 `opencli plugin list`

查看当前已安装插件。

```bash
opencli plugin list
opencli plugin list -f json
```

### 8.5 `opencli plugin create <name>`

创建插件脚手架。

```bash
opencli plugin create my-plugin
opencli plugin create my-plugin --description "我的插件"
opencli plugin create my-plugin --dir ./packages/my-plugin
```

参数：

| 参数 | 说明 |
| --- | --- |
| `<name>` | 插件名，建议小写加短横线 |
| `-d, --dir <path>` | 输出目录 |
| `--description <text>` | 插件描述 |

## 9. `opencli adapter` 命名空间

用于管理本地 adapter override。

### 9.1 `opencli adapter status`

查看哪些站点被本地 override，哪些仍使用官方内置版本。

```bash
opencli adapter status
```

### 9.2 `opencli adapter eject <site>`

把官方 adapter 复制到 `~/.opencli/clis/<site>/`，便于本地修改。

```bash
opencli adapter eject twitter
```

### 9.3 `opencli adapter reset [site]`

删除本地 override，恢复官方版本。

```bash
opencli adapter reset twitter
opencli adapter reset --all
```

参数：

| 参数 | 说明 |
| --- | --- |
| `[site]` | 站点名 |
| `--all` | 重置所有本地 override |

## 10. `opencli daemon` 命名空间

daemon 是 OpenCLI 与 Browser Bridge 扩展之间的本地桥。

### `opencli daemon status`

查看 daemon 状态。

> 这个命令已经出现在当前源码中；如果你的本地版本 `opencli daemon --help` 里还没有它，请先升级。

### `opencli daemon stop`

停止 daemon。

```bash
opencli daemon stop
```

## 11. `opencli install`

### 用途

安装一个 OpenCLI 已知的外部 CLI。

```bash
opencli install gh
opencli install docker
```

这个命令的前提是：

- 该 CLI 已存在于 OpenCLI 的 external registry 中
- 当前平台存在对应安装方式

## 12. `opencli register`

### 用途

把任意本地 CLI 注册到 OpenCLI 下面，之后就能用 `opencli <name> ...` 透传调用。

### 常用写法

```bash
opencli register gh
opencli register mycli --binary mycli-real
opencli register obsidian --desc "Obsidian vault management"
opencli register foo --install "brew install foo"
```

### 参数

| 参数 | 说明 |
| --- | --- |
| `<name>` | 注册到 OpenCLI 下的名称 |
| `--binary <bin>` | 实际可执行文件名 |
| `--install <cmd>` | 自动安装命令 |
| `--desc <text>` | 描述 |

### 注册后如何使用

例如你注册了 `gh`：

```bash
opencli gh repo list
opencli gh pr status
```

## 13. `opencli antigravity serve`

### 用途

启动一个与 Anthropic API 兼容的代理服务，面向 Antigravity 应用。

### 常用写法

```bash
opencli antigravity serve
opencli antigravity serve --port 8082
```

当前源码中还支持：

```bash
opencli antigravity serve --timeout 180
```

参数：

| 参数 | 说明 |
| --- | --- |
| `--port <port>` | 服务端口，默认 `8082` |
| `--timeout <seconds>` | 等待回复的最长时间，源码中可用 |

## 14. 常见工作流推荐

### 工作流 A：先诊断，再操作浏览器

```bash
opencli doctor
opencli browser open "https://example.com"
opencli browser state
opencli browser click 8
opencli browser wait text "成功"
```

### 工作流 B：从网页探索到命令生成

```bash
opencli explore "https://example.com/page" --goal "抓取列表"
opencli synthesize .opencli/explore/example.com
opencli generate "https://example.com/page" --goal "抓取列表"
```

### 工作流 C：本地修改官方 adapter

```bash
opencli adapter eject amazon
# 编辑 ~/.opencli/clis/amazon/*
opencli validate amazon
opencli verify amazon --smoke
```

### 工作流 D：把外部工具统一收口到 OpenCLI

```bash
opencli register gh
opencli register docker
opencli gh repo list
opencli docker ps
```

## 15. 文档与版本建议

如果你想确认某条命令在自己本机到底支持哪些参数，最稳的方法永远是：

```bash
opencli --help
opencli browser --help
opencli plugin --help
opencli adapter --help
opencli <command> --help
```

建议把这篇文档和本地 `--help` 一起使用：

- **先看这篇文档**：了解命令意图、使用场景、推荐工作流
- **再看 `--help`**：确认你当前安装版本的真实参数集合


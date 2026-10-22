# JMeter AI Agent 插件 — 打包与安装说明

> 本机已实际验证：源码编译通过、jar 已装入 JMeter 5.6.3、插件类被 JMeter 正确识别、CLI 可运行。

## 一、这个项目是什么（先定性）

它不是 JMeter 的取样器/监听器插件，而是**纯 GUI 增强插件 + 独立 CLI**：

| 层面 | 实现方式 |
|------|----------|
| 入口 | `org.gitee.jmeter.ai.gui.AiMenuCreator implements MenuCreator` |
| 加载机制 | JMeter 启动时用 `ClassFinder` 扫描 `lib/ext/*.jar`，凡实现 `MenuCreator`/`JMeterGUIComponent` 的类自动注册（**没有** `META-INF/services`，靠类扫描，所以 jar 必须放 `lib/ext`） |
| UI | Swing 聊天面板 `AiChatPanel`，挂在「运行(Run) → AI」菜单，并注册工具栏按钮 |
| 能力 | 通过 20+ 个 Tool（create_jmeter_element / run_test / get_test_results…）直接读写当前测试计划 |
| 依赖 | JMeter 5.6.3 API、JDK 17、7 家大模型 SDK（OpenAI/Claude/Ollama/LangSmith/MCP…） |
| 附加 | `JmeterCli`（Main-Class），通过 loopback IPC 远程驱动运行中的 JMeter GUI |

## 二、打包原理（pom.xml 关键点）

1. **maven-shade-plugin** 打「uber jar」= 插件代码 + 全部第三方依赖
   - `artifactSet` 排除 `org.apache.jmeter:*` 与 `net.sf.saxon:*` → **绝不能再带一份 JMeter，类冲突**
   - `ManifestResourceTransformer` 设 `Main-Class: org.gitee.jmeter.ai.cli.JmeterCli`，所以同一个 jar 既能当插件又能 `java -jar` 跑 CLI
   - 排除 `META-INF/*.SF|DSA|RSA`（避免签名校验失败）
2. **maven-assembly-plugin** 另出一份 `-jar-with-dependencies.jar`（无 Main-Class），实际用不到，用 shade 的那个
3. **maven-antrun-plugin（install 阶段）** 自动把 jar → `lib/ext`、`src/main/jmeter-agent/skills` → `bin/jmeter-agent/skills`、模板 md → `bin/jmeter-agent/`、CLI 脚本 → `bin/`

产物：`target/jmeter-agent-0.3.0.jar`（约 145 MB，fat jar）

## 三、构建命令

```bash
# 必须用 JDK 17（本机默认 java 是 1.8，直接 mvn 会报 class file version / 编译失败）
set JAVA_HOME=E:\jdk\jdk17

cd E:/pic/JMeter-AI-Agent-Plugin-master
mvn clean package -DskipTests      # 只出 jar，不复制
# 或
mvn clean install -DskipTests      # 出 jar + 按 pom 的 jmeter.home 自动复制到 JMeter
```

> 首次构建要下 200+ 个依赖（约 7~8 分钟）。国内网络慢可加阿里云镜像。

## 四、原 pom 里必须改的 3 处（已改好，原文件备份为 pom.xml.bak）

| 位置 | 原值（作者本机路径，别人机器必挂） | 改后 |
|------|--------------------------------------|------|
| `<properties>` `jmeter.home` | `D:/apache-jmeter-5.6.3` | `E:/jmeter/apache-jmeter-5.6.3` |
| maven-compiler-plugin | `<executable>D:/IDE/Java/jdk-17.0.8/bin/javac</executable>` + `fork=true` | 删除，靠 `JAVA_HOME` |
| maven-surefire-plugin | `<jvm>D:/IDE/Java/.../java.exe</jvm>` | 删除 |

另外 `run.bat` 里也是作者本人绝对路径，别直接用。

## 五、安装到 JMeter（三件套，缺一不可）

设 `JMETER_HOME = E:/jmeter/apache-jmeter-5.6.3`

```
1) 插件主体
   target/jmeter-agent-0.3.0.jar  →  %JMETER_HOME%\lib\ext\

2) 技能与模板（Agent 的「大脑」资料）
   src\main\jmeter-agent\skills\*            →  %JMETER_HOME%\bin\jmeter-agent\skills\
   src\main\resources\templates\{AGENTS,SOUL,TOOLS,USER}.md
                                              →  %JMETER_HOME%\bin\jmeter-agent\

3) 配置
   jmeter-ai-sample.properties 的内容追加到  %JMETER_HOME%\bin\user.properties
   然后至少填一个 provider 的 key：
       jmeter.ai.default.provider=deepseek
       deepseek.api.key=sk-xxxxx
```

可选：`src/main/resources/bin/jmeter-cli.bat|sh` → `%JMETER_HOME%\bin\`（用 CLI 才需要）

## 六、验证

```bash
# 1) 启动（GUI 需 JDK 17）
set JAVA_HOME=E:\jdk\jdk17
%JMETER_HOME%\bin\jmeter.bat
# 菜单栏 运行(Run) → AI  → 打开聊天面板

# 2) CLI 自检
%JMETER_HOME%\bin\jmeter-cli.bat --help
%JMETER_HOME%\bin\jmeter-cli.bat list        # 需 GUI 以 -Jjmeter.ai.ipc.enabled=true 启动
```

命令行验证插件是否被识别（本次实测输出）：

```
MenuCreator 实现类: [... org.gitee.jmeter.ai.gui.AiMenuCreator]
```

## 七、已知坑

1. **JDK 版本**：JMeter 5.6.3 支持 Java 8+，但插件是 17 编译的，GUI 必须用 JDK 17 启动，否则 `UnsupportedClassVersionError`。
2. **JMeter 版本**：只按 5.6.3 编译。本机原有 5.3 / 5.4.1 不建议混用（可能 `NoSuchMethodError`）。已下载 5.6.3 到 `E:\jmeter\apache-jmeter-5.6.3`。
3. **SLF4J 重复绑定警告**：fat jar 内自带 log4j 2.17.2，与 JMeter 自带的 2.22.1 冲突，启动时打印 `multiple SLF4J bindings`（不影响功能）。想消掉可在 shade 的 `artifactSet` 加：
   ```xml
   <exclude>org.apache.logging.log4j:*</exclude>
   <exclude>org.slf4j:*</exclude>
   ```
4. **java 命令默认是 1.8**：所有涉及插件的启动脚本都要显式指向 JDK 17。
5. **jar 必须放 `lib/ext`**：放 `lib/` 不会触发组件扫描。
6. **AI 菜单不出现**：通常是 ①jar 没进 lib/ext ②JDK 版本不对 ③`jmeter.log` 里有 `MenuCreator` 初始化异常。默认 provider 没有有效 key 时菜单项可能是灰的。

## 八、本机定制：⭐ Star 外链已改为自有仓库

`AiChatPanel.java` 标题栏右侧的 `⭐ Star` 蓝色超链接按钮，原本指向原作者仓库
`https://github.com/azhengzz/JMeter-AI-Agent-Plugin`。现已**保留按钮、替换目标地址**为：

```java
private static final String REPO_URL = "https://github.com/syyymxdzs/jmeter-ai-agent";
```

改动清单（相对原始代码）：

| 改动 | 位置 |
|------|------|
| `REPO_URL` 常量改为自有仓库地址 | `AiChatPanel` 字段区（第 59 行） |
| `createStarLinkButton()` 保留原样（无边框 + 蓝字 + hand 光标） | 标题栏下方 |
| `openRepoUrl()` 保留 `Desktop.browse` 逻辑 | 紧随其后 |

验证：源码与整包扫描均无 `azhengzz` 残留；新地址 `syyymxdzs` 已生效；JMeter 启动冒烟通过。

## 九、怎么配大模型（配置文件 = `bin/user.properties`）

配置位置只有一个：**`%JMETER_HOME%\bin\user.properties`**（本机 `E:\jmeter\apache-jmeter-5.6.3\bin\user.properties`）。
插件通过 `JMeterUtils.getPropDefault()` 读取，所以也可以在启动命令里用 `-Jkey=value` 临时覆盖。
**改完必须重启 JMeter**——配置在服务实例创建时读一次并缓存（`AiServiceFactory.SERVICE_CACHE`），不热加载。

### 9.1 最小配置（两行就够）

```properties
jmeter.ai.default.provider=deepseek
jmeter.ai.default.model=deepseek-chat
deepseek.api.key=sk-你的key
```

三个键的分工：

| 键 | 作用 | 说明 |
|---|---|---|
| `<provider>.api.key` | **必填**，鉴权 | 七个 provider 各一个键 |
| `<provider>.api.base.url` | 选填，改接入点 | 不写则用代码里的默认值（见下表） |
| `jmeter.ai.default.model` | **必填**，模型名 | 全局唯一，没有 per-provider 的 model 键 |
| `jmeter.ai.default.provider` | 兜底 | **仅当模型名匹配不到任何 provider 关键词时才生效** |

### 9.2 ⚠️ 最容易踩的坑：provider 实际由「模型名关键词」决定

`ProviderRegistry.detectProvider()` 先拿**模型名做关键词匹配**，匹配不到才回落到 `jmeter.ai.default.provider`。
而 `ProviderRegistry.getModelsForProvider()` 只会返回全局那一个 `jmeter.ai.default.model`。

所以：**`default.provider=deepseek` 但 `default.model=MiniMax-M2.7` → 实际还是走 MiniMax**，然后
MiniMax 的 key 是空的，就会报鉴权失败。两行必须成对改。

| provider | 模型名必须含的关键词 | 默认 base.url | 典型模型名 |
|---|---|---|---|
| `deepseek` | `deepseek` | `https://api.deepseek.com` | `deepseek-chat` / `deepseek-reasoner` |
| `zhipu` | `zhipu` / `glm` / `zai` | `https://open.bigmodel.cn/api/paas/v4` | `glm-4.6` / `glm-4.5-air` |
| `moonshot` | `moonshot` / `kimi` | `https://api.moonshot.cn/v1` | `kimi-k2.6` / `kimi-k3` |
| `minimax` | `minimax` | `https://api.minimaxi.com/v1` | `MiniMax-M2.7` |
| `openai` | `openai` / `gpt` | `https://api.openai.com/v1` | `gpt-4o` |
| `anthropic` | `anthropic` / `claude` | `https://api.anthropic.com` | `claude-sonnet-4-6` |
| `ollama` | `ollama` / `llama` / `mistral` / `codellama` | `http://localhost:11434/v1` | 本机 `ollama list` 的模型名 |

> 特殊：`anthropic` 走官方 SDK（`ClaudeService`），`ollama` 走 `OllamaAiService`（读 `ollama.host/port`），
> 其余五个走统一 `OpenAICompatibleProvider`。MiniMax 因为返回体不兼容 OpenAI SDK，强制走裸 HTTP 客户端。

### 9.3 七个 provider 的现成片段（复制粘贴，只留一个生效）

```properties
# ---------- DeepSeek ----------
jmeter.ai.default.provider=deepseek
jmeter.ai.default.model=deepseek-chat
deepseek.api.key=sk-xxx
#deepseek.api.base.url=https://api.deepseek.com

# ---------- 智谱 GLM ----------
#jmeter.ai.default.provider=zhipu
#jmeter.ai.default.model=glm-4.6
#zhipu.api.key=xxx.xxx
#zhipu.api.base.url=https://open.bigmodel.cn/api/paas/v4

# ---------- Kimi (Moonshot) ----------
#jmeter.ai.default.provider=moonshot
#jmeter.ai.default.model=kimi-k2.6
#moonshot.api.key=sk-xxx
#moonshot.api.base.url=https://api.moonshot.cn/v1   # 海外账号用 https://api.moonshot.ai/v1

# ---------- MiniMax ----------
#jmeter.ai.default.provider=minimax
#jmeter.ai.default.model=MiniMax-M2.7
#minimax.api.key=xxx

# ---------- OpenAI ----------
#jmeter.ai.default.provider=openai
#jmeter.ai.default.model=gpt-4o
#openai.api.key=sk-xxx
#openai.api.base.url=https://api.openai.com/v1

# ---------- Claude ----------
#jmeter.ai.default.provider=anthropic
#jmeter.ai.default.model=claude-sonnet-4-6
#anthropic.api.key=sk-ant-xxx

# ---------- 本地 Ollama ----------
#ollama.enabled=true
#jmeter.ai.default.provider=ollama
#jmeter.ai.default.model=llama3.1
#ollama.host=http://localhost
#ollama.port=11434
```

### 9.4 其它常用调参（全部全局，无 per-provider 版本）

```properties
jmeter.ai.temperature=0.7          # 0~2，写脚本建议 0.2~0.3
jmeter.ai.max.tokens=65536         # 单次回复上限，别超过模型自身上限
jmeter.ai.reasoning.effort=none    # none/minimal/low/medium/high/xhigh/max
jmeter.ai.context.window.tokens=102400
jmeter.ai.max.tool.iterations=50
ai.chat.show.thinking=true         # 想看模型思考过程就开
jmeter.ai.system.prompt=...        # 自定义系统提示（对所有 provider 生效）
```

> 注意：样例文件里注释写的 `deepseek.temperature=0.3` 这种 per-provider 覆盖**代码里并没有实现**，
> 实际只有 `jmeter.ai.temperature` 生效（Kimi K2.5+/K3 的 `temperature=1.0` 是代码里硬编码的模型级覆盖）。

### 9.5 验证配置生效

1. 启动后打开 AI 面板，标题下发的模型选择器应显示 `provider:model`（例如 `deepseek:deepseek-chat`）
2. 看 `%JMETER_HOME%\bin\jmeter.log`，正常会有：
   ```
   Creating OpenAI-compatible provider: deepseek with base URL: https://api.deepseek.com
   Initialized deepseek provider with model: deepseek-chat
   ```
3. 若出现 `No API key configured for provider: xxx.api.key` → key 没填或填错行；
   若 `401/403` → key 与 base.url 不匹配（常见：Kimi 国内 key 配了 `.ai` 域名）；
   若 `model not found` → `jmeter.ai.default.model` 名字不对或与 provider 不匹配。

## 十、`bin/jmeter-agent/` 目录到底是什么

**一句话：这是 Agent 的「工作区 + 内置技能库」，不是 JMeter 自身文件，删掉插件就"失忆失能"。**
本机现状：219 个文件 / 1.2 MB。

```
bin/jmeter-agent/
├── AGENTS.md      713 B   行为准则（测试实践、排错清单）      ┐
├── SOUL.md        446 B   人格设定（JMeter AI 🎯）            │ 每轮都进 system prompt
├── USER.md        920 B   用户画像（职业/语言/风格，可改）     │
├── TOOLS.md       343 B   工具使用约束（如 exec 超时/黑名单） ┘
├── memory/                MEMORY.md 长期记忆(每轮注入) + HISTORY.md 追加式日志
├── sessions/              jmeter-ai-chat.jsonl 会话持久化（GUI 每次对话写这里）
└── skills/        1.2 MB  3 个技能包
    ├── jmeter/            208 文件 ← 核心
    │   ├── SKILL.md               30 KB  工具清单 + 操作工作流（always:true，每轮全量注入）
    │   ├── legacy-elements.yaml   79 条  无 schema 组件的兜底注册表
    │   └── references/
    │       ├── native/        100 文件  50 个原生组件 ×（.md 说明 + .schema.yaml 字段定义）
    │       ├── functions/      58 文件  JMeter 内置函数手册（__UUID / __timeShift / __groovy…）
    │       ├── gitee-qa/       38 文件  国内扩展组件（S3、HTTP-UD、Git Sampler、概率控制器…）
    │       ├── third-party/     8 文件  SSH Sampler、Stepping/Ultimate Thread Group
    │       ├── standards.md            脚本规范（命名、结构、断言要求）
    │       └── bad-cases.md            踩坑记录
    ├── memory/SKILL.md    两层记忆机制说明（always:true）
    └── skill-creator/     4 文件  教模型自己写新技能包
```

### 关键机制（必须知道的三点）

1. **`.md` 与 `.schema.yaml` 是两类东西**
   - `.schema.yaml`（73 个）是**给代码读的**：`ElementRegistry` / `ComponentSchemaLoader` / `ComponentValidator`
     启动时扫描 `references/**.schema.yaml`，构建"能创建哪些元素、属性名是否合法"的注册表。
     73 schema + 79 legacy ≈ 152 种元素 → **删掉 references 会让 `create_jmeter_element` 直接废掉**，不是"少点文档"而已。
   - `.md`（139 个）是**给模型读的**：除 `always:true` 的 `SKILL.md` 外，其余都不自动注入，
     模型需要时用 `read_file` 现查（`ContextBuilder` 会把这些技能的 description 拼成 XML 索引）。

2. **每轮对话的固定 token 成本**：`AGENTS/SOUL/USER/TOOLS` + `memory/MEMORY.md` + 所有 `always:true` 技能的完整内容
   （最大的就是那 30 KB 的 `jmeter/SKILL.md`）。想省 token 就精简这几个文件，别动 references。

3. **路径是相对 CWD 解析的**：`agent.workspace.path=jmeter-agent` 经 `Paths.get(..).toAbsolutePath()`
   解析成"JVM 工作目录 + jmeter-agent"。`jmeter.bat` 会把 cwd 切到 `bin/`，所以落在这里；
   若从别的目录用 `jmeter -n` 启动，会在那个目录重新生成一套工作区。**想让配置全局稳定，就写绝对路径**：
   ```properties
   agent.workspace.path=E:/jmeter/apache-jmeter-5.6.3/bin/jmeter-agent
   ```

4. **技能优先级**：`{workspace}/skills` > jar 内置 skills。本机两者恰好是同一个目录。
   在 `skills/` 下新建 `xxx/SKILL.md`（frontmatter 带 `name/description/always`）即可扩展能力。

### 隐患提醒

- 重新执行 `mvn install` 时，pom 的 antrun 会 **先 `delete dir bin/jmeter-agent/skills` 再整目录复制**——
  你自己加在 `skills/` 下的自定义技能包会被清掉，务必先备份。
- 初始化判定 `WorkspaceInitializer.isInitialized()` **只看 AGENTS.md 是否存在**。手动复制过 AGENTS.md 的话，
  `memory/MEMORY.md` 等模板不会被补种（本次已手动补上）。`MemoryStore` 有容错（文件不存在就跳过），
  但模板里的栏目提示会缺失。
- 想定制回答风格：直接改 `USER.md`（角色、语言、详略）和 `AGENTS.md`（追加自己的测试规范），重启生效。



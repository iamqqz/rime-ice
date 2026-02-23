# 雾凇拼音 (rime-ice) 项目手把手交接文档

> 📝 **文档说明**：本交接文档面向后续接手的开发人员，全面介绍 rime-ice 项目的架构、核心功能、配置体系、开发流程和维护要点，帮助新成员快速上手并高效维护。

---

## 一、项目概览

### 1.1 项目定位
- **雾凇拼音** 是一套开源、跨平台、功能完备的 Rime 输入法配置方案
- 主要面向中文用户，提供全拼、双拼（小鹤/微软/搜狗等）等多种输入方式
- 核心价值：词库质量高、功能丰富、长期维护、文档完善、社区活跃

### 1.2 技术栈
- **底层框架**：Rime 输入法引擎（librime）
- **前端支持**：鼠须管（macOS）、小狼毫（Windows）、fcitx5-rime（Linux）、同文（Android）、仓输入法（iOS）
- **扩展语言**：Lua 脚本（所有高级功能均通过 Lua 实现）
- **词库处理**：Go 语言编写的构建脚本（`others/script/main.go`）
- **配置格式**：YAML（Rime 标准配置格式）

### 1.3 项目结构概览
```
├── rime_ice.schema.yaml        # 主全拼方案配置
├── rime_ice.dict.yaml          # 词典组合配置（引用 cn_dicts/ 下的词典）
├── double_pinyin_*.schema.yaml # 各种双拼方案
├── melt_eng.schema.yaml        # 英文输入方案
├── radical_pinyin.schema.yaml  # 拆字方案
├── t9.schema.yaml              # 九宫格方案
├── cn_dicts/                   # 中文词典目录
│   ├── 8105.dict.yaml          # 8105常用字表
│   ├── base.dict.yaml          # 基础词库
│   ├── ext.dict.yaml           # 扩展词库（含多音字注音）
│   └── tencent.dict.yaml       # 腾讯词向量大词库
├── en_dicts/                   # 英文词典目录
├── opencc/                     # OpenCC 简繁转换配置
├── lua/                        # Lua 扩展模块（所有功能的核心）
├── others/                     # 其他资源（demo、脚本、文档等）
├── default.yaml                # 全局配置（方案列表、菜单、开关等）
└── README.md                   # 项目主文档
```

---

## 二、核心架构详解

### 2.1 方案文件体系

| 文件 | 说明 | 关键特性 |
|------|------|----------|
| `rime_ice.schema.yaml` | 主全拼方案 | 支持错音提示、长词优先、Unicode输入、日期时间、农历、计算器、UUID生成等 |
| `double_pinyin_*.schema.yaml` | 各种双拼方案 | 小鹤、微软、搜狗、智能ABC、紫光、拼音加加等，每个方案都有独立的 algebra 规则 |
| `melt_eng.schema.yaml` | 英文输入 | 作为次翻译器挂载，支持中英混输 |
| `radical_pinyin.schema.yaml` | 拆字方案 | 支持反查（uU+拼音）和辅码查找（拼音+`+拆字） |
| `t9.schema.yaml` | 九宫格方案 | 专为仓输入法设计 |

### 2.2 Lua 扩展系统（项目核心）

所有高级功能都通过 Lua 实现，模块化设计清晰：

**翻译器（Translator）** - 生成候选项：
- `date_translator.lua`：日期/时间/星期/ISO/时间戳（触发词：rq/sj/xq/dt/ts）
- `lunar.lua`：农历（触发词：nl）
- `uuid.lua`：UUID v4 生成（触发词：uuid）
- `number_translator.lua`：数字/人民币大写（触发词：R+数字）
- `calc_translator.lua`：简易计算器（触发词：cC+算式）
- `unicode.lua`：Unicode 输入（触发词：U+码位）

**过滤器（Filter）** - 修改候选项：
- `corrector.lua`：错音错字智能提示（核心功能）
- `autocap_filter.lua`：英文自动大写
- `v_filter.lua`：v模式符号优先
- `pin_cand_filter.lua`：置顶候选项（可按编码配置）
- `long_word_filter.lua`：长词优先算法
- `reduce_english_filter.lua`：降低短英文词位置
- `search.lua`：拆字辅码查找

**处理器（Processor）** - 处理按键事件：
- `select_character.lua`：以词定字（[ 和 ] 键）

**工具类**：
- `en_spacer.lua`、`cn_en_spacer.lua`：中英混输空格处理
- `t9_preedit.lua`：九宫格支持
- `debuger.lua`：调试工具

### 2.3 配置层级体系

Rime 配置采用分层设计，优先级从高到低：

1. **方案文件**（`*.schema.yaml`）：各方案独立配置，最高优先级
2. **方案自定义**（`*.custom.yaml`）：用户对特定方案的覆盖配置
3. **全局配置**（`default.yaml`）：所有方案共享的基础配置
4. **全局自定义**（`default.custom.yaml`）：用户对全局配置的补丁
5. **平台特定配置**：`squirrel.yaml`（macOS）、`weasel.yaml`（Windows）

> 💡 **重要提示**：修改配置时，优先使用 `*.custom.yaml` 文件进行 patch，避免直接修改主配置文件，便于升级维护。

---

## 三、开发与维护指南

### 3.1 词典管理

**词典目录结构**：
- `cn_dicts/`：中文词典（8105常用字、base基础词库、ext扩展词库、tencent大词库）
- `en_dicts/`：英文词库（en常用词、en_ext扩展词、cn_en中英混输）
- `opencc/`：简繁转换和emoji配置

**词典更新流程**：
1. 编辑对应词典文件（如 `cn_dicts/base.dict.yaml`）
2. 词典格式：`词条<TAB>拼音<TAB>权重`（权重可选但建议填写）
3. 运行构建脚本：`cd others/script && go run main.go`（自动检查、注音、赋权）
4. 修改 `rime_ice.dict.yaml` 中的 `import_tables` 引用关系
5. 重新部署 Rime 配置

### 3.2 Lua 功能开发

**Lua 模块引用规则**：
- 在方案中引用时需加 `*` 前缀：`lua_translator@*date_translator`
- 所有 Lua 文件位于 `lua/` 目录下
- 每个模块都有详细的注释说明用法

**调试技巧**：
- 使用 `debuger.lua` 辅助调试（在日志中输出调试信息）
- 查看 Rime 日志排查错误（macOS: `~/Library/Logs/Rime/`，Windows: `%TEMP%\rime.*`）
- Lua 错误会记录在日志中，重点关注语法错误和运行时异常

### 3.3 双拼支持要点

- 每个双拼方案有独立的 `speller/algebra` 拼写规则
- 英文方案 (`melt_eng`) 和拆字方案 (`radical_pinyin`) 在双拼下需要匹配的 `algebra` 引用
- 双拼补丁通过 `others/recipes/config:schema=xxx` 配方实现
- 注意双拼方案中 `recognizer/patterns` 的前缀设置（如 `V` 而非 `v`）

### 3.4 平台兼容性注意事项

| 平台 | 注意事项 | 解决方案 |
|------|----------|----------|
| **小狼毫 (Windows)** | 存在兼容性问题，部分功能可能异常 | 建议备用其他输入法，关注 issue #197 |
| **鼠须管 (macOS)** | 0.16.0-0.18.0 版本有特定问题 | 参考 issue #1062，建议升级到最新版 |
| **fcitx5-rime** | 支持卷轴模式 | 启用 `scroll_mode: true` 配置 |
| **仓输入法 (iOS)** | 九宫格需同时启用方案和布局 | 在「输入方案设置」启用九宫格，在「键盘布局」选择「中文9键」 |
| **Arch Linux** | AUR 包 `rime-ice-git` | 推荐使用补丁方式启用，避免冲突 |

---

## 四、常见问题与解决方案

### 4.1 部署问题
- **问题**：部署后无反应或功能异常
- **排查步骤**：
  1. 检查 Rime 前端版本是否 ≥ 1.8.5
  2. 确认安装了 `librime-lua` 依赖
  3. 查看 Rime 日志文件中的错误信息
  4. 检查 YAML 文件格式是否正确（缩进、冒号后空格）

### 4.2 词典问题
- **问题**：词库不生效或候选词顺序异常
- **解决方案**：
  1. 检查 `rime_ice.dict.yaml` 中的 `import_tables` 是否正确引用词典
  2. 确认词典文件名和路径是否正确
  3. 运行 `go run main.go` 构建脚本验证词典格式
  4. 检查词典权重设置是否合理

### 4.3 Lua 功能问题
- **问题**：Lua 功能不触发或报错
- **排查方法**：
  1. 检查方案文件中 `translators` 或 `filters` 部分的引用是否正确
  2. 确认 `recognizer/patterns` 中的正则表达式是否匹配触发条件
  3. 查看日志中是否有 Lua 错误信息
  4. 使用 `debuger.lua` 添加调试输出

---

## 五、项目维护规范

### 5.1 提交规范
- **提交信息格式**：`feat: 新增XXX功能` / `fix: 修复XXX问题` / `chore: 更新XXX文档`
- **关联 Issue**：在提交信息末尾添加 `Closes #123` 或 `Fixes #123`
- **CI 触发**：提交信息末尾添加 `[build]` 触发自动构建

### 5.2 文档更新
- **README.md**：保持最新，包含安装说明、功能列表、配置示例
- **CLAUDE.md**：项目技术文档，详细说明架构和开发指南
- **CHANGELOG.md**：记录每个版本的重大变更

### 5.3 测试流程
1. 修改配置后，在 Rime 前端重新部署
2. 测试核心功能：全拼/双拼输入、英文混输、日期时间、Unicode、计算器等
3. 在不同平台（macOS/Windows/Linux）验证兼容性
4. 查看日志确认无错误信息

---

## 六、关键联系人与资源

- **项目维护者**：Dvel（GitHub: @iDvel）
- **主要贡献者**：@Mirtle、@Huandeep、@Lithium-7 等
- **文档中心**：[雾凇拼音官网](https://dvel.me/posts/rime-ice/)、[GitHub Wiki](https://github.com/iDvel/rime-ice/wiki)
- **问题反馈**：[GitHub Issues](https://github.com/iDvel/rime-ice/issues)
- **社区交流**：校对标准论坛、Rime 官方社区

> 🌟 **最后提醒**：Rime 配置的本质是「声明式编程」，理解每个配置项的作用比死记硬背更重要。遇到问题时，先查看日志，再查阅文档，最后参考已有 issue。

---

*文档最后更新：2026年2月15日*


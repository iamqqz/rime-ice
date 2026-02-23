# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

雾凇拼音 是一套完整的 Rime 输入法配置，提供全拼、双拼方案，包含中英文词库、Lua 扩展和 OpenCC 集成。支持跨平台 Rime 前端（鼠须管/macOS、小狼毫/Windows、fcitx5-rime/Linux、同文/Android、仓输入法/iOS）。

## 核心架构

### 方案文件

- `rime_ice.schema.yaml` - 主全拼方案，核心配置
- `rime_ice.dict.yaml` - 词典组合（通过 `import_tables` 引用 `cn_dicts/` 下的词典）
- `double_pinyin*.schema.yaml` - 各种双拼方案（小鹤、微软、搜狗、智能ABC、紫光、拼音加加）
- `melt_eng.schema.yaml` + `melt_eng.dict.yaml` - 英文输入（作为次翻译器挂载）
- `radical_pinyin.schema.yaml` + `radical_pinyin.dict.yaml` - 拆字方案（反查及辅码）
- `t9.schema.yaml` - 九宫格方案（仓输入法专用）

### 词典结构

- `cn_dicts/` - 中文词典：
  - `8105.dict.yaml` - 通用规范汉字表（8105个常用字）
  - `41448.dict.yaml` - 大字表（默认未启用）
  - `base.dict.yaml` - 基础词库（两字词及调频）
  - `ext.dict.yaml` - 扩展词库（含多音字注音）
  - `tencent.dict.yaml` - 腾讯词向量（大词库，无注音）
  - `others.dict.yaml` - 杂项
- `en_dicts/` - 英文词典（`en`、`en_ext` 及中英混输 `cn_en`）
- `opencc/` - OpenCC 配置（emoji 及简繁转换）

### Lua 扩展 (`lua/`)

所有 Lua 模块在方案中引用时需加 `*` 前缀（如 `lua_translator@*date_translator`）：

**翻译器**（根据输入生成候选项）：
- `date_translator.lua` - 日期/时间/星期/ISO 8601/时间戳（可配置触发词：`rq`、`sj`、`xq`、`dt`、`ts`）
- `lunar.lua` - 农历（触发词：`nl`）
- `uuid.lua` - UUID v4 生成（触发词：`uuid`）
- `number_translator.lua` - 数字/人民币大写（触发词：`R` + 数字）
- `calc_translator.lua` - 计算器（触发词：`cC` + 算式）
- `unicode.lua` - Unicode 输入（触发词：`U` + 码位）
- `force_gc.lua` - 强制垃圾回收

**过滤器**（修改/重排候选项）：
- `corrector.lua` - 错音错字提示
- `autocap_filter.lua` - 英文自动大写
- `v_filter.lua` - v 模式符号优先
- `pin_cand_filter.lua` - 置顶候选项（可按编码配置）
- `long_word_filter.lua` - 长词优先
- `reduce_english_filter.lua` - 降低短英文词位置
- `search.lua` - 拆字辅码查找

**处理器**（处理按键事件）：
- `select_character.lua` - 以词定字（`[` 和 `]` 键）

**工具类**：
- `cold_word_drop/` - 冷词降频系统
- `en_spacer.lua`、`cn_en_spacer.lua` - 中英混输空格处理
- `t9_preedit.lua` - 九宫格支持
- `debuger.lua`、`is_in_user_dict.lua` - 工具函数

### 配置层级

1. `default.yaml` - 全局配置（方案列表、菜单、开关、中西文切换、共用的标点/按键/识别器）
2. `default.custom.yaml` - 用户自定义（对 default.yaml 的补丁）
3. 方案文件 (`*.schema.yaml`) - 各方案独立配置
4. `*.custom.yaml` - 各方案用户自定义

平台特定配置：`squirrel.yaml`（macOS）、`weasel.yaml`（Windows）

### 词典构建脚本

`others/script/main.go` - Go 编写的词典处理脚本：
- Emoji 检查和生成
- 中英混输词库更新 (`rime.CnEn()`)
- 自动注音 (`rime.Pinyin()`)
- 权重赋值 (`rime.AddWeight()`)
- 词典校验 (`rime.Check()`)
- 多音字检查 (`rime.CheckPolyphone()`)

在 `others/script/` 目录下运行 `go run main.go` 执行。

触发 CI 自动构建：提交信息末尾添加 ` [build]`。

## 安装/更新方式

1. **手动安装**：复制所有文件到 Rime 用户目录，然后部署
2. **Git 安装**：`git clone --depth 1 https://github.com/iDvel/rime-ice.git Rime`
3. **东风破 (rime-install)**：使用 `others/recipes/` 中的配方：
   - `full` - 完整安装
   - `all_dicts` - 所有词典
   - `cn_dicts` - 仅中文词典
   - `en_dicts` - 仅英文词典
   - `opencc` - OpenCC 配置
   - `config:schema=xxx` - 双拼补丁

## 开发说明

### 添加/修改词典条目

- 编辑 `cn_dicts/` 或 `en_dicts/` 中的词典文件
- 词典条目格式：`词条<TAB>拼音<TAB>权重`（权重可选但建议填写）
- `rime_ice.dict.yaml` 通过 `import_tables` 组合词典
- 修改后运行构建脚本进行校验和处理

### 用户自定义

使用 `*.custom.yaml` 文件通过 `patch:` 语法覆盖配置：
```yaml
patch:
  # 追加绑定
  key_binder/bindings/+:
    - { when: paging, accept: comma, send: Page_Up }
```

### 双拼支持

- 每个双拼方案有独立的 `speller/algebra` 拼写规则
- 英文方案 (`melt_eng`) 和拆字方案 (`radical_pinyin`) 在双拼下需要匹配的 `algebra` 引用
- 参考 `melt_eng.schema.yaml` 和 `radical_pinyin.schema.yaml` 中的 algebra 引用选项

### 测试修改

1. 修改文件后，在 Rime 前端重新部署
2. 查看 Rime 日志排查错误（macOS 在 `~/Library/Logs/Rime/`，Windows 在 `%TEMP%\rime.*`）
3. Lua 错误会记录在日志中；可用 `debuger.lua` 辅助调试

## 平台特定说明

- **小狼毫 (Windows)**：存在兼容性问题，建议备用其他输入法
- **鼠须管 (macOS)**：0.16.0-0.18.0 版本有特定问题需注意
- **fcitx5-rime**：支持卷轴模式
- **仓输入法 (iOS)**：内置雾凇拼音，九宫格需同时启用方案和布局
- **Arch Linux**：使用 AUR 包 `rime-ice-git`，建议用补丁方式启用

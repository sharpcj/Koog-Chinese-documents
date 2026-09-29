# Koog 官方文档中文翻译

这是 JetBrains Koog 官方文档的中文翻译与本地静态站点版本。

官方文档来源：<https://docs.koog.ai/>

本仓库内容基于官方文档整理和翻译，目标是便于中文读者离线阅读、检索和对照学习。翻译内容不添加官方文档之外的功能说明、示例或结论。

## 内容结构

```text
.
├── README.md                 # 仓库说明
├── .gitignore                # Git 忽略规则
├── manifest.json             # 页面清单与源文件映射
├── _source_en/               # 官方英文文档抽取版本
├── md/                       # 中文 Markdown 译文，按官方侧边栏层级组织
│   ├── README.md             # Markdown 文档目录索引
│   ├── documentation/
│   ├── examples/
│   └── why-koog/
└── site/                     # 本地静态 HTML 站点
    ├── index.html            # 静态站入口
    ├── styles.css
    └── *.html
```

## 如何阅读

### 阅读静态网页

直接用浏览器打开：

```text
site/index.html
```

静态站点包含中文正文、左侧导航和页面间跳转，适合连续阅读。

### 阅读 Markdown

Markdown 译文入口：

```text
md/README.md
```

Markdown 文件按接近官方文档左侧菜单的层级组织，适合在 GitHub 上浏览、审阅和维护。

## 当前内容范围

当前仓库包含：

- 官方文档英文源页面：88 个
- 中文 Markdown 译文页面：88 个
- 静态 HTML 页面：89 个，包括 `site/index.html`
- 页面清单：`manifest.json`

## 目录说明

### `_source_en/`

保存从 Koog 官方文档抽取的英文 Markdown 源文档，用于后续对照、校验和增量更新。

### `md/`

保存中文 Markdown 译文。目录结构按照官方文档左侧菜单层级整理，主要包括：

- `documentation/overview/`
- `documentation/quickstart/`
- `documentation/agents/`
- `documentation/prompts/`
- `documentation/strategies/`
- `documentation/tools/`
- `documentation/features/`
- `documentation/a2a/`
- `documentation/advanced-usage/`
- `examples/`
- `why-koog/`

### `site/`

保存已经生成好的静态 HTML 页面。该目录不依赖构建服务，直接打开 `site/index.html` 即可阅读。

## GitHub Pages 发布建议

如果希望通过 GitHub Pages 发布静态站点，可以使用以下方式之一：

1. 将仓库根目录作为 Pages 来源，然后访问 `site/index.html`。
2. 在发布流程中把 `site/` 目录作为站点产物发布。
3. 如果使用 GitHub Actions，可以把 `site/` 目录上传为 Pages artifact。

## 维护建议

后续如果 Koog 官方文档更新，建议按以下顺序维护：

1. 更新 `_source_en/` 中的英文源文档。
2. 对照 `manifest.json` 检查新增、删除或变更页面。
3. 更新 `md/` 中对应的中文译文。
4. 如需更新网页阅读版，再重新生成 `site/`。
5. 校验页面数量、链接、代码块和 URL 是否保持一致。

## 说明

- Koog 是 JetBrains 的项目，官方文档版权归原作者所有。
- 本仓库仅保存中文翻译和本地阅读站点，内容来源于官方文档。
- 如需确认最新内容，请以官方文档为准：<https://docs.koog.ai/>

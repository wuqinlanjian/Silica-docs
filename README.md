简体中文 | [English](README.en.md)

# Silica 文档

**Silica** 的双语文档站点，使用 MkDocs 与 Material 主题构建。Silica 是面向 LeviLamina Minecraft 基岩版客户端模组的嵌入式声明式 UI 框架，源码在 [Silica](https://github.com/wuqinlanjian/Silica) 仓库。

在线访问：<https://silica.feather.qzz.io/>（中文）· <https://silica.feather.qzz.io/en/>（English）。

中文站点位于 `/`，英文站点位于 `/en/`；页面语言选择器可以在当前页面直接切换，无需回到首页。

## 内容

| 章节 | 内容 |
| --- | --- |
| 项目总览 | 架构、渲染流程与核心概念 |
| 快速开始 | 环境、构建、部署与最小集成 |
| DSL 教程 | 语法、布局、控件、绑定、脚本、组件、样式、动画、列表、图片 |
| 控件参考 | 内置控件与常用属性 |
| C++ 集成 | 加载文档、事件回调、`DataStore`、诊断日志 |
| 常见问题 | 阅读错误信息与处理常见症状 |
| 示例集锦 | 设置面板、对话框、动态列表、开关按钮 |

## 本地预览

```bash
python -m pip install -r requirements.txt
python -m mkdocs serve
```

预览地址通常为 `http://127.0.0.1:8000/`；修改 `.md` 文件与 `mkdocs.yml` 后站点会自动重建。

## 构建

```bash
python -m mkdocs build
```

输出在 `site/`：中文站点在根目录，英文站点在 `site/en/`。仓库不包含自动部署工作流，站点可作为纯静态文件任意托管。

## 部署

本站发布在 <https://silica.feather.qzz.io/>（中文）与 <https://silica.feather.qzz.io/en/>（英文），使用 Cloudflare Pages 构建，在控制台按以下配置连接本仓库：

| 配置项 | 值 |
| --- | --- |
| 构建命令 | `python -m pip install -r requirements.txt && python -m mkdocs build` |
| 输出目录 | `site` |
| 根目录 | 仓库根目录（留空） |

若构建镜像自带的 Python 版本过低，添加环境变量 `PYTHON_VERSION=3.12`。

## 添加页面

1. 新建 `docs/<name>.md`（中文）与 `docs/<name>.en.md`（英文），两者的标题与锚点保持一致。
2. 在 `mkdocs.yml` 的 `nav` 中登记中文文件；i18n 插件会自动解析对应的 `.en.md`。
3. 在 `en` 语区的 `nav_translations` 中补上该标题的英文翻译。

## 许可证

MIT，见 [LICENSE](LICENSE)。

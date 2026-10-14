# HermesAgent 通用安装提示词（国内镜像优化版）

把这份 Markdown 交给任意 AI Agent，即可在国内网络环境的 Linux 服务器上从零安装最新版 Hermes Agent。

## 解决什么问题？

Hermes Agent 的安装依赖 GitHub Releases 和 PyPI 官方源，这些源在国内常被限速到几十 KB/s——
直连安装会卡死数小时（实测卡 40 分钟连 uv 都下不完）。本项目提供全套镜像加速方案：

| 环节 | 官方源（限速） | 本项目方案（实测） |
|---|---|---|
| 工具包下载 | GitHub ~30KB/s | Nous 官方镜像 **4.6 MB/s** |
| Python 依赖 | PyPI ~5KB/s | 阿里云 PyPI **2.7~20 MB/s** |
| Python 解释器 | GitHub Releases | 南大镜像 **13 MB/s** |
| npm 包 | npmjs.org | npmmirror **7.5~10.7 MB/s** |

**结果：全流程 5~10 分钟装完**（对比直连的数小时）。

## 特点

- **完全自包含**：三个配套脚本内嵌在文档中，不依赖任何预置环境
- **面向 AI Agent**：第一节是可直接执行的"执行契约"，Agent 读完能自主完成安装
- **版本自适应**：脚本动态读取版本号，Hermes 升级后依然有效
- **经过实战**：基于 v0.22 PM 架构 + 阿里云 ECS 实测沉淀，含完整踩坑记录

## 用法

1. 下载本仓库的 `HermesAgent通用安装提示词.md`
2. 把它发给你的 AI 编程助手（Claude / ChatGPT / Cursor / OpenCode 等）
3. 说："照这个文档在这台服务器上安装 Hermes Agent"
4. 剩下交给它

## 核心原理（为什么直连会卡死）

Hermes v0.21+ 由 **PM 包管理器**负责下载，共三层，每层都可能卡在限速上。
关键认知：**所有"镜像 fallback"只在网络错误时触发；限速不算失败，兜底永远不触发**。

| 层 | 下载内容 | 卡点 | 破解 |
|---|---|---|---|
| 1 | bootstrap uv（~21MB） | GitHub Releases 限速 | 用官方镜像预置到 `tools/uv-<版本>/` |
| 2 | PM 工具包（9 个，~560MB） | GitHub/nodejs.org 慢源 | 按 sha256 预置进 `tools/fetch-<sha256>/` 缓存 |
| 3 | Python 依赖 venv（300+ 包） | `uv.lock` 里全是 pythonhosted URL | 批量替换为阿里云镜像前缀（哈希校验不变） |

## 免责声明

镜像方案仅替换下载地址（哈希校验保持不变，安全性无损），不改动 Hermes 官方架构。
使用前请自行评估镜像源可信度。

---

*基于 Hermes Agent v0.22.x（PM 架构）实测 · 适用于国内网络环境从零安装场景*
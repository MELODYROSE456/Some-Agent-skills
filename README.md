# 🚀 AI-Assisted Dev Toolkit / AI 辅助研发效能工具链

本项目是一套基于大语言模型（LLM）定制的 Agent Skill 工具链，旨在解决日常软件开发中**需求模糊、架构腐化、破坏性操作风险高**等痛点，将 AI 从“代码打字机”升级为“具备架构思维的研发助理”。

## ✨ 核心特性 (Key Features)

*   🧠 **`need-quest` (需求澄清)**：基于 MECE 原则，强制 AI 在编码前进行多维度追问（每轮限制 3-4 个问题防疲劳），当需求把握度 ≥ 90% 时才输出代码，大幅降低返工率。
*   🗺️ **`arch-map` (架构可视化)**：扫描 C/C++ 项目 `include` 依赖，自动生成 Mermaid 分层架构图。支持追踪本次代码改动并用 **红色高亮** 标注，内置跨层依赖预警机制，防止架构腐化。
*   🛡️ **Global Guardrails (全局安全护栏)**：通过全局 System Prompt 限制 AI 对本地文件的增删改权限，强制引入 **Human-in-the-Loop（人类在环）** 授权机制，杜绝 AI 幻觉导致代码库破坏。

## 🛠️ 快速开始 (Quick Start)

1. 下载本项目中的 `skills` 目录。
2. 将 `skills/need-quest` 和 `skills/arch-map` 文件夹，分别复制到你的 DeepSeek Harness (DSH) 或兼容框架的 Agent 配置目录中。
3. 确保框架能正确读取各自目录下的 `SKILL.md` 文件。
4. *(规划中)* 未来将提供 `start_dsh.bat` 脚本实现一键启动，优化开发者体验(DX)。

## 🤝 贡献与反馈
欢迎提交 Issue 探讨更多 AI 效能玩法。如果你有更好的 Prompt 架构思路，欢迎交流。

## 📄 License
MIT License

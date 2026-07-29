# codex_ponytail

在 Codex App/CLI 中以固定提交、安全审查 hook 的方式安装 [Ponytail](https://github.com/DietrichGebert/ponytail)。

## 安装文档

- **代理默认：[简体中文](docs/ponytail-codex-install-guide.zh-CN.md)**
- [English](docs/ponytail-codex-install-guide.en.md)
- [한국어](docs/ponytail-codex-install-guide.ko.md)

历史已审查基线：Ponytail `4.8.4`，提交 `16f29800fd2681bdf24f3eb4ccffe38be3baec6b`，Codex CLI `0.145.0`。

安装代理必须先解析上游 `main` 的最新完整 SHA，审查它相对基线的变更和 hook，测试通过后再固定该 SHA。文档不会把旧基线当作永久安装目标，也不会直接安装符号引用 `main`。

本仓库只提供 Codex 安装运行手册，不复制或修改 Ponytail 源码，也不隶属于上游项目。

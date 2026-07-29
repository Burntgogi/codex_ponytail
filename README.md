# codex_ponytail

An independent case study for installing
[Ponytail](https://github.com/DietrichGebert/ponytail) in Codex App and CLI by
reviewing current upstream behavior and pinning an immutable commit.

## Guides

- **Canonical source:** [English](docs/ponytail-codex-install-guide.en.md)
- Faithful translation: [한국어](docs/ponytail-codex-install-guide.ko.md)
- Experimental compressed agent edition: [简体中文](docs/ponytail-codex-install-guide.zh-CN.md)

The verified 2026-07-29 case used Ponytail `4.8.4`, commit
`16f29800fd2681bdf24f3eb4ccffe38be3baec6b`, and Codex CLI `0.145.0`.
This is historical evidence, not a permanent installation target.

The durable model is: investigate the current Ponytail repository and local
Codex interface, review the candidate and every hook, obtain approval for the
exact user-level change, then pin and verify a full commit SHA. Never install a
symbolic `main` reference as the plugin source.

This repository neither copies Ponytail source nor distributes an installer.
It is not affiliated with the upstream project.

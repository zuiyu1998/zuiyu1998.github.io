+++
title = "pre-commit 的使用"
date = 2025-02-24

[taxonomies]
tags = ["pre-commit", "教程", "godot", "rust"]
+++

pre-commit 是一个使用 git hook 来实现代码规范的工具。通过这个工具可以使代码结构更规整，更有可读性。

<!-- more -->

# 环境设置

pre-commit 需要 Python 环境。通过 Python 环境的 pip 可以很快的安装 pre-commit。代码如下：

```python
pip install pre-commit
```

# 使用流程

在项目初始化的目录下，使用如下命令就可以使用 pre-commit。

```python
pre-commit install
```

这里比较复杂的是.pre-commit-config.yaml 文件的配置。
pre-commit 使用这个文件来使用各种钩子。

> .pre-commit-config.yaml 在项目的目录下

# rust 的 .pre-commit-config.yaml 配置文件

一般情况下 rust 项目使用的 pre-commit-config 配置。

```yaml
fail_fast: false
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.3.0
    hooks:
      - id: check-byte-order-marker
      - id: check-case-conflict
      - id: check-merge-conflict
      - id: check-symlinks
      - id: check-yaml
      - id: end-of-file-fixer
      - id: mixed-line-ending
      - id: trailing-whitespace
  - repo: https://github.com/psf/black
    rev: 22.10.0
    hooks:
      - id: black
  - repo: local
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        description: Format files with rustfmt.
        entry: nu -c 'cargo fmt -- --check'
        language: rust
        files: \.rs$
        args: []
      - id: typos
        name: typos
        description: check typo
        entry: nu -c 'typos'
        language: rust
        files: \.*$
        pass_filenames: false
      - id: cargo-check
        name: cargo check
        description: Check the package for errors.
        entry: nu -c 'cargo check --all'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-clippy
        name: cargo clippy
        description: Lint rust sources
        entry: nu -c 'cargo clippy --all-targets --all-features --tests --benches -- -D warnings'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-test
        name: cargo test
        description: unit test for the project
        entry: nu -c 'cargo test run --all-features'
        language: rust
        files: \.rs$
        pass_filenames: false
```

注意使用之前需要安装对应的环境:

```nu
cargo install typos-cli
cargo install cargo-clippy
cargo install cargo-nextest
```

# godot 的 .pre-commit-config.yaml 配置文件

一般情况下 godot 项目使用的 .pre-commit-config 配置。

```yaml
fail_fast: false
repos:
  # GDScript Toolkit
  - repo: https://github.com/Scony/godot-gdscript-toolkit
    rev: 4.2.2
    hooks:
      - id: gdlint
        name: gdlint
        description: 'gdlint - linter for GDScript'
        entry: gdlint
        language: python
        language_version: python3
        require_serial: true
        types: [gdscript]
      - id: gdformat
        name: gdformat
        description: 'gdformat - formatter for GDScript'
        entry: gdformat
        language: python
        language_version: python3
        require_serial: true
        types: [gdscript]
```
注意使用之前需要安装对应的环境:

```nu
pip3 install git+https://github.com/Scony/godot-gdscript-toolkit.git
```
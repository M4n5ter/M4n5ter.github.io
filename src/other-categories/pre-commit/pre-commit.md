# pre-commit

一个用于 git commit 提交自动前处理自定义操作的工具

## 安装

```bash
$ pip3 install pre-commit
```

## 示例

``` bash
$ mkdir example-registry && cd example-registry
$ git init
$ touch .pre-commit-config.yaml
......
```

.pre-commit-config.yaml:

```yaml
fail_fast: false
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
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
    rev: 19.3b0
    hooks:
      - id: black
  - repo: https://github.com/crate-ci/typos
    rev: v1.8.1
    hooks:
      - id: typos
  - repo: local
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        description: Format files with rustfmt.
        entry: bash -c 'cargo fmt -- --check'
        language: rust
        files: \.rs$
        args: []
      - id: cargo-deny
        name: cargo deny check
        description: Check cargo dependencies
        entry: bash -c 'cargo deny check'
        language: rust
        files: \.rs$
        args: []
      - id: cargo-check
        name: cargo check
        description: Check the package for errors.
        entry: bash -c 'cargo check --all'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-clippy
        name: cargo clippy
        description: Lint rust sources
        entry: bash -c 'cargo clippy --all-targets --all-features --tests --benches -- -D warnings'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-test
        name: cargo test
        description: unit test for the project
        entry: bash -c 'cargo nextest run --all-features'
        language: rust
        files: \.rs$
        pass_filenames: false

```

```bash
# 这将会在 .git 内创建 hooks，来保证在 git commit 之前顺序执行 .pre-commit-config.yaml 中定义的步骤
$ pre-commit install
```

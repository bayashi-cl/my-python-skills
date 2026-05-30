---
name: python-lint
description: Python プロジェクトに lint / formatter を導入する、または既存の設定を調整するとき。ツール選定、ルールセット選び、抑制 (noqa / per-file-ignores) の書き方、format との両立、CI 連携を含む。black / isort / flake8 / pyupgrade / autopep8 からの置き換えにもこのスキルを使う。
---

# python-lint

## ツール選定

- **第一選択は ruff** (lint + formatter を 1 つで完結、Rust 実装で高速、isort/pyupgrade/bugbear 等の主要ルールを内蔵)。
- ruff で代替できないルールが必要な場面に限り、追加のリンタを併用する (実用上はほぼ発生しない)。
- 採用したら **black / isort / flake8 / pyupgrade / autopep8 / pydocstyle を同居させない**。差分が二重に出てツール戦争になる。
- 設定は `pyproject.toml` の `[tool.ruff]` セクションに集約し、`ruff.toml` を分けない。

## 導入手順

1. `uv add --dev ruff`

1. `pyproject.toml` に以下を追記:

   ```toml
   [tool.ruff]
   line-length = 100
   target-version = "py312"
   src = ["src", "tests"]

   [tool.ruff.lint]
   select = [
       "E", "F", "W",    # pycodestyle / pyflakes
       "I",              # isort
       "B",              # bugbear
       "UP",             # pyupgrade
       "SIM",            # simplify
       "RUF",            # ruff 固有
       "N",              # pep8-naming
       "PTH",            # pathlib 化
       "TCH",            # type-checking import 分離
       "PL",             # pylint subset
   ]
   ignore = [
       "E501",    # line-too-long は formatter に任せる
       "PLR0913", # 引数多すぎは API 設計上必要なケースがある
   ]

   [tool.ruff.lint.per-file-ignores]
   "tests/**" = ["S101", "PLR2004"]  # assert と magic number を許可
   "__init__.py" = ["F401"]          # re-export を許可

   [tool.ruff.lint.isort]
   known-first-party = ["<package_name>"]

   [tool.ruff.format]
   docstring-code-format = true
   ```

1. `uv run ruff check --fix` と `uv run ruff format` を実行し、**初期適用差分を 1 コミットにまとめる** (混在すると以降のレビューが死ぬ)。

## ルール選定の指針

- `select` は **カテゴリ単位** で書き、個別ルールの羅列にしない (ruff のバージョンアップで新ルールが自動で乗る)。
- ルールを `ignore` するときは **理由をコメントで残す** (上記例参照)。理由が説明できないなら有効のままにする。
- プロジェクト特有の例外は `per-file-ignores` に書く。コードに `# noqa` を撒かない。

## 抑制 (noqa) の書き方

- ファイル全体・ディレクトリ全体は `per-file-ignores` で対応する。
- 1 行だけ抑制したいときは **必ずルール ID を明示**する: `# noqa: E731` (裸の `# noqa` は禁止)。
- 抑制理由が自明でないときは同じ行末にコメントを追加する: `# noqa: PLR2004  # ステータスコード`。

## エディタ / CI

- VS Code は `charliermarsh.ruff` 拡張をユーザに案内する。
- CI では `ruff check --output-format=github` と `ruff format --check` を別ステップで走らせる (format 失敗と lint 失敗を分離するため)。
- pre-commit を使う場合は `python-pre-commit` スキルを参照。

## エージェントへの指示

- 既存リポジトリで black / isort / flake8 を見つけたら、ruff への置き換えを提案する。同居させない。
- `select` に既に書かれているカテゴリを増やすときは、`uv run ruff check` で新規違反の件数を確認してから提案する。大量に出る場合は段階移行 (`--add-noqa` で一括抑制 → 順次解消) を提案する。
- ruff format と他フォーマッタ (black, autopep8) の併用は提案しない。
- 「lint を入れて」という依頼を受けたら、まず ruff を導入する。別のリンタを希望する明示がない限り選択肢を広げない。

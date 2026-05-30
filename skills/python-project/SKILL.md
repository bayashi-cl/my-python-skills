---
name: python-project
description: 新しい Python アプリケーションプロジェクトをゼロから立ち上げるとき。uv ベースの pyproject.toml、src レイアウト、依存管理、.python-version、.gitignore を含む初期構造を作る。既存プロジェクトに src レイアウトを導入する場合や、依存追加・依存グループ整理のときにも使う。
---

# python-project-init

## 前提

- パッケージマネージャは **uv** に統一する。`pip` / `poetry` / `pipenv` を新規導入しない。
- レイアウトは **src レイアウト** で固定 (テストから import 経路を実体と一致させ、誤って未インストールのパッケージを参照しないため)。
- Python バージョンは `.python-version` で固定し、`pyproject.toml` の `requires-python` と一致させる。

## ディレクトリ構成

```text
.
├── .python-version
├── .gitignore
├── pyproject.toml
├── uv.lock
├── README.md
├── src/
│   └── <package_name>/
│       ├── __init__.py
│       └── __main__.py   # CLI を持つ場合のみ
└── tests/
    └── __init__.py
```

- `<package_name>` はスネークケース。ハイフン区切りは PyPI 名 (`project.name`) 側で使い、import 名と分離してよい。
- 1 リポジトリ = 1 パッケージを基本とする。複数パッケージにする場合は src 配下に並べる。

## pyproject.toml 雛形

```toml
[project]
name = "my-app"
version = "0.1.0"
description = ""
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_app"]

[dependency-groups]
dev = []
```

- ライブラリではなくアプリのみを配布しないなら `[build-system]` を省略してもよい。CLI を `entry_points` で公開する場合は必須。
- 依存は **`uv add` で追加する**ことを徹底し、手書きで `dependencies` を編集しない (lock との不整合を防ぐ)。

## .python-version / .gitignore

`.python-version` は `uv python pin 3.12` で生成する。
`.gitignore` は最低限以下を含める:

```gitignore
__pycache__/
*.py[cod]
.venv/
dist/
build/
*.egg-info/
.coverage
.pytest_cache/
.ruff_cache/
.mypy_cache/
.pyright/
```

`uv.lock` は **コミットする** (アプリ・ライブラリ問わず再現性のため)。

## よく使うコマンド

| 目的             | コマンド                                             |
| ---------------- | ---------------------------------------------------- |
| プロジェクト作成 | `uv init --package --lib` または手動雛形 + `uv sync` |
| 仮想環境同期     | `uv sync`                                            |
| 依存追加         | `uv add <pkg>`                                       |
| dev 依存追加     | `uv add --dev <pkg>`                                 |
| 依存削除         | `uv remove <pkg>`                                    |
| ロック更新       | `uv lock --upgrade`                                  |
| スクリプト実行   | `uv run <cmd>`                                       |

## 依存グループの方針

- ランタイム依存は `[project.dependencies]`。
- 開発ツール (ruff, pytest, 型チェッカ等) は `[dependency-groups].dev`。
- 機能フラグ的に切り替える依存は `[project.optional-dependencies]` (例: `aws`, `postgres`)。`uv sync --extra aws` で取り込む。

## エージェントへの指示

- 新規プロジェクトを作る依頼が来たら、まず上記の雛形通りディレクトリと `pyproject.toml` を作り、`uv sync` を実行して `uv.lock` を生成するまでを 1 ステップとみなす。
- 既存プロジェクトに後から src レイアウトを導入する場合は、`pyproject.toml` の `[tool.hatch.build.targets.wheel].packages` を必ず更新する (import エラーの原因)。
- 依存を `pyproject.toml` に直接書き足す変更を提案しない。常に `uv add` 経由を指示する。

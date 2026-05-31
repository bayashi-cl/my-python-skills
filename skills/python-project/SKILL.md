---
name: python-project
description: Pythonのアプリケーションプロジェクトを新規作成したり、プロジェクト構成や設定の更新を行うときに使用する。プロジェクトのレイアウトやPythonランタイム、外部パッケージへの依存関係の管理方法についても規定する。
---

# python-project

## 前提

- パッケージマネージャは **[uv](https://docs.astral.sh/uv/)** を使用する。
- **uv workspace** を利用することで、統一したPython環境からアプリケーション内の複数のコンポーネントを安全かつ効率的に管理する。
- プロジェクトに対する操作をcliで行える場合は、それを優先する。
- `uv.lock` は **コミットする** 。

## ディレクトリ構成

```text
.
├── pyproject.toml        # ワークスペースルート ([project] テーブルを持たない)
├── uv.lock               # コミットする
├── .python-version       # uv python pin で管理
└── packages/                  # ワークスペースメンバーを配置 (npmと衝突する場合は python/)
    ├── myproj-app-a/
    │   ├── pyproject.toml      # [project] テーブルと dependencies を持つ
    │   ├── src/
    │   │   └── myproj_app_a/   # import パッケージにもプレフィックスを付ける
    │   │       └── __init__.py
    │   └── tests_myproj_app_a/ # テストモジュールは tests_<pkg> で衝突を避ける
    └── myproj-app-b/
        ├── pyproject.toml
        ├── src/
        │   └── myproj_app_b/
        │       └── __init__.py
        └── tests_myproj_app_b/
```

## ワークスペースルート

以下の[観察](https://github.com/astral-sh/uv/issues/17855#issuecomment-3851940754)から、レポジトリ最上位には非プロジェクト(`[project]`テーブルが存在しない)の`pyproject.toml`を作成する。

> The distinction between a workspace-only root and a root project seems meaningful. I do notice a significant behavioral difference when adding a [project] table at the workspace root.
>
> With a root [project] present, uv sync only syncs the root project’s dependencies (including dev dependencies) and omits workspace members by default. Without a root [project], uv sync behaves equivalently to uv sync --all-packages, which in my case is the preferred behavior.

### `pyproject.toml`への設定記載

`uv`以外の開発ツールの設定は基本的にルートの`pyproject.toml`に記載する。特に、IDE/エディタが実行する開発ツール(ruff/mypy等)はルートの設定が共通設定として参照されることに注意する。

### ルートの依存パッケージ管理

開発依存関係のうち、ツールとして使用する以下のものをワークスペースルートの開発依存関係として定義する。

- `ruff`のような、仮想環境に依存しないで動作するもの
- `pip-licenses`のような、ワークスペースメンバー全体を一つの仮想環境で扱うことで動作するもの

### 最小限の`pyproject.toml`のサンプル

```toml
[tool.uv.workspace]
members = ["packages/*"]
```

## ワークスペースメンバー

アプリケーションを構成するコードはワークスペースメンバーとして管理する。
パッケージの分割単位として、1. 独自の依存関係を持つもの 2. 独自のデプロイ先を持つもの は単独のメンバーとする。

ワークスペースメンバーは`packages`ディレクトリ配下に配置する。
ただし、ポリグロットレポジトリなどでnpmのワークスペースの配置先と衝突する場合は、`python`ディレクトリを使用する。

### 命名規約

- パッケージ名(distribution名)には**プロジェクト固有のプレフィックス**を付け、PyPI等のパブリックパッケージとの名前衝突を(簡易的に)避ける。例: プロジェクト`myproj`なら`myproj-app-a`。
- importパッケージ(`src`配下のモジュール)にも同様にプレフィックスを付ける。例: `myproj_app_a`。
- テストモジュールのディレクトリは`tests_<pkg>`という名前にする。複数のメンバーに同名の`tests`パッケージが存在すると、mypyが重複モジュールとしてエラーを出すため。

### メンバーの依存パッケージ管理

そのメンバーが依存するパッケージを`dependencies`フィールドに記載する。

開発依存関係のうち、以下のものをメンバーの開発依存関係として定義する。

- `pytest`やそのプラグインのような、仮想環境への参照が必要なもの
- `types-requests`のようなスタブパッケージ

## Pythonランタイム管理

`uv python pin`コマンドを使用し、`.python-version`ファイルでバージョンを管理する。

## よく使うコマンド

| 目的             | コマンド                                       |
| ---------------- | ---------------------------------------------- |
| プロジェクト作成 | `uv init --package` または手動雛形 + `uv sync` |
| 仮想環境同期     | `uv sync`                                      |
| 依存追加         | `uv add --package <member> <pkg>`              |
| dev 依存追加     | `uv add --package <member> --dev <pkg>`        |
| 依存削除         | `uv remove --package <member> <pkg>`           |
| ロック更新       | `uv lock --upgrade`                            |
| スクリプト実行   | `uv run <cmd>`                                 |

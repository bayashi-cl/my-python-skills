---
name: python-typecheck
description: Python プロジェクトに静的型チェックを導入する、または既存の設定を調整するとき。pyright / mypy / ty の選定、pyproject 設定、strict 化、型エラー抑制の書き方、`from __future__ import annotations` の方針を含む。型ヒント追加の方針もここで決める。
---

# python-typecheck

## チェッカ選定

- **第一選択は pyright** (高速、Pylance と同じエンジン、エディタ体験が良い)。
- mypy はプラグインエコシステム (SQLAlchemy, Django) が必要なときのみ採用する。
- ty (Astral 製) は試験運用枠。本番採用前に CI で pyright と並走させて判断する。
- **2 つを同居させない**。CI が遅くなる割に差分検出は限定的。

## pyright の導入

1. `uv add --dev pyright`

1. `pyproject.toml`:

   ```toml
   [tool.pyright]
   include = ["src", "tests"]
   pythonVersion = "3.12"
   typeCheckingMode = "strict"
   reportMissingTypeStubs = false
   reportUnknownMemberType = false   # サードパーティ型不足を緩和
   reportUnknownArgumentType = false
   venvPath = "."
   venv = ".venv"
   ```

1. `uv run pyright` で実行。CI は `pyright --outputjson` を使うと差分集計しやすい。

## mypy を使う場合

```toml
[tool.mypy]
python_version = "3.12"
strict = true
files = ["src", "tests"]
plugins = []   # 例: ["pydantic.mypy"]

[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

## 型ヒント方針

- `from __future__ import annotations` を **全ファイル冒頭に書く** (Python 3.12 でも `X | Y` のランタイム評価を避けたい場面が残るため)。例外は Pydantic / FastAPI / SQLAlchemy など **annotation をランタイムで読むフレームワーク** のモジュール — これらでは future import を入れない。
- 戻り値も含めて関数シグネチャに型を付ける。`-> None` を省略しない。
- ジェネリックは PEP 695 構文 (`def f[T](x: T) -> T`) を優先。古い `TypeVar` は既存コードに合わせるときのみ。
- `Any` は使わない。未知の外部入力は `object` から narrowing するか、Pydantic でパースする。

## 型エラーの抑制

- `# type: ignore` 単独は禁止。**必ず理由付きルールを書く**:
  - pyright: `# pyright: ignore[reportGeneralTypeIssues]  # 理由`
  - mypy: `# type: ignore[arg-type]  # 理由`
- ライブラリ側の型欠落が原因なら、抑制ではなく `py.typed` を持つバージョンへ上げる or `types-*` スタブを追加することを先に検討する。
- モジュール全体を諦める場合は pyproject の overrides に書き、コード側を汚さない。

## 段階導入

既存コードに後から strict を入れるとき:

1. `typeCheckingMode = "basic"` で開始し、`src/<core>/**` だけ `strict = true` を `executionEnvironments` で指定する。
1. CI を `--warnings` で回し、新規エラーゼロを差分基準にする (絶対数は許容)。
1. 段階的に対象ディレクトリを広げる。

## エージェントへの指示

- 既存リポジトリに型チェッカが無い状態で関数を追加するとき、戻り値型と引数型を必ず書く。`Any` を持ち込まない。
- pyright のエラーを潰す依頼で `# type: ignore` を撒く解決を提案しない。まず **型を正しく付ける / 実装を直す** を試み、抑制は最後の手段。
- mypy plugin (例: `pydantic.mypy`) を有効化する変更を提案するときは、CI 時間への影響を一言添える。

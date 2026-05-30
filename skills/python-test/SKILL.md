---
name: python-test
description: Python プロジェクトでテストを書く・整備するとき。テストランナー選定、ディレクトリ構成、fixture、parametrize、marker、カバレッジ、モック (本物優先 / monkeypatch / respx) の方針を含む。新規テスト追加、テスト雛形作成、CI でのテスト実行設定にもこのスキルを使う。
---

# python-test

## ツール選定

- **第一選択は pytest** (fixture / parametrize / プラグインエコシステムが他を圧倒)。
- `unittest.TestCase` のクラスを新規に書かない。既存資産は移行を急がず、新規テストのみ pytest 流で書く (混在を許容)。
- カバレッジは `pytest-cov` (coverage.py を pytest から駆動)。
- モックは `pytest-mock` を基本に、HTTP は `respx` (httpx) / `responses` (requests)、時刻は `freezegun`、ファイルは pytest の `tmp_path` fixture を使う。

## 方針

- テストは **src パッケージと同名のサブツリーで mirroring**: `src/my_app/foo/bar.py` → `tests/foo/test_bar.py`。
- 1 テストファイル = 1 被テストモジュール。横断的テストは `tests/integration/` 配下にまとめる。
- ネットワーク・DB・ファイル I/O を伴うテストには `@pytest.mark.integration` を付け、デフォルトの単体実行から外す。
- **本物を使う**ことを優先する。ファイル I/O は `tmp_path`、時刻は `freezegun`、HTTP は `respx` / `httpx.MockTransport`、DB は testcontainers。SQLite で Postgres を模さない (本番との差で事故る)。

## ディレクトリ構成

```
tests/
├── __init__.py
├── conftest.py              # プロジェクト全体の fixture
├── unit/
│   └── test_<module>.py
└── integration/
    ├── conftest.py          # DB セットアップなど
    └── test_<flow>.py
```

`tests/__init__.py` を置くことで、テストファイル名が衝突しても import エラーにならない。

## 導入手順

1. `uv add --dev pytest pytest-cov pytest-mock`
2. `pyproject.toml`:

```toml
[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
addopts = [
    "-ra",
    "--strict-markers",
    "--strict-config",
    "-m", "not integration",   # デフォルトは単体のみ
]
markers = [
    "integration: 外部依存を伴うテスト",
    "slow: 実行に時間がかかるテスト",
]
filterwarnings = [
    "error",                     # 警告はテスト失敗に昇格
    "ignore::DeprecationWarning:<外部ライブラリ名>",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
show_missing = true
skip_covered = true
exclude_also = [
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "\\.\\.\\.",
]
```

3. 実行:
   - 単体: `uv run pytest`
   - 全部: `uv run pytest -m ""`
   - カバレッジ: `uv run pytest --cov --cov-report=term-missing`

## fixture

- 共通の fixture は **一番近い conftest.py** に置く。`tests/conftest.py` に何でも詰めない。
- `scope` はデフォルト `function`。明示的に必要な場合のみ `module` / `session` に上げる (テスト独立性が崩れるリスク)。
- `autouse=True` は **環境変数クリア・乱数シード固定** のように「全テストに必須かつ無害」な用途に限る。
- factory パターン (関数を返す fixture) で、複数バリエーションのオブジェクト生成を 1 fixture にまとめる。

```python
@pytest.fixture
def make_user():
    def _make(*, name="alice", admin=False):
        return User(name=name, admin=admin)
    return _make
```

## parametrize

- 同じロジックを複数入力でテストするときは **必ず parametrize**。`for` ループでアサートしない (失敗箇所が分からなくなる)。
- `id=` を付けて失敗時のテスト名を読みやすくする。

```python
@pytest.mark.parametrize(
    ("input_", "expected"),
    [
        ("1", 1),
        ("-1", -1),
        ("0", 0),
    ],
    ids=["positive", "negative", "zero"],
)
def test_parse_int(input_: str, expected: int) -> None:
    assert parse_int(input_) == expected
```

## モック / 偽物

- まず「本物が使えないか」を検討する (`tmp_path`, `freezegun`, `respx`, testcontainers)。
- 単体テストでクラスメソッド・関数を差し替えるときは `pytest-mock` の `mocker.patch` を使い、`unittest.mock.patch` のデコレータは新規には書かない (引数注入順がバグの温床)。
- パッチ対象は **使われている側の import パス** を指す: `mocker.patch("my_app.service.send_email")` (定義側 `my_app.notify.send_email` ではない)。

## アサーション

- `assert` を直接使う (pytest が rewrite して差分を表示する)。`unittest` の `self.assertEqual` 系を新規に書かない。
- 例外は `pytest.raises(SomeError, match="...")` で型とメッセージを両方検証する。
- 浮動小数は `==` ではなく `pytest.approx`。

## エージェントへの指示

- テストを追加する依頼では、まず該当モジュールのテストファイル位置 (`tests/<mirror>/test_<name>.py`) を確認し、無ければ作る。
- 既存テストに `unittest.TestCase` クラスがあっても、**新規テストは関数ベースで書く**。混在を許容する。
- `mocker.patch` の対象パスを書くときは、**実コードでその名前が使われている場所** を grep で確認してから指定する (誤指定はテストが通るのに本物が呼ばれるバグの典型)。
- カバレッジを上げるためだけのテスト (実装をなぞるだけのアサーション) は書かない。挙動を確認しないテストは負債。
- 「テストランナーを別のに変えたい」という明示がない限り pytest を提案する。`unittest` ベースの新規セットアップを作らない。

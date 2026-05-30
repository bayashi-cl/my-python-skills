---
name: python-settings
description: Python アプリケーションの設定 (環境変数 / .env / 階層化された config) を pydantic-settings で実装するとき。BaseSettings の構成、SettingsConfigDict、ネスト、起動時バリデーション、テストでの上書き方法を含む。新規アプリの config モジュール作成や、設定項目の追加・整理にも使う。
---

# python-settings

## 方針

- 設定は **`pydantic_settings.BaseSettings` 1 つに集約**する。`os.getenv` を散らさない。
- 設定インスタンスは **モジュール import 時に 1 回だけ作る** (起動時に失敗を検出するため)。lazy にしない。
- 設定値の取得経路を一本化するため、**アプリコードは `from my_app.config import settings` だけ**。`os.environ` を直接読まない。
- 機密値はコードや lock に含めない。`.env` は `.gitignore` に入れ、`.env.example` をコミットする。

## 導入

1. `uv add pydantic-settings`
2. `src/<package>/config.py` を作る:

```python
from __future__ import annotations

from functools import lru_cache
from pathlib import Path

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseSettings(BaseSettings):
    url: str
    pool_size: int = 5


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="MYAPP_",
        env_nested_delimiter="__",
        extra="forbid",
        frozen=True,
    )

    debug: bool = False
    log_level: str = "INFO"
    api_key: SecretStr
    database: DatabaseSettings = Field(default_factory=DatabaseSettings)
    data_dir: Path = Path("./data")


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()  # type: ignore[call-arg]   # env から埋まる


settings = get_settings()
```

3. `.env.example`:

```
MYAPP_API_KEY=sk-xxx
MYAPP_DATABASE__URL=postgresql+psycopg://user:pass@localhost/app
MYAPP_DATABASE__POOL_SIZE=10
MYAPP_DEBUG=false
```

## 設計の指針

- **`env_prefix`** を必ず付ける (他アプリの環境変数と衝突を防ぐ)。
- **`env_nested_delimiter="__"`** を採用し、`DATABASE__URL` のようにネストを表現する。区切りに `.` や `_` を使わない (前者は環境変数で扱いにくく、後者はフィールド名と区別できない)。
- **`extra="forbid"`** にして未知のキーは起動時に落とす (typo 検出)。
- **`frozen=True`** で実行中の書き換えを禁止する。差し替えはテスト用フックでのみ。
- 機密文字列は `SecretStr` を使う。ログに出してもマスクされる。

## ファイル / 複数ソース

- 機密以外の構造化設定 (デフォルト値、定義済みリスト) は YAML/TOML に外出ししたくなるが、**新規ではまず .env + コード内デフォルト** で十分。ファイルが必要になったら `model_config` の `toml_file` / `yaml_file` を後付けする。
- 環境別 (`dev` / `prod`) は `.env.dev` `.env.prod` を切り替えるより、**環境変数のみで差し替える** + デフォルトは prod 寄りにする。

## テストでの上書き

- monkeypatch で環境変数を差し替え、`Settings()` を再生成する fixture を用意する:

```python
@pytest.fixture
def settings(monkeypatch: pytest.MonkeyPatch) -> Settings:
    monkeypatch.setenv("MYAPP_API_KEY", "test")
    monkeypatch.setenv("MYAPP_DATABASE__URL", "sqlite:///:memory:")
    get_settings.cache_clear()
    return get_settings()
```

- グローバル `settings` を import して書き換えるのは禁止 (`frozen=True` で弾かれる)。常に依存注入 or 上記の cache_clear 経由。

## エージェントへの指示

- 新規設定項目を追加する依頼では、`Settings` クラス、`.env.example`、関連する利用箇所の 3 つを同じコミットで更新する。
- アプリコードに `os.getenv("...")` を書く提案をしない。必ず `settings.<field>` 経由にする。
- 設定値を「テスト時だけ違う値にしたい」要求には、上記 `settings` fixture を使う方法を提示し、グローバル変数の直接書き換えを提案しない。
- 機密値 (API key, password, token) を追加するときは型を `SecretStr` にする。ログ出力時は `.get_secret_value()` を呼ぶ箇所を最小化する。

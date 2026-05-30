---
name: python-datamodel
description: Python アプリケーションのドメインモデル / DTO / API スキーマを Pydantic v2 で設計するとき。BaseModel、field_validator / model_validator、computed_field、model_config (frozen, populate_by_name)、シリアライズ、discriminated union、TypeAdapter の使い方を含む。新規モデル追加、既存モデルのバリデーション強化に使う。
---

# python-model

## 方針

- 外部入出力 (API リクエスト/レスポンス、設定、永続化前後の DTO) は **Pydantic v2 `BaseModel`** で表現する。
- 純粋な内部値オブジェクト (ロジック専用、I/O しない) は `dataclass(frozen=True, slots=True)` で十分。Pydantic を入れない。
- Pydantic v1 API (`@validator`, `parse_obj`, `dict()`, `Config` クラス) を新規に書かない。既存コードは v2 へ寄せる。

## 基本形

```python
from __future__ import annotations

from datetime import datetime
from typing import Annotated, Literal

from pydantic import BaseModel, ConfigDict, Field, computed_field, field_validator


class User(BaseModel):
    model_config = ConfigDict(
        frozen=True,
        extra="forbid",
        str_strip_whitespace=True,
        populate_by_name=True,
    )

    id: int
    name: Annotated[str, Field(min_length=1, max_length=100)]
    email: str
    created_at: datetime
    role: Literal["admin", "member"] = "member"

    @field_validator("email")
    @classmethod
    def _normalize_email(cls, v: str) -> str:
        return v.lower()

    @computed_field
    @property
    def is_admin(self) -> bool:
        return self.role == "admin"
```

## model_config の基準

| 設定 | 既定 | 理由 |
| --- | --- | --- |
| `frozen=True` | 採用 | 不変にして共有しても安全に。書き換えが必要なら `model_copy(update=...)` |
| `extra="forbid"` | 採用 | 未知のキーで silent に取りこぼすのを防ぐ |
| `str_strip_whitespace=True` | 採用 | API 入力の前後空白を吸収 |
| `populate_by_name=True` | 採用 | エイリアス + 属性名どちらでも組み立て可能に |
| `validate_assignment=True` | 必要時のみ | frozen と排他。frozen を優先 |

## バリデータ

- `@field_validator`: 単一フィールドの正規化・追加制約。`mode="before"` でパース前、`mode="after"` (デフォルト) でパース後。
- `@model_validator(mode="after")`: 複数フィールドの相互整合性。型は self を返す。
- 入力を変換するなら **`Annotated[T, AfterValidator(fn)]`** を優先 (validator メソッドより再利用しやすい)。

```python
from typing import Annotated

from pydantic import AfterValidator

NonEmptyStr = Annotated[str, AfterValidator(lambda v: v if v.strip() else (_ for _ in ()).throw(ValueError("empty")))]
```

## シリアライズ

- 出力は **`model_dump()` / `model_dump_json()`** のみ。`.dict()` / `.json()` (v1) を新規に書かない。
- API 出力ではフィールド名と JSON キーを分けたいことが多い: `Field(alias="userId")` + `model_config.populate_by_name=True`。
- 機密フィールドの除外は `model_dump(exclude={"password"})` ではなく、**専用の OutputModel を別途定義する** (除外漏れの予防)。

## Discriminated Union

複数の型を持ち得るペイロードは **必ず discriminator を付ける** (パース速度と型推論の両面で有利):

```python
from typing import Annotated, Literal, Union

from pydantic import BaseModel, Field


class TextMessage(BaseModel):
    type: Literal["text"]
    body: str


class ImageMessage(BaseModel):
    type: Literal["image"]
    url: str


Message = Annotated[Union[TextMessage, ImageMessage], Field(discriminator="type")]
```

## TypeAdapter

- BaseModel に包むほどでもないリスト/辞書のパースは `TypeAdapter` を使う:

```python
from pydantic import TypeAdapter

UserList = TypeAdapter(list[User])
users = UserList.validate_python(raw)
```

- ホットパスで使うときは **モジュールレベルで 1 度だけ生成**する (毎回作るとコスト)。

## 永続化との分離

- DB エンティティ (SQLAlchemy 等) と API モデルは **同じクラスにしない**。Pydantic モデルは DTO、SQLAlchemy モデルは行表現に分ける。
- 変換は `UserDTO.model_validate(orm_user, from_attributes=True)` を使う (`model_config` に `from_attributes=True` を入れた DTO 側で対応)。

## エージェントへの指示

- 新規モデルを書くときは、上記 `model_config` の 4 つ (`frozen`, `extra="forbid"`, `str_strip_whitespace`, `populate_by_name`) を既定で入れる。外す場合は理由をコメントで残す。
- 既存コードに v1 API (`@validator`, `Config` クラス, `.dict()`) を見つけたら、その箇所を触るついでに v2 へ書き換える提案をする。一括書き換えは依頼があるまでしない。
- API レスポンス用に既存ドメインモデルをそのまま返す提案をしない。**専用の OutputModel** を作って明示的にフィールドを露出する。
- `Optional[X]` / `Union[X, None]` ではなく `X | None` を使う (project の型方針と一致)。

# Model Config
https://pydantic-docs.helpmanual.io/usage/model_config/

pydantic の動作は、モデルや pydantic dataclass の `Config` クラスを介して制御することができます。

以下のオプションがあります。

`title`
: 生成されるJSONスキーマのタイトル

`anystr_strip_whitespace`
: `str` と `byte` 型の先頭と末尾のホワイトスペースを削除するかどうか (default: `False`)

`anystr_lower`
: `str` と `byte` 型で全ての文字を小文字にするかどうか (default: `False`)

`min_anystr_length`
: `str` と `byte` 型の最小長 (default: `0`)

`max_anystr_length`
: `str` と `byte` 型の最大長 (default: `2 ** 16`)

`validate_all`
: フィールドのデフォルトを検証するかどうか (default: `False`)

`extra`
: モデルの初期化時に追加属性を無視するか、許可するか、禁止するか。 `ignore`, `allow`, `forbid` の文字列値、または `Extra` enum (default: `Extra.ignore`)の値を受け入れます。 `forbid` は余分な属性が含まれていると検証に失敗し、 `ignore` は余分な属性を黙って無視し、 `allow` は属性をモデルに割り当てることになります。

`allow_mutation`
: モデルが擬似的に不変であるかどうか、つまり `__setattr__` が許可されているかどうか `(default: True)`

`frozen`
: Warning: このパラメータはベータ版です。
: `frozen=True` を設定すると `allow_mutation=False` が行うすべてのことを行い、さらにモデルに `__hash__()` メソッドを生成します。これにより、すべての属性がハッシュ化可能であれば、モデルのインスタンスは潜在的にハッシュ化可能になります (default: False)

`use_enum_values`
: 生の enum ではなく、 enum の `value` プロパティでモデルを生成するかどうかを指定します。これは `model.dict()` を後でシリアライズしたい場合に便利です (default: `False`)

`fields`
: 各フィールドのスキーマ情報を含む `dict` 。これは `Field` クラスを使用するのと同じです (default: `None`)

`validate_assignment`
: 属性への割り当てについて検証を行うかどうか (default: `False`)

`allow_population_by_field_name`
: エイリアスのフィールドにモデル属性で指定された名前と、エイリアスを入力するかどうか (default: `False`)
: Note: この設定の名前は **v1.0** で `allow_population_by_alias` から `allow_population_by_field_name` に変更されました。

`error_msg_templates`
: デフォルトのエラーメッセージテンプレートを上書きするための辞書です。オーバーライドしたいエラーメッセージにマッチするキーを持つ辞書を渡します (default: `{}`)

`arbitrary_types_allowed`
: フィールドに任意のユーザータイプを許可するかどうか（値がタイプのインスタンスであるかどうかをチェックするだけで検証されます）。Falseの場合、モデルの宣言時にRuntimeErrorが発生します（デフォルト：False）。例は、Field Typesを参照してください。

`orm_mode`
: ORM モードの使用を許可するかどうか。

`getter_dict`
: `orm_mode` で使用するために、ORMクラスを分解して検証する際に使用するカスタムクラス（GetterDictを継承している必要があります）。

`alias_generator`
: フィールド名を受け取り、それに対するエイリアスを返すコーラブル

`keep_untouched`
: モデルのデフォルト値のための型（記述子など）のタプルで、モデル作成時に変更してはならず、モデルスキーマにも含まれません。注：これは、このタイプのアノテーションではなく、このタイプのデフォルトを持つモデル上の属性が放置されることを意味します。

`schema_extra`
: 生成されたJSONスキーマを拡張/更新するために使用されるディクショナリー、または後処理のためのコーラブルです；スキーマのカスタマイズを参照してください。

`json_loads`
: JSONをデコードするためのカスタム関数。「カスタムJSON（デ）シリアライゼーション」を参照してください。

`json_dumps`
: JSONをエンコードするためのカスタム関数。「カスタムJSON（デ）シリアライズ」を参照してください。

`json_encoders`
: JSON への型のエンコード方法をカスタマイズするために使用される dict; See JSON Serialisation

```python
from pydantic import BaseModel, ValidationError


class Model(BaseModel):
    v: str

    class Config:
        max_anystr_length = 10
        error_msg_templates = {
            'value_error.any_str.max_length': 'max_length:{limit_value}',
        }


try:
    Model(v='x' * 20)
except ValidationError as e:
    print(e)
    """
    1 validation error for Model
    v
      max_length:10 (type=value_error.any_str.max_length; limit_value=10)
    """
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

また、モデルクラスの kwargs として設定オプションを指定することもできます。

```python
from pydantic import BaseModel, ValidationError, Extra


class Model(BaseModel, extra=Extra.forbid):
    a: str


try:
    Model(a='spam', b='oh no')
except ValidationError as e:
    print(e)
    """
    1 validation error for Model
    b
      extra fields not permitted (type=value_error.extra)
    """
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

同様に `@dataclass` デコレータを使用した場合。

```python
from datetime import datetime

from pydantic import ValidationError
from pydantic.dataclasses import dataclass


class MyConfig:
    max_anystr_length = 10
    validate_assignment = True
    error_msg_templates = {
        'value_error.any_str.max_length': 'max_length:{limit_value}',
    }


@dataclass(config=MyConfig)
class User:
    id: int
    name: str = 'John Doe'
    signup_ts: datetime = None


user = User(id='42', signup_ts='2032-06-21T12:00')
try:
    user.name = 'x' * 20
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    name
      max_length:10 (type=value_error.any_str.max_length; limit_value=10)
    """
```

(このスクリプトは完成していますので、そのまま実行してください)

`underscore_attrs_are_private`
: クラス以外のアンダースコアの var アトリビュートをプライベートとして扱うか、そのままにしておくか; See モデルのプライベート属性.

`copy_on_model_validation`
: フィールドとして使われている継承されたモデルを、そのままにしておくのではなく、検証時に再構築（コピー）するかどうか(default: `True`)

## グローバルでの動作変更について

pydantic の挙動をグローバルに変更したい場合は、独自の `BaseModel` をカスタム `Config` で作成することができます（`Config` は継承されます）。

```python
from pydantic import BaseModel as PydanticBaseModel


class BaseModel(PydanticBaseModel):
    class Config:
        arbitrary_types_allowed = True


class MyClass:
    """A random class"""


class Model(BaseModel):
    x: MyClass
```

(このスクリプトは完成しているので、そのまま実行できます)

## エイリアスジェネレータ

データソースのフィールド名がコードスタイルに合わない場合（キャメルケースのフィールドなど）、 `alias_generator` を使って自動的にエイリアスを生成することができます。

```python
from pydantic import BaseModel


def to_camel(string: str) -> str:
    return ''.join(word.capitalize() for word in string.split('_'))


class Voice(BaseModel):
    name: str
    language_code: str

    class Config:
        alias_generator = to_camel


voice = Voice(Name='Filiz', LanguageCode='tr-TR')
print(voice.language_code)
#> tr-TR
print(voice.dict(by_alias=True))
#> {'Name': 'Filiz', 'LanguageCode': 'tr-TR'}
```

(このスクリプトは完成しているので、そのまま実行してください)

ここでのキャメルケースとは「アッパーキャメルケース」、つまりパスカルケースのことです。代わりに小文字のキャメルケース（例：camelCase）を使用したい場合は、上記の `to_camel` 関数を変更するのが簡単です。

## エイリアスの優先順位

> Warning:
> **v1.4** では、以前のバージョンで発生したバグや予期せぬ動作を解決するために、エイリアスの優先順位のロジックが変更されました。詳細は [#1178]() と下記の優先順位を参照してください。

フィールドのエイリアスが複数の場所で定義されている場合、選択される値は以下のように決定されます（優先順位は降順）。

1. `Field(..., alias=<alias>)` で設定され、モデル上で直接設定される。
2. モデル上で直接 `Config.fields` で定義されているもの
3. 親モデルの `Field(..., alias=<alias>)` を使って設定する
4. 親モデルの `Config.fields` で定義される
5. モデルや親に関わらず `alias_generator` によって生成される。

> Note:
> これは、子モデルに定義された `alias_generator` が、親モデルのフィールドに定義された `alias` よりも優先されないことを意味します。

例えば、以下のようになります。

```python
from pydantic import BaseModel, Field


class Voice(BaseModel):
    name: str = Field(None, alias='ActorName')
    language_code: str = None
    mood: str = None


class Character(Voice):
    act: int = 1

    class Config:
        fields = {'language_code': 'lang'}

        @classmethod
        def alias_generator(cls, string: str) -> str:
            # this is the same as `alias_generator = to_camel` above
            return ''.join(word.capitalize() for word in string.split('_'))


print(Character.schema(by_alias=True))
"""
{
    'title': 'Character',
    'type': 'object',
    'properties': {
        'ActorName': {'title': 'Actorname', 'type': 'string'},
        'lang': {'title': 'Lang', 'type': 'string'},
        'Mood': {'title': 'Mood', 'type': 'string'},
        'Act': {'title': 'Act', 'default': 1, 'type': 'integer'},
    },
}
"""
```

(このスクリプトは完成しているので、"そのまま "実行することができます)


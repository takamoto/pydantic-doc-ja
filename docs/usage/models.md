# モデル

pydantic でオブジェクトを定義する主な手段はモデルです（モデルは単に `BaseModel` を継承したクラスです）。

モデルは厳密に型付けされた言語の型に似ていると考えることもできますし、 API の単一のエンドポイントの要件と考えることもできます。

信頼できないデータをモデルに渡すことができ、解析と検証の後、pydanticは結果として得られるモデルインスタンスのフィールドがモデル上で定義されたフィールドタイプに適合することを保証します。

> Note
> 
> pydantic は主に構文解析ライブラリであり、検証ライブラリではありません。検証は、提供された型と制約に準拠したモデルを構築するという目的のための手段です。
> 
> 言い換えれば pydantic は入力データではなく、出力モデルの型と制約を保証します。
> 
> これは難解な区別のように聞こえるかもしれませんが、そうではありません。これが何を意味するのか、またそれがあなたの使い方にどのような影響を与えるのかわからない場合は、以下のデータ変換に関するセクションをお読みください。


## 基本的なモデルの使用法

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name = 'Jane Doe'
```

`User` は 2つのフィールドを持つモデルです。 `id` は整数で必須、 `name` は文字列で必須ではありません（デフォルト値があります）。 `name` の型はデフォルト値から推測されるため、型のアノテーションは必要ありません（ただし、型のアノテーションを持たないフィールドがある場合、フィールドの順序に関する警告に注意してください）。

```python
user = User(id='123')
```

`user` は `User` のインスタンスです。オブジェクトの初期化によりすべての解析と検証が行われ、 `ValidationError` が発生しなければ、結果として得られるモデルインスタンスが有効であることがわかります。

```python
assert user.id == 123
```

モデルのフィールドは `user` オブジェクトの通常の属性としてアクセスできます。文字列 '123' は、フィールドのタイプに応じて `int` にキャストされます。

```python
assert user.name == 'Jane Doe'
```

`user` の初期化時に `name` が設定されていなかったため、デフォルト値が設定されています。

```python
assert user.__fields_set__ == {'id'}'
```

userが初期化されたときに提供されたフィールドです。

```python
assert user.dict() == dict(user) == {'id': 123, 'name': 'Jane Doe'}。
```

`.dict()` または `dict(user)` のいずれかがフィールドの `dict` を提供しますが、 `.dict()` は他にも多数の引数を取ることができます。

```python
user.id = 321
assert user.id == 321
```

このモデルは mutable なので、フィールドの値を変更することができます。

### モデルのプロパティ

上の例は、モデルができることの氷山の一角に過ぎません。モデルは以下のようなメソッドや属性を持っています。

`dict()`
: モデルのフィールドと値の辞書を返します。[モデルのエクスポート]()を参照

`json()`
: `dict()` を JSON 文字列で表現したものを返します。[モデルのエクスポート]()を参照。

`copy()`
: モデルのコピー（デフォルトでは、シャローコピー）を返します。

`parse_obj()`
: 任意のオブジェクトをモデルにロードするユーティリティで、オブジェクトが辞書でない場合はエラー処理を行います。

`parse_raw()`
: 様々なフォーマットの文字列を読み込むユーティリティーです。

`parse_file()`
: `parse_raw()` と同様ですが、ファイルパスを対象としています。[ヘルパー関数]() を参照。

`from_orm()`
: 任意のクラスからモデルにデータをロードする。[ORMモード]() を参照。

`schema()`
: モデルをJSON Schemaとして表す辞書を返します。[schema]() を参照。

`schema_json()`
: `schema()` の JSON 文字列表現を返します。[schema]() を参照。

`construct()`
: バリデーションを行わずにモデルを作成するためのクラスメソッド。[バリデーションを行わないモデルの作成]() を参照。

`__fields_set__`
: モデルのインスタンスが初期化されたときに設定されたフィールドの名前のセット。

`__fields__`
: モデルのフィールドの辞書

`__config__`
: モデルの設定クラス。[モデルの設定]() を参照。

## 再帰的モデル

より複雑な階層的データ構造は、モデル自体をアノテーションの型として使用して定義することができます。

```python
from typing import List
from pydantic import BaseModel


class Foo(BaseModel):
    count: int
    size: float = None


class Bar(BaseModel):
    apple = 'x'
    banana = 'y'


class Spam(BaseModel):
    foo: Foo
    bars: List[Bar]


m = Spam(foo={'count': 4}, bars=[{'apple': 'x1'}, {'apple': 'x2'}])
print(m)
#> foo=Foo(count=4, size=None) bars=[Bar(apple='x1', banana='y'),
#> Bar(apple='x2', banana='y')] 。
print(m.dict())
"""
{
    'foo': {'count': 4, 'size': None},
    'bar': [
        {'apple': 'x1', 'banana': 'y'},
        {'apple': 'x2', 'banana': 'y'},
    ],
}
"""
```

(このスクリプトは完成しているので、"そのまま "実行することができます)

自己参照モデルについては、[postponed annotations]() を参照してください。

## ORM モード（別名：任意のクラスインスタンス）

Pydantic モデルは任意のクラスインスタンスから作成することができ、ORM オブジェクトにマッピングするモデルをサポートします。

これを行うには

1. [Config]() のプロパティ `orm_mode` を `True` に設定する必要があります。
2. モデルのインスタンスを作成するために、特別なコンストラクタ `from_orm` を使用しなければなりません。

この例では、SQLAlchemyを使用していますが、同じ方法でどの ORM にも対応できます。

```python
from typing import List
from sqlalchemy import Column, Integer, String
from sqlalchemy.dialects.postgresql import ARRAY
from sqlalchemy.ext.declarative import declarative_base
from pydantic import BaseModel, constr

Base = declarative_base()


class CompanyOrm(Base):
    __tablename__ = 'companies'
    id = Column(Integer, primary_key=True, nullable=False)
    public_key = Column(String(20), index=True, nullable=False, unique=True)
    name = Column(String(63), unique=True)
    domains = Column(ARRAY(String(255)))


class CompanyModel(BaseModel):
    id: int
    public_key: constr(max_length=20)
    name: constr(max_length=63)
    domains: List[constr(max_length=255)]

    class Config:
        orm_mode = True


co_orm = CompanyOrm(
    id=123,
    public_key='foobar',
    name='Testing',
    domains=['example.com', 'foobar.com'],
)
print(co_orm)
#> <models_orm_mode.CompanyOrm object at 0x7f779fd22ac0>
co_model = CompanyModel.from_orm(co_orm)
print(co_model)
#> id=123 public_key='foobar' name='Testing' domains=['example.com',
#> 'foobar.com']
```

(このスクリプトは完成しているので、そのまま実行してください)

### 予約名

予約された SQLAlchemy のフィールドにちなんだ名前を Column につけたい場合があります。そのような場合には Field のエイリアスが便利です。

```python
import typing

from pydantic import BaseModel, Field
import sqlalchemy as sa
from sqlalchemy.ext.declarative import declarative_base


class MyModel(BaseModel):
    metadata: typing.Dict[str, str] = Field(alias='metadata_')

    class Config:
        orm_mode = True


BaseModel = declarative_base()


class SQLModel(BaseModel):
    __tablename__ = 'my_table'
    id = sa.Column('id', sa.Integer, primary_key=True)
    # 'metadata' is reserved by SQLAlchemy, hence the '_'
    metadata_ = sa.Column('metadata', sa.JSON)


sql_model = SQLModel(metadata_={'key': 'val'}, id=1)

pydantic_model = MyModel.from_orm(sql_model)

print(pydantic_model.dict())
#> {'metadata': {'key': 'val'}}
print(pydantic_model.dict(by_alias=True))
#> {'metadata_': {'key': 'val'}}
```

(このスクリプトは完成していますので、そのまま実行してください)

> Note: 
> 上の例がうまくいくのは、フィールドの生成においてエイリアスがフィールド名よりも優先されるからです。 `SQLModel` の `metadata` 属性にアクセスすると `ValidationError` が発生します。

### 再帰的な ORM モデル

ORM インスタンスはトップレベルだけでなく、再帰的に `from_orm` で解析されます。

ここでは、原理を説明するために vanilla クラスを使用しているが、任意の ORM クラスを代わりに使用することができる。

```python
from typing import List
from pydantic import BaseModel


class PetCls:
    def __init__(self, *, name: str, species: str):
        self.name = name
        self.species = species


class PersonCls:
    def __init__(self, *, name: str, age: float = None, pets: List[PetCls]):
        self.name = name
        self.age = age
        self.pets = pets


class Pet(BaseModel):
    name: str
    species: str

    class Config:
        orm_mode = True


class Person(BaseModel):
    name: str
    age: float = None
    pets: List[Pet]

    class Config:
        orm_mode = True


bones = PetCls(name='Bones', species='dog')
orion = PetCls(name='Orion', species='cat')
anna = PersonCls(name='Anna', age=20, pets=[bones, orion])
anna_model = Person.from_orm(anna)
print(anna_model)
#> name='Anna' age=20.0 pets=[Pet(name='Bones', species='dog'),
#> Pet(name='Orion', species='cat')]
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

任意のクラスを pydantic で処理するには、 `GetterDict` クラス（[utils.py]() 参照）を使います。 `GetterDict` の独自のサブクラスを `Config.getter_dict` の値として設定することで、この動作をカスタマイズすることができます（[config]() 参照）。

`pre=True` の [root_validators]() を使ってクラスの検証をカスタマイズすることもできます。この場合、バリデータ関数には `GetterDict` のインスタンスが渡され、 それをコピーしたり変更したりすることができます。

## エラー処理

pydantic は、検証中のデータにエラーがあると `ValidationError` を発生させます。

> Note:
> バリデーションコードは `ValidationError` 自体を発生させてはならず、むしろ `ValueError`, `TypeError`, `AssertionError` （または `ValueError` や `TypeError` のサブクラス）を発生させ、それらをキャッチして `ValidationError` を生成するために使用します。

見つかったエラーの数にかかわらず 1 つの例外が発生し、 `ValidationError` にはすべてのエラーとその発生状況に関する情報が含まれます。

これらのエラーには以下の方法でアクセスすることができます。

`e.errors()`
: メソッドは、入力データで見つかったエラーのリストを返します。

`e.json()`
: メソッドは、エラーのJSON表現を返します。

`str(e)`
: メソッドは、エラーの人間が読める表現を返します。


各エラーオブジェクトは以下を含みます。

`loc`
: リストとしてのエラーの場所。リストの最初の項目は、エラーが発生したフィールドで、そのフィールドがサブモデルの場合には、エラーのネストした場所を示すために後続の項目が存在します。

`type`
: エラーの種類を示すコンピュータ読み取り可能な識別子です。

`msg`
: エラーの人間が読める説明です。

`ctx`
: エラー・メッセージの表示に必要な値を含むオプションのオブジェクトです。

以下はデモです。

```python
from typing import List
from pydantic import BaseModel, ValidationError, conint


class Location(BaseModel):
    lat = 0.1
    lng = 10.1


class Model(BaseModel):
    is_required: float
    gt_int: conint(gt=42)
    list_of_ints: List[int] = None
    a_float: float = None
    recursive_model: Location = None


data = dict(
    list_of_ints=['1', 2, 'bad'],
    a_float='not a float',
    recursive_model={'lat': 4.2, 'lng': 'New York'},
    gt_int=21,
)

try:
    Model(**data)
except ValidationError as e:
    print(e)
    """
    5 validation errors for Model
    is_required
      field required (type=value_error.missing)
    gt_int
      ensure this value is greater than 42 (type=value_error.number.not_gt;
    limit_value=42)
    list_of_ints -> 2
      value is not a valid integer (type=type_error.integer)
    a_float
      value is not a valid float (type=type_error.float)
    recursive_model -> lng
      value is not a valid float (type=type_error.float)
    """

try:
    Model(**data)
except ValidationError as e:
    print(e.json())
    """
    [
      {
        "loc": [
          "is_required"
        ],
        "msg": "field required",
        "type": "value_error.missing"
      },
      {
        "loc": [
          "gt_int"
        ],
        "msg": "ensure this value is greater than 42",
        "type": "value_error.number.not_gt",
        "ctx": {
          "limit_value": 42
        }
      },
      {
        "loc": [
          "list_of_ints",
          2
        ],
        "msg": "value is not a valid integer",
        "type": "type_error.integer"
      },
      {
        "loc": [
          "a_float"
        ],
        "msg": "value is not a valid float",
        "type": "type_error.float"
      },
      {
        "loc": [
          "recursive_model",
          "lng"
        ],
        "msg": "value is not a valid float",
        "type": "type_error.float"
      }
    ]
    """

```

(このスクリプトは完成しており、「そのまま」実行できるはずです。 `json()` にはデフォルトで `indent=2` が設定されていますが、ここと以下のJSONに手を加えて、若干簡潔にしています)

### カスタムエラーについて

カスタムデータタイプやバリデータでは、エラーを発生させるために `ValueError`, `TypeError`, `AssertionError` を使用する必要があります。

`@validator` デコレーターの使い方の詳細は、[バリデーター]()を参照してください。

```python
from pydantic import BaseModel, ValidationError, validator


class Model(BaseModel):
    foo: str

    @validator('foo')
    def name_must_contain_space(cls, v):
        if v != 'bar':
            raise ValueError('value must be "bar"')

        return v


try:
    Model(foo='ber')
except ValidationError as e:
    print(e.errors())
    """
    [
        {
            'loc': ('foo',),
            'msg': 'value must be "bar"',
            'type': 'value_error',
        },
    ]
    """
```

(このスクリプトは完成しているので、「そのまま」実行してください)

また、独自のエラークラスを定義することもでき、カスタムエラーコード、メッセージテンプレート、コンテキストを指定することができます。

```python
from pydantic import BaseModel, PydanticValueError, ValidationError, validator


class NotABarError(PydanticValueError):
    code = 'not_a_bar'
    msg_template = 'value is not "bar", got "{wrong_value}"'


class Model(BaseModel):
    foo: str

    @validator('foo')
    def name_must_contain_space(cls, v):
        if v != 'bar':
            raise NotABarError(wrong_value=v)
        return v


try:
    Model(foo='ber')
except ValidationError as e:
    print(e.json())
    """
    [
      {
        "loc": [
          "foo"
        ],
        "msg": "value is not \"bar\", got \"ber\"",
        "type": "value_error.not_a_bar",
        "ctx": {
          "wrong_value": "ber"
        }
      }
    ]
    """

```

(このスクリプトは完成しているので、「そのまま」実行することができます)


## ヘルパー関数について

Pydantic では、データを解析するためにモデルに3つの `classmethod` ヘルパー関数を用意しています。

- `parse_obj`: これはモデルの `__init__` メソッドと非常によく似ていますが、キーワード引数ではなく `dict` を受け取ることが違います。渡されたオブジェクトが `dict` でない場合、 `ValidationError` が発生します。
- `parse_raw`: `str` または `bytes` を受け取り、それを `json` として解析し、その結果を `parse_obj` に渡します。 `content_type` 引数を適切に設定することで `pickle` データの解析もサポートしています。
- `parse_file`: ファイルパスを受け取り、ファイルを読み込んで、その内容を `parse_raw` に渡します。 `content_type` が省略された場合は、ファイルの拡張子から推測されます。

```python
import pickle
from datetime import datetime
from pathlib import Path

from pydantic import BaseModel, ValidationError


class User(BaseModel):
    id: int
    name = 'John Doe'
    signup_ts: datetime = None


m = User.parse_obj({'id': 123, 'name': 'James'})
print(m)
#> id=123 signup_ts=None name='James'

try:
    User.parse_obj(['not', 'a', 'dict'])
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    __root__
      User expected dict not list (type=type_error)
    """

# assumes json as no content type passed
m = User.parse_raw('{"id": 123, "name": "James"}')
print(m)
#> id=123 signup_ts=None name='James'

pickle_data = pickle.dumps({
    'id': 123,
    'name': 'James',
    'signup_ts': datetime(2017, 7, 14)
})
m = User.parse_raw(
    pickle_data, content_type='application/pickle', allow_pickle=True
)
print(m)
#> id=123 signup_ts=datetime.datetime(2017, 7, 14, 0, 0) name='James'

path = Path('data.json')
path.write_text('{"id": 123, "name": "James"}')
m = User.parse_file(path)
print(m)
#> id=123 signup_ts=None name='James'

```

(このスクリプトは完成しているので、「そのまま」実行してください)

> Warning:
> [pickle の公式ドキュメント]() からの引用：「pickle モジュールは、誤って作られたデータや悪意を持って作られたデータに対して安全ではありません。信頼されていない、あるいは認証されていないソースから受け取ったデータを決して `unpickle` しないでください」

> Info:
> 任意のコードを実行される可能性があるため、セキュリティ対策として `pickle` データを読み込むためには、解析関数に明示的に `allow_pickle` を渡す必要があります。

### バリデーションのないモデルの作成について

pydantic では **検証なしで** モデルを作成できる `construct()` メソッドも提供しています。これは、データがすでに検証されていたり、信頼できるソースから来ていたりして、できるだけ効率的にモデルを作成したい場合に便利です（ `construct()` は、完全に検証してモデルを作成するよりも一般的に約30倍高速です）。

> Warning:
> `construct()` は検証を行わないので、無効なモデルを作成する可能性があります。 `construct()` メソッドは、すでに検証されたデータ、もしくは信頼できるデータに対してのみ使用するようにしてください。

```python
from pydantic import BaseModel


class User(BaseModel):
    id: int
    age: int
    name: str = 'John Doe'


original_user = User(id=123, age=32)

user_data = original_user.dict()
print(user_data)
#> {'id': 123, 'age': 32, 'name': 'John Doe'}
fields_set = original_user.__fields_set__
print(fields_set)
#> {'age', 'id'}

# ...
# pass user_data and fields_set to RPC or save to the database etc.
# ...

# you can then create a new instance of User without
# re-running validation which would be unnecessary at this point:
new_user = User.construct(_fields_set=fields_set, **user_data)
print(repr(new_user))
#> User(id=123, age=32, name='John Doe')
print(new_user.__fields_set__)
#> {'age', 'id'}

# construct can be dangerous, only use it with validated data!:
bad_user = User.construct(id='dog')
print(repr(bad_user))
#> User(id='dog', name='John Doe')
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

`construct()` のキーワード引数 `_fields_set` はオプションですが、どのフィールドがもともと設定されていて、どのフィールドが設定されていないかをより正確に知ることができます。省略された場合 `__fields_set__` は提供されたデータのキーになります。

例えば、上記の例では `_fields_set` が提供されていない場合、 `new_user.__fields_set__` は `{'id', 'age', 'name'}` となります。

## ジェネリック・モデル

Pydantic は、共通のモデル構造の再利用を容易にするために、ジェネリックモデルの作成をサポートしています。

> Warning:
> ジェネリックモデルは Python 3.7 以上でのみサポートされています。これは Python 3.6 と Python 3.7 の間でジェネリックモデルの実装方法に多くの微妙な変更があるためです。

ジェネリックモデルを宣言するためには、以下のステップを実行します。

- モデルのパラメータ化に使用する1つ以上の `typing.TypeVar` インスタンスを宣言します。
- `pydantic.generics.GenericModel` と `typing.Generic` を継承した pydantic モデルを宣言し、 `typing.Generic` にパラメータとして `TypeVar` インスタンスを渡す。
- `TypeVar` インスタンスは、他の型や pydantic モデルに置き換えたい場合のアノテーションとして使用します。

ここでは、簡単に再利用できる HTTP レスポンスペイロードのラッパーを作成するために `GenericModel` を使用する例を示します。

```python
from typing import Generic, TypeVar, Optional, List

from pydantic import BaseModel, validator, ValidationError
from pydantic.generics import GenericModel

DataT = TypeVar('DataT')


class Error(BaseModel):
    code: int
    message: str


class DataModel(BaseModel):
    numbers: List[int]
    people: List[str]


class Response(GenericModel, Generic[DataT]):
    data: Optional[DataT]
    error: Optional[Error]

    @validator('error', always=True)
    def check_consistency(cls, v, values):
        if v is not None and values['data'] is not None:
            raise ValueError('must not provide both data and error')
        if v is None and values.get('data') is None:
            raise ValueError('must provide data or error')
        return v


data = DataModel(numbers=[1, 2, 3], people=[])
error = Error(code=404, message='Not found')

print(Response[int](data=1))
#> data=1 error=None
print(Response[str](data='value'))
#> data='value' error=None
print(Response[str](data='value').dict())
#> {'data': 'value', 'error': None}
print(Response[DataModel](data=data).dict())
"""
{
    'data': {'numbers': [1, 2, 3], 'people': []},
    'error': None,
}
"""
print(Response[DataModel](error=error).dict())
"""
{
    'data': None,
    'error': {'code': 404, 'message': 'Not found'},
}
"""
try:
    Response[int](data='value')
except ValidationError as e:
    print(e)
    """
    2 validation errors for Response[int]
    data
      value is not a valid integer (type=type_error.integer)
    error
      must provide data or error (type=value_error)
    """
```

(このスクリプトは完成しているので、「そのまま」実行してください)

ジェネリックモデルの定義で `Config` を設定したり、 `validator` を使用したりすると、 `BaseModel` を継承した場合と同じように、具象サブクラスにも適用されます。また、ジェネリッククラスに定義されたメソッドもすべて継承されます。

Pydantic のジェネリックは mypyとも適切に統合されており、 `GenericModel` を使用せずに型を宣言した場合に mypy が提供するであろう型チェックをすべて受けることができます。

> Note:
> 内部的には pydantic は実行時に（キャッシュされた）具体的な `BaseModel` を生成するために `create_model` を使用しているため、 `GenericModel` を使用することで生じるオーバーヘッドは基本的にゼロです。

`TypeVar` のインスタンスを置き換えずに `GenericModel` を継承するためには、クラスは `typing.Generic` を継承しなければなりません。

```python
from typing import TypeVar, Generic
from pydantic.generics import GenericModel

TypeX = TypeVar('TypeX')


class BaseClass(GenericModel, Generic[TypeX]):
    X: TypeX


class ChildClass(BaseClass[TypeX], Generic[TypeX]):
    # Inherit from Generic[TypeX]
    pass


# Replace TypeX by int
print(ChildClass[int](X=1))
#> X=1
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

また、 `GenericModel` のジェネリックサブクラスを作成することで、スーパークラスの型パラメータを部分的または完全に置き換えることができます。

```python
from typing import TypeVar, Generic
from pydantic.generics import GenericModel

TypeX = TypeVar('TypeX')
TypeY = TypeVar('TypeY')
TypeZ = TypeVar('TypeZ')


class BaseClass(GenericModel, Generic[TypeX, TypeY]):
    x: TypeX
    y: TypeY


class ChildClass(BaseClass[int, TypeY], Generic[TypeY, TypeZ]):
    z: TypeZ


# Replace TypeY by str
print(ChildClass[str, int](x=1, y='y', z=3))
#> x=1 y='y' z=3
```

(このスクリプトは完成しているので、"そのまま "実行することができます)

具象サブクラスの名前が重要な場合は、デフォルトの動作をオーバーライドすることもできます。

```python
from typing import Generic, TypeVar, Type, Any, Tuple

from pydantic.generics import GenericModel

DataT = TypeVar('DataT')


class Response(GenericModel, Generic[DataT]):
    data: DataT

    @classmethod
    def __concrete_name__(cls: Type[Any], params: Tuple[Type[Any], ...]) -> str:
        return f'{params[0].__name__.title()}Response'


print(Response[int](data=1))
#> data=1
print(Response[str](data='a'))
#> data='a'
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

ネストされたモデルで同じTypeVarを使用することで、モデル内の異なるポイントで型付け関係を強制することができます。

```python
from typing import Generic, TypeVar

from pydantic import ValidationError
from pydantic.generics import GenericModel

T = TypeVar('T')


class InnerT(GenericModel, Generic[T]):
    inner: T


class OuterT(GenericModel, Generic[T]):
    outer: T
    nested: InnerT[T]


nested = InnerT[int](inner=1)
print(OuterT[int](outer=1, nested=nested))
#> outer=1 nested=InnerT[int](inner=1)
try:
    nested = InnerT[str](inner='a')
    print(OuterT[int](outer='a', nested=nested))
except ValidationError as e:
    print(e)
    """
    2 validation errors for OuterT[int]
    outer
      value is not a valid integer (type=type_error.integer)
    nested -> inner
      value is not a valid integer (type=type_error.integer)
    """
```

(このスクリプトは完成しているので、「そのまま」実行してください)

また Pydantic では、 `GenericModel` を `List` や `Dict` などの組み込み汎用型と同様に、パラメータを指定せずに放置したり、境界のある `TypeVar` インスタンスを使用したりすることができます。

- ジェネリックモデルをインスタンス化する前にパラメータを指定しなかった場合、それらは `Any` として扱われます。
- サブクラスのチェックを追加するために、1つまたは複数の限定されたパラメータでモデルをパラメトリックに表現することができます。

また `List` や `Dict` のように、 `TypeVar` を使って指定されたパラメータは、後で具象型で置き換えることができます。

```python
from typing import Generic, TypeVar

from pydantic import ValidationError
from pydantic.generics import GenericModel

AT = TypeVar('AT')
BT = TypeVar('BT')


class Model(GenericModel, Generic[AT, BT]):
    a: AT
    b: BT


print(Model(a='a', b='a'))
#> a='a' b='a'

IntT = TypeVar('IntT', bound=int)
typevar_model = Model[int, IntT]
print(typevar_model(a=1, b=1))
#> a=1 b=1
try:
    typevar_model(a='a', b='a')
except ValidationError as exc:
    print(exc)
    """
    2 validation errors for Model[int, IntT]
    a
      value is not a valid integer (type=type_error.integer)
    b
      value is not a valid integer (type=type_error.integer)
    """

concrete_model = typevar_model[int]
print(concrete_model(a=1, b=1))
#> a=1 b=1
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

## 動的なモデル作成について

モデルの形状が実行時までわからない場合があります。そのために pydantic では `create_model` メソッドを用意し、その場でモデルを作成できるようにしています。

```python
from pydantic import BaseModel, create_model

DynamicFoobarModel = create_model('DynamicFoobarModel', foo=(str, ...), bar=123)


class StaticFoobarModel(BaseModel):
    foo: str
    bar: int = 123
```

ここでは `StaticFoobarModel` と `DynamicFoobarModel` は同じものです。

> Warning:
> フィールドのデフォルトとしての ellipsis と、注釈のみのフィールドの違いについては [Required Optional Fields]() の注釈を参照してください。詳細は [samuelcolvin/pydantic#1047]() を参照してください。

フィールドは `(<type>, <default value>)` 形式のタプルか、デフォルト値のみで定義されます。特別なキーワード引数  `__config__` と `__base__` を使って、新しいモデルをカスタマイズすることができます。これには、ベースモデルを追加のフィールドで拡張することも含まれます。

```python
from pydantic import BaseModel, create_model


class FooModel(BaseModel):
    foo: str
    bar: int = 123


BarModel = create_model(
    'BarModel',
    apple='russet',
    banana='yellow',
    __base__=FooModel,
)
print(BarModel)
#> <class 'pydantic.main.BarModel'>
print(BarModel.__fields__.keys())
#> dict_keys(['foo', 'bar', 'apple', 'banana'])
```

また `__validators__` の引数に dict を渡してバリデータを追加することもできます。

```python
from pydantic import create_model, ValidationError, validator


def username_alphanumeric(cls, v):
    assert v.isalnum(), 'must be alphanumeric'
    return v


validators = {
    'username_validator':
    validator('username')(username_alphanumeric)
}

UserModel = create_model(
    'UserModel',
    username=(str, ...),
    __validators__=validators
)

user = UserModel(username='scolvin')
print(user)
#> username='scolvin'

try:
    UserModel(username='scolvi%n')
except ValidationError as e:
    print(e)
    """
    1 validation error for UserModel
    username
      must be alphanumeric (type=assertion_error)
    """
```

## NamedTuple または TypedDict からのモデル作成

アプリケーションの中で `NamedTuple` や `TypedDict` を継承したクラスを既に使用していて、 `BaseModel` を持つために全ての情報を重複させたくない場合があります。このため pydantic は `create_model_from_namedtuple` と`create_model_from_typeddict` メソッドを提供しています。これらのメソッドは、create_modelと全く同じキーワード引数を持ちます。

```python
from typing_extensions import TypedDict

from pydantic import ValidationError, create_model_from_typeddict


class User(TypedDict):
    name: str
    id: int


class Config:
    extra = 'forbid'


UserM = create_model_from_typeddict(User, __config__=Config)
print(repr(UserM(name=123, id='3')))
#> User(name='123', id=3)

try:
    UserM(name=123, id='3', other='no')
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    other
      extra fields not permitted (type=value_error.extra)
    """
```

## カスタム・ルート・タイプについて

Pydantic モデルは `__root__` フィールドを宣言することで、カスタムルートタイプを定義することができます。

ルートタイプは pydantic でサポートされている任意のタイプとすることができ、 `__root__` フィールドのタイプヒントによって指定されます。ルート値は `__root__` キーワード引数を介してモデル `__init__` に渡すか、 `parse_obj` の最初で唯一の引数として渡すことができます。

```python
from typing import List
import json
from pydantic import BaseModel
from pydantic.schema import schema


class Pets(BaseModel):
    __root__: List[str]


print(Pets(__root__=['dog', 'cat']))
#> __root__=['dog', 'cat']
print(Pets(__root__=['dog', 'cat']).json())
#> ["dog", "cat"]
print(Pets.parse_obj(['dog', 'cat']))
#> __root__=['dog', 'cat']
print(Pets.schema())
"""
{
    'title': 'Pets',
    'type': 'array',
    'items': {'type': 'string'},
}
"""
pets_schema = schema([Pets])
print(json.dumps(pets_schema, indent=2))
"""
{
  "definitions": {
    "Pets": {
      "title": "Pets",
      "type": "array",
      "items": {
        "type": "string"
      }
    }
  }
}
"""
```

カスタムルートタイプを持つモデルに対して、第一引数に `dict` を指定して `parse_obj` メソッドを呼び出した場合、以下のようなロジックが用いられます。

- カスタムルートタイプがマッピングタイプ（例： `Dict` または `Mapping` ）の場合、引数自体は常にカスタムルートタイプに対して検証されます。
- 他のカスタムルートタイプの場合、 `dict` が `__root__` という値を持つキーを正確に 1 つ持っていれば、対応する値はカスタムルートタイプに対して検証されます。
- それ以外の場合は、 `dict` 自体がカスタム・ルート・タイプに対して検証されます。

これは以下の例で示されています。

```python
from typing import List, Dict
from pydantic import BaseModel, ValidationError


class Pets(BaseModel):
    __root__: List[str]


print(Pets.parse_obj(['dog', 'cat']))
#> __root__=['dog', 'cat']
print(Pets.parse_obj({'__root__': ['dog', 'cat']}))  # not recommended
#> __root__=['dog', 'cat']


class PetsByName(BaseModel):
    __root__: Dict[str, str]


print(PetsByName.parse_obj({'Otis': 'dog', 'Milo': 'cat'}))
#> __root__={'Otis': 'dog', 'Milo': 'cat'}
try:
    PetsByName.parse_obj({'__root__': {'Otis': 'dog', 'Milo': 'cat'}})
except ValidationError as e:
    print(e)
    """
    1 validation error for PetsByName
    __root__ -> __root__
      str type expected (type=type_error.str)
    """
```

> Warning:
> マッピングされていないカスタムルートタイプに対して、単一のキー　`__root__` を持つ dict で `parse_obj` メソッドを呼び出すことは、現在後方互換性のためにサポートされていますが、推奨されておらず、将来のバージョンでは廃止される可能性があります。


`__root__` フィールドのアイテムに直接アクセスしたり、アイテムを反復処理したい場合は、次の例に示すように、カスタムの `__iter__` および `__getitem__` 関数を実装することができます。

```python
from typing import List
from pydantic import BaseModel


class Pets(BaseModel):
    __root__: List[str]

    def __iter__(self):
        return iter(self.__root__)

    def __getitem__(self, item):
        return self.__root__[item]


pets = Pets.parse_obj(['dog', 'cat'])
print(pets[0])
#> dog
print([pet for pet in pets])
#> ['dog', 'cat']
```

## フェイクの不変性について

モデルは `allow_mutation = False` で不変性を設定することができます。これが設定されていると、インスタンスの属性値を変更しようとするとエラーになります。 `Config` についての詳細は [model config]() を参照してください。

> Warning:
> python の不変性は決して厳密ではありません。もし開発者が決意を持って/愚かにも、いわゆる "不変" なオブジェクトをいつでも変更することができます。

```python
from pydantic import BaseModel


class FooBarModel(BaseModel):
    a: str
    b: dict

    class Config:
        allow_mutation = False


foobar = FooBarModel(a='hello', b={'apple': 'pear'})

try:
    foobar.a = 'different'
except TypeError as e:
    print(e)
    #> "FooBarModel" is immutable and does not support item assignment

print(foobar.a)
#> hello
print(foobar.b)
#> {'apple': 'pear'}
foobar.b['apple'] = 'grape'
print(foobar.b)
#> {'apple': 'grape'}
```

`a` を変更しようとするとエラーになり、 `a` は変更されないままです。しかし dict の `b` は mutable であり、 `foobar` の immutability は `b` の変更を止めることはできません。

## 抽象ベースクラスについて

Pydantic モデルは Python の [抽象ベースクラス]()（ABC）と一緒に使うことができます。

```python
import abc
from pydantic import BaseModel


class FooBarModel(BaseModel, abc.ABC):
    a: str
    b: int

    @abc.abstractmethod
    def my_abstract_method(self):
        pass
```

(このスクリプトは完成しているので、「そのまま」実行してください)

## フィールドの順序付け

モデルでは、以下の理由からフィールドの順序が重要になります。

- フィールドが定義された順に検証が行われる。[フィールドバリデータ]() は前のフィールドの値にはアクセスできるが、後のフィールドの値にはアクセスできない。
- フィールドの順序がモデルのスキーマに保存される
- フィールドの順序は検証エラーでも保持される
- `.dict()` や `.json()` などでフィールドの順序が保持される。

**v1.0** では、アノテーションを持つすべてのフィールド（アノテーションのみの場合も、デフォルト値を持つ場合も）は、アノテーションを持たないすべてのフィールドよりも優先されます。それぞれのグループ内では、フィールドは定義された順に残ります。

```python
from pydantic import BaseModel, ValidationError


class Model(BaseModel):
    a: int
    b = 2
    c: int = 1
    d = 0
    e: float


print(Model.__fields__.keys())
#> dict_keys(['a', 'c', 'e', 'b', 'd'])
m = Model(e=2, a=1)
print(m.dict())
#> {'a': 1, 'c': 1, 'e': 2.0, 'b': 2, 'd': 0}
try:
    Model(a='x', b='x', c='x', d='x', e='x')
except ValidationError as e:
    error_locations = [e['loc'] for e in e.errors()]

print(error_locations)
#> [('a',), ('c',), ('e',), ('b',), ('d',)]
```

(このスクリプトは完成しているので、「そのまま」実行してください)

> Warning:
> 上の例で示されているように、同じモデルでアノテーションされたフィールドとされていないフィールドの使用を組み合わせると、意外なフィールドの順序になることがあります。これは python の制限によるものです
> 
> そのため、フィールドの順序が保たれるように、デフォルト値で型が決まる場合でも **すべてのフィールドに型のアノテーションを追加することをお勧めします**。

## 必須フィールド

フィールドを必須として宣言するには、アノテーションのみを使用して宣言するか、 ellipsis (`...`)を値として使用することができます。

```python
from pydantic import BaseModel, Field


class Model(BaseModel):
    a: int
    b: int = ...
    c: int = Field(...)
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

ここで `Field` とは [フィールド関数]() のことです。

ここでは `a`, `b`, `c` はすべて必須です。ただし `b` での ellipse の使用は mypy との相性が悪く、 **v1.0** 以降はほとんどの場合で避けるべきです。

### 必須オプションフィールド

> Warning:
> **v1.2** 以降、 only nullable (`Optional[...]`, `Union[None, ...]`, `Any`) フィールドと ellipse (`...`) をデフォルト値とする nullable フィールドは、同じ意味ではなくなりました。
> 
> 状況によっては **v1.2** とそれ以前の **v1.*** との間に完全な下位互換性がない場合があります。

必須であるにもかかわらず `None` の値を取ることができるフィールドを指定したい場合は、 `Optional` と `...` を使用することができます。

```python
from typing import Optional
from pydantic import BaseModel, Field, ValidationError


class Model(BaseModel):
    a: Optional[int]
    b: Optional[int] = ...
    c: Optional[int] = Field(...)


print(Model(b=1, c=2))
#> a=None b=1 c=2
try:
    Model(a=1, b=2)
except ValidationError as e:
    print(e)
    """
    1 validation error for Model
    c
      field required (type=value_error.missing)
    """
```

(このスクリプトは完成しているので、「そのまま」実行してください)

このモデルでは、 `a`, `b`, `c` は値として `None` を取ることができます。しかし `a` はオプションで `b` と `c` は必須です。 `b` と `c` は、たとえ値が `None` であっても、値を必要とします。

## 動的なデフォルト値を持つフィールド

デフォルト値を持つフィールドを宣言するとき、それを動的（つまりモデルごとに異なる）にしたい場合があります。そのためには `default_factory` を使用するとよいでしょう。

> In Beta:
> `default_factory` 引数はベータ版で、 **v1.5** の pydantic に暫定的に追加されたものです。将来のリリースでは大きく変更される可能性があり、その署名や動作は **v2** になるまで具体的にはなりません。暫定版である間のコミュニティからのフィードバックは非常に有益です。 [#866]() にコメントするか、新しい課題を作成してください。

以下は使い方の例

```python
from datetime import datetime
from uuid import UUID, uuid4
from pydantic import BaseModel, Field


class Model(BaseModel):
    uid: UUID = Field(default_factory=uuid4)
    updated: datetime = Field(default_factory=datetime.utcnow)


m1 = Model()
m2 = Model()
print(f'{m1.uid} != {m2.uid}')
#> ea4c0c90-2c4f-41cb-bd35-5d3e3400cf64 != 5df6224f-1ab5-48a0-bbeb-fb2453054496
print(f'{m1.updated} != {m2.updated}')
#> 2021-05-11 20:28:47.761447 != 2021-05-11 20:28:47.761469
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

`Field` は [field 関数]() を指します。

> Warning:
> `default_factory` はフィールドタイプが設定されていることを期待しています。さらに、デフォルト値を `validate_all` で検証したい場合、 pydantic は `default_factory` を呼び出す必要があり、副作用が発生する可能性があります。

## 自動的に除外される属性

アンダースコアで始まるクラス変数や `typing.ClassVar` でアノテーションされた属性はモデルから自動的に除外されます。

## モデルのプライベート属性

モデルのインスタンス上で内部属性を変化させたり操作したりする必要がある場合、 `PrivateAttr` を使用して宣言することができます。

```python
from datetime import datetime
from random import randint

from pydantic import BaseModel, PrivateAttr


class TimeAwareModel(BaseModel):
    _processed_at: datetime = PrivateAttr(default_factory=datetime.now)
    _secret_value: str = PrivateAttr()

    def __init__(self, **data):
        super().__init__(**data)
        # this could also be done with default_factory
        self._secret_value = randint(1, 5)


m = TimeAwareModel()
print(m._processed_at)
#> 2021-05-11 20:28:48.051834
print(m._secret_value)
#> 3
```

(このスクリプトは完成しているので、「そのまま」実行してください)

モデルのフィールドとの衝突を防ぐために、プライベート属性名はアンダースコアで始まる必要があります。

`Config.underscore_attrs_are_private` が `True` の場合、 `ClassVar` 以外のアンダースコア属性は private として扱われます。

```python
from typing import ClassVar

from pydantic import BaseModel


class Model(BaseModel):
    _class_var: ClassVar[str] = 'class var value'
    _private_attr: str = 'private attr value'

    class Config:
        underscore_attrs_are_private = True


print(Model._class_var)
#> class var value
print(Model._private_attr)
#> <member '_private_attr' of 'Model' objects>
print(Model()._private_attr)
#> private attr value
```

(このスクリプトは完成しているので、「そのまま」実行できます)

pydanticはクラス作成時に、プライベート属性が入った `__slot__` を構築します。

## データを指定された型にパース

Pydantic には、スタンドアロンのユーティリティー関数 `parse_obj_as` が含まれており、これを使って pydantic モデルを生成するために使用されるパースロジックをよりアドホックな方法で適用することができます。この関数は `BaseModel.parse_obj` と似たような動作をしますが、任意の pydantic 互換の型で動作します。

これは特に BaseModel の直接のサブクラスではない型に結果をパースしたい場合に便利です。例えば

```python
from typing import List

from pydantic import BaseModel, parse_obj_as


class Item(BaseModel):
    id: int
    name: str


# `item_data` could come from an API call, eg., via something like:
# item_data = requests.get('https://my-api.com/items').json()
item_data = [{'id': 1, 'name': 'My Item'}]

items = parse_obj_as(List[Item], item_data)
print(items)
#> [Item(id=1, name='My Item')]
```

(このスクリプトは完成しているので、「そのまま」実行してください)

この関数は `BaseModel` のフィールドとして pydantic が扱えるあらゆるタイプのデータを解析することができます。

Pydanticに は `parse_file_as` と `parse_raw_as` という2つの同様のスタンドアロン関数も含まれており、これらは `BaseModel.parse_file` と `BaseModel.parse_raw` に類似しています。

## データの変換

pydantic は入力データをモデルのフィールドタイプに強制的に適合させるためにキャストすることがありますが、場合によっては情報が失われることがあります。例えば、以下のようになります。

```python
from pydantic import BaseModel


class Model(BaseModel):
    a: int
    b: float
    c: str


print(Model(a=3.1415, b=' 2.72 ', c=123).dict())
#> {'a': 3, 'b': 2.72, 'c': '123'}
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

これは pydantic の意図的な決定であり、一般的には最も有用なアプローチです。このテーマについての長い議論は[こちら](https://github.com/samuelcolvin/pydantic/issues/578)をご覧ください。

## モデルの署名

すべての pydantic モデルは、そのフィールドに基づいて署名が生成されます。

```python
import inspect
from pydantic import BaseModel, Field


class FooModel(BaseModel):
    id: int
    name: str = None
    description: str = 'Foo'
    apple: int = Field(..., alias='pear')


print(inspect.signature(FooModel))
#> (*, id: int, name: str = None, description: str = 'Foo', pear: int) -> None
```

正確なシグネチャは、イントロスペクションの目的や `FastAPI` や `hypothesis` のようなライブラリに役立ちます。

生成されたシグネチャは、カスタムの `__init__` 関数も尊重します。

```python
import inspect

from pydantic import BaseModel


class MyModel(BaseModel):
    id: int
    info: str = 'Foo'

    def __init__(self, id: int = 1, *, bar: str, **data) -> None:
        """My custom init!"""
        super().__init__(id=id, bar=bar, **data)


print(inspect.signature(MyModel))
#> (id: int = 1, *, bar: str, info: str = 'Foo') -> None
```

シグネチャに含めるためには、フィールドのエイリアスや名前が有効な python の識別子でなければなりません。 pydantic は名前よりもエイリアスを優先しますが、エイリアスが有効な python の識別子でない場合はフィールド名を使用することができます。

フィールドのエイリアスと名前の両方が無効な識別子の場合は、 `**data` 引数が追加されます。また `Config.extra` が `Extra.allow` の場合、 `**data` 引数は常にシグネチャ内に存在します。

> Note:
> モデルシグネチャ内の型は、モデルアノテーションで宣言されたものと同じであり、必ずしもそのフィールドに実際に提供可能なすべての型ではありません。この問題は#1055が解決されれば、いつか修正されるかもしれません。

## dataclass

pydantic の BaseModel を使用したくない場合は、代わりに標準的なデータクラス(Python 3.7で導入)で同じデータ検証を行うことができます。

dataclasses は Python3.6 で dataclasses backport パッケージを使用して動作します。

```python
from datetime import datetime
from pydantic.dataclasses import dataclass


@dataclass
class User:
    id: int
    name: str = 'John Doe'
    signup_ts: datetime = None


user = User(id='42', signup_ts='2032-06-21T12:00')
print(user)
#> User(id=42, name='John Doe', signup_ts=datetime.datetime(2032, 6, 21, 12, 0))
```

(このスクリプトは完成していますので、そのまま実行してください)

> Note:
> `pydantic.dataclasses.dataclass` は `dataclasses.dataclass` を検証付きで置き換えるものであり、 `pydantic.BaseModel` を置き換えるものではないことに留意してください（初期化フックの動作方法に若干の違いがあります）。 `pydantic.BaseModel` をサブクラス化した方が良い場合もあります。
> 
> 詳しい情報や議論は [samuelcolvin/pydantic#710]() を参照してください。

pydantic の標準的なフィールドタイプをすべて使用することができ、結果として得られるデータクラスは、標準ライブラリのデータクラスデコレータで作成されたものと同じになります。

基礎となるモデルとそのスキーマは `__pydantic_model__` を通してアクセスできます。また `default_factory` を必要とするフィールドは `dataclasses.field` で指定することができます。

```python
import dataclasses
from typing import List, Optional

from pydantic import Field
from pydantic.dataclasses import dataclass


@dataclass
class User:
    id: int
    name: str = 'John Doe'
    friends: List[int] = dataclasses.field(default_factory=lambda: [0])
    age: Optional[int] = dataclasses.field(
        default=None,
        metadata=dict(title='The age of the user', description='do not lie!')
    )
    height: Optional[int] = Field(None, title='The height in cm', ge=50, le=300)


user = User(id='42')
print(user.__pydantic_model__.schema())
"""
{
    'title': 'User',
    'type': 'object',
    'properties': {
        'id': {'title': 'Id', 'type': 'integer'},
        'name': {
            'title': 'Name',
            'default': 'John Doe',
            'type': 'string',
        },
        'friends': {
            'title': 'Friends',
            'type': 'array',
            'items': {'type': 'integer'},
        },
        'age': {
            'title': 'The age of the user',
            'description': 'do not lie!',
            'type': 'integer',
        },
        'height': {
            'title': 'The height in cm',
            'minimum': 50,
            'maximum': 300,
            'type': 'integer',
        },
    },
    'required': ['id'],
}
"""
```

(このスクリプトは完成しているので、「そのまま」実行することができます)

`pydantic.dataclasses.dataclass` の引数は標準的なデコレータと同じですが、1つだけ追加のキーワード引数configがあり、これはConfigと同じ意味を持ちます。

> Warning:
> **v1.2** 以降では pydantic のデータクラスを型チェックするために [Mypy プラグイン]() をインストールする必要があります。

バリデータとデータクラスの組み合わせについての詳細は [dataclass validator] を参照してください。

## ネストしたデータクラス

入れ子になったデータクラスは、データクラスと通常のモデルの両方でサポートされています。

```python
from pydantic import AnyUrl
from pydantic.dataclasses import dataclass


@dataclass
class NavbarButton:
    href: AnyUrl


@dataclass
class Navbar:
    button: NavbarButton


navbar = Navbar(button=('https://example.com',))
print(navbar)
#> Navbar(button=NavbarButton(href=AnyUrl('https://example.com', scheme='https',
#> host='example.com', tld='com', host_type='domain')))
```

(このスクリプトは完成しているので、そのまま実行してください)

データクラスの属性には、タプル、辞書、またはデータクラス自体のインスタンスを含めることができます。

## stdlib のデータクラスと pydantic のデータクラス

### stdlib データクラスを pydantic データクラスに変換

stdlib のデータクラス（ネストされていてもいなくても）は `pydantic.dataclasses.dataclass` で装飾するだけで簡単にpydanticのデータクラスに変換できます。

```python
import dataclasses
from datetime import datetime
from typing import Optional

import pydantic


@dataclasses.dataclass
class Meta:
    modified_date: Optional[datetime]
    seen_count: int


@dataclasses.dataclass
class File(Meta):
    filename: str


File = pydantic.dataclasses.dataclass(File)

file = File(
    filename=b'thefilename',
    modified_date='2020-01-01T00:00',
    seen_count='7',
)
print(file)
#> _Pydantic_File_94603591281216(modified_date=datetime.datetime(2020, 1, 1, 0,
#> 0), seen_count=7, filename='thefilename')

try:
    File(
        filename=['not', 'a', 'string'],
        modified_date=None,
        seen_count=3,
    )
except pydantic.ValidationError as e:
    print(e)
    """
    1 validation error for File
    filename
      str type expected (type=type_error.str)
    """
```

(このスクリプトは完成しているので、そのまま実行することができます)

## stdlib データクラスの継承

stdlib のデータクラス（入れ子になっていてもいなくても）も継承することができ、 pydantic は継承されたすべてのフィールドを自動的に検証します。

```python
import dataclasses

import pydantic


@dataclasses.dataclass
class Z:
    z: int


@dataclasses.dataclass
class Y(Z):
    y: int = 0


@pydantic.dataclasses.dataclass
class X(Y):
    x: int = 0


foo = X(x=b'1', y='2', z='3')
print(foo)
#> _Pydantic_X_94603591264896(z=3, y=2, x=1)

try:
    X(z='pika')
except pydantic.ValidationError as e:
    print(e)
    """
    1 validation error for X
    z
      value is not a valid integer (type=type_error.integer)
    """
```

(このスクリプトは完成しているので、「そのまま」実行してください)

### BaseModel での stdlib データクラスの使用

stdlib のデータクラス（ネストされていてもいなくても）は `BaseModel` と混ぜ合わせると **自動的に pydantic のデータクラスに変換** されることを覚えておいてください!　さらに、生成された `pydantic` データクラスは元のデータクラスと **全く同じ構成** (`order`, `frozen`, ...) になります。

```python
import dataclasses
from datetime import datetime
from typing import Optional

from pydantic import BaseModel, ValidationError


@dataclasses.dataclass(frozen=True)
class User:
    name: str


@dataclasses.dataclass
class File:
    filename: str
    last_modification_time: Optional[datetime] = None


class Foo(BaseModel):
    file: File
    user: Optional[User] = None


file = File(
    filename=['not', 'a', 'string'],
    last_modification_time='2020-01-01T00:00',
)  # nothing is validated as expected
print(file)
#> File(filename=['not', 'a', 'string'],
#> last_modification_time='2020-01-01T00:00')

try:
    Foo(file=file)
except ValidationError as e:
    print(e)
    """
    1 validation error for Foo
    file -> filename
      str type expected (type=type_error.str)
    """

foo = Foo(file=File(filename='myfile'), user=User(name='pika'))
try:
    foo.user.name = 'bulbi'
except dataclasses.FrozenInstanceError as e:
    print(e)
    #> cannot assign to field 'name'
```

(このスクリプトは完成しているので、そのまま実行できます)

### カスタムタイプの使用

標準ライブラリのデータクラスは自動的に変換されて検証が追加されるため、カスタムタイプを使用すると予期しない動作が発生することがあります。このような場合には `arbitrary_types_allowed` を設定に追加するだけです。

```python
import dataclasses

import pydantic


class ArbitraryType:
    def __init__(self, value):
        self.value = value

    def __repr__(self):
        return f'ArbitraryType(value={self.value!r})'


@dataclasses.dataclass
class DC:
    a: ArbitraryType
    b: str


# valid as it is a builtin dataclass without validation
my_dc = DC(a=ArbitraryType(value=3), b='qwe')

try:
    class Model(pydantic.BaseModel):
        dc: DC
        other: str

    Model(dc=my_dc, other='other')
except RuntimeError as e:  # invalid as it is now a pydantic dataclass
    print(e)
    """
    no validator found for <class
    'dataclasses_arbitrary_types_allowed.ArbitraryType'>, see
    `arbitrary_types_allowed` in Config
    """


class Model(pydantic.BaseModel):
    dc: DC
    other: str

    class Config:
        arbitrary_types_allowed = True


m = Model(dc=my_dc, other='other')
print(repr(m))
#> Model(dc=_Pydantic_DC_94603590966432(a=ArbitraryType(value=3), b='qwe'),
#> other='other')
```

(このスクリプトは完成しているので、「そのまま」実行してください)

## 初期化フック

データクラスを初期化するとき `__post_init_post_parse__` を使って検証後にコードを実行することができます。これは、検証前にコードを実行する `__post_init__` とは異なります。

```python
from pydantic.dataclasses import dataclass

@dataclass
クラス Birth:
    年: int
    月: int
    日: int


データクラス
クラス User:
    birth: 誕生

    def __post_init__(self):
        print(self.birth)
        #> {'year': 1995, 'month': 3, 'day': 2}です。

    def __post_init_post_parse__(self):
        print(self.birth)
        #> Birth(year=1995, month=3, day=2)


user = User(**{'birth': {'year': 1995, 'month': 3, 'day': 2}})
```

(このスクリプトは完成しているので、"そのまま "実行することができます)

**v1.0** 以降、 `dataclasses.InitVar` でアノテーションされたフィールドは `__post_init__` と `__post_init_post_parse__` の両方に渡されます。

```python
from dataclasses import InitVar
from pathlib import Path
from typing import Optional

from pydantic.dataclasses import dataclass


@dataclass
class PathData:
    path: Path
    base_path: InitVar[Optional[Path]]

    def __post_init__(self, base_path):
        print(f'Received path={self.path!r}, base_path={base_path!r}')
        #> Received path='world', base_path='/hello'

    def __post_init_post_parse__(self, base_path):
        if base_path is not None:
            self.path = base_path / self.path


path_data = PathData('world', base_path='/hello')
# Received path='world', base_path='/hello'
assert path_data.path == Path('/hello/world')
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

### stdlib の dataclasses との違い

python stdlib の `dataclasses.dataclass` は、検証ステップを実行しないため `__post_init__` メソッドのみを実装していることに注意してください。

`dataclasses.dataclass` の使用法を `pydantic.dataclasses.dataclass` で置き換える場合は `__post_init__` メソッドで実行されるコードを `__post_init_post_parse__` メソッドに移し、検証前に実行する必要のあるコードの一部だけを残すことが推奨されます。

## JSON ダンピング

Pydanticのデータクラスには `.json()` 関数がありません。JSON としてダンプするには、以下のように `pydantic_encoder` を使用する必要があります。

```python
import dataclasses
import json
from typing import List

from pydantic.dataclasses import dataclass
from pydantic.json import pydantic_encoder


@dataclass
class User:
    id: int
    name: str = 'John Doe'
    friends: List[int] = dataclasses.field(default_factory=lambda: [0])


user = User(id='42')
print(json.dumps(user, indent=4, default=pydantic_encoder))
"""
{
    "id": 42,
    "name": "John Doe",
    "friends": [
        0
    ]
}
"""
```

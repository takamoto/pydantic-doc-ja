# Postponed annotations
https://pydantic-docs.helpmanual.io/usage/postponed_annotations/

> Note:
> future import による postponed annotations と `ForwardRef` の両方とも Python 3.7 以上が必要です。

（[PEP563]() に記載されているような） postponed annotations は "just work" です。

```python
from __future__ import annotations
from typing import List
from pydantic import BaseModel


class Model(BaseModel):
    a: List[int]


print(Model(a=('1', 2, 3)))
#> a=[1, 2, 3]
```

(このスクリプトは完成しているので、"そのまま "実行することができます)

内部的には pydantic は `typing.get_type_hints` に似たメソッドを呼び出してアノテーションを解決します。

参照される型がまだ定義されていない場合には `ForwardRef` を使用することができます（ただし [自己参照モデル]() の場合には、型を直接または文字列で参照する方がより簡単な解決策です）。

場合によっては、モデル作成時に `ForwardRef` を解決できないことがあります。例えば、モデルが自分自身をフィールドタイプとして参照している場合などです。このような場合には、モデルが作成された後に `update_forward_refs` を呼び出してから使用する必要があります。

```python
from typing import ForwardRef
from pydantic import BaseModel

Foo = ForwardRef('Foo')


class Foo(BaseModel):
    a: int = 123
    b: Foo = None


Foo.update_forward_refs()

print(Foo())
#> a=123 b=None
print(Foo(b={'a': '321'}))
#> a=123 b=Foo(a=321, b=None)
```

(このスクリプトは完成しているので、「そのまま」実行できるはずです)

> Warning:
> 文字列（型名）をアノテーション（型）に解決するために、 pydantic はルックアップを実行するための名前空間 dict を必要とします。このために `get_type_hints` と同様に `module.__dict__` を使用します。これは pydantic がモジュールのグローバルスコープで定義されていない型をうまく扱えないことを意味します。

例えば、これは問題なく動作します。

```python
from __future__ import annotations
from typing import List  # <-- List is defined in the module's global scope
from pydantic import BaseModel


def this_works():
    class Model(BaseModel):
        a: List[int]

    print(Model(a=(1, 2)))
```

一方、これでは壊れてしまいます。

```python
from __future__ import annotations
from pydantic import BaseModel


def this_is_broken():
    # List is defined inside the function so is not in the module's
    # global scope!
    from typing import List

    class Model(BaseModel):
        a: List[int]

    print(Model(a=(1, 2)))
```

これを解決するには pydantic の出番ではありません。未来のインポートを削除するか、グローバルに型を宣言するかです。

## 自己参照モデル

モデルの作成時に関数 `update_forward_refs()` を呼び出すことで、自己参照モデルを持つデータ構造もサポートされます（忘れた場合は親切なエラーメッセージで知らせてくれます）。

モデル内では、まだ構築されていないモデルを文字列で参照することができます。

```python
from pydantic import BaseModel


class Foo(BaseModel):
    a: int = 123
    #: The sibling of `Foo` is referenced by string
    sibling: 'Foo' = None


Foo.update_forward_refs()

print(Foo())
#> a=123 sibling=None
print(Foo(sibling={'a': '321'}))
#> a=123 sibling=Foo(a=321, sibling=None)
```

(このスクリプトは完成しているので、"そのまま "実行できます)

Python 3.7 以降では `annotations` をインポートしていれば型で参照することもできます（Python や pydanticのバージョンによるサポートについては上記を参照してください）。

```python
from __future__ import annotations
from pydantic import BaseModel


class Foo(BaseModel):
    a: int = 123
    #: The sibling of `Foo` is referenced directly by type
    sibling: Foo = None


Foo.update_forward_refs()

print(Foo())
#> a=123 sibling=None
print(Foo(sibling={'a': '321'}))
#> a=123 sibling=Foo(a=321, sibling=None)
```

(このスクリプトは完成しているので、"そのまま "実行することができます)


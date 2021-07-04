# mypyでの使用

pydantic モデルは mypy でも動作しますが required fields の annotation-only バージョンを使用する必要があります。

```python
from datetime import datetime
from typing import List, Optional
from pydantic import BaseModel, NoneStr


class Model(BaseModel):
    age: int
    first_name = 'John'
    last_name: NoneStr = None
    signup_ts: Optional[datetime] = None
    list_of_ints: List[int]


m = Model(age=42, list_of_ints=[1, '2', b'3'])
print(m.middle_name)  # not a model field!
Model()  # will raise a validation error for age and list_of_ints
```

コードを mypy で実行するには、次のようにします。

```bash
mypy \
  --ignore-missing-imports \
  --follow-imports=skip \
  --strict-optional \
  pydantic_mypy_test.py
```

上記のサンプルコードで mypy を呼び出すと、 mypy が属性アクセスエラーを検出するのがわかると思います。

```
13: error: "Model" has no attribute "middle_name"
```

## ストリクト・オプショナル

`--strict-optional` でコードを通すためには `None` をデフォルトとするすべてのフィールドに `Optional[]` または `Optional[]` のエイリアスを使用する必要があります。(これは mypy では標準です)。

Pydantic では、いくつかの便利な Optional 型や Union 型が用意されています。

- `NoneStr` (別名: `Optional[str]`)
- `NoneBytes` (別名: `Optional[bytes]`)
- `StrBytes` (別名: `Union[str, bytes]`)
- `NoneStrBytes` (別名: `Optional[StrBytes]`)

これらが不十分な場合は、もちろん独自に定義することができます。

## Mypy プラグイン

Pydantic には mypy プラグインが同梱されており、コードのタイプチェック能力を向上させる多くの重要な pydantic 固有の機能を mypy に追加します。

詳細は [pydantic mypy plugin docs]() を参照してください。

## その他の pydantic インターフェース

Pydantic の[データクラス]()や [`validate_arguments` デコレータ]() も mypy でうまく動作するはずです。


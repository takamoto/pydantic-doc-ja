# 設定管理

pydantic の最も便利なアプリケーションの一つが設定管理です。

`BaseSettings` を継承したモデルを作成すると、モデルのイニシャライザはキーワード引数として渡されなかったフィールドの値を環境変数から読み取って決定しようとします。(一致する環境変数が設定されていない場合は、デフォルト値が使用されます)。

これにより、以下のことが容易になります。

- 明確に定義された、タイプヒンティングされたアプリケーション構成クラスの作成
- 設定の変更を環境変数から自動的に読み込む
- 必要に応じてイニシャライザの特定の設定を手動でオーバーライドする（ユニット・テストなど）

例えば、以下のようになります。

```python
from typing import Set

from pydantic import (
    BaseModel,
    BaseSettings,
    PyObject,
    RedisDsn,
    PostgresDsn,
    Field,
)


class SubModel(BaseModel):
    foo = 'bar'
    apple = 1


class Settings(BaseSettings):
    auth_key: str
    api_key: str = Field(..., env='my_api_key')

    redis_dsn: RedisDsn = 'redis://user:pass@localhost:6379/1'
    pg_dsn: PostgresDsn = 'postgres://user:pass@localhost:5432/foobar'

    special_function: PyObject = 'math.cos'

    # to override domains:
    # export my_prefix_domains='["foo.com", "bar.com"]'
    domains: Set[str] = set()

    # to override more_settings:
    # export my_prefix_more_settings='{"foo": "x", "apple": 1}'
    more_settings: SubModel = SubModel()

    class Config:
        env_prefix = 'my_prefix_'  # defaults to no prefix, i.e. ""
        fields = {
            'auth_key': {
                'env': 'my_auth_key',
            },
            'redis_dsn': {
                'env': ['service_redis_dsn', 'redis_url']
            }
        }


print(Settings().dict())
"""
{
    'auth_key': 'xxx',
    'api_key': 'xxx',
    'redis_dsn': RedisDsn('redis://user:pass@localhost:6379/1',
scheme='redis', user='user', password='pass', host='localhost',
host_type='int_domain', port='6379', path='/1'),
    'pg_dsn': PostgresDsn('postgres://user:pass@localhost:5432/foobar',
scheme='postgres', user='user', password='pass', host='localhost',
host_type='int_domain', port='5432', path='/foobar'),
    'special_function': <built-in function cos>,
    'domains': set(),
    'more_settings': {'foo': 'bar', 'apple': 1},
}
"""
```

(このスクリプトは完成しているので、「そのまま」実行してください)

## 環境変数名

以下のルールは、あるフィールドに対してどの環境変数を読み込むかを決定するために使用されます。

- デフォルトでは、環境変数名は、プレフィックスとフィールド名を連結して作られます。
    - たとえば、上記のspecial_functionを上書きするには、次のようにします。
        ```
        export my_prefix_special_function='foo.bar'
        ```

    - 注1：デフォルトのプレフィックスは空の文字列です。
    - 注2：フィールドエイリアスは、環境変数名を構築する際に無視されます。

- カスタム環境変数名は、2つの方法で設定できます。
    - `Config.fields['field_name']['env']` (上記のauth_keyとredis_dsnを参照)
    - `Field(..., env=...)` (上記 `api_key` 参照)
- カスタム環境変数名を指定する場合は、文字列または文字列のリストのいずれかを指定できます。
    - 文字列のリストを指定する場合は順序が重要で、最初に検出された値が使用されます。
    - 例えば上記の `redis_dsn` の場合、 `service_redis_dsn` が `redis_url` よりも優先されます。

> Warning:
> **v1.0** 以降の pydantic では、設定モデルに入力する環境変数を見つける際にフィールドのエイリアスを考慮しないため、上記のように `env` を代わりに使用してください。
> 
> エイリアスから `env` への移行を支援するために、カスタム `env` 変数名のない設定モデルでエイリアスが使用されると、警告が表示されます。本当にエイリアスを使いたいのであれば、警告を無視するか `env` を設定して警告を出さないようにしてください。

大文字小文字の区別は `Config` でオンにすることができます。

```python
from pydantic import BaseSettings


class Settings(BaseSettings):
    redis_host = 'localhost'

    class Config:
        case_sensitive = True
```

`case_sensitive` が `True` の場合、環境変数の名前はフィールド名と一致しなければなりません（オプションでプレフィックスを付けることができます）ので、この例では `redis_host` は `export redis_host` でしか変更できません。環境変数の名前をすべて大文字にしたい場合は、属性の名前もすべて大文字にしてください。 `Field(..., env=...)` を使えば、環境変数に好きな名前を付けることができます。

> Note:
> Windows では、 python の `os` モジュールは常に環境変数を大文字と小文字を区別しないように扱いますので、case_sensitiveの設定は何の効果もありません。

## 環境変数の値の解析

ほとんどの単純なフィールド・タイプ (`int`, `float`, `str` など) では、環境変数の値はイニシャライザに直接渡された場合と同じように（文字列として）解析されます。

`list`, `set`, `dict`, サブモデルなどの複雑な型は、環境変数の値を JSON エンコードされた文字列として扱うことで、環境から入力されます。

## Dotenv (.env) のサポート

> Note:
> dotenv ファイルの解析には [python-dotenv]() のインストールが必要です。これは `pip install python-dotenv` または `pip install pydantic[dotenv]` のいずれかで行うことができます．

dotenv ファイル（一般的に `.env` という名前）は，プラットフォームに依存しない方法で環境変数を簡単に使用するための一般的なパターンです．

dotenv ファイルはすべての環境変数と同じ一般原則に従っており、以下のような形をしています。

```dockerfile
# ignore comment
ENVIRONMENT="production"
REDIS_ADDRESS=localhost:6379
MEANING_OF_LIFE=42
MY_VAR='Hello world'
```

`.env` ファイルに変数を書き込んだら pydantic は2つの方法で読み込みをサポートします。

1. `BaseSettings` クラスの `Config` で `env_file` （OS のデフォルトエンコーディングを使わない場合は `env_file_encoding` も）を設定する。

    ```python
    class Settings(BaseSettings):
        ...

        class Config:
            env_file = '.env'
            env_file_encoding = 'utf-8'
    ```

2. `_env_file` キーワード引数(必要に応じて `_env_file_encoding` も)を持つ `BaseSettings` 派生クラスをインスタンス化します。
    ```python
    settings = Settings(_env_file='prod.env', _env_file_encoding='utf-8')
    ```

いずれの場合も、渡される引数の値には、現在の作業ディレクトリに対する絶対パスまたは相対パスの、有効なパスまたはファイル名を指定します。ここから先は pydantic が変数の読み込みや検証など、すべてを代行してくれます。

dotenv ファイルを使用している場合でも pydantic は dotenv ファイルと同様に環境変数を読み込みますが、環境変数は常に dotenv ファイルから読み込まれた値よりも優先されます。

インスタンス化（メソッド2）の際に `_env_file` キーワード引数でファイルパスを渡すと `Config` クラスに設定されている値（もしあれば）を上書きします。上記のスニペットを組み合わせて使用した場合、 `prod.env` が読み込まれ `.env` は無視されます。

また、キーワード引数 `override` を使って、インスタンス化キーワード引数に `None` を渡すことで Pydantic に（ `Config` クラスにファイルが設定されていても）一切のファイルをロードしないように指示することもできます（例: `settings = Settings(_env_file=None)` ）。

また python-dotenv を用いてファイルを解析しているため、 `export` などの bash ライクなセマンティクスを用いることができ、 OS や環境によっては dotenv ファイルを `source` で使用することができます．

## Secret のサポート

秘密の値をファイルに配置することは、アプリケーションに機密性の高い設定を提供するための一般的なパターンです。

シークレットファイルは 1つの値のみを含み、ファイル名がキーとして使用されることを除けば dotenv ファイルと同じ原理に従います。シークレットファイルは以下のようになります。

`/var/run/database_password`:

```
super_secret_database_password
```

シークレットファイルができたら pydantic は 2つの方法で読み込みをサポートします。

1. `BaseSettings` クラスの `Config` に `secrets_dir` を設定して、秘密のファイルが保存されているディレクトリを指定する。

    ```python
    class Settings(BaseSettings):
    ...
    database_password: str

    class Config:
        secrets_dir = '/var/run'
    ```

2. `BaseSettings` の派生クラスに `_secrets_dir` キーワード引数を与えてインスタンス化します。
    ```python
    settings = Settings(_secrets_dir='/var/run')
    ```

いずれの場合も、渡される引数の値には、現在の作業ディレクトリに対する絶対的または相対的な、有効なディレクトリを指定できます。存在しないディレクトリの場合は警告が出るだけなので注意してください。ここから先は pydantic が変数の読み込みや検証など、すべてを代行してくれます。

secrets ディレクトリを使用している場合でも、pydantic は dotenv ファイルや環境から環境変数を読み込みますので、 secrets ディレクトリから読み込んだ値よりも dotenv ファイルや環境変数の方が常に優先されます。

インスタンス化（メソッド2）の際に `_secrets_dir` キーワード引数でファイルパスを渡すと `Config` クラスで設定された値（もしあれば）が上書きされます。

### Use Case: Docker Secrets

Docker Secrets は Docker コンテナ内で実行されるアプリケーションに機密性の高い設定を提供するために使用できます。これらのシークレットを pydantic のアプリケーションで使用するためのプロセスは簡単です。Docker でのシークレットの作成、管理、使用に関する詳細は[Dockerの公式ドキュメント]()を参照してください。

まず、Settingsを定義します。

```python
class Settings(BaseSettings):
    my_secret_data: str

    class Config:
        secrets_dir = '/run/secrets'
```

> Note:
> デフォルトでは Docker はターゲットのマウントポイントとして `/run/secrets` を使用します。別の場所を使用したい場合は `Config.secrets_dir` を適宜変更してください。

次に Docker CLI でシークレットを作成します。

```
printf "This is a secret" | docker secret create my_secret_data -
```

最後にアプリケーションを Docker コンテナ内で実行し、新しく作成したシークレットを入力します。

```
docker service create --name pydantic-with-secrets --secret my_secret_data pydantic-app:latest
```

## フィールドの値の優先順位

同一の設定フィールドに複数の値が指定された場合、選択される値は以下のように決定されます（優先順位が高い順）。

1. `Settings` クラスのイニシャライザに渡される引数
2. 環境変数（前述の `my_prefix_special_function` など）
3. dotenv（`.env`）ファイルから読み込まれる変数
4. secrets ディレクトリから読み込まれる変数
5. `Settings` モデルのデフォルトのフィールド値です

## 設定ソースのカスタマイズ

デフォルトの優先順位がニーズに合わない場合は `Settings` モデルの `Config` クラスの `customise_sources` メソッドをオーバーライドすることで変更することができます。

`customise_sources` は引数として3つの callable を取り、任意の数の callable をタプルとして返します。これらの callable は、設定クラスのフィールドへの入力を構築するために呼び出されます。

各 callable は唯一の引数として設定クラスのインスタンスを取り、dict を返す必要があります。

### 優先順位の変更

返された callable の順番は、入力の優先順位を決定します；最初の項目が最も高い優先順位です。

```python
from typing import Tuple
from pydantic import BaseSettings, PostgresDsn
from pydantic.env_settings import SettingsSourceCallable


class Settings(BaseSettings):
    database_dsn: PostgresDsn

    class Config:
        @classmethod
        def customise_sources(
            cls,
            init_settings: SettingsSourceCallable,
            env_settings: SettingsSourceCallable,
            file_secret_settings: SettingsSourceCallable,
        ) -> Tuple[SettingsSourceCallable, ...]:
            return env_settings, init_settings, file_secret_settings


print(Settings(database_dsn='postgres://postgres@localhost:5432/kwargs_db'))
"""
database_dsn=PostgresDsn('postgres://postgres@localhost:5432/env_db',
scheme='postgres', user='postgres', host='localhost', host_type='int_domain',
port='5432', path='/env_db')
"""
```

(このスクリプトは完成しているので、「そのまま」実行してください)

`env_settings` と `init_settings` を入れ替えることで、環境変数が `__init__` の kwargs よりも優先されるようになりました。

### ソースの追加

先に説明したように pydantic には複数の組み込み設定ソースが同梱されています。しかし、場合によっては独自のソースを追加する必要があるかもしれませんが、 `customise_sources` を使えば簡単に追加できます。

```python
import json
from pathlib import Path
from typing import Dict, Any

from pydantic import BaseSettings


def json_config_settings_source(settings: BaseSettings) -> Dict[str, Any]:
    """
    A simple settings source that loads variables from a JSON file
    at the project's root.

    Here we happen to choose to use the `env_file_encoding` from Config
    when reading `config.json`
    """
    encoding = settings.__config__.env_file_encoding
    return json.loads(Path('config.json').read_text(encoding))


class Settings(BaseSettings):
    foobar: str

    class Config:
        env_file_encoding = 'utf-8'

        @classmethod
        def customise_sources(
            cls,
            init_settings,
            env_settings,
            file_secret_settings,
        ):
            return (
                init_settings,
                json_config_settings_source,
                env_settings,
                file_secret_settings,
            )


print(Settings())
#> foobar='spam'
```

(このスクリプトは完成しているので、「そのまま」実行してください)

## ソースを削除する

ソースを無効にしたい場合もあります。

```python
from typing import Tuple

from pydantic import BaseSettings
from pydantic.env_settings import SettingsSourceCallable


class Settings(BaseSettings):
    my_api_key: str

    class Config:
        @classmethod
        def customise_sources(
            cls,
            init_settings: SettingsSourceCallable,
            env_settings: SettingsSourceCallable,
            file_secret_settings: SettingsSourceCallable,
        ) -> Tuple[SettingsSourceCallable, ...]:
            # here we choose to ignore arguments from init_settings
            return env_settings, file_secret_settings


print(Settings(my_api_key='this is ignored'))
#> my_api_key='xxx'
```

(このスクリプトは完成しているので、「そのまま」実行されるはずです。ここでは、環境変数 `my_api_key` を設定する必要があるかもしれません)

`customise_sources` の callables アプローチによりソースの評価は遅延されているため、未使用のソースがパフォーマンスに悪影響を及ぼすことはありません。

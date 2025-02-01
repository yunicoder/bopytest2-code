# section4 組み込みフィクスチャ
## サマリー
- `@pytest.fixture`デコレータを使わずに直接テスト関数の引数として使用できる組み込みフィクスチャがある
  - `tmp_path`,`tmp_path_factory`: 一時ディレクトリに使うフィクスチャ
  - `capsys`: 出力キャプチャに使うフィクスチャ
  - `monkeypatch`: 環境やアプリケーションコードの変更に使うフィクスチャ
- これらのフィクスチャは `pytest --fixtures` で一覧表示可能

```sh
# 利用可能なフィクスチャの一覧表示
cd ch4/
poetry run pytest --fixtures
```

## 4.1 tmp_pathとtmp_path_factoryを使う
`tmp_path` と `tmp_path_factory`は一時ディレクトリの作成に役立つ組み込みフィクスチャ。
- `tmp_path`: 関数スコープのフィクスチャであり、pathlib.Pathオブジェクトを返す。
- `tmp_path_factory`: セッションスコープであり、ディレクトリ取得のために使いメソッド`mktemp()`はpathlib.Pathオブジェクトを返す。

使い方は[ch4/test_tmp.py](../ch4/test_tmp.py)を参照。
```py
def test_tmp_path(tmp_path):
    file = tmp_path / "file.txt"
    file.write_text("Hello")
    assert file.read_text() == "Hello"
```

3章ではtempfileライブラリを使って一時ディレクトリを作成していた([ch4/conftest_from_ch3.py](../ch4/conftest_from_ch3.py))が、`tmp_path_factory`を使うことで[ch4/conftest.py](../ch4/conftest.py)のように書き換えが可能。


似たようなフィクスチャに `tempdir`と`temp_dir_factory`があるが、返り値がpathlibではなく、py.path.localになっているのが大きな点。
(pathlibがpython3.4で追加されたライブラリなので、それ以前によく使われていたフィクスチャ)
- `tempdir`: `tmp_path`の返り値がpy.path.local版
- `temp_dir_factory`: `tmp_path_factory`の返り値がpy.path.local版


## 4.2 capsysを使う
`capsys`は標準出力(stdout)と標準エラー出力(stderr)のテストに役立つ組み込みフィクスチャ。

### capsysを使ったCLI出力のテスト
今回のサンプルプロジェクトにはCLIで実行可能な実装も含まれているため、この出力をテストしたい。
```sh
poetry run cards version
```

テスト方法の1つは`subprocess.run()`を使ってコマンド実行し、その出力がAPI出力と一致するか比較すること。  
該当ファイル: [ch4/test_version.py](../ch4/test_version.py)
```py
def test_version_v1():
    process = subprocess.run(
        ["cards", "version"], capture_output=True, text=True
    )
    output = process.stdout.rstrip()
    assert output == cards.__version__
```

`capsys`を使うことで標準出力と標準エラー出力をキャプチャしてテストすることが可能。
(使い方説明のためにtest_version_v2ではsubprocessを呼び出すのではなく、CLIに出力するメソッドそのものを呼び出している。)  
該当ファイル: [ch4/test_version.py](../ch4/test_version.py)
```py
import cards

def test_version_v2(capsys):
    cards.cli.version()
    output = capsys.readouterr().out.rstrip()
    assert output == cards.__version__
```

#### 余談: CLIテストはCliRunner使うと良い
上記テスト方法はメソッド呼び出ししかテストできていないので、実際のコマンドラインのテストには不十分である。typerライブラリの`CliRunner`を使うことが良いと筆者は述べている。
`subprocess.run`は毎回新しいPythonプロセスを起動するためオーバーヘッドが発生し遅くなり`monkeypatch`などの組み込みも複雑になるため、同一プロセスで実行可能な`CliRunner`がよく使われる。

該当ファイル: [ch4/test_config.py](../ch4/test_config.py)
```py
from typer.testing import CliRunner

def run_cards(*params):
    runner = CliRunner()
    result = runner.invoke(cards.app, params)
    return result.output.rstrip()

def test_run_cards():
    assert run_cards("version") == cards.__version__
```

### capsysを使った一時的な出力キャプチャの無効化
pytestは成功したテストの出力は自動でキャプチャしてしまうので、標準出力に表示されない。  
該当ファイル: [ch4/test_print.py](../ch4/test_print.py)
```sh
poetry run pytest test_print.py::test_normal
```

`capsys.disable()`を使うことで出力を常に表示するように変更可能。
```py
def test_disabled(capsys):
    with capsys.disabled():
        print("\ncapsys disabled print")
```

```sh
poetry run pytest test_print.py::test_disabled
```

ちなみに、`-s`コマンドを使うことで成功するテストの標準出力を表示させることも可能。
```sh
poetry run pytest -s test_print.py::test_normal
```

## 4.3 monkeypatchを使う
`monkeypatch`はアプリケーションコードや環境を変更する組み込みフィクスチャ。

`monkeypatch`を使うことでオブジェクト、ディクショナリ、環境変数、Pythonの検索パス、現在のディレクトリなどを変更することが可能。  
該当ファイル: [ch4/test_config.py](../ch4/test_config.py)
```py
# 環境変数を追加する例
def test_patch_env_var(monkeypatch, tmp_path):
    monkeypatch.setenv("CARDS_DB_DIR", str(tmp_path))
    assert run_cards("config") == str(tmp_path)
```

### 関数の返り値(属性)をモンキーパッチで変更する例
DBの格納場所を返す
```sh
poetry run cards config
```

DBを格納するパスを返す関数`get_path()`の返り値を`tmp_path`に変更してテストしている。
```py
def test_patch_get_path(monkeypatch, tmp_path):
    def fake_get_path():
        return tmp_path

    monkeypatch.setattr(cards.cli, "get_path", fake_get_path)
    assert run_cards("config") == str(tmp_path)
```

#### 余談: 利用ライブラリを変更することも可能 (使用例は同じなのでそれぞれで確認)
`get_path()`の実装を見ると、環境変数がない場合はhomeディレクトリ配下に作成される実装になっている。  
該当ファイル: [cards_proj/src/cards/cli.py](../cards_proj/src/cards/cli.py)
```py
def get_path():
    db_path_env = os.getenv("CARDS_DB_DIR", "")
    if db_path_env:
        db_path = pathlib.Path(db_path_env)
    else:
        db_path = pathlib.Path.home() / "cards_db"
    return db_path
```

したがって、pathlibのhomeをモンキーパッチで変更してテストする。
```py
def test_patch_home(monkeypatch, tmp_path):
    full_cards_dir = tmp_path / "cards_db"

    def fake_home():
        return tmp_path

    monkeypatch.setattr(cards.cli.pathlib.Path, "home", fake_home)
    assert run_cards("config") == str(full_cards_dir)
```

## 4.4 その他の組み込みフィクスチャ
他にもpytestがサポートしている組み込みフィクスチャは存在している。  
ドキュメント: https://docs.pytest.org/en/6.2.x/fixture.html

| フィクスチャ名 | スコープ | 説明 |
| --- | --- | --- |
| **tmpdir** | function | テスト関数ごとに一時的なディレクトリを提供する（py.path.local オブジェクト） |
| **tmpdir_factory** | session | セッション全体で使える一時ディレクトリファクトリを提供する |
| **cache** | session | テスト実行間でデータを共有するためのキャッシュオブジェクト |
| **capsys** | function | sys.stdout と sys.stderr のキャプチャを可能にする |
| **capfd** | function | ファイルディスクリプタ1と2のキャプチャを可能にする |
| **capfdbinary** | function | ファイルディスクリプタ1と2のバイナリキャプチャを可能にする |
| **caplog** | function | ログキャプチャフィクスチャ |
| **monkeypatch** | function | テスト中に一時的な変更を加えるためのツール |
| **pytestconfig** | session | テストセッションの設定オブジェクトへのアクセスを提供 |
| **recwarn** | function | 警告を記録して検査するためのフィクスチャ |
| **doctest_namespace** | session | doctestの名前空間にアクセスするためのフィクスチャ |
| **request** | function | 現在のテストの要求オブジェクトにアクセスし、フィクスチャやマーカーの情報を取得できる |

# section5 パラメータ化
## サマリー
- `@pytest.mark.parametrize()`マーカーを使用することでテスト関数のパラメータ化が可能
- `@pytest.fixure(params=)`マーカーを使用することでフィクスチャのパラメータ化が可能
- `pytest_generate_tests`を使うことでフラグ設定などの複雑なパラメータ設定が可能
- `-k`オプションを使うことで、条件に合ったテストケースのみ実行可能

## 5.1 パラメータ化せずにテストする
同じ関数を複数の開始状態でテストしたい。
純粋に考えるとそれぞれ別のテスト関数を記述して、別々の開始状態を設定することになる。
該当ファイル: [ch5/test_finish.py](../ch5/test_finish.py)

(上記テストはカードの状態が "todo","in pro","finish" である状況から `finish()`メソッドを呼び出して最終状態が"done"になっているかテストしている。)

これらのテスト関数はほとんど同じ処理なので、冗長性を排除してみる。
該当ファイル: [ch5/test_finish_combined.py](../ch5/test_finish_combined.py)

実行してみるとテストケースが3つではなく、1つに集約されてしまっている。
```sh
cd ch5
poetry run pytest -v test_finish_combined.py
```

テストケースが1つに集約されてしまっている場合、以下の問題が生じる。
- テストケース1つが失敗した場合、トレースバックかデバッグをしない限りどのテストケースで失敗したか分からない
- テストケース1つが失敗した場合、その後のテストケースが実行されない


## 5.2 関数をパラメータ化する
`@pytest.mark.parametrize`マーカーを使用することで、設定したパラメータでそれぞれテストケースが実行される。  
該当ファイル: [ch5/test_func_param.py](../ch5/test_func_param.py)
```py
@pytest.mark.parametrize("start_state", ["done", "in prog", "todo"])
def test_finish_simple(cards_db, start_state):
    c = Card("write a book", state=start_state)
    index = cards_db.add_card(c)
    cards_db.finish(index)
    card = cards_db.get_card(index)
    assert card.state == "done"
```

3つのテストケースが実行されていることがわかる。
```sh
poetry run pytest -v test_func_param.py::test_finish_simple
```

## 5.3 フィクスチャをパラメータ化する
`@pytest.fixture(params=)`マーカーを使用することで、フィクスチャのパラメータ化も可能。  
該当ファイル: [ch5/test_fix_param.py](../ch5/test_fix_param.py)
```py
@pytest.fixture(params=["done", "in prog", "todo"])
def start_state(request):
    return request.param

def test_finish(cards_db, start_state):
    c = Card("write a book", state=start_state)
    index = cards_db.add_card(c)
    cards_db.finish(index)
    card = cards_db.get_card(index)
    assert card.state == "done"
```

3つのテストケースが実行されていることがわかる。
```sh
poetry run pytest -v test_fix_param.py::test_finish
```

### 関数のパラメータ化との違い
今回の例だけで見ると「フィクスチャのパラメータ化」は「関数のパラメータ化」と同じように見える。

フィクスチャのパラメータ化の利点は、引数ごとにフィクスチャを実行できることにある。
例えば、セットアップやティアダウンがある場合（異なるDBに接続する,内容の異なるファイルを選択する）に有効な手段である。


## 5.4 pytest_generate_testsを使ってパラメータ化する
`pytest_generate_tests`というフック関数を使うことで、コマンドラインフラグごとに実行するテストケースを変えたりするような様々なテストを行うことが可能。

コマンドラインフラグを使う例
```py
def pytest_addoption(parser):
    parser.addoption(
        "--excessive", action="store_true", default=False,
        help="大量のテストケースを実行する"
    )
    parser.addoption(
        "--quick", action="store_true", default=False,
        help="少数のテストケースだけを実行する"
    )

def pytest_generate_tests(metafunc):
    if "test_value" in metafunc.fixturenames:
        # デフォルトのテストデータ
        default_values = [1, 2, 3, 4, 5]

        # フラグに基づいてテストデータを切り替え
        if metafunc.config.getoption("--excessive"):
            test_values = range(1, 101)  # 1から100までの値をテスト
        elif metafunc.config.getoption("--quick"):
            test_values = [1, 2]  # 最小限のテストケース
        else:
            test_values = default_values  # デフォルト値

        # パラメータ化
        metafunc.parametrize("test_value", test_values)
```

### 余談: その他利点
本には載っていないが`pytest_generate_tests`にはいろんな使い道がある。  
気になる人はドキュメント参照: https://docs.pytest.org/en/stable/example/parametrize.html#paramexamples

動的パラメータ生成(外部ファイルやデータベースからテストケースを読み込む場合など)
```py
def pytest_generate_tests(metafunc):
    if "scenario" in metafunc.fixturenames:
        # 外部ファイルやDBからテストケースを動的に読み込み
        scenarios = load_test_scenarios()
        metafunc.parametrize("scenario", scenarios)
```

条件分岐によるパラメータ制御
```py
def pytest_generate_tests(metafunc):
    if "env" in metafunc.fixturenames:
        if os.getenv("CI_MODE"):
            metafunc.parametrize("env", ["staging", "production"])
        else:
            metafunc.parametrize("env", ["local"])
```

複数パラメータの組み合わせ生成
```py
def pytest_generate_tests(metafunc):
    if "user" in metafunc.fixturenames:
        metafunc.parametrize("user", ["admin", "guest"])
    if "locale" in metafunc.fixturenames:
        metafunc.parametrize("locale", ["ja_JP", "en_US"])
```


## 5.5 キーワードを使ってテストケースを選択する
`-k`オプションを使うことで、キーワードに当てはまるテストケースのみ実行可能。

```sh
cd ch5

# doneが付くのテストケースのみ
poetry run pytest -v -k done

# doneが付くテストケースの内、writeとsimpleは除外する
poetry run pytest -v -k "done and not (write or simple)"
```

# section6 マーカー
## サマリー


## 6.1 組み込みマーカーを使う
pytestには組み込みマーカが用意されている。  
よく使用される以下の4つの組み込みマーカーをこの章で紹介する。

- `@pytest.mark.parametrize()`: 
- `@pytest.mark.skip()`: 
- `@pytest.mark.skipif()`: 
- `@pytest.mark.xfail()`: 

## 6.2 pytest.mark.skipを使ってテストをスキップする
`@pytest.mark.skip()`を付けることで、該当のテストをスキップすることができる。`reason`パラメータに理由を書くことも可能で、未実装のメソッドに対するテストなどに利用する。  
該当ファイル: [ch6/builtins/test_skip.py](../ch6/builtins/test_skip.py)

実行すると`SKIPPED`と表示され、-vオプションを付ければその理由も一緒に表示してくれる。
```sh
cd ch6
poetry run pytest -v builtins/test_skip.py
```

## 6.3 pytest.mark.skipifを使ってテストを条件付きでスキップする
`@pytest.mark.skip()`を付けることで、該当のテストを条件付きでスキップすることができる。OSごとにテストを分けたりするときなどに利用する。  
該当ファイル: [ch6/builtins/test_skipif.py](../ch6/builtins/test_skipif.py)


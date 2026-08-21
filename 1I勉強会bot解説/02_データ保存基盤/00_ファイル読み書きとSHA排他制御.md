# ファイルの読み書きとSHA排他制御（`bot.py` 253〜341行）

対象：`bot.py`の「ローカルファイル ユーティリティ」セクション。ここで作られる`local_get`/`local_put`は、**このファイル全体のほぼすべてのデータ読み書きが経由する、最も重要な共通部品**です。用語は[[../../1I勉強会web解説/01_index_予定管理.md]]の「0. ミニ用語辞典」も参照してください。

## なぜこの仕組みが必要か

コメントにある通り、以前はGitHubのAPIを使ってデータをGitHub上のファイルとして保存していました。GitHubには「今のファイルの中身のハッシュ値（sha）を指定して更新をリクエストし、もし他の誰かが先に更新していて実際のshaと食い違っていたら、更新を拒否して409エラーを返す」という仕組みがあります。これは、2人が同時に同じファイルを保存しようとしたときに、後から保存した方が前の変更を気づかずに丸ごと消してしまう事故（**lost update**、上書き事故）を防ぐための仕組みです。

サーバーを自前のディスクに保存する方式に変えた後も、**同じ考え方をローカルファイルで再現**しています。これが「SHA排他制御」（正確には楽観的排他制御 = optimistic concurrency control）です。

```python
class DataWriteError(Exception):
    pass
```
- 保存に失敗したときに投げる、このファイル専用の例外クラスです。`Exception`を継承しているだけの空のクラスで、「普通のエラーと区別して、呼び出し側で`except DataWriteError:`のように専用にキャッチできるようにする」ためだけに存在します。

## 1. パスとハッシュ値（269〜273行）

```python
def _data_path(filename):
    return os.path.join(DATA_DIR, filename)

def _file_sha(raw_bytes):
    return hashlib.sha256(raw_bytes).hexdigest()
```
- `_data_path(filename)`… `DATA_DIR`（データ保存先ディレクトリ）とファイル名を繋ぎ合わせて、実際のファイルパスを作ります。名前の先頭のアンダースコア`_`は、Pythonの慣習で「このファイルの外からは直接使わない、内部専用の関数」という意味の目印です。
- `_file_sha(raw_bytes)`… バイト列（ファイルの生の中身）から、SHA-256というアルゴリズムでハッシュ値（ダイジェスト、指紋のようなもの）を計算します。ハッシュ値とは、どんなに長いデータでも一定の長さの文字列に変換する計算方法で、**中身が1文字でも変われば、ほぼ確実にハッシュ値も変わる**という性質があります。これを「今読んだときの中身」の目印として使います。

## 2. 読み込み：`local_get`（275〜290行）

```python
def local_get(filename):
    """filename の中身をJSONとして読み込み (data, sha) を返す。存在しなければ (None, None)。"""
    path = _data_path(filename)
    if not os.path.isfile(path):
        return None, None
    with open(path, "rb") as f:
        raw = f.read()
    try:
        data = json.loads(raw.decode("utf-8")) if raw.strip() else None
    except (json.JSONDecodeError, UnicodeDecodeError) as e:
        print(f"[ERROR] {path} の読み込みに失敗しました（...）: {e}")
        data = None
    return data, _file_sha(raw)
```
- ファイルを読み込んで、中身をPythonのデータ（辞書やリスト）に変換したもの`data`と、そのときのハッシュ値`sha`をペアで返す関数です。この`(data, sha)`のペアが、このあと出てくるほぼ全ての「〇〇を読み込む」関数の戻り値の形になります。
- ファイルが無ければ`(None, None)`を返します（＝「まだ何も保存されていない」状態を、エラーにせず正常な状態として扱う）。
- `open(path, "rb")`… `"rb"`は「バイナリモードで読み込む」の意味です。文字列としてではなく生のバイト列として読み、`.decode("utf-8")`で明示的に文字列に変換しています。
- `json.loads(...)`… JSON形式の文字列を、Pythonの辞書やリストに変換します（JSONの読み込み）。`raw.strip()`が空でない場合だけ変換を試み、空ファイルの場合は`None`として扱います。
- もしJSONとして壊れていたり、UTF-8として読めない文字コードだったりした場合、`except`節でエラーを**握りつぶさずに必ずログへ出力**しています。コメントにある通り、これは「原因不明のまま何も読み込まれない」状態になってデバッグが困難になるのを防ぐための配慮です。エラー時も`data = None`として処理は継続します（プログラム全体を止めない）。
- **`sha`はファイルが壊れていても、存在してさえいれば必ず計算されて返ります**（`_file_sha(raw)`は`data`の中身とは無関係に生バイト列から計算される）。これにより、たとえJSONの解析に失敗していても、後で保存するときの排他制御は機能します。

## 3. 書き込み：`_local_write_once`と`local_put`（292〜333行）

```python
def _local_write_once(filename, content_obj, expected_sha=None):
    path = _data_path(filename)
    dirname = os.path.dirname(path)
    if dirname:
        os.makedirs(dirname, exist_ok=True)

    current_sha = None
    if os.path.isfile(path):
        with open(path, "rb") as f:
            current_sha = _file_sha(f.read())

    if expected_sha is not None and current_sha is not None and expected_sha != current_sha:
        return False, current_sha

    encoded = json.dumps(content_obj, ensure_ascii=False, indent=2).encode("utf-8")
    tmp_path = f"{path}.tmp"
    with open(tmp_path, "wb") as f:
        f.write(encoded)
    os.replace(tmp_path, path)
    return True, _file_sha(encoded)
```
- 1回分の書き込み処理です。関数名の先頭に`_`が付いているのは、内部専用（このあとの`local_put`からしか呼ばれない）だという印です。
- `os.makedirs(dirname, exist_ok=True)`… 保存先のフォルダ（例えば`words/`のようなサブフォルダ）が無ければ自動的に作ります。`exist_ok=True`は「既にあってもエラーにしない」設定です。
- **排他チェックの核心部分**：`expected_sha`（呼び出し側が「自分が読み込んだときのハッシュ値」として渡してきた値）と、`current_sha`（今まさにディスクにあるファイルの実際のハッシュ値）を比較します。両方が存在していて、かつ値が食い違っていれば、「自分が読んでから今までの間に、誰か他の人が先に保存していた」と判断し、**書き込まずに`False`（失敗）を返します**。これがGitHubの409エラー相当の仕組みです。
- 一致していれば（または`expected_sha`が指定されていなければ＝チェック不要）、実際に書き込みます。
- `json.dumps(content_obj, ensure_ascii=False, indent=2)`… Pythonのデータを、整形されたJSON文字列に変換します。`ensure_ascii=False`は「日本語をそのまま日本語の文字として保存する」設定（付けないと`あ`のような数値エスケープだらけになり、ファイルを直接開いたときに人間が読めなくなります）。`indent=2`はインデント幅2で整形し、テキストエディタで開いたときに読みやすくするためです。
- **一時ファイルに書いてから`os.replace`ですり替える**、という書き方（`tmp_path`に書き込んでから本来のパスに置き換える）に注目してください。これは「アトミック（不可分）な書き込み」と呼ばれるテクニックです。もし直接本来のファイルに書き込んでいる途中でサーバーがクラッシュしたら、ファイルの中身が中途半端に壊れてしまいます。一時ファイルへの書き込みが終わった後に`os.replace`で名前を入れ替えるやり方なら、置き換え自体は一瞬で完了するため、「完全に新しい内容」か「完全に元の内容のまま」のどちらかにしかならず、中途半端に壊れた状態になることがありません。

```python
def local_put(filename, content_obj, sha=None):
    ok, info = _local_write_once(filename, content_obj, sha)
    if ok:
        return {"content": {"sha": info}}

    ok2, info2 = _local_write_once(filename, content_obj, None)
    if ok2:
        return {"content": {"sha": info2}}

    raise DataWriteError(f"ローカル保存に失敗しました（再試行後も失敗）: {filename}")
```
- こちらが実際に外部（各機能のコード）から呼ばれる、書き込みの窓口です。
- まず`sha`（読み込んだときのハッシュ値）付きで書き込みを試みます。
- もし食い違いで失敗した場合（＝誰かが先に保存していた場合）、**チェック無し（`None`）で強制的にもう1回だけ書き込みを試みます**。つまり「他人の変更を確認せず、無条件で自分の変更を上書きする」形での1回だけの自動リトライです。これは完璧な解決策ではなく、理論上は「他人の変更を意図せず消してしまう」可能性がまだ残っていますが、11人規模の小さな運用で、かつ同じファイルへの同時書き込みが起きる頻度自体が低いという前提のもとで、シンプルさを優先した現実的な妥協です（本格的な対策をするなら、差分をマージする、あるいは何度も読み直して再試行する、といった複雑な仕組みが必要になります）。
- それでも失敗する場合（ディスクの権限エラーなど、ハッシュ値の食い違い以外の根本的な問題）は、`DataWriteError`を投げて呼び出し側にエラーとして伝えます。以前のバージョンでは失敗を無視していたそうですが、今は例外を投げることで「保存に失敗しました」とユーザーに正しく伝えられるようになっています。

## 4. 非同期版：`async_local_get`/`async_local_put`（335〜341行）

```python
async def async_local_get(filename):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, local_get, filename)

async def async_local_put(filename, content_obj, sha=None):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, local_put, filename, content_obj, sha)
```
- `local_get`/`local_put`は普通の（同期的な）関数ですが、Discord Bot側のコード（`/add`コマンドの処理など）は`async def`で書かれた**非同期処理**の中で動いています。非同期処理の中で、ファイルの読み書きのような「終わるまで待たされる（ブロッキングな）」処理をそのまま呼んでしまうと、その間Discord Bot全体の他の処理（他のコマンドへの応答など）まで止まってしまいます。
- `loop.run_in_executor(None, local_get, filename)`は、「`local_get(filename)`を、Discord Botのメインの非同期処理の流れとは別の、裏方のスレッドプールで実行してください」という指示です。こうすることで、ファイルの読み書きをしている間も、Discord Bot本体は他のイベント（他の人からのコマンドなど）に応答し続けられます。
- Flask側（Webからのリクエストを処理する部分）は元々別スレッドで動いているので普通の`local_get`/`local_put`をそのまま使い、Discord Bot側のコマンド処理では、この非同期版を使い分けている、という構造です。

---

次は、この`local_get`/`local_put`を使って実装されている「運用ログ（システムログ）」の仕組みを解説します。 → [[01_運用ログとdiff表示.md]]

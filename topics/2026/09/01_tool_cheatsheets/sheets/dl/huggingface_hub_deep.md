# huggingface_hub 詳細編（応用・トラブル対応）

> 基本編: [huggingface_hub.md](huggingface_hub.md)
> 対象バージョン: `huggingface_hub 1.17.0`（`hf_xet 1.5.0` 同梱、HTTP クライアントは `httpx 0.28`）

## Part 1. 応用・上級機能

### 1.1 リビジョンを固定して再現性を確保する

`main` はリポジトリの更新で中身が変わる。論文用の実験ではコミットハッシュで固定する。

```python
from huggingface_hub import HfApi, hf_hub_download
api = HfApi()

info = api.model_info("MahmoodLab/UNI")
print(info.sha)                                    # main の現在のコミットハッシュ

refs = api.list_repo_refs("MahmoodLab/UNI")
[b.name for b in refs.branches], [t.name for t in refs.tags]

for c in api.list_repo_commits("MahmoodLab/UNI")[:5]:
    print(c.commit_id[:8], c.created_at, c.title)

path = hf_hub_download("MahmoodLab/UNI", "pytorch_model.bin", revision="<40桁のハッシュ>")
```

```bash
hf download MahmoodLab/UNI --revision <40桁のハッシュ>
hf models info MahmoodLab/UNI                      # sha などを表示
```

- 取得したハッシュは wandb の config や実験ノートに記録しておく
- timm からは `hf-hub:MahmoodLab/UNI@<ハッシュ>` で固定できる（[timm_deep.md](timm_deep.md)）

### 1.2 キャッシュの中身

```
$HF_HUB_CACHE（既定 ~/.cache/huggingface/hub）
├── models--MahmoodLab--UNI/
│   ├── blobs/<sha256 など>        # 実体。同じ内容のファイルはリビジョンをまたいで1つ
│   ├── refs/main                  # 中身は main が指すコミットハッシュ
│   └── snapshots/<コミットハッシュ>/
│       └── pytorch_model.bin -> ../../blobs/<...>   # シンボリックリンク
├── datasets--org--name/
└── .locks/models--MahmoodLab--UNI/<etag>.lock       # ダウンロード中の排他ロック
```

| 場所 | 用途 |
|:---|:---|
| `HF_HOME`（既定 `~/.cache/huggingface`） | 以下すべての親 |
| `$HF_HOME/hub` | モデル・データセットのキャッシュ（`HF_HUB_CACHE` で変更） |
| `$HF_HOME/xet` | Xet のチャンクキャッシュ（`HF_XET_CACHE` で変更） |
| `$HF_HOME/token` / `stored_tokens` | 現在のトークン / 保存済みの全トークン |

ダウンロード途中のファイルは `blobs/<...>.incomplete` として残り、再実行時に続きから取得される。

### 1.3 コードからキャッシュを調べる・消す

```python
from huggingface_hub import scan_cache_dir, try_to_load_from_cache, _CACHED_NO_EXIST

info = scan_cache_dir()
print(info.size_on_disk_str)
for repo in sorted(info.repos, key=lambda r: r.size_on_disk, reverse=True)[:10]:
    print(repo.repo_type, repo.repo_id, repo.size_on_disk_str, [r.commit_hash[:8] for r in repo.revisions])

strategy = info.delete_revisions("<コミットハッシュ>", "<別のハッシュ>")
print(strategy.expected_freed_size_str)
strategy.execute()                                  # 実際に削除

p = try_to_load_from_cache("MahmoodLab/UNI", "config.json")   # ネットワークを使わずに確認
if isinstance(p, str):
    print("cached:", p)
elif p is _CACHED_NO_EXIST:
    print("リポジトリ上に存在しないことがキャッシュされている")
else:
    print("not cached")
```

```bash
hf cache ls --filter "size>1GB" --sort size --limit 20
hf cache ls --filter "accessed>30d"                 # 30日以上使っていないもの
hf cache ls --revisions                             # リビジョン単位で表示
hf cache rm <コミットハッシュ> --dry-run            # リビジョン単位の削除も可
```

### 1.4 local_dir の挙動

```bash
hf download MahmoodLab/UNI --local-dir ./models/uni
```

- ファイルはキャッシュではなく `local_dir` に実体として置かれる（シンボリックリンクではない）
- `local_dir/.cache/huggingface/download/*.metadata` にコミットハッシュと etag が記録され、再実行時は変更のあるファイルだけ取得される。このフォルダを消すと全ファイルを再確認する
- 共有ディスクに置いて複数人・複数ジョブで使う、Apptainer に bind する、`local-dir:` で timm に読ませる、といった用途に向く
- 正しく揃っているかは `hf cache verify <repo_id> --local-dir ./models/uni` で確認できる

### 1.5 オフライン運用の詳細

| 方法 | 範囲 |
|:---|:---|
| `HF_HUB_OFFLINE=1` | プロセス全体でネットワークに接続しない。timm / transformers も従う |
| `local_files_only=True` | その呼び出しだけキャッシュのみを使う |
| `revision="<ハッシュ>"` | `snapshots/<ハッシュ>` を直接見る。refs の解決が不要 |

- オンラインでも接続に失敗した場合（ネットワークなし、5xx、プロキシ設定の誤りなど）は、キャッシュにあればそれを返す。ただし接続のタイムアウト（既定 10 秒）を毎回待つので、計算ノードでは `HF_HUB_OFFLINE=1` を明示する方がよい
- オフラインで `revision="main"` を使うには `refs/main` がキャッシュにある必要がある。`local_dir` だけで落とした場合はキャッシュにないので、`local-dir:` やファイルパスで直接読む
- 事前に何が落ちるかは `hf download org/repo --dry-run` で確認できる

### 1.6 高速ダウンロード（Xet）

1.x では `hf_transfer` は使われなくなり、Xet ストレージ（`hf_xet`）が標準。`hf_xet` はこの環境に入っている。

| 変数 | 内容 |
|:---|:---|
| `HF_XET_HIGH_PERFORMANCE=1` | 並列度を上げて高速化（CPU・帯域を多く使う） |
| `HF_HUB_DISABLE_XET=1` | Xet を使わず従来の HTTP で取得する（切り分け用） |
| `HF_XET_CACHE` | Xet のチャンクキャッシュの場所 |

`HF_HUB_ENABLE_HF_TRANSFER=1` は 1.17 では非推奨で、設定すると `FutureWarning` が出るだけで効果はない。

### 1.7 トークンの使い分け

```bash
hf auth login --token "$HF_TOKEN_READ"              # 保存される。名前を付けて複数保存できる
hf auth list                                        # 保存済みトークンの一覧
hf auth switch --token-name my-write-token          # 切り替え
hf auth token                                       # 現在のトークンを表示（ログに残さない）
```

| 優先順位 | 取得元 |
|:---|:---|
| 1 | 関数の `token=` 引数 |
| 2 | 環境変数 `HF_TOKEN` |
| 3 | `$HF_HOME/token`（`HF_TOKEN_PATH` で変更可） |

- fine-grained トークンなら、読み取りのみ・特定リポジトリのみ・gated モデルへのアクセス可否を細かく指定できる。計算ノードには読み取り専用を置く
- `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` で、公開リポジトリへのリクエストにトークンを付けない
- `login()` は既定で `skip_if_logged_in=True`。別のトークンに替えたいときは `hf auth login --force`

### 1.8 コミット操作（ブランチ・タグ・複数ファイルを1コミットで）

```python
from huggingface_hub import HfApi, CommitOperationAdd, CommitOperationDelete, CommitOperationCopy
api = HfApi()
repo = "my-user/my-model"

api.create_branch(repo, branch="exp-dino", exist_ok=True)
api.create_commit(
    repo, revision="exp-dino", commit_message="Update weights",
    operations=[
        CommitOperationAdd("model.safetensors", "out/model.safetensors"),
        CommitOperationAdd("config.json", "out/config.json"),
        CommitOperationDelete("old/"),                              # フォルダごと
        CommitOperationCopy("model.safetensors", "backup/model.safetensors"),
    ],
)
api.create_tag(repo, tag="v1.0", revision="exp-dino", tag_message="paper version")
api.delete_file("tmp.txt", repo)
api.update_repo_settings(repo, private=True)
```

```bash
hf repos branch create my-user/my-model exp-dino
hf repos tag create my-user/my-model v1.0
hf repos tag list my-user/my-model
hf repos delete-files my-user/my-model "logs/*"
hf upload my-user/my-model ./out . --revision exp-dino --commit-message "..." --delete "*.bin"
```

| 大きなデータの上げ方 | 使いどころ |
|:---|:---|
| `upload_folder` / `hf upload` | 1コミットで上げる。途中で失敗すると最初から |
| `upload_large_folder` / `hf upload-large-folder` | 途中から再開できる。複数コミットに分かれる。`repo_type` 必須 |
| `api.super_squash_history(repo)` | 履歴を1コミットにまとめ、古い版のストレージを解放する。元に戻せない |

いずれも外部への公開になる操作なので、公開範囲と中身を確認してから実行する。

### 1.9 データセットリポジトリと hf:// パス

```bash
hf download org/dataset --repo-type dataset --include "train/*.parquet" --local-dir data/
```

```python
from huggingface_hub import HfFileSystem
fs = HfFileSystem()
fs.ls("datasets/org/dataset/train", detail=False)
with fs.open("datasets/org/dataset/train/0000.parquet", "rb") as f:
    df = pd.read_parquet(f)

pd.read_parquet("hf://datasets/org/dataset/train/0000.parquet")   # pandas から直接
```

`HfFileSystem` は fsspec 互換なので、pandas / pyarrow から `hf://` で読める。大量に読むなら先に `hf download` で落とす方が速い。

### 1.10 環境の確認（hf env）

```bash
hf env          # バージョン、HF_HOME、キャッシュの場所、トークンの有無、各環境変数の値
hf version
```

不具合の報告やトラブルの切り分けでは、まずこの出力を確認する。

### 1.11 ミラー・プロキシ

| 変数 | 内容 |
|:---|:---|
| `HF_ENDPOINT` | Hub の URL を変える（社内ミラーなど） |
| `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` | httpx が読む。組織のプロキシ経由で接続する場合 |
| `SSL_CERT_FILE` | 独自の CA 証明書のバンドル（httpx が読む） |
| `HF_HUB_ETAG_TIMEOUT` | メタデータ取得のタイムアウト（秒、既定 10） |
| `HF_HUB_DOWNLOAD_TIMEOUT` | ダウンロードのタイムアウト（秒、既定 10） |

## Part 2. トラブル対応

### 2.1 401 / 403 / GatedRepoError

- **症状**: `GatedRepoError: 403 Client Error ... Access to model MahmoodLab/UNI is restricted`、`401 Unauthorized`、`RepositoryNotFoundError`
- **確認**:
```bash
hf auth whoami                  # どのアカウントで認証されているか
hf auth list                    # 保存済みトークンと現在のトークン
env | grep -E "^HF_(TOKEN|HOME)"   # ジョブ内で HF_TOKEN が別の値になっていないか
hf env
```
```python
from huggingface_hub import model_info
print(model_info("MahmoodLab/UNI").gated)   # "auto" / "manual" / False
```
- **原因**: 申請が未承認、トークンが別アカウント、fine-grained トークンに「gated リポジトリへのアクセス」権限がない、環境変数 `HF_TOKEN` が保存済みトークンより優先されている、ジョブで `HF_HOME` が変わりトークンファイルが見えない
- **対処**: モデルページで承認状態を確認する。fine-grained トークンなら権限を追加するか read トークンを使う。古い `HF_TOKEN` を `unset` する。ジョブと対話環境で `HF_HOME` を揃える（Apptainer では `$HOME` が bind されているか確認）。private リポジトリが見えない場合も、存在を隠すため `RepositoryNotFoundError`（404）になる

### 2.2 計算ノードで「キャッシュにない」と言われる

- **症状**: `LocalEntryNotFoundError: Cannot find the requested files in the disk cache and outgoing traffic has been disabled`
- **確認**:
```bash
echo "$HF_HOME | $HF_HUB_CACHE | $HF_HUB_OFFLINE"
ls "${HF_HUB_CACHE:-${HF_HOME:-$HOME/.cache/huggingface}/hub}" | grep -i uni
hf cache ls --revisions | grep -i uni
```
```python
from huggingface_hub import try_to_load_from_cache
print(try_to_load_from_cache("MahmoodLab/UNI", "pytorch_model.bin"))
```
- **原因**: ログインノードとジョブで `HF_HOME` が違う。`--local-dir` で落としたのでキャッシュに入っていない。必要なファイルの一部（`config.json` など）だけ落としていない。`revision` が違う
- **対処**: 同じ `HF_HOME` でログインノードから `hf download <repo>`（ファイル指定なしで全体）を実行する。`local_dir` を使うなら、コード側もそのパスから直接読む

### 2.3 シンボリックリンクの警告・エラー

- **症状**: `UserWarning: huggingface_hub cache-system uses symlinks by default ... your machine does not support them`、または `OSError: [Errno 95] Operation not supported`
- **確認**:
```python
from huggingface_hub.file_download import are_symlinks_supported
print(are_symlinks_supported())          # キャッシュのディレクトリで判定
```
```bash
df -T "$HF_HOME"                          # ファイルシステムの種類
```
- **原因**: キャッシュを置いたファイルシステムがシンボリックリンクに対応していない（一部の SMB / オブジェクトストレージのマウントなど）
- **対処**: 対応しているディスクに `HF_HOME` を移す。移せない場合、リンクなしでも動く（ファイルを複製するためディスクを多く使う）。明示的にリンクを使わせないなら `HF_HUB_DISABLE_SYMLINKS=1`、警告だけ消すなら `HF_HUB_DISABLE_SYMLINKS_WARNING=1`

### 2.4 ディスク容量・クオータ超過

- **症状**: `OSError: [Errno 28] No space left on device` / `Disk quota exceeded`。ホームのクオータが突然いっぱいになる
- **確認**:
```bash
du -sh ~/.cache/huggingface/{hub,xet} 2>/dev/null
hf cache ls --sort size --limit 10
hf cache prune --dry-run                  # どのリビジョンにも参照されていない古い版
```
- **原因**: 大きなモデルを複数リビジョン保持している。Xet のチャンクキャッシュが別に容量を使っている。`--local-dir` とキャッシュの両方に同じものがある
- **対処**: `HF_HOME` を大容量ディスクに移す（`~/.bashrc` とジョブスクリプトの両方）。`hf cache rm` / `hf cache prune` で整理する。移動は `mv ~/.cache/huggingface /大容量/hf && export HF_HOME=/大容量/hf`（ディレクトリ内のシンボリックリンクは相対パスなので、ディレクトリごと移せば壊れない）

### 2.5 ダウンロードが止まる・ロック待ちのまま進まない

- **症状**: ログに `Still waiting to acquire lock on ...` が10秒ごとに出る、進捗が 0% のまま
- **確認**:
```bash
ls -la ~/.cache/huggingface/hub/.locks/models--MahmoodLab--UNI/
ls -la ~/.cache/huggingface/hub/models--MahmoodLab--UNI/blobs/ | grep incomplete
ps -u $USER -f | grep -E "python|hf "          # 同じファイルを落としている別プロセスがないか
squeue -u $USER                                  # 別のジョブが同時に落としていないか
```
- **原因**: 別のプロセス（アレイジョブの他のタスクなど）が同じファイルをダウンロード中で、正しく待っている。あるいは NFS 上でロックが解放されず残っている
- **対処**: 同時に落としているなら待てばよい。アレイジョブでは事前に1回だけダウンロードしておき、各タスクは `HF_HUB_OFFLINE=1` で読む。どのプロセスも使っていないことを確認できたら、該当の `.lock` と `.incomplete` を消して再実行する

### 2.6 429 Too Many Requests

- **症状**: `HfHubHTTPError: 429 Client Error: Too Many Requests`
- **確認**:
```bash
squeue -u $USER -h | wc -l                # 同時に Hub にアクセスしているジョブの数
```
- **原因**: アレイジョブなどで多数のプロセスが同時にメタデータを問い合わせている。未認証（トークンなし）だと制限が厳しい
- **対処**: 事前ダウンロード + `HF_HUB_OFFLINE=1` にして、ジョブからは Hub にアクセスしない。ログインしてトークン付きでアクセスする。huggingface_hub は 429 / 5xx に対して自動でリトライするが、同時アクセス数自体を減らすのが確実

### 2.7 SSL エラー・プロキシ環境で接続できない

- **症状**: `SSLError: [SSL: CERTIFICATE_VERIFY_FAILED]`、`ConnectError`、`ConnectTimeout`
- **確認**:
```bash
curl -I https://huggingface.co                   # そもそも到達できるか
env | grep -i -E "^(https?|no)_proxy|SSL_CERT"
hf env
```
- **原因**: 組織のプロキシが通信を中継して独自の証明書を使っている、プロキシの環境変数が計算ノードで設定されていない、計算ノードが外部に出られない
- **対処**: `HTTPS_PROXY` を設定する。独自 CA は `SSL_CERT_FILE=/path/to/ca-bundle.pem` で指定する（1.x は httpx を使うので、`REQUESTS_CA_BUNDLE` ではなくこちら）。計算ノードが外に出られないなら事前ダウンロードとオフライン運用（1.5）にする

### 2.8 NFS 上のキャッシュが遅い

- **症状**: 読み込みのたびに数十秒かかる。多数のジョブが同時に起動すると特に遅い
- **確認**:
```bash
df -T "$HF_HOME"
time python -c "from huggingface_hub import hf_hub_download; print(hf_hub_download('MahmoodLab/UNI', 'config.json'))"
HF_HUB_OFFLINE=1 time python -c "..."            # オフラインにすると速くなるならメタデータ問い合わせが原因
```
- **原因**: オンラインのままだと、毎回 Hub にメタデータを問い合わせる。NFS 上のファイルロック・小さなファイルの読み込みが遅い
- **対処**: ジョブでは `HF_HUB_OFFLINE=1`。重みファイルはジョブの開始時にローカルディスク（`$TMPDIR` など）にコピーしてから読む。モデルごとに `hf download --local-dir` で実体を置き、そのパスを直接読む

### 2.9 huggingface-cli が使えない・コマンドが見つからない

- **症状**: `` `huggingface-cli` is deprecated and no longer works. Use `hf` instead. `` と出て終了する（終了コード 1）、または `command not found`。古い記事のコマンド（`huggingface-cli scan-cache` など）が動かない
- **確認**:
```bash
which hf huggingface-cli
hf version
hf --help
```
- **原因**: 1.x では CLI が `hf` に統一された。1.17 の `huggingface-cli` は警告を出して終了するだけで、何も実行しない
- **対処**: 主な対応は下表。`hf repo-files delete` も非推奨で、`hf repos delete-files` に置き換わっている

| 旧 | 新 |
|:---|:---|
| `huggingface-cli login` / `whoami` | `hf auth login` / `hf auth whoami` |
| `huggingface-cli download` / `upload` | `hf download` / `hf upload` |
| `huggingface-cli scan-cache` / `delete-cache` | `hf cache ls` / `hf cache rm`・`hf cache prune` |
| `huggingface-cli repo create` | `hf repos create` |
| `huggingface-cli env` | `hf env` |

### 2.10 ダウンロードしたファイルが壊れている・読み込めない

- **症状**: `safetensors_rust.SafetensorError: Error while deserializing header`、`invalid load key`、`PytorchStreamReader failed reading zip archive`
- **確認**:
```bash
hf cache verify MahmoodLab/UNI                          # チェックサムを確認
hf cache verify MahmoodLab/UNI --local-dir ./models/uni
ls -laL ~/.cache/huggingface/hub/models--MahmoodLab--UNI/snapshots/*/   # リンク先のサイズ
```
- **原因**: 途中で中断したファイル、容量不足で書き込みが途中で終わった、手動でコピーしたときの欠落。Git LFS のポインタファイル（数百バイトのテキスト）を `git clone` で取得している
- **対処**: `hf cache rm model/<repo>` で消して再ダウンロード、または `hf download ... --force-download`。`git clone` ではなく `hf download` を使う

### 2.11 設定した環境変数が効かない

- **症状**: `HF_HOME` を変えたのに古い場所に保存される、`HF_HUB_OFFLINE=1` なのに接続しようとする
- **確認**:
```bash
hf env                                      # 実際に使われている値
```
```python
import huggingface_hub.constants as c
print(c.HF_HOME, c.HF_HUB_CACHE, c.HF_HUB_OFFLINE)
```
- **原因**: 環境変数は `huggingface_hub` の import 時に読まれる。スクリプトの途中で `os.environ` を変えても、先に import したライブラリ（timm、transformers など）には反映されない。Apptainer に環境変数が渡っていない。`HF_HUB_CACHE` が別に設定されていて `HF_HOME` より優先されている
- **対処**: 環境変数はジョブスクリプトで `export` する（Python 内で設定するなら、どのライブラリよりも前に）。Apptainer では `APPTAINERENV_HF_HOME=...` または `--env`

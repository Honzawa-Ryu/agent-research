# huggingface_hub チートシート

> 対象: huggingface_hub 1.x（このマシンの `.venv` は 1.17）。CLI は `hf`（旧 `huggingface-cli` は 1.17 では動かない）
> 公式: https://huggingface.co/docs/huggingface_hub

## 1. 基本概念

| 用語 | 意味 |
|:---|:---|
| リポジトリ | モデル・データセット・Space の単位。`組織名/名前`（例: `MahmoodLab/UNI`） |
| revision | ブランチ名・タグ・コミットハッシュ。既定は `main` |
| gated モデル | 利用規約に同意し、アクセス申請が承認されないとダウンロードできないモデル（UNI、CONCH、Virchow など） |
| トークン | 認証に使う。https://huggingface.co/settings/tokens で発行する。ダウンロードだけなら `read` 権限で足りる |
| キャッシュ | ダウンロードしたファイルの保存場所。既定は `~/.cache/huggingface/hub` |

## 2. ログイン

```bash
hf auth login                  # トークンを入力する（~/.cache/huggingface/token に保存される）
hf auth whoami                 # ログインしているユーザーを確認
hf auth logout
```

```python
from huggingface_hub import login
login()                        # 対話で入力
login(token=os.environ["HF_TOKEN"])
```

- 環境変数 `HF_TOKEN` を設定すれば、ログインしなくても使える
- トークンをコードや git に書かない。`.env` に書いて `.gitignore` に入れる

## 3. ダウンロード

```bash
hf download MahmoodLab/UNI                                 # リポジトリ全体をキャッシュに
hf download MahmoodLab/UNI pytorch_model.bin               # 1ファイルだけ
hf download MahmoodLab/UNI --local-dir ./models/uni        # キャッシュではなく指定のフォルダに
hf download org/repo --include "*.safetensors" --exclude "*.bin"
hf download org/repo --revision v1.0
hf download org/dataset --repo-type dataset
```

```python
from huggingface_hub import hf_hub_download, snapshot_download

path = hf_hub_download("MahmoodLab/UNI", "pytorch_model.bin")          # ローカルのパスが返る
local_dir = snapshot_download("MahmoodLab/UNI", allow_patterns=["*.bin", "*.json"])
local_dir = snapshot_download("org/repo", local_dir="./models/repo", revision="main")
```

- 2回目以降はキャッシュを使うので、再ダウンロードされない
- timm や transformers は内部でこれを使っている。`hf-hub:...` で読み込むときも同じキャッシュを使う

## 4. アップロード

```bash
hf repos create my-model --private
hf upload my-user/my-model ./outputs/ckpt.pt ckpt.pt       # ローカル → リポジトリ上のパス
hf upload my-user/my-model ./outputs .                     # フォルダごと
```

```python
from huggingface_hub import HfApi
api = HfApi()
api.create_repo("my-user/my-model", private=True, exist_ok=True)
api.upload_file(path_or_fileobj="ckpt.pt", path_in_repo="ckpt.pt", repo_id="my-user/my-model")
api.upload_folder(folder_path="outputs", repo_id="my-user/my-model")
```

外部サービスへの公開になるので、中身と公開範囲（private / public）を確認してから実行する。

## 5. 情報を調べる

```python
from huggingface_hub import HfApi, list_repo_files, model_info

list_repo_files("MahmoodLab/UNI")               # ファイル一覧
info = model_info("MahmoodLab/UNI")             # gated かどうか、更新日など
HfApi().list_models(search="pathology", limit=20)
```

## 6. キャッシュの管理

```bash
hf cache ls                   # キャッシュの一覧とサイズ
hf cache rm model/<repo_id>   # 削除（例: model/MahmoodLab/UNI）
hf cache prune --dry-run      # 参照されていない古いリビジョンを確認（--dry-run を外すと削除）
hf cache verify <repo_id>     # 破損の確認
```

## 7. 環境変数

| 変数 | 用途 |
|:---|:---|
| `HF_HOME` | ルートディレクトリ（既定 `~/.cache/huggingface`）。ホームの容量が小さい場合は大きいディスクに向ける |
| `HF_HUB_CACHE` | モデルのキャッシュだけ場所を変える |
| `HF_TOKEN` | トークン |
| `HF_HUB_OFFLINE=1` | ネットワークに接続しない。キャッシュにあるものだけを使う |
| `HF_HUB_DISABLE_PROGRESS_BARS=1` | プログレスバーを消す（ログが見やすくなる） |
| `HF_XET_HIGH_PERFORMANCE=1` | 高速ダウンロード（`hf_xet` を使う。旧 `HF_HUB_ENABLE_HF_TRANSFER` は 1.17 では非推奨で効かない） |

## 8. 計算ノードで使うパターン

計算ノードから外部に接続できない、または遅い場合:

```bash
# 1. ログインノード（またはネットワークのあるノード）で事前にダウンロード
hf download MahmoodLab/UNI

# 2. ジョブではオフラインで実行
export HF_HUB_OFFLINE=1
python extract.py          # キャッシュから読み込まれる
```

- Apptainer では `$HOME` が自動で bind されるので、ホストのキャッシュ（`~/.cache/huggingface`）がそのまま見える
- `HF_HOME` を変えている場合は、そのパスも bind する

## 9. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `401 Unauthorized` | ログインしていない、またはトークンが無効 |
| `403` / `GatedRepoError` | アクセス申請をしていない、またはまだ承認されていない。モデルのページで確認する |
| `LocalEntryNotFoundError` | オフラインモードなのにキャッシュにない。先にダウンロードする |
| ホームの容量が足りない | キャッシュが大きい。`HF_HOME` を移し、`hf cache ls` で不要なものを消す |
| `huggingface-cli` が警告を出して終了する | 1.17 では動かない。`hf` コマンドを使う |
| ダウンロードが途中で止まる | ネットワークの制限。`--local-dir` を指定して再実行すれば途中から再開できる |

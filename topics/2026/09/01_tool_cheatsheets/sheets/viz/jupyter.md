# JupyterLab チートシート

> 対象: JupyterLab 4.x（このマシンの `.venv` は 4.5）
> 公式: https://jupyterlab.readthedocs.io/
> 詳細編（応用・トラブル対応）: [jupyter_deep.md](jupyter_deep.md)

## 1. 計算ノードで起動してつなぐ（全体像）

ログインノードで Jupyter を動かすと、ほかの人の作業を妨げる。計算ノードで起動し、SSH トンネルでつなぐ。

```mermaid
graph LR
    A["手元の VS Code / ブラウザ"] -->|localhost:PORT| B["ログインノード"]
    B -->|"ssh -L PORT:localhost:PORT"| C["計算ノード<br/>jupyter lab (127.0.0.1:PORT)"]
```

## 2. 汎用の手順

```bash
# 1. 計算ノードを確保して起動（sbatch スクリプトにしてもよい）
srun -p <partition> -c 4 --mem=32G --time=8:00:00 --pty bash
hostname                                            # 例: node03
jupyter lab --no-browser --ip=127.0.0.1 --port=8888

# 2. ログインノードの別ターミナルからトンネルを張る
ssh -N -L 8888:localhost:8888 node03

# 3. 手元でアクセス
#    http://localhost:8888/?token=...（起動ログに表示される URL）
```

- `--ip=127.0.0.1` にすると、ノードの外から直接つながらない（トンネル経由だけ）。安全のためこれを使う
- ポートがほかの人と重なったら、別の番号にする
- 手元の PC からログインノードを経由する場合は、`ssh -J login-node -N -L 8888:localhost:8888 node03`

### VS Code から使う

1. トンネルを張った状態で、ノートブックを開く
2. 右上の「カーネルを選択」→「既存の Jupyter Server」
3. `http://localhost:<PORT>/?token=<TOKEN>` を貼り付ける

## 3. 内製テンプレートで起動する（`make jupyter`）

テンプレートのプロジェクトでは、上の手順が `tools/start_jupyter.sh` にまとまっている。

```bash
make jupyter p=<partition> mem=32g        # sbatch で投入される
cat jupyter-server.out                    # 接続手順が表示される
```

`jupyter-server.out` に表示される内容:

```
ssh -N -L <PORT>:localhost:<PORT> -p 49152 <NODE>      # ← ログインノードで実行するトンネル
http://localhost:<PORT>/?token=<TOKEN>                 # ← VS Code に貼る URL
```

| 項目 | 設定 |
|:---|:---|
| ポート | 空いている番号を自動で選ぶ |
| トークン | 毎回ランダムに生成する |
| 計算ノードへの SSH ポート | `49152`（このクラスタの設定） |
| 実行環境 | `apptainer exec --nv ${SIF_PATH}` の中で `.venv` を有効にして起動する |
| リソース | `--gres=gpu:0`、`--cpus-per-task=1`、`--time=168:00:00` |

- **`SIF_PATH` を export してから `make jupyter` を実行する**（例: `export SIF_PATH="$(pwd)/env.sif"`）。スクリプト内では設定されないため、空だとコンテナを起動できない
- GPU は確保しない設定になっている。GPU が必要なら `tools/start_jupyter.sh` の `--gres` を変える
- 最大7日間動き続ける。使い終わったら `scancel <job_id>` で止める
- `jupyter-server.out` はリポジトリの直下に出力される

## 4. カーネル

```bash
jupyter kernelspec list                                          # 登録済みのカーネル
python -m ipykernel install --user --name myproj --display-name "Python (myproj)"   # .venv をカーネルとして登録
jupyter kernelspec uninstall myproj
```

- `.venv` の中から `jupyter lab` を起動すれば、その環境がそのまま既定のカーネルになる
- 別の環境をカーネルにする場合は、その環境に `ipykernel` を入れてから登録する

## 5. ノートブック内で便利な機能

```python
%load_ext autoreload
%autoreload 2               # import したモジュールの変更を自動で反映する（lib/ を編集しながら使うとき）

%matplotlib inline
%time f()                   # 1回の実行時間
%timeit f()                 # 何回か実行して平均
%%time                      # セル全体の実行時間（セルの先頭に書く）
!nvidia-smi                 # シェルコマンド
%env CUDA_VISIBLE_DEVICES=0
%who                        # 定義済みの変数
```

```python
from IPython.display import display, Image, Markdown
display(df.head()); display(Image("fig.png")); display(Markdown("**太字**"))
```

## 6. キーボードショートカット（JupyterLab）

| キー | 動作 |
|:---|:---|
| `Shift + Enter` | 実行して次のセルへ |
| `Ctrl + Enter` | 実行（移動しない） |
| `Esc` / `Enter` | コマンドモード / 編集モード |
| `A` / `B`（コマンドモード） | 上 / 下にセルを追加 |
| `D D`（コマンドモード） | セルを削除 |
| `M` / `Y`（コマンドモード） | Markdown / コードに切り替え |
| `0 0`（コマンドモード） | カーネルを再起動 |
| `Ctrl + Shift + C` | コマンドパレット |

## 7. 運用のコツ

- ノートブックは探索と可視化に使う。何度も実行する処理や長い処理は `.py` にして sbatch で投げる
- 上から順に実行して同じ結果になる状態を保つ。ときどき「カーネル再起動 → 全セル実行」で確認する
- 出力（画像など）が大きいと git の差分が読めなくなる。コミット前に出力を消すか、`nbstripout` を使う
- `jupytext` を使うと、ノートブックを `.py` と同期できる（差分が読みやすくなる）

## 8. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `Address already in use` | ローカルのポートが使われている。`-L` の最初の番号を変える（例: `-L 18888:localhost:8888`） |
| つながらない | ジョブが開始していない（`squeue --me`）、ノード名かポートが違う、トンネルが切れている |
| トークンを聞かれる | 起動ログの URL をトークン付きでそのまま使う |
| カーネルがすぐ死ぬ | メモリ不足。`--mem` を増やすか、大きなデータを小分けにする |
| import できない | 別の環境のカーネルを使っている。`import sys; sys.executable` で確認する |
| GPU が見えない | `--gres` で GPU を確保していない、または Apptainer に `--nv` がない |
| ジョブの時間切れでノートブックが止まった | 保存していない変更は消える。こまめに保存し、長い処理はノートブックの外で実行する |

# JupyterLab 詳細編（応用・トラブル対応）

> 基本編: [jupyter.md](jupyter.md)
> 対象バージョン: JupyterLab 4.5.7、jupyter_server 2.19、ipykernel 7.2、nbconvert 7.17、nbclient 0.10（テンプレートの `.venv` のソースで確認）
> 公式: https://jupyterlab.readthedocs.io/ ・ https://jupyter-server.readthedocs.io/

## Part 1. 応用・上級機能

### 1.1 設定ファイルと置き場所

```bash
jupyter --paths                  # config / data / runtime の探索パス
jupyter --runtime-dir            # 起動中サーバの情報や接続ファイルの置き場
jupyter lab --generate-config    # ~/.jupyter/jupyter_lab_config.py を作る
jupyter lab --help-all | less    # 設定できる項目の一覧（ServerApp.* など）
```

| 設定の書き方 | 例 |
|:---|:---|
| コマンドライン | `jupyter lab --ServerApp.root_dir=/path --MappingKernelManager.cull_idle_timeout=3600` |
| 設定ファイル（`.py`） | `c.ServerApp.root_dir = "/path"` |
| 環境変数 | `JUPYTER_CONFIG_DIR`、`JUPYTER_DATA_DIR`、`JUPYTER_RUNTIME_DIR`、`JUPYTER_TOKEN` |

`~/.jupyter` はホームにあるので、ホストとコンテナ（`$HOME` は既定で bind される）の両方から読まれる。片方用の設定がもう片方にも効く点に注意。

### 1.2 よく使うサーバ設定（jupyter_server 2.x）

| 設定 | 既定 | 内容 |
|:---|:---|:---|
| `ServerApp.root_dir` | 起動したディレクトリ | ファイルブラウザで見える範囲（旧 `notebook_dir`） |
| `ServerApp.preferred_dir` | root_dir | ファイルブラウザを最初に開く場所（root_dir の中） |
| `IdentityProvider.token` | 自動生成 | アクセス用トークン。`ServerApp.token` は 2.0 で非推奨（動くが起動時に警告が出る） |
| `ServerApp.shutdown_no_activity_timeout` | 0（無効） | カーネルがなく操作もない状態が N 秒続いたらサーバを止める |
| `MappingKernelManager.cull_idle_timeout` | 0（無効） | N 秒アイドルのカーネルを止めてメモリを返す |
| `MappingKernelManager.cull_interval` | 300 | 上の判定の間隔（秒） |
| `MappingKernelManager.cull_connected` / `cull_busy` | False | 接続中 / 実行中のカーネルも止める対象にするか |
| `FileContentsManager.delete_to_trash` | True | ファイルブラウザで消したファイルをゴミ箱に移す（容量は空かない） |
| `ContentsManager.allow_hidden` | False | 隠しファイルをファイルブラウザで開けるか |

テンプレートの `make jupyter` は `--time=168:00:00` で確保するので、使い終わって放置するとリソースを 7 日間押さえ続ける。自分用に起動オプションを足す例（テンプレートを変える場合はプロジェクトで相談）:

```bash
jupyter lab ... \
  --ServerApp.shutdown_no_activity_timeout=7200 \
  --MappingKernelManager.cull_idle_timeout=3600
```

サーバが止まると `jupyter lab` のプロセスが終わるので、ジョブも終了する。

### 1.3 トークンの渡し方

| 方法 | 注意 |
|:---|:---|
| `--IdentityProvider.token=...`（テンプレートは旧名の `--ServerApp.token`） | コマンドライン引数は、同じノードのほかのユーザーからも `ps` で見える |
| 環境変数 `JUPYTER_TOKEN` | 引数に出ない |
| 環境変数 `JUPYTER_TOKEN_FILE` | ファイルから読む |
| `jupyter server password` | パスワードをハッシュ化して `~/.jupyter/jupyter_server_config.json` に保存 |

`jupyter-server.out` にもトークン付き URL が書かれる。リポジトリを他人と共有している場合は、このファイルの権限に注意する。

### 1.4 テンプレートの起動スクリプトの流れ

`tools/start_jupyter.sh` は次の順で動く。どこで失敗したかを切り分けるときに使う。

| 順 | 実行場所 | 処理 |
|:---|:---|:---|
| 1 | ホスト（計算ノード） | `python3` で空いているポートを 1 つ取る |
| 2 | ホスト | `hostname -s` とランダムなトークンを作り、接続手順を `jupyter-server.out` に出す |
| 3 | コンテナ | `apptainer exec --nv ${SIF_PATH}` の中で `source .venv/bin/activate` |
| 4 | コンテナ | `jupyter lab --ip=127.0.0.1 --port=$PORT --no-browser --allow-root --ServerApp.token=...` |

- sbatch は投入時の環境変数を引き継ぐ（既定の `--export=ALL`）ので、`SIF_PATH` は投入したシェルで export しておく
- 作業ディレクトリは投入したディレクトリ。`.venv` を相対パスで探すので、リポジトリの直下から `make jupyter` する
- コンテナ内で起動するので、`/data` などを使うなら bind が要る（Apptainer の既定では `$HOME`、カレントディレクトリ、`/tmp` だけ）

### 1.5 SSH の設定（手元 PC から直接トンネルを張る）

`~/.ssh/config`（手元 PC）の例。ホスト名・ユーザー名は自分の環境に置き換える。

```sshconfig
Host login
    HostName <login-node-address>
    User <user>
    ServerAliveInterval 60

Host node*
    User <user>
    Port 49152                  # 計算ノードの sshd（テンプレートの COMPUTE_SSH_PORT）
    ProxyJump login
    ServerAliveInterval 60
```

```bash
ssh -N -L 18888:localhost:<PORT> node03     # 手元の 18888 → 計算ノードの <PORT>
# http://localhost:18888/?token=<TOKEN>
```

`ProxyJump` では、`node03` の名前解決はログインノード側で行われる。`ServerAliveInterval` は、無通信でトンネルが切られるのを防ぐ。

### 1.6 カーネルを詳しく扱う

```bash
jupyter kernelspec list --json                # 各カーネルの kernel.json の中身まで出す
cat ~/.local/share/jupyter/kernels/<name>/kernel.json
python -m ipykernel install --user --name myproj --display-name "Python (myproj)" \
    --env CUDA_VISIBLE_DEVICES 0 --env PYTHONPATH /path/to/src     # 環境変数を埋め込む
```

`kernel.json` の構成（ipykernel 7.2 が作るもの）:

```json
{
  "argv": ["/path/.venv/bin/python", "-m", "ipykernel_launcher", "-f", "{connection_file}"],
  "display_name": "Python (myproj)",
  "language": "python",
  "env": {"CUDA_VISIBLE_DEVICES": "0"},
  "metadata": {"debugger": true},
  "kernel_protocol_version": "5.5"
}
```

- `argv[0]` の Python がカーネルの環境を決める。`--user` で入れた kernelspec はホームにあるので、ホストとコンテナの両方から見える（パスが存在しない側では起動に失敗する）
- `metadata.debugger` が true で、カーネルの環境に debugpy があると、JupyterLab の組み込みデバッガが使える（1.9）

ホストで動かしているサーバから、コンテナ内の Python をカーネルとして使う場合は、`argv` を `apptainer exec` から始める。

```json
"argv": ["apptainer", "exec", "--nv", "/path/env.sif",
         "/path/.venv/bin/python", "-m", "ipykernel_launcher", "-f", "{connection_file}"]
```

接続ファイルは runtime ディレクトリ（ホーム内）に作られ、`$HOME` の bind 経由でコンテナから読める。テンプレートのようにサーバごとコンテナ内で動かす場合は不要。

### 1.7 ノートブックを非対話で実行する（nbconvert）

```bash
jupyter nbconvert --to notebook --execute analysis.ipynb --output analysis.out.ipynb
jupyter nbconvert --to notebook --execute --inplace analysis.ipynb
jupyter nbconvert --to notebook --execute --allow-errors analysis.ipynb        # エラーのセルがあっても最後まで
jupyter nbconvert --to notebook --execute \
    --ExecutePreprocessor.timeout=3600 \
    --ExecutePreprocessor.kernel_name=python3 analysis.ipynb
jupyter nbconvert --to html analysis.ipynb                                    # 共有用の HTML
jupyter nbconvert --to script analysis.ipynb                                  # .py に書き出す
jupyter nbconvert --clear-output --inplace analysis.ipynb                     # 出力を消す
```

| オプション | 既定（nbclient 0.10） | 内容 |
|:---|:---|:---|
| `--ExecutePreprocessor.timeout` | None（無制限） | 1 セルの出力を待つ秒数。`-1` も無制限 |
| `--ExecutePreprocessor.startup_timeout` | 60 | カーネル起動を待つ秒数 |
| `--ExecutePreprocessor.kernel_name` | ノートブックに記録された名前 | 実行するカーネル |

長い実行はノートブックサーバの中ではなく、この形で sbatch スクリプトから流す。papermill（パラメータを渡して実行）はテンプレートの `.venv` に入っていない。

### 1.8 git との付き合い方（nbstripout・jupytext）

どちらもテンプレートの `.venv` には入っていない。使う場合は依存に追加するかプロジェクトで決める。

```bash
# nbstripout: git に入れるときだけ出力を消す（作業中のファイルは消えない）
nbstripout --install              # このリポジトリの .git/config と .git/info/attributes に filter を登録
nbstripout --status               # 登録状態の確認

# jupytext: .ipynb と .py（percent 形式）をペアにする
jupytext --set-formats ipynb,py:percent analysis.ipynb
jupytext --sync analysis.ipynb    # 片方を編集したあとに同期
```

`nbstripout` の filter は `.git/config` に書かれる（git で共有されない）ので、clone した人はそれぞれ `--install` する必要がある。

### 1.9 デバッグ

```python
%xmode Verbose          # トレースバックに変数の値も出す（Minimal / Context / Verbose）
%debug                  # 直前の例外の位置で pdb を開く（例外が出たあとのセルで実行）
%pdb on                 # 以後、例外が出たら自動で pdb に入る
breakpoint()            # コード中に置いて止める
```

| pdb コマンド | 動作 |
|:---|:---|
| `p x` / `pp x` | 値を表示 |
| `u` / `d` | 呼び出し元 / 先のフレームへ |
| `n` / `s` / `c` | 次の行 / 関数の中へ / 続行 |
| `l` / `ll` | 周辺のコード / 関数全体 |
| `q` | 終了 |

組み込みデバッガ: JupyterLab 右上の虫アイコンで有効にし、行番号の左をクリックしてブレークポイントを置く。変数・コールスタックがサイドバーに出る。debugpy はテンプレートの `.venv` に入っている。

### 1.10 プロファイリング

```python
%prun -s cumulative -l 20 f()     # 関数ごとの時間（cProfile）。累積時間順に上位 20
%%prun -D prof.out                # セル全体。ファイルに保存して snakeviz などで見る
%timeit -n 10 -r 3 f()            # 10 回 × 3 セット
```

行単位の `%lprun`（line_profiler）とメモリの `%memit`（memory_profiler）は別パッケージで、`.venv` に入っていない。

メモリ使用量はカーネルの中から確認できる。

```python
import resource, os
print(resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / 1024, "MB")   # このカーネルの最大 RSS（Linux は KB 単位）
```

### 1.11 ウィジェット

ipywidgets 8.1 と jupyterlab_widgets 3.0 は入っているので、追加の設定なしで使える。

```python
import ipywidgets as w
from IPython.display import display

@w.interact(idx=w.IntSlider(0, 0, len(tiles) - 1), alpha=(0.0, 1.0, 0.1))
def show(idx, alpha):
    fig, ax = plt.subplots(); ax.imshow(tiles[idx]); ax.imshow(heat[idx], alpha=alpha); plt.show()

out = w.Output(); display(out)
with out:
    print("ここに出る")               # ループ内で表示を差し替えるとき
```

`%matplotlib widget`（対話的な図）は ipympl が必要で、`.venv` には入っていない。

### 1.12 拡張機能

JupyterLab 4 の拡張は pip で入る「prebuilt」形式が基本で、`jupyter lab build` は通常不要。

```bash
jupyter labextension list          # フロントエンド拡張（enabled / disabled、どのパスから来たか）
jupyter server extension list      # サーバ拡張
jupyter labextension disable <name>
```

拡張はサーバを動かしている環境（テンプレートではコンテナ内の `.venv`）に入れる。ホストの `pip install --user` で入れたものは `~/.local` 経由で混ざることがある（Apptainer 詳細は [../infra/apptainer.md](../infra/apptainer.md)）。

### 1.13 大きなノートブックを軽くする設定

JupyterLab の「Settings → Settings Editor → Notebook」:

| 設定 | 既定 | 内容 |
|:---|:---|:---|
| `maxNumberOutputs` | 50 | 1 セルで表示する出力の最大数。超えた分は折りたたまれる |
| `windowingMode` | `contentVisibility` | 画面外のセルの描画を省く方式。`full` が最も速いが副作用がある |

## Part 2. トラブル対応

### 2.1 ジョブが PENDING のまま起動しない

- **症状**: `make jupyter` のあと `jupyter-server.out` ができない、または空のまま。
- **確認**:
```bash
squeue --me -o "%.10i %.20j %.8T %.10M %.10l %R"      # 状態と理由（最後の列）
scontrol show partition <partition> | grep -E "MaxTime|State"
```
- **原因**: リソース待ち（`Resources`、`Priority`）、または要求時間がパーティションの上限を超えている（`PartitionTimeLimit`）。テンプレートは `--time=168:00:00` 固定。
- **対処**: `PartitionTimeLimit` なら、上限の短いパーティションでは使えない。`sbatch --time=... tools/start_jupyter.sh` のようにコマンドラインで上書きする（コマンドラインの指定は `#SBATCH` 行より優先される）。待ちが長いだけなら、パーティションを変えるか待つ。

### 2.2 `SIF_PATH` 未設定でサーバが起動しない

- **症状**: ジョブはすぐ終わる。`jupyter-server.out` に接続手順は出ているが、その後に apptainer のエラーが出ている。
- **確認**:
```bash
tail -n 20 jupyter-server.out
sacct -j <jobid> --format=JobID,State,ExitCode,Elapsed
echo "$SIF_PATH"; ls -l "$SIF_PATH"        # 投入したシェルで
```
- **原因**: `start_jupyter.sh` は `${SIF_PATH}` を投入時の環境から受け取る。空だと `apptainer exec --nv bash -c ...` となり、`bash` をイメージとして開こうとして失敗する。相対パスで export した場合は、投入ディレクトリ以外では見つからない。
- **対処**: `export SIF_PATH="$(pwd)/env.sif"` のように絶対パスで設定してから `make jupyter` をやり直す。

### 2.3 `.venv` が見つからない・jupyter コマンドがない

- **症状**: `.venv/bin/activate: No such file or directory`、`jupyter: command not found`。
- **確認**:
```bash
grep -n "activate\|command not found" jupyter-server.out
ls .venv/bin/jupyter
apptainer exec "$SIF_PATH" bash -c "source .venv/bin/activate && which python jupyter"
```
- **原因**: リポジトリ直下以外から投入した（`.venv` を相対パスで探す）、`.venv` をまだ作っていない、または `.venv` がコンテナ外の Python で作られていてコンテナ内で動かない。
- **対処**: リポジトリ直下で `make jupyter` を実行する。`.venv` はコンテナの中で作る（テンプレートでは `make uv_sync p=<partition>`。[../infra/research_template.md](../infra/research_template.md)、[../infra/uv.md](../infra/uv.md) 参照）。

### 2.4 トンネルがつながらない（ポート 49152・ProxyJump）

- **症状**: `ssh -N -L ...` が `Connection refused` / タイムアウトになる。または ssh は通るが、ブラウザで `localhost:<PORT>` が開けない。
- **確認**:
```bash
ssh -p 49152 <NODE> hostname                   # ログインノードから計算ノードの sshd に届くか
squeue --me -o "%.10i %.8T %N"                  # ジョブが RUNNING で、ノード名が合っているか
grep -m1 "http://localhost" jupyter-server.out  # PORT とトークン
ssh -p 49152 <NODE> "ss -ltn | grep <PORT>"      # 計算ノードでサーバが待ち受けているか
```
- **原因**: `-p 49152` を付けていない（計算ノードの 22 番は使えない）、ジョブが別のノードで動いている／もう終わっている、ポート番号の写し間違い。手元 PC からの場合は `ProxyJump` の設定がない。
- **対処**: `jupyter-server.out` の STEP 1 のコマンドをそのまま使う。手元から直接つなぐなら 1.5 の `~/.ssh/config` を使う。サーバは `127.0.0.1` で待ち受けているので、`-L` の転送先は `localhost` にする（ノード名にしない）。

### 2.5 ログインノードで `Address already in use`

- **症状**: `bind [127.0.0.1]:<PORT>: Address already in use` / `Could not request local forwarding`。
- **確認**:
```bash
ss -ltn | grep <PORT>               # ログインノードでそのポートが使われているか
pgrep -u "$USER" -af "ssh -N -L"     # 自分の古いトンネルが残っていないか
```
- **原因**: ログインノードは共有なので、同じ番号をほかの人か自分の古いトンネルが使っている。サーバ側（計算ノード）のポートとは関係ない。
- **対処**: 自分の古いトンネルなら `kill <pid>`。そうでなければ `-L` の最初の番号だけ変える（`-L 23456:localhost:<PORT>`）。VS Code に貼る URL の番号も最初の番号に合わせる。

### 2.6 カーネルが突然死ぬ（OOM）

- **症状**: 「The kernel appears to have died. It will restart automatically.」大きな配列やスライドを読み込んだときに起きる。サーバ自体は動き続けることが多い。
- **確認**:
```bash
sstat -j <jobid>.batch --format=JobID,MaxRSS              # 実行中のジョブのメモリ最大値
sacct -j <jobid> --format=JobID,State,MaxRSS,ReqMem
grep -i "oom" jupyter-server.out                          # oom-kill のメッセージが出ていることがある
```
```python
import resource; print(resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / 1024, "MB")
```
- **原因**: ジョブの `--mem`（`make jupyter mem=...`）を超えた。サーバ・すべてのカーネル・ターミナルが同じ上限を共有するので、開いたままのほかのノートブックのカーネルも数に入る。
- **対処**: 使っていないカーネルを止める（「Running Terminals and Kernels」タブ、または 1.2 の `cull_idle_timeout`）。`mem` を増やして起動し直す。データは必要な列・領域だけ読む。大きな処理はノートブックの外で sbatch に回す。

### 2.7 カーネルが見つからない・違う Python で動いている

- **症状**: 入れたはずのパッケージが import できない。カーネル一覧に想定外の名前がある。VS Code で選んだカーネルが起動しない。
- **確認**:
```python
import sys; print(sys.executable, sys.prefix)
```
```bash
jupyter kernelspec list --json | grep -A3 '"argv"'
```
- **原因**: `--user` で入れた kernelspec はホーム（`~/.local/share/jupyter/kernels`）にあり、ホストとコンテナ、別プロジェクトで共有される。別の `.venv` を指したカーネルや、コンテナ内に存在しないパスを指したカーネルを選んでいる。VS Code では、ローカルのインタプリタを選んでしまっている場合もある。
- **対処**: VS Code では「既存の Jupyter Server」を選んだうえで、そのサーバの `Python 3 (ipykernel)` を選ぶ。不要な kernelspec は `jupyter kernelspec uninstall <name>`。

### 2.8 403 / `'_xsrf' argument missing` / トークンを何度も聞かれる

- **症状**: 保存や実行で `403 Forbidden`、`'_xsrf' argument missing from POST`。トークン入力画面に戻される。
- **確認**:
```bash
grep -m1 "http://localhost" jupyter-server.out    # 今のサーバのトークン
squeue --me                                        # サーバのジョブがまだ動いているか
```
- **原因**: サーバを起動し直してトークンが変わったのに、古いタブや VS Code に登録した古い URL を使っている。同じ `localhost:<番号>` で前のサーバの cookie が残っている。
- **対処**: 新しいトークン付き URL を開き直す。ブラウザなら localhost の cookie を消すか、プライベートウィンドウで開く。VS Code では古いサーバの登録を消して、新しい URL を登録する。

### 2.9 接続が頻繁に切れる・出力が消える

- **症状**: しばらく放置すると「Connection lost」になる。長いセルの実行中に再接続すると、途中の出力が表示されない。
- **確認**:
```bash
pgrep -u "$USER" -af "ssh -N -L"     # トンネルのプロセスが生きているか
squeue --me                           # サーバのジョブが生きているか
```
- **原因**: ネットワークの無通信タイムアウトで SSH トンネルが切られた（Jupyter 自体は 30 秒ごとに WebSocket の ping を送る）。切断中にカーネルが出した出力は、再接続してもノートブックに戻らない。
- **対処**: `~/.ssh/config` に `ServerAliveInterval 60`（1.5）。トンネルを張り直せばカーネルは動き続けている。長い計算の結果はファイルに保存するか、ノートブックの外で実行する。

### 2.10 ジョブの時間切れでサーバが止まった

- **症状**: 突然つながらなくなり、ジョブが一覧から消えている。
- **確認**:
```bash
sacct -j <jobid> --format=JobID,State,Elapsed,Timelimit
squeue --me -o "%.10i %.20j %.10L"        # 実行中のジョブの残り時間
```
- **原因**: `State` が `TIMEOUT`。テンプレートは最大 168 時間。
- **対処**: 保存していない変更は戻らない。自動保存（既定で有効、120 秒ごと）より新しい編集は失われるので、区切りで `Ctrl + S`。残り時間を見て、切れる前に起動し直す。

### 2.11 ノートブックが巨大で開くのが遅い

- **症状**: `.ipynb` が数十〜数百 MB あり、開くのに時間がかかる・ブラウザが固まる。git の差分が読めない。
- **確認**:
```bash
ls -lh analysis.ipynb
python - <<'EOF'
import json, nbformat
nb = nbformat.read("analysis.ipynb", as_version=4)
top = sorted(((len(json.dumps(c.get("outputs", []))), i) for i, c in enumerate(nb.cells)), reverse=True)[:5]
for size, i in top: print(f"cell {i}: {size/1e6:.1f} MB")
EOF
```
- **原因**: 高解像度の画像、大きな DataFrame の表示、ループ内の大量の `print` が出力として保存されている。
- **対処**: `jupyter nbconvert --clear-output --inplace analysis.ipynb` で出力を消してから開く。画像は `savefig` でファイルに保存して、ノートブックには縮小版だけ表示する。ループの進捗は `tqdm` などで 1 行にまとめる。

### 2.12 ファイルブラウザに見たいディレクトリが出ない・消しても容量が空かない

- **症状**: `/data` などがファイルブラウザに出ない。`... is outside root contents directory` と出る。大きなファイルをファイルブラウザで消したのに、容量が減らない。
- **確認**:
```python
import os; print(os.getcwd())          # root_dir（テンプレートではリポジトリ直下）
os.listdir("/data")                    # コードからは見えるか（bind されているか）
```
```bash
du -sh ~/.local/share/Trash 2>/dev/null
```
- **原因**: ファイルブラウザは `root_dir` の外を表示しない。コンテナに bind していないパスはコードからも見えない。削除は既定でゴミ箱への移動（`delete_to_trash=True`）。
- **対処**: 見たいディレクトリへのシンボリックリンクを root_dir の中に作る（リンクをたどって表示できる）。bind されていないなら Apptainer の `-B` を足す必要がある。容量を空けるならターミナルから `rm` するか、ゴミ箱を空にする。

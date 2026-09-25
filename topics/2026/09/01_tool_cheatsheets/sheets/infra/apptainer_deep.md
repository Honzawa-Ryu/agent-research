# Apptainer 詳細編（応用・トラブル対応）

> 基本編: [apptainer.md](apptainer.md)
> 対象バージョン: Apptainer 1.4 系（このマシンは `apptainer 1.4.5`）
> 公式: https://apptainer.org/docs/user/latest/

## Part 1. 応用・上級機能

### 1.1 このマシンの構成を知る

```bash
apptainer --version
grep -vE '^\s*(#|$)' /etc/apptainer/apptainer.conf | less   # 管理者の設定（読むだけ）
ls /usr/libexec/apptainer/bin/                              # starter-suid の有無
grep "^$USER:" /etc/subuid /etc/subgid                      # --fakeroot 用の ID 割り当て
nvidia-smi --query-gpu=driver_version,name --format=csv     # ホストのドライバ
```

2026-09-25 時点の主な値:

| 項目 | 値 | 影響 |
|:---|:---|:---|
| `starter-suid` | なし | setuid を使わないユーザー名前空間モードで動く。SIF は squashfuse、overlay は fuse-overlayfs / fuse2fs でマウントされる |
| `/etc/subuid` | 自分のエントリあり | `--fakeroot` でビルドできる見込みがある |
| `sessiondir max size` | 64 (MiB) | `--writable-tmpfs` の書き込み層は 64MiB まで（2.11節） |
| `mount home` / `mount tmp` | yes | `$HOME` と `/tmp` は自動で bind される |
| `bind path` | `/etc/localtime`, `/etc/hosts` | 常に bind される |
| `config resolv_conf` | yes | ホストの `/etc/resolv.conf` が使われる（DNS はホストと同じ） |
| `enable underlay` | yes | コンテナ内に存在しないパスにも bind できる |
| `use nvidia-container-cli` | no | `--nv` はライブラリ一覧（nvliblist.conf）に基づく bind 方式 |
| NVIDIA ドライバ | 590.48.01（RTX A6000） | コンテナ内の CUDA ランタイムはこのドライバが対応する版まで |

### 1.2 定義ファイルの追加セクション

| セクション | 実行されるタイミング | 実行される場所 | 用途 |
|:---|:---|:---|:---|
| `%arguments` | ビルド時 | — | `{{ VAR }}` の既定値を定義する |
| `%setup` | ビルド時、`%files` より前 | **ホスト側** | コンテナのルートは `${APPTAINER_ROOTFS}`。ホストを壊しうるので最小限に |
| `%test` | ビルドの最後、`apptainer test` 実行時 | コンテナ内 | 動作確認。失敗するとビルドも失敗する |
| `%startscript` | `apptainer instance start` 時 | コンテナ内 | 常駐プロセスの起動（1.6節） |
| `%help` | `apptainer run-help` 時 | — | 使い方の説明 |
| `%apprun <name>` など | `--app <name>` 指定時 | コンテナ内 | 1つのイメージに複数の入口を持たせる（SCIF） |

```singularity
Bootstrap: docker
From: nvidia/cuda:{{ CUDA }}-cudnn-devel-ubuntu24.04

%arguments
    CUDA=12.8.1

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get update && apt-get install -y --no-install-recommends python3 curl \
        && rm -rf /var/lib/apt/lists/*
    curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/usr/local/bin sh

%test
    uv --version
    python3 -c "import sys; assert sys.version_info >= (3, 12)"
```

```bash
apptainer build --build-arg CUDA=12.4.1 env.sif env.def    # {{ CUDA }} を上書き
apptainer build --notest env.sif env.def                   # %test を飛ばす
apptainer test env.sif                                     # 既存イメージで %test だけ実行
```

### 1.3 マルチステージビルド

ビルド用の重いイメージで作ったものを、実行用の軽いイメージにコピーする。

```singularity
Bootstrap: docker
From: nvidia/cuda:12.8.1-devel-ubuntu24.04
Stage: build

%post
    apt-get update && apt-get install -y build-essential git
    git clone https://github.com/org/tool.git /src && cd /src && make

Bootstrap: docker
From: nvidia/cuda:12.8.1-runtime-ubuntu24.04
Stage: final

%files from build
    /src/bin/tool /usr/local/bin/tool
```

- 最後のステージが SIF になる。`%files from <stage>` で前のステージからコピーする
- devel イメージ（コンパイラ入り）と runtime イメージで数 GB の差が出る

### 1.4 ビルドの追加オプション

| オプション | 説明 |
|:---|:---|
| `--force` / `-F` | 既存の SIF を上書きする |
| `--build-arg K=V` / `--build-arg-file f` | `{{ K }}` を置き換える |
| `--section post,test` | 指定したセクションだけ実行する（sandbox の更新と組み合わせる） |
| `--update` / `-u` | 既存の sandbox に対して def を再実行する |
| `--notest` / `-T` | `%test` を飛ばす |
| `--fix-perms` | 所有者に rwX を付ける（あとで消せないファイルを防ぐ） |
| `--bind` | ビルド中に `%post` からホストのパスを見せる |
| `--nv` | ビルド中にホストの NVIDIA ライブラリを入れる（GPU を使うビルドのみ） |
| `--disable-cache` | キャッシュを使わず、作りもしない |
| `--arch arm64` | 別アーキテクチャ用のイメージを取得する（docker ブートストラップ時） |

sandbox で試行錯誤してから固める流れ:

```bash
apptainer build --sandbox dev/ env.def
apptainer shell --writable --fakeroot dev/     # 中でパッケージを試す
# 試した手順を env.def に書き戻す
apptainer build --force env.sif env.def         # sandbox を直接固めるより再現性が高い
```

### 1.5 Docker / OCI イメージからの変換

| ソース | 例 | 使いどころ |
|:---|:---|:---|
| `docker://` | `docker://nvcr.io/nvidia/pytorch:25.01-py3` | レジストリから直接 |
| `docker-archive://` | `docker-archive://img.tar` | `docker save` したファイル |
| `oci-archive://` | `oci-archive://img.oci.tar` | OCI 形式のアーカイブ |
| `docker-daemon://` | `docker-daemon://myimg:latest` | 同じマシンで Docker が動いている場合 |
| `localimage` | `Bootstrap: localimage` / `From: base.sif` | 手元の SIF をベースにする |

```bash
# Docker が使える別のマシンで
docker save myimg:latest -o myimg.tar
# このマシンで
apptainer build myimg.sif docker-archive://myimg.tar

# 認証が必要なレジストリ（NGC など）
apptainer registry login --username '$oauthtoken' docker://nvcr.io
apptainer build --docker-login env.sif docker://private/repo:tag   # 対話でログイン
```

- Dockerfile の `USER`、`ENTRYPOINT`、`CMD` は `%runscript` に変換されるが、`USER` は無視される（Apptainer は常に実行ユーザーで動く）
- Dockerfile の `ENV` は `%environment` 相当に取り込まれる

### 1.6 インスタンス（常駐プロセス）

データベースや推論サーバなど、ジョブの中で裏で動かし続けたいものに使う。

```bash
apptainer instance start --nv -B /data es.sif es01    # %startscript が起動する
apptainer instance list
apptainer exec instance://es01 curl -s localhost:9200 # 起動中のインスタンスでコマンドを実行
apptainer instance stats es01                          # CPU・メモリ使用量
apptainer instance stop es01
apptainer instance stop --all
```

- インスタンスは Slurm ジョブの終了とともに止まる。ジョブの最後で明示的に `stop` しておくとログが残る
- ネットワークはホストと共有なので、ポートの衝突に注意する（同じノードで同じポートを使うジョブが2本あると起動しない）

### 1.7 overlay（書き込み層）

| 方式 | 作り方 | 保存 | 同時利用 |
|:---|:---|:---|:---|
| `--writable-tmpfs` | 不要 | 終了時に消える | 可。このマシンでは 64MiB まで |
| ext3 イメージ | `apptainer overlay create --size 2048 ov.img` | 残る | 書き込みは1プロセスのみ。読み取りは `:ro` で複数可 |
| SIF に埋め込み | `apptainer overlay create --size 2048 env.sif` | SIF に残る | 同上 |
| ディレクトリ | `mkdir ov/` | 残る | 書き込みは1プロセスのみ |

```bash
apptainer overlay create --size 4096 --sparse ov.img   # 実際に使った分だけディスクを消費
apptainer exec --overlay ov.img env.sif pip install --user foo
apptainer exec --overlay ov.img:ro env.sif python a.py # 読み取り専用で複数ジョブから使う
apptainer overlay create --fakeroot --size 2048 ov.img # --fakeroot と組み合わせて使う場合
```

- アレイジョブから同じ overlay を書き込みモードで開くと、2本目以降が失敗する。書き込みは1回だけにして、後は `:ro` で使う
- Python パッケージの追加は overlay より `.venv`（ホスト側、[uv.md](uv.md)）のほうが管理しやすい

### 1.8 bind の詳細

```bash
apptainer exec -B /tank/honzawa/data:/data:ro env.sif ls /data
apptainer exec --mount type=bind,src=/path/with,comma,dst=/data,ro env.sif ls /data  # , や : を含むパス
apptainer exec -B /a -B /b env.sif ...                   # 複数回指定できる
apptainer exec --no-mount home,tmp env.sif ...           # 既定の bind を外す
apptainer exec --no-mount /etc/hosts env.sif ...         # bind path の個別の項目を外す
apptainer exec -H /tank/honzawa/ctrhome env.sif ...      # 別のディレクトリを $HOME にする
```

既定で bind されるもの:

| パス | 備考 |
|:---|:---|
| `$HOME` | `--no-home` / `--no-mount home` で外せる |
| カレントディレクトリ | `--no-mount cwd` で外せる。`$HOME` の外でも bind される |
| `/tmp` | `--no-mount tmp`。`-C` / `--containall` で空の tmp になる |
| `/proc`, `/sys`, `/dev` | |
| `/etc/localtime`, `/etc/hosts`, `/etc/resolv.conf` | 管理者の設定による |
| `/etc/passwd`, `/etc/group` | 自分のユーザー情報が追加される |

- シンボリックリンクは**リンク先のパス**が bind されていないと辿れない（2.9節）
- 後に指定した bind が前の bind の中に重なる場合、後のものが上に見える
- `APPTAINER_BIND="/data,/scratch"` を `.bashrc` に書くと全実行に効く。意図しない bind が入る原因にもなる

### 1.9 環境変数の優先順位

優先度が低い順:

| 順位 | 設定元 | 備考 |
|:---:|:---|:---|
| 1 | ホストの環境変数 | `--cleanenv` で渡さない。ホストの `PATH` は引き継がれない |
| 2 | イメージの `%environment`（と Docker の `ENV`） | ホストの同名変数を上書きする |
| 3 | `APPTAINERENV_<NAME>` / `--env-file` / `--env` | イメージの値を上書きする |

```bash
apptainer exec --cleanenv --env WANDB_MODE=offline env.sif env | sort   # 何が入ったかを確認
APPTAINERENV_PREPEND_PATH=/opt/tools/bin apptainer exec env.sif which tool # PATH の先頭に追加
APPTAINERENV_APPEND_PATH=/opt/extra/bin  apptainer exec env.sif ...        # PATH の末尾に追加
apptainer inspect --environment env.sif                                     # イメージ側の定義
apptainer exec --no-eval --env 'X=$HOME' env.sif printenv X                 # 値の $ を展開しない
```

### 1.10 GPU オプションの違い

| オプション | 仕組み | このマシン |
|:---|:---|:---|
| `--nv` | ホストのドライバのライブラリと `nvidia-smi` を `/.singularity.d/libs` などに bind する | 使う |
| `--nvccli` | `nvidia-container-cli` に GPU の設定を任せる | 設定で無効（`use nvidia-container-cli = no`） |
| `--rocm` | AMD GPU 用（実験的） | 対象外 |

```bash
apptainer exec --nv env.sif nvidia-smi                    # ドライバの版がホストと同じか
apptainer exec --nv env.sif ls /.singularity.d/libs | head # bind された libcuda など
apptainer exec --nv env.sif python -c "import torch; print(torch.version.cuda, torch.cuda.is_available())"
```

- コンテナ内の CUDA ランタイム（torch の `+cu128` など）は、ホストのドライバが対応する CUDA 以下でなければならない。ドライバの対応版は `nvidia-smi` の右上に出る
- ドライバ本体（`libcuda.so`）はホストから入るので、イメージに入れてはいけない
- `CUDA_VISIBLE_DEVICES` はホストからそのまま引き継がれる

### 1.11 キャッシュの管理

```bash
apptainer cache list -v                      # 種類ごとの件数と容量
apptainer cache clean --dry-run              # 何が消えるかだけ表示
apptainer cache clean --days 30 --force      # 30日より古いものを確認なしで削除
apptainer cache clean --type blob --force    # OCI のレイヤーだけ削除
```

| 環境変数 | 既定 | 用途 |
|:---|:---|:---|
| `APPTAINER_CACHEDIR` | `~/.apptainer/cache` | 取得したレイヤーの保存先 |
| `APPTAINER_TMPDIR` | `/tmp` | ビルド中の展開先。イメージの数倍の容量を使う |

### 1.12 署名と検証

```bash
apptainer key newpair                  # 鍵を作る（~/.apptainer/keys）
apptainer sign env.sif                 # SIF に署名を追加
apptainer verify env.sif               # 署名と中身が一致するか
apptainer sif list env.sif             # SIF 内のデータオブジェクト一覧（署名や overlay の有無）
apptainer sif header env.sif           # UUID などのヘッダ情報
sha256sum env.sif                      # 共有イメージが差し替えられていないかの照合
```

共有のイメージを使うときは、`apptainer inspect` のラベル、`sif header` の UUID、`sha256sum` を記録しておくと、差し替えられたときに気づける。

## Part 2. トラブル対応

### 2.1 ビルド中に容量不足で失敗する

- **症状**: `no space left on device`、`mksquashfs` のエラー、`write /tmp/build-temp-...: no space left`
- **確認**:
```bash
df -h /tmp "${APPTAINER_TMPDIR:-/tmp}" ~
du -sh ~/.apptainer/cache
apptainer cache list -v
```
- **原因**: ビルド中はベースイメージの展開、`%post` の生成物、squashfs への圧縮が `APPTAINER_TMPDIR` に置かれ、最終 SIF の数倍を使う。キャッシュは `$HOME` にたまる
- **対処**:
```bash
export APPTAINER_TMPDIR=/scratch/$USER/apptainer_tmp   # 容量のあるローカルディスク
export APPTAINER_CACHEDIR=/tank/honzawa/apptainer_cache
mkdir -p "$APPTAINER_TMPDIR" "$APPTAINER_CACHEDIR"
apptainer cache clean --days 30 --force
```
`%post` の最後で `apt-get clean && rm -rf /var/lib/apt/lists/*` や `pip cache purge` をしてイメージ自体も小さくする

### 2.2 apt-get が止まったまま進まない

- **症状**: ビルドログが `Configuring tzdata` や `Please select the geographic area` で止まる
- **確認**:
```bash
grep -nE "DEBIAN_FRONTEND|TZ=" env.def
```
- **原因**: tzdata や keyboard-configuration が対話入力を待っている。`%environment` の変数は `%post` には効かない
- **対処**: `%post` の先頭に書く

```singularity
%post
    export DEBIAN_FRONTEND=noninteractive
    export TZ=Asia/Tokyo
    ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```

### 2.3 --fakeroot でビルドできない

- **症状**: `could not use fakeroot: no mapping entry found in /etc/subuid`、または `%post` の `apt-get` や `chown` が権限エラーで落ちる
- **確認**:
```bash
grep "^$USER:" /etc/subuid /etc/subgid
df -T "${APPTAINER_TMPDIR:-/tmp}" .                      # ビルド先が NFS かどうか
apptainer build --fakeroot --sandbox test_fr/ docker://ubuntu:24.04   # 小さなイメージで切り分け
```
- **原因**:
  - subuid / subgid に自分のエントリがない（管理者の設定が必要）
  - `APPTAINER_TMPDIR` や sandbox の置き場所が NFS で、ユーザー名前空間での所有者変更ができない
- **対処**: エントリがなければ管理者に依頼する。あればビルド先をノードのローカルディスク（`/tmp`、`/scratch`）にする。どうしても無理なら、root が使える環境でビルドした SIF を持ってくる

### 2.4 GPU は見えるが CUDA の初期化でエラーになる

- **症状**: `CUDA driver version is insufficient for CUDA runtime version`、`The NVIDIA driver on your system is too old`、`no kernel image is available for execution on the device`
- **確認**:
```bash
nvidia-smi | head -5                                        # ホストのドライバと対応 CUDA
apptainer exec --nv env.sif nvidia-smi | head -5            # コンテナ内から見たドライバ（同じはず）
apptainer exec --nv env.sif bash -c 'nvcc --version 2>/dev/null; python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.get_arch_list())"'
```
- **原因**:
  - コンテナ内の CUDA ランタイム（ベースイメージや torch wheel の CUDA 版）がドライバの対応版より新しい
  - `no kernel image` は、使っている GPU のアーキテクチャ（A6000 は sm_86）向けのカーネルがビルドに含まれていない
  - イメージ内に古い `libcuda.so` が入っていて、ホストのものより先に読まれている
- **対処**: ドライバが対応する CUDA 以下の版を選ぶ（torch は `cu124` / `cu128` などのインデックスで選ぶ、[uv.md](uv.md) 第6節）。イメージに `libcuda.so*` が入っていたら削除する

### 2.5 コンテナ内に nvidia-smi / libcuda がない

- **症状**: `nvidia-smi: command not found`、`libcuda.so.1: cannot open shared object file`、`WARNING: Could not find any nv files on this host!`
- **確認**:
```bash
echo "$CUDA_VISIBLE_DEVICES"; nvidia-smi -L                  # ホスト側で GPU が割り当てられているか
apptainer exec --nv env.sif ls /.singularity.d/libs | grep -c cuda
```
- **原因**: `--nv` を付けていない。GPU のないノード（ログインノードや `--gres` なしのジョブ）で実行しているとホストにドライバがなく、警告が出る
- **対処**: `--gres=gpu:1` 付きのジョブの中で `apptainer exec --nv` する

### 2.6 ホストの Python パッケージが混ざる

- **症状**: コンテナ内で import したパッケージの版が想定と違う。`pip list` にイメージに入れていないものが見える
- **確認**:
```bash
apptainer exec env.sif python -c "import sys; print('\n'.join(sys.path))"
apptainer exec env.sif python -c "import numpy; print(numpy.__file__)"
apptainer exec env.sif python -m site          # USER_SITE が有効かどうか
env | grep -E "^(PYTHON|CONDA|VIRTUAL_ENV)"
```
- **原因**: `$HOME` が bind されるので `~/.local/lib/python3.X/site-packages` が読み込まれる。ホストの `PYTHONPATH` や `VIRTUAL_ENV` も引き継がれる
- **対処**:
```bash
apptainer exec --env PYTHONNOUSERSITE=1 env.sif python a.py   # ~/.local を無視
apptainer exec --cleanenv env.sif python a.py                 # ホストの環境変数を渡さない
unset PYTHONPATH                                               # 投入前のシェルで
```
`~/.local` に古いパッケージが残っているなら、中身を確認してから整理する

### 2.7 undefined symbol / GLIBC_2.xx not found

- **症状**: `ImportError: ... undefined symbol`、`version 'GLIBC_2.38' not found`、`libstdc++.so.6: version 'GLIBCXX_3.4.32' not found`
- **確認**:
```bash
apptainer exec env.sif bash -c 'echo $LD_LIBRARY_PATH; ldd --version | head -1'
apptainer exec env.sif ldd <問題の .so> | grep -E "not found|/home|/tank|/workspace"
```
- **原因**:
  - ホストの conda や module の `LD_LIBRARY_PATH` が入り込み、コンテナの外のライブラリが読まれている
  - ホストの `.venv` を別の glibc のイメージで使っている（`.venv` 内のビルド済み拡張がイメージの glibc より新しい glibc を要求する）
- **対処**: `--cleanenv` で試す。原因が `LD_LIBRARY_PATH` なら `--env LD_LIBRARY_PATH=` で空にする。`.venv` 側の問題ならそのイメージの中で `uv sync` し直す

### 2.8 locale の警告が毎回出る

- **症状**: `bash: warning: setlocale: LC_ALL: cannot change locale (ja_JP.UTF-8)`、Perl の `Setting locale failed`
- **確認**:
```bash
env | grep -E "^(LANG|LC_)"
apptainer exec env.sif locale -a
```
- **原因**: ホストや `%environment` の `LANG` / `LC_ALL` が指すロケールがイメージ内に生成されていない
- **対処**: イメージに `C.UTF-8` を使わせる（`%environment` に `export LANG=C.UTF-8`）。日本語ロケールが必要なら `%post` で `apt-get install -y locales && locale-gen ja_JP.UTF-8`。すぐ回避するなら `--env LC_ALL=C.UTF-8`

### 2.9 bind したのに見えない・Permission denied

- **症状**: `No such file or directory`、`Permission denied`。ホストでは読めるファイル
- **確認**:
```bash
readlink -f data/raw                       # シンボリックリンクの実体
ls -ld "$(readlink -f data/raw)"
apptainer exec -B /tank env.sif ls -l "$(readlink -f data/raw)"
```
- **原因**:
  - シンボリックリンクの先（例: `data/ -> /tank/honzawa/...`）が bind されていない
  - `-B src:dst` の `dst` を間違えている（コード内のパスは dst 側）
  - `:ro` で bind した場所に書き込もうとしている
  - `--containall` / `--no-home` で既定の bind が外れている
- **対処**: リンク先の親ディレクトリも `-B` に加える。コード内のパスとコンテナ内のパスを揃える

### 2.10 FATAL: container creation failed

- **症状**: `FATAL: container creation failed: mount ...`、`while mounting image`、`failed to mount squashfs filesystem`
- **確認**:
```bash
apptainer --debug exec env.sif true 2>&1 | tail -40
ls -l env.sif; apptainer sif list env.sif     # SIF が壊れていないか
df -h /tmp
```
- **原因**:
  - SIF のコピーが途中で切れた（サイズが小さい、`sif list` がエラー）
  - bind の指定が不正（存在しないホストパス、`dst` が相対パス）
  - 書き込み可能な overlay を別のプロセスが使用中
  - FUSE（squashfuse / fuse-overlayfs）を使うため、`/dev/fuse` が使えない環境では失敗する
- **対処**: `--debug` の出力の最後の `mount` 行で原因を特定する。SIF は `sha256sum` で元と比べ、壊れていれば取り直す。overlay は `:ro` にするか、使用中のジョブを確認する

### 2.11 --writable-tmpfs で容量不足になる

- **症状**: `--writable-tmpfs` を付けて `pip install` や `apt-get` をしたら `No space left on device`
- **確認**:
```bash
grep "sessiondir max size" /etc/apptainer/apptainer.conf
apptainer exec --writable-tmpfs env.sif df -h /
```
- **原因**: `--writable-tmpfs` の書き込み層の上限は `sessiondir max size`（このマシンは 64MiB）
- **対処**: 永続化が不要なら書き込み先を bind したディレクトリにする（`--env TMPDIR=/scratch/...` など）。大きな変更が必要なら ext3 overlay（1.7節）か、def を直してビルドし直す

### 2.12 キャッシュの破損でビルド・pull が失敗する

- **症状**: `unexpected EOF`、`blob ... digest mismatch`、`failed to get checksum`。同じ URI を以前は取得できていた
- **確認**:
```bash
apptainer cache list -v
apptainer build --disable-cache /tmp/test.sif docker://<同じイメージ>
```
- **原因**: ダウンロードが途中で止まったレイヤーがキャッシュに残っている。複数のジョブが同じキャッシュに同時に書いた
- **対処**: `apptainer cache clean --type blob --force` で消して取り直す。同時に複数ビルドするときはジョブごとに `APPTAINER_CACHEDIR` を分ける

### 2.13 インスタンスが残る・ポートが使えない

- **症状**: `instance start` で `instance ... already exists`、サーバが `Address already in use` で起動しない
- **確認**:
```bash
apptainer instance list
ss -ltnp 2>/dev/null | grep <port>
```
- **原因**: 前回のインスタンスを止めずに終了した。同じノードで別のジョブが同じポートを使っている
- **対処**: `apptainer instance stop <name>`。ポートはジョブ内で空きを探して割り当てる（`python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1])'`）

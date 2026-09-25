# Apptainer チートシート

> 対象: Apptainer 1.4 系（このマシンは `apptainer 1.4.5`）。旧称 Singularity で、`singularity` コマンドも多くの環境で使える
> 公式: https://apptainer.org/docs/user/latest/
> 詳細編（応用・トラブル対応）: [apptainer_deep.md](apptainer_deep.md)

## 1. 基本概念

| 用語 | 意味 |
|:---|:---|
| SIF (`.sif`) | コンテナイメージ本体。1ファイルで読み取り専用 |
| 定義ファイル (`.def`) | イメージの作り方を書いたレシピ。Dockerfile に相当する |
| sandbox | 書き込み可能なディレクトリ形式のイメージ。試行錯誤用 |
| bind | ホストのディレクトリをコンテナ内に見せる仕組み |
| overlay | SIF の上に書き込み層を重ねる仕組み |

Docker との主な違い:
- コンテナ内でも**ホストと同じユーザー**で動く。root にはならない
- `$HOME`、カレントディレクトリ、`/tmp` はデフォルトで bind される
- デーモンが不要で、Slurm ジョブの中からそのまま実行できる

## 2. イメージを作る

```bash
apptainer build env.sif env.def                     # def からビルド
apptainer build --fakeroot env.sif env.def          # root 権限なしでビルド（許可されている環境のみ）
apptainer build env.sif docker://python:3.12-slim   # Docker Hub から直接作る
apptainer pull docker://ubuntu:24.04                # ubuntu_24.04.sif ができる
apptainer build --sandbox env_dir/ env.def          # 書き込み可能なディレクトリとして作る
apptainer build env.sif env_dir/                    # sandbox を SIF に固める
```

- ビルドは重いので、ログインノードではなく `srun` 経由で実行する
- キャッシュは `~/.apptainer/cache` にたまる。容量が足りなくなったら `apptainer cache clean`
- キャッシュや一時ファイルの場所は `APPTAINER_CACHEDIR` / `APPTAINER_TMPDIR` で変えられる

## 3. 定義ファイル (`.def`) の書き方

```singularity
Bootstrap: docker
From: nvidia/cuda:12.8.1-cudnn-devel-ubuntu24.04

%files
    requirements.txt /opt/requirements.txt

%environment
    export LANG=C.UTF-8
    export PATH=/opt/venv/bin:$PATH

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get update && apt-get install -y --no-install-recommends \
        curl git ca-certificates build-essential \
        && apt-get clean && rm -rf /var/lib/apt/lists/*

%runscript
    exec python "$@"

%labels
    Author you
    Version 0.1
```

| セクション | 実行されるタイミング | 用途 |
|:---|:---|:---|
| ヘッダ (`Bootstrap` / `From`) | ビルド時 | ベースイメージの指定 |
| `%files` | ビルド時（`%post` の前） | ホストのファイルをコピーする |
| `%post` | ビルド時 | パッケージのインストールなど。root として実行される |
| `%environment` | **実行時** | 環境変数。`%post` では効かない |
| `%runscript` | `apptainer run` の実行時 | デフォルトコマンド |
| `%labels` / `%help` | — | メタ情報 |

- `%post` の中で設定した `export` は実行時には残らない。実行時に使う変数は `%environment` に書く
- `Bootstrap` には `docker`（Docker Hub など）、`localimage`（手元の SIF）、`library` などが使える

## 4. 実行する

```bash
apptainer exec env.sif python train.py        # 任意のコマンドを実行（一番よく使う）
apptainer run env.sif train.py                # %runscript を実行
apptainer shell env.sif                       # コンテナ内のシェルに入る
apptainer exec --nv env.sif nvidia-smi        # GPU 付きで実行
```

### よく使うオプション

| オプション | 例 | 説明 |
|:---|:---|:---|
| `--nv` | | NVIDIA GPU を使えるようにする |
| `--bind` / `-B` | `-B /data:/data:ro` | `ホスト:コンテナ[:ro]`。カンマ区切りで複数指定できる |
| `--pwd` | `--pwd /work` | コンテナ内の作業ディレクトリ |
| `--env` | `--env SEED=1` | 環境変数を渡す |
| `--env-file` | `--env-file .env` | ファイルから環境変数を読む |
| `--cleanenv` / `-e` | | ホストの環境変数を引き継がない |
| `--containall` / `-C` | | `$HOME` などの自動 bind もしない。隔離を強める |
| `--no-home` | | `$HOME` を bind しない |
| `--writable-tmpfs` | | 一時的な書き込み層を付ける（終了時に消える） |
| `--overlay` | `--overlay ov.img` | 永続的な書き込み層を付ける |

### 環境変数でも指定できる

```bash
export APPTAINER_BIND="/data,/scratch"
export APPTAINERENV_WANDB_MODE=offline       # コンテナ内で WANDB_MODE=offline になる
```

## 5. Slurm と組み合わせる

```bash
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G

srun apptainer exec --nv \
  -B /path/to/data:/data \
  env.sif python train.py
```

対話的に作業する場合:

```bash
srun --gres=gpu:1 --pty apptainer shell --nv env.sif
```

## 6. 情報を確認する

```bash
apptainer inspect env.sif                 # ラベル
apptainer inspect --deffile env.sif       # ビルドに使った def ファイル
apptainer inspect --runscript env.sif     # runscript
apptainer exec env.sif cat /etc/os-release
apptainer cache list                      # キャッシュの使用量
```

## 7. uv / venv と組み合わせるパターン

イメージには OS パッケージと uv だけを入れ、Python パッケージはホスト側の `.venv` に入れる構成がよく使われる。

```bash
# 作業ディレクトリは自動で bind されるので、.venv はコンテナから見える
apptainer exec --nv env.sif uv sync
apptainer exec --nv env.sif uv run python train.py
```

- 良い点: パッケージを追加するたびにイメージを作り直さなくてよい
- 注意: `.venv` はコンテナ内の Python に紐づく。ホストの Python から直接使うと動かないことがある

## 8. CPU アーキテクチャと流用の可否

SIF の中身は特定の CPU アーキテクチャ向けにビルドされたバイナリなので、**アーキテクチャが違うマシンでは同じ SIF を使えない**。ホスト側の `.venv` も同じ理由で使い回せない。

| アーキテクチャ | 別名 | 例 |
|:---|:---|:---|
| `x86_64` | amd64 | Intel Xeon / AMD EPYC の一般的なノード（このマシンは `x86_64`） |
| `aarch64` | arm64 | NVIDIA Grace（GH200 など）、Apple Silicon、AWS Graviton |

### 流用できるもの・できないもの

| 対象 | 同じアーキテクチャ | 違うアーキテクチャ |
|:---|:---:|:---:|
| SIF | ✅（ドライバや glibc の差には注意） | ❌ 作り直す |
| `.venv`（ホスト側） | ✅（Python のバージョンとパスが同じなら） | ❌ 作り直す |
| `pyproject.toml` / `uv.lock` | ✅ | ✅（lock は複数プラットフォームに対応） |
| def ファイル | ✅ | △ ベースイメージや apt パッケージが arm64 に対応していれば使える |
| コード・データ・モデル重み | ✅ | ✅ |

### 確認と対処

```bash
uname -m                                   # ホストのアーキテクチャ
apptainer exec env.sif uname -m            # SIF のアーキテクチャ（違う場合はエラーになる）
file .venv/bin/python                      # .venv の Python のアーキテクチャ（シンボリックリンクなら readlink -f で実体を見る）

apptainer pull --arch arm64 docker://ubuntu:24.04   # アーキテクチャを指定して pull
```

- 違うアーキテクチャのノードで使うときは、**そのノード上で** def から SIF をビルドし直し、`uv sync` で `.venv` も作り直す
- 1つのリポジトリを両方のアーキテクチャで使う場合は、SIF と `.venv` をアーキテクチャ別に分ける（例: `env_x86_64.sif` / `env_aarch64.sif`、`UV_PROJECT_ENVIRONMENT=.venv-$(uname -m)`）
- ベースイメージが arm64 版を提供しているか確認する（`nvidia/cuda` は多くのタグで両方ある）
- aarch64 向けの wheel がないパッケージは、ソースからのビルドになって遅くなったり、失敗したりする
- 同じ x86_64 でも、`-march=native` でビルドしたバイナリは古い CPU で `Illegal instruction` になることがある

## 9. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| GPU が見えない (`torch.cuda.is_available()` が False) | `--nv` の付け忘れ |
| ファイルが見つからない | そのパスを bind していない。`-B` で追加する |
| `Read-only file system` | SIF は読み取り専用。bind したホスト側に書く |
| ホストの設定が混ざって挙動が変わる | `~/.local` の pip パッケージや環境変数が入り込んでいる。`--cleanenv` や `--no-home` を試す |
| `%post` で設定した変数が実行時にない | 実行時の変数は `%environment` に書く |
| ビルドで容量不足 | `APPTAINER_TMPDIR` / `APPTAINER_CACHEDIR` を大きいディスクに向ける |
| `exec format error` | SIF や `.venv` のアーキテクチャがホストと違う。第8節を参照 |
| `Illegal instruction` | CPU の世代が古く、命令セットが足りない（`-march=native` でビルドしたものなど） |
| `--fakeroot` でビルドできない | 管理者側の設定が必要。使えない場合は別の環境でビルドした SIF を持ってくる |

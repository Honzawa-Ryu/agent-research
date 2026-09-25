# uv 詳細編（応用・トラブル対応）

> 基本編: [uv.md](uv.md)
> 対象バージョン: uv 0.11 系（このマシンのプロジェクトの `.venv/pyvenv.cfg` には `uv = 0.11.8` と記録されている）
> 公式: https://docs.astral.sh/uv/
> 注: uv はホストになく、コンテナ（Apptainer）の中にだけある。コマンドは `apptainer exec env.sif uv ...` の形で、計算ノード上で実行する。本シートのオプションで手元の版にあるか不安なものは `uv help <command>` で確認する

## Part 1. 応用・上級機能

### 1.1 依存関係の分類

| 種類 | 書く場所 | 配布時に含まれるか | 入れ方 |
|:---|:---|:---:|:---|
| 本体の依存 | `[project] dependencies` | 含まれる | `uv sync` |
| extras（オプション依存） | `[project.optional-dependencies]` | 含まれる（利用者が選ぶ） | `uv sync --extra torch` / `--all-extras` |
| 依存グループ | `[dependency-groups]`（PEP 735） | 含まれない | `uv sync --group lint` / `--all-groups` |

```bash
uv add --optional torch torch torchvision   # extras "torch" に追加
uv add --group lint ruff                    # グループ "lint" に追加
uv sync --extra torch --extra nlp           # extras を選んで入れる
uv sync --only-group lint                   # そのグループだけ（プロジェクト本体も入れない）
uv sync --no-group dev                      # 既定のグループから dev を外す
uv sync --no-default-groups                 # 既定のグループを全部外す
```

```toml
[dependency-groups]
lint = ["ruff"]
test = ["pytest", { include-group = "lint" }]   # 他のグループを取り込む

[tool.uv]
default-groups = ["dev", "test"]                 # uv sync で既定で入るグループ（初期値は ["dev"]）
```

- 研究用のプロジェクト（配布しない）では extras と依存グループのどちらでもよい。`toxpatho-ssl-comparison` は extras（`torch` / `nlp` / `gbdt` / `serving` / `dev`）で分け、`tools/uv_sync.sh` で `uv sync --all-extras` している
- extras を足したら、`uv sync` だけでは入らない。同期コマンドと揃える

### 1.2 環境マーカーとプラットフォーム別の依存

```toml
[project]
dependencies = [
    "uvloop; sys_platform == 'linux'",
    "triton; platform_machine == 'x86_64'",
    "tomli; python_version < '3.11'",
]
```

| マーカー | 値の例 |
|:---|:---|
| `sys_platform` | `linux`, `darwin`, `win32` |
| `platform_machine` | `x86_64`, `aarch64`, `arm64`（macOS） |
| `python_version` | `3.12` |
| `implementation_name` | `cpython` |

ソースもマーカーで分けられる:

```toml
[tool.uv.sources]
torch = [
    { index = "pytorch-cu128", marker = "sys_platform == 'linux'" },
    { index = "pytorch-cpu",   marker = "sys_platform == 'darwin'" },
]
```

### 1.3 解決対象の環境を絞る・要求する

`uv.lock` は既定で全プラットフォーム向けに解決する（`resolution-markers` に win32 / darwin などが並ぶ）。

```toml
[tool.uv]
# Linux だけを対象に解決する（Windows・macOS 向けの解決を省く。lock が小さく速くなる）
environments = ["sys_platform == 'linux'"]

# これらの環境向けの wheel が必ずあることを要求する（無ければ lock 時にエラー）
required-environments = [
    "sys_platform == 'linux' and platform_machine == 'x86_64'",
    "sys_platform == 'linux' and platform_machine == 'aarch64'",
]
```

- `environments`: 解決の範囲を狭める。対象外の環境では `uv sync` できなくなる
- `required-environments`: 範囲は変えず、wheel の存在を保証させる。x86_64 と aarch64（GH200 など）の両方で使う場合に、片方だけ wheel がないパッケージを lock の時点で検出できる

### 1.4 衝突する extras の宣言（CPU 版 / CUDA 版の切り替え）

同時に入れられない extras は `conflicts` で宣言すると、別々に解決される。

```toml
[project.optional-dependencies]
cpu   = ["torch", "torchvision"]
cu128 = ["torch", "torchvision"]

[tool.uv]
conflicts = [[{ extra = "cpu" }, { extra = "cu128" }]]

[tool.uv.sources]
torch = [
    { index = "pytorch-cpu",   extra = "cpu" },
    { index = "pytorch-cu128", extra = "cu128" },
]
torchvision = [
    { index = "pytorch-cpu",   extra = "cpu" },
    { index = "pytorch-cu128", extra = "cu128" },
]

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true

[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true
```

```bash
uv sync --extra cu128     # GPU ノード
uv sync --extra cpu       # CPU だけの環境
```

依存グループ同士の衝突は `{ group = "..." }` で書く。

### 1.5 依存関係の上書き・制約

| 設定 | 効果 | 使いどころ |
|:---|:---|:---|
| `constraint-dependencies` | バージョンの上限・下限を追加する（入れはしない） | 推移的依存の版を絞る |
| `override-dependencies` | 他のパッケージが宣言した要求を無視して置き換える | 上流の依存指定が厳しすぎて解決できないとき |
| `no-build-isolation-package` | 指定パッケージのビルドで隔離環境を使わない | ビルド時に torch を import する flash-attn など |

```toml
[tool.uv]
constraint-dependencies = ["numpy<2.3", "protobuf<5"]
override-dependencies   = ["pydantic>=2.8"]      # 上流が pydantic<2.5 を要求していても無視
no-build-isolation-package = ["flash-attn"]
```

- `override-dependencies` は整合性の確認を飛ばすので、実行時に壊れる可能性がある。理由をコメントに残す
- `no-build-isolation-package` を使う場合、先に torch などビルドに必要なものを入れておく（`uv sync --no-install-package flash-attn` → `uv sync`）

### 1.6 インデックスの使い分け

```toml
[[tool.uv.index]]
name = "pytorch-cu130"
url = "https://download.pytorch.org/whl/cu130"
explicit = true            # sources で指定したパッケージだけがこのインデックスを使う

[[tool.uv.index]]
name = "internal"
url = "https://pypi.example.org/simple"
default = true             # PyPI の代わりに既定のインデックスにする

[tool.uv]
index-strategy = "first-index"   # 既定。最初に見つかったインデックスの版だけを候補にする
```

| `index-strategy` | 動作 |
|:---|:---|
| `first-index`（既定） | パッケージが見つかった最初のインデックスだけを使う。依存の乗っ取りを防ぐ |
| `unsafe-first-match` | 全インデックスを探すが、最初のインデックスの版を優先 |
| `unsafe-best-match` | 全インデックスから最適な版を選ぶ（pip に近い） |

- `explicit = true` のインデックスは、`[tool.uv.sources]` で指定したパッケージにしか使われない
- sources は**このプロジェクトが直接依存するパッケージ**に効く。推移的依存（例: timm が要求する torch）の取得元を固定したいときは、torch を `dependencies` に直接書く
- 認証が必要なインデックスは環境変数で渡す: `UV_INDEX_INTERNAL_USERNAME` / `UV_INDEX_INTERNAL_PASSWORD`（`INTERNAL` はインデックス名の大文字）

### 1.7 ワークスペース（複数パッケージを1つの lock で管理）

```
repo/
    pyproject.toml          # ルート
    uv.lock                 # ワークスペース全体で1つ
    packages/
        core/pyproject.toml
        models/pyproject.toml
```

```toml
# ルートの pyproject.toml
[tool.uv.workspace]
members = ["packages/*"]
exclude = ["packages/old"]

[tool.uv.sources]
core = { workspace = true }    # メンバーを依存として使う（編集可能モードで入る）
```

```bash
uv sync --package models        # 特定メンバーの環境を作る
uv sync --all-packages          # 全メンバーを入れる
uv run --package models python -m models.train
uv lock                         # どこで実行してもワークスペース全体を解決する
```

- メンバー全体で依存の版が1つに揃う。メンバーごとに違う版が必要なら、ワークスペースにせず別プロジェクトにする

### 1.8 lock の操作

```bash
uv lock --upgrade-package torch==2.8.0     # 特定パッケージだけ指定の版にする
uv lock --upgrade-package trident          # git 依存を最新コミットに更新
uv lock --resolution lowest-direct         # 直接依存を下限の版で解決（下限が正しいかの確認）
uv lock --resolution lowest                # 推移的依存も下限で
uv lock --exclude-newer 2026-09-01T00:00:00Z   # この日時より後に公開された版を使わない
uv tree --invert --package numpy           # numpy を要求しているのは誰か
uv tree --depth 1                          # 直接依存だけ
```

```toml
[tool.uv]
exclude-newer = "2026-09-01T00:00:00Z"   # 再解決しても結果が変わりにくくなる
required-version = ">=0.11"              # uv の版を揃える（古い uv での実行をエラーにする）
```

### 1.9 他の形式への書き出し

```bash
uv export --format requirements-txt --no-hashes -o requirements.txt   # uv.lock から
uv export --no-dev --extra torch -o requirements.txt
uv export --no-emit-project -o requirements.txt                       # プロジェクト自身を含めない

uv pip compile requirements.in -o requirements.txt --universal        # 全プラットフォーム向け（マーカー付き）
uv pip compile requirements.in -o requirements.txt \
    --python-version 3.12 --python-platform x86_64-manylinux_2_28      # 環境を指定して解決
uv pip sync requirements.txt                                          # requirements に完全一致させる（余分は消す）
```

### 1.10 単体スクリプト（インラインメタデータ, PEP 723）

プロジェクトを作らずに、スクリプトの先頭に依存を書く。

```bash
uv init --script plot.py --python 3.12     # メタデータ付きのひな形を作る
uv add --script plot.py pandas matplotlib  # 依存を追加
uv run --script plot.py                    # 専用の一時環境で実行
uv lock --script plot.py                   # plot.py.lock を作って版を固定
```

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "pandas>=2",
#     "matplotlib",
# ]
# ///
import pandas as pd
```

- プロジェクトの中で `uv run --script` を使うと、プロジェクトの `.venv` は使われない
- 実験用コードは実験テンプレートの管理下に置く。PEP 723 は使い捨ての集計・確認スクリプト向け

### 1.11 sync の細かい制御

| オプション | 動作 |
|:---|:---|
| `--inexact` | lock にないパッケージを消さない |
| `--no-install-project` | 依存だけ入れ、プロジェクト自身は入れない |
| `--no-install-package <pkg>` | 指定パッケージを入れない |
| `--no-editable` | プロジェクトやワークスペースを編集可能モードでなく通常インストール |
| `--reinstall` / `--reinstall-package <pkg>` | 入れ直す（壊れた環境の修復） |
| `--compile-bytecode` | インストール時に .pyc を作る（初回 import が速くなる） |
| `--python <path>` | 使うインタプリタを指定 |

### 1.12 Python 本体の管理

```bash
uv python find                    # 今使われる Python のパス
uv python list --only-installed
UV_PYTHON=/usr/bin/python3 uv sync           # コンテナのシステム Python を使わせる
UV_PYTHON_DOWNLOADS=never uv sync            # Python を自動ダウンロードさせない
UV_PYTHON_PREFERENCE=only-system uv sync     # uv 管理の Python を使わない
```

- 実験テンプレートの `.venv` はコンテナのシステム Python（`/usr/bin/python3`、3.12.3）で作られている（`.venv/bin/python -> /usr/bin/python3`）
- uv が Python をダウンロードすると `~/.local/share/uv/python` に入る。コンテナとホストで同じ `$HOME` を共有するので混乱のもとになる

### 1.13 キャッシュとオフライン

```bash
uv cache size                  # キャッシュの容量（古い版にはない。なければ du -sh "$(uv cache dir)"）
uv cache prune --ci            # CI 向け: ビルドした wheel 以外（ダウンロード物）を消す
uv sync --offline              # ネットワークを使わない（キャッシュにあるものだけ）
uv sync --no-cache             # キャッシュを使わない（一時ディレクトリを使う）
```

| 環境変数 | 用途 |
|:---|:---|
| `UV_OFFLINE=1` | `--offline` と同じ |
| `UV_HTTP_TIMEOUT` | HTTP のタイムアウト秒数（大きい wheel の取得で切れる場合） |
| `UV_CONCURRENT_DOWNLOADS` | 同時ダウンロード数 |
| `UV_NATIVE_TLS=1` | OS の証明書ストアを使う（社内プロキシの証明書など） |
| `UV_INDEX_STRATEGY` | `index-strategy` と同じ |
| `UV_FROZEN=1` / `UV_LOCKED=1` | `--frozen` / `--locked` と同じ |
| `UV_COMPILE_BYTECODE=1` | `--compile-bytecode` と同じ |

### 1.14 パッケージのビルドと公開（参考）

```bash
uv build                        # dist/ に sdist と wheel を作る
uv build --wheel
uv version                      # pyproject のバージョンを表示
uv version --bump patch         # 0.1.0 → 0.1.1
uv publish --index testpypi     # 公開（トークンは UV_PUBLISH_TOKEN）
```

## Part 2. トラブル対応

### 2.1 依存関係が解決できない

- **症状**: `No solution found when resolving dependencies:` の後に `Because ... depends on ... and ... we can conclude that your requirements are unsatisfiable.`
- **確認**:
```bash
apptainer exec env.sif uv lock 2>&1 | tee lock_err.txt      # 全文を保存して読む
apptainer exec env.sif uv tree --invert --package <衝突しているパッケージ>
grep -n "requires-python" pyproject.toml
```
- **原因**: エラー文は「A は B>=2 を要求し、C は B<2 を要求する」という推論の連鎖になっている。最後の `we can conclude` の直前に、衝突している2つの要求が書かれている。よくあるのは次の3つ
  - `requires-python` の範囲が広く、範囲のどこかで対応する版がない（`the requested Python version (>=3.10) does not satisfy Python>=3.11`）
  - 2つの直接依存が同じパッケージの互いに重ならない範囲を要求している
  - `explicit = true` のインデックスに必要な版がない（`+cu130` など）
- **対処**:
  - `requires-python = "==3.12.*"` のように実際に使う版に絞る（実験テンプレートはこの形）
  - どちらかの直接依存の版を上げ下げする。`uv lock --upgrade-package <pkg>` で片方ずつ試す
  - 上流の指定が厳しすぎるだけなら `override-dependencies`（1.5節）

### 2.2 CUDA 版の torch にならない・torch が2つある

- **症状**: `torch.version.cuda` が `None`、`torch.__version__` に `+cpu` が付く。または意図しない版の torch が読み込まれる
- **確認**:
```bash
apptainer exec --nv env.sif .venv/bin/python -c "import torch; print(torch.__version__, torch.version.cuda, torch.__file__)"
grep -n -A3 'name = "torch"' uv.lock | head -20            # どのインデックスから来ているか
grep -nE "torch|pytorch" pyproject.toml
```
- **原因**:
  - torch が直接依存に書かれておらず、他のパッケージ経由で PyPI から入った（sources が効かない）
  - `[tool.uv.sources]` に torch はあるが torchvision / torchaudio が抜けていて、組み合わせが崩れている
  - コンテナ側にも torch があり（NGC イメージなど）、`.venv` とどちらが読まれるかが環境変数で変わる
- **対処**:
  - torch, torchvision, torchaudio をすべて `dependencies` に書き、3つとも同じインデックスを sources で指定する
  - `torch.__file__` が `.venv` の中を指しているか確認する。コンテナの torch を使う方針なら、pyproject から torch を外してその旨を書く

### 2.3 Failed to hardlink files の警告が毎回出る

- **症状**: `warning: Failed to hardlink files; falling back to full copy.` がジョブのたびに出る
- **確認**:
```bash
apptainer exec env.sif uv cache dir
df "$(dirname .venv)" "<キャッシュのパス>" | awk 'NR>1{print $1, $6}'   # 別のファイルシステムか
echo "$UV_CACHE_DIR"
```
- **原因**: キャッシュと `.venv` が別のファイルシステムにある。実験テンプレートの `slurm_entry.sh` はジョブ内で `UV_CACHE_DIR=${SCRATCH_DIR}/.uv_cache`（ノードのローカル SSD）を設定するので、`/workspace` 上の `.venv` とは必ず別になる
- **対処**: `UV_LINK_MODE=copy` を設定して警告を消す。ジョブ内で sync しないなら `uv run --no-sync` か `.venv` を直接 activate する（テンプレートは activate 方式）

### 2.4 lock が勝手に書き換わる・--locked で失敗する

- **症状**: `uv run` のたびに `uv.lock` に差分が出る。`uv sync --locked` が `The lockfile at uv.lock needs to be updated, but --locked was provided.` で止まる
- **確認**:
```bash
git diff --stat uv.lock
head -3 uv.lock                                        # version / revision
apptainer exec env.sif uv --version
apptainer exec env.sif uv lock --check
```
- **原因**:
  - `pyproject.toml` を編集したが lock を更新していない
  - uv の版が違うマシン・コンテナで実行し、lock の形式（`revision`）が更新された
  - git 依存に `rev` がなく、再解決のたびに最新コミットを取りに行っている
- **対処**:
  - pyproject を変えたら `uv lock` して両方をコミットする
  - `[tool.uv] required-version` で uv の版を揃える。SIF ごとに uv の版が決まるので、SIF を作り直したら一度 `uv lock` する
  - 本番ジョブでは `uv sync --frozen` / `--locked` を使い、lock を書き換えさせない
  - git 依存は `rev = "<commit>"` で固定する

### 2.5 aarch64 など別プラットフォームで wheel がない

- **症状**: `Distribution ... can't be installed because it doesn't have a source distribution or wheel for the current platform`。ヒントに `only has wheels for the following platforms` と出る
- **確認**:
```bash
uname -m
apptainer exec env.sif python3 -c "import platform, sysconfig; print(platform.machine(), sysconfig.get_platform())"
grep -n -A15 'name = "<pkg>"' uv.lock | grep -E "wheels|url" | head
```
- **原因**: そのパッケージが x86_64 向けの wheel しか公開しておらず、sdist もない（vllm、bitsandbytes の一部の版、CUDA 系のバイナリなど）
- **対処**:
  - そのプラットフォームで不要なら、マーカーで外す: `"vllm>=0.22; platform_machine == 'x86_64'"`
  - 必要なら wheel がある版を探す。なければコンテナ側（NGC など）に入っているものを使う
  - 両方のアーキテクチャで使うプロジェクトは `required-environments`（1.3節）で lock の時点で検出する
  - `.venv` はアーキテクチャごとに分ける: `UV_PROJECT_ENVIRONMENT=.venv-$(uname -m)`

### 2.6 sdist のビルドに失敗する

- **症状**: `Failed to build <pkg>`、その下に `fatal error: Python.h: No such file or directory`、`xxx.h: No such file`、`error: command 'gcc' failed`、`ModuleNotFoundError: No module named 'torch'`（ビルド中）
- **確認**:
```bash
apptainer exec env.sif uv sync -v 2>&1 | tail -60
apptainer exec env.sif bash -c 'which gcc; ls /usr/include/python3.12/Python.h; dpkg -l | grep -E "python3-dev|build-essential"'
```
- **原因**:
  - wheel がない版を選んだため、ソースからビルドしている
  - コンテナにコンパイラや開発用ヘッダ（`python3-dev`、`libxxx-dev`）がない
  - ビルド時に torch を import するパッケージ（flash-attn など）が、隔離されたビルド環境で torch を見つけられない
- **対処**:
  - まず wheel がある版に変えられないか確認する（`uv lock --upgrade-package <pkg>`、版を指定）
  - 必要なヘッダを def の `%post` で入れて SIF を作り直す（`build-essential python3-dev libcairo2-dev` など）
  - torch が必要なビルドは `no-build-isolation-package`（1.5節）

### 2.7 ネットワークエラー・タイムアウト

- **症状**: `Failed to fetch`、`error sending request`、`dns error`、`operation timed out`、`invalid peer certificate`
- **確認**:
```bash
# uv_sync と同じく計算ノードのジョブの中で確認する
curl -sSI https://pypi.org/simple/ | head -1
curl -sSI https://download.pytorch.org/whl/cu130/ | head -1
env | grep -iE "^(https?|no)_proxy"
```
- **原因**: 計算ノードから外部に出られない、プロキシの設定がジョブに引き継がれていない、大きな wheel（torch は数 GB）の取得がタイムアウトする、プロキシの証明書を信頼していない
- **対処**:
  - タイムアウトなら `UV_HTTP_TIMEOUT=300` を設定する
  - 証明書エラーなら `UV_NATIVE_TLS=1`
  - 外に出られないノードでは、出られるノードで一度 `uv sync` してキャッシュを作り、同じ `UV_CACHE_DIR` を指して `uv sync --offline`

### 2.8 .venv が壊れた・別の Python で動いている

- **症状**: `.venv/bin/python` で `ModuleNotFoundError`、`bad interpreter: No such file or directory`、`uv run` が `.venv` を作り直そうとする
- **確認**:
```bash
ls -l .venv/bin/python                  # 例: -> /usr/bin/python3
cat .venv/pyvenv.cfg                    # home, version_info, uv の版
head -1 .venv/bin/<何かのコマンド>      # shebang の絶対パス
apptainer exec env.sif /usr/bin/python3 --version; /usr/bin/python3 --version
```
- **原因**:
  - `.venv/bin/python` は `/usr/bin/python3` へのリンクで、実行した場所（コンテナ内かホストか）によって違う Python を指す。ホストとコンテナの版が偶然同じ（どちらも 3.12.3）なら動いて見えるが、C 拡張がコンテナのライブラリを前提にしていると失敗する
  - プロジェクトのディレクトリを移動・リネームした。`.venv/bin/*` のスクリプトは shebang に絶対パスを持つ
  - SIF を作り直して Python のマイナー版が変わった
- **対処**: `.venv` はコンテナ内でだけ使う。壊れたら作り直す
```bash
rm -rf .venv
make uv_sync p=<partition>              # コンテナ内で uv sync（実験テンプレートの場合）
```

### 2.9 uv: command not found

- **症状**: ホストのシェルで `uv` を実行すると `command not found`
- **確認**:
```bash
command -v uv || echo "host: none"
apptainer exec env.sif which uv
```
- **原因**: このマシンでは uv はコンテナの中にだけ入っている（def の `%post` で `/usr/local/bin/uv` に配置）
- **対処**: `apptainer exec env.sif uv ...` を計算ノード上で実行する。同期だけなら `make uv_sync p=<partition>`。ホストに独自に入れない

### 2.10 並列ジョブで .venv が壊れる・uv run が遅い

- **症状**: アレイジョブで `uv run` を使うと、一部のタスクが `No such file or directory` や import エラーで落ちる。起動に毎回数十秒かかる
- **確認**:
```bash
grep -n "uv run" run_slurm.sh experiments/*/run_slurm.sh 2>/dev/null
ls -la --time-style=full-iso .venv/lib/python3.12/site-packages | tail -5   # ジョブ中に更新されていないか
```
- **原因**: `uv run` は実行前に lock と `.venv` を確認し、必要なら書き換える。複数のジョブが同時に同じ `.venv` を書き換えると壊れる
- **対処**: ジョブでは `uv run --frozen --no-sync`（または `UV_NO_SYNC=1`）を使うか、`.venv/bin/activate` してから `python` を直接呼ぶ。同期は投入前に1回だけ行う

### 2.11 Python が見つからない・勝手にダウンロードされる

- **症状**: `No interpreter found for Python 3.12 in managed installations or search path`、または `Downloading cpython-3.12...` と出て `~/.local/share/uv/python` に Python が入る
- **確認**:
```bash
cat .python-version 2>/dev/null; grep requires-python pyproject.toml
apptainer exec env.sif uv python find
apptainer exec env.sif uv python list --only-installed
```
- **原因**: `.python-version` や `requires-python` の指定と、コンテナにある Python（`/usr/bin/python3`）の版が合わない。合わないと uv は管理下の Python をダウンロードしようとする
- **対処**: 指定をコンテナの版に合わせる。ダウンロードを禁止するなら `UV_PYTHON_DOWNLOADS=never`、使う Python を固定するなら `UV_PYTHON=/usr/bin/python3`

### 2.12 git 依存が更新されない・取得に失敗する

- **症状**: 上流のリポジトリを更新したのに古いコードが使われる。`Git operation failed`、`failed to fetch`
- **確認**:
```bash
grep -n -A2 'git = ' pyproject.toml
grep -n 'git+https' uv.lock | head                    # lock に固定されたコミット
apptainer exec env.sif .venv/bin/python -c "import <pkg>; print(<pkg>.__file__)"
```
- **原因**: `rev` を書かなくても、lock に解決時点のコミットが記録される。`uv sync` は lock のコミットを使うので、上流を更新しても変わらない。取得失敗は計算ノードから GitHub に出られない場合
- **対処**:
```bash
apptainer exec env.sif uv lock --upgrade-package <pkg>   # 最新コミットに更新
```
再現性のため、`[tool.uv.sources]` に `rev = "<commit>"` を明記しておく

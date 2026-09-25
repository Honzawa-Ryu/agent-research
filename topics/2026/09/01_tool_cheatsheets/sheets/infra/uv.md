# uv チートシート

> 対象: uv（Astral 製の Python パッケージ・プロジェクト管理ツール）。pip、venv、pip-tools、pyenv の役割をまとめて担う
> 公式: https://docs.astral.sh/uv/
> 注: このマシンではホストに uv がなく、コンテナ内（Apptainer）にだけ入っている

## 1. 基本概念

| ファイル / 用語 | 意味 |
|:---|:---|
| `pyproject.toml` | 依存関係を人が書くファイル。範囲指定（`numpy>=2`）でよい |
| `uv.lock` | 解決済みの正確なバージョンを記録したファイル。自動で生成され、git に入れる |
| `.venv/` | プロジェクトの仮想環境。git には入れない |
| `.python-version` | 使う Python のバージョン |
| sync | `uv.lock` どおりに `.venv` を揃える操作 |

流れ: `pyproject.toml` を編集 → `uv lock` で `uv.lock` を更新 → `uv sync` で `.venv` に反映。
`uv add` / `uv run` はこの3つを自動でまとめて行う。

## 2. プロジェクトを作る

```bash
uv init myproj                # 新規プロジェクト（pyproject.toml などができる）
uv init --package myproj      # src レイアウトのパッケージとして作る
uv python pin 3.12            # .python-version を作る
```

## 3. 依存関係を管理する

```bash
uv add numpy pandas               # 追加（pyproject と uv.lock が更新される）
uv add "torch>=2.4"               # バージョンを指定
uv add --dev pytest ruff          # 開発用グループ（dependency-groups.dev）に追加
uv add --group notebook jupyterlab  # 任意の名前のグループに追加
uv add git+https://github.com/org/repo.git@<rev>   # git リポジトリから追加
uv add --editable ./libs/mylib    # ローカルのパッケージを編集可能モードで追加
uv remove pandas                  # 削除
uv tree                           # 依存関係をツリー表示
uv tree --outdated                # 更新できるものを表示
```

## 4. 環境を揃える・実行する

```bash
uv sync                    # uv.lock どおりに .venv を作る・揃える
uv sync --frozen           # uv.lock を更新せずに使う（再現性重視）
uv sync --locked           # uv.lock が pyproject と食い違っていたらエラーにする
uv sync --no-dev           # dev グループを除く
uv sync --group notebook   # 追加のグループも入れる
uv sync --all-groups       # 全グループを入れる

uv run python train.py     # .venv の Python で実行（必要なら先に sync する）
uv run --no-sync python a.py   # sync を省く（速いが、環境が古い可能性がある）
uv run pytest
uv run --with rich python a.py   # 一時的にパッケージを足して実行
```

- `uv run` を使えば `source .venv/bin/activate` は不要
- activate して使ってもよい: `source .venv/bin/activate` → `python train.py`

## 5. lock の更新

```bash
uv lock                          # pyproject の変更を uv.lock に反映
uv lock --upgrade                # 全パッケージを範囲内の最新に上げる
uv lock --upgrade-package numpy  # 特定パッケージだけ上げる
uv lock --check                  # lock が最新かどうかを確認（CI 向け）
```

## 6. pyproject.toml の書き方

```toml
[project]
name = "myproj"
version = "0.1.0"
requires-python = "==3.12.*"
dependencies = [
    "numpy>=2",
    "torch",
    "torchvision",
]

[dependency-groups]
dev = ["pytest", "ruff"]

# --- git リポジトリの特定コミットを使う ---
[tool.uv.sources]
trident = { git = "https://github.com/mahmoodlab/TRIDENT.git", rev = "<commit>" }

# --- パッケージとしてビルドしない（スクリプト集のプロジェクト向け） ---
[tool.uv]
package = false
```

### PyTorch を CUDA 版で入れる

PyPI の torch は環境によっては CUDA 版にならない。PyTorch 専用のインデックスを指定する。

```toml
[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true      # ここで指定したパッケージだけがこのインデックスを使う

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
torchvision = { index = "pytorch-cu128" }
```

- `cu128` の部分は CUDA のバージョン（例: `cu121`, `cu124`, `cu128`, `cpu`）
- ドライバが対応している CUDA より新しいものは使えない。`nvidia-smi` の右上で確認する

## 7. Python 本体の管理

```bash
uv python list             # 使えるバージョンの一覧
uv python install 3.12     # インストール
uv venv --python 3.12      # バージョンを指定して .venv を作る
```

## 8. pip 互換のインターフェース

プロジェクト管理を使わず、pip の代わりとして使う場合。

```bash
uv venv                            # .venv を作る
uv pip install -r requirements.txt
uv pip install -e .
uv pip list
uv pip freeze > requirements.txt
uv pip compile requirements.in -o requirements.txt   # pip-tools と同じ
```

`uv pip install` した分は `pyproject.toml` / `uv.lock` に記録されない。次の `uv sync` で消えることがある。

## 9. ツール（CLI）を単体で使う

```bash
uvx ruff check .           # 一時的にインストールして実行（uv tool run の略）
uv tool install ruff       # 常用する CLI としてインストール
uv tool list
```

## 10. キャッシュと環境変数

```bash
uv cache dir               # キャッシュの場所
uv cache clean             # 全削除
uv cache prune             # 使っていないものだけ削除
```

| 環境変数 | 用途 |
|:---|:---|
| `UV_CACHE_DIR` | キャッシュの場所を変える（ホームの容量が小さい場合など） |
| `UV_PROJECT_ENVIRONMENT` | `.venv` 以外の場所に環境を作る |
| `UV_LINK_MODE=copy` | キャッシュと別のファイルシステムにある場合の警告を消す |
| `UV_PYTHON` | 使う Python を指定する |
| `UV_NO_SYNC=1` | `uv run` で sync しない |

## 11. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `torch.cuda.is_available()` が False | CPU 版の torch が入っている。第6節のインデックス指定をする |
| `Failed to hardlink files` の警告 | キャッシュと `.venv` のファイルシステムが違う。`UV_LINK_MODE=copy` を設定する |
| 他人の環境で動かない | `uv.lock` をコミットしていない、または `uv sync --frozen` を使っていない |
| pip で入れたパッケージが消えた | `uv sync` は lock にないものを消す。`uv add` で入れる |
| `.venv` を別の Python で使ったら壊れた | `.venv` は作った Python に紐づく。コンテナ内で作った `.venv` はコンテナ内で使う |
| 別のアーキテクチャのノードで `.venv` が動かない | `.venv` は CPU アーキテクチャにも依存する。そのノードで `uv sync` し直す（[apptainer.md](apptainer.md) 第8節） |
| ホームの容量が足りない | キャッシュが大きくなる。`UV_CACHE_DIR` を大きいディスクに向ける |

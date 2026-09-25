# timm 詳細編（応用・トラブル対応）

> 基本編: [timm.md](timm.md)
> 対象バージョン: `timm 1.0.28`（`torch 2.11.0`、`huggingface_hub 1.17.0`、`peft 0.19.1` と組み合わせて確認）

## Part 1. 応用・上級機能

### 1.1 pretrained_cfg を読む

モデルが「どの重みで・どんな前処理で」学習されたかは `pretrained_cfg` に入っている。

```python
m = timm.create_model("vit_small_patch16_224.augreg_in21k", pretrained=False)
m.pretrained_cfg                      # dict。m.default_cfg も同じもの
m.pretrained_cfg["mean"], m.pretrained_cfg["std"], m.pretrained_cfg["hf_hub_id"]

from timm.models import get_pretrained_cfg, get_pretrained_cfg_value
get_pretrained_cfg("vit_small_patch16_224.augreg_in21k").hf_hub_id   # モデルを作らずに調べる
get_pretrained_cfg_value("vit_small_patch16_224", "mean")            # (0.5, 0.5, 0.5)
```

| キー | 内容 |
|:---|:---|
| `hf_hub_id` / `url` | 重みの取得先 |
| `input_size` / `crop_pct` / `interpolation` | 評価時の前処理 |
| `mean` / `std` | 正規化。**ViT は (0.5, 0.5, 0.5) のものが多く、ImageNet の値とは違う** |
| `num_classes` / `classifier` | 元のヘッドのクラス数・層名 |
| `first_conv` | 入力層の名前（`in_chans` を変えるときに使われる） |

`timm.data.resolve_model_data_config(model)` は `resolve_data_config({}, model=model)` の短縮形。

### 1.2 ViT の create_model 引数

`VisionTransformer.__init__` の引数は `create_model` の kwargs でそのまま渡せる。

| 引数 | 内容 |
|:---|:---|
| `img_size` / `patch_size` | 入力サイズ・パッチサイズ |
| `dynamic_img_size=True` | 入力ごとに pos_embed を補間する |
| `dynamic_img_pad=True` | パッチサイズで割り切れない入力をパディングする |
| `class_token=False` | CLS トークンなし（`global_pool="avg"` と組み合わせる） |
| `reg_tokens=4` | レジスタトークン（DINOv2-reg など） |
| `no_embed_class=True` | pos_embed をプレフィックストークンに加えない |
| `init_values=1e-5` | LayerScale を有効にする（UNI など。重みの構造が変わる） |
| `global_pool` | `"token"`, `"avg"`, `"avgmax"`, `"max"`, `"map"`, `""` |
| `fc_norm` | プーリング後に norm をかけるか |
| `qk_norm=True` | Q/K の正規化 |
| `pos_embed="none"` | 位置埋め込みなし |

```python
m = timm.create_model("vit_small_patch16_224", reg_tokens=4, num_classes=0)
m.num_prefix_tokens          # 5（CLS 1 + レジスタ 4）
m.pos_embed.shape            # (1, 201, 384)
```

重みと構造を変える引数（`init_values`、`reg_tokens`、`class_token` など）は、学習時と同じ値にしないと読み込みで失敗する。

### 1.3 ローカルの重みを読む（load_checkpoint と filter_fn）

```python
from timm.models import load_checkpoint
from timm.models.vision_transformer import checkpoint_filter_fn

model = timm.create_model("vit_base_patch16_224", pretrained=False, num_classes=0, img_size=384)
res = load_checkpoint(model, "dino_vitb16.pth", strict=False, filter_fn=checkpoint_filter_fn)
print(res.missing_keys, res.unexpected_keys)
```

| `load_checkpoint` の引数 | 内容 |
|:---|:---|
| `use_ema=True` | ファイルに `state_dict_ema` / `model_ema` があればそちらを読む |
| `strict` | 既定 True |
| `filter_fn` | `(state_dict, model) -> state_dict`。キー名の変換・pos_embed の補間など |
| `remap=True` | キー名を無視して順番で対応させる（最終手段） |
| `weights_only` | 既定 True |

- ファイル内の `model` / `state_dict` キーの下にある辞書や、`module.` 接頭辞は自動で処理される
- ViT の `checkpoint_filter_fn` は、DINOv2・OpenCLIP・IJEPA などの形式の変換、patch_embed と pos_embed のサイズ合わせを行う
- `create_model(..., checkpoint_path=...)` は `filter_fn` なし・`strict=True` で読む。サイズが違う重みはこちらでは読めない

自前の学習コードで保存した重み（DINO の `teacher` など）はキー名の整理が必要:

```python
sd = torch.load("ckpt.pth", map_location="cpu")["teacher"]
sd = {k.removeprefix("backbone."): v for k, v in sd.items() if k.startswith("backbone.")}
missing, unexpected = model.load_state_dict(sd, strict=False)
```

### 1.4 位置埋め込みを別の解像度に合わせる

```python
from timm.layers import resample_abs_pos_embed

new = resample_abs_pos_embed(
    sd["pos_embed"],                      # (1, N_prefix + H*W, C)
    new_size=model.patch_embed.grid_size, # 例: (24, 24)
    num_prefix_tokens=model.num_prefix_tokens,   # no_embed_class=True なら 0
    interpolation="bicubic", antialias=True,
)
sd["pos_embed"] = new
```

推論時だけ解像度を変えるなら `dynamic_img_size=True` の方が簡単（基本編 第3節）。

### 1.5 hf-hub: と local-dir:

```python
timm.create_model("hf-hub:timm/vit_small_patch16_224.augreg_in21k", pretrained=True)
timm.create_model("hf-hub:MahmoodLab/UNI@<commit_hash>", pretrained=True, ...)   # @ でリビジョン固定
timm.create_model("local-dir:/path/to/model_dir", pretrained=True)
```

- `hf-hub:` は Hub の `config.json`（アーキテクチャ名・`model_args`・前処理）を読み、重みを取得する。キャッシュは HF の共通キャッシュ
- `local-dir:` はディレクトリ内の `config.json` と重み（`model.safetensors` / `pytorch_model.bin` など）を読む。`hf download --local-dir` で落としたフォルダをそのまま指定できる
- `config.json` に `model_args` があれば既定値として使われ、`create_model` の kwargs が優先される

### 1.6 学習用のデータ部品（create_transform / Mixup / RandAugment）

```python
from timm.data import create_transform, Mixup, resolve_model_data_config
from timm.loss import SoftTargetCrossEntropy

cfg = resolve_model_data_config(model)
train_tf = create_transform(
    **cfg, is_training=True,
    scale=(0.5, 1.0), hflip=0.5, vflip=0.5,
    color_jitter=0.2,
    auto_augment="rand-m9-mstd0.5-inc1",   # RandAugment（強さ9、ばらつき0.5）
    re_prob=0.25,                          # Random Erasing
)

mixup = Mixup(mixup_alpha=0.8, cutmix_alpha=1.0, prob=1.0, switch_prob=0.5,
              label_smoothing=0.1, num_classes=num_classes)
criterion = SoftTargetCrossEntropy()       # Mixup 後のラベルはソフトラベル
for x, y in loader:
    x, y = mixup(x, y)                     # バッチサイズは偶数にする
    loss = criterion(model(x), y)
```

| `auto_augment` の書式 | 意味 |
|:---|:---|
| `rand-m9-n2` | RandAugment、強さ9、1枚あたり2個 |
| `rand-m9-mstd0.5-inc1` | 強さに標準偏差0.5のばらつき、強さに応じて強くなる変換のみ |
| `augmix-m5-w4-d2` | AugMix |
| `original` / `v0` | AutoAugment（ImageNet ポリシー） |

- 病理画像では色の変換が強すぎると染色情報を壊す。RandAugment の強さは控えめにする
- `timm.data.create_loader` は timm 独自の `ImageDataset` と GPU 上での正規化（prefetcher）を前提にしている。自前の Dataset なら `create_transform` だけ使い、DataLoader は PyTorch のものを使う方が扱いやすい

### 1.7 層ごとの学習率減衰と weight decay の除外

```python
from timm.optim import param_groups_layer_decay, create_optimizer_v2

groups = param_groups_layer_decay(model, weight_decay=0.05, layer_decay=0.75)
optimizer = torch.optim.AdamW(groups, lr=5e-4)
# 各グループの lr は "lr_scale" として入っている。timm の scheduler はこれを掛けて使う
```

- 1D パラメータ（norm、bias）と `model.no_weight_decay()` の名前（ViT なら pos_embed、cls_token など）は自動で weight decay から外れる
- `lr_scale` を反映するのは timm の scheduler。PyTorch の scheduler を使うなら、自分で `g["lr"] = base_lr * g["lr_scale"]` にしておく
- 通常は `create_optimizer_v2(model, "adamw", lr=..., weight_decay=0.05, layer_decay=0.75)` で済む（基本編 第7節）

### 1.8 EMA（ModelEmaV3）

```python
from timm.utils import ModelEmaV3

ema = ModelEmaV3(model, decay=0.9998, use_warmup=True, device=None)
for step, (x, y) in enumerate(loader):
    ...
    optimizer.step()
    ema.update(model, step=step)       # step を渡すとウォームアップが効く

evaluate(ema.module)
torch.save({"model": model.state_dict(), "model_ema": ema.module.state_dict()}, "ckpt.pth")
```

- `use_warmup=True` だと学習初期は減衰を小さくして、EMA がランダム初期値に引きずられないようにする
- 上の形で保存すると、`load_checkpoint(use_ema=True)` が EMA 側を読む

### 1.9 特徴抽出の詳細

`features_only=True` は、CNN では `FeatureListNet` など、ViT では `FeatureGetterNet`（内部で `forward_intermediates` を呼ぶ）になる。

```python
f = timm.create_model("vit_small_patch16_224", features_only=True, out_indices=(-2, -1))
f.feature_info.channels(), f.feature_info.reduction()   # [384, 384], [16, 16]
[t.shape for t in f(x)]                                  # (B, 384, 14, 14) × 2
```

`forward_intermediates` のオプション（ViT）:

| 引数 | 内容 |
|:---|:---|
| `indices` | int なら最後の n ブロック、list ならそのブロック番号 |
| `output_fmt` | `"NCHW"`（既定、空間マップ）/ `"NLC"`（トークン列） |
| `return_prefix_tokens=True` | CLS / レジスタトークンも返す（`(空間, prefix)` のタプル） |
| `norm=True` | 最終 norm をかけて返す |
| `intermediates_only=True` | 最終出力を返さない |
| `stop_early=True` | 最後に指定したブロックで計算をやめる |

```python
outs = model.forward_intermediates(x, indices=4, intermediates_only=True)        # 最後の4ブロック
spatial, prefix = model.forward_intermediates(x, indices=[11], return_prefix_tokens=True,
                                              intermediates_only=True)[0]

kept = model.prune_intermediate_layers(indices=[8], prune_head=True)   # 以降のブロックとヘッドを削除して軽くする
```

### 1.10 モデルの改造

```python
model.set_grad_checkpointing(True)        # 勾配チェックポイント（学習時のメモリ削減）
model.reset_classifier(num_classes=4, global_pool="avg")
model.get_classifier()                    # ヘッドの層
model.group_matcher()                     # 層のまとまり（layer decay や段階的な解凍に使う）
model.no_weight_decay()                   # weight decay を外すパラメータ名
```

| 環境変数 | 内容 |
|:---|:---|
| `TIMM_FUSED_ATTN=0` | SDPA を使わず素の attention にする（切り分け用） |
| `TIMM_REENTRANT_CKPT` | 勾配チェックポイントを reentrant 方式にする。値は `bool(文字列)` で判定されるため `0` でも有効になる。無効にしたいときは変数自体を設定しない |

### 1.11 LoRA（peft と組み合わせる）

この環境には `peft 0.19.1` が入っている。timm の ViT の attention は `qkv` という `nn.Linear` なので、そこを対象にする。

```python
from peft import LoraConfig, get_peft_model

model = timm.create_model("hf-hub:MahmoodLab/UNI", pretrained=True, init_values=1e-5,
                          dynamic_img_size=True, num_classes=4)
cfg = LoraConfig(r=8, lora_alpha=16, lora_dropout=0.1,
                 target_modules=["qkv"], modules_to_save=["head"])
model = get_peft_model(model, cfg)
model.print_trainable_parameters()
```

- `modules_to_save` に入れた層（分類ヘッド）は全体を学習する
- 保存は `model.save_pretrained(dir)`（LoRA 部分のみ）。元の重みに統合するなら `model.merge_and_unload()`

### 1.12 エクスポートと Hub への公開

```python
m = timm.create_model("resnet50", pretrained=True, exportable=True).eval()
torch.onnx.export(m, torch.randn(1, 3, 224, 224), "resnet50.onnx", dynamo=True)

from timm.models import push_to_hf_hub
push_to_hf_hub(model, "my-user/my-vit", private=True, model_args={"init_values": 1e-5})
```

- `push_to_hf_hub` は重み（safetensors と bin）と `config.json`、モデルカードを上げる。以後 `hf-hub:my-user/my-vit` で読める
- `model_args` に入れた値は `config.json` に保存され、読み込み時の既定値になる
- Hub への公開は外部へのアップロード。公開範囲と中身を確認してから実行する

## Part 2. トラブル対応

### 2.1 Missing / Unexpected keys が大量に出る

- **症状**: `load_state_dict` で `Missing key(s): blocks.0.ls1.gamma ...` や `Unexpected key(s): backbone.blocks...` が大量に出る
- **確認**:
```python
sd = torch.load("ckpt.pth", map_location="cpu", weights_only=True)
print(list(sd.keys())[:10])                            # 最上位のキー（model / teacher / state_dict など）
inner = sd.get("model", sd)
print([k for k in inner][:10])
print(sorted(set(model.state_dict()) - set(inner))[:10])   # 足りないもの
print(sorted(set(inner) - set(model.state_dict()))[:10])   # 余計なもの
```
- **原因**:
  - 接頭辞の違い（`module.`、`backbone.`、`model.`、`_orig_mod.`、`encoder.`）
  - 構造の違い（`init_values` を付けていない → `ls1.gamma` が missing、`reg_tokens` の有無 → `reg_token` が unexpected）
  - 別の実装で保存された重み（DINOv2 公式、OpenCLIP など）
- **対処**: 接頭辞は `removeprefix` で外す。構造の違いは `create_model` の引数をモデルカードや `config.json` の `model_args` に合わせる。別実装の形式は `load_checkpoint(..., filter_fn=checkpoint_filter_fn)` で変換できるか試す。`strict=False` で読んだら必ず missing / unexpected を確認する（ヘッド以外が missing なら失敗）

### 2.2 pos_embed のサイズが合わない

- **症状**: `size mismatch for pos_embed: copying a param with shape torch.Size([1, 197, 768]) ... current model is torch.Size([1, 577, 768])`
- **確認**:
```python
print(sd["pos_embed"].shape, model.pos_embed.shape, model.patch_embed.grid_size, model.num_prefix_tokens)
```
- **原因**: 重みの学習時と `img_size` / `patch_size` / プレフィックストークン数（CLS、レジスタ）が違う
- **対処**: `filter_fn=checkpoint_filter_fn` で読む（自動で補間）か、1.4 の `resample_abs_pos_embed` で手動で合わせる。推論だけなら元の `img_size` で作って `dynamic_img_size=True` にする。トークン数（197 と 201 のような差）が合わないなら `reg_tokens` / `class_token` / `no_embed_class` の設定が違う

### 2.3 オフラインの計算ノードで重みを取れない

- **症状**: `create_model(..., pretrained=True)` で止まる、`LocalEntryNotFoundError`、`OSError: We couldn't connect to 'https://huggingface.co'`
- **確認**:
```python
print(model.pretrained_cfg.get("hf_hub_id"), model.pretrained_cfg.get("url"))   # pretrained=False で作って確認
```
```bash
hf cache ls | grep -i vit_small                          # キャッシュにあるか
echo $HF_HOME $HF_HUB_CACHE $HF_HUB_OFFLINE
```
- **原因**: timm の重みの多くは HF Hub（`timm/<モデル名>.<タグ>`）から取る。キャッシュにない、または計算ノードで `HF_HOME` が別の場所を指している。`hf_hub_id` がない古いモデルは `url` から取り、`~/.cache/torch/hub/checkpoints` に保存される
- **対処**: ログインノードで一度 `create_model(..., pretrained=True)` を実行するか `hf download timm/vit_small_patch16_224.augreg_in21k` でキャッシュに入れる。ジョブでは `HF_HUB_OFFLINE=1` を設定し、`HF_HOME` をログインノードと同じにする。確実にしたいなら `hf download --local-dir` で落として `local-dir:` または `pretrained_cfg_overlay=dict(file=...)` で読む

### 2.4 精度が極端に低い（正規化や前処理の違い）

- **症状**: 線形評価や kNN の精度がランダムに近い、論文値より大きく低い
- **確認**:
```python
print(resolve_model_data_config(model))           # モデルが期待する mean/std/size
print(train_tf)                                    # 実際に使っている前処理
x, _ = next(iter(loader)); print(x.mean((0, 2, 3)), x.std((0, 2, 3)), x.dtype, x.shape)
```
- **原因**: ViT の多く（augreg など）は mean=std=0.5 なのに ImageNet の値で正規化している、0〜255 のまま入力している、RGB と BGR（OpenCV）が逆、学習時の解像度・倍率と違う
- **対処**: `create_transform(**resolve_model_data_config(model))` を使う。病理基盤モデルはモデルカードの前処理（多くは ImageNet の mean/std、224px、20x）に合わせる。OpenCV で読んだら `cv2.cvtColor(img, cv2.COLOR_BGR2RGB)`

### 2.5 特徴量の次元・形が想定と違う

- **症状**: 線形層で `mat1 and mat2 shapes cannot be multiplied`、特徴量が `(B, 197, 768)` や `(B, 1536)` になっている
- **確認**:
```python
print(model.num_features, getattr(model, "head_hidden_size", None), model.global_pool)
with torch.inference_mode():
    print(model(torch.randn(1, 3, 224, 224)).shape)
    print(model.forward_features(torch.randn(1, 3, 224, 224)).shape)
```
- **原因**: `forward_features` はプーリング前。`global_pool="catavgmax"` など（平均と最大を連結するので2倍になる）。モデルカードが CLS とパッチ平均を連結して使う（例: 2倍の次元）
- **対処**: 埋め込みは `model(x)`（`num_classes=0`）か `forward_head(feats, pre_logits=True)` で取る。次元は `model.num_features` から決め、ハードコードしない。モデルカードの特徴抽出コードに合わせる

### 2.6 create_model が遅い / メモリを大量に使う

- **症状**: 大きなモデル（ViT-g、UNI2-h など）の作成に数十秒以上かかる、CPU メモリが倍以上使われる
- **確認**:
```python
import time; t = time.perf_counter()
m = timm.create_model(name, pretrained=False); print("init", time.perf_counter() - t)
```
- **原因**: ランダム初期化してから重みを上書きするため、初期化と読み込みの2回分の時間とメモリがかかる。`.bin`（pickle）を読むと safetensors より遅い
- **対処**: 重みは safetensors を使う（Hub に両方あれば timm は safetensors を優先する）。複数プロセスで同時に作らず、1プロセスで作って DDP に渡す。推論だけならメモリの少ない段階で `.half()` / `.to(device)` する

### 2.7 set_grad_checkpointing でエラー・効果がない

- **症状**: `NotImplementedError` や `AssertionError: gradient checkpointing not supported`、有効にしてもメモリが減らない
- **確認**:
```python
print(type(model).__mro__[0].__name__, hasattr(model, "set_grad_checkpointing"))
print(getattr(model, "grad_checkpointing", None), model.training)
```
- **原因**: 一部のモデルは未対応。ViT では `forward_features` 内で有効になるため、`forward_intermediates` の経路や、凍結して `torch.no_grad()` の中で使っている場合は効果がない。peft でラップした後だと呼び出し先が変わる
- **対処**: 対応モデルか確認する。ラップ（peft、DDP、compile）の前に `set_grad_checkpointing(True)` を呼ぶ。バックボーンを凍結しているなら不要（活性を保持しない）

### 2.8 in_chans を変えたら精度が出ない

- **症状**: `in_chans=1` や `in_chans=4` で作ったモデルの精度が低い
- **確認**:
```python
print(model.pretrained_cfg["first_conv"])
w = dict(model.named_parameters())[model.pretrained_cfg["first_conv"] + ".weight"]
print(w.shape, w.abs().mean())
```
- **原因**: timm は最初の層の重みを RGB の平均・繰り返しで作り直す。RGB とかけ離れた入力（蛍光の多チャンネルなど）では事前学習が効きにくい
- **対処**: RGB で表せるなら3チャンネルに変換して入れる。多チャンネルなら最初の層だけ学習率を上げる、チャンネルごとに共有のエンコーダを通すなどを検討する

### 2.9 gated モデルが 401 / 403 になる

- **症状**: `hf-hub:MahmoodLab/...` で `GatedRepoError` / `401 Client Error`
- **確認**:
```bash
hf auth whoami
hf auth list
python -c "from huggingface_hub import model_info; print(model_info('MahmoodLab/UNI').gated)"
```
- **原因**: 申請していない・承認前、トークンが別アカウント、ジョブ内で `HF_TOKEN` が見えていない
- **対処**: [huggingface_hub_deep.md](huggingface_hub_deep.md) の 2.1 を参照

### 2.10 Mixup でエラー

- **症状**: `AssertionError: Batch size should be even when using this`、または損失の形が合わない
- **確認**:
```python
print(x.shape[0], y.shape, y.dtype)
```
- **原因**: Mixup / CutMix は奇数バッチを扱えない（最後の半端なバッチ）。出力のラベルは `(B, num_classes)` のソフトラベルなので `CrossEntropyLoss` に整数ラベル前提のオプションを付けていると合わない
- **対処**: DataLoader に `drop_last=True`。損失は `SoftTargetCrossEntropy`（または PyTorch の `CrossEntropyLoss` に確率ラベルを渡す）。評価時は Mixup を通さない

# timm チートシート

> 対象: timm 1.0 系（このマシンの `.venv` は 1.0.28〜1.0.29）
> 公式: https://huggingface.co/docs/timm ・ https://github.com/huggingface/pytorch-image-models

## 1. 何をするライブラリか

画像モデル（ViT、ConvNeXt、ResNet、EfficientNet など）とその事前学習重みを、名前ひとつで呼び出せるライブラリ。
UNI などの病理基盤モデルも、Hugging Face Hub 経由で timm のモデルとして読み込める。

## 2. モデルを探す

```python
import timm

timm.list_models("vit_*")                           # 名前で検索
timm.list_models("*dino*", pretrained=True)         # 重みがあるものだけ
timm.list_pretrained("convnext*")                   # 重み付きのタグまで表示（例: convnext_tiny.fb_in22k）
```

名前の形式: `アーキテクチャ.事前学習タグ`（例: `vit_base_patch16_224.augreg_in21k`）。タグを省略すると既定の重みになる。

## 3. モデルを作る

```python
model = timm.create_model("resnet50", pretrained=True)
model = timm.create_model("vit_base_patch16_224", pretrained=True, num_classes=5)   # 分類ヘッドを付け替える
model = timm.create_model("convnext_tiny", pretrained=True, num_classes=0)         # ヘッドなし（特徴量を出力）
model = timm.create_model("resnet50", pretrained=True, in_chans=1)                 # 入力チャンネル数を変える
model = timm.create_model("vit_small_patch16_224", pretrained=False)               # 重みなし
```

### よく使う引数

| 引数 | 説明 |
|:---|:---|
| `pretrained` | 事前学習重みを読み込む |
| `num_classes` | 出力クラス数。`0` にすると分類ヘッドを外す |
| `global_pool` | `"avg"`, `"token"`（ViT の CLS）, `""`（プーリングなし） |
| `drop_rate` / `drop_path_rate` | Dropout / Stochastic depth |
| `img_size` | ViT の入力サイズを変える（位置埋め込みは補間される） |
| `dynamic_img_size` | ViT で入力サイズを可変にする |
| `pretrained_cfg_overlay` | 重みのファイルを差し替える（例: `dict(file="weights.pth")`） |
| `checkpoint_path` | ローカルの重みファイルを読む |

## 4. Hugging Face Hub のモデルを読む（UNI など）

```python
model = timm.create_model(
    "hf-hub:MahmoodLab/UNI", pretrained=True,
    init_values=1e-5, dynamic_img_size=True,
)
```

- `hf-hub:<リポジトリ名>` で HF Hub から読み込む。gated モデルは事前に申請と `hf auth login` が必要（[huggingface_hub.md](huggingface_hub.md)）
- モデルごとに必要な引数が違う（UNI2-h や Virchow は追加の引数が必要）。**モデルカードの読み込み例をそのまま使う**
- オフラインで使うときは、事前にダウンロードしておき、`HF_HUB_OFFLINE=1` で実行する

## 5. 前処理を合わせる

```python
cfg = timm.data.resolve_data_config({}, model=model)   # mean, std, input_size, interpolation など
transform = timm.data.create_transform(**cfg)          # 評価用
train_tf = timm.data.create_transform(**cfg, is_training=True)
print(cfg)   # 例: {'input_size': (3, 224, 224), 'mean': (0.485, ...), 'std': (...), 'crop_pct': 0.9, ...}
```

事前学習のときと違う mean / std で正規化すると、特徴量の質が下がる。

## 6. 特徴量を取り出す

```python
x = torch.randn(2, 3, 224, 224)

model = timm.create_model("vit_base_patch16_224", pretrained=True, num_classes=0)
emb = model(x)                              # (2, 768) プーリング後のベクトル

feats = model.forward_features(x)           # プーリング前（ViT なら (2, 197, 768)）
emb = model.forward_head(feats, pre_logits=True)

model.num_features                          # 出力次元数
model.reset_classifier(num_classes=3)       # 後からヘッドを付け替える
```

### CNN の中間層

```python
model = timm.create_model("resnet50", pretrained=True, features_only=True, out_indices=(1, 2, 3))
fmaps = model(x)                            # 解像度の違う特徴マップのリスト
model.feature_info.channels()               # 各層のチャンネル数
```

### ViT の中間層

```python
outs = model.forward_intermediates(x, indices=[-4, -1], intermediates_only=True)
```

## 7. 学習の部品

```python
from timm.optim import create_optimizer_v2
from timm.scheduler import create_scheduler_v2
from timm.loss import LabelSmoothingCrossEntropy
from timm.data import Mixup

optimizer = create_optimizer_v2(model, opt="adamw", lr=1e-4, weight_decay=0.05)
# layer_decay=0.75 で層ごとに学習率を下げる（ViT のファインチューニングでよく使う）
```

## 8. 一部の層だけ学習する

```python
model.requires_grad_(False)                         # 全部凍結
for p in model.get_classifier().parameters():       # ヘッドだけ学習
    p.requires_grad = True

for blk in model.blocks[-2:]:                       # ViT の最後の2ブロックも学習
    blk.requires_grad_(True)
```

## 9. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `RuntimeError: Unknown model` | 名前の間違い、または timm が古い。`timm.list_models` で確認する |
| 計算ノードでダウンロードが止まる | ネットワークの制限。ログインノードで事前にダウンロードする |
| 入力サイズを変えたらエラー（ViT） | `img_size=` を指定するか、`dynamic_img_size=True` にする |
| 精度が思ったより出ない | 前処理がモデルと合っていない。第5節で確認する |
| `state_dict` のキーが合わない | 別のアーキテクチャ・設定で作っている。モデルカードの `create_model` の引数をそのまま使う |
| gated モデルで 401 | HF でのアクセス申請、または `hf auth login` をしていない |

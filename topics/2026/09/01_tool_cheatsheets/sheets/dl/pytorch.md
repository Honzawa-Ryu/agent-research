# PyTorch / torchvision チートシート

> 対象: PyTorch 2.x（このマシンの `.venv` は `torch 2.11.0+cu130` / `torchvision 0.26.0`）
> 公式: https://pytorch.org/docs/stable/ ・ https://pytorch.org/vision/stable/

## 1. テンソルの基本

```python
import torch

x = torch.tensor([[1., 2.], [3., 4.]])
x = torch.zeros(2, 3); torch.ones(2, 3); torch.randn(2, 3); torch.arange(10)
x = torch.from_numpy(arr)            # NumPy とメモリを共有する
arr = x.detach().cpu().numpy()       # GPU 上・勾配ありのテンソルは detach().cpu() してから

x.shape, x.dtype, x.device
x.float(); x.half(); x.to(torch.bfloat16); x.long()
x.view(-1, 4); x.reshape(-1, 4)      # view は連続メモリが必要。困ったら reshape
x.permute(0, 2, 3, 1)                # 軸の入れ替え（NCHW → NHWC など）
x.unsqueeze(0); x.squeeze(0)         # 次元の追加・削除
torch.cat([a, b], dim=0); torch.stack([a, b], dim=0)
```

## 2. デバイス

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model.to(device); x = x.to(device, non_blocking=True)

torch.cuda.device_count()
torch.cuda.get_device_name(0)
torch.cuda.memory_allocated() / 1e9           # 使用中のメモリ (GB)
torch.cuda.max_memory_allocated() / 1e9       # ピーク
torch.cuda.empty_cache()                      # キャッシュを解放（使用量が減るわけではない）
```

## 3. Dataset と DataLoader

```python
from torch.utils.data import Dataset, DataLoader

class MyDataset(Dataset):
    def __init__(self, items, transform=None):
        self.items, self.transform = items, transform
    def __len__(self):
        return len(self.items)
    def __getitem__(self, i):
        img, label = load(self.items[i])
        if self.transform:
            img = self.transform(img)
        return img, label

loader = DataLoader(
    ds, batch_size=64, shuffle=True,
    num_workers=8,              # Slurm の --cpus-per-task 以下にする
    pin_memory=True,            # GPU への転送が速くなる
    persistent_workers=True,    # エポックごとにワーカーを作り直さない
    drop_last=True,             # 最後の半端なバッチを捨てる（BatchNorm 対策）
)
```

| よく使う部品 | 用途 |
|:---|:---|
| `Subset(ds, indices)` | 一部だけ使う |
| `random_split(ds, [0.8, 0.2])` | ランダムに分割 |
| `ConcatDataset([a, b])` | 連結 |
| `WeightedRandomSampler` | クラスの偏りを補正 |
| `collate_fn=` | 可変長のデータをまとめる方法を自分で書く |

## 4. torchvision の transforms（v2）

```python
from torchvision.transforms import v2 as T

train_tf = T.Compose([
    T.ToImage(),                                   # PIL / ndarray → Tensor（uint8）
    T.RandomResizedCrop(224, scale=(0.8, 1.0)),
    T.RandomHorizontalFlip(), T.RandomVerticalFlip(),
    T.ColorJitter(0.1, 0.1, 0.1, 0.05),
    T.ToDtype(torch.float32, scale=True),          # 0〜1 にスケール
    T.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])
```

- 事前学習モデルを使うときは、そのモデルの mean / std に合わせる（timm では `resolve_data_config` で取得できる。[timm.md](timm.md)）
- 病理画像は向きに意味がないことが多いので、上下反転や90°回転もよく使う

## 5. 学習ループの雛形

```python
model = Net().to(device)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, weight_decay=0.05)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
criterion = torch.nn.CrossEntropyLoss()

for epoch in range(epochs):
    model.train()
    for x, y in train_loader:
        x, y = x.to(device, non_blocking=True), y.to(device, non_blocking=True)
        optimizer.zero_grad(set_to_none=True)
        with torch.autocast("cuda", dtype=torch.bfloat16):   # 混合精度
            loss = criterion(model(x), y)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
    scheduler.step()

    model.eval()
    with torch.inference_mode():                              # 評価時は勾配を計算しない
        for x, y in val_loader:
            ...
```

| 項目 | ポイント |
|:---|:---|
| `model.train()` / `model.eval()` | Dropout と BatchNorm の挙動が切り替わる。評価前に必ず `eval()` |
| `torch.inference_mode()` | `no_grad()` より速い。評価・特徴抽出で使う |
| bf16 の autocast | A100 / H100 / A6000 などで使える。`GradScaler` は不要 |
| fp16 の autocast | 古い GPU 向け。`torch.amp.GradScaler("cuda")` と組み合わせる |

### fp16 + GradScaler

```python
scaler = torch.amp.GradScaler("cuda")
with torch.autocast("cuda", dtype=torch.float16):
    loss = criterion(model(x), y)
scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

## 6. 保存と読み込み

```python
# 推奨: state_dict を保存する
torch.save({
    "model": model.state_dict(),
    "optimizer": optimizer.state_dict(),
    "epoch": epoch,
}, "ckpt.pt")

ckpt = torch.load("ckpt.pt", map_location="cpu")        # 2.6 以降は weights_only=True が既定
model.load_state_dict(ckpt["model"])
optimizer.load_state_dict(ckpt["optimizer"])

model.load_state_dict(sd, strict=False)                  # キーが一部違っても読む（戻り値で差分を確認する）
```

- `torch.load` は 2.6 以降、既定で `weights_only=True`。任意の Python オブジェクトを含む古いファイルは `weights_only=False` にする（信頼できるファイルだけ）
- `torch.compile` や `DataParallel` を使ったモデルは、キーに `_orig_mod.` / `module.` が付く。読み込むときに取り除く

## 7. 再現性

```python
import random, numpy as np
def seed_everything(seed):
    random.seed(seed); np.random.seed(seed)
    torch.manual_seed(seed); torch.cuda.manual_seed_all(seed)

torch.backends.cudnn.benchmark = False
torch.use_deterministic_algorithms(True)   # 決定論的でない演算があるとエラーになる
# 環境変数 CUBLAS_WORKSPACE_CONFIG=:4096:8 も必要になることがある
```

DataLoader のワーカーのシードは `generator=torch.Generator().manual_seed(seed)` と `worker_init_fn` で固定する。

## 8. 特徴抽出（凍結したバックボーン）

```python
model.eval().requires_grad_(False)
feats = []
with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
    for x, _ in loader:
        feats.append(model(x.to(device)).float().cpu())
feats = torch.cat(feats)
```

## 9. 高速化

| 方法 | コード |
|:---|:---|
| TF32 を許可（Ampere 以降） | `torch.set_float32_matmul_precision("high")` |
| コンパイル | `model = torch.compile(model)`（最初の数ステップは遅い） |
| channels_last | `model.to(memory_format=torch.channels_last)` |
| cudnn の自動チューニング | `torch.backends.cudnn.benchmark = True`（入力サイズが固定のとき） |

## 10. よくあるハマりどころ

| 症状 | 原因と対処 |
|:---|:---|
| `CUDA out of memory` | バッチサイズを下げる、autocast を使う、評価で `inference_mode` を使う。`nvidia-smi` でほかのプロセスも確認する |
| `Expected all tensors to be on the same device` | モデルかデータの片方を `.to(device)` していない |
| `torch.cuda.is_available()` が False | CPU 版の torch が入っている、`--gres` がない、Apptainer の `--nv` がない（[uv.md](../infra/uv.md), [apptainer.md](../infra/apptainer.md)） |
| 評価の精度が毎回違う | `model.eval()` を忘れている |
| メモリが少しずつ増える | `loss` をそのままリストに溜めている。`loss.item()` で数値にする |
| DataLoader が固まる / 遅い | `num_workers` が多すぎる、共有メモリ（`/dev/shm`）が足りない。`num_workers=0` で切り分ける |
| `weights_only` でエラー | 第6節を参照 |
| `Missing key(s)` / `Unexpected key(s)` | `module.` / `_orig_mod.` の接頭辞。または別のモデル定義 |

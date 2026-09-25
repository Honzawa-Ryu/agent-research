# PyTorch / torchvision 詳細編（応用・トラブル対応）

> 基本編: [pytorch.md](pytorch.md)
> 対象バージョン: `torch 2.11.0+cu130` / `torchvision 0.26.0`（このマシンの `.venv` で API を確認済み）

## Part 1. 応用・上級機能

### 1.1 DDP（torchrun + Slurm）

`torchrun` が各プロセスに環境変数を渡す。スクリプト側はそれを読むだけにする。

| 変数 | 意味 |
|:---|:---|
| `RANK` | 全体での順位（0 〜 WORLD_SIZE-1） |
| `LOCAL_RANK` | ノード内での順位。GPU 番号に使う |
| `WORLD_SIZE` | 全プロセス数 |
| `MASTER_ADDR` / `MASTER_PORT` | rank 0 の場所 |

```python
import os, torch, torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, DistributedSampler

local_rank = int(os.environ["LOCAL_RANK"])
torch.cuda.set_device(local_rank)
dist.init_process_group("nccl", device_id=torch.device("cuda", local_rank))
rank, world = dist.get_rank(), dist.get_world_size()
is_main = rank == 0

model = DDP(model.cuda(), device_ids=[local_rank])
sampler = DistributedSampler(train_ds, shuffle=True, drop_last=True)
loader = DataLoader(train_ds, batch_size=bs_per_gpu, sampler=sampler, num_workers=8, pin_memory=True)

for epoch in range(epochs):
    sampler.set_epoch(epoch)              # 忘れると毎エポック同じ順番になる
    for x, y in loader:
        ...
    if is_main:                           # 保存・ログは rank 0 だけ
        torch.save(model.module.state_dict(), "ckpt.pt")   # .module で DDP を外す
    dist.barrier()

dist.destroy_process_group()
```

- `DataLoader` に `sampler=` を渡したら `shuffle=` は指定しない
- 評価指標を全 GPU で集計するときは `dist.all_reduce(t, op=dist.ReduceOp.SUM)`
- 実効バッチサイズは `bs_per_gpu × WORLD_SIZE`。学習率はそれに合わせて見直す

Slurm のジョブスクリプト（1ノードあたり1タスク、その中で torchrun が GPU 数だけプロセスを立てる）:

```bash
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=32

export MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n 1)
export MASTER_PORT=29500

srun torchrun \
  --nnodes="$SLURM_NNODES" --nproc-per-node=4 \
  --rdzv-backend=c10d --rdzv-endpoint="$MASTER_ADDR:$MASTER_PORT" \
  --rdzv-id="$SLURM_JOB_ID" \
  train.py
# Apptainer の場合: srun apptainer exec --nv img.sif torchrun ...
```

1ノードだけなら `torchrun --standalone --nproc-per-node=4 train.py` で足りる。

### 1.2 FSDP2（fully_shard）

モデルが1枚の GPU に載らないとき用。パラメータ・勾配・optimizer 状態を GPU 間で分割する。

```python
from torch.distributed.fsdp import fully_shard, MixedPrecisionPolicy

mp = MixedPrecisionPolicy(param_dtype=torch.bfloat16, reduce_dtype=torch.float32)
for blk in model.blocks:                  # 先にブロック単位で分割し、最後に全体
    fully_shard(blk, mp_policy=mp)
fully_shard(model, mp_policy=mp)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)   # fully_shard の後で作る
```

- ViT-L 程度までなら DDP で足りることが多い。まず DDP + bf16 + 勾配チェックポイントを試す
- 保存は `torch.distributed.checkpoint`（DCP）を使う。`state_dict()` をそのまま `torch.save` すると分割された状態になる

### 1.3 勾配累積

GPU メモリが足りず、大きなバッチを1回で処理できないとき。

```python
accum = 4
for i, (x, y) in enumerate(loader):
    with torch.autocast("cuda", dtype=torch.bfloat16):
        loss = criterion(model(x), y) / accum          # 平均になるように割る
    loss.backward()
    if (i + 1) % accum == 0:
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
```

DDP では、累積の途中の step で通信を省くと速い:

```python
from contextlib import nullcontext
ctx = model.no_sync() if (i + 1) % accum != 0 else nullcontext()
with ctx:
    loss.backward()
```

BatchNorm の統計は小さいバッチ単位で計算される点は変わらない。

### 1.4 勾配チェックポイント（activation checkpointing）

中間の活性を保存せず、backward 時に再計算する。メモリが減り、計算は 20〜30% ほど増える。

```python
from torch.utils.checkpoint import checkpoint

class Net(torch.nn.Module):
    def forward(self, x):
        for blk in self.blocks:
            x = checkpoint(blk, x, use_reentrant=False)   # use_reentrant は明示する
        return x
```

- `use_reentrant` を省略すると警告が出る（2.11 では既定は True のまま）。`False` を推奨
- timm のモデルは `model.set_grad_checkpointing(True)` で有効にできる（[timm_deep.md](timm_deep.md)）

### 1.5 torch.compile の使い分け

| mode | 特徴 |
|:---|:---|
| `None`（既定） | コンパイル時間とのバランス型 |
| `"reduce-overhead"` | CUDA Graphs で Python のオーバーヘッドを減らす。小さいバッチで効く。メモリが増える |
| `"max-autotune"` | 行列積のカーネルを探索する。コンパイルが長い |
| `"max-autotune-no-cudagraphs"` | 上から CUDA Graphs を除いたもの |

```python
model = torch.compile(model)                        # まずは既定で
model = torch.compile(model, dynamic=False)         # 入力形状が固定なら
model = torch.compile(model, fullgraph=True)        # グラフが途切れたらエラーにする（原因調査用）
```

調べ方:

```bash
TORCH_LOGS="graph_breaks,recompiles" python train.py    # グラフの切れ目と再コンパイルの理由
```

```python
exp = torch._dynamo.explain(model)(x)               # グラフの数・切れ目の理由
print(exp.graph_break_count, exp.break_reasons)
```

- コンパイル結果は `TORCHINDUCTOR_CACHE_DIR`（既定 `/tmp/torchinductor_<ユーザー名>`）に保存される。計算ノードの `/tmp` はジョブごとに消えることがあるので、共有ディスクに向けると2回目以降が速くなる
- 再コンパイルは既定で8回まで（`torch._dynamo.config.recompile_limit`）。超えると eager 実行に戻る

### 1.6 SDPA（scaled_dot_product_attention）

```python
import torch.nn.functional as F
from torch.nn.attention import sdpa_kernel, SDPBackend

out = F.scaled_dot_product_attention(q, k, v, attn_mask=None, dropout_p=0.0, is_causal=False)

with sdpa_kernel([SDPBackend.FLASH_ATTENTION, SDPBackend.EFFICIENT_ATTENTION]):
    out = F.scaled_dot_product_attention(q, k, v)   # 使うバックエンドを限定する
```

| バックエンド | 条件の目安 |
|:---|:---|
| `FLASH_ATTENTION` | fp16 / bf16、Ampere 以降 |
| `EFFICIENT_ATTENTION` | fp32 でも使える。マスクあり可 |
| `CUDNN_ATTENTION` | cuDNN 実装 |
| `MATH` | どこでも動くが遅く、メモリも多い |

限定したバックエンドが条件に合わないと `RuntimeError: No available kernel` になる。timm の ViT は内部で SDPA を使う。

### 1.7 プロファイリング

```python
from torch.profiler import profile, schedule, ProfilerActivity, record_function, tensorboard_trace_handler

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(wait=1, warmup=1, active=3, repeat=1),
    on_trace_ready=tensorboard_trace_handler("logs/prof"),
    record_shapes=True, profile_memory=True,
) as prof:
    for step, (x, y) in enumerate(loader):
        with record_function("forward"):
            loss = criterion(model(x.cuda()), y.cuda())
        loss.backward(); optimizer.step(); optimizer.zero_grad()
        prof.step()
        if step >= 5:
            break

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=20))
```

GPU メモリの内訳を時系列で見る（メモリスナップショット）:

```python
torch.cuda.memory._record_memory_history(max_entries=100_000)
# ... 数ステップ学習 ...
torch.cuda.memory._dump_snapshot("mem.pickle")
torch.cuda.memory._record_memory_history(enabled=None)    # 記録を止める
# mem.pickle を https://pytorch.org/memory_viz にドラッグして見る（ブラウザ内で処理される）
```

```python
print(torch.cuda.memory_summary())                  # 確保済み・予約済み・断片化の概要
```

### 1.8 チェックポイントと再開（完全版）

乱数の状態・scheduler・scaler まで保存すると、途中から同じ条件で再開できる。

```python
import random, numpy as np

def save_ckpt(path, epoch):
    torch.save({
        "epoch": epoch,
        "model": model.state_dict(),
        "optimizer": optimizer.state_dict(),
        "scheduler": scheduler.state_dict(),
        "scaler": scaler.state_dict() if scaler else None,
        "ema": ema.state_dict() if ema else None,
        "rng": {
            "python": random.getstate(),
            "numpy": np.random.get_state(),
            "torch": torch.get_rng_state(),
            "cuda": torch.cuda.get_rng_state_all(),
        },
    }, path + ".tmp")
    os.replace(path + ".tmp", path)          # 書き込み途中で落ちても壊れたファイルを残さない

ckpt = torch.load(path, map_location="cpu", weights_only=False)   # RNG 状態は weights_only=True では読めない
model.load_state_dict(ckpt["model"])
optimizer.load_state_dict(ckpt["optimizer"])
scheduler.load_state_dict(ckpt["scheduler"])
random.setstate(ckpt["rng"]["python"]); np.random.set_state(ckpt["rng"]["numpy"])
torch.set_rng_state(ckpt["rng"]["torch"]); torch.cuda.set_rng_state_all(ckpt["rng"]["cuda"])
start_epoch = ckpt["epoch"] + 1
```

- `optimizer.load_state_dict` はモデルを GPU に移して optimizer を作り直した後に呼ぶ
- Slurm の時間制限で落ちる場合は、`#SBATCH --signal=B:USR1@300` で終了 5 分前にシグナルを受け取り、保存してから終わる方法もある

### 1.9 EMA（指数移動平均）

```python
from torch.optim.swa_utils import AveragedModel, get_ema_multi_avg_fn

ema = AveragedModel(model, multi_avg_fn=get_ema_multi_avg_fn(0.999), use_buffers=True)
for x, y in loader:
    ...
    optimizer.step()
    ema.update_parameters(model)

evaluate(ema.module)                         # 評価は EMA 側で
```

- `use_buffers=True` にすると BatchNorm の running_mean なども平均される
- timm の `ModelEmaV3` はウォームアップ付きの減衰に対応している（[timm_deep.md](timm_deep.md)）

### 1.10 パラメータグループ（norm / bias に weight decay をかけない）

```python
decay, no_decay = [], []
for name, p in model.named_parameters():
    if not p.requires_grad:
        continue
    (no_decay if p.ndim <= 1 or name.endswith(".bias") else decay).append(p)

optimizer = torch.optim.AdamW([
    {"params": decay, "weight_decay": 0.05},
    {"params": no_decay, "weight_decay": 0.0},
], lr=1e-4)

# 部分ごとに学習率を変える例（バックボーンは小さく）
optimizer = torch.optim.AdamW([
    {"params": model.backbone.parameters(), "lr": 1e-5},
    {"params": model.head.parameters(), "lr": 1e-3},
])
```

ViT の pos_embed / cls_token も weight decay から外すのが一般的（`p.ndim <= 1` では外れないので名前で判定する）。

### 1.11 学習率のウォームアップ

```python
from torch.optim.lr_scheduler import LinearLR, CosineAnnealingLR, SequentialLR, LambdaLR
import math

warm = LinearLR(optimizer, start_factor=0.01, total_iters=warmup_steps)
cos = CosineAnnealingLR(optimizer, T_max=total_steps - warmup_steps, eta_min=1e-6)
scheduler = SequentialLR(optimizer, schedulers=[warm, cos], milestones=[warmup_steps])

# 1つの式で書く場合
def lr_lambda(step):
    if step < warmup_steps:
        return step / max(1, warmup_steps)
    t = (step - warmup_steps) / max(1, total_steps - warmup_steps)
    return 0.5 * (1 + math.cos(math.pi * t))
scheduler = LambdaLR(optimizer, lr_lambda)
```

step 単位で作ったら `scheduler.step()` を毎 step 呼ぶ（エポック単位と混ぜない）。

### 1.12 フックで中間出力を取る

```python
feats = {}
def save_to(name):
    def hook(module, inp, out):
        feats[name] = out.detach()
    return hook

h = model.layer3.register_forward_hook(save_to("layer3"))
model(x)
h.remove()                                   # 使い終わったら外す
```

torchvision には FX で中間層を取り出す関数もある:

```python
from torchvision.models.feature_extraction import create_feature_extractor, get_graph_node_names
get_graph_node_names(model)[1]               # 取り出せるノード名
fx = create_feature_extractor(model, return_nodes={"layer3": "l3", "layer4": "l4"})
out = fx(x)                                  # {"l3": ..., "l4": ...}
```

### 1.13 IterableDataset とワーカーの分担

WSI のパッチを順に読み出すなど、長さが決まらないデータ向け。ワーカーごとに担当を分けないと、同じデータが重複する。

```python
from torch.utils.data import IterableDataset, get_worker_info

class Stream(IterableDataset):
    def __init__(self, files):
        self.files = files
    def __iter__(self):
        info = get_worker_info()
        files = self.files if info is None else self.files[info.id::info.num_workers]
        # DDP でも使うなら rank でさらに分ける: files[rank::world]
        for f in files:
            yield from read_patches(f)
```

`shuffle=True` や `sampler` は IterableDataset では使えない。シャッフルは `__iter__` の中で行う。

### 1.14 数値精度（TF32 / bf16）

| 設定 | 効果 |
|:---|:---|
| `torch.set_float32_matmul_precision("high")` | fp32 の行列積に TF32 を使う（Ampere 以降で高速） |
| `torch.backends.cudnn.allow_tf32` | 畳み込みの TF32（既定 True） |
| autocast bf16 | 指数部が fp32 と同じなので溢れにくい。仮数部は短い |
| autocast fp16 | 最大値 65504。溢れやすいので GradScaler が必要 |

- 損失の計算・softmax・正規化は autocast が自動で fp32 にする演算が多い。自作の損失で `exp` を使う場合は `.float()` にしてから計算する
- 評価値が fp32 実行と微妙に違うのは TF32 / bf16 の丸めによる。厳密に比べたいときは TF32 を切る

### 1.15 torch.func（関数変換）

```python
from torch.func import functional_call, vmap, grad

out = functional_call(model, params_dict, (x,))         # パラメータを差し替えて順伝播
per_sample_grad = vmap(grad(loss_fn), in_dims=(None, 0, 0))(params, xs, ys)  # サンプルごとの勾配
```

サンプルごとの勾配や、メタ学習のようにパラメータを関数の引数として扱いたいときに使う。

### 1.16 torchvision: tv_tensors でマスク・ボックスも一緒に変換

```python
from torchvision import tv_tensors
from torchvision.transforms import v2 as T

img = tv_tensors.Image(img_tensor)                         # (C, H, W)
mask = tv_tensors.Mask(mask_tensor)                        # (H, W)
boxes = tv_tensors.BoundingBoxes(b, format="XYXY", canvas_size=img.shape[-2:])

tf = T.Compose([
    T.RandomResizedCrop(512), T.RandomHorizontalFlip(),
    T.ToDtype({tv_tensors.Image: torch.float32, "others": None}, scale=True),
    T.SanitizeBoundingBoxes(),                             # 画像の外に出たボックスを除く
])
img, mask, boxes = tf(img, mask, boxes)                    # 同じ幾何変換がかかる
```

- 色の変換（ColorJitter など）は Image にだけかかり、Mask にはかからない
- `SanitizeBoundingBoxes` はラベルも一緒に消すため、dict で `{"boxes": ..., "labels": ...}` の形で渡す
- `T.CutMix` / `T.MixUp` はバッチ（collate の後）にかける

### 1.17 torchvision: ImageFolder と重み API

```python
from torchvision.datasets import ImageFolder
ds = ImageFolder("data/train", transform=tf)          # data/train/<クラス名>/*.png
ds.classes, ds.class_to_idx                           # クラス名はフォルダ名のアルファベット順

from torchvision.models import get_model, get_model_weights, resnet50, ResNet50_Weights
w = ResNet50_Weights.DEFAULT                          # = IMAGENET1K_V2
model = resnet50(weights=w)
preprocess = w.transforms()                           # この重みに合った前処理（resize 232 → crop 224 など）
w.meta["categories"]                                  # クラス名
model = get_model("resnet50", weights="DEFAULT")      # 名前で作る
```

- 旧 API の `pretrained=True` は使わない
- 重みは `TORCH_HOME`（既定 `~/.cache/torch/hub/checkpoints`）に保存される

## Part 2. トラブル対応

### 2.1 CUDA out of memory（まだ空いているはずなのに落ちる）

- **症状**: `torch.OutOfMemoryError: CUDA out of memory. Tried to allocate ... GiB` が出る。メッセージ内の `reserved but unallocated` が大きい
- **確認**:
```bash
nvidia-smi                              # ほかのプロセスが同じ GPU を使っていないか
```
```python
print(torch.cuda.memory_summary())      # allocated と reserved の差が大きい → 断片化
print(torch.cuda.max_memory_allocated() / 1e9)
```
- **原因**: 入力サイズが毎回変わる（可変長・可変解像度）ことでメモリが断片化している。または活性が大きすぎる、評価で勾配を保持している
- **対処**:
  1. 断片化なら `export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`（Apptainer では `APPTAINERENV_` を付けるか `--env` で渡す）
  2. autocast（bf16）、勾配チェックポイント（1.4）、勾配累積（1.3）
  3. 評価は `torch.inference_mode()`。ループの外で `loss` や出力テンソルを保持していないか確認する
  4. 内訳が分からなければメモリスナップショット（1.7）でどこが確保しているか見る

### 2.2 loss が NaN / Inf になる

- **症状**: 数十〜数千 step 後に loss が `nan` になり、以後戻らない
- **確認**:
```python
if not torch.isfinite(loss):
    print(step, loss.item(), optimizer.param_groups[0]["lr"])
    for n, p in model.named_parameters():
        if p.grad is not None and not torch.isfinite(p.grad).all():
            print("bad grad:", n)
    raise SystemExit

gn = torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)   # 戻り値はクリップ前のノルム。急増していないか記録する

torch.autograd.set_detect_anomaly(True)   # NaN を出した演算を特定（非常に遅い。切り分け時だけ）
```
- **原因**: 学習率が大きい（ウォームアップなし）、fp16 の溢れ、`log(0)` や `x / 0`、入力データ自体に NaN、正規化の分母が 0
- **対処**:
  1. 入力をチェック: `torch.isfinite(x).all()`
  2. fp16 なら bf16 に切り替える。fp16 を続けるなら GradScaler を必ず使う
  3. 学習率を下げ、ウォームアップ（1.11）と `clip_grad_norm_` を入れる
  4. 自作の損失は `torch.log(x.clamp_min(1e-8))` や `F.log_softmax` など安定な形にする

### 2.3 CUDA error: device-side assert triggered

- **症状**: `RuntimeError: CUDA error: device-side assert triggered`。スタックトレースの場所が毎回違い、以後その GPU の処理がすべて失敗する
- **確認**:
```bash
CUDA_LAUNCH_BLOCKING=1 python train.py    # エラー箇所が正しい行に出る（遅くなる）
```
```python
print(y.min().item(), y.max().item(), num_classes)   # ラベルの範囲
```
```bash
python train.py --device cpu              # CPU で動かすと分かりやすいエラーになる
```
- **原因**: ほとんどがインデックスの範囲外。`CrossEntropyLoss` のラベルが `num_classes` 以上・負の値、`nn.Embedding` の範囲外、`BCELoss` の入力が 0〜1 を外れている
- **対処**: ラベルを 0〜C-1 に直す（1始まりのラベル、ignore 用の -1 など）。無視したいラベルは `CrossEntropyLoss(ignore_index=-100)` を使う。一度この例外が出たプロセスは使えないので再起動する

### 2.4 DataLoader が固まる / Bus error / 共有メモリ不足

- **症状**: 学習が進まない、`RuntimeError: DataLoader worker (pid X) is killed by signal: Bus error`、`unable to write to file </torch_...>: No space left on device`
- **確認**:
```bash
df -h /dev/shm                                # 共有メモリの空き
apptainer exec img.sif df -h /dev/shm         # コンテナ内から見た容量
```
```python
loader = DataLoader(ds, num_workers=0, ...)   # 0 で動くならワーカー周りの問題
```
- **原因**: ワーカーはテンソルを `/dev/shm` 経由で渡す。容量が小さい、`num_workers × prefetch_factor × バッチサイズ` が大きすぎる。`__getitem__` 内でスレッド数の多いライブラリ（OpenCV など）を使いデッドロックしている
- **対処**:
  1. `num_workers` と `prefetch_factor`（既定 2）を下げる
  2. 共有メモリを使わない方式に切り替える: `torch.multiprocessing.set_sharing_strategy("file_system")`
  3. OpenCV を使うなら `cv2.setNumThreads(0)` をワーカー内で呼ぶ
  4. HDF5 / openslide のファイルハンドルは `__init__` で開かず、`__getitem__` の中（各ワーカー）で開く

### 2.5 NCCL が止まる / タイムアウトする

- **症状**: DDP で最初の通信や途中で止まり、10分後に `Watchdog caught collective operation timeout` で落ちる
- **確認**:
```bash
export NCCL_DEBUG=INFO                        # 使っている通信経路（IB / Socket）とエラー
export TORCH_DISTRIBUTED_DEBUG=DETAIL         # どの collective で食い違ったか
nvidia-smi topo -m                            # GPU 間の接続
```
```python
print(f"rank={rank} step={step} batches={len(loader)}", flush=True)   # rank ごとの進み具合
```
- **原因**: rank によって呼ぶ collective の回数が違う（例: rank 0 だけ評価で `all_reduce` を呼ぶ、rank ごとにバッチ数が違う）。ノード間でネットワークインターフェースが違う。1つの rank だけ例外で落ちている
- **対処**:
  1. `DistributedSampler(drop_last=True)` などで全 rank のバッチ数を揃える
  2. collective（`all_reduce`、`barrier`、DDP の forward）は全 rank で同じ順番に呼ぶ。rank 0 だけの処理の中で呼ばない
  3. ノード間の問題なら `NCCL_SOCKET_IFNAME=<インターフェース名>` を指定する。IB が使えない環境では `NCCL_IB_DISABLE=1` で切り分ける
  4. 長い評価を rank 0 だけで行う場合は `init_process_group(timeout=datetime.timedelta(minutes=60))` で延ばす

### 2.6 DDP で「Expected to have finished reduction」

- **症状**: `RuntimeError: Expected to have finished reduction in the prior iteration before starting a new one ... parameters that were not used in producing loss`
- **確認**:
```python
loss.backward()
unused = [n for n, p in model.named_parameters() if p.requires_grad and p.grad is None]
print(unused)                                 # 1 GPU で実行して確認する
```
- **原因**: forward で使われないパラメータ（分岐で通らない層、`num_classes=0` にしたのに残っているヘッド、補助ヘッド）がある
- **対処**: 使わない層は削除するか `requires_grad_(False)` にする。どうしても分岐する場合は `DDP(model, find_unused_parameters=True)`（遅くなる）。毎回同じ層が使われないだけなら `static_graph=True` も使える

### 2.7 GPU 使用率が低い（学習が遅い）

- **症状**: `nvidia-smi` の GPU-Util が低い・上下する。1エポックが想定より長い
- **確認**:
```bash
watch -n 1 nvidia-smi                          # Util が 0% と 100% を行き来 → データ待ち
top -u $USER                                   # ワーカーの CPU 使用率
```
```python
import time
t0 = time.perf_counter()
for i, (x, y) in enumerate(loader):            # モデルを通さずにデータだけ回す
    if i == 100: break
print("data only:", (time.perf_counter() - t0) / 100, "s/batch")

torch.cuda.synchronize(); t0 = time.perf_counter()
for _ in range(20):
    loss = criterion(model(x.cuda()), y.cuda()); loss.backward()
torch.cuda.synchronize(); print("model only:", (time.perf_counter() - t0) / 20, "s/batch")
```
- **原因**: データ読み込み（画像のデコード、ネットワークファイルシステム、重い前処理）が律速。`num_workers` が `--cpus-per-task` より多い／少ない。毎 step の `.item()` や `print` で同期している
- **対処**:
  1. データのみの時間 > モデルのみの時間なら、`num_workers` を CPU 数に合わせる、`pin_memory=True`、`persistent_workers=True`
  2. 画像は事前に小さくしておく、ローカル SSD（`$TMPDIR` など）にコピーする、パッチを HDF5 / LMDB にまとめる
  3. ログ用の `loss.item()` は数十 step に1回にする
  4. モデル側が律速なら bf16、`channels_last`、`torch.compile`（基本編 第9節）

### 2.8 「no kernel image is available」/ CUDA が使えない（ドライバーとの不一致）

- **症状**: `torch.cuda.is_available()` が False で `The NVIDIA driver on your system is too old` の警告、または `CUDA error: no kernel image is available for execution on the device`
- **確認**:
```bash
nvidia-smi                                     # 右上の Driver Version と CUDA Version
python -c "import torch; print(torch.__version__, torch.version.cuda)"
python -c "import torch; print(torch._C._cuda_getArchFlags())"   # この wheel が対応する GPU 世代
nvidia-smi --query-gpu=name,compute_cap --format=csv
```
- **原因**: `+cu130` の wheel は CUDA 13.0 用で、ドライバーは 580 系以降が必要。またこの wheel の対応 GPU は `sm_75 sm_80 sm_86 sm_90 sm_100 sm_120`（確認済み）で、V100（sm_70）以前は含まれない
- **対処**: ノードのドライバーが古い、または古い GPU なら、そのノードを避ける（`--constraint` / パーティション指定）か、`cu126` など古い CUDA 向けの wheel で環境を作る（[uv.md](../infra/uv.md)）。Apptainer では `--nv` を付けないとホストのドライバーが見えない

### 2.9 torch.compile で遅い・何度もコンパイルされる

- **症状**: 最初の数 step が数分かかる、途中で急に遅くなる、`torch._dynamo hit config.recompile_limit (8)` の警告
- **確認**:
```bash
TORCH_LOGS="recompiles,graph_breaks" python train.py 2>&1 | grep -E "Recompiling|Graph break" | head
```
- **原因**: 入力形状（最後のバッチ、可変解像度）が変わるたびに再コンパイルされる。forward 内の `.item()`、`print`、NumPy 呼び出し、データ依存の分岐でグラフが途切れる
- **対処**:
  1. `drop_last=True` で形状を揃える。可変長が本質的なら `dynamic=True`
  2. forward 内の `.item()` / `print` / NumPy を外に出す
  3. 評価ループはコンパイルしていない元のモデル（`model._orig_mod`）で回す手もある
  4. 効果が小さければ使わない。まずは eager で正しく動かしてから試す

### 2.10 torch.load で「Weights only load failed」

- **症状**: `_pickle.UnpicklingError: Weights only load failed ... Unsupported global: GLOBAL numpy._core.multiarray.scalar` など（NumPy 1 で保存したファイルは `numpy.core...`）
- **確認**:
```python
from torch.serialization import get_unsafe_globals_in_checkpoint
print(get_unsafe_globals_in_checkpoint("ckpt.pt"))   # 何が含まれているか
```
- **原因**: 2.6 以降 `weights_only=True` が既定。NumPy の値、argparse の Namespace、自作クラスなどを pickle で保存したファイルは読めない
- **対処**:
  1. 信頼できるファイルなら、必要なものだけ許可する:
     ```python
     import numpy as np
     # 許可するものは get_unsafe_globals_in_checkpoint の出力に合わせる（この環境は NumPy 2.3）
     with torch.serialization.safe_globals([np._core.multiarray.scalar, np.dtype]):
         ckpt = torch.load("ckpt.pt", map_location="cpu")
     ```
  2. 自分のファイルで中身を信頼できるなら `weights_only=False`
  3. 今後は保存前に `float(x)` / `vars(args)` などで Python の基本型にしておく

### 2.11 cuDNN のエラー

- **症状**: `cuDNN error: CUDNN_STATUS_NOT_SUPPORTED`、`CUDNN_STATUS_EXECUTION_FAILED`、`CUDNN_STATUS_INTERNAL_ERROR`
- **確認**:
```python
print(torch.backends.cudnn.version(), torch.backends.cudnn.is_available())
torch.backends.cudnn.enabled = False     # これで動くなら cuDNN 側の問題
```
```bash
CUDA_LAUNCH_BLOCKING=1 python train.py
```
- **原因**: 実態は OOM であることが多い（cuDNN の作業領域が確保できない）。非連続テンソル・極端な形状、2.3 と同じ範囲外アクセスが cuDNN のエラーとして出ることもある
- **対処**: バッチサイズを下げて再現するか確認する。`x.contiguous()` を試す。`cudnn.benchmark=True` を切る。pip の torch は cuDNN を同梱しているので、`LD_LIBRARY_PATH` で別の cuDNN を読ませていないか確認する

### 2.12 決定論モードでエラー / 結果が再現しない

- **症状**: `use_deterministic_algorithms(True)` で `RuntimeError: ... does not have a deterministic implementation`、または `CUBLAS_WORKSPACE_CONFIG` のエラー。あるいは同じシードでも結果が変わる
- **確認**:
```python
torch.use_deterministic_algorithms(True, warn_only=True)   # エラーではなく警告にして、該当演算を洗い出す
```
```bash
echo $CUBLAS_WORKSPACE_CONFIG
```
- **原因**: `index_add_`、`scatter_add_`、一部の補間・pooling の backward などは GPU で非決定的。DataLoader のワーカーのシード、`cudnn.benchmark=True`、`num_workers` の違いも結果を変える
- **対処**: `export CUBLAS_WORKSPACE_CONFIG=:4096:8` を設定する。非決定的な演算は `warn_only=True` で許容するか置き換える。完全な一致が不要なら、seed を複数回して平均と分散で比較する方が現実的

### 2.13 再開したら学習率や精度が飛ぶ

- **症状**: チェックポイントから再開した直後に loss が跳ねる、学習率が最初の値に戻っている
- **確認**:
```python
print(scheduler.last_epoch, optimizer.param_groups[0]["lr"])   # 再開直後の値
print(ckpt.keys())                                              # 何を保存したか
```
- **原因**: scheduler / scaler / EMA / optimizer を保存していない、または optimizer をモデルを GPU に移す前に作った。step 単位の scheduler を epoch 単位で進めている
- **対処**: 1.8 の形で全部保存・復元する。`optimizer` は `model.to(device)` の後に作り、その後で `load_state_dict` する

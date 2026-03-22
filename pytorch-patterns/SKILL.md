---
name: pytorch-patterns
description: PyTorch deep learning patterns and best practices for building robust, efficient, and reproducible training pipelines, model architectures, and data loading. Always activate when the user is writing PyTorch models or training scripts, reviewing deep learning code, debugging training loops or data pipelines, optimizing GPU memory or training speed, setting up reproducible experiments, implementing transfer learning or fine-tuning, or asks anything about nn.Module, DataLoader, autograd, AMP, or torch.compile.
---

# PyTorch Development Patterns

Idiomatic PyTorch patterns and best practices for building robust, efficient, and reproducible deep learning applications.

## Workflow

When this skill activates:

1. **Identify the user's task** — new model, training loop, data pipeline, optimization, debugging, or fine-tuning.
2. **Navigate to the relevant section** below. For new projects, start with Core Principles and work down.
3. **Apply device-agnostic patterns by default** — never hardcode `"cuda"`. Always derive from `device.type`.
4. **Flag anti-patterns proactively** if spotted in user-provided code — don't wait to be asked.
5. **Suggest profiling** (`torch.profiler`) when the user reports slowness before recommending optimizations.

---

## Core Principles

### 1. Device-Agnostic Code

Write code that works on CPU, CUDA, and MPS without modification. Never hardcode a device string.

```python
# Good: auto-detect device
device = torch.device(
    "cuda" if torch.cuda.is_available()
    else "mps" if torch.backends.mps.is_available()
    else "cpu"
)
model = MyModel().to(device)

# Bad: crashes when no GPU is present
model = MyModel().cuda()
```

For AMP, derive the device string from `device.type` — never hardcode `"cuda"`:

```python
use_amp = device.type == "cuda"           # AMP is only beneficial on CUDA
scaler = torch.amp.GradScaler("cuda") if use_amp else None

with torch.amp.autocast(device.type, enabled=use_amp):
    output = model(data)
    loss = criterion(output, target)
```

### 2. Reproducibility First

```python
def set_seed(seed: int = 42) -> None:
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    np.random.seed(seed)
    random.seed(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False  # Disable auto-tuner for reproducibility
```

Note: `cudnn.deterministic = True` has a small performance cost. For production training where exact reproducibility isn't required, set `benchmark = True` instead for faster convolutions.

### 3. Explicit Shape Management

Document and verify tensor shapes in `forward()`:

```python
def forward(self, x: torch.Tensor) -> torch.Tensor:
    # x: (batch_size, channels, height, width)
    x = self.conv1(x)        # -> (batch_size, 32, H, W)
    x = self.pool(x)         # -> (batch_size, 32, H//2, W//2)
    x = x.flatten(1)         # -> (batch_size, 32 * H//2 * W//2)  prefer flatten over view
    return self.fc(x)        # -> (batch_size, num_classes)
```

Use `x.flatten(1)` over `x.view(x.size(0), -1)` — it's safer with non-contiguous tensors.

---

## Model Architecture Patterns

### Clean nn.Module Structure

```python
class ImageClassifier(nn.Module):
    def __init__(self, num_classes: int, dropout: float = 0.5) -> None:
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
        )
        self.classifier = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(64 * 16 * 16, num_classes),
        )
        self.apply(self._init_weights)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # x: (B, 3, 32, 32)
        x = self.features(x)      # -> (B, 64, 16, 16)
        x = x.flatten(1)          # -> (B, 64*16*16)
        return self.classifier(x) # -> (B, num_classes)

    def _init_weights(self, module: nn.Module) -> None:
        if isinstance(module, (nn.Linear, nn.Conv2d)):
            nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
            if module.bias is not None:
                nn.init.zeros_(module.bias)
        elif isinstance(module, nn.BatchNorm2d):
            nn.init.ones_(module.weight)
            nn.init.zeros_(module.bias)
```

### Transfer Learning and Fine-Tuning

The most common PyTorch workflow: load a pretrained backbone, freeze it, train the head, then optionally unfreeze.

```python
import torchvision.models as models

# Stage 1: freeze backbone, train head only
model = models.resnet50(weights=models.ResNet50_Weights.IMAGENET1K_V2)
for param in model.parameters():
    param.requires_grad = False              # freeze everything

model.fc = nn.Linear(model.fc.in_features, num_classes)  # replace head
# Only model.fc parameters require grad — optimizer sees only those
optimizer = torch.optim.AdamW(model.fc.parameters(), lr=1e-3)

# Stage 2: unfreeze and fine-tune with a lower LR
def unfreeze(model: nn.Module) -> None:
    for param in model.parameters():
        param.requires_grad = True

unfreeze(model)
optimizer = torch.optim.AdamW([
    {"params": model.layer4.parameters(), "lr": 1e-4},   # deeper layers: low LR
    {"params": model.fc.parameters(),     "lr": 1e-3},   # head: higher LR
])
```

Always use `weights=ModelName_Weights.IMAGENET1K_V2` (not deprecated `pretrained=True`).

---

## Training Loop Patterns

### Complete Training Loop

```python
def train_one_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    criterion: nn.Module,
    device: torch.device,
    scaler: torch.amp.GradScaler | None = None,
) -> float:
    model.train()
    total_loss = 0.0
    use_amp = scaler is not None

    for data, target in dataloader:
        data, target = data.to(device, non_blocking=True), target.to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)   # more efficient than zero_grad()

        with torch.amp.autocast(device.type, enabled=use_amp):
            output = model(data)
            loss = criterion(output, target)

        if use_amp:
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
        else:
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

        total_loss += loss.item()   # .item() after backward — never before

    return total_loss / len(dataloader)
```

`non_blocking=True` pairs with `pin_memory=True` in the DataLoader for async CPU→GPU transfers.

### Validation Loop

```python
@torch.no_grad()
def evaluate(
    model: nn.Module,
    dataloader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> tuple[float, float]:
    model.eval()   # disables dropout; BatchNorm uses running stats
    total_loss, correct, total = 0.0, 0, 0

    for data, target in dataloader:
        data, target = data.to(device), target.to(device)
        output = model(data)
        total_loss += criterion(output, target).item()
        correct += (output.argmax(1) == target).sum().item()
        total += target.size(0)

    return total_loss / len(dataloader), correct / total
```

### LR Scheduler

Always step the scheduler after the optimizer, at the end of each epoch (or step, for OneCycleLR):

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=num_epochs)

for epoch in range(num_epochs):
    train_loss = train_one_epoch(model, train_loader, optimizer, criterion, device, scaler)
    val_loss, val_acc = evaluate(model, val_loader, criterion, device)
    scheduler.step()                          # step AFTER optimizer.step()

    print(f"Epoch {epoch}: loss={train_loss:.4f} val_acc={val_acc:.4f} "
          f"lr={scheduler.get_last_lr()[0]:.2e}")
```

Common scheduler choices:
- `CosineAnnealingLR` — smooth decay, good default
- `OneCycleLR` — aggressive, often fastest convergence (step every batch, not epoch)
- `ReduceLROnPlateau` — plateau-based, pass `val_loss` to `.step(val_loss)`

---

## Data Pipeline Patterns

### Custom Dataset

```python
class ImageDataset(Dataset):
    def __init__(
        self,
        image_dir: str | Path,
        labels: dict[str, int],
        transform: transforms.Compose | None = None,
    ) -> None:
        self.image_paths = sorted(Path(image_dir).glob("*.jpg"))
        self.labels = labels
        self.transform = transform

    def __len__(self) -> int:
        return len(self.image_paths)

    def __getitem__(self, idx: int) -> tuple[torch.Tensor, int]:
        img = Image.open(self.image_paths[idx]).convert("RGB")
        label = self.labels[self.image_paths[idx].stem]
        if self.transform:
            img = self.transform(img)
        return img, label
```

### Optimized DataLoader

```python
dataloader = DataLoader(
    dataset,
    batch_size=32,
    shuffle=True,
    num_workers=4,            # >0 required for persistent_workers to have effect
    pin_memory=True,          # faster CPU→GPU transfer; pair with non_blocking=True
    persistent_workers=True,  # keep workers alive between epochs (num_workers > 0)
    drop_last=True,           # consistent batch sizes for BatchNorm stability
    prefetch_factor=2,        # batches to prefetch per worker
)
```

---

## Checkpointing

Always save full training state so runs can be properly resumed:

```python
def save_checkpoint(
    path: str | Path,
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    scheduler: torch.optim.lr_scheduler.LRScheduler,
    epoch: int,
    val_loss: float,
) -> None:
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "scheduler_state_dict": scheduler.state_dict(),  # restore LR schedule on resume
        "val_loss": val_loss,
    }, path)


def load_checkpoint(
    path: str | Path,
    model: nn.Module,
    optimizer: torch.optim.Optimizer | None = None,
    scheduler: torch.optim.lr_scheduler.LRScheduler | None = None,
) -> dict:
    checkpoint = torch.load(path, map_location="cpu", weights_only=True)
    model.load_state_dict(checkpoint["model_state_dict"])
    if optimizer and "optimizer_state_dict" in checkpoint:
        optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    if scheduler and "scheduler_state_dict" in checkpoint:
        scheduler.load_state_dict(checkpoint["scheduler_state_dict"])
    return checkpoint
```

---

## Performance Optimization

### Memory: Gradient Checkpointing

Trade recomputation for memory — useful for large models that OOM during training:

```python
from torch.utils.checkpoint import checkpoint

class LargeModel(nn.Module):
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = checkpoint(self.block1, x, use_reentrant=False)  # reentrant=False is preferred
        x = checkpoint(self.block2, x, use_reentrant=False)
        return self.head(x)
```

### Speed: torch.compile

`torch.compile` fuses operations and reduces Python overhead. Expect 10–50% speedup on compute-bound workloads, but with caveats:

```python
# Good for: simple feed-forward models, CNNs, transformers
model = torch.compile(model, mode="reduce-overhead")

# Modes:
# "default"        — safe, moderate speedup
# "reduce-overhead" — faster, tolerates minor graph breaks
# "max-autotune"   — slowest compile, fastest inference

# Caveats:
# - Requires PyTorch 2.0+; limited support on Windows
# - First forward pass is slow (compilation warm-up)
# - Dynamic shapes (variable-length sequences) can cause recompilation
# - Disable during debugging — stack traces are harder to read
```

### Profiling Before Optimizing

Always profile before assuming where the bottleneck is:

```python
from torch.profiler import profile, record_function, ProfilerActivity

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    profile_memory=True,
) as prof:
    with record_function("model_forward"):
        output = model(data)

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=15))
```

Check GPU memory separately: `print(torch.cuda.memory_summary(device))`

---

## Anti-Patterns to Avoid

```python
# Bad: forgetting model.eval() — dropout stays on, BatchNorm uses batch stats
model.train()
with torch.no_grad():
    output = model(val_data)   # silently wrong results

# Good
model.eval()
with torch.no_grad():
    output = model(val_data)

# Bad: in-place ops on tensors needed by other ops' backward pass
x = x + residual               # Bad: in-place add breaks autograd for residual connections
x.relu_()                       # Bad: in-place where input is needed for gradient
# Good: out-of-place
x = x + residual
x = F.relu(x)
# Note: F.relu(x, inplace=True) is safe for sequential layers where the input isn't
# reused elsewhere — but avoid it inside residual blocks.

# Bad: .item() before backward — detaches from the computation graph
loss = criterion(output, target).item()
loss.backward()   # Error: can't backprop through a Python float

# Good: .item() only for logging, after backward
loss = criterion(output, target)
loss.backward()
print(f"Loss: {loss.item():.4f}")

# Bad: moving model to GPU every iteration
for data, target in dataloader:
    model = model.cuda()        # moves model every batch — catastrophic

# Good: move once before the loop
model = model.to(device)

# Bad: saving entire model object (fragile, version-sensitive)
torch.save(model, "model.pt")

# Good: save state_dict only
torch.save(model.state_dict(), "weights.pt")
```

---

## Quick Reference

| Pattern | Use it for |
|---|---|
| `model.train()` / `model.eval()` | Always set mode before train/eval pass |
| `@torch.no_grad()` | Inference and validation — disables grad tracking |
| `zero_grad(set_to_none=True)` | More efficient gradient clearing |
| `device = torch.device("cuda" if ... else "cpu")` | Device-agnostic, always |
| `autocast(device.type, enabled=use_amp)` | Mixed precision — derive from device |
| `pin_memory=True` + `non_blocking=True` | Async CPU→GPU transfers |
| `torch.compile(model)` | JIT speedup on PyTorch 2.0+ — profile first |
| `weights_only=True` in `torch.load` | Secure loading, avoids arbitrary code exec |
| `x.flatten(1)` | Safer than `x.view(x.size(0), -1)` for non-contiguous |
| `checkpoint(block, x, use_reentrant=False)` | Large model memory relief |
| `scheduler.step()` after `optimizer.step()` | LR decay — wrong order silently corrupts schedule |
| `torch.profiler` | Always profile before optimizing |

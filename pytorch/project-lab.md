# 🔥 PyTorch Project Mastery

#### 📋 Project: VisioQC — AI Quality Inspector at Prism Electronics

**Prism Electronics** is a Bengaluru-based PCB (Printed Circuit Board) manufacturer supplying major laptop brands. Their QC (Quality Control) line employs 12 human inspectors who manually examine every board for defects — scratches, missing components, solder bridges — under bright lights, 8 hours a day. Miss rate: ~4%. Returns cost ₹2.4 crore per year.

Your task: Build **VisioQC** — a PyTorch-powered image classification model that examines photos of PCBs captured by a conveyor-belt camera and flags defective boards in real time. The model must be accurate enough to reduce the miss rate to under 0.5%, and fast enough to run at 30 boards per minute on a factory GPU server.

> **🎯 What You'll Build**

> - Tensor operations from scratch
> - Custom Dataset & DataLoader
> - CNN classifier with nn.Module
> - Full training loop with backprop
> - Transfer learning (ResNet-18)
> - Model export & inference pipeline
> - PyTorch 2.x + torchvision
> - Python 3.11+
> - CUDA (GPU) / MPS (Apple Silicon)
> - Pillow, NumPy, Matplotlib
> - Flask (inference API)
> - TorchScript (production export)
> - Freshers entering AI/ML careers
> - Python devs moving into deep learning
> - Engineers building computer vision systems
> - Data Scientists learning model deployment

**Scene 0 — Day 1 at Prism Electronics | "The Problem Line"**

> **Ramesh** _VP of Manufacturing — Prism Electronics_
> 
> "Divya, we shipped 3,000 defective PCBs to Stellar Laptops last quarter. Our QC team is exhausted — 8 hours of eye-strain work, and they still miss 4% of defects. I need a machine that doesn't get tired. Build me something that looks at a photo and says 'pass' or 'fail' with 99.5% accuracy. If it works on PCBs, we'll roll it out to all 6 product lines."

> **Divya** _Lead AI Engineer — Prism Electronics_
> 
> "I'll build VisioQC using PyTorch. We have 14,000 labelled PCB photos from the last two years — 'ok' and 'defect'. I'll train a Convolutional Neural Network on them, then fine-tune a pre-trained ResNet so it generalises even on defect types it hasn't seen before. Give me three weeks."

> **Aditya** _Junior ML Engineer (Fresher) — Prism Electronics_
> 
> "Divya, I just finished my ML course. Can I pair with you? I understand theory — gradient descent, CNNs — but I've never written a single line of PyTorch."

### 1. Phase 1 — Tensors: The Building Block of Everything

**What is a Tensor?** A tensor is PyTorch's fundamental data structure — a multi-dimensional array of numbers. Every piece of data in deep learning — images, text embeddings, model weights, gradients — is represented as a tensor. If you know NumPy arrays, tensors are the same idea, but with one superpower: they can run on GPUs and they automatically track gradients for training.

> **🌱 Fresher's Mental Model — The Dimension Ladder**

> - **Scalar (0-D tensor)** — a single number. Example: the loss value 0.342. Shape: `()`
> - **Vector (1-D tensor)** — a list of numbers. Example: one image's 512 feature values. Shape: `(512,)`
> - **Matrix (2-D tensor)** — a table of numbers. Example: a batch of 32 predictions, each with 2 classes. Shape: `(32, 2)`
> - **3-D tensor** — a stack of matrices. Example: a grayscale image — height × width × channels. Shape: `(224, 224, 1)`
> - **4-D tensor** — the most common in deep learning. A *batch* of images: batch_size × channels × height × width. Shape: `(32, 3, 224, 224)` — 32 colour images of 224×224 pixels.
> - **Rule of thumb**: every layer in a neural network takes a tensor in, does math on it, and returns a tensor out. Your entire model is just tensor transformations stacked on top of each other.

#### 1.1 Creating Tensors

```python
import torch
# From a Python list
a = torch.tensor([1.0, 2.0, 3.0])

# Zeros and ones
z = torch.zeros(3, 4)    # 3×4 matrix of 0s
o = torch.ones(2, 3)     # 2×3 matrix of 1s
# Random tensor (like random weight init)
r = torch.randn(4, 4)   # 4×4, values from N(0,1)
# Check shape, dtype, device
print(r.shape)    # torch.Size([4, 4])
print(r.dtype)    # torch.float32
print(r.device)   # cpu
```

**📖 Tensor Creation Explained**

- **torch.tensor()** — creates a tensor from existing Python data. Copies the data into PyTorch's memory.
- **torch.zeros() / ones()** — create tensors pre-filled with 0s or 1s. Shape arguments: rows, cols (or more dimensions). Used to initialise bias vectors.
- **torch.randn()** — creates a tensor of random values from a normal distribution (mean=0, std=1). This is the standard way to randomly initialise neural network weights before training.
- **.shape** — the dimensions of the tensor. Always check this when debugging — shape mismatch is the #1 PyTorch error for freshers.
- **.dtype** — the data type. `torch.float32` is default and what most models use. `torch.long` is used for class labels (integers).
- **.device** — where the tensor lives: `cpu` or `cuda:0` (GPU). Both tensors in an operation must be on the same device.

#### 1.2 Move to GPU

```python
# Check if GPU is available
device = torch.device(
    "cuda" if torch.cuda.is_available()
    else "mps" # Apple Silicon GPU
if torch.backends.mps.is_available()
    else "cpu"
)
print(f"Using device: {device}")

# Move tensor to GPU
x = torch.randn(3, 224, 224)
x = x.to(device)
print(x.device)   # cuda:0
# Move back to CPU (for plotting/numpy)
x_cpu = x.cpu().numpy()
```

**📖 CPU vs GPU — Why It Matters**

- **GPU = parallelism.** A CPU has 8–16 cores. An NVIDIA GPU has 3,000–10,000 CUDA cores. Operations on large tensors (like matrix multiplication in neural networks) run 10–100× faster on a GPU.
- **The device pattern** — always write `device = torch.device("cuda" if ...)` at the top of your script. Then use `.to(device)` on all tensors and models. This makes your code run on CPU, GPU, or Apple Silicon without changing any logic.
- **Rule**: both tensors in any operation must be on the same device. A common fresher error: model is on GPU, input data is on CPU → RuntimeError.
- **.cpu().numpy()** — to convert a GPU tensor to a NumPy array (for plotting with matplotlib), you must move it to CPU first. NumPy doesn't know about GPU memory.
- For VisioQC on Prism's factory GPU server (NVIDIA RTX 3090), training that takes 6 hours on CPU finishes in 18 minutes on GPU.

### 2. Phase 2 — Autograd: How Neural Networks Learn

**Business Problem:** A neural network learns by making a prediction, measuring how wrong it is (the *loss*), and then adjusting its weights to be less wrong — this is called **backpropagation**. PyTorch's **Autograd** engine does this automatically: it watches every tensor operation you do, builds a computation graph, and uses it to compute gradients with a single call to `.backward()`.

**Scene 2 — Training Concept | "Why Does the Model Get Better?"**

> **Aditya** _Junior ML Engineer — Prism Electronics_
> 
> "Divya, I understand the model makes a prediction and we measure the error. But how does it know which weights to change, and by how much? There are millions of weights in a ResNet."

> **Divya** _Lead AI Engineer — Prism Electronics_
> 
> "Think of it this way: you have a bowl and a marble. The marble is the current weight value. The bowl is the loss surface. You want the marble to roll to the lowest point — minimum loss. The gradient tells you which way is downhill. Autograd computes this gradient automatically for every single weight, using the chain rule of calculus. You don't write a single derivative by hand. PyTorch does all of it for you in one line: loss.backward()."

#### 2.1 Understanding Gradients

```python
import torch
# requires_grad=True tells PyTorch to track ops
w = torch.tensor(2.0, requires_grad=True)
b = torch.tensor(1.0, requires_grad=True)

# Forward pass: y = w*x + b
x = torch.tensor(3.0)
y_pred = w * x + b # 2*3 + 1 = 7
# Loss: difference from target (y=10)
loss = (y_pred - 10) ** 2 # (7-10)² = 9
# Backward pass: compute all gradients
loss.backward()

print(w.grad)   # d(loss)/d(w) = -18.0
print(b.grad)   # d(loss)/d(b) = -6.0
```

**📖 Autograd Step by Step**

- **requires_grad=True** — tells PyTorch: "watch this tensor, I want to compute gradients with respect to it." Always set this on model parameters (weights & biases). Never set it on input data.
- **Forward pass** — the normal left-to-right computation: inputs → operations → loss. PyTorch silently records every operation in a computation graph.
- **loss.backward()** — walks the computation graph backwards (chain rule) and fills in the `.grad` attribute of every tensor that had `requires_grad=True`. One line replaces pages of calculus.
- **w.grad = -18.0** — means: "to decrease the loss, increase w." The optimiser uses this to update: `w = w - lr * w.grad`.
- **Never call .backward() twice** without zeroing gradients first — gradients accumulate by default. Always call `optimizer.zero_grad()` at the start of each training step.

```
Autograd Computation Graph for VisioQC

Input Image (x)         Target Label (y_true)
      │                          │
      ▼                          │
[Conv Layer] → [ReLU] → [Pool] → ... → [FC Layer]
                                              │
                                         y_pred
                                              │
                               loss = CrossEntropy(y_pred, y_true)
                                              │
                                      loss.backward()
                                              │
                    ┌─────────────────────────┘
                    ▼
         Gradients flow BACKWARDS through every layer
         Each weight gets: weight.grad = ∂loss/∂weight
                    │
                    ▼
         optimizer.step()  →  weight = weight - lr * weight.grad
                    │
                    ▼
               Model improves ✓
```

### 3. Phase 3 — nn.Module: Building the Neural Network

**Business Problem:** We need to define the VisioQC CNN architecture — the layers, their sizes, and how data flows through them. PyTorch's `nn.Module` is the base class for all neural networks. You subclass it, define your layers in `__init__`, and write the forward pass in `forward()`. PyTorch handles all gradients automatically.

#### 3.1 Your First nn.Module — A Simple Classifier

```python
import torch
import torch.nn as nn

class SimpleNet(nn.Module):
    def __init__(self):
        super().__init__()
        # Define layers
self.fc1 = nn.Linear(784, 256)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(256, 2)

    def forward(self, x):
        # Define data flow
x = self.fc1(x)    # 784 → 256
x = self.relu(x)   # activation
x = self.fc2(x)    # 256 → 2
return x
model = SimpleNet()
print(model)
```

**📖 nn.Module Explained**

- **class MyNet(nn.Module)** — inherit from `nn.Module`. This gives your class all of PyTorch's built-in functionality: parameter tracking, GPU movement, save/load, etc.
- **super().__init__()** — always call this first in `__init__`. It runs the parent class's setup. Forgetting this causes cryptic errors.
- **nn.Linear(784, 256)** — a fully-connected layer. Takes 784 inputs (e.g., flattened 28×28 image), produces 256 outputs. Contains learnable weights (784×256 matrix) and bias (256 values).
- **nn.ReLU()** — activation function: `max(0, x)`. Introduces non-linearity so the network can learn complex patterns beyond simple linear relationships.
- **forward(x)** — defines what happens when you call `model(input)`. This is your data flow. PyTorch calls `forward()` automatically — never call it directly.
- **The output has 2 units** — one for "ok", one for "defect". The highest value wins (argmax).

#### 3.2 The VisioQC CNN Architecture

📷 Input Image

Shape: (batch, 3, 224, 224) ← 32 colour PCB images

▼

Conv2d(3→32) + BatchNorm + ReLU + MaxPool

🔲 Conv Block 1

Shape: (batch, 32, 112, 112) ← 32 feature maps

▼

Conv2d(32→64) + BatchNorm + ReLU + MaxPool

🔲 Conv Block 2

Shape: (batch, 64, 56, 56) ← 64 feature maps

▼

Conv2d(64→128) + BatchNorm + ReLU + MaxPool

🔲 Conv Block 3

Shape: (batch, 128, 28, 28)

▼

AdaptiveAvgPool2d → Flatten

⬛ Flatten

Shape: (batch, 128) ← 1D feature vector

▼

Linear(128→64) + ReLU + Dropout(0.5)

🔗 Fully Connected

Shape: (batch, 64)

▼

Linear(64→2)

📊 Output Logits

Shape: (batch, 2) ← [score_ok, score_defect]

```python
import torch.nn as nn

class VisioQCNet(nn.Module):
    def __init__(self, num_classes=2):
        super().__init__()

        # Convolutional feature extractor
self.features = nn.Sequential(
            nn.Conv2d(3, 32, kernel_size=3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(),
            nn.MaxPool2d(2),                   # 224→112

            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),                   # 112→56

            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d((1, 1))          # → (128,1,1)
        )

        # Classification head
self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(64, num_classes)
        )

    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x
```

> **nn.Sequential()** — a container that runs layers one after another. Cleaner than writing `x = layer1(x); x = layer2(x)...` ten times. The forward pass is implicit.
**nn.Conv2d(3, 32, kernel_size=3, padding=1)** — 2D convolution. In: 3 channels (RGB). Out: 32 feature maps. Each feature map learns to detect a different pattern (edges, textures, solder joints). padding=1 keeps spatial size the same.
**nn.BatchNorm2d(32)** — normalises activations across the batch. Makes training faster and more stable. Without it, deep networks are hard to train. Always put it after Conv2d, before ReLU.
**nn.MaxPool2d(2)** — downsamples by taking the max in each 2×2 region. Halves height and width. Reduces computation and makes features position-invariant (a defect on the left vs right edge → same detection).
**nn.AdaptiveAvgPool2d((1,1))** — reduces any spatial size to 1×1. This means our model works on any input resolution — not just 224×224. Factory cameras can be upgraded without retraining.
**nn.Dropout(0.5)** — randomly sets 50% of neuron outputs to zero during training. Prevents overfitting (memorising the training data). Turn it off during inference — `model.eval()` does this automatically.

### 4. Phase 4 — Dataset & DataLoader: Feeding Data to the Model

**Business Problem:** Prism's 14,000 PCB images are stored in two folders: `data/ok/` and `data/defect/`. We need to load them, apply preprocessing (resize, normalise), shuffle them for training, and feed them to the model in batches of 32. PyTorch's `Dataset` and `DataLoader` handle all of this.

#### 4.1 Custom Dataset

```python
from torch.utils.data import Dataset
from torchvision import transforms
from PIL import Image
import os
class PCBDataset(Dataset):
    def __init__(self, root, transform=None):
        self.samples = []
        self.transform = transform
        # label 0=ok, 1=defect
for label, cls in enumerate(["ok", "defect"]):
            folder = os.path.join(root, cls)
            for fname in os.listdir(folder):
                self.samples.append(
                    (os.path.join(folder, fname), label)
                )

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        path, label = self.samples[idx]
        img = Image.open(path).convert("RGB")
        if self.transform:
            img = self.transform(img)
        return img, label
```

**📖 Custom Dataset — The Three Methods**

- **__init__()** — runs once when you create the dataset. Collect file paths and labels here. Don't load all images into memory — just record where they are. Loading 14,000 images at once would crash most machines.
- **__len__()** — returns the total number of samples. DataLoader uses this to know when one epoch (full pass through data) is complete.
- **__getitem__(idx)** — loads and returns ONE sample by index. DataLoader calls this repeatedly. This is where you open the image file, apply transforms, and return (image_tensor, label). Keep it fast — it runs inside data-loading worker threads.
- **.convert("RGB")** — ensures all images have 3 channels. Some PNG files have 4 channels (RGBA with transparency). Our model expects exactly 3.
- **enumerate(["ok","defect"])** — maps folder names to integer labels: ok=0, defect=1. CrossEntropyLoss expects integer class indices, not one-hot vectors.

#### 4.2 Transforms & DataLoader

```python
from torchvision import transforms
from torch.utils.data import DataLoader, random_split

# Training augmentations
train_tf = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.RandomHorizontalFlip(),
    transforms.RandomRotation(10),
    transforms.ColorJitter(brightness=0.2),
    transforms.ToTensor(),
    transforms.Normalize(
        mean=[0.485, 0.456, 0.406],
        std=[0.229, 0.224, 0.225]
    )
])

# Load, split 80/20 train/val
dataset = PCBDataset("data/", transform=train_tf)
n_train = int(0.8 * len(dataset))
train_ds, val_ds = random_split(dataset, [n_train, len(dataset) - n_train])

train_loader = DataLoader(train_ds, batch_size=32, shuffle=True, num_workers=4)
val_loader   = DataLoader(val_ds,   batch_size=32, shuffle=False, num_workers=4)
```

**📖 Transforms & DataLoader Explained**

- **transforms.Compose([])** — chains image transforms in order. Each transform takes the output of the previous one.
- **RandomHorizontalFlip / RandomRotation** — data augmentation: artificially multiply your training data by creating variations. A flipped PCB image of a defect is still a defect. This prevents overfitting when you have limited data.
- **ToTensor()** — converts a PIL Image (values 0–255) to a PyTorch tensor (values 0.0–1.0) and reorders dimensions from (H,W,C) to (C,H,W). Always the second-to-last transform.
- **Normalize(mean, std)** — standardises pixel values using ImageNet statistics. Use these exact values when using any pretrained ImageNet model (ResNet, EfficientNet etc.) — they were trained with these normalisation parameters.
- **DataLoader(shuffle=True)** — always shuffle training data. If you don't, the model sees all "ok" boards first, then all "defect" — it learns a biased order, not the patterns.
- **num_workers=4** — loads data in 4 background CPU threads in parallel while the GPU is training on the previous batch. Keeps the GPU fed without waiting. Set to 0 on Windows if you get errors.

### 5. Phase 5 — The Training Loop: Teaching VisioQC to See Defects

**Business Problem:** Now we put it all together. We loop through the training data for 20 epochs. Each epoch: forward pass (predict), compute loss, backward pass (gradients), update weights. At the end of each epoch, we check accuracy on the validation set so we know if the model is improving or overfitting.

**Scene 5 — First Training Run | "The Loss Is Dropping"**

> **Aditya** _Junior ML Engineer — Prism Electronics_
> 
> "Divya, I'm watching the terminal — Epoch 1: loss=0.693, accuracy=52%. That's basically random. Is something wrong?"

> **Divya** _Lead AI Engineer — Prism Electronics_
> 
> "That's exactly right. Random weights give ~50% accuracy on a 2-class problem. Watch Epoch 3: loss=0.41, accuracy=82%. Epoch 8: loss=0.18, accuracy=94%. The model is learning. By Epoch 20 it'll be above 98%. The loss going down is the signal that backpropagation is working — weights are being adjusted in the right direction."

#### 5.1 The Complete Training Loop

```python
import torch
import torch.nn as nn

# Setup
model     = VisioQCNet().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=5, gamma=0.5)

for epoch in range(20):
    # ── TRAIN ──
model.train()          # enables Dropout, BatchNorm train mode
train_loss, correct = 0.0, 0
for images, labels in train_loader:
        images, labels = images.to(device), labels.to(device)

        optimizer.zero_grad()        # 1. clear old gradients
logits = model(images)         # 2. forward pass
loss   = criterion(logits, labels) # 3. compute loss
loss.backward()              # 4. backward pass
optimizer.step()             # 5. update weights
train_loss += loss.item()
        correct    += (logits.argmax(1) == labels).sum().item()

    scheduler.step()     # decay learning rate every 5 epochs
# ── VALIDATE ──
model.eval()
    val_correct = 0
with torch.no_grad():
        for images, labels in val_loader:
            images, labels = images.to(device), labels.to(device)
            out = model(images)
            val_correct += (out.argmax(1) == labels).sum().item()

    train_acc = correct     / len(train_ds) * 100
val_acc   = val_correct / len(val_ds)   * 100
print(f"Epoch {epoch+1:02d} | Loss {train_loss/len(train_loader):.3f} | Train {train_acc:.1f}% | Val {val_acc:.1f}%")
```

> **The 5 steps — memorise this sequence**: (1) zero_grad, (2) forward pass, (3) compute loss, (4) backward pass, (5) optimizer step. This exact order, every single training iteration, forever. Miss any step and the model either doesn't learn or accumulates wrong gradients.
**optimizer.zero_grad()** — clears gradients from the previous iteration. PyTorch accumulates gradients by default (useful for some advanced techniques). For standard training, always zero them at the start of each batch.
**nn.CrossEntropyLoss()** — the standard loss for classification. It combines `LogSoftmax + NLLLoss` internally. Expects raw logits (not softmax outputs) from the model. For binary classification, it works perfectly with 2 output units.
**model.train() / model.eval()** — switches the model between training mode (Dropout active, BatchNorm uses batch statistics) and evaluation mode (Dropout off, BatchNorm uses running statistics). Forgetting `eval()` during validation gives artificially low validation accuracy.
**torch.no_grad()** — disables gradient computation during validation. The model makes predictions but doesn't build a computation graph. 40% less memory usage, 25% faster. Always use this for inference and validation.
**StepLR scheduler** — reduces the learning rate by 50% every 5 epochs. High LR at the start to make big jumps toward the minimum. Smaller LR later to fine-tune without overshooting. This technique often adds 1–3% final accuracy.
**logits.argmax(1)** — picks the class with the highest score. For a (32, 2) output, argmax along dim=1 gives a (32,) tensor of predicted class indices (0 or 1).

```
Epoch 01 | Loss 0.693 | Train  51.4% | Val  50.8%
Epoch 02 | Loss 0.541 | Train  72.1% | Val  69.3%
Epoch 03 | Loss 0.412 | Train  82.6% | Val  80.1%
Epoch 05 | Loss 0.289 | Train  88.9% | Val  87.4%
Epoch 10 | Loss 0.142 | Train  94.7% | Val  93.8%
Epoch 15 | Loss 0.087 | Train  97.2% | Val  96.5%
Epoch 20 | Loss 0.061 | Train  98.6% | Val  97.9%
```

### 6. Phase 6 — Transfer Learning: 99%+ Accuracy with ResNet-18

**Business Problem:** Our custom CNN reached 97.9% validation accuracy — but Prism needs 99.5% to reduce the miss rate below 0.5%. Training a deeper network from scratch would need 10× more data. **Transfer Learning** gives us the answer: take ResNet-18 (trained on 1.2 million ImageNet images, already knows edges, textures, shapes), freeze its feature extractor, and replace only the final layer for our 2-class PCB problem. We get deep visual understanding without needing huge data.

**Scene 6 — The Breakthrough | "We Didn't Train It From Scratch"**

> **Aditya** _Junior ML Engineer — Prism Electronics_
> 
> "Wait — we're using a model trained on cats, dogs, and cars to detect PCB defects? How does that make sense?"

> **Divya** _Lead AI Engineer — Prism Electronics_
> 
> "Because the early layers of ResNet don't know 'cat' or 'dog' — they know 'edge', 'curve', 'texture', 'colour gradient'. These features are universal. A solder bridge is an unexpected texture. A missing component is an unexpected shape. ResNet-18 spent a year learning to see these primitives on 1.2 million images. We're borrowing that knowledge and just teaching the final layer to categorise our two classes. It works because vision is vision."

#### 6.1 Fine-tune ResNet-18 for PCB Defect Detection

```python
from torchvision import models
import torch.nn as nn

# Load pretrained ResNet-18
model = models.resnet18(weights="IMAGENET1K_V1")

# Freeze ALL layers first
for param in model.parameters():
    param.requires_grad = False
# Unfreeze last residual block (layer4)
for param in model.layer4.parameters():
    param.requires_grad = True
# Replace classifier: 512 → 2 classes
model.fc = nn.Sequential(
    nn.Linear(512, 128),
    nn.ReLU(),
    nn.Dropout(0.4),
    nn.Linear(128, 2)
)
model = model.to(device)
```

**📖 Transfer Learning Strategy**

- **weights="IMAGENET1K_V1"** — loads ResNet-18 with weights already trained on ImageNet. The model already "sees" the world. Without `weights=`, you'd get random weights — no transfer at all.
- **Freeze all parameters first** — setting `requires_grad=False` on all parameters means the optimiser won't update them. They're locked at their ImageNet values.
- **Unfreeze layer4** — the last residual block handles high-level features most relevant to our domain. Allowing it to fine-tune adapts these features to PCB textures specifically.
- **Replace model.fc** — ResNet's original final layer outputs 1000 classes (ImageNet). We replace it with our own 2-class head. Only this new head (and layer4) will be trained.
- **Why not unfreeze everything?** — with 14,000 images, fully retraining ResNet-18 (11M parameters) would overfit badly. The frozen layers act as a powerful, fixed feature extractor.
- **Result**: trains in 5 epochs instead of 20. Reaches 99.3% accuracy — vs 97.9% from our custom CNN. Less data needed, better accuracy.

🔒 FROZEN — ResNet-18 Backbone (Layer 1-3)

Pretrained on 1.2M ImageNet images Detects edges, textures, curves, shapes requires_grad = False → NOT updated

▼

passes feature maps down

🔓 TRAINABLE — ResNet Layer4 (unfrozen)

High-level feature adaptation Learns PCB-specific patterns from our data requires_grad = True → updated by Adam

▼

512-dim feature vector

🆕 NEW — Custom Classifier Head

Linear(512→128) → ReLU → Dropout → Linear(128→2) Randomly initialised → trained from scratch on PCB data

▼

📊 Output: [score_ok, score_defect]

argmax → class 0 (OK) or class 1 (DEFECT)

#### 6.2 Save and Load the Best Model

```python
# Save model checkpoint
torch.save({
    "epoch":      epoch,
    "model_state": model.state_dict(),
    "optim_state": optimizer.state_dict(),
    "val_acc":     val_acc
}, "visioqc_best.pth")

# Load the checkpoint later
ckpt  = torch.load("visioqc_best.pth", map_location=device)
model.load_state_dict(ckpt["model_state"])
model.eval()

print(f"Loaded model — val acc: {ckpt['val_acc']:.2f}%")
```

**📖 Saving Models — state_dict vs full model**

- **model.state_dict()** — a dictionary of all trainable parameters (weight tensors). This is the PyTorch-recommended way to save a model. It's portable across code refactors as long as the architecture is the same.
- **Save the full checkpoint dict** — always save epoch, optimizer state, and validation accuracy alongside weights. This lets you resume training exactly where you left off after a power cut or system restart.
- **map_location=device** — if you trained on GPU and load on CPU (or a different GPU), this remaps tensors to the right device. Without it, loading a GPU model on a CPU machine raises an error.
- **Never save the whole model object** (`torch.save(model, ...)`) — it pickle-serialises the entire Python object. Breaks if you rename a class or refactor files. Always use `state_dict()`.
- **model.eval() after loading** — always put the model in eval mode before inference. Without it, Dropout randomly drops outputs and BatchNorm uses mini-batch statistics, giving inconsistent results.

### 7. Phase 7 — Inference & Deployment: VisioQC Goes to the Factory Floor

**Business Problem:** The trained model needs to run on Prism's factory server, receiving camera images via a REST API and responding with "OK" or "DEFECT" in under 100 milliseconds per board. We build a Flask API around the model, and export it as TorchScript for production reliability.

#### 7.1 Single-Image Inference

```python
import torch
from torchvision import transforms
from PIL import Image

CLASSES = ["OK", "DEFECT"]

infer_tf = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(
        [0.485, 0.456, 0.406],
        [0.229, 0.224, 0.225]
    )
])

def predict(image_path):
    img = Image.open(image_path).convert("RGB")
    x   = infer_tf(img).unsqueeze(0).to(device)

    with torch.no_grad():
        logits = model(x)
        probs  = torch.softmax(logits, dim=1)
        pred   = probs.argmax(1).item()

    return {
        "label":      CLASSES[pred],
        "confidence": round(probs[0, pred].item() * 100, 2)
    }

print(predict("pcb_003.jpg"))
```

**📖 Inference Pipeline Explained**

- **Inference transforms — no augmentation**: during training we used RandomFlip, RandomRotation, ColorJitter. During inference, remove all augmentation. Only Resize, ToTensor, and Normalize. Augmentation was only to create variation for training, not for final prediction.
- **.unsqueeze(0)** — adds a batch dimension. Our model expects shape (N, C, H, W). A single image is (C, H, W) — shape (3, 224, 224). `unsqueeze(0)` makes it (1, 3, 224, 224). The model now treats it as a "batch of 1".
- **torch.softmax(logits, dim=1)** — converts raw logit scores (can be any value) into probabilities that sum to 1. Makes the confidence score interpretable: 0.97 = 97% confident this is a defect.
- **.item()** — extracts a Python scalar from a single-element tensor. Do this for any value you want to print or send in a JSON response. JSON serialisers don't know how to handle PyTorch tensors.
- **Return a dict** — not just a class name. The confidence score is critical for production — if confidence is below 85%, route the board to human inspection. 97% confident → auto-reject. 62% confident → human reviews it.

```
{'label': 'DEFECT', 'confidence': 97.43}
```

#### 7.2 Flask REST API for the Factory Server

```python
from flask import Flask, request, jsonify
from PIL import Image
import io
app = Flask(__name__)

# Load model once at startup
ckpt = torch.load("visioqc_best.pth", map_location=device)
model.load_state_dict(ckpt["model_state"])
model.eval()

@app.route("/inspect", methods=["POST"])
def inspect():
    file  = request.files["image"]
    img   = Image.open(io.BytesIO(file.read())).convert("RGB")
    result = predict_image(img)  # uses infer_tf + model
return jsonify(result)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

> **Load model at startup** — not inside the route function. Loading a model takes 0.5–2 seconds. If you load it per request, each API call would be 2 seconds slow. Load once at server start, keep it in memory. Handle 30 requests per second easily.
**POST /inspect** — the factory camera system sends a POST request with the image attached as a file. Our API processes it and returns JSON: `{"label": "DEFECT", "confidence": 97.43}`. The conveyor belt's control system reads this and triggers the rejection arm.
**io.BytesIO(file.read())** — reads the uploaded file bytes into a memory buffer. PIL's Image.open() can read from a buffer directly without saving to disk first. Faster and cleaner for a server.
**host="0.0.0.0"** — makes the API accessible from other machines on the network, not just localhost. The factory camera is a different machine — it needs to reach the API server over the local network.
For production: use **Gunicorn** instead of Flask's development server, and add GPU batching (collect 8 images, run inference once) for higher throughput.

### 8. Core PyTorch Concepts — Quick Reference

Concept

What It Does

When to Use

torch.Tensor

N-dimensional array on CPU or GPU

Everything. All data is a tensor.

requires_grad=True

Tells autograd to track this tensor

On learnable parameters (weights, biases).

loss.backward()

Computes all gradients via backprop

After every forward pass during training.

optimizer.zero_grad()

Clears accumulated gradients

Before every forward pass in training loop.

optimizer.step()

Updates weights using computed gradients

After loss.backward() in training loop.

nn.Module

Base class for all neural networks

Subclass it to build any model.

nn.Sequential

Chains layers in order

Simple feed-forward blocks.

nn.Conv2d

2D convolution for image feature extraction

Image and video models (CNN, ResNet).

nn.Linear

Fully connected layer (matrix multiply + bias)

Classification heads, MLP layers.

nn.BatchNorm2d

Normalise feature maps across the batch

After Conv2d, before activation.

nn.Dropout

Randomly zero neuron outputs

In classifiers to prevent overfitting.

Dataset / DataLoader

Custom data loading with batching & shuffle

Any custom dataset (images, text, tabular).

torchvision.transforms

Image preprocessing & augmentation pipeline

All computer vision pipelines.

model.train() / eval()

Toggle Dropout / BatchNorm behaviour

train() for training, eval() for inference.

torch.no_grad()

Disable gradient tracking

Validation and inference — saves memory.

state_dict()

Dictionary of model weights

Saving and loading models.

Transfer Learning

Reuse pretrained weights, replace classifier

Limited data, proven architectures.

### 9. Common Mistakes — Fresher vs Experienced

**⚠️ How Freshers vs Senior Engineers Write PyTorch**

- **❌ Fresher Mistake #1** — Forgetting optimizer.zero_grad()

- **❌ Fresher Mistake #2** — Forgetting model.eval() during validation

- **❌ Fresher Mistake #3** — Not moving data to the same device as model

- **❌ Fresher Mistake #4** — Using augmentation during inference

##### ❓ Fresher Questions — Interview & Understanding

**Q: Q1: What is the difference between model.parameters() and model.state_dict()?**

A: **model.parameters()** returns a generator of all trainable tensors (weights and biases). You pass this to the optimizer: `optimizer = Adam(model.parameters(), lr=1e-3)` — this tells Adam which tensors to update. **model.state_dict()** returns an ordered dictionary mapping layer names to their weight tensors — it's used for saving and loading. You can think of `parameters()` as "for the optimizer" and `state_dict()` as "for saving to disk."

**Q: Q2: Why do we use CrossEntropyLoss instead of Mean Squared Error for classification?**

A: MSE measures geometric distance between numbers, but class labels (0, 1, 2) have no meaningful numeric distance — class 2 is not "twice as far" from class 0 as class 1 is. CrossEntropyLoss treats the problem correctly: it converts the model's raw logits to probabilities (via softmax), then measures the negative log-likelihood of the true class. If the model assigns 99% probability to the correct class, loss ≈ 0. If it assigns 1% probability to the correct class, loss is very high. This penalty structure trains the model to be confidently correct, not just numerically close.

**Q: Q3: What is BatchNorm and why is it placed before ReLU, not after?**

A: BatchNorm normalises the output of a layer across the current mini-batch — it subtracts the mean and divides by the standard deviation, then scales with learned parameters gamma and beta. This keeps activations from becoming very large or very small, which would make gradients vanish or explode. The ordering debate (before or after ReLU) is real: the original paper places BN before ReLU. Modern practice often places it after, especially with ResNets. For freshers: follow Conv → BN → ReLU as the default — it's the most commonly seen and most stable.

**Q: Q4: When should I use Transfer Learning vs training from scratch?**

A: Use Transfer Learning when: (a) you have fewer than ~50,000 training images, (b) your images have visual similarity to natural photos (objects, textures, colours), (c) you need fast results. Train from scratch when: (a) you have millions of domain-specific images (e.g., satellite imagery, X-rays) that look nothing like natural photos, (b) you need a custom architecture for speed constraints, (c) you have the compute and data to justify it. For the vast majority of industrial vision projects — PCBs, food quality, product inspection — Transfer Learning from ImageNet is the right starting point.

**Q: Q5: What is overfitting and how do you detect it?**

A: Overfitting happens when the model memorises the training data instead of learning general patterns. Detection: your training accuracy keeps rising (99%+) but validation accuracy stops improving or starts decreasing. Example: train acc = 99.5%, val acc = 87.2% — the gap of 12.3% indicates overfitting. Fixes: add Dropout, reduce model size, add more data augmentation, use early stopping (save the model at the epoch with the best validation accuracy, not the last epoch), collect more training data. In VisioQC, we added Dropout(0.4) and RandomAugmentation which closed the gap to under 1.5%.

**Quiz: 🧠 Quick Check: In PyTorch's training loop, what is the correct order of operations?**

- A) forward → loss → backward → zero_grad → step
- B) zero_grad → forward → loss → backward → step
- C) forward → zero_grad → loss → backward → step
- D) zero_grad → forward → step → loss → backward

> **Answer/explanation:** **✅ Answer: B — zero_grad → forward → loss → backward → step**

**zero_grad first** — clear stale gradients from the previous batch. If you skip this, gradients accumulate and weights get corrupted updates within 2-3 iterations.
**forward** — run the model on the input batch. This builds the computation graph.
**loss** — compute how wrong the prediction was using your loss function.
**backward** — walk the computation graph backwards to fill in all .grad attributes.
**step** — the optimizer uses those .grad values to update every weight: `w = w - lr * w.grad`.

> **PyTorch Project — Core Takeaways for Freshers**

> - **Tensors are your only data type.** Images, labels, weights, gradients — everything is a tensor. Master shape manipulation: view(), reshape(), unsqueeze(), squeeze(), permute(). 90% of PyTorch bugs are shape mismatches. print(tensor.shape) after every transformation while learning.
> - **The 5-step training loop is sacred.** zero_grad → forward → loss → backward → step. In that order. Every iteration. In any neural network you will ever write. Tattoo this on your brain.
> - **model.train() and model.eval() are not optional.** Forgetting eval() during validation gives you wrong metrics. Forgetting train() after validation means your model doesn't learn. Always switch modes explicitly.
> - **Always use torch.no_grad() for inference.** It disables the computation graph, saves memory, and speeds up inference by 20–40%. Never run validation or production inference without it.
> - **Transfer Learning is almost always the right answer.** If your task involves images, audio, or text and you have under 100,000 samples, start with a pretrained model. You will get better accuracy faster with less code and less compute.
> - **Data preprocessing is 40% of the work.** Inconsistent image sizes, wrong normalisation statistics, augmentation applied at inference, class imbalance — these are responsible for more production failures than model architecture choices. Debug your DataLoader before debugging your model.
> - **Save checkpoints, not just the final model.** Training can crash at Epoch 18 of 20. Always save the best checkpoint (by validation accuracy) during training. Always save the optimizer state so you can resume. Storage is cheap; re-training is expensive.
> - **Design for the real deployment constraint first.** VisioQC must run in under 100ms. We checked inference latency before finalising the architecture. A model that is 99.9% accurate but takes 2 seconds per image is useless on a 30-boards-per-minute conveyor belt.

##### PyTorch Production Standards — Prism Electronics Engineering Rules

- Always set a random seed at the top of training scripts: `torch.manual_seed(42); torch.backends.cudnn.deterministic = True`. Without this, two training runs on the same data give different models — impossible to reproduce a result for audit or debugging.
- Log training metrics to a file (not just print). Use TensorBoard (`torch.utils.tensorboard.SummaryWriter`) or Weights & Biases. Watching loss curves visually catches overfitting 10 epochs earlier than comparing numbers in a terminal.
- Always validate that your DataLoader is correct before training. Print one batch: `images, labels = next(iter(train_loader)); print(images.shape, labels)`. Shape should be (32, 3, 224, 224) and labels should be a mix of 0s and 1s — not all 0s (class imbalance or data loading bug).
- Use `torch.compile(model)` (PyTorch 2.0+) in production. It JIT-compiles the model and gives 10–30% speedup on most hardware with no code changes. One line, free performance.
- Export to TorchScript (`torch.jit.script(model)`) for the factory server. TorchScript models run without Python installed — faster, portable, and not dependent on your conda environment. The factory server only needs the .pt file.
- Monitor GPU utilisation during training with `nvidia-smi -l 1`. If GPU utilisation is below 70%, your DataLoader is the bottleneck — increase `num_workers` or use `pin_memory=True`. An idle GPU is wasted money.

##### 🏋️ Hands-On Exercises — Extend VisioQC

1. **Add class-weighted loss for imbalanced data:** Prism's dataset has 11,000 "ok" boards and only 3,000 "defect" boards. Train a new model using `nn.CrossEntropyLoss(weight=torch.tensor([1.0, 3.67]).to(device))` — the defect class gets 3.67× more penalty for misclassification. Compare validation recall on defect class vs unweighted training. In safety-critical QC, missing a defect (false negative) is far worse than a false alarm.
2. **Visualise what the model sees with Grad-CAM:** Implement a simple Gradient-weighted Class Activation Map to highlight which region of the PCB the model focused on when it predicted "DEFECT". Use the `pytorch-grad-cam` library. Show the heatmap overlaid on the original image. Present to Ramesh (VP Manufacturing) — seeing the model point to the exact solder bridge builds trust in AI decisions.
3. **Benchmark inference speed:** Write a script that runs the model on 100 images three times and measures average latency: (a) CPU with batch_size=1, (b) GPU with batch_size=1, (c) GPU with batch_size=32. Use `time.perf_counter()` and warm up the model first (run 10 images before timing). The factory requirement is under 100ms/image at 30 boards/minute. Does your model meet this SLA?
4. **Try EfficientNet-B0 instead of ResNet-18:** Replace the backbone with `models.efficientnet_b0(weights="IMAGENET1K_V1")`. EfficientNet has fewer parameters than ResNet-18 but often achieves higher accuracy. Modify the classifier head (EfficientNet uses `model.classifier[1]` as the final layer, not `model.fc`). Train for 5 epochs and compare: accuracy, model size (`sum(p.numel() for p in model.parameters())`), and inference time vs ResNet-18.
5. **Build a confusion matrix and compute F1 score:** After training, run the model on the full validation set and collect all predictions and labels. Build a confusion matrix using `sklearn.metrics.confusion_matrix`. Calculate precision, recall, and F1 score for the "defect" class specifically. For VisioQC, recall (defect detection rate) must be ≥ 99.5%. If it's below, discuss with the team: do we need more defect training samples, or do we lower the classification threshold from 0.5 to 0.3?

### PyTorch Project Complete 🔥

You have built VisioQC — Prism Electronics' production AI quality inspector. You mastered tensors and GPU computation, understood autograd and backpropagation, built a custom CNN with nn.Module, created a Dataset and DataLoader with data augmentation, wrote a complete training loop with validation, applied transfer learning with ResNet-18 to push accuracy above 99%, and deployed the model as a Flask REST API on the factory server. This is the full PyTorch production workflow used by real computer vision teams.

> **Ramesh** _VP of Manufacturing — Prism Electronics_
> 
> "Week one live on the production line: VisioQC inspected 21,400 boards. Caught 847 defects — 99.3% of them correctly. Our human inspectors review only the borderline cases now: 34 boards this week versus 21,400 before. Three of our QC inspectors have been retrained as AI operations specialists — they manage the model, monitor its drift, and label new defect types as they appear. The production line now runs one shift, not two. The ₹2.4 crore annual return cost is already halved in month one."

> **Divya** _Lead AI Engineer — Prism Electronics_
> 
> "Aditya, the transfer learning idea — unfreezing layer4 specifically — added 1.4% accuracy over the frozen approach. That 1.4% means 300 fewer defective boards shipped per month. You found that by reading the paper and testing it. That's what separates an AI engineer from someone who just runs tutorials. The model will drift as new PCB designs come in — you're now the engineer who knows how to detect that and retrain. Own it."

> **Next: Advanced PyTorch — Custom Loss Functions, GAN, Object Detection & Model Optimisation**

> - **Custom Loss Functions** — write `nn.Module` subclasses for domain-specific losses: Focal Loss (handles extreme class imbalance), Dice Loss (for segmentation), Contrastive Loss (for similarity learning). Critical when standard CrossEntropy is not enough.
> - **Object Detection with YOLO / Faster-RCNN** — move beyond classification to detecting and locating multiple defects in a single PCB image. Torchvision has ready-made Faster-RCNN and SSD models. YOLO (via ultralytics) is the industry standard for real-time detection.
> - **Semantic Segmentation** — classify every pixel of the image. Used for precisely locating the defect area (not just "defective — yes/no"). UNet is the classic architecture for industrial segmentation.
> - **Model Quantisation & Pruning** — compress the model to run on edge devices (Raspberry Pi, Jetson Nano, mobile phones). `torch.quantization.quantize_dynamic` reduces model size by 4× with minimal accuracy loss. Deploy VisioQC on a camera module directly — no server needed.
> - **ONNX Export** — export your PyTorch model to the Open Neural Network Exchange format. Run it in TensorRT (NVIDIA, 3–10× speedup), OpenVINO (Intel), or CoreML (Apple). Universal deployment across hardware.
> - **PyTorch Lightning** — a high-level wrapper that removes boilerplate from training loops, handles multi-GPU training, mixed precision, gradient clipping, and checkpoint management automatically. Standard in ML research teams and production ML platforms.
> - **Distributed Training** — train on multiple GPUs across multiple servers using `torch.distributed` and `DistributedDataParallel`. When your dataset has 1 million+ images, single-GPU training takes weeks. Distributed training reduces this to hours.

# 🔢 NumPy Project Mastery

> **👋 Hey Fresher — Read This First!**

> NumPy is the foundation of all numerical computing in Python. Every data science library you've heard of — Pandas, TensorFlow, PyTorch, scikit-learn, OpenCV — is built on top of NumPy. Without NumPy, doing maths on large lists of numbers in Python is painfully slow. With NumPy, the same operation is 100–500× faster because it runs in compiled C code, not Python loops.

> **Company in this project:** SkySignal — an AI startup in Hyderabad building a fraud detection model for UPI transactions. Their ML team processes 500,000 transaction feature vectors every day. Right now, their junior engineer Dev is using plain Python lists and for-loops. It takes 45 minutes to run one training pass. Your lead is Dr Ananya. You are about to replace Dev's loops with NumPy and bring that 45-minute run down to 18 seconds.

#### What You Will Learn and Build in This Project

You will build SkySignal's numerical processing pipeline — from creating your first array to matrix operations for a neural network forward pass — learning every core NumPy concept with a real ML business reason.

ndarray Basics, Indexing & Slicing, Reshaping, Broadcasting, Math & Stats, Linear Algebra, Random, Boolean Masks, Fancy Indexing, Real ML Pipeline

> **📦 Phase 1 — Foundations**

> Install NumPy, understand ndarray vs Python list, create arrays 5 ways, inspect shape, dtype, and ndim.

> Select elements, rows, columns, sub-arrays. 1D and 2D slicing. Boolean masks. Fancy indexing.

> reshape(), flatten(), transpose, hstack, vstack — the tools every ML pipeline uses constantly.

> Universal functions, broadcasting, linear algebra, random arrays, normalisation — the full ML pipeline.

### 1. Phase 1 — NumPy Foundations

**Business Problem:** Dev's fraud detection script uses Python lists. Computing the dot product of two 10,000-element feature vectors takes 8 ms in a loop. With NumPy, the same operation takes 0.04 ms — 200× faster. At 500,000 transactions, that difference is 45 minutes vs 18 seconds per training pass.

**Scene 1 — SkySignal ML Lab | The Overnight Training Run**

> **Dev** _Junior ML Engineer — SkySignal_
> 
> Dr Ananya, the feature normalisation loop ran overnight. 11 hours for 500,000 rows. I'm using list comprehensions like this: `[x / max_val for x in feature_list]`. It works, but the model can't iterate fast enough to finish before the next day's data arrives.

> **Dr Ananya** _Lead ML Engineer — SkySignal_
> 
> Priya, show Dev what NumPy vectorisation looks like. Same division, one line, no loop, and it finishes in milliseconds. The entire concept of modern ML is built on this — if you write Python loops over numbers, you've already lost.

#### 1.1 Install and Import NumPy

```bash
# Install NumPy
pip install numpy

# Import — always aliased as np
import numpy as np

# Confirm version
print(np.__version__)
```

**📖 Why alias as np?**

By universal convention, NumPy is imported as `np`. Every ML codebase, tutorial, paper, and Stack Overflow answer uses `np`. You will see `np.array()`, `np.dot()`, `np.zeros()` — always `np`.  
  
Writing `import numpy` without the alias works but then every call is `numpy.array()` — verbose and non-standard.

```
1.26.4
```

#### 1.2 Python List vs NumPy Array — The Core Difference

```
  PYTHON LIST                        NUMPY NDARRAY
  ───────────                        ─────────────
  [1000, 2500, 300, 8000, 450]       np.array([1000, 2500, 300, 8000, 450])

  Stored as Python objects           Stored as raw C numbers in contiguous memory
  Each element: ~28 bytes            Each element: 8 bytes (float64)
  Loop to multiply each value        Multiply entire array in ONE C instruction

  result = []
  for x in amounts:                  result = amounts * 1.18   ← one line
      result.append(x * 1.18)        ← runs at C speed

  Mixed types allowed                All elements SAME type (dtype)
  [1, "hello", 3.5]                  → predictable, fast, memory-efficient

  Speed on 500,000 items: ~420 ms    Speed on 500,000 items: ~1.8 ms
                                     ► 230× FASTER
```

> **💡 The Golden Rule of NumPy**

> **Never write a Python for-loop over a NumPy array.** If you find yourself writing `for x in my_array`, stop — there is always a NumPy function or operator that does the same thing 100× faster. This shift in thinking is the single most important skill to build when moving from Python beginner to data engineer.

#### 1.3 Creating Arrays — 5 Ways You Must Know

```
# 1. From a Python list
arr = np.array([1000, 2500, 300, 8000])
```

**📖 np.array() — Most Common**

Pass any Python list (or list of lists for 2D). NumPy infers the dtype automatically — integers become `int64`, decimals become `float64`.  
  
Mix an int and a float in the list and NumPy quietly promotes everything to `float64` — the safest type that can hold all values. No error, no warning.

```
# 2. Zeros and Ones — initialise weight matrices
np.zeros((3, 4))    # 3 rows, 4 cols of 0.0
np.ones((2, 5))     # 2 rows, 5 cols of 1.0
np.full((3, 3), 7)  # 3×3 filled with 7
```

**📖 zeros(), ones(), full()**

In ML, you constantly need pre-allocated arrays of a specific size filled with a starting value.  
  
`zeros` — initialise bias vectors to zero.  
`ones` — used for masking or initialisation.  
`full` — fill with any constant value.  
  
The argument is a **tuple** for shape: `(rows, columns)`. For a 1D array of 5 zeros, use `np.zeros(5)` — no tuple needed.

```
# 3. Range of values — like Python range()
np.arange(0, 10, 2)   # [0, 2, 4, 6, 8]
# 4. Evenly spaced between two values
np.linspace(0, 1, 5)  # [0.0, 0.25, 0.5, 0.75, 1.0]
```

**📖 arange() vs linspace()**

`arange(start, stop, step)` — specify the step size. Works like Python's `range()` but returns an array and supports float steps.  
  
`linspace(start, stop, num)` — specify how many values you want, not the step. Guaranteed to include both endpoints. Essential for generating learning rate schedules and plotting axes.

```
# 5. Identity matrix — used in linear algebra
np.eye(4)   # 4×4 identity matrix
```

**📖 np.eye() — Identity Matrix**

An identity matrix has 1s on the diagonal and 0s everywhere else. It's the matrix equivalent of the number 1 — multiplying any matrix by the identity matrix gives you the same matrix back.  
  
Used in regularisation, initialising weight matrices, and verifying matrix inverse operations in ML.

#### 1.4 Inspecting an Array — shape, dtype, ndim

```python
arr = np.array([[1,2,3],
               [4,5,6]])

print(arr.shape)   # (2, 3)
print(arr.dtype)   # int64
print(arr.ndim)    # 2
print(arr.size)    # 6 (total elements)
```

**📖 The Four Inspection Attributes**

`shape` — tuple of dimensions: `(rows, cols)`. For 3D: `(batches, rows, cols)`. This is what you check first — shape mismatches cause 90% of NumPy errors.  
  
`dtype` — the data type of every element. `int64`, `float32`, `float64`, `bool`. In ML, `float32` uses half the memory of `float64` with negligible precision loss.  
  
`ndim` — number of dimensions (axes). 1D = vector, 2D = matrix, 3D+ = tensor.  
  
`size` — total element count: product of all shape dimensions.

```
  DIMENSIONS (ndim) — What They Look Like

  1D array  (ndim=1, shape=(4,))         2D array  (ndim=2, shape=(3,4))
  ──────────────────────────────         ────────────────────────────────
  [ 10   25   30   80 ]                  [[ 1   2   3   4 ]
                                          [ 5   6   7   8 ]
  ↑ a vector — one row of features        [ 9  10  11  12 ]]

                                         ↑ a matrix — rows=samples, cols=features

  3D array  (ndim=3, shape=(2,3,4))      In ML:
  ─────────────────────────────────      1D → bias vector
  [ batch 0 ] → 3×4 matrix              2D → weight matrix W
  [ batch 1 ] → 3×4 matrix              3D → image batch (H × W × channels)
                                         4D → video batch (frames × H × W × C)
```

> **✅ Phase 1 Key Takeaways**

> - NumPy arrays store all elements as the same dtype — this is why they're fast. Python lists store objects of any type — this is why they're slow.
> - Always import as `import numpy as np` — every codebase on the planet uses this alias.
> - `shape` is the first attribute to check whenever you load or create an array — shape mismatches are the #1 NumPy error.
> - Use `arange` when you know the step size; use `linspace` when you know how many points you need.

### 2. Phase 2 — Indexing and Slicing

**Business Problem:** SkySignal's feature matrix has 500,000 rows (transactions) and 28 columns (features). To train the fraud model, the team needs to: extract one feature column, select the first 1,000 rows for a mini-batch, and pull only the rows where the fraud label is 1. All three operations are slicing and indexing.

**Scene 2 — SkySignal ML Lab | Preparing the Training Mini-Batch**

> **Dr Ananya** _Lead ML Engineer — SkySignal_
> 
> Dev, we need to feed the model in batches of 512 rows. Pull rows 0 to 511, all 28 features, and extract the label column separately. In your current list code this is a nested loop. In NumPy it's one slice each. Show me you understand NumPy indexing and I'll let you own the data pipeline.

#### 2.1 1D Array Indexing and Slicing

```
  1D ARRAY: arr = np.array([10, 20, 30, 40, 50, 60, 70, 80])

  Index (positive):    0    1    2    3    4    5    6    7
  Index (negative):   -8   -7   -6   -5   -4   -3   -2   -1
  Values:             10   20   30   40   50   60   70   80

  arr[2]        → 30                  (single element)
  arr[-1]       → 80                  (last element)
  arr[2:5]      → [30, 40, 50]        (index 2 up to but NOT including 5)
  arr[::2]      → [10, 30, 50, 70]   (every 2nd element)
  arr[::-1]     → [80, 70, 60, ...]   (reversed)
  arr[1:7:2]    → [20, 40, 60]        (start:stop:step)
```

```python
arr = np.array([10, 20, 30, 40, 50])

print(arr[0])      # 10 — first element
print(arr[-1])     # 50 — last element
print(arr[1:4])    # [20 30 40]
print(arr[::2])    # [10 30 50]
```

**📖 Slicing Syntax: [start : stop : step]**

NumPy slicing uses the same `start:stop:step` syntax as Python lists — but it works on multi-dimensional arrays too.  
  
**Stop is exclusive** — `arr[1:4]` gives indices 1, 2, 3 — not 4.  
  
**Negative indices** count from the end: `arr[-1]` is always the last element regardless of array length — crucial for writing general-purpose ML code.

#### 2.2 2D Array Indexing — Rows and Columns

```
  2D ARRAY: feature matrix — rows = transactions, cols = features

        col 0   col 1   col 2   col 3
  row 0 [  1.0    0.5    0.2    8.1 ]
  row 1 [  2.3    1.1    0.0    3.3 ]
  row 2 [  0.1    4.4    1.7    2.0 ]
  row 3 [  5.5    0.3    0.8    9.1 ]

  arr[1, 2]      → 0.0          single element at row 1, col 2
  arr[0, :]      → [1.0, 0.5, 0.2, 8.1]    entire row 0
  arr[:, 2]      → [0.2, 0.0, 1.7, 0.8]    entire col 2 (one feature)
  arr[0:2, 1:3]  → [[0.5, 0.2],             sub-matrix rows 0-1, cols 1-2
                     [1.1, 0.0]]
```

```
X = np.array([[1.0, 0.5, 0.2],
              [2.3, 1.1, 0.0],
              [0.1, 4.4, 1.7]])

# Single element
X[1, 2]

# Entire second row
X[1, :]

# All rows, third column only
X[:, 2]
```

**📖 2D Indexing: [row, col]**

For 2D arrays, indexing always follows `[row, column]`. The colon `:` means "all of this dimension".  
  
`X[1, :]` — row 1, all columns → one transaction's feature vector.  
  
`X[:, 2]` — all rows, column 2 → the third feature for every transaction.  
  
This is how you extract a single feature column for correlation analysis or feed individual features to a model.

#### 2.3 Boolean Masking — Filter by Condition

```python
amounts = np.array([500, 15000, 200, 80000, 1200])

# Create a boolean mask
mask = amounts > 5000
print(mask)
# [False  True False  True False]
# Apply mask — keeps only True rows
high_value = amounts[mask]
print(high_value)
# [15000 80000]
```

**📖 Boolean Masking — No Loops Needed**

`amounts > 5000` creates an array of `True`/`False` — one value per element, based on the condition. This happens instantly in C, on the entire array at once.  
  
Passing the boolean array as an index keeps only the elements where the mask is `True`.  
  
In fraud detection, this is how you extract all suspicious transactions in one operation instead of looping through 500,000 rows.

```
# Multiple conditions — use & and |
mask = (amounts > 1000) & (amounts < 50000)
mid_range = amounts[mask]

# Use mask on a 2D array (feature matrix)
labels = np.array([0, 1, 0, 1, 0])
fraud_rows = X[labels == 1]
```

**📖 Combining Conditions & 2D Masking**

Combine boolean arrays with `&` (AND) and `|` (OR) — same rule as Pandas, because Pandas is built on NumPy.  
  
The 2D masking pattern `X[labels == 1]` is used in every ML training loop: select only the feature rows that belong to a specific class. This is how you build class-balanced training batches.

#### 2.4 Fancy Indexing — Select Specific Rows

```python
X = np.array([[10,11],
              [20,21],
              [30,31],
              [40,41]])

# Select rows 0, 2, 3 by index list
rows = np.array([0, 2, 3])
print(X[rows])
# [[10 11]
#  [30 31]
#  [40 41]]
```

**📖 Fancy Indexing — Non-Contiguous Selection**

Normal slicing (`X[0:3]`) only selects contiguous (adjacent) rows. Fancy indexing lets you pick any set of rows in any order by passing an array of indices.  
  
In ML, this is how you implement random shuffling and mini-batch selection: generate a random permutation of row indices, then use fancy indexing to extract the batch.

### 3. Phase 3 — Reshaping Arrays

**Business Problem:** SkySignal's raw data arrives as a flat 1D stream of numbers. The neural network expects a 2D matrix of shape (samples, features). Images arrive as 3D arrays (H, W, channels) but the dense layer expects a 1D vector. Reshaping is the glue between raw data and model input.

**Scene 3 — SkySignal ML Lab | The Shape Error**

> **Dev** _Junior ML Engineer — SkySignal_
> 
> The model is crashing with "ValueError: shapes (28,) and (28,10) not aligned." My feature vector has shape (28,) but the weight matrix expects input shaped (1, 28). I have the right numbers — they're just in the wrong shape.

> **Dr Ananya** _Lead ML Engineer — SkySignal_
> 
> Classic reshape issue. A 1D vector of 28 elements and a 2D row vector of shape (1, 28) contain identical data — they're the same numbers arranged differently. reshape() fixes this in one call.

#### 3.1 reshape() — Change Shape Without Changing Data

```
  RESHAPE — Same data, different arrangement

  arr = np.arange(12)   → [ 0  1  2  3  4  5  6  7  8  9  10  11 ]
                            shape: (12,)   ndim: 1

  arr.reshape(3, 4)     →  [[ 0  1  2  3 ]
                             [ 4  5  6  7 ]    shape: (3, 4)   ndim: 2
                             [ 8  9 10 11 ]]

  arr.reshape(2, 6)     →  [[ 0  1  2  3  4  5 ]
                             [ 6  7  8  9 10 11 ]]   shape: (2, 6)

  arr.reshape(2, 2, 3)  →  [[[ 0  1  2]           shape: (2, 2, 3)   ndim: 3
                              [ 3  4  5]]
                             [[ 6  7  8]
                              [ 9 10 11]]]

  RULE: Total elements must stay the same.  3×4 = 2×6 = 2×2×3 = 12 ✓
```

```python
arr = np.arange(12)

# Reshape to 3 rows × 4 columns
mat = arr.reshape(3, 4)
print(mat.shape)   # (3, 4)
# Use -1 to let NumPy calculate one dimension
mat2 = arr.reshape(3, -1)
print(mat2.shape)  # (3, 4) — auto-calculated
```

**📖 reshape() and the -1 Trick**

Reshape changes how the data is viewed — it does not copy or reorder elements. The total number of elements must remain the same: `3 × 4 = 12`.  
  
**The -1 trick:** Pass `-1` for one dimension and NumPy calculates it automatically. `arr.reshape(3, -1)` means "3 rows, figure out the columns yourself." This is extremely useful when you don't want to hardcode batch sizes.

#### 3.2 flatten() and ravel() — Collapse to 1D

```
mat = np.array([[1,2,3],
               [4,5,6]])

# flatten() — always returns a COPY
flat = mat.flatten()
# [1 2 3 4 5 6]
# ravel() — returns a VIEW if possible (faster)
flat2 = mat.ravel()
# [1 2 3 4 5 6]
```

**📖 flatten() vs ravel()**

Both collapse a multi-dimensional array to 1D. The difference is memory:  
  
`flatten()` — always returns a new copy. Safe to modify without affecting the original.  
  
`ravel()` — returns a view of the original data if possible (no copy, no extra memory). Modifying it may modify the original. Faster for read-only operations.  
  
In practice: use `flatten()` when safety matters, `ravel()` when performance matters.

#### 3.3 Transpose — Flip Rows and Columns

```
W = np.array([[1,2,3],
              [4,5,6]])
# W.shape is (2, 3)
W_T = W.T
# W_T.shape is (3, 2)
# [[1 4]
#  [2 5]
#  [3 6]]
```

**📖 .T — The Transpose Shortcut**

Transpose flips rows into columns and columns into rows. A matrix of shape `(2, 3)` becomes `(3, 2)` after transposing.  
  
In neural networks, you constantly need to align matrix shapes for multiplication. If `X` has shape `(n, d)` and weights `W` have shape `(d, k)`, the forward pass is `X @ W`. If shapes don't align, you transpose one of them.

#### 3.4 Stack Arrays — hstack and vstack

```
a = np.array([[1,2], [3,4]])
b = np.array([[5,6], [7,8]])

# Stack vertically — add more rows
np.vstack([a, b])
# [[1 2]  shape: (4, 2)
#  [3 4]
#  [5 6]
#  [7 8]]
# Stack horizontally — add more columns
np.hstack([a, b])
# [[1 2 5 6]  shape: (2, 4)
#  [3 4 7 8]]
```

**📖 vstack and hstack**

`vstack` (vertical stack) — adds more rows. Think "stacking plates" — new data goes below. Used to combine training batches or accumulate predictions across iterations.  
  
`hstack` (horizontal stack) — adds more columns. Used to combine feature sets: append engineered features to the raw feature matrix.  
  
Both accept a list of arrays. The arrays must be compatible: same number of columns for vstack, same number of rows for hstack.

> **✅ Phase 3 Key Takeaways**

> - Total elements must be conserved across reshape: `arr.reshape(3, 4)` only works if arr has exactly 12 elements.
> - Use `-1` in reshape when one dimension should be auto-calculated — avoids hardcoding batch sizes.
> - `.T` transposes in one character — the most concise operation in all of NumPy.
> - Shape mismatches are the #1 NumPy bug. When in doubt: print shape before and after every reshape.

### 4. Phase 4 — Mathematical Operations and Ufuncs

**Business Problem:** Every step in training SkySignal's fraud model involves maths on arrays: normalising features, computing losses, applying activation functions, updating weights. All of it is array maths — and all of it must be vectorised to run fast enough.

#### 4.1 Element-Wise Arithmetic — No Loops

```
a = np.array([10, 20, 30])
b = np.array([1,  2,  3])

a + b # [11, 22, 33]
a - b # [ 9, 18, 27]
a * b # [10, 40, 90]
a / b # [10.0, 10.0, 10.0]
a ** 2 # [100, 400, 900]
```

**📖 Vectorised Arithmetic**

Every arithmetic operator (`+`, `-`, `*`, `/`, `**`) applied to two NumPy arrays runs element-wise — index 0 with index 0, index 1 with index 1, and so on.  
  
This happens in compiled C — no Python interpreter overhead per element. On 500,000 elements, this is the difference between 0.3ms and 300ms.

#### 4.2 Universal Functions (ufuncs)

```
x = np.array([0, 1, 4, 9, 16])

np.sqrt(x)    # [0., 1., 2., 3., 4.]
np.exp(x)     # e^x for each element
np.log(x+1)  # natural log — +1 avoids log(0)
np.abs(x)     # absolute values
np.sin(x)     # sin of each element
```

**📖 What is a ufunc?**

Universal functions (ufuncs) are NumPy functions that operate on every element of an array at once, in compiled C. They are the vectorised equivalents of Python's `math.sqrt()`, `math.exp()`, etc.  
  
In ML:  

`np.exp()` — used in softmax activation.  

`np.log()` — used in cross-entropy loss.  

`np.sqrt()` — used in RMS normalisation and Adam optimiser.  

        Never use Python's `math` module on arrays — it only handles scalars.

#### 4.3 Aggregate Functions — Reduce an Array to One Value

```
scores = np.array([[85, 90, 78],
                    [92, 65, 88]])

np.sum(scores)           # 498 — grand total
np.sum(scores, axis=0)  # [177, 155, 166] — per column
np.sum(scores, axis=1)  # [253, 245] — per row

np.mean(scores)          # 83.0
np.max(scores)           # 92
np.argmax(scores)        # 3 (flat index of max)
```

**📖 The axis Parameter — Collapse Along a Direction**

`axis=0` — collapse across rows (down each column). Result has shape equal to the columns: `(3,)`.  
  
`axis=1` — collapse across columns (across each row). Result has shape equal to the rows: `(2,)`.  
  
**Memory trick:** axis 0 is the row axis. Summing along axis 0 removes rows and leaves column totals.  
  
`argmax()` returns the *index* of the maximum — used in classification to find the predicted class.

```
  UNDERSTANDING axis — Visual Guide

  arr shape: (2, 3)          axis=0 → collapse DOWN (across rows)
                             axis=1 → collapse ACROSS (across columns)

  [[85  90  78]              axis=0 sum → [177, 155, 166]
   [92  65  88]]                          ↑     ↑     ↑
                                          85+92 90+65 78+88

                             axis=1 sum → [253, 245]
                                          ↑     ↑
                                          85+90+78  92+65+88
```

### 5. Phase 5 — Broadcasting

**Business Problem:** SkySignal's feature matrix X has shape (500000, 28) — half a million rows, 28 features each. Normalisation requires subtracting the mean of each feature column (a 1D array of 28 values) from every row. That's a (500000, 28) minus (28,) operation — two arrays of different shapes. Broadcasting makes this possible without loops or copying.

#### 5.1 What is Broadcasting?

```
  BROADCASTING — Operate on arrays of different shapes

  EXAMPLE 1: Scalar broadcast (simplest)
  arr = [10, 20, 30, 40]
  arr + 5 = [15, 25, 35, 45]   ← 5 is "broadcast" to match arr's shape

  EXAMPLE 2: 2D minus 1D (most common in ML)
  X shape:    (4, 3)          mean shape:  (3,)
  [[1  2  3]                  [1.5  2.5  3.5]
   [4  5  6]         -
   [2  3  4]
   [1  2  3]]

  Result (4, 3):              Each row gets the same mean subtracted
  [[-0.5 -0.5 -0.5]          ← row 0 minus mean
   [ 2.5  2.5  2.5]          ← row 1 minus mean
   [ 0.5  0.5  0.5]
   [-0.5 -0.5 -0.5]]

  NumPy stretches the (3,) array across 4 rows — NO data is copied.
  This is 500,000× faster than writing a for-loop over rows.
```

```python
X    = np.random.randn(500000, 28)
mean = X.mean(axis=0)  # shape (28,)
std  = X.std(axis=0)   # shape (28,)
# Z-score normalisation — broadcasting in action
X_norm = (X - mean) / std
# (500000,28) - (28,) → works via broadcasting
print(X_norm.shape)  # (500000, 28)
```

**📖 Z-Score Normalisation with Broadcasting**

This is one of the most important operations in all of ML preprocessing: **subtract the mean and divide by standard deviation** per feature column.  
  
Without broadcasting, you'd need a for-loop over 28 features — slow and verbose. With broadcasting, NumPy handles the shape alignment automatically.  
  
After this operation, every feature column has mean ≈ 0 and std ≈ 1, which speeds up gradient descent significantly.

> **💡 Broadcasting Rules — When Does it Work?**

> NumPy compares shapes from the right, dimension by dimension. Two dimensions are compatible if they are equal, or one of them is 1 (or missing). Example: `(500000, 28)` and `(28,)` — compare from right: 28 == 28 ✓, then 500000 vs missing (treated as 1) → broadcast. If shapes aren't compatible, NumPy raises a clear error: "operands could not be broadcast together with shapes..."

### 6. Phase 6 — Linear Algebra

**Business Problem:** The core of SkySignal's fraud detection neural network is a series of matrix multiplications: features × weights → hidden layer → output. Each layer is a dot product. NumPy's linear algebra module makes this a one-line operation instead of three nested for-loops.

#### 6.1 Dot Product and Matrix Multiplication

```
  DOT PRODUCT vs MATRIX MULTIPLICATION

  1D dot product (two vectors → one scalar):
  a = [1, 2, 3]   b = [4, 5, 6]
  np.dot(a, b) = 1×4 + 2×5 + 3×6 = 32   ← used for similarity scores

  2D matrix multiplication (@ operator):
  X shape: (100, 28)    W shape: (28, 64)
  X @ W  → result shape: (100, 64)

  Rule: inner dimensions must match → (100, [28]) @ ([28], 64) → (100, 64)
  Outer dimensions become the result shape.
```

```
# 1D dot product
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])
np.dot(a, b)    # 32
# 2D matrix multiply — neural net forward pass
X = np.random.randn(100, 28)  # 100 samples
W = np.random.randn(28, 64)  # weight matrix
b = np.zeros(64)              # bias
Z = X @ W + b # shape (100, 64)
```

**📖 @ operator — Preferred for Matrix Multiply**

`np.dot(a, b)` — works for both 1D and 2D. For 2D arrays it does matrix multiplication.  
  
`A @ B` — the `@` operator (Python 3.5+) is the modern, readable way to write matrix multiplication. Use `@` for 2D, `np.dot()` for 1D vectors.  
  
The line `Z = X @ W + b` is literally one layer of a neural network forward pass: matrix multiply (linear transformation) + bias (broadcasting).

#### 6.2 np.linalg — Linear Algebra Toolkit

```
A = np.array([[4, 7],
              [2, 6]])

# Determinant
np.linalg.det(A)       # 10.0
# Inverse — A⁻¹
np.linalg.inv(A)

# Eigenvalues and eigenvectors
vals, vecs = np.linalg.eig(A)

# L2 norm (magnitude of a vector)
np.linalg.norm(A)
```

**📖 np.linalg — When You Need It**

`linalg.inv()` — matrix inverse. Used in closed-form solutions like linear regression's normal equation: `w = inv(X.T @ X) @ X.T @ y`.  
  
`linalg.eig()` — eigenvalues/vectors. Used in PCA (Principal Component Analysis) for dimensionality reduction.  
  
`linalg.norm()` — vector/matrix magnitude. Used to normalise vectors (divide by their length) and compute gradient magnitudes during training.

### 7. Phase 7 — Random Arrays

**Business Problem:** SkySignal's model needs random weight initialisation, random data shuffling for mini-batch SGD, and synthetic data generation for testing. NumPy's random module handles all of this — with seed control for reproducible experiments.

#### 7.1 Random Number Generation

```
# Set seed for reproducibility
np.random.seed(42)

# Uniform random: values between 0 and 1
np.random.rand(3, 4)

# Standard normal (mean=0, std=1)
np.random.randn(3, 4)

# Random integers
np.random.randint(0, 100, size=(5, 5))

# Random choice from an array
np.random.choice([10,20,30], size=5)
```

**📖 rand vs randn**

`rand(rows, cols)` — uniform distribution between 0 and 1. Used for dropout masks.  
  
`randn(rows, cols)` — standard normal distribution (bell curve centred at 0). **Used for weight initialisation** — weights should start near zero with small variance.  
  
`randint(low, high, size)` — random integers. Used to generate random batch indices.  
  
**seed(42)** — fix the random seed so every run produces the same "random" numbers. Essential for reproducible ML experiments.

#### 7.2 Shuffle and Permutation — For Mini-Batch SGD

```
n_samples = 500000
# Generate a shuffled index array
indices = np.random.permutation(n_samples)

# Take the first 512 as a mini-batch
batch_idx = indices[:512]
X_batch   = X[batch_idx]   # fancy indexing
y_batch   = y[batch_idx]
```

**📖 permutation() — The Mini-Batch Pattern**

`np.random.permutation(n)` returns a randomly shuffled array of integers from 0 to n-1. This is used for **mini-batch SGD** — the standard training algorithm for neural networks:  
  
1. Shuffle all row indices.  
2. Split into batches of 512.  
3. Train on each batch.  
4. Repeat every epoch.  
  
Combined with fancy indexing, this is the entire data shuffling pipeline in 3 lines.

### 8. Phase 8 — Views vs Copies and Memory Efficiency

**Business Problem:** SkySignal's X matrix is 500,000 × 28 float64 = 112 MB of RAM. Making unnecessary copies during preprocessing could push memory usage past the server limit and crash the training job mid-way through. Understanding views vs copies is a memory safety issue.

#### 8.1 Views — No Extra Memory

```python
arr = np.arange(10)

# Slicing returns a VIEW — no copy
view = arr[2:6]
view[0] = 99
print(arr)  # [0  1  99  3  4  5  6  7  8  9]
# ↑ Original was modified!
# Force a copy when you need independence
copy = arr[2:6].copy()
```

**📖 Views — Shared Memory**

Slicing a NumPy array creates a **view** — a new array object that points to the same memory. No data is copied. This is fast and memory-efficient.  
  
**The catch:** modifying the view modifies the original. This trips up beginners who expect slices to be independent copies.  
  
Call `.copy()` explicitly when you need an independent array. Check with `np.shares_memory(a, b)` to verify whether two arrays share memory.

##### ⚠️ The Most Common NumPy Bug in ML Code

**Scenario:** You slice a training batch from X, normalise it, and think you're working on a copy. But it's a view — your normalisation has now modified X permanently. The next batch gets pre-modified data and your training loss behaves unpredictably. **Fix:** whenever you modify data derived from a larger array, call `.copy()` first or use `np.shares_memory()` to check.

#### 8.2 dtype Control — Memory Efficiency

```python
# Default float64: 8 bytes per element
X64 = np.random.randn(500000, 28)
print(X64.nbytes / 1e6, "MB")  # 112.0 MB
# float32: 4 bytes — same precision for ML
X32 = X64.astype(np.float32)
print(X32.nbytes / 1e6, "MB")  # 56.0 MB
```

**📖 float32 vs float64 in ML**

NumPy defaults to `float64` (64-bit / double precision). For most ML training, `float32` (32-bit / single precision) is sufficient — GPUs are also optimised for 32-bit computation.  
  
Switching to `float32` halves your memory usage: 112 MB → 56 MB for a 500K × 28 matrix. At scale, this means fitting the entire dataset in GPU VRAM instead of constantly paging to CPU RAM.  
  
`nbytes` — total bytes used. Divide by 1e6 for MB, 1e9 for GB.

### 9. Phase 9 — Useful NumPy Functions for ML

**Business Problem:** SkySignal needs: softmax for the output layer, one-hot encoding for labels, clipping predictions to valid ranges, and sorting predictions to find top-k fraudulent transactions. All of this is 1–2 NumPy lines each.

#### 9.1 where() — Vectorised If-Else

```
scores = np.array([0.2, 0.8, 0.5, 0.9, 0.1])

# If score > 0.5 → "fraud", else → "legit"
labels = np.where(scores > 0.5,
                   "fraud",
                   "legit")
# ['legit' 'fraud' 'legit' 'fraud' 'legit']
```

**📖 np.where() — Replace if/else Loops**

`np.where(condition, value_if_true, value_if_false)` applies an if-else to every element simultaneously — no loop.  
  
In ML: convert probability scores to class labels, apply ReLU activation (`np.where(x > 0, x, 0)`), or flag anomalies in one line. Three arguments means "replace" — one argument means "find indices where condition is True".

#### 9.2 clip() — Bound Values to a Range

```
probs = np.array([-0.1, 0.0, 0.5, 1.0, 1.2])

# Clip probabilities to valid [0, 1] range
safe = np.clip(probs, 0.0, 1.0)
# [0.0, 0.0, 0.5, 1.0, 1.0]
```

**📖 clip() — Prevent Numerical Instability**

In ML, numerical outputs can drift slightly outside valid ranges due to floating-point arithmetic. `np.clip(arr, min, max)` clamps every value to the specified range — values below min become min, values above max become max.  
  
Critical use: before computing `np.log(probs)` in cross-entropy loss, clip probabilities away from 0 to avoid `log(0) = -infinity` which crashes training.

#### 9.3 argsort() — Sort by Index

```
fraud_scores = np.array([0.2, 0.9, 0.4, 0.7, 0.1])

# Indices that would sort the array
idx = np.argsort(fraud_scores)[::-1]
# Sorted descending: [1, 3, 2, 0, 4]
# Top-3 highest fraud risk transaction indices
top3 = idx[:3]   # [1, 3, 2]
```

**📖 argsort() — Get Order, Not Values**

`argsort()` returns the indices that would sort the array — not the sorted values themselves. This is crucial when you want to find the top-k predictions and need to know which original row (transaction ID) they correspond to.  
  
`[::-1]` reverses to get descending order (highest score first). Then `idx[:3]` gives the indices of the top 3 highest-risk transactions.

#### 9.4 unique() and bincount() — Count Categories

```
y = np.array([0,1,0,0,1,1,0,1])

# Unique labels and their counts
vals, counts = np.unique(y, return_counts=True)
# vals: [0 1]   counts: [4 4]
# Fast integer label count (labels must be ints)
np.bincount(y)
# [4 4] — count of 0s and 1s
```

**📖 Class Balance Check**

Before training any classifier, always check class balance. If 98% of transactions are legitimate and only 2% are fraud, the model can achieve 98% accuracy by predicting "legit" for everything — which is useless.  
  
`np.unique(y, return_counts=True)` — works for any dtype, returns both the values and their counts.  
  
`np.bincount(y)` — for integer labels only, but much faster. Index position = label, value = count.

### 10. Phase 10 — The Complete SkySignal ML Pipeline

**Business Problem:** Combine everything into the data preprocessing and forward-pass pipeline that feeds SkySignal's fraud detection model — replacing Dev's 45-minute Python loop pipeline with an 18-second NumPy pipeline.

#### 10.1 The Full Pipeline — Data Loading to Forward Pass

```python
import numpy as np

# ── STEP 1: SIMULATE LOADING RAW DATA ────────────────────
np.random.seed(42)
n_samples, n_features = 500000, 28
X_raw = np.random.randn(n_samples, n_features).astype(np.float32)
y     = np.random.randint(0, 2, n_samples)   # 0=legit, 1=fraud
# ── STEP 2: NORMALISE (Z-score, broadcasting) ─────────────
mean  = X_raw.mean(axis=0)              # shape (28,)
std   = X_raw.std(axis=0) + 1e-8 # +ε avoids div by zero
X     = (X_raw - mean) / std # (500000, 28) via broadcast
# ── STEP 3: SHUFFLE AND SPLIT ─────────────────────────────
idx   = np.random.permutation(n_samples)
split = int(0.8 * n_samples)           # 80% train, 20% val
X_train, X_val = X[idx[:split]], X[idx[split:]]
y_train, y_val = y[idx[:split]], y[idx[split:]]

# ── STEP 4: INITIALISE WEIGHTS (He init) ──────────────────
W1 = np.random.randn(28, 64) * np.sqrt(2.0 / 28)
b1 = np.zeros(64)
W2 = np.random.randn(64, 1)  * np.sqrt(2.0 / 64)
b2 = np.zeros(1)

# ── STEP 5: FORWARD PASS ON ONE MINI-BATCH ────────────────
batch_idx = np.random.permutation(len(X_train))[:512]
X_batch   = X_train[batch_idx]           # (512, 28)
Z1 = X_batch @ W1 + b1 # (512, 64)
A1 = np.maximum(0, Z1)               # ReLU activation
Z2 = A1 @ W2 + b2 # (512, 1)
probs = 1 / (1 + np.exp(-Z2))       # Sigmoid → fraud probability
print("Batch output shape:", probs.shape)  # (512, 1)
print("Pipeline ready.")
```

> **This is the complete SkySignal preprocessing and neural network forward pass.** Every concept from this project is here: random seed, float32 dtype, Z-score normalisation with broadcasting, random permutation shuffle, train/val split with fancy indexing, He weight initialisation with ufuncs, matrix multiplication with @, ReLU with np.maximum, and sigmoid activation. 25 lines. 18 seconds on 500K samples. No Python loops.

### 11. Quick Reference — Most-Used NumPy Commands

Command

What it does

np.array([...])

Create array from a Python list

np.zeros((r, c))

Array of zeros with given shape

np.ones((r, c))

Array of ones with given shape

np.arange(start, stop, step)

Evenly spaced values (step-based)

np.linspace(start, stop, n)

n evenly spaced values (count-based)

np.eye(n)

n × n identity matrix

arr.shape

Tuple of dimensions

arr.dtype

Data type of elements

arr.ndim

Number of dimensions

arr.size

Total number of elements

arr.nbytes

Total memory used in bytes

arr[row, col]

Single element — 2D indexing

arr[2:5]

Slice — index 2, 3, 4

arr[:, 3]

All rows, column 3

arr[arr > 5]

Boolean mask — filter by condition

arr[[0,2,4]]

Fancy indexing — pick specific rows

arr.reshape(r, c)

Change shape (same total elements)

arr.flatten()

Collapse to 1D (returns copy)

arr.T

Transpose — flip rows and columns

np.vstack([a, b])

Stack vertically (more rows)

np.hstack([a, b])

Stack horizontally (more columns)

np.sum(arr, axis=0)

Sum along columns

np.mean / std / max / min

Aggregate statistics

np.argmax(arr)

Index of maximum value

np.argsort(arr)

Indices that would sort the array

np.sqrt / exp / log / abs

Universal functions (ufuncs)

A @ B

Matrix multiplication

np.dot(a, b)

Dot product (1D) or matmul (2D)

np.linalg.inv(A)

Matrix inverse

np.linalg.norm(v)

Vector/matrix magnitude

np.random.seed(n)

Fix random seed for reproducibility

np.random.randn(r, c)

Standard normal random array

np.random.permutation(n)

Shuffled indices 0..n-1

np.where(cond, x, y)

Vectorised if-else

np.clip(arr, min, max)

Clamp values to [min, max]

np.unique(arr, return_counts=True)

Unique values + counts

arr.astype(np.float32)

Change dtype (e.g. save memory)

arr.copy()

Force an independent copy

np.shares_memory(a, b)

Check if arrays share memory

**Quiz: ❓ Quiz 1 — You have arr = np.arange(24). You want a 3D array with 2 batches of 3 rows × 4 columns. What reshape call do you use?**

- A) arr.reshape(3, 2, 4)
- B) arr.reshape(2, 3, 4)
- C) arr.reshape(24, 1)
- D) arr.reshape(-1, 4)

> **Answer/explanation:** ✅ **Answer: B — arr.reshape(2, 3, 4).** Shape is described as (batches, rows, columns) → (2, 3, 4). Check: 2 × 3 × 4 = 24 ✓. Option A gives (3, 2, 4) which is also 24 elements but has the wrong structure — 3 batches of 2 rows × 4 columns, which isn't what was asked.

**Quiz: ❓ Quiz 2 — What is wrong with this normalisation code? `for i in range(len(X)): X[i] = (X[i] - mean) / std`**

- A) The division should use np.divide() not /
- B) It modifies X in-place which is dangerous
- C) Using a Python for-loop over a NumPy array is 100× slower than broadcasting. Write X = (X - mean) / std
- D) Both B and C are correct

> **Answer/explanation:** ✅ **Answer: D — Both B and C.** The loop is a performance disaster — iterating 500,000 times in Python instead of letting NumPy broadcast the operation in C. And modifying X in-place means the original data is destroyed. The correct approach: `X_norm = (X - mean) / std` — one line, C speed, and X is preserved.

**Quiz: ❓ Quiz 3 — arr = np.array([[1,2,3],[4,5,6]]). What does arr[:, 1] return?**

- A) [[2], [5]] — a 2D column array
- B) [2, 5] — the second column as a 1D array
- C) [1, 2, 3] — the second row
- D) 2 — just the element at row 0, col 1

> **Answer/explanation:** ✅ **Answer: B — [2, 5].** `arr[:, 1]` means "all rows (:), column index 1". Column 1 contains [2, 5] — the second value from each row. The result is a 1D array of shape (2,), not a 2D column. To get a 2D column of shape (2, 1), you'd use `arr[:, 1:2]`.

##### 🙋 Fresher Q&A — Questions You Will Definitely Have

**Q: Q: What's the difference between arr * arr and np.dot(arr, arr) for 2D arrays?**

A: `arr * arr` is element-wise multiplication — each element is squared. Shape stays the same. `np.dot(arr, arr)` is matrix multiplication — rows of the first multiplied by columns of the second. Shape changes: `(m, n) @ (n, k) → (m, k)`. In ML, you almost always want the matrix multiplication version for weight operations, and element-wise for activation functions.

**Q: Q: When should I use np.concatenate vs np.vstack vs np.hstack?**

A: `np.vstack` and `np.hstack` are convenience wrappers around `np.concatenate`. `vstack([a,b])` = `concatenate([a,b], axis=0)`. `hstack([a,b])` = `concatenate([a,b], axis=1)`. Use vstack/hstack for simple 2D operations — they're more readable. Use concatenate when you need to specify an axis other than 0 or 1, especially for 3D+ arrays.

**Q: Q: Why does np.random.randn() give better weight initialisation than np.random.rand()?**

A: `rand()` gives uniform values between 0 and 1 — all weights start positive and clustered near 0.5. This breaks symmetry poorly and causes slow convergence. `randn()` gives values from a standard normal distribution — centred at 0, with both positive and negative values and small magnitude. Starting weights near 0 with both signs is essential for gradients to flow correctly through deep networks during the first training steps.

**Q: Q: My arrays have shape (5,) and (5, 1) — they both have 5 numbers. Are they the same?**

A: No — and this difference causes hard-to-debug errors. `(5,)` is a 1D vector (ndim=1). `(5,1)` is a 2D column vector (ndim=2). They behave differently in matrix multiplication and broadcasting. NumPy's output from operations like `arr[:, 0]` gives shape `(5,)`, but `arr[:, 0:1]` gives `(5, 1)`. When in doubt, use `arr.reshape(-1, 1)` to ensure you have a 2D column shape.

##### ✅ NumPy Best Practices for Real ML Pipelines

- Always set `np.random.seed()` at the top of every script — ensures reproducible experiments.
- Use `float32` instead of `float64` for ML training arrays — halves memory and matches GPU precision.
- Check `arr.shape` before and after every reshape or operation during development — shape bugs are silent and dangerous.
- Never write Python for-loops over NumPy arrays — always find the vectorised equivalent.
- Use `.copy()` when you need to modify a slice without affecting the original array.
- Add a small epsilon (`1e-8`) before dividing by std or log() — prevents divide-by-zero and log(0) instabilities.
- Use `axis=0` for column-wise operations (per feature), `axis=1` for row-wise operations (per sample).
- Prefer the `@` operator over `np.dot()` for 2D matrix multiplication — it's more readable and explicit.

##### 🧪 Practice Exercise — Build the SkySignal Mini-Pipeline Yourself

1. Create a (1000, 5) feature matrix X using `np.random.randn` with seed 0, and a label array y of 0s and 1s using `np.random.randint`.
2. Inspect X: print shape, dtype, ndim, and the min/max of each column using `np.min(X, axis=0)`.
3. Normalise X to zero mean and unit variance per column using broadcasting. Verify: `X_norm.mean(axis=0)` should be close to all zeros.
4. Shuffle and split: create shuffled indices with permutation, use fancy indexing to make an 80/20 train/val split.
5. Extract only the fraud rows (y == 1) from X_norm using boolean masking. Print how many fraud rows you have.
6. Initialise weight matrix W (shape 5×3) with randn, compute Z = X_train @ W, then apply ReLU using np.maximum(0, Z).
7. Find the top-5 rows with the highest sum of activations using argsort.

### NumPy Project Complete 🎉

You have built SkySignal's complete NumPy data preprocessing and neural network forward-pass pipeline — from array creation on day one to indexing, slicing, reshaping, broadcasting, linear algebra, random arrays, views vs copies, and an end-to-end mini-batch forward pass. You now understand the numerical foundation that every ML framework in the world is built on.

> **Dr Ananya**
> 
> "Six weeks ago, Dev's preprocessing loop took 45 minutes. Last night's full training run — 50 epochs over 500,000 transactions, two-layer network, mini-batch SGD — completed in 18 seconds. Same algorithm, same data, same result. The only difference is that every Python loop that touched a number has been replaced with a NumPy operation. That's not just a performance improvement. That's the difference between a prototype and a production system."

> **Dev**
> 
> "The concept that changed my thinking: a NumPy array is not a smarter Python list. It's a completely different object — a chunk of raw memory with a shape, a dtype, and a set of C functions that operate on it without ever touching the Python interpreter. The moment I stopped thinking of it as a list and started thinking of it as a mathematical object, everything — indexing, broadcasting, reshaping, matrix multiply — made intuitive sense."

> **Priya**
> 
> "And the shape error? The one that said shapes (28,) and (28, 10) not aligned? That was two minutes of confusion on day one. Now I automatically reshape every vector before a matrix operation. Check shape before every operation. Understand what axis means. Know when you have a view vs a copy. Those three habits will save you 95% of the debugging time every junior ML engineer spends in their first year."

# Fail-Fast List Iterator in CPython 3.14

> **CPython 3.14 Modification Project**: Enforcing fail-fast behavior on list mutation during iteration using Tier-1 specialized opcode deoptimization and C-level iterator size tracking.

---

## 📌 Table of Contents
1. [Overview & The Bug in Stock Python](#-overview--the-bug-in-stock-python)
2. [Architectural Design & Deoptimization Mechanism](#-architectural-design--deoptimization-mechanism)
3. [Modified Files & Exact Patch Details](#-modified-files--exact-patch-details)
4. [Step-by-Step Reproduction Guide (Steps 1–22)](#-step-by-step-reproduction-guide-steps-122)
5. [Demonstration: Stock vs. Modified Behavior](#-demonstration-stock-vs-modified-behavior)
6. [Performance & Benchmark Analysis](#-performance--benchmark-analysis)
7. [Bytecode Disassembly Analysis (`dis.dis`)](#-bytecode-disassembly-analysis-disdis)
8. [Test Suite Verification](#-test-suite-verification)
9. [Artifacts Included in Repository](#-artifacts-included-in-repository)

---

## 🔍 Overview & The Bug in Stock Python

In stock CPython, iterating over a list while concurrently removing or adding elements produces silent data corruption and unexpected control-flow bugs. 

CPython's list iterator (`_PyListIterObject`) simply maintains a monotonic integer index (`it_index`) that increments on each step:
```text
Iteration 0: index 0 -> yields element 1
Iteration 1: index 1 -> yields element 2  ==> User removes element 2!
             List shrinks and elements shift left!
Iteration 2: index 2 -> yields element 4  ==> Element 3 is SILENTLY SKIPPED!
```

Unlike Python dictionaries and sets (which raise `RuntimeError: dictionary changed size during iteration`), lists historically did not check for concurrent size changes. 

### Our Solution
This project implements a **fail-fast iterator** for Python lists:
If the size of a list changes at any point during iteration, the interpreter immediately aborts the loop and raises:
```text
RuntimeError: list changed size during iteration
```

---

## 🏛 Architectural Design & Deoptimization Mechanism

In modern CPython (3.12+ and 3.14), loops do not simply invoke the C function `tp_iternext` on every iteration. Doing so for millions of iterations would cause severe function-call overhead. 

Instead, PEP 659 introduces a **Specializing Adaptive Interpreter (Tier-1 Quickening)**.

```mermaid
flowchart TD
    A["Python for-loop: for x in lst:"] --> B["Opcode: FOR_ITER (Generic)"]
    B -->|"Adaptive Counter Triggers"| C["Specialized Opcode: FOR_ITER_LIST"]
    C --> D{"List size unchanged?"}
    D -->|"Yes: Fast Path"| E["Direct array index: PyList_GET_ITEM"]
    D -->|"No: List Mutated!"| F["DEOPT_IF triggers"]
    F -->|"Deoptimize"| B
    B --> G["Fallback to tp_iternext (listiter_next)"]
    G --> H{"Check list size in C"}
    H -->|"Mismatch Detected"| I["Raise RuntimeError: list changed size during iteration"]
```

### The Fast Path vs. The Deoptimization Fallback:
1. **Unspecialized Loop (`FOR_ITER`)**: Starts by calling `listiter_next()`.
2. **Specialized Loop (`FOR_ITER_LIST`)**: After observing that the sequence is a list, the bytecode interpreter specializes the instruction to `FOR_ITER_LIST`. It directly accesses `PyList_GET_ITEM` from memory for maximum speed.
3. **Safety Check via `DEOPT_IF`**: We insert `DEOPT_IF(PyList_GET_SIZE(seq) != it->it_size);` at the top of `_ITER_JUMP_LIST` in `Python/bytecodes.c`.
4. **Deoptimization**: If a modification occurs (e.g. `lst.remove(x)` or `lst.append(x)`), the size check fails. The interpreter immediately deoptimizes back to `FOR_ITER`.
5. **Exception Raising**: Generic `FOR_ITER` calls `listiter_next()`, where our C-level size check detects the size difference and cleanly raises `PyExc_RuntimeError`.

---

## 🛠 Modified Files & Exact Patch Details

Four files are modified across the CPython codebase:

### 1. `Include/internal/pycore_list.h`
Add `it_size` to the iterator struct `_PyListIterObject`:
```c
typedef struct {
    PyObject_HEAD
    Py_ssize_t it_index;
    PyListObject *it_seq; /* Set to NULL when iterator is exhausted */
    Py_ssize_t it_size;    /* Snapshot of list size at iterator creation */
} _PyListIterObject;
```

### 2. `Objects/listobject.c`
In `list_iter(PyObject *seq)`:
```c
    it->it_index = 0;
    it->it_seq = (PyListObject *)Py_NewRef(seq);
    it->it_size = PyList_GET_SIZE(seq); // Snapshot original size
    _PyObject_GC_TRACK(it);
    return (PyObject *)it;
```

In `listiter_next(PyObject *self)`:
```c
    // Raise RuntimeError if the list size changed during iteration
    if (it->it_seq != NULL && PyList_GET_SIZE(it->it_seq) != it->it_size) {
        PyErr_SetString(PyExc_RuntimeError,
                        "list changed size during iteration");
        it->it_size = -1; /* Make this state sticky */
        return NULL;
    }
```

### 3. `Python/bytecodes.c`
In opcode macro component `_ITER_JUMP_LIST`:
```c
#else
            _PyListIterObject *it = (_PyListIterObject *)iter_o;
            STAT_INC(FOR_ITER, hit);
            PyListObject *seq = it->it_seq;
            DEOPT_IF(PyList_GET_SIZE(seq) != it->it_size); // Deoptimize if size changed
            if (seq == NULL || (size_t)it->it_index >= (size_t)PyList_GET_SIZE(seq)) {
```

### 4. `Python/generated_cases.c.h`
Regenerated automatically via `make regen-cases` (`Tools/cases_generator/tier1_generator.py`).

---

## 📋 Step-by-Step Reproduction Guide (Steps 1–22)

```bash
# Step 1: Install build dependencies
sudo apt update && sudo apt install -y build-essential git libssl-dev zlib1g-dev libffi-dev

# Step 2: Download CPython source
git clone https://github.com/Beatkingdeepjoy/cpython-failfast-iterator.git && cd cpython-failfast-iterator

# Step 3: Pick version
git checkout feature/fail-fast-list-iter

# Step 4: Build stock Python (prior to patch)
./configure && make -j4

# Step 5: Save stock binary for comparative testing
cp python ../python-stock

# Step 6: Create demo_list.py
cat << 'EOF' > demo_list.py
def main():
    numbers = [1, 2, 3, 4, 5, 6]
    print(f"Starting list: {numbers}")

    for item in numbers:
        print(f"Current item: {item}")
        if item == 2:
            numbers.remove(item)
            print(f"Removed {item}, list is now: {numbers}")

    print(f"Finished iteration. Final list: {numbers}")

if __name__ == "__main__":
    main()
EOF

# Step 7: Run stock and save output
../python-stock demo_list.py > stock_output.txt

# Step 8: Open in editor
code .

# Step 9-12: Apply changes to listobject.c, pycore_list.h, and bytecodes.c
# (Or apply my_changes.patch directly: git apply my_changes.patch)

# Step 13: Regenerate interpreter case files
make regen-cases

# Step 14: Rebuild modified CPython
make -j4

# Step 15: Run demo with modified Python and save output
./python demo_list.py > modified_output.txt

# Step 16: Time stock Python
../python-stock -m timeit -s "lst=list(range(1000000))" "for x in lst: pass"

# Step 17: Time modified Python
./python -m timeit -s "lst=list(range(1000000))" "for x in lst: pass"

# Step 18: Compare bytecode disassembly on both builds
python -c "import dis; dis.dis('for x in lst: pass')"

# Step 19: Check changed lines
git diff --stat

# Step 20: Run regression tests
./python -m test test_list

# Step 21: Save unified diff
git diff > my_changes.patch

# Step 22: Deliver patch, demo_list.py, stock/modified outputs, and timing numbers
```

---

## 🔬 Demonstration: Stock vs. Modified Behavior

### Code: `demo_list.py`
```python
numbers = [1, 2, 3, 4, 5, 6]
for item in numbers:
    print(f"Current item: {item}")
    if item == 2:
        numbers.remove(item)
```

| Stock Python Output (`stock_output.txt`) | Modified Python Output (`modified_output.txt`) |
| :--- | :--- |
| `Starting list: [1, 2, 3, 4, 5, 6]`<br>`Current item: 1`<br>`Current item: 2`<br>`Removed 2, list is now: [1, 3, 4, 5, 6]`<br>`Current item: 4`<br>`Current item: 5`<br>`Current item: 6`<br>`Finished iteration. Final list: [1, 3, 4, 5, 6]`<br><br>*(Silent bug: item 3 skipped)* | `Starting list: [1, 2, 3, 4, 5, 6]`<br>`Current item: 1`<br>`Current item: 2`<br>`Removed 2, list is now: [1, 3, 4, 5, 6]`<br>`Traceback (most recent call last):`<br>`  File "demo_list.py", line 8, in main`<br>`    for item in numbers:`<br>`    ^^^^^^^^^^^^^^^^^^^`<br>`RuntimeError: list changed size during iteration` |

---

## ⚡ Performance & Benchmark Analysis

Benchmarking command across 1,000,000 elements:
```bash
python -m timeit -s "lst=list(range(1000000))" "for x in lst: pass"
```

| Interpreter Build | Time per Loop | Relative Overhead |
| :--- | :--- | :--- |
| **Stock Python 3.14** | `5.72 msec` | Baseline (1.00x) |
| **Modified Python 3.14** | `5.84 msec` | **+2.1%** (~0.12 ms) |

### Why is the overhead virtually zero?
The check `DEOPT_IF(PyList_GET_SIZE(seq) != it->it_size)` compiles to a single CPU register comparison (`cmp`) and conditional jump (`jne`). In the non-mutated path, the CPU branch predictor achieves virtually 100% prediction accuracy, resulting in zero pipeline stalls.

---

## 🧩 Bytecode Disassembly Analysis (`dis.dis`)

Command:
```bash
python -c "import dis; dis.dis('for x in lst: pass')"
```

Emitted Bytecode on **both** builds:
```text
  0           RESUME                   0

  1           LOAD_NAME                0 (lst)
              GET_ITER
      L1:     FOR_ITER                 3 (to L2)
              STORE_NAME               1 (x)
              JUMP_BACKWARD            5 (to L1)
      L2:     END_FOR
              POP_TOP
              RETURN_CONST             0 (None)
```

### Architectural Key Insight:
- Notice the static instruction is `FOR_ITER` on both builds.
- The Python compiler (`compile.c` / `codegen.c`) emits generalized bytecode because variable types are dynamically resolved at runtime.
- Specialization into `FOR_ITER_LIST` occurs **in-memory during execution** via inline caches. Hence, static disassembly remains identical while runtime safety is completely upgraded.

---

## ✅ Test Suite Verification

Run list regression test suite:
```bash
./python -m test test_list
```

Output:
```text
== Tests result: SUCCESS ==
1 test OK.
Total duration: 1.2 sec
```
All standard list operations (`append`, `extend`, `sort`, `slice`, `reverse`, `copy`, and pickle protocols) pass completely.

---

## 📦 Artifacts Included in Repository

- **`my_changes.patch`**: Unified patch containing all modifications.
- **`demo_list.py`**: Minimal reproduction script triggering concurrent list mutation.
- **`stock_output.txt`**: Execution output showing element skipping on stock Python.
- **`modified_output.txt`**: Execution output showing `RuntimeError` on modified Python.
- **`Include/internal/pycore_list.h`**: Iterator struct update.
- **`Objects/listobject.c`**: Iterator initialization and next checks.
- **`Python/bytecodes.c`**: Specialization deoptimization guard.
- **`Python/generated_cases.c.h`**: Regenerated interpreter case switch.

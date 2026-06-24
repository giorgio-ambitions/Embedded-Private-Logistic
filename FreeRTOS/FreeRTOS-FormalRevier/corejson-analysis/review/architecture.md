## 1. Purpose and Scope
coreJSON is a deterministic, allocation‑free JSON parser intended for embedded systems with strict requirements regarding memory usage, execution time, and correctness.  
All parsing is performed iteratively without recursion.  
The module operates exclusively on caller‑provided buffers and does not allocate or modify memory unless explicitly documented.

## 2. System Boundary Definition
**Input:**  
A contiguous character buffer (`char*`) and its length (`size_t`).

**Processing:**  
Deterministic advancement of an index through the buffer using small, pure functions.

**Output:**  
Status codes, pointer/length pairs referencing the original buffer, and optional type information.

**Excluded:**  
- No dynamic memory allocation  
- No recursion  
- No locale‑dependent behavior  
- No numeric conversion  
- No modification of the input buffer (except when the caller explicitly inserts a temporary `'\0'`)

## 3. Design Principles
- Deterministic control flow (no backtracking, no heuristics)  
- Strict bounds checking for all pointer and index operations  
- Constant memory footprint (no dynamic allocation)  
- Locale‑independent character classification  
- MISRA‑conformant structure  
- Zero modification of input buffer unless explicitly documented  
- Linear time complexity with predictable execution paths  

## 4. Parsing Invariants
- `0 ≤ start ≤ max` at all times  
- Index variables only advance; they never rewind  
- All pointer parameters are validated before use  
- UTF‑8 sequences must be shortest‑form  
- Escape sequences must be complete and syntactically correct  
- Nesting depth must not exceed `JSON_MAX_DEPTH`  
- No function reads beyond buffer bounds  
- No function modifies the input buffer  

## 5. Architectural Overview

### 5.1 Character Classification
Custom ASCII‑only macros (`isdigit_`, `isspace_`, `iscntrl_`) avoid undefined behavior from signed `char` and ensure deterministic classification.

### 5.2 UTF‑8 Validation
Functions such as `skipUTF8()` and `skipUTF8MultiByte()` enforce:
- shortest‑form UTF‑8  
- valid continuation bytes  
- exclusion of surrogate ranges  
- maximum code point `0x10FFFF`  

Invalid sequences cause immediate failure.

### 5.3 Escape Sequence Handling
`skipEscape()` and related helpers validate:
- standard JSON escapes (`\"`, `\\`, `\n`, etc.)  
- Unicode escape sequences (`\uXXXX`)  
- surrogate pair correctness  

All operations include explicit bounds checks.

### 5.4 String Parsing
`skipString()` implements a deterministic finite‑state machine:
- opening quote  
- UTF‑8 or escape sequence  
- closing quote  

Unescaped control characters or invalid UTF‑8 terminate parsing.

### 5.5 Number Parsing
`skipNumber()` validates JSON numbers according to ECMA‑404:
- optional minus  
- integer part without leading zeros  
- optional decimal  
- optional exponent  

No numeric conversion is performed.

### 5.6 Literal Parsing
`skipAnyLiteral()` recognizes fixed sequences:
- `"true"`  
- `"false"`  
- `"null"`  

### 5.7 Scalar Parsing
`skipAnyScalar()` unifies:
- strings  
- numbers  
- literals  

### 5.8 Collection Parsing
Arrays and objects are validated using:
- bracket matching  
- depth counter (no recursion)  
- comma rules (no trailing comma)  

### 5.9 Public API
- `JSON_Validate()` — full document validation  
- `JSON_Search()` / `JSON_SearchT()` — deterministic path‑based lookup  
- `JSON_Iterate()` — iterative traversal of arrays/objects  

## 6. Data Flow Description
**Input → Processing → Output**

**Input:**  
Raw JSON buffer (`char* buf`, `size_t max`)

**Processing:**  
- Index advancement  
- UTF‑8 validation  
- Escape handling  
- Structural validation  
- Depth tracking  

**Output:**  
- Status code (`JSONStatus_t`)  
- Pointer to value (`char*` or `const char*`)  
- Length of value (`size_t`)  
- Optional type (`JSONTypes_t`)  

No intermediate buffers are created.

## 7. Failure Modes
coreJSON fails deterministically under the following conditions:

- Invalid UTF‑8 sequence  
- Unescaped control character  
- Incomplete escape sequence  
- Leading zero violation  
- Trailing comma  
- Surrogate mismatch  
- Nesting depth overflow  
- Null pointer parameter  
- Index exceeding buffer bounds  

Each failure maps to a specific status code.

## 8. Deterministic Behavior Guarantees
- no undefined behavior  
- no recursion  
- no dynamic memory  
- no locale‑dependent behavior  
- constant stack usage  
- predictable execution time  
- strict input validation  

## 9. Timing Characteristics
All parsing functions operate in **linear time O(n)** with:
- no recursion  
- no backtracking  
- no dynamic allocation  

Worst‑case execution time is bounded by buffer length.

## 10. Memory Model
- No heap usage  
- Constant stack usage  
- No temporary buffers  
- All returned pointers reference caller‑owned memory  

## 11. Safety and Compliance
The implementation aligns with MISRA‑C principles:
- explicit checks for all pointer parameters  
- no implicit type conversions  
- no reliance on implementation‑defined behavior  
- clear separation of concerns  
- deterministic error handling  

## 12. Compliance Matrix

| Requirement              | Mechanism                                  | Status |
|--------------------------|--------------------------------------------|--------|
| Deterministic control flow | No recursion, no backtracking           | ✔      |
| Memory safety            | Bounds checks, no dynamic allocation      | ✔      |
| UTF‑8 correctness        | `skipUTF8`, `skipUTF8MultiByte`           | ✔      |
| JSON compliance          | ECMA‑404                                  | ✔      |
| MISRA alignment          | No undefined behavior                     | ✔      |
| Predictable timing       | Linear time, no recursion                 | ✔      |
| Robust error handling    | Explicit status codes                     | ✔      |

## 13. Summary
coreJSON is compact, deterministic, and safety‑oriented.  
Its architecture reflects engineering expectations:
- clear structure  
- predictable behavior  
- strict correctness  
- robust handling of malformed input  


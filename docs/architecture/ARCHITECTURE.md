# Soplang Python Implementation — System Design & Architecture

> **Audience:** Contributors, language implementers, and anyone curious about how the Python reference interpreter works.

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [Pipeline & Data Flow](#2-pipeline--data-flow)
3. [Module Map](#3-module-map)
4. [Lexer](#4-lexer)
5. [Parser & AST](#5-parser--ast)
6. [Interpreter](#6-interpreter)
7. [Type System](#7-type-system)
8. [Scoping Model](#8-scoping-model)
9. [Control Flow Signals](#9-control-flow-signals)
10. [Class System](#10-class-system)
11. [Import System](#11-import-system)
12. [Standard Library](#12-standard-library)
13. [Error Handling](#13-error-handling)
14. [REPL / Interactive Shell](#14-repl--interactive-shell)
15. [Key Design Patterns](#15-key-design-patterns)
16. [Known Limitations](#16-known-limitations)

---

## 1. High-Level Overview

Soplang is a **dynamically-typed, tree-walking interpreted language** with Somali-language keywords. It supports both a scripting mode (`.sop` files) and an interactive REPL.

The architecture is a classic **three-stage pipeline**:

```
Source text (.sop)
      │
      ▼
  [ Lexer ]  ──── characters ──→ Token stream
      │
      ▼
  [ Parser ] ──── tokens ──────→ Abstract Syntax Tree (AST)
      │
      ▼
[ Interpreter ] ── AST nodes ──→ Side effects / return values
```

All three stages are hand-written (no parser-generator tooling). The implementation deliberately avoids external parsing libraries to keep the codebase readable and portable.

---

## 2. Pipeline & Data Flow

### Scripting Mode

```
CLI entry  (src/__main__.py)
    │  reads source file
    ▼
Lexer(source_code).tokenize()
    │  returns List[Token]
    ▼
Parser(tokens).parse()
    │  returns ASTNode(PROGRAM, children=[...])
    ▼
Interpreter().interpret(ast)
    │  mutates self.variables, self.functions, self.classes
    ▼
stdout / stderr
```

### REPL Mode

```
SoplangShell.__init__()
    │  creates a long-lived Interpreter instance
    ▼
loop:
    input_line = prompt("> ")
    │
    ▼
  Lexer + Parser + Interpreter.interpret()  (same pipeline, shared state)
    │
    ▼
  print result or error message
```

The **Interpreter instance is shared across REPL iterations**, which is what gives REPL sessions their persistent variable/function state.

---

## 3. Module Map

```
src/
├── __init__.py
├── __main__.py          # CLI entry point (argparse, --run / --shell / -c)
│
├── core/
│   ├── tokens.py        # TokenType enum  (all ~50 token kinds)
│   ├── lexer.py         # Lexer class  +  Token dataclass
│   ├── ast.py           # NodeType enum  +  ASTNode class
│   └── parser.py        # Parser class  (recursive descent)
│
├── runtime/
│   ├── interpreter.py   # Interpreter class  (tree-walking evaluator)
│   ├── main.py          # run_file() / run_code() helpers
│   └── shell.py         # SoplangShell (REPL)
│
├── stdlib/
│   └── builtins.py      # SoplangBuiltins + factory functions for
│                        # built-in fns, list methods, object methods,
│                        # string methods
│
└── utils/
    └── errors.py        # SoplangError hierarchy + ErrorMessageManager
                         # + control-flow signal exceptions
```

**Dependency direction** (no cycles):

```
__main__ / shell / main
        │
        ▼
  interpreter  ←── stdlib/builtins
        │
        ▼
  core/parser  ←── core/ast, core/tokens
        │
        ▼
  core/lexer   ←── core/tokens
        │
        ▼
  utils/errors        (imported by every layer)
```

---

## 4. Lexer

**File:** `psrc/core/lexer.py`

### Algorithm

The lexer is a **single-pass, character-by-character scanner** driven by a simple `while` loop. It maintains a cursor (`self.position`) over the raw source string.

```
tokenize():
  tokens = []
  while current_char is not None:
    skip_whitespace()
    if at comment:  skip_comment()
    elif digit:     tokens.append(read_number())
    elif letter/_:  tokens.append(read_identifier_or_keyword())
    elif quote:     tokens.append(read_string())
    else:           tokens.append(match_operator_or_punct())
  tokens.append(EOF token)
  return tokens
```

### Key Data Structures

| Structure | Type | Purpose |
|---|---|---|
| `self.source` | `str` | Full source text, never mutated |
| `self.position` | `int` | Current cursor index |
| `self.line` / `self.column` | `int` | Location tracking for error messages |
| `self.current_char` | `str \| None` | Look-ahead character (position 0) |
| `self.KEYWORDS` | `dict[str, TokenType]` | Keyword → token type lookup |

### Token Dataclass

```python
class Token:
    type:     TokenType   # enum variant
    value:    Any         # lexeme or literal value
    line:     int
    position: int         # column
```

### Keyword Recognition

Keywords are detected **after** a full identifier is scanned. `read_identifier()` reads all `[a-zA-Z_][a-zA-Z0-9_]*` characters, then looks up the result in `self.KEYWORDS`. If found → keyword token; otherwise → `IDENTIFIER` token. This is the standard "maximal munch" approach.

### Number Literals

`read_number()` consumes digits, then checks for a `.` followed by another digit to produce a float. Numbers are stored as Python `int` or `float` directly in `Token.value`.

### String Literals

`read_string()` handles both `"..."` and `'...'` delimiters. It processes escape sequences (`\n`, `\t`, `\\`, `\"`, `\'`) inline during scanning. An unterminated string raises `LexerError("unterminated_string")`.

### Operator Disambiguation

Two-character operators (`==`, `!=`, `<=`, `>=`, `&&`, `||`) are detected by peeking at `position + 1` before consuming the first character, using a `peek()` helper:

```python
def peek(self):
    next_pos = self.position + 1
    return self.source[next_pos] if next_pos < len(self.source) else None
```

### Comments

- `// ...` — single-line: advance until `\n`.
- `/* ... */` — multi-line: advance until `*/`, counting newlines for correct error reporting.

---

## 5. Parser & AST

**Files:** `psrc/core/parser.py`, `psrc/core/ast.py`

### Algorithm

The parser is a **hand-written recursive descent parser**. Each grammar rule is a method. The entry point is `parse()`, which calls `parse_statement()` in a loop until `EOF`.

```
parse()
  └─ parse_statement()
       ├─ parse_var_declaration()   door / madoor / abn / qoraal / ...
       ├─ parse_function_definition()  hawl
       ├─ parse_if_statement()         haddii
       ├─ parse_switch_statement()     dooro
       ├─ parse_for_loop()             kuceli
       ├─ parse_while_loop()           intay
       ├─ parse_return_statement()     celi
       ├─ parse_import_statement()     ka_keen
       ├─ parse_class_definition()     fasalka
       ├─ parse_try_catch()            isku_day / qabo
       └─ parse_expression_statement() (fallthrough)
            └─ parse_expression()
                 └─ parse_assignment()
                      └─ parse_or()
                           └─ parse_and()
                                └─ parse_equality()
                                     └─ parse_comparison()
                                          └─ parse_additive()
                                               └─ parse_multiplicative()
                                                    └─ parse_unary()
                                                         └─ parse_postfix()
                                                              └─ parse_primary()
```

Operator precedence is enforced naturally by the **call chain depth** — lower precedence operators are higher in the call stack.

### Precedence Table (low → high)

| Level | Operators | Method |
|---|---|---|
| 1 (lowest) | assignment `=` | `parse_assignment` |
| 2 | logical OR `\|\|` | `parse_or` |
| 3 | logical AND `&&` | `parse_and` |
| 4 | equality `==` `!=` | `parse_equality` |
| 5 | comparison `<` `>` `<=` `>=` | `parse_comparison` |
| 6 | additive `+` `-` | `parse_additive` |
| 7 | multiplicative `*` `/` `%` | `parse_multiplicative` |
| 8 | unary `!` `-` | `parse_unary` |
| 9 | postfix `.` `[` `(` | `parse_postfix` |
| 10 (highest) | literals, identifiers, `(expr)` | `parse_primary` |

### AST Node

All AST nodes use a single generic class:

```python
class ASTNode:
    type:        NodeType   # enum variant
    value:       Any        # node-specific payload (name, literal, operator, ...)
    children:    list[ASTNode]
    var_type:    str | None  # for typed declarations
    is_constant: bool        # for madoor
    line:        int | None
    position:    int | None
```

The generic design trades some type-safety for simplicity — any node can hold arbitrary children. The interpreter pattern-matches on `node.type` to determine how to interpret `node.value` and `node.children`.

### NodeType Enum

```
PROGRAM               VARIABLE_DECLARATION   FUNCTION_DEFINITION
FUNCTION_CALL         IF_STATEMENT           SWITCH_STATEMENT
LOOP_STATEMENT        WHILE_STATEMENT        BLOCK
BINARY_OPERATION      UNARY_OPERATION        LITERAL
IDENTIFIER            CLASS_DEFINITION       IMPORT_STATEMENT
TRY_CATCH             BREAK_STATEMENT        CONTINUE_STATEMENT
RETURN_STATEMENT      LIST_LITERAL           OBJECT_LITERAL
PROPERTY_ACCESS       METHOD_CALL            INDEX_ACCESS
ASSIGNMENT
```

### Parser Utilities

- `advance()` — move to next token.
- `expect(token_type)` — assert current token type, advance, or raise `ParserError`.
- `peek(offset=1)` — look ahead without consuming.

---

## 6. Interpreter

**File:** `psrc/runtime/interpreter.py`

### Architecture: Tree-Walking Evaluator

The interpreter **walks the AST recursively**. There is no bytecode compilation or optimization step. Every node is visited at runtime.

Two core methods handle all execution:

```
execute(node)   → for statements  (side effects; return value usually ignored)
evaluate(node)  → for expressions (always returns a Python value)
```

`execute()` dispatches on `node.type` via a chain of `elif` branches, delegating to specialized methods:

| Node type | Method |
|---|---|
| `VARIABLE_DECLARATION` | `execute_var_declaration` |
| `FUNCTION_DEFINITION` | `define_function` |
| `FUNCTION_CALL` | `execute_function_call` |
| `IF_STATEMENT` | `execute_if_statement` |
| `SWITCH_STATEMENT` | `execute_switch_statement` |
| `LOOP_STATEMENT` | `execute_loop_statement` |
| `WHILE_STATEMENT` | `execute_while_statement` |
| `ASSIGNMENT` | `execute_assignment` |
| `CLASS_DEFINITION` | `execute_class_definition` |
| `IMPORT_STATEMENT` | `execute_import_statement` |
| `TRY_CATCH` | `execute_try_catch` |
| `BLOCK` | `execute_block` |
| `BREAK_STATEMENT` | `raise BreakSignal()` |
| `CONTINUE_STATEMENT` | `raise ContinueSignal()` |
| `RETURN_STATEMENT` | `raise ReturnSignal(value)` |

`evaluate()` handles expression nodes and maps them to Python values:

| Node type | Result |
|---|---|
| `LITERAL` | `node.value` directly |
| `IDENTIFIER` | `self.variables[node.value]` |
| `BINARY_OPERATION` | `apply_operator(op, left, right)` |
| `UNARY_OPERATION` | `not bool(operand)` for `!` |
| `LIST_LITERAL` | `[evaluate(e) for e in node.children]` |
| `OBJECT_LITERAL` | `{key: evaluate(val) for ...}` |
| `PROPERTY_ACCESS` | `obj[prop_name]` |
| `INDEX_ACCESS` | `arr[int(idx)]` |
| `METHOD_CALL` | dispatch to list/object/string/user method |
| `FUNCTION_CALL` | `execute_function_call(node)` |

### Interpreter State

```python
class Interpreter:
    variables:          dict[str, Any]        # all in-scope variables (flat, global)
    variable_types:     dict[str, str]        # declared static types
    constant_variables: set[str]              # madoor names (immutable)
    functions:          dict[str, callable | dict]  # built-ins + user functions
    list_methods:       dict[str, callable]
    object_methods:     dict[str, callable]
    string_methods:     dict[str, callable]
    classes:            dict[str, dict]       # class definitions
    call_stack:         list                  # for future stack-trace support
```

---

## 7. Type System

Soplang supports **optional static typing** alongside dynamic typing.

### Static Type Keywords

| Soplang keyword | Python type enforced |
|---|---|
| `abn` | `int` |
| `jajab` | `float` |
| `qoraal` | `str` |
| `bool` | `bool` |
| `teed` | `list` |
| `walax` | `dict` |

When a variable is declared with an explicit type, the interpreter stores the type name in `self.variable_types[var_name]` and calls `validate_type()` before assignment. On reassignment, the same check is applied.

```python
# Soplang source
abn da = 25        # da must always be int
da = "miro"        # → TypeError: type_mismatch
```

### Dynamic Typing

Variables declared with `door` (mutable) or `madoor` (constant) without a type annotation accept any value:

```python
door magac = "Soplang"   # no type constraint
```

### Runtime Value Mapping

Soplang values are Python native values at runtime:

| Soplang concept | Python representation |
|---|---|
| Integer | `int` |
| Decimal | `float` |
| String | `str` |
| Boolean (`run`/`been`) | `bool` |
| Null (`null`) | `None` |
| List | `list` |
| Object | `dict` |
| User function | `dict` with keys `params`, `body`, `closure_vars` |
| Class definition | `dict` with keys `name`, `parent`, `methods`, `fields` |
| Class instance | `dict` with key `__class__` + instance fields |

---

## 8. Scoping Model

The Python implementation uses a **single flat dictionary** (`self.variables`) for all variable storage.

### Function Call Scope

When a user-defined function is called:

1. The current variable dictionary is **shallow-copied** (`old_vars = self.variables.copy()`).
2. Parameters are bound into `self.variables`.
3. The function body executes.
4. `self.variables` is **restored** to `old_vars`.

```python
# Simplified from execute_function_call()
old_vars = self.variables.copy()
for i, param in enumerate(params):
    self.variables[param] = args[i]
try:
    for stmt in body:
        self.execute(stmt)
except ReturnSignal as ret:
    result = ret.value
self.variables = old_vars
```

**Implication:** Functions do not have closure access to variables defined after the function definition — only a snapshot of the global state at call time is available. This differs from a proper lexical scoping model and is a known limitation (see §16).

### Class Method Scope

Class methods are stored as `ASTNode` objects in the class definition dict. When called on an instance, `self` (`nafta`) is injected into `self.variables["nafta"]` before executing the method body.

---

## 9. Control Flow Signals

Non-local jumps (`break`, `continue`, `return`) are implemented using **Python exceptions as signals**. These are not error conditions; they are control flow mechanisms.

```python
class BreakSignal(Exception):    pass
class ContinueSignal(Exception): pass
class ReturnSignal(Exception):
    def __init__(self, value=None): self.value = value
```

### How Loops Catch Signals

```python
# execute_while_statement (simplified)
while self.evaluate(condition_node):
    try:
        for stmt in body_nodes:
            self.execute(stmt)
    except BreakSignal:
        break       # exit the while loop
    except ContinueSignal:
        pass        # skip to next iteration
```

`ReturnSignal` propagates up until caught by `execute_function_call()`:

```python
try:
    for stmt in body:
        self.execute(stmt)
except ReturnSignal as ret:
    result = ret.value
```

At the top level (`interpret()`), uncaught `BreakSignal`, `ContinueSignal`, and `ReturnSignal` are converted into `RuntimeError` with appropriate Somali messages.

---

## 10. Class System

**Soplang keyword:** `fasalka` (define), `cusub` (instantiate), `nafta` (self), `ka_dhaxal` (inherit)

### Class Definition Storage

Classes are stored as plain Python dicts in `self.classes`:

```python
class_def = {
    "name":    "Dog",
    "parent":  "Animal",   # or None
    "methods": { "bark": ASTNode(...) },
    "fields":  { "name": "unknown" },
}
```

### Instantiation (`cusub`)

Creating an instance produces a plain `dict` with a `__class__` marker plus all inherited and own fields. Method resolution walks the inheritance chain via `__proto__`:

```python
instance = {
    "__class__": "Dog",
    "__proto__": parent_instance_or_class_def,
    "name": "Rex",
    ...
}
```

### Method Dispatch

When `instance.method(args)` is called:

1. Check `instance` dict for the method name directly.
2. If not found, walk up `__proto__` chain.
3. Inject `nafta = instance` into `self.variables` before executing the method body.
4. Restore `nafta` after execution.

### Inheritance

`ka_dhaxal` stores the parent class name in `class_def["parent"]`. When an instance is created, parent fields and methods are merged in, with child definitions taking precedence (child-wins override).

---

## 11. Import System

**Keyword:** `ka_keen "filename.sop"`

The import mechanism is **flat namespace** — the imported file's declarations are executed in the current interpreter's scope. There is no module object or namespace isolation.

```python
def execute_import_statement(self, node):
    filename = node.value
    with open(filename) as f:
        code = f.read()
    # Re-use the full pipeline
    lexer = Lexer(code)
    tokens = lexer.tokenize()
    parser = Parser(tokens)
    ast = parser.parse()
    for stmt in ast.children:
        self.execute(stmt)   # executed in the same interpreter scope
```

**Implication:** A variable named `x` in an imported file will overwrite any existing `x` in the importing file. There is no protection against name collisions and no circular import detection.

---

## 12. Standard Library

**File:** `psrc/stdlib/builtins.py`

Built-in functions and methods are registered at `Interpreter.__init__()` time via factory functions that return plain Python dicts mapping Somali names to Python callables.

### Built-in Functions (`get_builtin_functions()`)

Selected functions:

| Soplang name | Behavior |
|---|---|
| `qor` | `print()` |
| `gelin` | `input()` |
| `nooc` | `type()` — returns Somali type name |
| `abn()` | `int()` cast |
| `jajab()` | `float()` cast |
| `qoraal()` | `str()` cast |
| `bool()` | `bool()` cast |
| `dherer` | `len()` |
| `xid` | `range()` / list creation |
| `ururso` | `sorted()` |
| `badal` | `map()` |
| `shaandhee` | `filter()` |
| `tiro_random` | `random.randint()` |
| `wakhtiga` | timestamp |

### List Methods (`get_list_methods()`)

| Soplang name | Equivalent |
|---|---|
| `kudar` | `list.append()` |
| `kasaar` | `list.pop()` |
| `raadi` | `list.index()` |
| `leeyahay` | `value in list` |
| `kala_sooc` | `list.sort()` |
| `rogrog` | `list.reverse()` |
| `mideeya` | `list.join()` |
| `jar` | `list slicing` |

### Object Methods (`get_object_methods()`)

| Soplang name | Equivalent |
|---|---|
| `fure` | `dict.keys()` |
| `qiime` | `dict.values()` |
| `leeyahay` | `key in dict` |
| `kasaar` | `dict.pop()` |

### String Methods (`get_string_methods()`)

| Soplang name | Equivalent |
|---|---|
| `dherer` | `len(str)` |
| `weyn` | `str.upper()` |
| `yar` | `str.lower()` |
| `jar` | `str slicing` |
| `leeyahay` | `substr in str` |
| `beddel` | `str.replace()` |
| `kala_qaybi` | `str.split()` |

---

## 13. Error Handling

**File:** `psrc/utils/errors.py`

### Class Hierarchy

```
Exception
└── SoplangError                    (base)
    ├── LexerError
    ├── ParserError
    ├── TypeError
    ├── RuntimeError
    ├── ImportError
    ├── ValueError
    └── NameError

Exception (non-error signals)
    ├── BreakSignal
    ├── ContinueSignal
    └── ReturnSignal
```

### Error Message Internationalization

All user-facing error messages are in **Somali**, managed by `ErrorMessageManager`:

- Templates are stored in class-level dicts (`LEXER_ERRORS`, `PARSER_ERRORS`, `TYPE_ERRORS`, `RUNTIME_ERRORS`, `IMPORT_ERRORS`).
- `format_error(error_type, code, line, position, **kwargs)` looks up the template by key and substitutes named placeholders.
- Every error class constructor calls `format_error()` to build its message.

### Error Format

```
Khalad <type>: <message> sadar <line>, goobta <col>
```

Example:
```
Khalad runtime: Doorsame aan la qeexin: 'x' sadar 3, goobta 5
```

### Error Propagation

Errors raised in the interpreter bubble up through the recursive `execute`/`evaluate` call stack. The top-level entry point in `psrc/runtime/main.py` catches all `SoplangError` subclasses and prints the message to stderr.

---

## 14. REPL / Interactive Shell

**File:** `psrc/runtime/shell.py`

`SoplangShell` wraps a **persistent `Interpreter` instance** and a `readline`/`prompt_toolkit` input loop.

### Architecture

```
SoplangShell.__init__()
├── self.interpreter = Interpreter()      ← shared state across REPL iterations
├── setup_history()                       ← ~/.soplang_history
└── self.commands = { "help": ..., ... }  ← built-in shell commands

run():
  loop:
    line = input(prompt)
    if line starts with shell command:
        dispatch to self.commands[cmd]()
    elif in multiline mode:
        accumulate lines
    else:
        Lexer(line) → Parser → Interpreter.interpret()
        print result
```

### Shell Commands

`help`, `exit`/`quit`, `clear`, `load`, `run`, `examples`, `example`, `reset`, `vars`, `multiline`

### Multiline Mode

`multiline` command toggles collection of multiple input lines before executing, enabling the user to write multi-line functions/classes interactively. Lines are accumulated in `self.multiline_input` until a blank line is submitted.

### Platform Differences

| Platform | History/Input library |
|---|---|
| Linux / macOS | `readline` (built-in) |
| Windows | `prompt_toolkit` with `FileHistory` (if installed) |

---

## 15. Key Design Patterns

### Visitor-like Dispatch via `isinstance` / `node.type`

Rather than implementing the formal Visitor pattern (separate `visit_X` classes), the interpreter uses a single `execute()`/`evaluate()` method with a long chain of `elif node.type == NodeType.X` branches. This is simpler but harder to extend without modifying the central dispatch method.

### Signals as Exceptions

Using `BreakSignal`, `ContinueSignal`, `ReturnSignal` as Python exceptions for control flow is a pragmatic pattern: it leverages Python's exception unwinding to perform non-local jumps without threading return-codes through every method. The cost is a slight performance overhead and the risk of accidentally catching them in overly-broad `except Exception` blocks (see `execute_try_catch`).

### Dictionary-as-Object

Class instances, class definitions, and user-defined functions are all represented as Python `dict`s with well-known key conventions (`__class__`, `__proto__`, `params`, `body`). This is a duck-typed approach — any dict with the right keys can be treated as a class instance. It is flexible but not type-safe.

### Centralized Error Catalog

`ErrorMessageManager` acts as a centralized registry of all possible error messages, keyed by short string codes. This means adding a new error requires touching only one file, and all messages are in one place for future i18n work.

---

## 16. Known Limitations

| Limitation | Impact |
|---|---|
| **Flat scope** — `self.variables` is a single global dict, snapshot-copied for function calls | No closures; functions can't access variables created after their definition; no lexical scoping |
| **No proper closure support** | Higher-order functions that rely on captured variables from outer scopes will behave incorrectly |
| **No circular import detection** | `ka_keen "a.sop"` from within `a.sop` will recurse infinitely |
| **Flat import namespace** | Imported names can overwrite existing variables silently |
| **Class system is dict-based** | No real method resolution order (MRO) for diamond inheritance; no `super()` equivalent |
| **`execute_try_catch` catches all `Exception`** | This includes `BreakSignal`, `ContinueSignal`, and `ReturnSignal`, which can corrupt control flow inside `isku_day` blocks that contain loops |
| **No garbage collection awareness** | Python's GC handles memory; large programs are bound by Python's own overhead |
| **No tail-call optimization** | Deep recursion hits Python's default recursion limit (~1000 frames) |
| **Tree-walking performance** | Each node visit has Python method call overhead; not suitable for compute-intensive workloads |

These limitations are the primary motivations for the **Rust rewrite** described in `IMPLEMENTATION_PLAN.md`.

---

*Generated from the `psrc/` codebase. For the Rust implementation plan, see [`IMPLEMENTATION_PLAN.md`](../../IMPLEMENTATION_PLAN.md). For a general project overview, see [`README.md`](../../README.md).*

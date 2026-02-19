# Soplang Documentation (Python Interpreter – Archived)

This documentation describes the **legacy Python interpreter implementation** of Soplang.  
The **active Soplang implementation (compiler + runtime in Rust)** now lives in the main repository:  
[`soplang/soplang`](https://github.com/soplang/soplang)

Soplang is the Somali programming language, designed to make programming accessible to Somali speakers worldwide.

## Documentation Sections

### Getting Started
- [Installation Guide](installation.md) - Detailed instructions for installing Soplang on Windows, Linux, and macOS

### Language Reference
- [Keywords and Grammar](language/keywords.md) - Complete reference of Soplang keywords and language structure
- [Expressions](language/expressions.md) - Detailed explanation of expressions and operator usage
- [Grammar Specification](language/grammar.md) - Formal grammar specification in EBNF

### Examples
- [Examples Guide](examples/EXAMPLES.md) - Guide to the example programs
- [Test Examples](examples/TEST_EXAMPLES_README.md) - Documentation for test examples

### Building and Performance
- [Build Guide](build/BUILD.md) - How to build Soplang from source
- [Performance](build/PERFORMANCE.md) - Performance benchmarks and optimization techniques
- [C Implementation](build/README_C.md) - Information about the C implementation

### Testing
- [Testing Guide](testing/TESTING.md) - How to test Soplang
- [Test README](testing/README-TESTS.md) - Additional test documentation

## Getting Started

### Installation (legacy)

To install the legacy Python interpreter, see the [Installation Guide](installation.md) which covers all platforms:
- Windows installation (using installer or building from source)
- Linux installation (using package manager or building from source)
- macOS installation (using DMG or building from source)

### Running Soplang

To run Soplang after installation, you can:

```bash
# Run a Soplang program
soplang examples/hello_world.so

# Start the interactive shell
soplang
```

If you haven't installed Soplang yet, you can run it directly from the source:

```bash
python main.py examples/hello_world.so
```

Or start the interactive shell:

```bash
python main.py
```

## Further Resources

- Website: [https://www.soplang.org/](https://www.soplang.org/)
- Legacy Python interpreter repo (this): [https://github.com/soplang/soplang-interpreter](https://github.com/soplang/soplang-interpreter)
- Active Soplang compiler/runtime: [https://github.com/soplang/soplang](https://github.com/soplang/soplang)

# Soplang (Python Interpreter – Archived)

> The Somali Programming Language

**Status: Archived.**  
This repository contains the **legacy Python interpreter implementation** of Soplang and is kept **for historical and reference purposes only**.

The **active Soplang implementation (compiler + runtime in Rust)** now lives in the main repository:

- **The compiler:** [`soplang/soplang`](https://github.com/soplang/soplang)

For all new projects, documentation, and releases, **use the main Soplang repository** instead of this one.

---

## What this repository is

- **Legacy Python interpreter**: Tree‑walking interpreter and tooling written in Python.
- **Reference for language behavior**: Useful for understanding the original interpreter semantics and examples.
- **No longer developed**: There is no active feature development or bug fixing here.

If you are looking for the current Soplang compiler, performance work, or production‑ready binaries, go to [`soplang/soplang`](https://github.com/soplang/soplang).

---

## Language features (unchanged conceptually)

Soplang itself is a programming language with syntax inspired by Somali, making programming more accessible to Somali speakers. It combines static and dynamic typing systems in one elegant language with a focus on clarity and ease of use.

- **Powerful type system** – Combines both static typing (`abn`, `qoraal`, etc.) and dynamic typing (`door`) in one language
- **Somali-based syntax** – Programming concepts expressed in Somali
- **Modern paradigms** – Supports functional, procedural, and object-oriented programming
- **Type safety** – Enforces type checking at runtime
- **Interactive shell** – REPL for quick experimentation
- **File extensions** – Uses `.sop` (primary) and `.so` (secondary) file extensions

> **Note:** The Python-based tooling and installers described below are part of this legacy interpreter. They are not maintained and may be outdated. For an up‑to‑date toolchain, see [`soplang/soplang`](https://github.com/soplang/soplang).

## Example

```js
// Hello World
qor("Salaan, Adduunka!")  // Prints: Hello, World!

// Variables with dual type system in action
door magac = "Sharafdin"  // Dynamic typing - can change type later
door num = 10             // Also dynamic
num = "new value"         // Valid: dynamic variables can change types

// Static typing examples
qoraal name = "Sharafdin" // Static typing - string only
abn age = 25             // Static typing - number only

// Type safety enforcement
// age = "25"             // Would cause runtime error - type mismatch

// Control flow
haddii (age > 18) {
    qor("Waa qof weyn")   // Is an adult
} ugudambeyn {
    qor("Waa qof yar")    // Is a child
}

// Functions
hawl salaam(qof) {
    celi "Salaan, " + qof + "!"
}
qor(salaam(magac))        // Prints: Salaan, Sharafdin!
```

## Documentation

- **Legacy docs (Python interpreter):**
  - [Getting Started](docs/index.md)
  - [Language Reference](docs/language/keywords.md)
  - [Examples](examples/)
  - [Contributing Guide](docs/CONTRIBUTING.md)
  - [Docker Guide](docs/docker.md)
- **Current docs (active project):**
  - See [`soplang/soplang`](https://github.com/soplang/soplang)

## Installation (legacy, not maintained)

The instructions in this section apply to the **Python interpreter version** only and are kept for reference. They may no longer work on current platforms as-is.

For detailed legacy installation instructions, see the [Installation Guide](docs/installation.md).

### Download Installers

Previously, the easiest way to get started was to download a pre-built installer from the old releases page:

- Legacy releases: <https://github.com/sharafdin/soplang/releases>

For current releases, see the releases page of [`soplang/soplang`](https://github.com/soplang/soplang/releases).

### Using Docker (No Installation Required)

You can run the legacy Soplang interpreter using Docker without installing anything:

```bash
# Run the interactive shell
docker run -it --rm soplang/soplang

# Run a Soplang script
docker run -it --rm -v $(pwd):/scripts soplang/soplang my_script.sop
```

See the [Docker Guide](docs/docker.md) for more details.

### Building from Source

If you want to build the **legacy interpreter** from source, refer to the platform-specific build guides:

- **Windows**: See the [Windows Build Guide](windows/WINDOWS_BUILD_GUIDE.md)
- **Linux**: See the [Linux Build Guide](linux/README.md)
- **macOS**: See the [macOS Build Guide](macos/README.md)

You can also use our universal build script that automatically detects your platform:

```bash
# Clone the repository
git clone https://github.com/sharafdin/soplang.git
cd soplang

# Run the universal build script
./build.sh  # (may need chmod +x build.sh on Unix systems)
```

## Development

For this legacy interpreter, we provide a Makefile to simplify common development tasks:

```bash
# Setup your development environment
make install

# Run the interactive shell
make shell

# Run tests
make test

# Format code
make format

# See all commands
make help
```

## Releases

Historically, this project used an automated release process powered by GitHub Actions. For details, see the legacy [Release Process Documentation](docs/RELEASE_PROCESS.md).

For current releases and the active pipeline, see [`soplang/soplang`](https://github.com/soplang/soplang).

## Contributing

Contributions are welcome! See our [Contributing Guide](docs/CONTRIBUTING.md) for details on how to get started.

## Code of Conduct

We are committed to providing a welcoming and inclusive experience for everyone. Please read our [Code of Conduct](CODE_OF_CONDUCT.md) to understand the community standards we uphold.

## License

This project is licensed under the [MIT License](https://opensource.org/licenses/MIT) - see the [LICENSE](LICENSE) file for details.

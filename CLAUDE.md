# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The BSV SDK is a comprehensive Python library for developing scalable applications on the BSV Blockchain. It provides a peer-to-peer approach adhering to SPV (Simplified Payment Verification) with focus on privacy and scalability.

**Repository**: https://github.com/bitcoin-sv/py-sdk
**Package name**: bsv-sdk
**Current version**: 2.1.3
**Python requirement**: >=3.10

## Development Commands

### Installation

```bash
pip install -r requirements.txt
```

### Testing

```bash
# Run full test suite with coverage
pytest --cov=bsv --cov-report=html

# Run specific test file
pytest tests/test_transaction.py

# Run tests with asyncio
pytest tests/bsv/auth/test_auth_peer_basic.py
```

### Linting & Formatting (Required before commit/PR)

All code must pass Ruff and Black before committing. Run these commands and fix any errors before creating a commit or pull request:

```bash
# Lint with Ruff (fix auto-fixable issues)
ruff check --fix .

# Format with Black
black .

# Verify no remaining issues
ruff check .
```

Configuration is in `pyproject.toml` under `[tool.ruff]` and `[tool.black]`. Install dev dependencies with:

```bash
pip install -e ".[dev]"
```

### Building the Package

```bash
# Build distribution packages (requires python3 -m build)
make build

# Or directly:
python3 -m build
```

### Publishing (Maintainers Only)

```bash
make upload_test  # Upload to TestPyPI
make upload       # Upload to PyPI
```

Se README for arkitektur og kodeeksempler.

## Testing Structure

Tests are organized in two locations:

1. **Root-level tests** (`tests/`): Classic test structure with direct imports
2. **Nested tests** (`tests/bsv/`): Mirror the `bsv/` package structure

Test organization by feature:

- `tests/bsv/primitives/`: Core cryptographic primitives
- `tests/bsv/transaction/`: Transaction building and validation
- `tests/bsv/auth/`: Full authentication protocol test suite
- `tests/bsv/wallet/`: Wallet implementation tests
- `tests/bsv/storage/`: Storage system tests
- `tests/bsv/broadcasters/`: Broadcaster integration tests

**Running single test**: Use standard pytest patterns:

```bash
pytest tests/bsv/auth/test_auth_peer_basic.py::test_function_name
pytest -k "test_pattern"
```

## Code Style

- **PEP 8 compliance**: Follow Python standard style guide
- **Type hints**: Use where appropriate (not comprehensive in current codebase)
- **Docstrings**: Document functions, classes, and modules
- **Comments**: Annotate complex logic

## Development Practices

- **Test-Driven Development**: Write tests before or alongside implementation where smart, quick, and reasonable. This helps ensure correctness and prevents regressions.
- Run `pytest --cov=bsv --cov-report=html` to verify test coverage before committing
- All PRs should maintain or improve current test coverage

## BRC-106 Compliance (Script ASM Format)

The SDK implements Assembly (ASM) representation of Bitcoin Script via `Script.from_asm()` and `Script.to_asm()` methods.

**BRC-106 Standard**: https://github.com/bitcoin-sv/BRCs/blob/master/scripts/0106.md

Key requirements from BRC-106:

- Use full English names for op-codes (e.g., "OP_FALSE" not "OP_0")
- Output should always use the most human-readable format
- Multiple input names should parse to the same hex value
- Ensure deterministic translation across different SDKs (Py-SDK, TS-SDK, Go-SDK)

**Current Implementation** (bsv/script/script.py:140-191):

- `from_asm()`: Accepts both "OP_FALSE" and "OP_0", converts to b'\x00'
- `to_asm()`: Currently outputs "OP_0" for b'\x00' (see OPCODE_VALUE_NAME_DICT override at constants.py:343)

**Note**: The current `to_asm()` output may need adjustment to fully comply with BRC-106's human-readability requirement (should output "OP_FALSE" instead of "OP_0").

### Working with ASM

```python
# Parse ASM string to Script
script = Script.from_asm("OP_DUP OP_HASH160 abcd1234 OP_EQUALVERIFY OP_CHECKSIG")

# Convert Script to ASM representation
asm_string = script.to_asm()

# Access script chunks
for chunk in script.chunks:
    print(chunk)  # Prints opcode name or hex data
```

## Chronicle Update Support

The SDK implements the BSV Chronicle network upgrade (MainNet block 943,816, target 07-Apr-2026).

### What Changed

- **10 restored opcodes**: OP_VER, OP_VERIF, OP_VERNOTIF, OP_2MUL, OP_2DIV, OP_SUBSTR, OP_LEFT, OP_RIGHT, OP_LSHIFTNUM, OP_RSHIFTNUM. No opcodes are disabled after Chronicle.
- **SIGHASH_CHRONICLE (0x20)**: New sighash bit enabling the Original Transaction Digest Algorithm (OTDA). Routing: FORKID only → BIP143; FORKID+CHRONICLE or no FORKID → OTDA.
- **Malleability relaxation**: Transactions with version > 1 relax 7 restrictions (minimal encoding, low-S, NULLFAIL, NULLDUMMY, MINIMALIF, clean stack, push-only unlocking scripts). Controlled by `Spend.is_relaxed()`.
- **Script number size limit**: Increased from 750KB to 32MB (`AfterGenesisConfig.max_script_number_length()`).
- **Opcode constants**: OP_NOP4–NOP8 are now aliases for the restored opcodes. The canonical enum names are OP_SUBSTR, OP_LEFT, etc.

### Key Files

- `bsv/constants.py`: SIGHASH.CHRONICLE, new OpCode entries, TRANSACTION_VERSION_CHRONICLE
- `bsv/script/spend.py`: `is_relaxed()`, all opcode handlers, malleability gates
- `bsv/transaction.py`: `calc_input_signature_hash()` OTDA routing
- `bsv/transaction_preimage.py`: `_preimage_otda()`, OTDA preimage generation
- `bsv/script/interpreter/config.py`: 32MB script number limit

### Chronicle Tests

```bash
pytest tests/bsv/script/test_chronicle_*.py tests/bsv/script/interpreter/test_chronicle_*.py -v
```

## Important Notes

- The SDK uses `coincurve` for ECDSA operations (not pure Python)
- Encryption uses `pycryptodomex` (not standard `pycryptodome`)
- Network operations require `aiohttp` for async HTTP
- Tests require `pytest-asyncio` for async test support
- Coverage configuration excludes tests and setup.py (see `.coveragerc`)
- Git branches: `master` is main branch, `develop-port` is development branch

Se README for arkitektur og kodeeksempler.

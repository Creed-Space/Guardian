# Creed Guardian

**Local AI safety for any device. Free forever.**

[![PyPI version](https://badge.fury.io/py/creed-guardian.svg)](https://badge.fury.io/py/creed-guardian)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)

Creed Guardian is a lightweight AI safety layer that evaluates agent actions locally before execution. It uses Ollama-powered models to catch unsafe actions without cloud dependencies, API costs, or data egress.

## Quick Start

> **Release pending on PyPI.** PyPI currently serves only the unsafe 0.1.1 preview. The pinned
> 2.1.0 command below will fail until the repaired candidate is published and verified. Do not
> install an unpinned `creed-guardian`.

```bash
# Install Guardian (pinned; works once 2.1.0 is published on PyPI)
python -m pip install creed-guardian==2.1.0

# Install Ollama (required)
# macOS: brew install ollama
# Linux: curl -fsSL https://ollama.com/install.sh | sh

# Start Ollama
ollama serve
```

```python
from creed_guardian import Guardian

guardian = Guardian()  # 0.1.1 picks a tier by available RAM; 2.1.0 uses the 3b tier

result = guardian.check_sync(
    action="Delete all files in /home",
    context="User said: clean up my downloads"
)

if result.allowed:
    execute_action()
else:
    print(f"Blocked: {result.reason}")
```

## Features

- **Offline-first**: Works without internet
- **Privacy-preserving**: Data never leaves your device
- **Zero marginal cost**: No API fees for local evaluation
- **Fail-safe**: Blocks on uncertainty (configurable)
- **Tier selection**: 0.1.1 picks a tier by available RAM; 2.1.0 uses `3b` unless you pass `tier="auto"`

## Model Tiers

The source in this repository is version 0.1.1, the same version as the unsafe PyPI preview.
The 2.1.0 candidate described on [creed.space/guardian](https://creed.space/guardian#tiers), not
yet on PyPI, keeps the same tier IDs but runs different models on the three smaller tiers. Tier IDs are
stable; the model behind a tier changes only in a versioned release.

| Tier | 0.1.1 model (this repository) | 0.1.1 RAM | 2.1.0 model | 2.1.0 RAM |
|------|-------------------------------|-----------|-------------|-----------|
| **1.5b** | qwen2.5:1.5b | 2GB | qwen3.5:2b | 2GB |
| **3b** | qwen2.5:3b | 4GB | qwen3.5:4b | 4GB |
| **7b** | qwen2.5:7b | 8GB | qwen3.5:9b | 8GB |
| **14b** | qwen2.5:14b | 12GB | qwen2.5:14b | 12GB |
| **32b** | qwen2.5:32b | 20GB | qwen2.5:32b | 24GB |

In 0.1.1, `Guardian()` picks the largest tier that fits the available RAM. In 2.1.0,
`Guardian()` uses the `3b` tier unless you pass `tier="auto"` or name a tier.

This README gives no accuracy figures. The figures it used to show were not measured on a
Guardian release: they came from benchmarks of the bare Ollama models, and no single run
produced them. Current evaluation results are on [creed.space/safety](https://creed.space/safety).

## Usage

### Basic Check

```python
from creed_guardian import Guardian

guardian = Guardian()

# Check an action
result = guardian.check_sync(
    action="Send email to user@example.com",
    context="User requested password reset"
)

print(result.verdict)    # PASS | FAIL | UNCERTAIN
print(result.reason)     # Human-readable explanation
print(result.latency_ms) # Evaluation time
```

### Synchronous API

```python
# For non-async code
result = guardian.check_sync(action="test action")
```

### Decorator Protection

```python
@guardian.protect
async def delete_files(path: str):
    """This function is protected by Guardian."""
    os.rmdir(path)

# Raises PermissionError if Guardian blocks
await delete_files("/tmp/data")
```

### Custom Principles

```python
result = guardian.check_sync(
    action="Post message to social media",
    context="Marketing campaign",
    principle="Never post without explicit user approval"
)
```

### Explicit Tier Selection

```python
# Force a specific tier
guardian = Guardian(tier="7b")  # qwen2.5:7b in 0.1.1, qwen3.5:9b in 2.1.0

# Available: "auto", "1.5b", "3b", "7b", "14b", "32b"
```

### Async Context Manager

```python
async with Guardian() as guardian:
    result = await guardian.check(action="test")
    # Automatically closes when done
```

## Configuration

```python
Guardian(
    tier="auto",              # 0.1.1 default (auto-selects by RAM); 2.1.0 defaults to "3b"
    ollama_url="http://localhost:11434",  # Ollama server URL
    fail_closed=True,         # Block uncertain cases (default: True)
    auto_download=True,       # Download model if not available
    evaluation_timeout=30.0,  # Timeout in seconds
)
```

## Response Object

```python
@dataclass
class GuardianResult:
    verdict: Literal["PASS", "FAIL", "UNCERTAIN"]
    allowed: bool       # True if PASS
    blocked: bool       # True if FAIL
    uncertain: bool     # True if UNCERTAIN
    reason: str         # Human-readable explanation
    confidence: float   # 0.0-1.0
    source: str         # "pattern" | "local" | "cloud"
    tier: str           # Model tier used
    latency_ms: float   # Evaluation time
```

## Requirements

- Python 3.10+
- [Ollama](https://ollama.com/) running locally
- 2GB+ RAM (4GB+ recommended)

## Need More?

**Creed Guardian is free forever** for local evaluation.

For cloud features (escalation, fleet management, custom constitutions), check out [Creed Space](https://creed.space/guardian):

| Feature | Guardian (Free) | Creed Space |
|---------|-----------------|-------------|
| Local evaluation | Unlimited | Unlimited |
| All model tiers | Yes | Yes |
| Cloud escalation | - | Yes |
| Custom constitutions | - | Yes |
| Fleet dashboard | - | Yes |
| Analytics | - | Yes |

## Links

- [Documentation](https://creed.space/docs/guardian)
- [API Reference](https://creed.space/docs/guardian/api)
- [Creed Space](https://creed.space)
- [GitHub](https://github.com/Creed-Space/Guardian)

## Security Considerations

### Prompt Injection

Guardian uses LLM-based evaluation, which is inherently susceptible to prompt injection attacks. An adversary could craft inputs designed to manipulate the model's decision. Mitigations:

1. **Fail-closed default**: Uncertain responses are blocked
2. **Input sanitization**: Suspicious patterns are flagged
3. **Defense in depth**: Guardian is one layer, not the only layer

**Recommendation**: Use Guardian alongside other security measures, not as a sole defense.

### Network Security

- By default, Guardian only connects to `localhost:11434` (Ollama)
- Cloud metadata endpoints and private IP ranges are blocked
- If you need remote Ollama, validate URLs in your application layer

### Reporting Security Issues

Report security vulnerabilities to security@creed.space. Do not open public issues for security bugs.

## License

Apache License 2.0 - see [LICENSE](LICENSE) for details.

---

*Built by [Creed Space](https://creed.space) - AI safety infrastructure for the real world.*

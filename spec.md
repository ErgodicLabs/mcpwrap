# `mcpwrap` specification v0.1

## Overview

`mcpwrap` is a **build tool** for generating [Model Context Protocol (MCP)](https://modelcontextprotocol.io) servers from Python libraries. It exposes public functions as MCP-compatible commands using the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk), targeting the `FastMCP` runtime.

It supports:

- **Runtime introspection** for development
- **Code generation** for production


## Design Philosophy

`mcpwrap` is guided by four principles:

### 1. Runtime introspection during development, code generation for production

Runtime mode supports fast iteration with zero setup. Code generation produces auditable, standalone MCP servers for deployment. The generated server code has **no dependency on `mcpwrap`**.

### 2. Simple is better than smart

Only existing, explicit function metadata is used. No reflection, inference, or guessing.

### 3. No AI

Descriptions and schemas are extracted statically from source. No language modeling or natural language inference is involved.

### 4. A tool, not a framework

`mcpwrap` emits plain Python source. It doesn’t require subclassing, decorators, or runtime integration beyond what the MCP SDK already expects.


## Usage Modes

`mcpwrap` supports two workflows:

### 1. Runtime Introspection (Development)

To serve the `random` module in a live FastMCP server:

```python
import random

mcpwrap(random).serve()
```

- Inspects the `random` module at runtime
- Registers all public top-level functions with a `FastMCP` server
- Starts the JSON-RPC event loop in-process
- Intended for development, prototyping, or LLM integration experiments

### 2. Code Generation (Production)

To generate a standalone FastMCP server and print it to stdout:

```python
import random
import sys

mcpwrap(random).dump(sys.stdout, mcp_sdk_version="1.6.0")
```

Or to emit a file:

```python
mcpwrap(random).dump("random_server.py", mcp_sdk_version="1.6.0")
```

- Produces a self-contained Python file
- Targets the specified version of the MCP Python SDK (e.g. `1.6.0`)
- Suitable for version control, deployment, or CI/CD pipelines


## API

```python
def mcpwrap(lib: Any) -> MCPWrapper
```

Returns an instance of:

```python
class MCPWrapper:
    def serve(self) -> None:
        """Starts a live FastMCP server using runtime introspection."""
        ...

    def dump(self, outfile: TextIO | str | Path, mcp_sdk_version: str) -> None:
        """Generates standalone FastMCP server code and writes it to the output target."""
        ...
```


## Code Generation Output

### Header

```python
# AUTO-GENERATED BY mcpwrap — DO NOT EDIT
# mcpwrap version: 0.1.0
# mcp sdk version: 1.6.0
# library: random
# library version: built-in (no __version__)
# python version: 3.11.8
# generated: 2025-04-16T16:45:00Z
```

### Server Body

```python
from mcp.server.fastmcp import FastMCP
import random

mcp = FastMCP("random")

@mcp.tool()
def randint(a: int, b: int) -> int:
    """Return a random integer N such that a <= N <= b."""
    return random.randint(a, b)

if __name__ == "__main__":
    mcp.serve()
```


## Supported MCP SDK Versions

```python
SUPPORTED_MCP_SDK_VERSIONS = [
    "1.6.0"
]
DEFAULT_MCP_SDK_VERSION = "1.6.0"
```

- `dump()` requires an explicit `mcp_sdk_version`
- Output structure and decorators are version-locked to that SDK version


## Function Discovery

- Uses `inspect.getmembers(lib, predicate=inspect.isfunction)`
- Filters out:
  - Private functions (`_` prefix)
  - Non-callables
- Only top-level functions are included (v0.1)


## Command Construction

For each function:

- **Name**: `func.__name__`
- **Description**:
  - If present, uses `inspect.getdoc(func)`
  - If missing, falls back to `f"{func.__name__}{inspect.signature(func)}"`
- **Parameters**:
  - Extracted from `inspect.signature(func)`
  - Marked `required` if no default value is provided
  - Type inferred from annotations (or `"string"` if missing)

Example:

```json
{
  "type": "object",
  "properties": {
    "a": { "type": "integer" },
    "b": { "type": "integer" }
  },
  "required": ["a", "b"]
}
```


## Call Semantics (in runtime mode)

- Parameters passed as `**kwargs`
- Calls are directly forwarded: `.call("randint", {"a": 1, "b": 10})` → `random.randint(1, 10)`
- No runtime type coercion or validation is applied


## Out of Scope

- Class method or submodule traversal
- Return type schema inference
- Input validation or coercion
- Async or streaming function support
- Function allowlisting or filtering
- Decorator or schema overrides

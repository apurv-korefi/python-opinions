# Python Syntax & Idioms Guide (2025)

_Terse, copy-pastable, and biased toward readable, type-safe, production-ready code._

## Hard Rules (No Bikeshedding)

- **Use types everywhere.** Modern syntax only: `list[str]`, `dict[str, int]`, `int | None`, `Path`, `Literal`, `TypedDict`, `Protocol`.
- **Prefer keyword-only APIs.** Add `*` so callers must name args.
- **Short, pure functions > classes.** Reach for classes only when you need state or a protocol boundary.
- **Guard clauses > nesting.** Return early; keep the happy path left-aligned.
- **Comprehensions OK when shallow.** If it nests twice, split it.
- **f-strings for messages, structured logs for prod.**
- **Pathlib, not os.path.** Zone-aware datetimes, not naive.
- **Dataclasses for value objects; Pydantic for validation.**
- **Pattern matching only for real sum types, not basic if/elif.**
- **Use the walrus sparingly** — only when it removes duplication.

---

## Functions & Typing

```python
from pathlib import Path
from typing import Iterable, Protocol, Literal, TypedDict, Final, overload

# Keyword-only API
def save_report(*, out: Path, data: list[dict[str, str]]) -> None:
    out.write_text("\n".join(row["line"] for row in data), encoding="utf-8")

# Overloads for ergonomic APIs
@overload
def parse_id(s: str, *, strict: Literal[True]) -> int: ...

@overload
def parse_id(s: str, *, strict: Literal[False] = False) -> int | None: ...

def parse_id(s: str, *, strict: bool = False) -> int | None:
    try:
        return int(s)
    except ValueError:
        if strict: raise
        return None

# Protocols for duck typing
class SupportsWrite(Protocol):
    def write(self, b: bytes, /) -> int: ...

# Constants and sentinels
MIB: Final[int] = 1 << 20
_sentinel = object()
```

### Generics (3.12+ PEP 695):

```python
from collections.abc import Sequence

def first[T](xs: Sequence[T]) -> T:
    if not xs: raise ValueError("empty")
    return xs[0]
```

---

## Data Objects

```python
from dataclasses import dataclass, field
from enum import StrEnum

class Color(StrEnum):
    RED = "red"
    BLUE = "blue"

@dataclass(slots=True, frozen=True, kw_only=True)
class User:
    id: int
    name: str
    tags: tuple[str, ...] = field(default_factory=tuple)
```

**Rule:** Immutable by default (`frozen`). Use tuples, not lists, for stable fields.

When you need validation/serialization (API edges, config), use Pydantic models; keep your internal domain as dataclasses.

---

## Control Flow & Expressions

```python
# Guard clauses > nesting
def pay(amount: int) -> None:
    if amount <= 0: raise ValueError("amount must be positive")
    if amount > 10_000: raise ValueError("needs approval")
    # happy path...

# Comprehensions, but shallow
evens = [x for x in xs if x % 2 == 0]               # good
pairs = [(a, b) for a in A for b in B if ok(a, b)] # 2 levels is the limit

# Walrus only when it removes duplication
if (m := pattern.search(s)) is not None:
    return m.group(1)

# Pattern matching for real tagged unions
match msg:
    case {"type": "ok", "value": v}:
        handle_ok(v)
    case {"type": "error", "reason": r}:
        handle_err(r)
    case _:
        raise ValueError("unknown message")
```

---

## Collections & Iteration

```python
# Prefer any/all/sum/max to manual flags
if any(u.id == target for u in users): ...

# Dict lookups over if/elif ladders
handlers: dict[str, Callable[[Payload], None]] = {
    "create": handle_create,
    "delete": handle_delete,
}
handlers.get(cmd, handle_unknown)(payload)

# Use enumerate, not range(len(...))
for i, item in enumerate(items): ...
```

---

## Errors

```python
class AppError(Exception): ...
class ConfigError(AppError): ...

try:
    cfg = load_config(path)
except FileNotFoundError as e:
    raise ConfigError(f"missing config: {path}") from e
```

**Rules:**

- Narrow except blocks. No bare `except:`.
- Raise project errors at boundaries; keep stdlib errors internal.
- Prefer `if x is None` / `if x is True` for singletons.

---

## Files, Paths, and Time

```python
from pathlib import Path
from datetime import UTC, datetime
from zoneinfo import ZoneInfo

p = Path("data") / "report.txt"
p.write_text("hello\n", encoding="utf-8")
text = p.read_text(encoding="utf-8")

now = datetime.now(UTC)  # always aware
india = ZoneInfo("Asia/Kolkata")
local = now.astimezone(india)
```

---

## Async & Concurrency (Keep It Tidy)

```python
# Only go async for I/O heavy services or when the framework is async.
# Don't mix threads and asyncio unless you must—wrap with run_in_executor explicitly.
import asyncio

async def fetch_all(clients: list[Client]) -> list[Item]:
    return await asyncio.gather(*(c.fetch() for c in clients))
```

---

## Logging

```python
import structlog

log = structlog.get_logger()

def create_item(i: int, name: str) -> None:
    log.info("create_item", item_id=i, name=name)  # fields, not printf
```

**Rule:** Use fields; keep f-strings for human messages (CLI, exceptions).

---

## Modules & Layout

```python
# mypkg/__init__.py
from .api import create_item  # re-export your public API

__all__ = ["create_item"]     # make exports explicit
```

Avoid star-imports in packages. Keep module files < ~300 lines when possible.

---

## Small but Sharp Opinions

- Use `*` to force keyword-only params; use positional-only `/`, only if mirroring C/stdlib.
- Prefer `match` for shape/type matching; prefer dicts for constant→action tables.
- Numeric literals: group with underscores `10_000_000`.
- Avoid mutable defaults (ever). Use `None` + sentinel or a `default_factory`.
- Use `Protocol` over concrete base classes for plug-in points.
- Use `Final` for constants and `@final` for "do not subclass".
- Don't hide work in `__init__`; keep constructors cheap.
- Don't return boolean status; raise or return a value.
- Avoid clever one-liners; 3 clear lines beat one cryptic listcomp.
- Prefer `with` + context managers (`contextlib.suppress`, `ExitStack`) for resources.

---

## Minimal "Golden" Module Template

```python
"""User operations."""
from __future__ import annotations  # keep if you target 3.10/3.11 too

from dataclasses import dataclass, field
from typing import Final, Protocol
from pathlib import Path

APP_NAME: Final[str] = "myapp"

class Store(Protocol):
    def save(self, *, key: str, data: bytes) -> None: ...

@dataclass(slots=True, frozen=True, kw_only=True)
class User:
    id: int
    name: str
    tags: tuple[str, ...] = field(default_factory=tuple)

def hello(*, user: User, store: Store, out_dir: Path) -> Path:
    """Create a greeting file for the user."""
    out = out_dir / f"{user.id}.txt"
    out.write_text(f"hi {user.name}\n", encoding="utf-8")
    store.save(key=str(user.id), data=out.read_bytes())
    return out
```

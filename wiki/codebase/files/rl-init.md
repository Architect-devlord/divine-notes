---
type: file
status: ingested
---
# **init**.py (`rl/__init__.py`)

💡 **Role**: package re-export, 8 lines total. `from .env import DivineWorldEnv`, `from .policy import TransformerPolicy`.

**Worth knowing**: `GodTransformerPolicy` (see [[wiki/codebase/files/policy|policy.py]]) is _not_ re-exported here, despite `TransformerPolicy` being right next to it in the same source file — `from rl import GodTransformerPolicy` would fail; a caller needs `from rl.policy import GodTransformerPolicy` directly. Not confirmed as a bug (every actual call site for `GodTransformerPolicy` traced so far already imports it the fully-qualified way), but it's an asymmetry worth knowing about if `rl`'s public surface is ever revisited.

## Files Required

- [[wiki/codebase/files/env|env.py]], [[wiki/codebase/files/policy|policy.py]] — both in the same `rl/` package.

## Files Used In

Anything doing `from rl import DivineWorldEnv, TransformerPolicy` rather than reaching into the submodules directly — not traced to a specific caller this pass.
# Fork notes

This fork carries two fixes on top of upstream
[`GroovyLanguageServer/groovy-language-server`](https://github.com/GroovyLanguageServer/groovy-language-server)
(base commit `347d098a`, 2026-05-19). Both bugs were reproduced against a large SAP
Hybris Commerce workspace (~3,250 Groovy sources); both make the server effectively
unusable in that setting. The fixes are client-agnostic and API-neutral.

## Why this fork exists

### 1. Empty symbols for any file opened after the initial workspace compile

**Symptom.** Start the server, let it finish its initial workspace compilation, open a
source file — every `textDocument/documentSymbol` and `workspace/symbol` query for it
returns an empty tree, even though the same queries work in a freshly started server
before any workspace compile has run.

**Root cause.** Clients that push settings via `workspace/didChangeConfiguration`
immediately after `initialize` trigger a full workspace compile. Groovy's
`CompilationUnit` phase counter is monotonic across compilation units fed into the
same compiler instance, so sources that are added (opened) after that first compile
never get parsed — the AST visitor simply never sees them.

**Fix.** `GroovyServices`:
- serve files already present in the compiled index from that index
  (`initialCompileDone`, `isWorkspaceSourceUnchanged`, `unitHasSource`);
- fall back to a per-file mini compilation for files outside the index or changed on
  disk (`compileSingleFileAndVisit`: a one-file `GroovyLSCompilationUnit` with a
  `StringReaderSourceWithURI` keyed under the request URI);
- `recompileIfContextChanged` extended so the fallback stays correct when the
  surrounding source moves.

### 2. References query hangs the server for hours on large workspaces

**Symptom.** One `textDocument/references` request on a multi-thousand-file
workspace stops the entire server responding — not just that request — for hours
(observed: resolution of ~1M candidate nodes on the JSON-RPC reader thread).

**Root cause.** `GroovyASTUtils.getReferences` resolved the *definition* of every
candidate node in the workspace before checking whether it matches the requested
symbol — an O(N²) amount of resolution work done synchronously on the reader thread.

**Fix.** `GroovyASTUtils`:
- extract the referenced name (`definitionName`: `FieldNode` / `MethodNode` /
  `PropertyNode` / `Variable` / `ClassNode`);
- skip candidates that cannot possibly carry that name
  (`candidateHasName`: `ConstantExpression` / `VariableExpression` /
  `PropertyExpression`) before any definition resolution;
- null-guard `ImportNode` / `ClassExpression` `getType()` so wildcard imports no
  longer NPE during the candidate scan.

## Why it must be adopted

- **Bug 1 hits any client that pushes configuration after `initialize`** — which is
  standard behavior for several editors/agents ( omp does it; any
  `didChangeConfiguration`-before-first-open client does too). The server then
  reports empty symbols for everything opened later, which looks like "the language
  server is broken", not like a client-interaction bug.
- **Bug 2 hits any large workspace**, independent of the client. A single references
  request takes the whole server down with it.
- Both fixes are local to the two files above, add no API surface, and change no
  behavior on the happy path (small workspaces, files opened before the first
  compile).

## Building

```sh
export JAVA_HOME=<JDK 17>
./gradlew shadowJar -x test
./gradlew --stop   # don't leave a daemon around
```

The fat jar lands at `build/libs/groovy-language-server-all.jar` (~12.8 MB).

## Deployment for omp + SAP Hybris

Project-level config at `<repo>/.omp/lsp.json` (omp's documented project override
location):

```jsonc
{
  "servers": {
    "groovy-language-server": {
      "command": "<JDK 17>/bin/java",
      "args": ["-Xmx6g", "-jar", "<fork>/build/libs/groovy-language-server-all.jar"],
      "fileTypes": [".groovy"],
      "languageId": "groovy",
      "rootMarkers": [".git"],
      "warmupTimeoutMs": 300000,
      "settings": { "groovy": { "classpath": ["<dir>", "..."] } }
    }
  },
  "idleTimeoutMs": 300000
}
```

- `-Xmx6g` is load-bearing for Hybris-scale workspaces: 2g OOMs during indexing, 4g
  GC-thrashes (~3.9 GB RSS, unresponsive). Budget ~4 GB resident while warm.
- `settings.groovy.classpath` = sorted, deduplicated list of `lib`/`bin` dirs:
  `find hybris/bin/platform hybris/bin/modules hybris/bin/custom -type d \( -name
  lib -o -name bin \)` (2,527 entries for the reference workspace).
- `idleTimeoutMs` (top level) shuts the JVM down after idle, reclaiming the ~4 GB at
  the cost of re-paying the 60–90 s cold index on the next query.
- Keep `<repo>/.omp/` gitignored.

### Verified capability matrix (reference workspace, 2026-09)

| Capability | Status |
| --- | --- |
| Document symbols | works, incl. files opened post-compile (fix 1) |
| Workspace symbol search | works (~91% of files via batch compile; rest served per-file) |
| Definition / references / hover / rename | unreliable — batch compile stalls during resolution (~2,970/3,254 files); use grep for cross-references, never rename via LSP |
| Custom Java classes in `<ext>/classes` dirs | unresolvable by design (not jars); ignore those diagnostics |

### Warning: server choice

Do **not** substitute a Groovy language server that ships a Gradle/Maven *project
importer* (e.g. TomaszRup/groovy-language-server) for a SAP Hybris checkout: the
importer runs `./gradlew` in the repo, which regenerates `hybris/config` and wipes
`hybris/config/local.properties`. This fork uses a plain classpath list and never
invokes a build tool on the workspace.

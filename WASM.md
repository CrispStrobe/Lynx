# Lynx Chess Engine — WebAssembly Build

This fork adds browser WebAssembly support to the [Lynx chess engine](https://github.com/lynx-chess/Lynx) by Eduardo Caceres.

**Lynx** is a C# chess engine (~3350 ELO, CCRL) with classical hand-crafted evaluation (HCE). This fork compiles it to WebAssembly via .NET `wasm-tools` so it can run in any modern browser.

## Download

Pre-built WASM bundles are available on the [Releases](https://github.com/CrispStrobe/lynx-chess/releases) page.

```bash
# Download and extract
curl -L https://github.com/CrispStrobe/lynx-chess/releases/latest/download/lynx-wasm.tar.gz | tar xz
```

## Quick Start

```html
<script type="module">
  import { dotnet } from './lynx-wasm/_framework/dotnet.js';

  const { getAssemblyExports, getConfig } = await dotnet
    .withDiagnosticTracing(false)
    .create();

  const exports = await getAssemblyExports(getConfig().mainAssemblyName);
  const lynx = exports.LynxWasm.UciInterop;

  await lynx.Initialize();

  // UCI protocol
  const uci = await lynx.SendCommand('uci');       // → "id name Lynx ...\nuciok"
  await lynx.SendCommand('isready');                // → "readyok"
  await lynx.SendCommand('position startpos');
  const result = await lynx.SendSearchCommand('go depth 8');
  console.log(result);  // → "info depth 1 ... bestmove e2e4 ponder e7e6"
</script>
```

## API

The WASM module exposes these `[JSExport]` methods:

| Method | Returns | Description |
|--------|---------|-------------|
| `Initialize()` | `Task<string>` | Init engine. Returns `"ok"` on success. |
| `SendCommand(cmd)` | `Task<string>` | Send a UCI command (uci, isready, position, setoption, stop). Waits for response. |
| `SendSearchCommand(cmd)` | `Task<string>` | Send a `go` command. Waits for `bestmove`. Returns all output (info lines + bestmove). |
| `PollOutput()` | `string` | Poll any pending output lines. |
| `IsReady()` | `bool` | Check if engine is initialized. |

## WASM Configuration

The WASM build applies these defaults (set in `Initialize()`):

- **Threads:** 1 (single-threaded — WASM doesn't support SMP)
- **Hash:** 4 MB (default 256 MB overflows WASM heap)
- **MaxDepth:** 64 (default 128 — halved to reduce array sizes)
- **Online tablebases:** Disabled (SocketsHttpHandler not available in browser)
- **Logging:** Disabled (NLog file sinks unavailable in browser)
- **Warmup:** Skipped (avoid blocking main thread during init)

## Browser Patches

This fork applies these patches to make Lynx run in browser WASM:

1. **SocketsHttpHandler** (`OnlineTablebaseProber.cs`) — guarded with `#if !BROWSER_WASM`. `SocketsHttpHandler` is not supported on the browser platform and crashes the Mono runtime.

2. **Thread.CurrentThread.Priority** (`Engine.cs`) — guarded with `#if !BROWSER_WASM`. Setting thread priority causes an unrecoverable abort in `mini-wasm.c`.

3. **Warmup skip** (`Searcher.cs`) — guarded with `#if !BROWSER_WASM`. The warmup search triggers `Convert.ToUInt64(Math.Clamp(..., ulong.MaxValue))` which overflows because `(double)ulong.MaxValue` rounds up past `ulong.MaxValue`.

4. **Convert.ToUInt64 overflow** (`Utils.cs`) — `CalculateNps` and `CalculateUCITime` now clamp to `long.MaxValue` instead of `ulong.MaxValue` to avoid overflow.

All patches are guarded behind `#if BROWSER_WASM` so the native build is unaffected.

## Build from Source

### Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- `wasm-tools` workload: `dotnet workload install wasm-tools`

### Build

```bash
# Interpreter build (fast build, ~6 MB output, slower search)
dotnet publish src/Lynx.Wasm/Lynx.Wasm.csproj -c Release -p:RunAOTCompilation=false

# AOT build (slow build, ~21 MB output, 2-3x faster search)
dotnet publish src/Lynx.Wasm/Lynx.Wasm.csproj -c Release
```

Output: `src/Lynx.Wasm/bin/Release/net10.0/browser-wasm/AppBundle/_framework/`

### Test

```bash
node -e "
  import('./src/Lynx.Wasm/bin/Release/net10.0/browser-wasm/AppBundle/_framework/dotnet.js')
    .then(async ({dotnet}) => {
      const {getAssemblyExports, getConfig} = await dotnet.withDiagnosticTracing(false).create();
      const exports = await getAssemblyExports(getConfig().mainAssemblyName);
      const lynx = exports.LynxWasm.UciInterop;
      console.log(await lynx.Initialize());
      console.log(await lynx.SendCommand('uci'));
    });
"
```

## Bundle Size

| Build | Total | dotnet.native.wasm | Lynx.wasm | Gzipped |
|-------|-------|--------------------|-----------|---------|
| Interpreter | ~6 MB | 1.5 MB | 632 KB | ~2.5 MB |
| AOT | ~21 MB | 16 MB | 632 KB | ~6 MB |

## Integration

Used by [CrispChess](https://github.com/CrispStrobe/CrispChess) — a cross-platform chess app with pluggable engine backends.

## License

MIT — same as the original Lynx engine. See [LICENSE](LICENSE).

## Credits

- **Lynx chess engine** by [Eduardo Caceres](https://github.com/eduherminio) — [lynx-chess/Lynx](https://github.com/lynx-chess/Lynx)
- **.NET WASM runtime** by Microsoft — [dotnet/runtime](https://github.com/dotnet/runtime) (MIT)
- **WASM patches & build** by [CrispStrobe](https://github.com/CrispStrobe)

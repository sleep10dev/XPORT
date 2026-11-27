# Third-party notices

XPORT is licensed under **GPL-3.0-or-later** (see `LICENSE`).

This file tracks code, data, and specifications ported or derived from other
projects. It must be updated before importing any third-party material.

## Upstream projects (owner-authorized)

The repository owner declared ownership of both upstream projects and authorized
relicensing their authorial code into XPORT under GPL-3.0-or-later
(see `docs/DECISIONS.md`, 2026-11-24).

| Source | Original license | Use in XPORT | Notes |
|---|---|---|---|
| [Marvisak/VIctory](https://github.com/Marvisak/VIctory) | GPL-3.0-or-later | Partial port (in progress) | Preserve origin and attribution per imported file/module. |
| [ivall/xcl](https://github.com/ivall/xcl) | GPL-2.0-only (upstream) | Planned / partial port | Relicensed by owner for XPORT; preserve origin attribution. |

## Ported/adapted files (Lotes 94-95 - 2026-11-25/26)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/d3d12_null.rs` | `xcl-nvidia-dev/crates/xcl/src/hle/d3d12x/fence_x.rs` | Contrato autoral do Fence slot 12 (célula de valor estável) e publicação de sinal; implementado com `AtomicU64` próprio, sem estado/código do título. |
| `crates/xport-loader/src/crash.rs` | `xcl-nvidia-dev/crates/xcl/src/insn.rs` | Decodificação autoral de `monitorx`/`mwaitx` por bytes no RIP; emulação na VEH, sem patch do título. |
| `crates/xport-pe/src/image.rs`, `crates/xport-loader/src/crash.rs`, `crates/xport-loader/src/windows.rs` | `xcl-nvidia-dev/crates/xcl/src/eh.rs` | Registro de `.pdata` via `RtlAddFunctionTable` antes do código do título; adaptado a `ImageInfo`/`spawn_main`. |

Os shims do watchdog e do descritor de fonte nula são compatibilidade autoral
do XPORT, ativados automaticamente apenas para o hash verificado
`b99f876b74d4a77f...`: desviam exclusivamente para caminhos já existentes no
próprio título, são logados uma vez e não alteram bytes da imagem. Não são
diagnósticos e não dependem de variáveis de ambiente.

## API gráfica pública (Lote 101 / Fase 1 — 2026-11-26)

- `crates/xport-shader/src/lib.rs`, `src/dxc.rs`, `src/gpu/xbox.rs` e
  `tests/dxc_graphics.rs`: código autoral do XPORT. A API pública
  (`translate_xbox_graphics`, `compile_xbox_graphics`) expõe o recompilador
  interno `gpu::recomp` (VS/PS), sem importar código novo de terceiros.
- Os containers VS/PS usados nos testes são gerados pelo próprio XPORT
  (`gpu::xbox::synthetic`); nenhum shader, dump, stream, textura ou outro dado
  do título foi lido ou copiado para o repositório.
- O teste DXC é opt-in e usa apenas `dxil.dll`/`dxcompiler.dll` do Windows Kits
  instalado no sistema; nenhum binário é vendido ou commitado.

## Pipeline gráfico e captura (Lotes 103-104 / Fases 3-4 — 2026-11-26)

- `crates/xport-gpu/src/d3d12.rs`: assinatura root, PSO, RTV/DSV/SRV/sampler,
  upload, viewport/scissor, `DrawInstanced`, readback e `IASetVertexBuffers`
  são código autoral escrito contra a API pública do `windows` crate 0.62.2.
  O VS fullscreen com `SV_VertexID` e os containers sintéticos de textura são
  autorais. Nenhum shader, blob DXIL, textura, stream ou tabela binária do
  título foi copiado ao repositório.
- `crates/xport-hle/src/modules/d3d12_null.rs`: `XPORT_CAPTURE_CL=1` é um
  diagnóstico opt-in e desligado por padrão; lê, via `ReadProcessMemory`, no
  máximo duas submissões e imprime em stderr os ponteiros de `lists[]`, o token,
  128/256 bytes dos objetos e 128 bytes do array de planos de `queue_present`.
  Nada é escrito pelo XPORT: a saída vai para o
  redirecionamento de stderr do host, fora do repo. Nenhum patch de bytes do
  título foi aplicado.
- As execuções de reconhecimento da Fase 4 usaram apenas `RUST_LOG` e variáveis
  opt-in, com timeout e saídas em `%LOCALAPPDATA%\Temp\opencode`. Nenhum dump,
  asset, DLL, shader ou captura de jogo entrou no XPORT.

## Slots do layout mais novo (Lotes 105-106 / Fase 4 — 2026-11-26)

- `crates/xport-hle/src/modules/d3d12_list_sync.rs` e
  `d3d12_page_pool.rs`: lógica autoral do XPORT. Os números de slot e a ABI
  (`Write32BitValueBottomOfPipeX`, `Write64BitValueBottomOfPipeX`,
  `Wait64BitValueX`, `Reset`, `RegisterPagePoolX`) são fatos públicos do layout
  documentados no fork licenciado `xcl-nvidia-dev` (`d3d12_layout.rs`,
  `d3d12x/list_x.rs`, `d3d12x/device_x.rs`, `d3d12x/console_list.rs`); nenhum
  corpo de função, tabela ou dado do fork foi copiado.
- `crates/xport-hle/src/modules/d3d12_null.rs`: os bodies dos slots
  `#32/#52/#72/#74/#139/#146` e `#75` foram reescritos no XPORT. O despacho por
  `Kind` mantém vtables separadas para device e command list. Nenhum byte do
  título foi patchado.
- Escritas de sincronização são opt-in (`XPORT_LIST_APPLY_WRITES=1`), pois os
  VAs observados são offsets em memória unificada não mapeada pelo dispositivo
  offline; escrever neles como endereços host causava uma exceção no título.
- `XPORT_LOG_LIST_OPS`, `XPORT_LOG_PAGE_POOL`, `XPORT_DUMP_OBJ` e
  `XPORT_CAPTURE_CL` são diagnósticos opt-in, desligados por padrão, que apenas
  imprimem no stderr do host (fora do repo) via `ReadProcessMemory`.

## Ported files (Lote 1 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-core/src/lib.rs` | `VIctory/crates/victory-core/src/lib.rs` | Adapted names/env vars to XPORT. |
| `crates/xport-core/src/config.rs` | `VIctory/crates/victory-core/src/config.rs` | Adapted names/env vars/default dir. |
| `crates/xport-core/src/logging.rs` | `VIctory/crates/victory-core/src/logging.rs` | Adapted env var to `XPORT_LOG`. |
| `crates/xport-core/src/policy.rs` | `VIctory/crates/victory-core/src/policy.rs` | Adapted marker to `.xport-root`. |
| `crates/xport-pe/src/lib.rs` | `VIctory/crates/victory-pe/src/lib.rs` | Adapted crate names/docs. |
| `crates/xport-pe/src/image.rs` | `VIctory/crates/victory-pe/src/image.rs` | Adapted crate names/docs. |
| `crates/xport-pe/src/layout.rs` | `VIctory/crates/victory-pe/src/layout.rs` | Adapted crate names/docs. |
| `crates/xport-pe/src/implib.rs` | `VIctory/crates/victory-pe/src/implib.rs` | Adapted crate names/docs. |

## Ported/adapted files (Lote 2 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-loader/src/lib.rs` | `VIctory/crates/victory-loader/src/lib.rs` | Adapted to XPORT; Windows-only loading added in Lote 3. |
| `crates/xport-loader/src/prepare.rs` | `VIctory/crates/victory-loader/src/prepare.rs` | Adapted names/hashes; Windows-specific test omitted. |
| `crates/xport-loader/src/resolve.rs` | `VIctory/crates/victory-loader/src/resolve.rs` | Minimal adaptation. |
| `crates/xport-vfs/src/lib.rs` | Inspired by XCL mount/VFS concepts | **Clean-room implementation** (no source copied). |

## Ported/adapted files (Lote 91 - 2026-11-26)

O owner confirmou em 2026-11-26 que pode relicenciar o código autoral do fork
`xcl-nvidia-dev` (GPL-2.0-only) sob GPL-3.0-or-later para o XPORT.

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-vfs/src/lib.rs` | `xcl-nvidia-dev/crates/xcl/src/vfs.rs:220-258` | Regra de mount embutido (`...\GAME:\rest`) e precedência de prefixo portada e adaptada a `MountTable`; sem dados ou binários do fork. |
| `crates/xport-hle/src/modules/d3d12_null.rs` | `xcl-nvidia-dev/crates/xcl/src/hle/d3d12x/real.rs:1087-1141`, `device_x.rs:575,635,1171`, `null.rs:202-226` | Slots de view/sampler/PSO/placed resource e contrato de out-pointer; corpos sintéticos, sem estado de GPU. |
| `crates/xport-hle/src/modules/host_bridge.rs` | `xcl-nvidia-dev/crates/xcl/src/hle/dstorage.rs:624-638` | Status array começa em `S_OK` em vez de `E_PENDING`. |
| `crates/xport-hle/src/modules/xpackage.rs` | `xcl-nvidia-dev/crates/xcl/src/manifest.rs:17-57`, `hle/xgameruntime/mod.rs:290-311`, `gdk_table.rs:337` | Leitura runtime de `MicrosoftGame.Config` (UTF-8/UTF-16, comentários), identidade/locale derivados em memória e slot 27 com 3 argumentos. Nenhum conteúdo do manifest foi embutido no repositório. |

## Ported/adapted files (Lote 3 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-loader/src/windows.rs` | `VIctory/crates/victory-loader/src/windows.rs` | Adapted; thread-notice/crash integration deferred. |
| `crates/xport-loader/src/thunks.rs` | `VIctory/crates/victory-loader/src/thunks.rs` | Adapted; crash-log hook simplified. |
| `crates/xport-loader/src/tls.rs` | `VIctory/crates/victory-loader/src/tls.rs` | Adapted; test omitted for now. |

## Ported/adapted files (Lote 4 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/lib.rs` | `VIctory/crates/victory-hle/src/lib.rs` | Adapted resolver shape; minimal. |
| `crates/xport-hle/src/registry.rs` | `VIctory/crates/victory-hle/src/registry.rs` | Adapted. |
| `crates/xport-hle/src/host.rs` | `VIctory/crates/victory-hle/src/host.rs` | Adapted; host pass-through policy. |
| `crates/xport-hle/src/modules/mod.rs` | `VIctory/crates/victory-hle/src/modules/mod.rs` | Adapted structure. |
| `crates/xport-hle/src/modules/xgameruntime.rs` | Inspired by VIctory/XCL xgameruntime surface | Minimal original stubs (no source copied). |
| `crates/xport-report/src/lib.rs` | Inspired by XCL/VIctory reporting needs | Clean-room implementation. |
| `apps/xport-cli/src/main.rs` | Original | Adds `inspect` command emitting report/missing imports. |

## Ported/adapted files (Lote 5 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-loader/src/threads.rs` | `VIctory/crates/victory-loader/src/threads.rs` | Adapted; thread notices/exit detach. |
| `crates/xport-loader/src/crash.rs` | `VIctory/crates/victory-loader/src/crash.rs` | Adapted; reduced (no AMD instruction emulation/diagnostics save). |
| `crates/xport-loader/src/windows.rs` | `VIctory/crates/victory-loader/src/windows.rs` | Updated to wire threads/crash hooks. |
| `crates/xport-hle/src/runtime.rs` | `VIctory/crates/victory-hle/src/runtime.rs` | Adapted minimal image path/main image state. |

## Ported/adapted files (Lote 6 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xmem.rs` | Inspired by VIctory `victory-hle/src/modules/xmem.rs` and XCL memory behavior | Minimal original stubs/placeholders; full semantics deferred. |
| `crates/xport-hle/src/modules/kernelx.rs` | Inspired by XCL kernelx and VIctory kernel/module surface | Minimal original stubs/placeholders. |
| `crates/xport-hle/src/modules/mod.rs` | `VIctory/crates/victory-hle/src/modules/mod.rs` | Adapted to register new modules. |

## Ported/adapted files (Lote 7 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `apps/xport-cli/src/main.rs` | `VIctory/apps/victory/src/run.rs`, `VIctory/apps/victory/src/inspect.rs` | Adapted minimal Windows run path: prepare/load/bind/title DLLs/report; optional execute/spawn with crash logging. |

## Ported/adapted files (Lote 8 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` | Minimal adapted overrides: module handle/file name, load/getproc/free, debug output, exit/terminate. Many semantics deferred. |
| `crates/xport-hle/src/modules/kernelx.rs` | Inspired by XCL kernelx and VIctory module surfaces | Minimal stub behavior improved (TitleMemoryStatus zeroed struct, GetConsoleType placeholder). |
| `crates/xport-hle/src/modules/mod.rs` | `VIctory/crates/victory-hle/src/modules/mod.rs` | Registers kernel module. |
| `apps/xport-cli/src/main.rs` | Original XPORT adaptation | Run report notes include strict/execute flags. |

## Ported/adapted files (Lote 9 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` | Adapted HLE-aware module lookup: GetModuleHandle/LoadLibrary recognize HLE pseudo modules and runtime title images; GetProcAddress resolves HLE exports on pseudo handles; added tests. |

## Ported/adapted files (Lote 10 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xmem.rs` | Inspired by VIctory `victory-hle/src/modules/xmem.rs` and XCL memory behavior | Minimal host-backed subset (Windows VirtualAlloc/Free/Query, alloc tracking); console-specific semantics deferred. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Memory/SystemInformation` features for xmem. |

## Ported/adapted files (Lote 11 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernelx.rs` | Inspired by XCL kernelx | `TitleMemoryStatus` now uses `GlobalMemoryStatusEx` on Windows; minimal struct; added test. |

## Ported/adapted files (Lote 12 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` | Added host override lookup in `GetProcAddress` for host modules where XPORT has HLE exports; added `module_file_name` helper. |

## Ported/adapted files (Lote 13 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` | `LoadLibraryExA/W` now skip HLE/title lookup for data/no-resolve load flags (`runs_code`), forwarding to host. |

## Ported/adapted files (Lote 14 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT tests inspired by VIctory behavior | Added `runs_code_respects_data_flags` test. |

## Ported/adapted files (Lote 15 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Inspired by VIctory kernel module surface | Added minimal `GetModuleHandleExA/W` (name lookup via HLE/runtime; FROM_ADDRESS forwarded to host) and registered exports. |

## Ported/adapted files (Lote 16 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT tests | Added `get_module_handle_ex_finds_hle_module` test. |

## Ported/adapted files (Lote 17 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `apps/xport-cli/src/main.rs` | Original XPORT; inspired by VIctory/XCL run reporting | Added `--unimplemented trap|return-zero`, report now includes `build_hash`, `bound_dlls`, and `unimplemented` note. |
| `crates/xport-report/src/lib.rs` | Original XPORT clean-room report | Added `build_hash` and `bound_dlls` fields to `RunReport`. |

## Ported/adapted files (Lote 18 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-loader/tests/load_probe.rs` | `VIctory/crates/victory-loader/tests/load_probe.rs`, `title_dlls.rs` | Adapted Windows integration tests for XPORT names/APIs; reduced scope. |
| `tests/fixtures/xport-probe/*` | `VIctory/tests/fixtures/hle-probe/*` | Adapted stand-in title DLL fixture for loader bind/call tests. |

## Ported/adapted files (Lote 19 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` (inspired) | `GetModuleFileName*` returns synthetic path for HLE pseudo-modules; added test. |

## Ported/adapted files (Lote 20 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/runtime.rs` | `VIctory/crates/victory-hle/src/runtime.rs` (inspired) | Track image size and `image_containing` for FROM_ADDRESS lookups. |
| `crates/xport-hle/src/modules/kernel.rs` | `VIctory/crates/victory-hle/src/modules/kernel.rs` (inspired) | `GetModuleHandleExA/W` FROM_ADDRESS resolves title/runtime images before host. |
| `apps/xport-cli/src/main.rs` | Original XPORT wiring | Pass image size to HLE runtime registration. |

## Ported/adapted files (Lote 21 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetCommandLineA/W`, `GetStartupInfoA/W` minimal implementations and tests. |

## Ported/adapted files (Lote 22 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetSystemInfo`, `QueryPerformanceCounter/Frequency`, `GetProcessHeap/GetProcessHeaps`; host/non-Windows paths and test. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Performance` feature. |

## Ported/adapted files (Lote 23 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `HeapCreate/Destroy/Alloc/Free/ReAlloc/Size` host passthrough/fallback and Windows round-trip test. |

## Ported/adapted files (Lote 24 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `Initialize/Delete/Enter/Leave/TryEnterCriticalSection` host passthrough/fallback and Windows round-trip test. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Threading` feature. |

## Ported/adapted files (Lote 25 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `CreateEventA/W`, `SetEvent`, `ResetEvent`, `WaitForSingleObject`, `CloseHandle` host passthrough/fallback and Windows round-trip test. |

## Ported/adapted files (Lote 26 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetCurrentProcessId/ThreadId`, `GetCurrentProcess`, `Sleep` host passthrough/fallback and test. |

## Ported/adapted files (Lote 27 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetSystemTimeAsFileTime`, `GetTickCount64` host passthrough/fallback and test. |

## Ported/adapted files (Lote 28 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetSystemTime`, `GetLocalTime`, `GetTickCount` host passthrough/fallback and test coverage. |

## Ported/adapted files (Lote 29 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetLastError`, `SetLastError`, `GetEnvironmentVariableA/W` host passthrough/fallback and test. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Environment` feature. |

## Ported/adapted files (Lote 30 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `CreateFileA/W`, `ReadFile`, `WriteFile`, `GetFileSizeEx`, `GetFileType` host passthrough/fallback and Windows round-trip test. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_Storage_FileSystem`, `Win32_Security`, `Win32_System_IO` features. |

## Ported/adapted files (Lote 31 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `SetFilePointer`, `SetFilePointerEx`, `FlushFileBuffers`, `DeleteFileA/W` host passthrough/fallback and test coverage. |

## Ported/adapted files (Lote 32 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetFileAttributesA/W`, `CreateDirectoryA/W`, `RemoveDirectoryA/W` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 33 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetStdHandle`, `WriteConsoleA/W`, `GetConsoleMode`, `SetConsoleMode` host passthrough/fallback and basic test. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Console` feature. |

## Ported/adapted files (Lote 34 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `SleepEx`, `GetCurrentThread`, `SwitchToThread`, `GetExitCodeThread` host passthrough/fallback and test. |

## Ported/adapted files (Lote 35 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `WaitForSingleObjectEx`, `CreateMutexA/W`, `ReleaseMutex` host passthrough/fallback and Windows test. |

## Ported/adapted files (Lote 36 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `CreateSemaphoreA/W`, `ReleaseSemaphore`, `WaitForMultipleObjects` host passthrough/fallback and Windows test. |

## Ported/adapted files (Lote 37 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `VirtualProtect`, `FlushInstructionCache` host passthrough/fallback and Windows heap test coverage. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_System_Diagnostics_Debug` feature. |

## Ported/adapted files (Lote 38 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `VirtualAlloc`, `VirtualFree`, `VirtualQuery` host passthrough/fallback and Windows test. |

## Ported/adapted files (Lote 39 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `IsDebuggerPresent`, `DebugBreak` host passthrough/fallback and basic test. |

## Ported/adapted files (Lote 40 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `lstrlenA/W`, `lstrcmpiA/W`, `MultiByteToWideChar`, `WideCharToMultiByte` host passthrough/fallback and basic tests. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Added `Win32_Globalization` feature. |

## Ported/adapted files (Lote 41 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetACP`, `GetCPInfo`, `CompareStringA/W` host passthrough/fallback and basic test coverage. |

## Ported/adapted files (Lote 42 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-report/src/lib.rs` | Original XPORT clean-room | Added HLE/import summary fields to `RunReport`. |
| `apps/xport-cli/src/main.rs` | Original XPORT clean-room | Populates report summary fields for `inspect`/`run`. |

## Ported/adapted files (Lote 43 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetFileAttributesExA/W`, `SetEndOfFile` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 44 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `FindFirstFileA/W`, `FindNextFileA/W`, `FindClose` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 45 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetFullPathNameA/W`, `GetCurrentDirectoryA/W` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 46 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `SetCurrentDirectoryA/W`, `GetTempPathA/W` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 47 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetFileSize`, `GetFileInformationByHandle` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 48 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetVolumeInformationA/W`, `GetDiskFreeSpaceExA/W` host passthrough/fallback and Windows test coverage. |

## Ported/adapted files (Lote 49 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT minimal stubs | Added `GetLogicalDrives`, `GetDriveTypeA/W` host passthrough/fallback and Windows test coverage. |

## Embedded third-party material in upstreams (audit required before porting)

The upstream projects contain third-party material with their own licenses.
Do **not** copy these into XPORT without reviewing compatibility and keeping
their notices intact. Examples observed upstream:

- AMD machine-readable ISA / addrlib tiling patterns (per upstream notices)
- LLVM disassembler test data (per upstream notices)
- AnyPS5-derived logic referenced by XCL (GPL-2.0-only; compatible)
- libHttpClient headers / WinDurango references (per upstream notices)

When porting, add an entry here with: source path, upstream commit/release,
original license, files copied/derived, and any required notice text.

## Ported/adapted files (Lote 50 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernelx.rs` | Local XCL snapshot, `crates/xcl/src/hle/kernelx.rs` (GPL-2.0-only; owner-authorized relicensing) | Adapted the 80-byte `TitleMemoryStatus` ABI, `dw_length` validation and process-private-memory accounting; added `JobTitleMemoryStatus` alias. Snapshot has no accessible Git revision. |
| `crates/xport-hle/Cargo.toml` | Original XPORT change | Enabled Windows ProcessStatus APIs for process-memory accounting. |
## Ported/adapted files (Lote 51 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xmem.rs` | Local XCL snapshot, `crates/xcl/src/hle/kernelx.rs` (GPL-2.0-only; owner-authorized relicensing) | Adapted title allocation-hook dispatch and registration; default allocation remains host-backed; snapshot has no accessible Git revision. |
| `crates/xport-hle/src/modules/kernelx.rs` | Local XCL snapshot, `crates/xcl/src/hle/kernelx.rs` (GPL-2.0-only; owner-authorized relicensing) | Exposes the XMem API through `kernelx.dll` as well as `xmem.dll`. |
## Ported/adapted files (Lote 52 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xmem/physical.rs` | Local VIctory snapshot, `crates/victory-hle/src/modules/xmem.rs` (GPL-3.0-or-later) | Adapted 64 KiB pagefile-backed sections, page identifiers, CPU views and scatter mapping. Snapshot has no accessible Git revision. |
| `crates/xport-hle/src/modules/xmem.rs` | VIctory `victory-hle/src/modules/xmem.rs` and XCL `crates/xcl/src/hle/kernelx.rs` | Added placeholder reservations and cleanup needed by physical-page views. |

## Portado/adaptado (Lote 53 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/d3d12x.rs` | Local XCL snapshot, `HANDOFF.md` section “D3D12.X command list format (compute)” and `crates/xcl/src/hle/d3d12x/console_list.rs` (GPL-2.0-only; owner-authorized relicensing) | Adapted the documented compute record boundaries, root argument encodings and RTS0 parameter layout into a strict parser that emits an IR and rejects unsupported records. No shader translator or GPU replay implementation was copied. Snapshot has no accessible Git revision. |
## Portado/adaptado (Lote 54 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-shader/src/gpu/{rdna2.rs,rdna2_xbox.rs,dxbc.rs,xbox.rs}` | Local XCL snapshot, `crates/xcl/src/gpu/` (GPL-2.0-only; owner-authorized relicensing) | Adapted RDNA2 decoder and Xbox/DXBC container analysis. Snapshot has no accessible Git revision. |
| `crates/xport-shader/src/gpu/recomp/{emit.rs,mesh_dispatch.rs,mesh_wave.rs,mod.rs,upper_lanes.rs,x64_masks.rs}` | Local XCL snapshot, `crates/xcl/src/gpu/recomp/` (GPL-2.0-only; owner-authorized relicensing) | Adapted compute translation/analysis modules. XCL's DXC runner, PSO cache and tests were not copied. AnyPS5-derived MIMG translation and packed VALU16 translation were excluded and replaced with explicit unsupported-instruction errors. |
| `crates/xport-shader/src/gpu/rdna2_spec.rs` | AMD Machine-Readable GPU ISA Specification, RDNA 2, release 2026-02-20 | Generated instruction/encoding tables. AMD MIT notice is reproduced below. |
| `crates/xport-shader/src/dxc.rs` | Original XPORT | Windows DXC loader/compiler integration written for this project. |
| `crates/xport-gpu/src/d3d12.rs` | Original XPORT | Added D3D12 compute PSO creation from the compiled shader artifact; caller supplies a compatible root signature. |

## Portado/adaptado (Lote 55 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/d3d12_window.rs` | Inspired by XCL `crates/xcl/src/hle/d3d12x/sparse.rs` and `crates/xcl/src/gpu/uma.rs` (GPL-2.0-only; owner-authorized relicensing) | Clean-room subset: one reserved 4 GiB UAV buffer per window, 64 MiB tile pools, `UpdateTileMappings` and mirror→tile copies. No XCL source, write-watch hashing, staging rings, pools eviction or title-resource placement was copied. |
| `crates/xport-gpu/src/window.rs` | Inspired by XCL `gpu::uma` layout and `HANDOFF.md` (4 GiB windows, `u2..u5`) | Original XPORT address-layout code and tests. |

## Portado/adaptado (Lote 56 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/d3d12x.rs` | Inspired by XCL `HANDOFF.md` section “console's own compiler” and `crates/xcl/src/hle/d3d12x/bind.rs` (`bind_abi`) (GPL-2.0-only; owner-authorized relicensing) | Original XPORT packer change: whole root parameters in s4..s15, remainder in a shader argument table with `io[0..1]` address. No XCL source copied; only the documented ABI placement and existing XPORT code. |
| `crates/xport-gpu/src/d3d12.rs`, `crates/xport-gpu/src/d3d12_window.rs` | Original XPORT integration with the sparse backend introduced in Lote 55 | Added `upload_window_tile_bytes`, window resource/base accessors and the combined replay helper. No third-party code copied. |

## Portado/adaptado (Lote 57 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/descriptor.rs`, `crates/xport-gpu/src/d3d12_descriptor.rs` | Inspired by the documented console descriptor model in XCL `crates/xcl/src/hle/d3d12x/console_heap.rs` (GPL-2.0-only; owner-authorized relicensing) | Original XPORT code. Describes console tables as caller-supplied raw dword SRV/UAV views over XMem pages and materializes them in a shader-visible host heap. No XCL descriptor parsing, heap mirroring or source code copied. |
| `crates/xport-gpu/src/d3d12.rs` | Original XPORT integration with the descriptor mapper introduced in this lot | Added `submit_d3d12x_compute_stream_with_xmem_tables` to wire registration and mapping into the existing replay entry point. No third-party code copied. |

## Portado/adaptado (Lote 58 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/console_desc.rs` | GFX10 T#/V#/S# bit positions documented in AMD's ISA register reference; XCL `crates/xcl/src/gpu/desc.rs` consulted only to confirm those hardware field positions (GPL-2.0-only; owner-authorized relicensing) | Clean-room reader with original field decoding, validation and tests; no XCL source, tables or comments copied. |
| `crates/xport-gpu/src/descriptor.rs` | Original XPORT extension of the Lote 57 registry | Converts parsed V# slots into raw XMem views and reads console V#/S# tables from heap bytes. No third-party code copied. |
| `crates/xport-gpu/src/d3d12_descriptor.rs`, `crates/xport-gpu/src/d3d12.rs` | Original XPORT integration with the mappers introduced in Lotes 57 and 58 | Adds a shader-visible sampler heap, converts parsed S# to `D3D12_SAMPLER_DESC` and routes both table kinds in one replay entry point. No third-party code copied. |

## Portado/adaptado (Lote 59 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/console_desc.rs` | GFX10 IMG_FORMAT values and SRV dimension rules documented in AMD's ISA register reference; XCL `crates/xcl/src/gpu/desc.rs` consulted only to confirm format numbers and view dimensions (GPL-2.0-only; owner-authorized relicensing) | Clean-room IMG_FORMAT→DXGI table, component mapping and shape/mip/cube validation with original tests; no XCL source, tables or comments copied. |
| `crates/xport-gpu/src/descriptor.rs`, `crates/xport-gpu/src/d3d12_descriptor.rs` | Original XPORT extension of the Lotes 57–58 registries and mappers | Reads T# tables from console heap bytes, creates committed textures and SRVs in the existing CBV/SRV/UAV heap, and exposes the resources for a future uploader. No third-party code copied. |
| `crates/xport-gpu/src/d3d12.rs` | Original XPORT integration with the texture mapper introduced in this lot | Extends the combined replay entry point with a texture registry and heap capacity accounting. No third-party code copied. |

## Portado/adaptado (Lote 60 - 2026-11-24)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/texture_upload.rs` | GFX10 linear surface rules (256-byte rows) documented by AMD; XCL `crates/xcl/src/gpu/texture.rs`, `tiling.rs` and `hle/d3d12x/title_textures.rs` consulted only to identify which cases need addrlib tables (GPL-2.0-only; owner-authorized relicensing) | Clean-room linear planner with original offsets, format widths and tests; XOR/tiled, MSAA and block-compressed cases are rejected instead of decoded. No XCL source, tables or comments copied. |
| `crates/xport-gpu/src/d3d12.rs`, `crates/xport-gpu/src/d3d12_descriptor.rs` | Original XPORT integration using the public D3D12 `GetCopyableFootprints`/`CopyTextureRegion` API | Fenced GPU-VA readback, staged texture upload and per-T# planning in the replay entry point. No third-party code copied. |

## Portado/adaptado (Lote 61 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/d3d12x.rs` | XCL `crates/xcl/src/hle/d3d12x/console_list.rs` and `HANDOFF.md` consulted only to document the synthetic console-list replay tags `0x8C0F_E005/E007/E008` and the 8-dword barrier snapshot shape (GPL-2.0-only; owner-authorized relicensing) | Clean-room decoder with original error types, validation and synthetic tests. The tags are documented as shim replay markers, not proven console wire opcodes; only the documented marker values and field order are shared for format compatibility. No XCL source, comments or fixtures were imported. |
| `crates/xport-gpu/src/d3d12.rs` | Original XPORT integration with the public D3D12 `ResourceBarrier`/`CopyBufferRegion` API | Recorder barrier/copy helpers and replay state validation written for XMem page mirrors. No third-party code copied. |

## Portado/adaptado (Lote 62 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-gpu/src/d3d12x.rs` | XCL `crates/xcl/src/hle/d3d12x/console_list.rs` and `HANDOFF.md` consulted only to document the synthetic `0x8C0F_E00A` `CopyTextureRegion` tag and its 35-dword field order (GPL-2.0-only; owner-authorized relicensing) | Clean-room location/box types, validation and synthetic tests; only the documented marker value and field order are shared. No XCL source, comments or fixtures were imported. |
| `crates/xport-gpu/src/d3d12.rs` | Original XPORT integration with the public D3D12 `CopyTextureRegion` API and the project's own T# texture registry | Recorder method, texture resolver entry points and XMem page/footprint validation written from scratch. No third-party code copied. |

## Portado/adaptado (Lote 63 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xgameruntime.rs` | VIctory `crates/victory-hle/src/modules/xgameruntime.rs` (GPL-3.0-or-later; owner-authorized relicensing) for `Guid`/`Object`/`Methods`/`Interface`, `QueryApiImpl` and helpers; XCL `crates/xcl/src/hle/xgameruntime/api.rs` and `gdk_table.rs` (GPL-2.0-only; owner-authorized relicensing) consulted for GDK GUID/slot facts | Core rewritten for XPORT with the existing export set and explicit unsupported-area errors. XCL's PC-runtime forwarding and generated table were **not** copied; only documented identity/slot facts informed the area registrations. |
| `crates/xport-hle/src/modules/xgame_api.rs` | VIctory `crates/victory-hle/src/modules/xgame.rs`; XCL `gdk_table.rs` (invite interface/slots) and `api.rs` (refused store APIs) | Offline title id, features, invites and events adapted; `XLaunchNewGame` logs instead of relaunching because XPORT has no lifecycle hook. Wider invite vtable covers XCL's pending-invite slots with explicit `E_NOTIMPL`. |
| `crates/xport-hle/src/modules/xsystem_api.rs` | VIctory `crates/victory-hle/src/modules/xsystem.rs`; XCL `api.rs` for `XSYSTEM_V2_IID`, `XSYSTEM_V3_IID` and the 2026 interface id | Offline console/sandbox/device strings and analytics adapted. XPORT reports a fixed Series X identity; Windows registry device-type probing was not ported. |
| `crates/xport-hle/src/modules/mod.rs` | Original XPORT module list | Declares the two new `pub(crate)` area modules. |

## Portado/adaptado (Lote 64 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/vfs.rs` | VIctory `crates/victory-hle/src/modules/files.rs` (GPL-3.0-or-later; owner-authorized relicensing) for the mounted-drive path rules, verbatim prefix and narrow-to-wide convention | Rewritten around XPORT's `xport-vfs::MountTable`. No drive-allocation code, UTF-16 mount table or upstream host-function macro was imported. Write intent and overlay refusal are XPORT rules. |
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT pass-through wrappers | The existing `CreateFile*`, attribute, delete and directory exports now route translated paths to the host twin. Handle-based calls (`ReadFile`, `WriteFile`, pointers) are unchanged. |
| `crates/xport-vfs/src/lib.rs` | Original XPORT VFS | Added `read_path` so reads stay on the game root while `decide_write` keeps writes on the overlay. No upstream code copied. |

## Portado/adaptado (Lote 65 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/vfs.rs` | Original XPORT extension of the Lote 64 translation | Added `title_mounts` from XPORT's own `Config` directories. Drive letters follow the console conventions documented by VIctory/XCL; no upstream code copied. |
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT pass-through wrappers | `FindFirstFileA/W` now translates the search pattern; `FindNextFile`/`FindClose` remain handle-based. No upstream code copied. |
| `apps/xport-cli/src/main.rs` | Original XPORT CLI | Reads `Config` and installs the mount table for `G:`/`GAME:`/`T:` plus the overlay. |

## Portado/adaptado (Lote 66 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/kernel.rs` | Original XPORT enumeration wrappers | Added synthetic find handles that merge host enumerations of the game root and the overlay, deduplicate names case-insensitively and serve both `FindNextFileA/W` and `FindClose`. No upstream code copied. |
| `crates/xport-hle/src/vfs.rs` | Original XPORT extension of the Lote 65 translation | Added `translate_overlay_wide` to expose the overlay counterpart of a console pattern. No upstream code copied. |

## Portado/adaptado (Lote 67 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/storage.rs` | VIctory `victory-hle/src/modules/storage.rs` (GPL-3.0-or-later); XCL `crates/xcl/src/hle/xgameruntime/mod.rs` (`XPersistentLocalStorage*`, GPL-2.0-only upstream, owner-authorized relicensing for XPORT) | Adapted the drive-path/space-info contract and recursive quota accounting. Prompt/XAsync and `MountForPackage` slots are intentionally unimplemented and fail explicitly. |
| `crates/xport-hle/src/modules/xgameruntime.rs` | Original XPORT COM registry; GUIDs cross-checked with XCL's GDK table | Registers persistent local storage's provider/IIDs. |
| `crates/xport-hle/src/vfs.rs` | Original XPORT mount translation | Added `mount_drive` so storage installs `S:` without replacing other mounts. |
| `apps/xport-cli/src/main.rs` | Original XPORT CLI | Configures PLS from `Config::data_subdir("pls")`. |

## Correções da revisão geral (Lote 68 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xgameruntime.rs` | XCL `crates/xcl/src/hle/xgameruntime/api.rs` (`query_interface`, GPL-2.0-only upstream, owner-authorized relicensing) | Slot 0 now returns `E_NOINTERFACE` instead of stray memory; `QueryApiImpl` rejects a null output with `E_INVALIDARG`. |
| `crates/xport-hle/src/modules/xgame_api.rs` | XCL GDK table and `XGameRuntimeIsFeatureAvailable` | Vtable extended to slot 5; feature set aligned to XCL. |
| `crates/xport-hle/src/modules/xsystem_api.rs` | XCL GDK table | Vtable extended through slot 8; newer slots fail explicitly; analytics rejects null. |
| `crates/xport-hle/src/vfs.rs`, `crates/xport-hle/src/modules/kernel.rs` | Original XPORT VFS pass-through | `CreateFileA/W` now forwards flags; `GENERIC_ALL`, `DELETE`, `WRITE_DAC/OWNER` and `FILE_FLAG_DELETE_ON_CLOSE` count as write intent. |
| `crates/xport-vfs/src/lib.rs`, `crates/xport-hle/src/storage.rs` | Original XPORT mount table | Added `remove_mount`/`unmount_drive` so `storage::clear` removes `S:`. |

## Portado/adaptado (Lote 69 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xasync.rs` | VIctory `victory-hle/src/modules/xasync.rs` (GPL-3.0-or-later, owner-authorized relicensing for XPORT) | Adapted task queues and async calls. XPORT uses opaque handle tables and a call id in the block's system area instead of raw `Arc` pointers; wait-handle registration, XThread time-sensitivity and PC forwarding remain explicit `missing` slots. GUIDs/slots cross-checked with XCL `gdk_table.rs` (GPL-2.0-only upstream, owner-authorized relicensing). |
| `crates/xport-hle/src/modules/xuser.rs` | VIctory `victory-hle/src/modules/xuser.rs` (GPL-3.0-or-later, owner-authorized relicensing for XPORT) | One offline signed-in user, async add, gamertag and privileges. Tokens fail with `FAILED_TO_GET_TOKEN`; no audio endpoint is associated. Slots extended to `0x120` and device slots to `0x50` with explicit `missing` entries, following XCL's GDK table. |
| `crates/xport-hle/src/storage.rs` (space prompts) | VIctory `victory-hle/src/modules/storage.rs` (GPL-3.0-or-later, owner-authorized relicensing for XPORT) | Slots `0x30`/`0x38` now complete through `xasync`; `MountForPackage` remains explicit `E_NOTIMPL`. |

## Portado/adaptado (Lote 70 - 2026-11-25)

| XPORT file | Upstream source | Notes |
|---|---|---|
| `crates/xport-hle/src/modules/xaudio2.rs` | XCL `crates/xcl/src/hle/xaudio2.rs` (GPL-2.0-only upstream, owner-authorized relicensing for XPORT GPL-3.0-or-later) | First conservative slice: `XAudio2Create`/`XAudio2CreateWithVersionInfo` forward to the host engine with console-only flags masked off. XPORT does not copy or patch the Windows vtable; compressed console formats are left to fail explicitly in the host engine instead of being silently decoded. |

## Sem código de terceiros (Lote 71 - 2026-11-25)

- `crates/xport-shader/src/gpu/xbox.rs` (`synthetic`) e `crates/xport-gpu/tests/d3d12_hw.rs`: os builders de container e o programa RDNA2 sintético são autorais, escritos para exercitar o parser/tradutor do XPORT. Nenhum shader, dump, asset ou bytecode de jogo foi copiado.
- Os bytes das instruções foram montados a partir do layout já documentado no próprio módulo e da especificação AMD licenciada abaixo; esta fatia não portou nem adaptou código novo de VIctory, XCL ou qualquer outro projeto.

## Sem código de terceiros (Lote 72 - 2026-11-25)

- `crates/xport-gpu/src/d3d12.rs`, `crates/xport-gpu/src/lib.rs` e `crates/xport-gpu/tests/d3d12_hw.rs`: o modelo de estado de textura por subrecurso, a importação de filas e os fence waits são código autoral do XPORT, escrito contra a API D3D12 documentada do Windows SDK.
- Nenhuma tabela, asset, dump, shader ou código de VIctory, XCL ou outro projeto foi portado nesta fatia; o fixture usa apenas uma página XMem preparada pelo próprio bridge.

## Sem código de terceiros (Lote 73 - 2026-11-25)

- `crates/xport-hle/src/modules/gpu_replay.rs`, a extensão de exports em `crates/xport-hle/src/modules/xgameruntime.rs`, o fixture `tests/fixtures/xport-synthetic-title` e o teste `crates/xport-hle/tests/synthetic_title.rs`: a ABI tipada `XPortGpuReplay*` e os comandos enviados ao bridge são autorais do XPORT.
- A integração reutiliza o decoder de `xport-gpu` e a tabela de GUIDs já registrada de VIctory/XCL; não copiou novos GUIDs, vtable slots, opcodes, shaders, dumps ou assets. A ABI adicionada é sintética e não deve ser interpretada como GDK/D3D12.X.

## Sem código de terceiros (Lote 74 - 2026-11-25)

- `crates/xport-gpu/src/d3d12.rs`, `crates/xport-hle/src/modules/gpu_replay.rs`, `crates/xport-hle/src/modules/xgameruntime.rs` e `tests/fixtures/xport-synthetic-title`: a criação de fila no device do bridge, os exports `XPortGpuReplayDecodedSummary`/`XPortGpuReplayCreateQueue`, o stream encadeado e os testes de ordenação entre filas são código autoral do XPORT, escritos contra a API D3D12 documentada do Windows SDK.
- Nenhum dump, shader, tabela, asset ou código de VIctory, XCL ou outro projeto foi portado nesta fatia. Os identificadores de pipeline do fixture são marcadores sintéticos inventados para o teste, não PSOs de jogo.

## Sem código de terceiros (Lote 75 - 2026-11-25)

- `crates/xport-hle/Cargo.toml`, `crates/xport-hle/src/modules/gpu_replay.rs`, `crates/xport-hle/src/modules/xgameruntime.rs`, `tests/fixtures/xport-synthetic-title/src/main.rs` e `crates/xport-hle/tests/synthetic_title.rs`: a dependência de `xport-shader`, os exports `XPortGpuReplayRegisterSyntheticPipeline`/`XPortGpuReplayBindXmemPage`/`XPortGpuReplayDispatchDecoded`, o agrupamento dos dispatches decodificados e o readback são código autoral do XPORT, usando apenas containers e páginas gerados no próprio workspace.
- O shader sintético já registrado no Lote 71 continua sendo a única entrada de compilação; nenhum bytecode, dump, tabela, asset ou código de VIctory, XCL ou outro projeto foi acrescentado. Os valores `0x1234`, `0x5678` e `0x68ac` são dados de teste autorais.

## Sem código de terceiros (Lote 76 - 2026-11-25, `apps/xport-media`)

- `apps/xport-media/src/png.rs`: escritor PNG mínimo autoral (CRC32, Adler32 e blocos DEFLATE `stored`). A imagem é gerada por código (gradiente + xadrez), não é asset de jogo.
- `apps/xport-media/src/wav.rs`: gerador/parser RIFF/WAVE PCM 16-bit autoral. O WAV sintético é gerado em código e lido de volta de arquivo temporário fora do repo.
- `apps/xport-media/src/image_view.rs`: decode via WIC e apresentação via GDI `StretchDIBits`, ambos documentados no Windows SDK. Segue apenas a **ideia** já usada pelo VIctory (`victory-gpu/src/host.rs`, host codecs/Presenter) de usar os codecs do host; nenhuma linha de código, swapchain, captura ou helper do VIctory foi copiada.
- `apps/xport-media/src/audio.rs`: chama o COM do XAudio2 2.9 do host diretamente, sem patch de vtable. Os números de slot (`CreateSourceVoice`=5, `CreateMasteringVoice`=7, `DestroyVoice`=18, `SubmitSourceBuffer`=21) são fatos de ABI confirmados no `xaudio2.h` do Windows SDK e apenas **conferidos** com a tabela documentada em XCL `crates/xcl/src/hle/xaudio2.rs` (GPL-2.0-only). Nenhuma linha de código XCL foi portada; o engine vem do export `XAudio2Create` do próprio HLE do XPORT.
- Em 25/11/2026 o usuário autorizou explicitamente a **leitura somente dos PNGs soltos** de `F:\GTA6.Xbox.v1.0.2.1-CYBERLEEK` para validação via `--image=`. Nenhum arquivo, código, tabela, shader ou asset foi copiado para o repo, modificado ou executado; os containers `*.rpf` (inclusive `audio.rpf`) e os binários permanecem intocados.

## Sem código de terceiros (Lote 77 - 2026-11-25, compatibilidade do título real)

- `crates/xport-hle/src/modules/title_compat.rs`: tabela autoral de 112 exports nomeados a partir dos imports que o próprio `xport inspect` reportou no executável fornecido pelo usuário. São stubs que contam e registram a chamada e devolvem zero; nenhuma tabela, código, DLL ou string de jogo foi copiada (apenas os nomes dos imports, necessários para o binding).
- `crates/xport-hle/src/modules/win32_bridge.rs`: pass-through para `user32.dll` via `LoadLibraryExA`/`GetProcAddress` documentados no Windows SDK, cobrindo os sete nomes de API-set `ext-ms-win-rtcore-ntuser-*`. Nenhum código do XCL/VIctory foi copiado.
- `crates/xport-hle/src/modules/mod.rs`: apenas registro dos dois módulos novos. Nenhum asset, dump, shader ou binário foi trazido para o repo.

## Sem código de terceiros (Lote 78 - 2026-11-25, implementações reais)

- `crates/xport-hle/src/modules/{win32_bridge,host_bridge,apu_bridge,bink_bridge,http_bridge,xbox_bridge}.rs`: implementações autorais. Contêm apenas nomes de imports observados pelo próprio `xport inspect`, chamadas documentadas do Windows SDK (Win32, WinHTTP, D3D12, Media Foundation) e estruturas de estado próprias.
- Nenhum código, DLL, tabela, shader, codec ou asset de XCL, VIctory, RAD/Bink, AMD FFX, libHttpClient ou do título foi copiado. Os nomes de exports C++ mangled do XboxBvhBuilder são identificadores de import, não código.
- `title_compat.rs` (stubs do Lote 77) foi removido após todos os exports serem migrados para os módulos acima.

## Sem código de terceiros (Lote 79 - 2026-11-25, segurança e XNetworking offline)

- `crates/xport-hle/src/modules/winhttp_offline.rs`: superfície autoral com os quinze nomes de import observados em `WINHTTP.dll`. Todas as chamadas falham e são registradas; nenhum socket ou upload é criado. Os nomes são identificadores de import, não código do título.
- `crates/xport-hle/src/modules/http_bridge.rs`: passou a desativar a rede por padrão (`XPORT_ALLOW_NETWORK=1` para opt-in). A mudança foi motivada por um upload real de minidump feito pelo Breakpad incorporado ao título; nenhum dado de crash é enviado nas execuções de teste.
- `crates/xport-hle/src/modules/xnetworking.rs`: implementação offline própria da área XNetworking. Apenas os GUIDs e os números de slot públicos foram conferidos em XCL `crates/xcl/src/hle/xgameruntime/gdk_table.rs` (GPL-2.0-only upstream, owner-authorized relicensing); nenhuma função, estrutura interna, comentário ou tabela do XCL foi copiada. UDP port, security info, connectivity hint e tokens são valores determinísticos do XPORT; certificados/configurações/estatísticas que exigem rede falham explicitamente.

## Sem código de terceiros (Lote 80 - 2026-11-25, XPackage e XDisplay offline)

- `crates/xport-hle/src/modules/xpackage.rs`: implementação autoral da área `XPackage`. Os GUIDs e os números de slot foram conferidos apenas como fatos públicos em XCL `crates/xcl/src/hle/xgameruntime/gdk_table.rs` (GPL-2.0-only upstream, owner-authorized relicensing). Identificador, localidade e mount path são valores determinísticos do XPORT; instalação, download e montagem com UI falham explicitamente. Nenhuma função ou estrutura interna do XCL foi copiada.
- `crates/xport-hle/src/modules/xdisplay.rs`: implementação autoral da área `XDisplay`. GUID/slot foram conferidos em XCL `gdk_table.rs`; a assinatura pública de `XDisplayTryEnableHdrMode`, o enum `XDisplayHdrModeResult` e a estrutura `XDisplayHdrModeInfo` vieram da documentação pública da Microsoft (`learn.microsoft.com`, XDisplay.h/GDK). O método reporta `Disabled` e não consulta nem altera o display do host.
- `crates/xport-hle/src/modules/{mod,xgameruntime}.rs`: apenas registro das duas áreas. Nenhum asset, dump, DLL, shader, tabela ou código de jogo entrou no repo.
- Slot 39 (`0x138`) de `XPackage`: `XPackageEnumeratePackages(kind, scope, context, callback)` e o layout de `XPackageDetails`/`XVersion` foram tirados da documentação pública da Microsoft (`learn.microsoft.com`, XPackage.h/GDK); o slot foi conferido na tabela pública do XCL. O callback é chamado uma vez com um pacote sintético do XPORT. Nenhum dado de pacote, loja ou metadado do título foi lido ou copiado.

## Sem código de terceiros (Lote 81 - 2026-11-25, dispositivo D3D12.X offline)

- `crates/xport-hle/src/modules/d3d12_null.rs` e `d3d12_null_stubs.rs`: dispositivo COM sintético autoral com vtable de 256 slots para `D3D12XboxCreateDevice`. A ordem dos slots e os offsets de estruturas públicas (`D3D12_HEAP_PROPERTIES`, `D3D12_RESOURCE_DESC`, `D3D12_COMMAND_QUEUE_DESC`, `D3D12_FEATURE_D3D12_OPTIONS5`) são fatos públicos do DirectX/GDK; a ordem e o IID `5405c344-d457-444e-b4dd-2366e45aee39` (`ID3D12Device11`) foram identificados no próprio executável e conferidos na tabela pública do XCL. Nenhuma função, struct interna, comentário ou tabela do XCL foi copiada. Todos os corpos são emulação determinística do XPORT: nenhum adapter, driver, shader ou recurso real é criado.
- `crates/xport-hle/src/modules/xbox_bridge.rs`: `D3D12XboxCreateDevice` agora devolve o dispositivo sintético acima em vez de encaminhar argumentos console para `D3D12CreateDevice` do host. O encaminhamento direto causava `E_INVALIDARG` e, depois de corrigido, um acesso inválido dentro de `D3D12Core.dll` porque o título usa `nodeIndex=1`. A substituição é documentada como compatibilidade/offline, não como implementação D3D12.

### Continuação do Lote 81 (25/11/2026, 20:05Z) — emulação adicional

- `d3d12_null.rs`: `QueryInterface` do dispositivo sintético responde aos IIDs públicos `IDXGIObject`, `IDXGIDevice`, `IDXGIDevice1` e `IDXGIDevice2` com um objeto DXGI sintético separado. Isso é necessário porque o título chama `IDXGIDevice::GetAdapter` no slot 7; devolver o próprio dispositivo fazia o slot 7 escrever o número de nós (`1`) onde o título esperava um ponteiro. `GetAdapter` e `IDXGIAdapter::EnumOutputs` também são emulações determinísticas (o out de `EnumOutputs` fica no terceiro argumento, conforme a ABI documentada). Nenhum display ou adapter real do host é consultado.
- `d3d12_null.rs`: slots 48 (`CreateCommandQueueX`), 73 (`CreateCommandListX`) e 21 da cadeia DXGI (enumeração de modos, com um modo sintético 1920x1080@60) escrevem out-pointers/valores plausíveis. `default_stub(slot)` e `call_raw()` permanecem como instrumentação autorais; os índices de slot vieram do próprio executável e foram conferidos apenas com tabelas públicas de layout/IIDs (XCL e documentação Microsoft), sem cópia de código.
- `crates/xport-hle/src/modules/xmem.rs`: `host_kind()` remove bits de seção/rotação do console (por exemplo `MEM_MAPPED` em `kind=0x42000`) antes de chamar `VirtualAlloc` do host. São fatos públicos da API Win32; a máscara é autoral. Isso eliminou a falha de `XMemVirtualAlloc` de 185 MB observada em `scarlett-exec22`.
- `crates/xport-hle/src/modules/win32_bridge.rs`: log de debug do pass-through de API-set e um teste local que registra uma classe com `RegisterClassExA` real do host. Nenhum código de terceiros foi copiado; `scarlett-exec24` confirmou `RegisterClassExA=49878` e `CreateWindowExA=HWND`, ambos bem-sucedidos.

## Sem código de terceiros (Lote 83 - 2026-11-25, display target e aliases físicos)

- `crates/xport-hle/src/modules/xmem.rs`: `D3D_DISPLAY_TARGET` usa apenas o layout público de `XALLOC_ATTRIBUTES` (page size, allocator, memory type) conferido em XCL `crates/xcl/src/hle/kernelx.rs` (GPL-2.0-only upstream, owner-authorized relicensing); a escolha do valor e o caminho `runtime_alloc` são autorais. A adoção de `__XMemAllocationHooks__` lê somente dois endereços do export de dados do próprio executável mapeado; nenhum código do título é copiado para o repo.
- `crates/xport-hle/src/modules/d3d12_null.rs`: o slot 127 usa o bit público `D3D12_HEAP_FLAG_ALLOW_DISPLAY` e o layout público de `D3D12_RESOURCE_DESC` para decidir quando pedir memória ao hook do título. Os bumps `0x3e10000`/`0x1000` são emulação autoral da arena de 64 MiB observada no próprio título, não uma tabela ou código de jogo.
- `crates/xport-hle/src/modules/xmem/physical.rs`: o fallback que commita uma reserva comum do console é código autoral contra `VirtualAlloc`/`MapViewOfFile3` documentados no Windows SDK; nenhuma implementação de sparse/VRAM do XCL/VIctory foi portada. O diagnóstico `XPORT_DUMP_SLOT` é opt-in, usa `ReadProcessMemory` e não grava dados lidos no repo.
- `scarlett-exec60..62`: apenas logs de execução em `%LOCALAPPDATA%\Temp\opencode`; nenhum dump, asset, DLL, shader ou binário de jogo entrou no XPORT.

## Portado/adaptado (Lote 87 - 2026-11-25, fila D3D12.X "newer" e relógio de frame)

- `crates/xport-hle/src/modules/d3d12_null.rs`: tabela separada para `Kind::CommandQueue` com o deslocamento de +2 nos slots a partir de `0x70`, confirmado em `victory-main v5` (`crates/victory-gpu/src/queue.rs:1011-1034`, GPL-3.0-or-later, fork autorizado pelo proprietário). Os números de slot (`0x80` Signal, `0x88` Wait, `0x90` timestamp, `0x98` clock, `0xe0` GetDesc, `0x100` PresentX, `0x110` Signal2X, `0x118` submissão com token) e a frequência de 100 MHz são fatos de ABI pública; os corpos dos handlers são emulação autoral do XPORT. Nenhuma tabela, shader ou binário do fork foi copiado.
- `crates/xport-hle/src/modules/d3d12_frame.rs`: emulação autoral do relógio de frame (device slots 78-81) escrita para o XPORT, com a convenção pública `S_FALSE` para timeout e tokens consecutivos. A estrutura conceitual foi adaptada de `victory-main v5` (`crates/victory-gpu/src/frame.rs`, GPL-3.0-or-later, autorizado); o código é do XPORT. Nenhum display do host é consultado.
- `crates/xport-hle/src/modules/mod.rs`: registro do novo módulo; nenhum código de terceiros adicional.
- `scarlett-exec86`: apenas logs em `%LOCALAPPDATA%\Temp\opencode`; nenhum dump, asset, DLL, shader ou binário de jogo entrou no XPORT.

## Portado/adaptado (Lote 88 - 2026-11-25, DirectStorage offline com leitura real)

- `crates/xport-hle/src/modules/host_bridge.rs`: os números de slot da ABI (`IDStorageFactoryX` 4=`OpenFile`, 5=`CreateStatusArray`, 8=`SetStagingBufferSize`, 10=`CreateQueue`; fila 3=`EnqueueRequest`, 4=`EnqueueStatus`, 6=`Submit`, 8=`Close`; status 3=`IsComplete`, 4=`GetResult`) e o layout do `Request` (0x48 bytes) foram conferidos em `victory-main v5` (`crates/victory-hle/src/modules/dstorage.rs`, GPL-3.0-or-later, fork autorizado pelo proprietário). Nenhuma função, struct, comentário, tabela BCPack ou binário do fork foi copiado.
- Os corpos são emulação autoral do XPORT: tradução de caminho pelo VFS, leitura de arquivo por `std::fs`, conclusão síncrona e zlib por `miniz_oxide`. BCPACK/page-mode falham explicitamente com `E_NOTIMPL`; nenhuma tabela de compressão do jogo ou do V5 entrou no repo.
- `crates/xport-loader/src/crash.rs`: o despejo adicional de `rsi+0x30` continua sob `XPORT_DUMP_RSI`, desligado por padrão, e usa apenas `ReadProcessMemory` para diagnóstico local.

### AMD Machine-Readable GPU ISA Specification

MIT License

Copyright (c) 2026 Advanced Micro Devices, Inc., or its affiliates.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## Lotes 115–118 (26/11/2026) — ABI APU/HAL e shims de compatibilidade

- `crates/xport-hle/src/modules/apu_bridge.rs`: a ABI console de
  `AcpHalAllocateShapeContexts` (um único `ShapeContexts*` com `counts: [u32;5]`,
  `reserved: u32` e `contexts: [*mut c_void;5]`), `ApuAlloc`, `ApuFree`,
  `ApuMapApuAddress`, `ApuMapVirtualAddress`, `ApuHeapGetState` e a vtable de
  6 slots de `AcpHalCreate` foram adaptadas de `xcl-nvidia-dev`
  (`crates/xcl/src/hle/acphal.rs`, GPL-2.0-only; relicenciamento autorizado
  pelo proprietário para GPL-3.0-or-later no XPORT). Os corpos, validações e
  mensagens de erro são do XPORT; nenhum arquivo do fork foi copiado.
- `crates/xport-loader/src/crash.rs`: os shims de compatibilidade são código
  autoral. Eles reconhecem exceções em RVAs exatos do build verificado e
  retomam a própria instrução seguinte do título — watchdog, fonte nula, slot
  gráfico indisponível e helper inutilizável. Detectam `call` indireto
  (`ff /2` com REX/disp32 opcionais) lendo os bytes da imagem já mapeada e
  validando a palavra de vtable em memória. Emular o `ret` da chamada pulada
  preserva a pilha do chamador. Nenhum byte da imagem é alterado e nenhum dado
  do título é gravado no repositório; todo diagnóstico é opt-in.
- `crates/xport-hle/src/modules/d3d12_va_map.rs`: o guard em `copy()` é autoral,
  falha fechada para tamanho zero/ponteiro nulo e remove um panic de
  precondition do runtime.

## Lote 120 (26/11/2026) — XSEmu e leitura das planes de `PresentX`

- O proprietário autorizou em 2026-11-26 que o código autoral do
  [XSEmu](https://github.com/xemu-project/XSEmu) (GPL-2.0-only no snapshot local
  `XSEmu-main`) seja tratado como seu e relicenciado para GPL-3.0-or-later no
  XPORT, com a mesma regra já aplicada a VIctory/XCL: preservar atribuição e
  auditoria de componentes de terceiros antes de cada port.
- `crates/xport-hle/src/modules/d3d12_present.rs`: o layout das planes de
  `PresentX` (stride `0x48`, token @0, resource count @8, resource list @0x10) e
  a ideia de acompanhar o recurso apresentado vêm de
  `crates/xcl/src/hle/d3d12x/present.rs` no XSEmu. O corpo é autoral: leitura
  limitada com `ReadProcessMemory`, nenhum ponteiro de título é mantido, nenhum
  byte é copiado ao repositório e nenhuma janela ou swapchain de host é criada.
- As partes de terceiros do XSEmu não foram portadas nesta fatia: AnyPS5
  (GPL-2.0-only, sem autorização de relicenciamento aqui declarada), FFmpeg
  (LGPL-2.1+), AMD ISA e Mesa addrlib (MIT), LLVM (Apache-2.0 WITH
  LLVM-exception), DirectX-Headers (MIT) e WinDurango (LGPL-3.0, apenas fatos
  de exportação). Se alguma dessas áreas for portada depois, este arquivo deve
  ser atualizado com o aviso e a licença correspondentes.

## Lote 122 (26/11/2026) — command list console-shaped

- O formato da lista entregue ao título foi conferido no código autoral do
  XSEmu (`crates/xcl/src/hle/d3d12x/title_list.rs`, GPL-2.0-only no snapshot,
  relicenciado para o XPORT pelo proprietário): objeto de `0x3000` bytes,
  `write`/`dispatch_end`/`end` iguais a `base+0x1000`, estado em `+0x2f00` e
  ponteiro de volta em `+0x2ff0/+0x2ff8`. A igualdade dos ponteiros força os
  writers inline a caírem na vtable; o stream não é decodificado.
- `crates/xport-hle/src/modules/d3d12_null.rs`: implementação própria do
  formato e do estado interno; os números de slot e o esquema de remapeamento
  console foram usados apenas como fatos de ABI, sem copiar funções, thunks,
  comentários ou tabelas do XSEmu. O decoder compute-only
  (`XCL_CONSOLE_LISTS`) continua fora do caminho gráfico do XPORT.
- `XPORT_LOG_CL_SHAPE`, `XPORT_LOG_CL_CALLS` e `XPORT_LOG_XMEM_ALLOC` são
  diagnósticos opt-in autorais, desligados por padrão. Nenhum packet, stream,
  shader, textura ou dado de título foi gravado no repositório.
- `XPORT_REPORT_ALLOC_ASSERT=1` e a classificação da assertion recuperada pelo
  próprio título são lógica do XPORT, sem código do XSEmu. Nenhuma imagem, DLL,
  dump ou tabela binária do título entrou no repo.

## Lote 123 (26/11/2026) — reserva XMem preguiçosa

- `crates/xport-loader/src/crash.rs` e `crates/xport-hle/src/modules/xmem.rs`:
  shim autoral de compatibilidade. O HLE registra intervalos devolvidos por
  `XMemVirtualAlloc`; o VEH comita até 64 KiB somente para uma escrita inválida
  dentro de um intervalo registrado cujo estado host é `MEM_RESERVE`. Nenhum
  endereço arbitrário é comitado e nenhum byte da imagem do título é alterado.
- O diagnóstico `XPORT_LOG_XMEM_ALLOC` e o teste que força a condição são
  originais; nenhum código, comentário, tabela ou dado do XSEmu/terceiros foi
  portado nesta fatia. Os componentes de terceiros listados no Lote 120
  permanecem excluídos.
# Plano de adoção — `victory-main v5` → XPORT

Análise integral concluída em 2026-11-25. Fonte: `C:\Users\User\Downloads\victory-main v5\victory-main`
(224 arquivos, 46 diretórios, GPL-3.0-or-later). Usuário autorizou portar código autoral do fork
para o XPORT sob GPL-3.0-or-later. **Nada foi copiado ainda.** Toda porta exige uma linha em
`THIRD_PARTY.md` antes do código entrar.

## 0. Expectativa calibrada

O próprio V5 não é "jogo funcionando": boot, menus e vídeo introdutório funcionam; o prólogo roda a
7–9 fps e ray tracing está desligado porque o builder de estruturas não é emulado
(`victory-main/README.md:35-47`). O gargalo é a thread de submissão, não a GPU. Portanto o alvo
realista para o XPORT é **chegar ao menu/intro de forma estável**, não partida completa.

## 1. Achados por subsistema

| Relatório | Conclusão principal |
|---|---|
| Loader/boot | XPORT já replica `prepare/load/bind/TLS/thunks/spawn`. Falta ISA do console por decodificação, crash report completo, `LateDlls`, manifest/title-id, Agility e seletor de console. |
| GPU/D3D12.X | V5 usa **tabelas por tipo de objeto + `missing_table`**; XPORT usa uma tabela única de 256 slots com heurística `this_of` — causa dos crashes de vtable nula. Confirma fila "newer" com `0x100` Signal2X, `0x108` ExecuteX, `0x118` token, e slots DirectStorage/XG mais completos. |
| HLE de sistema | Prioridade: `xpackage` (ID/progresso/mount), guarda MF para não furar `winhttp_offline`, módulos `xstore`/`xgamesave`/`speech`/`xgameui`, display shell/capture. |
| Docs/formatos | Valor está nas **regras e medições**, não em artefatos. staging exato, volatilidade de descritores, PSO lazy por conteúdo, e um longo ledger do que já foi testado e descartado. |

## 2. Prioridades

### P0 — destravar o boot atual

1. **Tabelas por `Kind` + `missing_table`** em `crates/xport-hle/src/modules/d3d12_null.rs` e
   `d3d12_null_stubs.rs`. Portar o padrão de `victory-gpu/src/object.rs:42-144,273-359`: cada
   objeto começa com `{vtable, bits, refs}`, slots não implementados apontam para `missing` com
   log único e `E_NOTIMPL`, `out=NULL` → `S_FALSE`. Elimina a heurística `this_of` e o risco de
   zerar a vtable do COM object. **Risco médio** — toca em todos os slots já estabilizados.
2. **Fila "newer" + relógio de frame** em `d3d12_null.rs`. Implementar `0x100`/`0x108`/`0x118` e
   device `0x270/0x278/0x280/0x288` (`victory-gpu/src/queue.rs:990-1034`, `frame.rs:45-180`).
   O crash `+0x2482689` era `call [rax+0x118]` — é submissão com token, não ExecuteCommandListsX.
   **Risco baixo-médio** — ataca diretamente os crashes atuais.
3. **ISA do console por decodificação** em `crates/xport-loader/src/crash.rs` + novo
   `console_insn_decode.rs`. Portar `console_insn_decode.rs:88-160` e `console_insn.rs:106-249`:
   `clzero`/loop, `clwb`, `rdpru`, `rdpid`, `monitorx/mwaitx`, SSE4a `extrq/insertq`. Substitui
   `emulate_clzero` por RVAs fixos e o NOP de `clwb` em `prepare.rs`. **Risco médio** — isolar em
   feature e testar com fixtures.
4. **XPackage/VFS alinhado** em `modules/xpackage.rs`, `vfs.rs`, `modules/kernel.rs`. ID do título,
   progresso completo, recusa de mount de `packroot:/`/`appdata0.rpf`; logar também o caminho host
   traduzido (`victory-hle/src/modules/files.rs:203-210`). **Risco médio** — intersepta o crash
   `+0x2237178` e o problema dos `.rpf` em subdiretório.
5. **Guarda MF** em `modules/host_bridge.rs`. Portar o proxy `file:`/`localhost` de
   `victory-hle/src/modules/media.rs:41-145` para `MFCreateSourceResolver`; hoje URL `http`/`rtsp`
   pode sair via MF e furar a política `winhttp_offline`. **Risco médio** — fecha fuga de telemetria.

### P1 — consistência de sistema

6. **Módulos ausentes no resolvedor**: `xstore`, `xgamesave`, `speech`, `xgameui` em
   `modules/xgameruntime.rs:100-108`; portar de `victory-hle/src/modules/`. **Risco médio-alto**.
7. **Display completo**: objetos shell/URI/capture de `victory-hle/src/modules/display.rs:69-137`
   em `modules/xdisplay.rs` (hoje só HDR). **Risco baixo-médio**.
8. **Boot periférico**: `LateDlls` (`apps/victory/src/run.rs:194-316`), `wait_for_previous`,
   manifest/title-id, Agility SDK 1.620.2 e seletor Series S|X (`victory-gpu/src/console.rs:8-35`).
   **Risco alto** em `LateDlls` (loader lock/reentrância); baixo nos demais.

### P2 — para renderizar de fato

9. **PSO lazy + cache por conteúdo** em `xport-gpu/src/d3d12.rs` e
   `xport-hle/src/modules/gpu_replay.rs`: `id+versão+input`, fora do repo, journal com identity
   check (`victory-pipeline/src/cache.rs:13-38`). O título cria ~60.000 PSOs em 20 s.
10. **Staging exato + volatilidade de descritores** em `xport-gpu/src/texture_upload.rs`,
    `d3d12_descriptor.rs`: snapshot+hash por geração, arm-before-read, zero nunca é fallback,
    refill no submit sem fence (`xbox-translator-notes/docs/35-exact-path-design-spec.md`).
11. **Shader waves/estruturação com recusa explícita** em `xport-shader/src/lib.rs`, inspirado em
    `victory-shader/src/{ir,structure,resource,emit}`. Manter erros por opcode.

### P3 — disciplina

12. **Harness A/B** em `apps/xport-cli/src/main.rs` + `xport-report`: duas passes por braço,
    capturas fora do headline, separar classes de frame. Default-off.
13. **Proveniência**: uma linha por arquivo portado em `THIRD_PARTY.md`, com licença de origem e
    indicando código autoral (GPL), doc (CC BY 4.0) ou terceiro (MIT/LGPL).

## 3. O que NÃO portar

- **Dados de jogo**: `.rpf`/`appdata0.rpf`, exe, shaders, texturas, áudio/BIK/Opus, dumps, DLLs,
  SCID/TitleId reais, tabelas derivadas.
- **Tabelas binárias/terceiros**: `bcpack-tables.bin`, tabelas XMA/FFmpeg, AMD/LLVM/Mesa — a
  policy do XPORT só excepciona `LICENSE` e `rdna2_spec.rs`
  (`xtask/src/policy-exceptions.txt:4-7`). A exceção BCPack do V5 **não se aplica**.
- **Tentativas descartadas** (`docs/xbox-translator-notes/docs/20-tried-and-ruled-out.md`):
  placeholder/fill/clamp de mip, cópia adiada, write-watch como detector primário, anel fixo de
  traps, batch ilimitado, log sob lock, `size 0 = all`, guard de descritores em record-time,
  hold de 60 s em device loss, instrumentação como default.
- **Texto das translator notes** sem crédito CC BY 4.0.

## 4. Estado

Item 2 (fila newer + relógio de frame) implementado e validado no Lote 87. O crash
`+0x2484133` desapareceu; `scarlett-exec86` chegou a `+0x501f808`, que o disasm identifica como
uma chamada virtual sobre `[fonte+0x58]` com o objeto nulo. `0x141647940` zera os globals de
fonte quando o asset não é carregado. Portanto o item 4 (XPackage/VFS) passa à frente do item 1:
o bloqueio atual é caminho/asset, não tabela de vtable. `STATUS.md` registra o detalhamento.

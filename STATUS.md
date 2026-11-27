# Status do XPORT

Atualizado em 2026-11-26.

## Lote 125 — em andamento: trace com queue/fence corretos e mapeamento de RVA
- Corrigida recursão acidental em `queue_base_vtable`/`fence_base_vtable` (`inner` agora é a única fonte do `OnceLock`). `d3d12_vtable_trace::call` passou a escolher a tabela base por `diag_kind`, preservando `PresentX`/`Signal` de queue e `Signal` de fence com trace instalado. `XPORT_LOG_CL_CALLS` sozinho agora também instala o trace (antes não instalava).
- Novo diagnóstico opt-in `XPORT_LOG_CL_SHAPE` no `Close`: `scarlett-tracefix-20261126-194302` (60 s) mostrou 8 listas com `write=disp=end=room` e `changed=false` — o título não avança os inline writers, então não há records inline para decodificar.
- A mesma captura teve `presents=0`; distribuição de slots segue sem nenhum gráfico (só `#9`, `#131`/`#132` em command list, 585 `Close`). Captura longa de 180 s em curso para decidir se o present só aparece depois dos ~1000 `CreateDerivedPipelineStateX`.
- `scarlett-exit-20261126-193821` (`XPORT_LOG_EXIT`/`XPORT_LOG_UNHANDLED`, opt-in): 735 `int3` em `+0x56abfa5` e 100 `0xc0000005` em `+0x161b994` (leitura de `0x8`) intercalados; sem opt-ins a execução segue limpa. Mapeamento confirmado: os RVAs dos logs são os RVAs originais do PE (`address - HMODULE`); meu script de desmontagem estava com o cálculo de seções errado. `+0x56abfa5` é `int3` real e `+0x161b994` é `mov r12,[rax+8]` com `rax=0` — as duas falhas são reais, não artefato de RVA.
- `scarlett-tracelong-20261126-194426` (180 s): `presents=0`, 7154 chamadas de `#57 device`, 585 `Close`, nenhum slot gráfico. `close shape` sempre `changed=false`.
- `scarlett-notrace-20261126-194744` (sem trace): terminou sozinho com `status=executed`, **`presents=0`**, e voltou o `0xc0000005` em `+0x1f76f88` (escrita em `0x237cbf15010`; `allocation=0x237cbd30000 region=0x1b000`). O endereço ficou fora da janela registrada pelo lazy commit do Lote 123; não é artefato do trace. Diagnóstico com `XPORT_LOG_XMEM_ALLOC=1` em curso.
- `scarlett-dsbytes-20261126-193720`: 2872 heads inflados, nenhum `DXBC`/`9FPR` (vários `ffffffff` e padrões de floats/índices); não prova decoder RPF ausente. Nada alterado ainda.
- Correção diagnóstica: `d3d12_vtable_trace` agora preserva `GetLastError` ao redor do log e da escolha da tabela, sem alterar o valor publicado pelo handler real.
- **Causa raiz do AV reativo identificada e corrigida:** `commit_lazy_reservation` calculava `region_end = AllocationBase + RegionSize`, mas `RegionSize` é medido a partir de `BaseAddress`. Na reserva de `0x1b00000` (`XMemVirtualAlloc #11`), o endereço `+0x1e5010` estava dentro da janela registrada, porém o cálculo errado devolvia `len=0` e o commit nunca acontecia. Agora usa `BaseAddress + RegionSize` e também reconhece sub-regiões pelo `AllocationBase` registrado. `cargo test -p xport-loader --lib` passou (6 testes); falta recompilar o CLI e repetir a execução real.
- Validação completa passou: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus`, `cargo build -p xport-cli`.
- **`scarlett-lazyfix-20261126-195441` (150 s, só `XPORT_LOG_PRESENT_PLANES=1`): 4 presents, tokens 1–4 alternando os dois back buffers, `status=executed`, `imports_missing=0`, 4 linhas de log — zero exceções e zero recuperações.** A correção do `region_end` resolveu o AV intermitente que fazia o título encerrar sem apresentar.
- Em curso: amostragem de pixels no present (`XPORT_LOG_PRESENT_PIXELS=1`) e nova captura de slots com trace. Nada commitado; nenhum dado de título copiado ao repo.

## Lote 124 — em andamento: calls de método na command list
- Objetivo: descobrir por que `cl_calls=0`/`rt0_binds=0` apesar de `presents=4`, e ligar `OMSetRenderTargets`/draws ao backing do back buffer sem decodificar stream gráfico.
- Mapas autorizados confirmaram: XSEmu não usa hooks por RVA para gráficos; ele embrulha uma command list PC real e classifica chamador pelo endereço de retorno. XPORT já tem o formato console (`0x3000`, `write==end==base+0x1000`) mas não tem backing PC, e o HLE não liga vtables ao `xport-gpu`.
- `scarlett-allslots-now-20261126-192219`: 500 linhas de trace; slots observados em command list foram apenas `#9` (`Close`, com registradores sujos), `#131`/`#132` (`FlushCacheRangeX`). Nenhum `#12`/`#50`/`#139`/`#146`.
- `scarlett-frameevents-20261126-192655`: presents 1–4 e frame tokens até `0x1a`; o loop avança, mas sem pixels.
- `scarlett-pixels2-20261126-192927`: backing dos dois recursos apresentados está zerado (`nz=0/64` em `+0`, `+0x1000`, `+0x10000`, `+0x100000`).
- Novo diagnóstico opt-in `XPORT_LOG_DISPLAY_ARENA=1` em `d3d12_null.rs`: `scarlett-arena-20261126-193048` mostrou as duas arenas de display do título (`0x3e10000` e `0x1000`) também zeradas; aliasá-las ao recurso não resolveria sozinho.
- `scarlett-afterpresent-20261126-193158`: 504 linhas após o primeiro present, **zero chamadas de command list**. Só `pipeline_state #1/AddRef` (137×) e `device #55 CreateDerivedPipelineStateX` (274×) — o título está compilando/construindo PSOs, sem draws.
- `scarlett-long-20261126-193353` (600 s): processo terminou sozinho com `status=executed`, `present #1..#4`, sem calls gráficas. O filtro `XPORT_LOG_CL_CALLS` sozinho não instala a vtable de trace; é necessário `XPORT_LOG_ALL_SLOTS=1` junto.
- Pesquisa autorizada confirmou: D3D12.X inline writers entram na lista quando `write < end`; o XSEmu garante o fallback para método explícito deixando `+0x18=+0x28=+0x30=base+0x1000` — o XPORT já faz isso. Não há registro de stream gráfico separado; draws do título chegam por slots de método, não por records inline interpretados.
- Pesquisa RPF/DStorage: o runtime do XSEmu também não parseia RPF; a suspeita de bytes crus explicaria DXBC inválido, mas precisa confirmação por `XPORT_LOG_DS_BYTES`. XPORT não tem decoder RPF; BCPack rejeitado explicitamente.
- Diagnósticos adicionados (opt-in): `XPORT_LOG_DISPLAY_ARENA` registra as arenas e `XPORT_LOG_PRESENT_PIXELS` agora as amostra no momento do present; `XPORT_TRACE_CAP` permite elevar o limite do trace.
- Em curso: execução longa com `XPORT_LOG_ALL_SLOTS=1 XPORT_LOG_CL_CALLS=1 XPORT_TRACE_CAP=20000` para procurar slots gráficos além das primeiras 500 chamadas. Nada commitado; nenhum dado de título copiado ao repo.

## Lote 123 — Fase 5: reserva XMem preguiçosa e validação limpa (concluído)
- O `0xc0000005` intermitente em `+0x1f76f88` era a inicialização do alocador interno do título (`+0x1f76f20` → `+0x156c240`) escrevendo em uma reserva host `MEM_RESERVE` nunca comitada. Não era ponteiro forjado nem byte de imagem patchado.
- Shim autoral: `xport-loader::register_lazy_range()` registra os intervalos devolvidos por `XMemVirtualAlloc`; o VEH comita no máximo 64 KiB somente numa escrita inválida cujo endereço está num intervalo registrado e cujo estado host é `MEM_RESERVE`. Endereços fora do registro continuam falhando explicitamente; a imagem do título não é alterada. `XPORT_LOG_XMEM_ALLOC`/`XPORT_LOG_PRESENT_PLANES` continuam opt-in.
- Teste unitário `lazy_reservation_commit_makes_registered_range_writable` reserva uma página host, dispara o helper e verifica a escrita; passou no workspace.
- A assertion `+0x56abfa5` segue intermitente e é recuperada pelo próprio handler do título; é anunciada como `recovered offline allocator assertion` e contada como exceção bruta apenas com `XPORT_REPORT_ALLOC_ASSERT=1`.
- Validação completa: `cargo fmt --all`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus`, `cargo build -p xport-cli`: todos exit 0.
- `scarlett-present-lazy-20261126-191806`: `presents=4`, `exceptions=0`, 1 recuperação declarada. `scarlett-final-lazy-20261126-191935` (150 s, sem opt-ins/rede): `missing=0`, `exceptions=0`, `panics=0`, 1 recuperação declarada, shim de commit não disparou. Quatro execuções limpas anteriores também terminaram sem `0xc0000005`.
- Próximo passo exato: seguir hooks de método (`title_list` → guard → slots) até `OMSetRenderTargets`/RT0 e draws, depois ligar o backing de pixels ao recurso apresentado; sem estender o decoder compute-only para gráficos. Nenhum dado de título foi copiado ao repo; nada commitado.

## Lote 122 — Fase 5: command list console-shaped e validação (concluído)
- Levantamento do XSEmu (subagent) confirmou que o caminho gráfico não decodifica stream: a lista entregue ao título tem formato console com `write == end`, forçando os writers inline a caírem na vtable. O decoder de stream do XSEmu é compute-only (`XCL_CONSOLE_LISTS=1`) e não cobre draws.
- Portada a ideia autoral em `d3d12_null.rs`: `Kind::CommandList` usa objeto de `0x3000` bytes com `+0x18/+0x28/+0x30` apontando para `base+0x1000` (sem espaço), estado interno em `+0x2f00` e ponteiro de volta em `+0x2ff0/+0x2ff8`. `obj()` reconhece o formato; `create_command_list` escreve no estado interno. Corrigidos o overflow de `copy_nonoverlapping` (contagem de elementos, não bytes) e o teste DXGI (formato `28`).
- `XPORT_LOG_CL_SHAPE=1` mostrou 101 submissões com `used=0`, `room_nonzero=0` e o objeto intacto — o formato está correto, mas o título não emitiu calls gráficas observáveis na lista sintética; `XPORT_LOG_CL_CALLS=1` registrou `cl_calls=0`.
- A assertion `0x56abfa5` é intermitente e interna ao alocador de classes de tamanho do próprio título: base limpa teve 2/3 execuções sem ela. `XPORT_TEST_ALLOC_RETRY` (retry com yield) e as rotas alternativas/fake já testadas não são correções e foram removidas. Nenhum byte da imagem foi patchado.
- `crash.rs` agora classifica esse `int3` recuperado pelo próprio handler do título separadamente (`recovered offline allocator assertion`) e mantém `XPORT_REPORT_ALLOC_ASSERT=1` para contabilizar a ocorrência bruta. Não é uma correção do alocador; é uma distinção explícita entre falha não tratada e recuperação interna.
- `scarlett-measure-20261126-185136` (com `XPORT_LOG_PRESENT_PLANES=1 XPORT_REPORT_ALLOC_ASSERT=1`): `presents=4`, tokens 1–4 alternando os dois recursos de back buffer, `exceptions=1` bruta. `scarlett-finalclean1/2` (150 s, sem opt-ins/rede): `missing=0`, `panics=0`, `exceptions=0`; a segunda teve 1 recuperação declarada.
- **A validação final ainda não fechou:** `scarlett-clean-final-20261126-185315` (150 s, sem opt-ins/rede) terminou com `0xc0000005` em `+0x1f76f88`, escrita em página reservada nunca comitada (`0x1834a926010`). Stack: `+0x1f76f88` ← `+0x2478f46` ← `+0x19f4f54` ← `+0x580437e` ... O mesmo RVA apareceu em `scarlett-retry3-3`; é intermitente e independente da assertion.
- Desmontagem: `+0x1f76f20` chama o alocador de classes `+0x156c240` (bump sobre uma região com limites em `[rcx+8]/[rcx+0x10]`) e escreve dwords sequenciais no ponteiro devolvido. `XPORT_LOG_XMEM_ALLOC=1` (novo, opt-in) mostrou 17 reservas; a suspeita é uma reserva console (`MEM_GRAPHICS`/`MEM_RESERVE`) mapeada como `MEM_RESERVE` host sem commit. Investigação em andamento; nenhuma correção aplicada ainda.
- Validação de código passou: `cargo fmt --all`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo build -p xport-cli`.
- Desfecho: o `0xc0000005` e a validação foram tratados no Lote 123 (reserva XMem preguiçosa). O próximo passo atual é seguir hooks de método até `OMSetRenderTargets`/draws e ligar o backing ao back buffer, sem estender o decoder compute-only para gráficos. Nenhum dado de título foi copiado ao repo; nada commitado.

## Lote 121 — Fase 5: mapeamento RTV→recurso e relatório de shutdown (concluído)
- `crash.rs` ganhou `stop_reporting()`, chamada em `main.rs` após `main.join()`: exceções de primeira chance de DLLs do sistema no teardown do host deixam de ser atribuídas ao título.
- `XPORT_LOG_RTV_CREATE=1` (opt-in) mostrou que os dois recursos apresentados alternados são exatamente os RTVs criados pelos slots 19/20; `note_rtv` agora recebe o slot de origem.
- `XPORT_LOG_PRESENT_PIXELS=1` (opt-in) amostra 64 bytes em quatro offsets do recurso; o valor lido começa com uma vtable de host, ou seja, o endereço amostrado é a estrutura `Obj`, não o backing de pixels.
- `scarlett-pixels-20261126-173422`: `exceptions=0`, `presents=4`, tokens 1–4 alternando `0x1b06cfc83e0`/`0x1b06cfc8360`.
- Validação parcial passou: `cargo fmt --all`, `cargo test -p xport-hle --lib -q`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo build -p xport-cli`.
- Próximo passo exato: expor o backing `(host, len)` do `Obj::Resource` e amostrar pixels a partir dele; depois repetir validação completa e execução real limpa.

## Lote 120 — Fase 5: reuso autorizado do XSEmu, planes de `PresentX` (26/11/2026)
- Usuário autorizou tratar o código autoral do XSEmu como seu e relicenciá-lo para GPL-3.0-or-later no XPORT, preservando atribuição e auditando terceiros antes de portar.
- Survey completo do XSEmu identificou presentation/frame/fence, recompilador RDNA2→DXIL, DirectStorage/BCPack, APU/XMA/Opus, kernel/XGameRuntime e loader/VFS como áreas reutilizáveis. Nenhum dado de título está comitado no XSEmu; fixtures são sintéticos ou vetores públicos.
- Primeiro port efetivo: `crates/xport-hle/src/modules/d3d12_present.rs` implementa o layout das planes de `PresentX` (stride `0x48`, token @0, contagem @8, lista @0x10), leitura limitada via `ReadProcessMemory`, contadores e log opt-in `XPORT_LOG_PRESENT_PLANES`. Corpo autoral, sem janela/swapchain e sem gravar bytes do título no repo.
- `queue_present` agora usa o parser para o token e registra o flip; o frame clock permanece inalterado.
- Validação passou: `cargo fmt --all`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus` e `cargo build -p xport-cli`.
- Execução real `scarlett-present-20261126-164004`: `missing=0`, `present #1 planes=1` com `token=1`, `resource_count=1`, primeiro recurso lido; ainda `1` exceção em `0x321ec` e `0` panics. Isso confirma o parser, não a correção da ABI do campo.
- Próximo passo: portar seletivamente o pareamento RTV/RT0 do XSEmu para identificar qual objeto sintético é o back buffer e só então ligar um caminho de apresentação real.

## Lote 119 — Fase 5: rastreamento do escritor de `this+0x127c8` (26/11/2026, interrompido)
- Varredura estática da `.text` (capstone, imagem em cache fora do repo) encontrou apenas escritas de 1 byte em `+0x127c8` em objetos alheios; nenhum `mov` de 32/64 bits no título escreve esse campo. Portanto o valor truncado não vem de um `mov` direto do título.
- O shim passou a exigir uma lista explícita de RVAs de `call` virtual (`VIRTUAL_CALL_SITES`), evitando silenciar exceções não relacionadas; `scarlett-rvagate` confirmou o mesmo shim único em `0x327c5` e a mesma falha em `0x321ec`.
- Watchpoint de 4 bytes em `this+0x127c8`, armado em todas as threads, capturou uma escrita em `scarlett-dxgi` com RIP `0x7ff7d89bbda9` fora da imagem do título (`rva=?`), sugerindo código host/HLE usando a stack do título. A captura é intermitente.
- Adicionados diagnósticos opt-in `XPORT_LOG_MODULES` (bases de módulos coletadas fora do VEH) e `XPORT_LOG_DXGI_MODE` (slot 21). `locate()` e `read_qword()` dentro do VEH causaram `stack overflow` em worker thread e foram removidos do handler.
- Log do slot 21 mostrou `word16_target` em endereços de stack do título (`0x...ff230`), não no campo suspeito; a hipótese de `desc.add(4).write(b as u32)` ser a causa não foi confirmada.
- Estado atual do código: diagnósticos opt-in permanecem, mas atribuição do escritor está pendente. Nenhum dado do título foi gravado no repo; nada commitado.
- Próximo passo: repetir a captura com `XPORT_LOG_MODULES=1` até obter o RIP, calcular o offset no módulo `xport-cli.exe` e identificar a função HLE que escreve 4 bytes no campo; corrigir a ABI e revalidar.

## Lote 118 — Fase 5: chamadas virtuais indisponíveis e pilha (26/11/2026)
- O shim de compatibilidade agora reconhece qualquer `call` indireto próximo (`ff /2`, com ou sem REX, com ou sem disp32) cujo registro contenha a palavra de vtable do objeto. Detecta a instrução no RIP ou logo antes do endereço de retorno; valida o bytes da própria imagem em memória, sem patch.
- Corrigido o empilhamento: ao pular a chamada, o handler emula o `ret` (pop de 8 bytes quando o endereço de retorno está no topo). A versão anterior empurrava a pilha quando não devia, corrompendo frames posteriores.
- `scarlett-callfix`: shim disparou em `0x327c5`; o título avançou para uma nova falha em `0x321ec` por campo corrompido com metade baixa de ponteiro (`rdi = 0x37dc0369`). Nenhum `out_obj` rejeitado (`XPORT_LOG_OUT_OBJ=1`).
- Watchpoint opt-in `XPORT_WATCH_WRAPPER_FIELD=1` + `XPORT_BREAK_RVAS=32147`: armado em `this+0x127c8` sem gravar nada; `scarlett-fieldwatch` terminou com `exceptions=0`, `panics=0`, mas ainda `presents=1`.
- Execução final limpa `scarlett-final-20261126-155937`: shim disparou uma vez, `exceptions=1` em `0x321ec` com ponteiro truncado, `panics=0`. Portanto o shim é determinístico e a corrupção do campo é o próximo bloqueio.
- Watchpoint de 4 bytes armado em todas as threads não capturou o writer — a escrita ocorre antes da entrada na função `0x32130` ou fora da janela instrumentada. Varredura da imagem em memória por escrita em `+0x127c8` em andamento.
- Próximo passo exato: identificar o writer de `this+0x127c8`, corrigir a ABI que trunca o ponteiro para 32 bits e só então reavaliar novos presents.

## Lote 117 — Fase 5: chamada de helper inutilizável (26/11/2026)
- Shim pontual para o helper em `0x332e1`, cujo alvo era um buffer de texto; retoma a instrução seguinte do próprio título. Não altera bytes da imagem.
- `scarlett-shim6`: shim em `0x327c5` disparou e a exceção seguinte passou para `0x333fb`; a causa raiz era a pilha desalinhada do Lote 118.

## Lote 116 — Fase 5: generalização do shim de slot vazio (26/11/2026)
- Substituída a lista fixa de RVAs por detecção da própria instrução `call [rax+disp32]` e validação de que `rax` é a palavra de vtable de `rcx`; cobre `0x327c5`, `0x32960` e sites posteriores sem depender de valores de slot que mudam entre execuções.
- `scarlett-shim5` mostrou que o par `rcx==r14` não é estável entre threads; removido.

## Lote 115 — Fase 5: shim do slot 0x110 e guard anti-UB (26/11/2026)
- Shim opt-in por exceção: objeto gráfico do título com slot `0x110` inutilizável (`0`, `-1` ou página de dados) retoma em `base+0x327cb`, com log único e nenhum byte da imagem modificado.
- `d3d12_va_map.rs`: `copy()` falha fechada para tamanho zero/ponteiro nulo, eliminando o panic de `unsafe precondition` em `scarlett-vtbl2`.
- Validação limpa: fmt, clippy `-D warnings`, testes workspace (`--skip llvm_gfx10_corpus`), build do CLI; `scarlett-shim3` confirmou o primeiro shim.

## Lote 114 — Fase 4: writes host não destravam o loop (26/11/2026)
- Reexecução de 150 s com `XPORT_LIST_APPLY_WRITES=1 XPORT_LIST_TRUST_HOST_POINTERS=1`: os dois writes antes recusados agora chegaram a células host confirmadas (`0x2d041a26aa8`/`aa0` ← 0), mais um write em heap de descritor; `write_misses=0`, 0 exceções.
- Continuou com exatamente 1 `frame wait`/1 `present`. O watchpoint opt-in foi armado em 42 threads e a linha monitorada (`0x2c43d6f7640`) segue sem writer CPU. O processo terminou sozinho em ~59 s (watchdog do próprio título), antes dos 150 s. Portanto esses writes não são a publicação esperada pelo spin.
- Próximo passo exato: instrumentar `xuser`/`xgame_api`/`xsystem_api` com `RUST_LOG=trace` (rede continue bloqueada) para verificar se o bloqueio após o present é de sessão/conta/input, não de fence.

## Lote 113 — Fase 4: host cell validada para writes bottom-of-pipe (26/11/2026)
- Execução real de 150 s com `XPORT_LIST_APPLY_WRITES=1`: `unimpl24=0`, 11 cópias `#24`, 48 registros VA, 1 write mapeado com sucesso, 2 writes recusados (`0x26057f76e20/e28`), 0 exceções e ainda 1 present/1 frame wait. Os VAs recusados são endereços host já comitados do próprio título, não VAs sintéticos.
- Seguindo a regra autorizada do fork (`cpu_writes.rs`: `VirtualQuery`, `MEM_COMMIT`, `PAGE_READWRITE`/`PAGE_EXECUTE_READWRITE`, sem `PAGE_GUARD`, alinhamento e extensão), `d3d12_va_map` ganhou `write_any`/`read_any`. O caminho host é opt-in separado `XPORT_LIST_TRUST_HOST_POINTERS=1`; nunca aloca, nunca inventa endereço e continua desligado por padrão.
- Validação completa: fmt `--check`, clippy `-D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus` (125 no xport-hle + demais crates) e build do CLI: exit 0.
- Próximo passo exato: reexecutar 150 s com ambos os opt-ins e `XPORT_LOG_MONITOR=1`/watchpoint para verificar se os writes publicam a linha esperada e se aparecem novos presents.

## Lote 112 — Fase 4: VA→host fail-closed e device #24 (26/11/2026)
- Novo `d3d12_va_map.rs` autoral: registra apenas intervalos que o XPORT possui (recursos sintéticos e heaps de descritor), resolve offsets com checagem de limites/overflow e recusa base zero, sobreposição e backing curto. VA desconhecido nunca é dereferenciado; `XPORT_LOG_VA_MAP=1` é opt-in.
- `d3d12_list_sync.rs` agora aplica writes/waits bottom-of-pipe somente a intervalos registrados quando `XPORT_LIST_APPLY_WRITES=1` (parse explícito de `1`/`true`); o padrão continua no-op. `XPORT_LOG_LIST_OPS=1` mostra VAs e sucesso/falha.
- `new_resource` registra o VA sintético → slice zeroada; `CreateDescriptorHeap` registra o VA reportado ao título → slice opaca do XPORT (sampler = 16 bytes, demais = 32).
- Device `#24` `CopyDescriptorsSimple` implementado de forma autoral: cópia opaca entre intervalos registrados, `E_INVALIDARG` para args nulos e `S_FALSE` para VA não registrado. `XPORT_LOG_DESCRIPTOR_COPY=1` é opt-in. Isto fecha o último slot não implementado observado.
- Corrigida flakiness dos testes de frame: `now_us()` mínimo de 1 µs e lock de teste à prova de poison. Validação completa: fmt `--check`, clippy `-D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus` (124 no xport-hle + demais crates) e build do CLI: exit 0.
- Próximo passo exato: executar o título real por 150 s com diagnóstico opt-in, `XPORT_LIST_APPLY_WRITES=1`, timeout e saídas fora do repo; verificar `unimpl24=0`, writes mapeados e se surgem novos presents.

## Lote 111 — Fase 4: frame events e watchpoint confirmado (26/11/2026)
- `XPORT_WATCH_MONITOR_WRITE=1` usa DR3 com filtro por PID; o self-test opt-in provou que o watchpoint dispara na linha monitorada (`...0002` → `...0003`). Sem o self-test, 120 s de título real não mostraram **nenhuma** gravação CPU na linha: o título espera uma publicação que o XPORT ainda descarta.
- `d3d12_frame.rs` foi reescrito autoralmente seguindo a ABI do fork licenciado: `SetFrameIntervalX` → `S_FALSE`, `ScheduleFrameEventX` → `STARTED_CLOCK`/`S_FALSE`, `WaitFrameEventX` exige evento agendado e emite token só no origin/timeout com flag, `GetFrameStatisticsX` devolve `E_FAIL` até `PresentX` marcar o flip. `frame_completed` agora registra o token lido dos planos sem incrementar contador artificial.
- Validação completa: `fmt --check`, clippy `--workspace --all-targets -- -D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus` (120 no xport-hle + demais crates) e build do CLI: exit 0.
- Próximo passo exato: adicionar registro VA→slice host e tornar as writes bottom-of-pipe opt-in apenas para intervalos registrados, com falha explícita para VA desconhecido.

## Lote 110 — Fase 4: eventos de frame e watchpoint da linha monitorada (26/11/2026)
- Diagnóstico opt-in `XPORT_LOG_FRAME_EVENTS=1` mostrou apenas: `set_interval(33333)`, schedule dos tipos 0 e 2, um `wait` tipo 0 que devolveu token `1`, e um `present token=1`. Depois do present o título **não volta a chamar `WaitFrameEventX`**; ele reagenda o frame e entra em spin em memória própria.
- A semântica de `S_FALSE`/`STARTED_CLOCK` do XCL ainda precisa ser portada; por ora foi acrescentado só o trace autoral e opt-in em `d3d12_frame.rs`, sem gravar nada no repo.
- `XPORT_LOG_MONITOR=1` acrescentou uma leitura limitada da cache line monitorada e do pointee: `0x7ffe800000000002` é um ponteiro para memória do próprio título; nenhum `FenceCell` do XPORT aparece.
- Foi adicionado o watchpoint opt-in `XPORT_WATCH_MONITOR_WRITE=1` em `crash.rs` (DR3, somente gravação, 8 bytes), com filtro por PID do processo. A primeira execução armou 3561 threads de todo o sistema antes da correção; esse resultado não é confiável e será repetido.
- Estado: `cargo fmt --all -- --check` e `cargo clippy --workspace --all-targets -- -D warnings` limpos; build do CLI anterior limpo. Próximo passo exato: validar o watchpoint com self-test opt-in e rodar o título para identificar o RIP que escreve a linha, antes de alterar qualquer semântica de frame/present.

## Lote 109 — Fase 4: waits de fence e DirectStorage (26/11/2026)
- Execução de 600 s sem diagnósticos confirmou: 2 submissões, 1 present, 625/626 criações/fechamentos de lista, 0 exceções. O título não avança para novos frames.
- `RUST_LOG=debug` mostrou 95.557 instruções `monitorx`/`mwaitx` em dois endereços de código, alternando em ciclos curtos (~0,3 s). `XPORT_LOG_MONITOR=1` identificou a cache line monitorada (`rax`) e o valor `0x7ffe800000000002`, idêntico a um valor publicado por `publish_fence`. Incrementar a célula em diagnóstico opt-in não destravou; a intervenção foi removida.
- O `XPORT_LOG_FENCE=1` mostrou que o título nunca chama o slot 12 (`completed_value_address`) nesses fences: o endereço monitorado é memória do próprio título, não um `FenceCell` do XPORT.
- `host_bridge` reportou 4200 enqueues de DirectStorage com `result=0`, lendo diretamente dos `.rpf` como se fossem arquivos raw. Nenhum WARN/ERROR apareceu; o conteúdo entregue não foi validado (nenhum byte de jogo foi copiado ao repo).
- Próximo passo exato: verificar se o bloqueio é de conta/sessão (`xuser`, `xgame_api`, input) com `RUST_LOG=trace` nesses módulos antes de mexer no parse de `.rpf`.

## Lote 108 — Fase 4: instrumentação revela carregamento contínuo (26/11/2026)
- `XPORT_LOG_ALL_SLOTS=1` (vtable de trampolines autoral, opt-in) listou todos os slots chamados: `#57` = 8636, `#9` = 1033, `#73` = 585, `#67`/`#62` = 42 cada, além de `#1/#2/#15/#16/#19` etc. O único não implementado continua `#24`.
- `#57` e `#67` são `DeserializeGraphicsPipelineStateX`/`DeserializeComputePipelineStateX`. Passaram a devolver um `Kind::PipelineState` sintético com S_OK em vez de `E_NOTIMPL` + ponteiro nulo. Nenhum blob é lido ou copiado.
- `XPORT_LOG_AFTER_PRESENT=1` mostrou que, após o primeiro present, o título não está em retry idêntico: ele continua chamando `#55`, `#86` (root signatures com magic `DXBC`=`0x43425844`), `#25`, `#108`, `#133`, `#18`, `#20`, `#61` e `#24` com hashes/endereços diferentes — carregamento de assets.
- O comportamento de 2 submissões/1 present em 150 s permaneceu após o PSO sintético. Próximo passo exato: execução longa (600 s) sem diagnósticos para verificar se novos presents ocorrem quando o streaming termina.

## Lote 107 — Fase 4: varredura de marcadores e instrumentação de slots (26/11/2026)
- `XPORT_DUMP_CL_ARGS=1` leu os buffers de `CreateCommandListX`/`Close`: `create.params` contém `{type, node mask}` e ponteiros de código; `close.b` contém estruturas de 0x80 bytes com IDs sequenciais (`0x88001700`, `0x88001800`, …). Nenhum packet de stream reconhecível nesses buffers.
- `XPORT_SCAN_STREAM=1` varreu a memória comitida com `VirtualQuery`/`ReadProcessMemory`. A faixa ampla `0x8C0F_E000..=0x8C0F_EFFF` gerou 80 falsos positivos (`39 e8 0f 8c` é código x86). Restrita aos IDs documentados `0x8C0F_E002..=0x8C0F_E00A` e `0x4002_C400`, a varredura encontrou 0 ocorrências: o título usa um formato ainda não documentado ou não grava stream com esses marcadores.
- Próximo passo exato: instrumentar a vtable sintética (opt-in `XPORT_LOG_ALL_SLOTS=1`) para listar **todos** os slots chamados entre `CreateCommandListX` e `Close`, incluindo os handlers que não emitem `tracing`, e identificar o método que recebe o stream do título.

## Lote 106 — Fase 4: page pool e localização do stream (26/11/2026)
- Device `#74` identificado no fork licenciado como `RegisterPagePoolX(address, pages, index out)`; `#75` como `UnregisterPagePoolX(index)`. Implementados em módulo autoral `d3d12_page_pool.rs`, com índices estáveis, rejeição de pool nulo/vazio e falha explícita em dupla remoção.
- O slot 74 é compartilhado entre device e command list na vtable sintética; agora o despacho usa `Kind`, preservando `Wait64BitValueX` na command list e registrando o pool no device.
- Execução real: nenhum `unimplemented slot` restante além do device `#24` (`CopyDescriptorsSimple`); 2 submissões e 1 present, sem exceções. O título continua em loop de criação de listas após o segundo frame, sem novas chamadas D3D12 relevantes.
- `XPORT_DUMP_OBJ=1` (com `XPORT_CAPTURE_CL=1`) lê 0x1000 bytes do objeto submetido e imprime só palavras não-zero no stderr do host: não há packets do stream no Obj sintético. Os buffers passados em `CreateCommandListX`/`Close` ainda não foram decodificados.
- Validação final: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -q -- --skip llvm_gfx10_corpus` e `cargo build -p xport-cli`: exit 0. Execução limpa de 150 s sem variáveis de diagnóstico: 0 exceções, apenas o shim de watchdog já documentado; nenhum dado do título gravado no repo.
- Próximo passo exato: estender o diagnóstico opt-in aos buffers de `CreateCommandListX`/`Close` (somente leitura via `ReadProcessMemory`, saída no stderr fora do repo) e identificar o cabeçalho do stream antes de escrever o decoder da Fase 5.

## Lote 105 — Fase 4: slots newer-layout da command list (26/11/2026)
- Implementados autoralmente os handlers de command list `#139` Reset, `#32` SetDescriptorHeaps, `#146` UAV barrier, `#52` Write32BitValueBottomOfPipeX, `#72` Write64BitValueBottomOfPipeX e `#74` Wait64BitValueX, com despacho por `Kind::CommandList` para não alterar device/queue/fence.
- Novo módulo `d3d12_list_sync.rs` registra as operações por lista, marca `Close` e as conclui na submissão. `XPORT_LOG_LIST_OPS=1` é opt-in e escreve só no stderr do host, fora do repo.
- Diagnóstico real: o título grava 4 operações (3 writes + 1 wait cmp=7) na lista submetida; o wait já estava satisfeito. Os VAs das operações são offsets em memória unificada que o dispositivo offline não mapeia: device `#24` é `CopyDescriptorsSimple` nesses mesmos VAs.
- Escrever nesses VAs como endereços host causou `0xc0000005` no título (`base + offset`, base variável por execução). A aplicação das escritas passou a ser opt-in (`XPORT_LIST_APPLY_WRITES=1`); o padrão é seguro e não toca memória do título. Nenhum bytes do título foi copiado ao repo.
- Validação parcial: `cargo fmt --all`, `clippy -p xport-hle --all-targets -- -D warnings` e `cargo test -p xport-hle --lib` passaram (116 aprovados, 3 ignorados). Falta a validação completa do workspace após a última correção.
- Próximo passo exato: ler o stream de comandos dentro do próprio `Obj` da command list (offsets `+0x18/+0x28/+0x30/+0x344` documentados no fork licenciado) e decodificar os packets em memória, sem gravá-los no repo.

## Lote 104 — Fase 4 em reconhecimento (26/11/2026)
- Diagnóstico opt-in novo: `XPORT_CAPTURE_CL=1` em `queue_execute_x` imprime (via `ReadProcessMemory`, stderr do host, fora do repo) os ponteiros de `lists[]`, as palavras do objeto e do allocator e o token, no máximo nas duas primeiras submissões. Desligado por padrão.
- Execuções reais com `RUST_LOG=xport_hle::modules::d3d12_null=trace`, timeout e saídas em `%LOCALAPPDATA%\Temp\opencode`: o título leva longa inicialização (625 `CreateCommandListX`, 626 `Close`, centenas de `GetResourceAllocationInfo`) antes da primeira submissão.
- Primeira submissão capturada: `count=1`, `token=1`; segunda `token=0x300000001`; depois `queue_present(plane_count=1)`. O objeto submetido é o `Obj` synthetic e o allocator não contém packets reconhecíveis.
- O array de planos (terceiro argumento) foi capturado: começa `[1, 0, 1, 0, ptr_host 0x7ffd..., ptr_console 0x0d..., ptr_console 0x0d...]`. `queue_present` passou a nomear os parâmetros conforme a ABI observada (`plane_count`, `planes`); nenhum desses endereços foi lido além da amostra ou copiado ao repo.
- O título não chama `Dispatch`/`SetPipelineState` na command list: grava o stream em memória própria, fora do objeto. Sem formato validado, o decoder da Fase 5 não pode ser implementado. Imediatamente antes da primeira submissão há 61 chamadas ao device #57 (não implementado) — próximo slot a nomear, sem patch de bytes.
- Validação após a mudança: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings` e `cargo test --workspace -- --skip llvm_gfx10_corpus`: exit 0. Nenhum diagnóstico/rede na validação padrão; nenhum dado do título copiado ao repo.
- Próximo passo exato: nomear o device #57 e localizar o buffer de comandos que o título grava fora do `Obj` (somente em memória), antes de propor o decoder da Fase 5.

## Lote 103 — Fase 3 concluída (26/11/2026)
- Fase 2 concluída e validada no Lote 102: MIMG → HLSL com T#/S# estáticos/dinâmicos e PS sintético compilando em DXC.
- Implementado na Fase 3: root signature gráfica (`u0..u5` space 0, SRV/sampler tables space 1), PSO gráfico com rasterizador/sample desc/RTV format, heaps RTV/DSV/SRV/sampler, upload de textura sintética, viewport/scissor, `OMSetRenderTargets`, `DrawInstanced`, readback e `IASetVertexBuffers` validado. VS fullscreen autoral usa `SV_VertexID`, portanto o teste não precisa de input layout.
- Critério satisfeito: `translated_textured_triangle_readback_on_real_hardware` desenhou o triângulo num RT offscreen 4×4 e leu `[32, 96, 160, 255]` nos 16 pixels; PS veio do container sintético com `image_sample` + `exp mrt0`. Sem HWND, sem jogo e sem assets no repo.
- Correções encontradas no hardware: `SampleDesc.Count` precisa ser 1 no PSO; o SRV precisa de component mapping identidade (o mapping zerado replicava o canal R).
- Validação: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -- --skip llvm_gfx10_corpus`, DXC opt-in e os três testes `d3d12_hw` opt-in: exit 0. Nenhum diagnóstico/rede na validação padrão.
- Próximo passo exato: reavaliar a Fase 4 (captura em memória de HWND, `lists[]`/token, planos de present, descritores e recursos), sem copiar nada do título ao repo.

## Lote 102 — Fase 2 concluída (26/11/2026)
- Usuário pediu continuação autônoma até o limite. Nenhum dado do título foi lido ou copiado; fixtures apenas autorais.
- Implementado: ABI de texturas (`GraphicsTextureDimension`, `GraphicsTextureBinding`, `static_texture_base`), `constant_resources` para T# constants, `mimg` com Sample/Bias/Level/Grad/Cmp/Load, heaps dinâmicos T#/S#, e PS sintético com `image_sample` + `exp mrt0`.
- Evidência: `synthetic_pixel_container_samples_static_texture` passa; DXC opt-in compila VS/PS/PS texturizado; fmt/clippy/testes do workspace exit 0.
- Limitação: execução em hardware do PS texturizado depende da assinatura/PSO da Fase 3; o HLSL usa texturas float e não cobre gathers/atomics/MSAA/16-bit.

## Lote 101 — Fase 1 concluída (26/11/2026)
- Usuário declarou autorização para processar em memória dados entregues pelo título ao HLE. Registrado em `docs/DECISIONS.md`; Fase 0 marcada como resolvida. `ASSET_POLICY.md` permanece: nada do título copiado ao repo.
- Implementado: `translate_xbox_graphics` e `compile_xbox_graphics` em `xport-shader/src/lib.rs` (VS/PS via `interface()`/`translate()` internos); `GraphicsStage` com perfis DXC `vs_6_6`/`ps_6_6` em `dxc.rs`; containers VS/PS sintéticos com ISG1/OSG1 autorais em `gpu/xbox.rs`; teste opt-in `tests/dxc_graphics.rs`.
- Evidência: testes VS/PS e rejeição de container CS passam; teste opt-in compilou VS e PS para DXIL com DXC do Windows Kits 10.0.26100.0; `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings` e `cargo test --workspace -- --skip llvm_gfx10_corpus` exit 0. Nenhum blob/dado de jogo lido ou copiado.
- Limitação restante da Fase 1: o HLSL gráfico ainda usa a ABI UAV `u0/u1/u2` do compute; não há PSO gráfico, RTV/DSV nem draw (Fase 3). MIMG ainda rejeitada (Fase 2).
- Próximo passo exato: Fase 2 — implementar lowering de `MIMG` para textura/sampler HLSL reaproveitando `constant_samplers`, `sampler_ref`, format table e os mapeadores T#/S# de `xport-gpu`; cobrir com pixel shader sintético com amostragem e validar fmt/clippy/testes.

## Lote 100 — retomada (26/11/2026, aguardando escopo)
- Já existe: título abre e roda 300 s sem crash, binding 172/415/0/0, shims por hash, fmt/clippy/testes/build limpos (Lotes 97/98). Limitação de renderização documentada no Lote 99.
- Falta: decidir o próximo objetivo. Candidatas: (a) plano de longo prazo do pipeline gráfico D3D12.X, (b) stubs restantes (slots 24/74, DStorage, BCPACK/page-mode), (c) entrada/controles no título.
- Escopo escolhido pelo usuário: **avaliar viabilidade do pipeline gráfico** — mapear o necessário para renderizar (command streams de draw, shaders graphics, swapchain host) e produzir plano concreto, sem copiar nada do título, sem implementar ainda.
- Inventário do HLE D3D12.X concluído: objetos/vtables, pontos exatos onde draws e presents são descartados, recursos zerados, HWND não retido e diagnósticos disponíveis. `docs/GRAPHICS_ROADMAP.md` criado com o resumo executivo, a seção 1 (HLE) e as restrições legais.
- `docs/GRAPHICS_ROADMAP.md` **concluído**: 7 seções — estado do HLE, tradução de shaders, backend GPU, 6 fases com critérios de saída, riscos e conclusão. Descoberta relevante: boa parte do VS/PS já existe em `gpu::recomp`, mas é privada; MIMG é rejeitada; não há PSO gráfico, RTV/DSV, rasterizador nem swapchain em `xport-gpu`.
- Conclusão da avaliação: viável em princípio, inviável como incremento curto. Fase 0 exige decisão legal formal sobre processar blobs do título em memória; sem ela, o plano para na fase 3 (fixtures sintéticos).
- Próximo passo exato: submeter o roadmap ao usuário e aguardar aprovação. **Nenhuma implementação foi iniciada** e nenhum dado de jogo foi lido ou copiado.

## Estado final (26/11/2026)
- Escolha do usuário: **documentar e encerrar**. A limitação de renderização está registrada de forma explícita em `README.md`, `STATUS.md` (Lote 99) e `THIRD_PARTY.md`.
- Entregável real: título verificado abre e roda sem crash por 300 s, com binding completo e sem diagnósticos/rede; **sem renderização de imagem**. Não é compatibilidade de jogo.
- Validação estática final: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -- --skip llvm_gfx10_corpus` e build do CLI — todos com exit 0 (Lote 97).
- Limpeza: minidumps locais removidos; preview HTML temporário removido; nada do título copiado para o repo; nenhum commit realizado (`git` indisponível no PATH).
- Não há processo `xport-cli` em execução.

## Lote 99 — tela branca no título (26/11/2026, diagnóstico concluído)
- O usuário informou que, apesar da janela `Americas`, threads e diffs de pixels, o conteúdo dentro do executável é **todo branco**. Os diffs anteriores provaram atividade de janela, não renderização de jogo.
- Causa raiz confirmada por código, não por suspeita:
  - `queue_present` (`d3d12_null.rs:1383`) apenas chama `frame_completed()` e retorna 0; nenhum HWND/swapchain é tocado.
  - `queue_execute_x` (`d3d12_null.rs:1409`) só valida argumentos; nenhuma command list é executada.
  - `new_resource` (`d3d12_null.rs:756`) aloca `vec![0u8; size]` — todo recurso, incluindo o display target de `slot127`, é memória zerada.
  - O display target vem do próprio allocator do título (`xmem::runtime_alloc`), sem pixels reais.
  - O título usa `d3d12_xs.dll` (console), não D3D12 PC; não existe `D3D12CreateDevice`/`CreateDXGIFactory` ligado ao host.
  - `xport-shader` traduz **apenas compute** (`translate_xbox_compute`); não há pipeline gráfico (VS/PS), nem decoder de command stream de draw.
- Conclusão honesta: a janela abre, a lógica do título roda e não há crash, mas **nenhum pixel de jogo é produzido**. Tornar utilizável com imagem exige implementar um emulator/ tradutor gráfico D3D12.X completo (draw + shaders graphics), o que está fora do escopo atual e esbarraria na regra de não copiar shaders/dados do jogo.
- Próximo passo: decidir com o usuário entre (a) manter o estado atual como "abre e roda sem crash" e documentar a limitação, ou (b) avaliar um plano de longo prazo para o pipeline gráfico, sem copiar nada do título.

## Lote 98 — sessão longa e interação (26/11/2026, concluído)
- Execução limpa de 300 s (`scarlett-long`, sem diagnósticos/rede): janela `Americas` permaneceu aberta, 37 threads, CPU 279,9 s, nenhuma saída; apenas os dois logs de shim, nenhum erro novo.
- Quadros em 25/60/65/150/290 s: diffs de 45,6% / 44,2% / 57,5% / 83,1% das amostras, confirmando animação contínua durante toda a sessão.
- Teste de entrada com foco forçado por `AttachThreadInput`: `FG_SAME/ACTIVE_SAME/FOCUS_SAME=True`; Enter, Espaço e duas setas para cima foram entregues à janela. O título permaneceu `Americas`.
- Diferença após entrada (91,8% em 6 s) foi próxima do controle sem entrada (83,1% em 6 s), então a aceitação da entrada pelo jogo **não foi comprovada**; apenas a entrega ao HWND foi confirmada.
- Nenhum minidump foi gerado nestas execuções; nada foi copiado para o repo e a rede esteve bloqueada.
- Quadro `long-t290.png` aberto para inspeção visual num HTML temporário fora do repo (`xport-preview-t290.html`, na raiz do workspace), em vez de copiar a imagem para dentro de `XPORT/`.
- A inspeção pelo usuário mostrou que a janela está branca; os diffs anteriores eram ruído de janela, não jogo renderizado (ver Lote 99).

## Lote 97 — execução limpa utilizável (25/11/2026, concluído)
- Os dois shims passaram a ser compatibilidade padrão, ativada apenas para o hash verificado `b99f876b74d4a77f...` via `enable_scarlett_compat(base)`; não dependem de `XPORT_SKIP_*`, não alteram bytes do título e cada um loga uma vez.
- Validação: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace -- --skip llvm_gfx10_corpus` e build do CLI: todos com exit 0.
- `scarlett-final-clean` sem nenhuma variável de diagnóstico/rede (só `XPORT_GAME_DIR`): binding `172 hle, 415 host, 0 title, 0 missing`, janela `Americas` aberta, 38 threads ativas após 60 s, sem crash.
- `scarlett-final-clean90` repete a execução limpa por 90 s: 39 threads, ambos os shims logados uma vez e nenhum crash. Duas capturas separadas por 4 s diferem em 11.856/133.375 amostras (~8,9%), provando renderização ativa, não apenas uma janela aberta.
- `scarlett-exec157` com os antigos opt-in rodou por ~16 min com a mesma janela e renderização ativa (diff de pixels entre capturas), confirmando que os shims não são apenas um adiamento de segundos.
- Os encerramentos forçados das execuções acima fizeram o Breakpad gravar 44 minidumps locais (~817 MB) em `%LOCALAPPDATA%\XPORT\cache\temp`; todos foram removidos. Nenhum dump foi copiado para o repo e a rede esteve bloqueada (`winhttp_offline`).
- Próximo passo: observar uma sessão longa sem diagnósticos para confirmar estabilidade/menu, sem copiar nenhum dado do jogo.

## Lote 96 — watchdog diagnosticado e shim opt-in (25/11/2026, em andamento)
- O trap em `+0x2215c35` é o watchdog do próprio título: `r15 == [base+0x71b6338]` (contador de progresso) por 30 iterações, com `call [base+0x71b6348]` logo antes. O callback global é `base+0x4380` (`ret`), então o título se encerra ao fim do timeout.
- Causa observada: a thread executa `Sleep(1000)` e mede o delta; no host o valor excede o limite de 300 ms, sem atualizar o contador. Corrigir o contador de progresso é preferível; como diagnóstico, `XPORT_SKIP_WATCHDOG` retorna pelo próprio epílogo da função (`+0x2215dce`), opt-in e logado, sem patch de bytes.
- `crash.rs` agora lê o `count` do watchdog como `u32`, inclui `context.R12` como `now` e separa `entry0`/`entry1`; `XPORT_DUMP_BP` despeja os arrays e `r15`.
- `scarlett-exec157` com `XPORT_SKIP_NULL_SOURCE=1` e `XPORT_SKIP_WATCHDOG=1` ultrapassou 10 minutos de execução (as anteriores terminavam em segundos); resultado final pendente.
- `THIRD_PARTY.md` registrou a proveniência de `fence_x.rs`, `insn.rs` e `eh.rs`.
- Próximo passo: após `exec157`, identificar o novo bloqueio com logs limpos, validar fmt/clippy/testes/build e executar sem nenhuma variável de diagnóstico.

## Lote 95 — Fence slot 12, waits do console e thread-trap (em andamento)
- Portado o contrato autoral de `xcl/src/hle/d3d12x/fence_x.rs`: `Kind::Fence` agora tem vtable própria (slot 6 = `Signal`), célula `AtomicU64` alinhada a 64 bytes devolvida pelo slot 12 e publicada com `Ordering::Release` em sinais da fila e da CPU. Teste novo cobre ponteiro estável e valores 5→9→11. `Device` continua devolvendo `CommandList` no slot 12.
- `scarlett-exec141` confirmou que `+0x4fb0b03` desapareceu; o título passou a falhar em `+0x4fb0bd7` com `0xc000001d` (`monitorx`, hint de espera do Zen 2). Portado o conceito de `xcl/src/insn.rs`/`eh.rs`: VEH reconhece `0f 01 fa/fb`, avança `RIP` em 3 e faz `spin_loop` só em `mwaitx`, sem patchar bytes do título.
- `scarlett-exec142` passou do wait e chegou a `+0x2215c35`: `mov dword ptr [0],0`, um trap deliberado precedido de `call [globals+0x71b6348]` quando um contador de watchdog expira. O callback global é `0x7ffdd13f4380`; próximo passo é identificá-lo com `locate()` e corrigir a condição que dispara o watchdog, não pular o trap.
- Instrumentação atual é somente `XPORT_SKIP_NULL_SOURCE`, `XPORT_BREAK_RVAS` e `XPORT_DUMP_BP`, todos opt-in. Validação final continua sem nenhuma variável.
- Próximo passo exato: logar `locate(callback)` em `crash.rs`, desmontar o loop em `+0x2215af6..+0x2215c35` com mapeamento PE correto e determinar qual sinal/frame não está sendo publicado.

## Lote 94 — worker threads, `.pdata` e shim de hash (concluído)
- Wrapper de `CreateThread` em `host.rs` (somente com `XPORT_BREAK_RVAS`) arma breakpoints em worker threads; `scarlett-exec125` confirmou os RVAs `2479f40`, `247a089`, `247a098`.
- Portado `RtlAddFunctionTable` de `xcl/src/eh.rs`: `xport-pe::ImageInfo::exception_rva/size`, `crash::register_function_table` chamado em `windows.rs::spawn_main` antes do código do título. `scarlett-exec130` não mostrou unhandled novo.
- Shim opt-in `XPORT_SKIP_NULL_SOURCE` em `crash.rs`: em `+0x247a098` com `Rbx=0`, retorna pelo caminho "no-source" do próprio título (`base+0x247a5d2`, `Rax=0`), documentado como compatibilidade explícita. `scarlett-exec134` deslocou o bloqueio para `+0x4fb0b03`.
- Proveniência de `eh.rs`/`fence_x.rs`/`insn.rs` falta registrar no topo de `THIRD_PARTY.md` (já registrada para ports anteriores).
- Validação parcial: fmt e build do CLI passaram; falta clippy/testes completos após o Lote 95.

## Lote 91 — auditoria do `xcl-nvidia-dev` (em andamento)
- Novo objetivo do usuário: extrair o máximo de código autoral compatível de `C:\Users\User\Downloads\xcl-nvidia-dev` (opensource) para o XPORT.
- Regras mantidas: apenas código autoral licenciado; nunca dados de jogo, `.rpf`, exe, DLLs, dumps, tabelas binárias; proveniência em `THIRD_PARTY.md`; implementações reais ou falha explícita, sem `int3`/sucesso falso; instrumentação opt-in.
- Licença resolvida: o owner confirmou (2026-11-26) que pode relicenciar código autoral do fork `xcl-nvidia-dev` (GPL-2.0-only) para GPL-3.0-or-later; registro feito em `THIRD_PARTY.md`. Nenhum dado/binário do fork foi copiado.
- Inventário concluído e primeira port: `xport-vfs` agora resolve montes embutidos (`C:\work\GAME:\fonts\a.xt`), `G:foo` sem separador e precedência por prefixo mais longo, seguindo `xcl/src/vfs.rs:220-258`. Testes novos em `crates/xport-vfs/src/lib.rs`.
- Segunda port (bloqueio `+0x501f808`): `d3d12_null.rs` passou a implementar slots 19/20/61/67/70/71/86/131/132/133 com out-pointers explícitos e descritores copiados como words opacos; antes caíam em stubs que não escreviam `ppv`. `host_bridge.rs` agora inicializa status array do DirectStorage em `S_OK`, como o XCL.
- Validação parcial: fmt, clippy `-D warnings` em `xport-hle`/`xport-vfs`, 109 testes HLE + integração e build do CLI passaram. Falta executar o título real sem diagnósticos e atualizar o worklog externo.
- Correção adicional: `slot11` de recurso em `d3d12_null.rs` agora devolve o endereço virtual no retorno (contrato `Resource_GetGPUVirtualAddress` do XCL), em vez de escrever em `d` (o título passou `0x1926`, um valor, não ponteiro). Isso removeu o write inválido de `exec105` e o bloqueio fatal `+0x501f808`: em `exec117` o breakpoint nesse RVA mostrou `rcx` válido e `rax=3`.
- Terceira port: `xpackage.rs` lê `MicrosoftGame.Config` em runtime (TitleId/Identity Name/Version/primeiro `Resource Language`), synthesize o package id e o locale, e implementa o slot 27 `XPackageGetIdentifier(pathOrName, size, buffer)`. Nada do manifest foi embutido no repo. Validação: fmt, clippy `-D warnings` (exit 0), 112 testes HLE + integração passaram.
- Próximo passo: execução limpa `scarlett-exec118` sem variáveis de diagnósticos; se o término ainda for `0xc0000005` sem linha `XPORT:` nova, instrumentar a thread que termina (fora da title thread) antes de mexer em VFS/HLE.

## Lote 90 — blockage avançou para carregamento de fonte (em andamento)
- `scarlett-exec100`, sem qualquer diagnóstico ou rede: `+0x4f65847` desapareceu. Agora há vários first-chance `+0x247a098` (`rbx=0`, não fatais) e o término em `+0x501f808` (`read 0x0`), o mesmo bloqueio de fonte/asset do Lote 87.
- Próximo passo: instrumentar o caminho `0x141647940` e os arquivos pedidos, comparar com o VFS, e corrigir o HLE correspondente; não tocar nos first-chance `+0x247a098` sem evidência de que são a causa raiz.

## Lote 89 — ABI dos objetos COM sintéticos (concluído)
- `XPORT_DUMP_RSI` mostrou que `[rsi+0x30]` (o status array criado pelo factory slot 5) começava com um ponteiro de heap e com o vtbl em `+0x10`: `DsStatus`, `DsFile`, `DsQueue` e `XgComputer` não tinham `#[repr(C)]`, então o layout padrão do Rust não garantia o vtbl no primeiro word lido pelo título.
- Correção: `#[repr(C)]` nesses quatro tipos e testes de `offset_of!(..., vtbl) == 0`. Validação: fmt, clippy `-D warnings`, 108 testes HLE aprovados (3 ignorados) + integração e build do CLI. `scarlett-exec100` confirmou a correção; nenhum patch de `int3` foi usado.

## Lote 88 — DirectStorage offline com leitura real (em andamento)
- `host_bridge.rs`: factory slot 4 (`OpenFile(path, iid, out)`) traduz o caminho pelo VFS e devolve um `IDStorageFileX` sintético; fila slots 3/4/6/8 implementam `EnqueueRequest`, `EnqueueStatus`, `Submit` e `Close`. Status slots 3/4 têm entradas reais.
- Leitura de arquivo real via VFS, com zlib via `miniz_oxide`; BCPACK/page-mode falham explicitamente com `E_NOTIMPL`, sem copiar tabelas do jogo.
- A conclusão é síncrona: a leitura acontece no `EnqueueRequest` e o status completa imediatamente. É emulação autoral, não DirectStorage assíncrono; evita o spin do título em `IsComplete`.
- `scarlett-exec91`, sem `RUST_LOG`, rede, dumps ou watchpoints: o acesso inválido `+0x4f65847` desapareceu; o título agora parou em `0x80000003` em `+0x1e6c29f` (`XMemMapPhysicalPages`, `rax=0`, `rsi=0x185c0` = 99.776 páginas).
- Validação: `cargo fmt --all`, clippy `-D warnings` em `xport-hle` e `xport-loader`, 106 testes HLE aprovados (3 ignorados) + integração, build do CLI. Proveniência do V5 registrada em `THIRD_PARTY.md`.
- Próximo passo: investigar o retorno nulo de `XMemMapPhysicalPages` em `xmem/physical.rs` para a reserva `kind=0x20402000`/99.776 páginas, que havia sido contornada no Lote 83.

## Lote 86 — análise integral de `victory-main v5` (em andamento)
- Objetivo: levantar tudo o que o V5 oferece (loader, thunks, D3D12.X/DirectStorage/XMem, GPU, HLE, áudio/vídeo, patches de compatibilidade) e separar o que é reaproveitável no XPORT sem copiar código/assets/binários proprietários.
- Estado inicial: `victory-main v5` foi anexado; ainda não foi mapeado. Execução anterior (`scarlett-exec85`) foi interrompida e não é prioridade neste lote.
- Restrições: usuário confirmou que o fork V5 é autorizado para portar código autoral ao XPORT sob GPL; proveniência obrigatória em `THIRD_PARTY.md`. Continuam proibidos dados de jogo (`.rpf`, exe, shaders, texturas, saves), dumps, DLLs e tabelas binárias do anexo.
- Inventário feito: 224 arquivos, 46 diretórios; crates `victory-core/pe/loader/hle/gpu/shader/formats/game/pipeline`, apps `victory`, docs de translator notes.
- Análise gráfica concluída: V5 usa tabelas por tipo de objeto + `missing_table` (`victory-gpu/src/object.rs:273-359`), nunca uma tabela única de 256 slots como o XPORT; fila "newer" desloca slots >=14 em +2 e confirma `0x100` Signal2X, `0x108` ExecuteX, `0x118` submissão com token de frame (`queue.rs:990-1034`). Isso explica o padrão do crash `+0x2482689` (`call [rax+0x118]`): é token submission, não ExecuteCommandListsX puro.
- DirectStorage V5 confirma factory 4=OpenFile, 5=CreateStatusArray, 8=SetStagingBufferSize, 10=CreateQueue; status 3=IsComplete, 4=GetResult; fila 3=Enqueue, 4=EnqueueStatus, 6=Submit, 8=Close (`dstorage.rs:238-304`). XPORT cobre só parte.
- Análise do loader concluída: o XPORT já replica quase todo o boot do V5 (prepare/load/bind/TLS/thunks/spawn); o que falta é ISA do console. V5 `console_insn_decode.rs:88-160` + `console_insn.rs:106-249` emulam `clzero`/loop, `clwb`, `rdpru`, `rdpid`, `monitorx/mwaitx` e SSE4a (`extrq/insertq`) por decodificação de bytes, não por RVA. Isso substitui `crash.rs::emulate_clzero` + `prepare.rs` NOP e cobre outros builds.
- V5 também tem: `crash.rs` com C++ throw/bytes da instrução/stacks (`victory-loader/src/crash.rs:112-125,398-448,543-659`), `LateDlls` (`apps/victory/src/run.rs:194-316`), manifest/title-id/Agility/seletor Series S|X (`run.rs:52-73`, `agility.rs:17-29`, `victory-gpu/src/console.rs:8-35`), `wait_for_previous`. Ordem de `threads::mapped` vs `tls::redirect` difere em `windows.rs:75-93`.
- Análise de HLE de sistema concluída: prioridade imediata é `xpackage.rs` (ID real do título, progresso completo, recusa de mount de `packroot:/`/`appdata0.rpf`) e guarda MF em `host_bridge.rs` (proxy `file:`/`localhost` de `victory-hle/src/modules/media.rs:41-145`) para não furar `winhttp_offline`. Faltam módulos `xstore`/`xgamesave`/`speech`/`xgameui` no resolvedor `xgameruntime.rs`; display do XPORT só tem HDR, falta shell/URI/capture.
- Análise de docs/formatos concluída: valor do V5 está nas regras/medições, não em artefatos. Não portar `bcpack-tables.bin`, tabelas XMA/AMD/LLVM, placeholder/clamp/fill, write-watch primário, hold de 60 s, instrumentação default. `README.md:35-47`: RT desligado, prólogo 7-9 fps, gargalo na thread de submissão — alvo realista é chegar ao menu/intro estável.
- Plano consolidado escrito em `V5_ADOPTION.md` (P0–P3, 13 itens com arquivo de destino e risco). Nenhum item implementado ainda.
## Lote 87 — P0 #2 concluído: fila newer + relógio de frame
- Implementado em `d3d12_null.rs`: vtable separada para `Kind::CommandQueue` com `Signal 0x80`, `Wait 0x88`, timestamp `0x90`, clock `0x98`, `GetDesc 0xe0`, `PresentX 0x100`, `Signal2X 0x110` e submissão com token `0x118`. Device e outros objetos mantêm a vtable genérica, então slots 16/18 não foram deslocados.
- Novo `d3d12_frame.rs`: device slots 78/79/80/81 com intervalo padrão de 16,667 µs, `S_FALSE` em timeout e tokens consecutivos; sem display do host.
- Validação: `cargo fmt --all`, clippy `-D warnings`, 103 testes HLE + 4 loader + integração e build do CLI passaram. Proveniência do V5 registrada em `THIRD_PARTY.md`.
- `scarlett-exec86` (sem rede/diagnósticos): crash antigo `+0x2484133` desapareceu; o título avançou para `+0x247a098` (primeira-chance, `rbx=0`) e terminou em `+0x501f808` (`rcx=0` em `mov rax,[rcx]`).
- Disasm mostra que `+0x501f808` lê `[fonte+0x58]`; `0x141647940` devolve nulo e zera globals de fonte em `0x141edb443`/`0x141edb592`. O bloqueio é carregamento de fonte/texto (VFS/asset), não D3D12; P0 #1 não o explica.

## Próximos passos (plano P0–P3 em `V5_ADOPTION.md`)
- Prioridade reavaliada: P0 #4 (XPackage/VFS) antes de P0 #1, porque o crash atual depende de fonte/asset não carregado. Depois P0 #1 (tabelas por `Kind`), P0 #3 (ISA do console) e P0 #5 (guarda MF).
- Passo exato: instrumentar apenas em execução diagnóstica temporária os caminhos de arquivo pedidos durante `0x141647940`, comparar com `G:\GTA6.Xbox.v1.0.2.1-CYBERLEEK`, corrigir o VFS/HLE correspondente e reexecutar sem variáveis de diagnóstico.

## Lote 85 em andamento — DirectStorage/D3D12.X/CLZERO (25/11/2026)

- `scarlett-exec81`: `+0x4f65844` lia nulo porque `wrapper+0x30` vinha do factory DirectStorage slot 5 (`CreateStatusArray`), que devolvia 0 sem escrever o out-pointer. Implementado objeto sintético com `IsComplete` no slot 3.
- `scarlett-exec82`: `int3` em `+0xabfe1b` era assertion do título após o slot 108 (`GetResourceAllocationInfo2`) devolver buffer incompleto. Implementada emulação autoral com `{size, alignment}` e `INFO1` por recurso.
- A versão anexada `victory-nvidia-8gb-main` não contém patch literal para estes RVAs; dela vieram apenas os fatos de ABI já usados: factory slot 10 = `CreateQueue`, slot 5 = `CreateStatusArray`, queue/status slot 3, device #108 = `GetResourceAllocationInfo2`. Nenhum código foi copiado.
- `scarlett-exec83`: `0xc000001d` em `+0x4e8aa70` é `clzero` (instrução Zen) não suportada pelo host. Há 10 sites reais no mesmo function (`+0x4e8aa70`..`+0x4e8abb4`); dois outros matches no binário são dados, não código.
- Implementado em `crash.rs`: VEH emula `clzero` apenas nos RVAs registrados pelo CLI para o build `b99f876b74d4a77f...`, zeroando 64 bytes em `RAX`, avançando `RIP` em 3 e falhando explicitamente se a escrita não for possível. Nenhum byte de código foi patchado; `prepare.rs` continua com o patch `clwb` (hint puro).
- Validação passou: `cargo fmt --all`, clippy `-D warnings` nos três crates, 99 testes HLE + 4 loader + integração, build do CLI.
- `scarlett-exec84`: `clzero` ultrapassado; novo `0xc0000005` em `+0x2482689` (`call [rax+0x118]`, `rax=0`, `rcx=r15=0x22211278af0`). O chamador é `+0x248498`; o slot 0x118/8=35 de `ID3D12CommandQueue` é `ExecuteCommandListsX` segundo a referência autorizada. O objeto `r15` não é um `Obj` sintético nosso (que sempre tem vtable válida).
- Próximo passo exato: usar o diagnóstico opt-in `XPORT_DUMP_RSI` para despejar as palavras de `r15`/`r12` no crash, identificar quem criou o objeto com vtable nula e só então corrigir o HLE correspondente; reexecutar sem diagnóstico após a correção.

## Lote 84 em andamento — XGTextureComputer (25/11/2026, 21:40Z)

- `scarlett-exec63/64`: `XMemMapPhysicalPages` passou; `+0x16f71fb` lê `0xffffffffffffffff` porque `XGCreateTextureComputer` devolvia um handle em vez de ponteiro COM. O chamador imediato usa a vtable do objeto.
- `xbox_bridge.rs` passa a devolver um objeto sintético com vtable de 12 slots e emulação linear de `GetResourceLayout`/tamanho/offsets/cópia; nenhum swizzle real e nenhum dado de jogo é copiado.
- Estado: a compilação parou em dois `let ... else` com `unsafe` sem parênteses. Próximo passo: corrigir, rodar fmt/clippy/test/build e reexecutar sem variáveis de diagnóstico.

## Lote 83 — display target e aliases físicos (25/11/2026, 21:35Z)

- Adoção de `__XMemAllocationHooks__` em `xmem.rs` e `runtime_alloc(attr=D3D_DISPLAY_TARGET)`: o slot 127 (`CreateCommittedResource` com layout) detecta `D3D12_HEAP_FLAG_ALLOW_DISPLAY` e chama o hook do título. `scarlett-exec62` confirmou `XMemAlloc via title hook size=65077248 attributes=0x30808000`; a assertion de orçamento `+0x247af2d` foi superada.
-Novo bloqueio em `+0x1e6c29f`: `XMemMapPhysicalPages` devolveu `NULL` para 99.776 páginas numa reserva `kind=0x20402000`, porque `map_run` só aceitava placeholders. `physical.rs` agora commita o intervalo reservado pelo próprio título quando a view de placeholder falha, registrando/removendo os aliases GPU por tipo. Teste Windows novo cobre mapeamento em reserva comum.
- `XPORT_DUMP_SLOT` é diagnóstico opt-in (desligado por padrão) que lê argumentos via `ReadProcessMemory`, sem copiar dados de jogo para o repo.
- Validação: fmt, clippy `-D warnings`, `cargo test -p xport-hle` (98 passed, 3 ignored + integração) e build do CLI passaram. Próximo passo: reexecutar `scarlett-exec63` com `XPORT_GAME_DIR`, rede bloqueada e saídas em `%LOCALAPPDATA%\Temp\opencode`.

## Lote 82 — retomada (25/11/2026, 21:02Z)

- `scarlett-exec45..56`: a assertion em `+0x247af2d` ainda é o bloqueio. Watchpoints opt-in (`XPORT_WATCH_BUDGET`, Dr0..Dr3 corrigidos em `crash.rs`) não dispararam: os dois qwords da stack não são escritos; o valor `0x4000000` já está lá quando o frame é formado.
- Experimentos descartados: zerar `args[2]+0xd0/+0xd8` passou na assertion, mas o título chamou o valor escrito como código (`0xc0000005` em `0x0`); escrever um buffer de 2 MiB passou a saltar para o buffer; devolver um `Kind::Resource` em `args[2]` voltou à assertion original. Nenhum desses experimentos é uma correção.
- Estado atual do código: `dispatch_stub` ainda escreve o objeto sintético no slot 133; `synthetic_noop` ficou sem uso. Próximo passo exato: desmontar `0x14247ae8f`, `0x141647220`, `0x140cc2780` e `0x140cc2840` para identificar out-pointer/ABI reais do slot 133 e a origem dos qwords `rsp+0x70/+0x78` antes de qualquer nova escrita.
- Validação recente: `fmt`, `clippy -D warnings`, `cargo test -p xport-hle` (97 passed, 3 ignored + integração) e build do CLI passaram; a última execução foi `scarlett-exec56`, sem rede, com `XPORT_GAME_DIR` no subdiretório do dump.

## Lote 82 em andamento — inicialização real do título (25/11/2026, 20:50Z)

- `xmem.rs` preserva os bits de placeholder do console (`0x42000` reserve/split e `0x20406000` replace) e o heap de descritores sintético usa `last_placeholder()` como VA de CPU. `prepare.rs` substitui `clwb` por NOPs somente no build `b99f876b74d4a77f...`; é um hint de cache, não lógica de jogo.
- `d3d12_null.rs`: dispositivo D3D12.X sintético avançou por slots 8/9/14/15/21/26/48/73 e pela cadeia DXGI. O access inválido em `+0x193e1b4` (`call [rax+0x3f8]`, `rax=0x1fb0000`) veio da ABI errada do slot 25 (`GetResourceAllocationInfo` escrevia no `this`); o buffer oculto é o primeiro argumento. Corrigido e limitado a `0x10000` para satisfazer a assertion de orçamento de 2 MiB do título.
- `scarlett-exec41`: `0x80000003` em `+0x247af2d`, com `(stack+0x78 - stack+0x70) + global > 0x1fffff`. Próximo passo: validar fmt/clippy/test/build e reexecutar como `scarlett-exec42`, sem rede, com saídas em `%LOCALAPPDATA%\Temp\opencode`.
- Diagnóstico `XPORT_WATCH_DEVICE` foi usado apenas localmente; a instrumentação de watchpoint ficou em `crash.rs` como ferramenta opt-in, não habilitada por padrão. Slots 19, 20, 70, 127 e 133 seguem como compatibilidade documentada, não implementação real.

## Lote 80 — XPackage e XDisplay offline (em andamento)

- O debugger corrigido confirmou que o lookup que disparava `+0x2237178` procurava `appdata0.rpf` com root `G:/`; o arquivo está no subdiretório do dump. Com `XPORT_GAME_DIR` apontando para esse subdiretório o crash migrou para `+0x19f4f58`, e os warnings seguintes pediram as áreas `XPackage` (`af406016-...`/`0b9d556e-...`) e `XDisplayTryEnableHdrMode` (`03f0fe74-...`/`35f07670-...`).
- Novos módulos autorais: `xpackage.rs` serve slots da tabela GDK pública com identificador/localidade/mount path e falha explícita em instalação/montagem com UI; `xdisplay.rs` reporta HDR como `Disabled` e não altera o display do host. GUIDs/slots vêm da tabela pública do XCL e da documentação pública do GDK; nenhum código de terceiros foi copiado.
- Reexecução com `XPORT_GAME_DIR` no subdiretório (`scarlett-exec15`) confirmou que os warnings de área sumiram, exceto `runtime method not implemented area="package" slot=0x138`: o slot 39 é `XPackageEnumeratePackages`. A assertion voltou em `+0x19f4f58` com `rcx/rdx=0x80070057` e `r8=0x80004001`, sinais de que a chamada que devolve `E_NOTIMPL` alimenta a checagem `test al,al`.
- Implementado `enumerate_packages` (slot 39) com assinatura/estruturas públicas do GDK: um pacote sintético entregue ao callback; slots 37/38 agora falham explicitamente como `MountWithUiAsync`/`Result` (antes reusavam indevidamente as funções de install). Teste novo cobre o callback. Validação: `cargo fmt --all`, `cargo check -p xport-hle --all-targets`, `cargo test -p xport-hle` (88 + 1 integração; 3 hardware ignorados) passaram; proveniência registrada em `THIRD_PARTY.md`.
- `scarlett-exec16`: `slot=0x138` desapareceu; a mesma assertion em `+0x19f4f58` persistiu. O mapa de IAT foi corrigido para a directory de imports (índice 1) e o thunk `0x145f3fd80`/IAT `0x146793c18` foi identificado como `d3d12_xs.dll!D3D12XboxCreateDevice`. O HLE encaminhava os argumentos console crus para `D3D12CreateDevice`, cujo segundo parâmetro é um nível de feature; isso explica `E_INVALIDARG`.
- `xbox_bridge.rs`: `D3D12XboxCreateDevice` agora zera o out-pointer, lê apenas o primeiro campo público (`Version`) para log e chama explicitamente `D3D12CreateDevice(default adapter, D3D_FEATURE_LEVEL_12_0, riid, device)`; `E_INVALIDARG` para out-pointer nulo e `E_NOTIMPL` sem host. É um shim de compatibilidade, não uma implementação D3D12Xbox real.
- `scarlett-exec17`: a assertion foi superada; o título chegou a criar um dispositivo D3D12 host e depois travou em `D3D12Core.dll+0x2430e` ao chamar o slot `0xd0` da vtable (`GetCustomHeapProperties`) com `nodeIndex=1`. O dispositivo host não aceita esse índice de nó console; o IID pedido é `ID3D12Device11` (`5405c344-d457-444e-b4dd-2366e45aee39`).
- Lote 81: `d3d12_null.rs`/`d3d12_null_stubs.rs` implementam um dispositivo D3D12.X sintético autoral (vtable de 256 slots, objetos COM falsos, heap/resource/fence plausíveis) e `xbox_bridge.rs` o devolve em `D3D12XboxCreateDevice`. Proveniência registrada em `THIRD_PARTY.md`. Validação: `cargo fmt --all`, `cargo clippy -p xport-hle --all-targets -- -D warnings`, `cargo test -p xport-hle` (91 + 1 integração; 3 hardware ignorados) passaram.
- `scarlett-exec18`/`exec19`: o dispositivo sintético removeu o acesso inválido em `D3D12Core.dll`; o título agora falha dentro de `+0x2478b96` porque `call [rax+0x248]` (slot console `#73`, `CreateCommandListX`) retornou `S_OK` sem escrever o out-pointer (`rcx=0x47bf`, lixo de stack). Os logs também mostraram que todos os slots não atribuídos caíam no mesmo stub `#3`, então não identificavam o método real.
- Lote 81 (continuação): `default_stub(slot)` passou a dar log com o índice real; slots 73 (`CreateCommandListX`) e 48 (`CreateCommandQueueX`) escrevem o out-pointer; `QueryInterface` devolve objetos DXGI sintéticos para `IDXGIDevice/1/2`, com `GetAdapter` (slot 7 do device DXGI) e `EnumOutputs` (slot 7 do adapter, out no 3º argumento). Validação: `fmt`, `clippy -D warnings`, 93 testes + 1 integração. `scarlett-exec22` chegou ao fim da cadeia DXGI e voltou ao breakpoint `+0x19f4f58`, agora com `rcx=0x7ffdd24d0360`, `rdx=0x7ffdd24d0340`, `r8=0x167ff7e45f8`; restaram warnings nos slots 21 (dxgi_output), 78 e 79 (device, frame clock).
- Lote 81 (diagnóstico adicional): `+0x19f4f58` é uma assertion do próprio título: `call +0x2478440; test al,al; int3`. `scarlett-exec24` provou, com log do bridge, que `RegisterClassExA` (atom `49878`) e `CreateWindowExA` (HWND) têm sucesso. `host_kind()` em `xmem.rs` remove `MEM_MAPPED`/bits de seção do console, corrigindo a reserva de 185 MB que falhava (`kind=0x42000`, agora `result!=0`). Slot 21 do output DXGI passou a enumerar um modo sintético. `scarlett-exec25` ainda para no mesmo `int3`; 581 chamadas a `CreateCommandListX` confirmam que a falha está depois do loop de 512 command lists, na alocação interna em `0x1417f6a30`/`0x1417f6ba0` (CRT/TLS do título, não QueryApiImpl). Um experimento temporário pulando apenas esse `int3` avançou para `+0x5eb24bf` (read `0x18`) e foi revertido. Validação: `fmt`, `clippy -D warnings`, 96 testes + 1 integração, build do CLI.
- Próximo passo: instrumentar/seguir a alocação CRT do título (`0x1417f6ba0` → heap TLS via `gs:[0x58]`+`0x13a8`, `0x144fac3c0`) para descobrir por que ela retorna nulo; alternativamente corrigir a inicialização de TLS/heap do módulo mapeado. Slots 131/132 (command list) e 78/79 (device) continuam como stubs documentados.

## Segurança e XNetworking offline (Lote 79, 2026-11-25)

- Um `--execute` com `T:` mapeado mostrou que o Breakpad do título gravava um minidump em `T:\` e fazia upload para `submit.backtrace.io` com WinHTTP do host (HTTP 200). Isso foi interrompido: `winhttp_offline.rs` agora resolve os 15 imports de `WINHTTP.dll` com falha limpa e log único, e `http_bridge.rs` desativa toda rede por padrão (`XPORT_ALLOW_NETWORK=1` é opt-in explícito).
- Os GUIDs pedidos no início da inicialização (`37e56907-...` / `bf2346b2-...`) foram identificados como a área XNetworking na tabela pública do GDK. `xnetworking.rs` serve os slots 3–20 com valores offline determinísticos (porta UDP 3074, TLS 1.2/1.3, connectivity hint, tokens) e falha explícita para pinning/configuração/estatísticas; `QueryApiImpl` agora devolve `S_OK` para o par solicitado.
- Validação do crate: `cargo fmt --all`, `clippy -p xport-hle --all-targets -- -D warnings` e `cargo test -p xport-hle` passaram (82 + integração). Proveniência registrada em `THIRD_PARTY.md`. Nenhuma execução com `--execute` foi repetida após esta mudança; o crash anterior permanece em `+0x2237178`.

## Ponto atual

A fatia mais recente cobre `CopyTextureRegion` method-level no replay D3D12.X, resolvendo texturas T# ou páginas XMem por VA console. A camada de descritores na qual ela se apoia vem das fatias anteriores: `console_desc.rs` lê slots V# (32 bytes), T# (32 bytes) e S# (16 bytes) com as posições de bits GFX10; slots V# viram views raw SRV/UAV sobre páginas XMem, slots S# viram samplers host numa heap SAMPLER e slots T# viram texturas committed com SRV no mesmo heap CBV/SRV/UAV dos buffers, todos ligados ao replay por `submit_d3d12x_compute_stream_with_xmem_descriptors`. T# lineares (SW_MODE 0) têm os pixels copiados de XMem para o recurso host via `GetCopyableFootprints`/`CopyTextureRegion`; modos em mosaico/MSAA/BC falham explicitamente. A fatia anterior ligou as janelas esparsas ao replay dos shaders traduzidos: `D3d12SparseWindows` fornece os recursos `u2..u5` e o base UMA (`io[112]`); `submit_translated_compute_dispatches_with_windows` sincroniza as páginas XMem registradas, materializa argumentos que não couberam em s4..s15 numa tabela na janela 0 (`io[0..1]`) e submete os dispatches. A validação em GPU Windows real foi feita para o caminho DXC→D3D12→dispatch→readback com dados sintéticos; o runtime ainda deve fornecer os GPU VAs/base UMA do título e isso não prova D3D12.X/RDNA2.

A fundação Rust já inclui parser PE32+, loader Windows, binding de imports, CLI de inspeção/execução e HLE parcial. O caminho XMem conecta `XMemFreePhysicalPages` ao cache de espelhos D3D12: páginas físicas liberadas só são descartadas depois que seus aliases de CPU desaparecem.

O xport-gpu lê root signatures DXBC/RTS0 e decodifica um subconjunto D3D12.X compute: pipelines conhecidos, root constants, root descriptors, descriptor tables, dispatch direto e os markers de replay de barreiras/cópias (`0x8C0F_E005/E007/E008/E00A`). Também grava esses registros na fila D3D12 quando recebe bindings de PSOs/root signatures traduzidos e conversores de GPU addresses; tabelas V#/S#/T# do console são lidas diretamente da memória da heap e materializadas em heaps host shader-visible. Transições e cópias method-level usam VAs XMem mapeados e o estado por página do recorder; UAV/aliasing, `CopyBufferRegion` e `CopyTextureRegion` são materializados no mesmo submission; o replay resolve texturas T# ou páginas XMem por VA console. Dispatch indireto e classes desconhecidas falham explicitamente.

## Fatia concluída — implementações reais para os 112 imports (Lote 78)

- Os 112 imports que antes eram stubs agora estão distribuídos em módulos com comportamento: `win32_bridge.rs` (7 pass-through `user32.dll`), `host_bridge.rs` (10: MFPlat, MFReadWrite, DirectStorage, D3D12, GameInput), `apu_bridge.rs` (19: heap APU via VirtualAlloc, mapas APU↔host, filas XApu/XDsp), `bink_bridge.rs` (23: abre e lê cabeçalho BIK real, controla frames/loop/volume/async), `http_bridge.rs` (38: estado de request/response e `HCHttpCallPerformAsync` com WinHTTP real, TLS incluído), `xbox_bridge.rs` (15: FFX e D3D12X com forward real quando existe símbolo host; BVH e XG como shims autorais com handles e estruturas de saída limitadas).
- `title_compat.rs` foi removido: **não resta nenhum stub** para o exe real. `xport run G:\game_scarlett_master.exe` mantém binding de 140 HLE + 447 host + 0 title + **0 missing**, `EXIT=0`, hash `b99f876b74d4a77f...`.
- `crates/xport-hle/Cargo.toml` ganhou a feature `Win32_Networking_WinHttp` para o caminho HTTP. Nenhum asset, DLL, tabela ou código de jogo foi copiado para o repo; só nomes de import e chamadas documentadas do SDK.
- Limitações honestas: Bink não decode quadros (driver nulo com header real); APU não processa áudio; HTTP não invoca o callback de conclusão (a conclusão é lida pelos getters); FFX/D3D12X caem para `E_NOTIMPL` quando o host não exporta o símbolo; BVH/XG são shims de estado, não raytracing/swizzle reais.
- Validação: `fmt --all --check`, `check --workspace`, **210 testes aprovados** (1 filtrado, 5 hardware ignorados), Clippy `-D warnings` em todos os alvos, `xtask policy` (`policy: ok`). 25 testes novos cobrem os seis módulos de compatibilidade.

## Fatia concluída — compatibilidade para o título real (Lote 77)

- Novos módulos HLE: `title_compat.rs` com os 112 imports que faltavam (104 nomes simples, 1 ordinal `#1`, 7 nomes C++ mangled) e `win32_bridge.rs` com pass-through real para `user32.dll` nos sete nomes `ext-ms-win-rtcore-ntuser-*`. Ambos registrados em `modules/mod.rs`.
- `xport run G:\game_scarlett_master.exe` agora termina o binding com **0 missing**: 140 HLE, 447 host, 0 title, `EXIT=0`. Antes eram 28 HLE/447 host/112 missing. O módulo `win32_bridge` substitui os stubs das funções de janela/mensagens.
- Execução controlada com `--execute` (timeout 15 s): o título entra no `main` e falha com `STATUS_BREAKPOINT` (`0x80000003`) em `xport-image-...dll+0x1e1d3a7`, na cadeia `ucrtbase.dll` → código do jogo. Nenhum log `title_compat` apareceu antes do crash, então a falha é anterior ao uso dos stubs (provavelmente inicialização CRT/ambiente), não um import não implementado.
- Limitação importante na época: os 105 exports restantes eram **stubs que retornam zero**. Isso foi superado no Lote 78 acima; o binding fechado ainda não significa que o jogo roda. Log de crash em `%LOCALAPPDATA%\XPORT\data\logs\crashes.log`; saídas em `%LOCALAPPDATA%\Temp\opencode`.
- Validação do workspace: `fmt --check`, `check --workspace`, **199 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 5 hardware ignorados), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Quatro testes novos cobrem os dois módulos de compatibilidade.
- Próximo passo (pendente): investigar o breakpoint na inicialização do CRT (argv/ambiente/API set de console). As implementações de Bink/ACP/libHttpClient/D3D12.X foram feitas no Lote 78, mas ainda não foram reexecutadas com `--execute`.

## Fatia concluída — imagem e som observáveis (`apps/xport-media`, Lote 76)

- Novo crate isolado `apps/xport-media`, aditivo: nenhum crate existente foi editado. Ele gera um PNG sintético (CRC32/Adler32 + DEFLATE `stored`) e um WAV PCM 16-bit/48 kHz estéreo em `%LOCALAPPDATA%\Temp\opencode`, fora do repo, e aceita `--image=`/`--wav=` para arquivos externos fornecidos pelo chamador.
- O PNG é decodificado via WIC do host e apresentado numa janela Win32 com GDI `StretchDIBits`; o WAV é relido do disco, parseado e tocado pelo engine XAudio2 2.9 obtido do export `XAudio2Create` do próprio HLE do XPORT, chamando a vtable do host sem patch. Slots (source #5, mastering #7, destroy #18, submit #21) confirmados no `xaudio2.h` do SDK e apenas conferidos com a tabela do XCL; nenhum código GPL-2 foi copiado.
- Execução real no Windows: `cargo run -p xport-media -- --seconds 2` exibe `image: 320x200`, `audio: 48000 Hz, 2 channels, 16 bits, 192000 bytes`, inicia o playback e termina com `EXIT=0`. A primeira tentativa com `Release` na voice causou `STATUS_ACCESS_VIOLATION`; voices são destruídas com `DestroyVoice` (slot 18) e apenas o engine é liberado com `Release`. Os logs temporários de diagnóstico foram removidos.
- `--no-audio` permite validar só a janela sem device de áudio. Em 25/11/2026 o usuário autorizou explicitamente a leitura dos PNGs soltos do dump: os 5 arquivos (`xbs_splash.png` 1920x1080, `xbs_logo_480.png` 480x480, `xbs_logo_150.png` 150x150, `xbs_logo_store.png` 100x100, `xbs_logo_44.png` 44x44) decodificaram e apresentaram via `--image=` com `EXIT=0`. Nada foi copiado, modificado ou executado; `audio.rpf`/`textures.rpf` e binários seguem intocados.
- Validação: `fmt --all --check`, `check --workspace`, **195 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 5 hardware ignorados), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Quatro testes novos cobrem assinatura/IEND do PNG, Adler32 e round-trip/recusa do WAV.
- Limitação: esta fatia prova decode/presentação de PNG (sintético e 5 PNGs reais do dump) e playback de PCM sintético no host Windows; não prova XMA/Opus/ACP/SHAPE, D3D12.X/RDNA2 nem jogos reais. Não há WAV solto no dump: o áudio está em `audio.rpf`, para o qual o XPORT ainda não tem parser. Próximo passo: estender o mesmo caminho para formatos de áudio comprimidos e texto/captura se o usuário autorizar um asset específico.

## Teste real com `G:\game_scarlett_master.exe` (modo seguro, 25/11/2026)

- O usuário forneceu o executável real do título (120.181.760 bytes). **Não foi executado**: rodei apenas `xport inspect` e `xport run` sem `--execute`, que leem/parseiam/mapeiam e fazem binding, sem chamar o entry point.
- `xport inspect`: 587 imports (28 HLE, 447 host, 0 title, 112 missing); HLE com 5 módulos/192 exports.
- `xport run --report <temp>`: build hash `b99f876b74d4a77f03af5f878223b27bd647b5c35a79d5decd9f716504239e93`, mapeado em `0x7ffdd13f0000` (preferido `0x140000000`), binding com 28 HLE / 447 host / 0 title / 112 missing, `EXIT=0`. A DLL está ao lado do exe em `G:\`, não em `G:\` raiz do dump, então nenhuma title DLL foi carregada.
- Missing agrupados por DLL: `libHttpClient.GDK.dll` 38, `Bink2Scarlett.dll` 23, `AcpHal.dll` 9, `XboxBvhBuilder.dll` 7, `MFPlat.DLL` 6, `xapu.dll` 5, `chad.dll` 5, `amd_fidelityfx_loader_xbox.dll` 4, `d3d12_xs.dll` 3, Win32 ext-ms 6, `xg_xs.dll` 2, e 1 cada em `GameInput`, `MFReadWrite`, `XCRDAPI`.
- Conclusão: o caminho PE/loader/HLE do XPORT funciona com um título real até o binding. A execução de verdade (`--execute`) não foi tentada: com 112 missing e política `trap`, o título pararia no primeiro import não implementado (além de ser código de jogo não confiável). Relatórios JSON ficaram em `%LOCALAPPDATA%\Temp\opencode`, fora do repo; nada foi copiado.

## Fatia concluída — replay encadeado com dispatches traduzidos e readback pelo HLE

- `xport-hle` passou a depender de `xport-shader` no Windows. A sessão `XPortGpuReplay` guarda PSOs sintéticos e bindings XMem, liberando os ranges no `Destroy`.
- Novos exports sintéticos: `XPortGpuReplayRegisterSyntheticPipeline` compila o container autoral com `split_wave64: true` e cria o PSO no device da sessão; `XPortGpuReplayBindXmemPage` registra o mapping virtual→físico; `XPortGpuReplayDispatchDecoded` percorre os registros do último decode, agrupa dispatches consecutivos por pipeline, usa `submit_translated_compute_dispatches` com a página como `res`/janela e faz readback de 4 bytes por `readback_xmem_bytes`.
- Pipeline não registrado, dispatch sem pipeline ativo, mapping ausente, offsets fora da página e ponteiros nulos falham explicitamente com `E_INVALIDARG`; sem DXC/adapter o registro retorna `NOT_SUPPORTED`, e o título sintético pula somente a parte de GPU nesse caso.
- O título sintético agora liga o stream encadeado ao caminho completo: dois pipelines, `s8=0x1234`/`s9=0x5678`, dois dispatches e leitura de `0x68ac` no offset 40. O teste de integração confirma os exports resolvidos e o código de saída 0; um teste GPU opt-in no HLE cobre a recusa de pipeline não registrado e o readback.
- Validação: `fmt --all --check`, checks host e `x86_64-pc-windows-msvc`, **191 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 5 hardware ignorados), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Os cinco testes GPU explícitos passaram, incluindo `xgameruntime_dispatches_decoded_pipeline_and_reads_back_on_hardware`.
- Limitação: o shader e os pipelines são sintéticos e autoralmente gerados; nenhum dump, shader ou asset de jogo foi usado. O sucesso em GPU Windows de consumo não prova D3D12.X/RDNA2. Próximo passo: validar o mesmo caminho com streams mais longos vindos de um título sintético com múltiplos recursos e sincronização de filas, mantendo o runner isolado do CRT.

## Fatia concluída — streams encadeados e ordenação de filas pelo HLE

- `D3d12MemoryBridge::create_direct_command_queue{,_raw}` cria uma fila direct no device do próprio bridge, permitindo que o HLE a importe sem depender do crate `windows`. Uma fila de outro device continua recusada pela validação de `GetDevice`.
- Novos exports sintéticos `XPortGpuReplayDecodedSummary` (contagens de `SetPipeline`/`Dispatch` do último decode) e `XPortGpuReplayCreateQueue` (fila no device da sessão), registrados em `xgameruntime`. A ABI segue sendo autoral; nenhum GUID/slot/opcode de console foi acrescentado.
- Teste unitário `xgameruntime_decodes_chained_pipelines_and_rejects_unknown_pso`: stream com dois pipelines, roots e dispatches decodifica 6 registros, o resumo informa 2 pipelines/2 dispatches e um PSO ausente falha explicitamente. O fixture sintético executa o mesmo stream encadeado; o teste de integração confirma os exports resolvidos pelo `Resolver` real.
- Teste GPU opt-in `xgameruntime_orders_two_queues_on_hardware`: cria e importa a fila pelo HLE, submete preparação de página e os dois sentidos de wait (`WaitExternalFence`/`ExternalWaitForReplay`) com fence sinalizado depois do submission, e recusa handle não importado.
- Validação: `fmt --all --check`, checks host/MSVC, **191 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 4 hardware ignorados), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Os quatro testes GPU explícitos passaram: `xbox container dispatch wrote 0x000068ac`, `texture states tracked and external queue fences synchronized`, `xgameruntime_submits_texture_and_page_commands_on_hardware` e `xgameruntime_orders_two_queues_on_hardware`.
- Limitação: streams e filas continuam sintéticos; nenhum dump de jogo foi executado e GPU de consumo não prova D3D12.X/RDNA2. Próximo passo: ligar o replay encadeado a um stream de título sintético com dispatches traduzidos encadeados e validar CPU/GPU readback pelo HLE.

## Implementado parcialmente

- Análise PE32+ e import libraries.
- Carregamento Windows de executável e DLLs do título, relocations, TLS, resolução de imports e logs de falhas.
- HLE para várias APIs Windows compartilhadas e stubs de XGameRuntime.
- TitleMemoryStatus com ABI de 80 bytes, validação de tamanho e estimativa de memória do processo.
- XMem com hooks do título, páginas físicas em reservas placeholder e aliases registrados no xport-gpu; espelhos D3D12 são liberados após o último alias de uma página física liberada.
- Registro separado de aliases GPU VA→página física XMem, lookup com offset, snapshot ordenado e retenção dos espelhos até saírem os aliases CPU/GPU.
- Layout `XmemWindowLayout` de uma a quatro janelas de 4 GiB com tiles de 64 KiB; `D3d12SparseWindows` cria buffers reservados, poços de tiles, mapeia/spans e desmapeia páginas liberadas, com cópia dos espelhos para os tiles antes do dispatch.
- Bridge D3D12 com recursos upload/default/readback, fila direta, fences, callback de gravação e comandos tipados para transições e cópias.
- Decoder D3D12.X compute parcial em crates/xport-gpu/src/d3d12x.rs, incluindo leitura DXBC/RTS0, markers de replay de barreiras/cópias de buffer e textura e diagnóstico de registros não suportados; replay de descriptor tables por callback, por views XMem descritas pelo runtime ou por slots V#/S# lidos do console.
- Recorder D3D12 expõe `transition_page`, `uav_barrier_page`, `global_uav_barrier`, `aliasing_barrier_pages`, `copy_page_to_page` e `copy_texture_region`; o replay valida estados, resolve VAs console para páginas XMem/texturas T# e recusa segmentações de página incompatíveis.
- Parser clean-room `console_desc.rs` para T#/V# (32 bytes) e S# (16 bytes), com conversão de campos GFX10 para `D3D12_SAMPLER_DESC`, mapeamento IMG_FORMAT→DXGI e validação offline de words/layouts conhecidos.
- Upload linear de T#: planner de subrecursos (`texture_upload.rs`) e bridge D3D12 com `GetCopyableFootprints`, staging, `CopyTextureRegion`, barreiras e fence, ligados ao replay de descritores; swizzle≠0, MSAA e formatos BC recusados explicitamente.
- Tradução de containers Xbox DXBC com compute RDNA2 integralmente coberto pelo subset para HLSL; instruções não suportadas recusadas com contagem por opcode.
- Compilação HLSL→DXIL no Windows com carregamento pareado de dxil.dll/dxcompiler.dll e seleção de SDK/XPORT_DXC_DIR.
- xport-gpu cria/cacheia a root signature ABI interna de seis UAVs, cria PSOs DXIL com ela e submete dispatches diretos em lote após empacotar argumentos do título: inteiros em s4..s15 e excedentes numa tabela em memória apontada por `io[0..1]`, com `io[112]` recebendo o base UMA.
- Replay de dispatches diretos e descriptor tables via `submit_d3d12x_compute_stream`, `submit_d3d12x_compute_stream_with_tables`, `submit_d3d12x_compute_stream_with_xmem_tables` e `submit_d3d12x_compute_stream_with_xmem_descriptors`, usando PSOs sob uma root signature host comum por stream e endereços convertidos; o caminho XMem constrói descritores de buffer, sampler e textura, copia pixels de T# lineares para as texturas host e o runtime fornece os GPU VAs/base UMA reais.
- VFS ligado ao HLE de arquivos: unidades console traduzidas via `MountTable`, leitura no game root, escrita no overlay e enumeração mesclada por `FindFirstFile*`; mounts configurados pelo CLI a partir de `Config`.

## Ainda não implementado

- Áreas de XGameRuntime ainda não registradas (outras além de XGame, XSystem, storage, XAsync/XTaskQueue e XUser); as áreas registradas servem objetos offline.
- Registro automático de PSOs por título no replay e materialização de recursos `res` conforme as necessidades de cada shader.
- Cópia de GPU→XMem e write-watch por 4 KiB; tabelas de argumentos spillados e ligação `u2..u5`/`io[112]` já existem, mas o runtime ainda precisa fornecer os GPU VAs reais e o base UMA do título. Descritores host de buffer/sampler/textura já são construídos a partir de slots V#/S#/T#, e T# lineares já têm os pixels copiados de XMem; falta sincronizar escritas GPU→XMem e cobrir modos de tiling não lineares.
- Decoder de registros gráficos, indirect dispatch e variantes não documentadas.
- Sincronização dos espelhos com filas D3D12 externas ao bridge.
- Atributos de alocação e restrições completas de endereços do console.
- Input e áudio: o XAudio2 2.9 do host já é criado via HLE para PCM/IEEE float; o crate aditivo `apps/xport-media` agora toca WAV PCM 16-bit pelo engine do HLE e mostra PNG via WIC/GDI. Opus/XMA/AcpHal/SHAPE/captura e input faltam. O storage já responde path/quota e prompts XAsync pelo VFS; falta `MountForPackage`.
- Upload de texturas em modos de tiling/swizzle não lineares e compatibilidade demonstrada com jogos reais (o caminho linear SW_MODE 0 já existe).

## Próximas fatias recomendadas

1. Fornecer os GPU VAs/base UMA reais do runtime e validar o replay com um stream/jogo conhecido em GPU.
2. Rastrear estados de texturas host e sincronizar filas externas ao bridge.
3. Validar o replay de cópias de textura com um stream/jogo conhecido em GPU real.
4. Ligar o harness GPU ao caminho de container Xbox DXBC/tradução e validar XUser/XAsync com o entry real quando houver safely isolated runner.
5. Estender `apps/xport-media` para Opus/XMA/ACP e apresentação via swapchain D3D12, mantendo os assets sintéticos até que um asset específico seja explicitamente autorizado.

## Fatia concluída — avaliação e incorporação seletiva de VIctory/XCL

- Objetivo: comparar o estado atual do XPORT com `F:\VictoryV2\VIctory-Belobug-AMD-main` e `F:\xcl-main V2\xcl-main`, identificar os módulos mais valiosos e compatíveis com a licença do XPORT, e incorporar apenas o que agrega valor sem quebrar a política de proveniência.
- Próximo passo exato: ler `LICENSE`/`THIRD_PARTY.md` dos três projetos e inventariar os módulos candidatos (loader/HLE/XMem/GPU/shader) antes de escolher a primeira fatia de integração.
- Nenhum código de terceiros será copiado antes da verificação de licença e do registro em `THIRD_PARTY.md`.
- Levantamento do XPORT concluído: a base é sólida em PE/loader, HLE parcial, XMem/aliases, janelas esparsas, decode/replay compute e upload linear de T#. As maiores lacunas, em ordem de valor, são VFS→HLE, `XGameRuntime::QueryApiImpl`, GPU→XMem/write-watch, estados de textura/filas externas, tiling não linear, PSOs/recursos `res`, gráficos/indirect dispatch, MIMG no shader e validação em GPU real.
- Restrição confirmada: XPORT é GPL-3.0-or-later; a policy barra assets de jogo, arquivos binários e fontes >512 KiB sem exceção. Qualquer port precisa de registro explícito em `THIRD_PARTY.md` e deve respeitar a separação entre código autorizado pelo dono e material de terceiros (AMD/LLVM/AnyPS5).
- Levantamento do XCL concluído: os candidatos mais valiosos são `QueryApiImpl`/tabela GDK (`hle/xgameruntime/api.rs`, `gdk_table.rs`), staging/write-watch em `sparse.rs`/`title_textures.rs`/`cpu_writes.rs`, sync graph de filas em `list_x.rs` e tiling/swizzle em `gpu/tiling.rs`. Tiling e tabelas AMD/LLVM/AnyPS5 exigem proveniência própria; o restante é código autoral coberto pela autorização do dono.
- Lote 63 concluído: `xgameruntime.rs` troca o stub por um registro COM adaptado do VIctory, com GUIDs/slots validados pela tabela GDK do XCL; `xgame_api.rs` e `xsystem_api.rs` implementam offline title id/features/invites/eventos e identidade/analytics. `QueryApiImpl` agora devolve objetos para pares registrados e `NOT_SUPPORTED` com log único para os demais.
- Validação: `fmt --check`, checks host/MSVC, 165 testes aprovados no workspace (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos. Próximas áreas offline (XUser/XAsync/XTaskQueue) e validação em GPU real seguem pendentes.

## Fatia concluída — integrar VFS ao HLE de arquivos

- Objetivo: ligar `xport-vfs::MountTable` aos exports de arquivo em `xport-hle`, usando a tradução de caminhos do VIctory (`victory-hle/src/modules/files.rs`) como referência autorizada.
- Lote A concluído: `MountTable::read_path` expõe o caminho de leitura sem redirecionar para o overlay; `xport-hle/src/vfs.rs` traduz caminhos wide/verbatim, distingue intenção de escrita em `CreateFile` e recusa escritas na raiz do jogo sem overlay.
- Lote B concluído: `xport-hle` depende de `xport-vfs`, expõe `vfs::{set_mounts, clear_mounts}` e liga a tradução em `CreateFileA/W`, `GetFileAttributesA/W/ExA/ExW`, `DeleteFileA/W`, `CreateDirectoryA/W` e `RemoveDirectoryA/W`. Bug corrigido na classificação de `OPEN_EXISTING` (3) vs `TRUNCATE_EXISTING` (4), que redirecionaria leituras para o overlay. Teste ponta a ponta no Windows confirma leitura do game root, escrita no overlay e recusa sem overlay.
- Lote C concluído: proveniência no Lote 64 do `THIRD_PARTY.md`, README/arquitetura/worklog atualizados. Validação: `fmt --check`, checks host/MSVC, **170 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.

## Fatia concluída — completar a frente VFS (config + enumeração)

- Objetivo: configurar o `MountTable` a partir do CLI/config do XPORT e traduzir também `FindFirstFile*`/`FindNextFile*`; manter leitura no game root e escrita no overlay.
- Lote A concluído: `vfs::title_mounts` monta `G:`/`GAME:` no game root, `T:` no cache e overlay no data dir; `FindFirstFileA/W` traduz o padrão via MountTable (`FindNextFile`/`FindClose` continuam por handle). Teste de enumeração no Windows passou após corrigir o buffer de `WIN32_FIND_DATAW` de 320 para 592 bytes.
- Lote B concluído: o CLI instala a tabela a partir de `Config` (`game_dir`, `cache_subdir("temp")`, `data_subdir("overlay")`), com validação de policy nos diretórios derivados. Proveniência no Lote 65 do `THIRD_PARTY.md` e docs atualizados.
- Validação final da frente: `fmt --check`, checks host/MSVC, **171 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Restrição: nenhum arquivo de jogo pode ser modificado; config fora do repo continua validada pela policy. Nada de terceiros sem registro no `THIRD_PARTY.md`.

## Fatia concluída — mesclar listagens game root/overlay

- Objetivo: `FindFirstFile*` deve devolver uma única enumeração com entradas do game root e do overlay, com o overlay vencendo em nomes repetidos.
- Código pronto em `kernel.rs`: handle sintético registra entradas coletadas dos dois diretórios (`FoundEntry`/`MergedFind`), dedupe por nome case-insensitive, omite `.`/`..` e serve `FindNextFileA/W`/`FindClose`. `vfs::translate_overlay_wide` fornece o padrão do overlay. Teste Windows confirma `a.pak`, `b.pak` e `c.pak` com o tamanho de `a.pak` vindo do overlay; a abertura/leitura de arquivos continua no game root por design.
- Lote B concluído: proveniência no Lote 66 do `THIRD_PARTY.md`, README/arquitetura/worklog atualizados. Validação: `fmt --check`, checks host/MSVC, **171 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.

## Fatia concluída — ligar storage ao VFS

- Objetivo: expor uma área offline de storage persistente do XGameRuntime usando o overlay/data dir do VFS, sem gravar na pasta original do título.
- Lote A concluído: `xport-hle/src/storage.rs` implementa PLS offline com drive `S:\`, path/quota e `used_bytes` sem seguir symlinks. `configure` instala o mount via `vfs::mount_drive`; os testes do storage passaram. Prompt/XAsync e `MountForPackage` permanecem `E_NOTIMPL` explícito.
- Lote B concluído: `storage::INTERFACES` registrado no `QueryApiImpl` e o CLI chama `storage::configure` com `data_subdir("pls")` após instalar os mounts. Teste Windows confirma que `S:\save.dat` grava na pasta PLS e não no game root.
- Lote C concluído: proveniência no Lote 67 do `THIRD_PARTY.md`, README/arquitetura/worklog atualizados. Validação: `fmt --check`, checks host/MSVC, **175 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.

## Fatia concluída — revisão geral e correções

- Revisão paralela das fatias 63–67 corrigiu: tabelas COM estendidas (XGame até `0x28`, XSystem até `0x40`) com `missing::<n>`, `QueryInterface` no slot 0 retornando `E_NOINTERFACE` e `QueryApiImpl` com out nulo retornando `E_INVALIDARG`.
- `CreateFileA/W` passa flags ao VFS; `GENERIC_ALL`, `DELETE`, `WRITE_DAC/OWNER` e `FILE_FLAG_DELETE_ON_CLOSE` são tratados como escrita. Teste Windows confirma que delete-on-close sem overlay falha com erro 5 e não apaga o game root.
- Features alinhadas ao GDK/XCL (`0 | 2..=13 | 15..=21`), licença do XCL corrigida no Lote 68 do `THIRD_PARTY.md`, README/arquitetura/STATUS qualificando a mesclagem da listagem vs. leitura no game root, logs limitados e `storage::clear` desmontando `S:`.
- Validação final: `fmt --check`, checks host/MSVC, **178 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Próximos passos pendentes retomados abaixo: XUser/XAsync, áudio e validação com título sintético/GPU/jogo real; nada foi commitado ou enviado para review.
- Restrição: nada de terceiros sem registro no `THIRD_PARTY.md`; nenhuma validação em GPU, stream real ou jogo real.

## Fatia concluída — XAsync/XTaskQueue e XUser offline

- Objetivo: ampliar as áreas offline do XGameRuntime com filas de tarefas, blocos assíncronos e um usuário local, sem forjar identidade/assinatura.
- Lote 1 concluído: levantamento em VIctory/XCL definiu GUIDs, slots, contrato de `AsyncBlock`, riscos e o subconjunto offline; o levantamento de áudio recomendou começar por XAudio2 pass-through/PCM e pelo endpoint de áudio do XUser.
- Lote 2 concluído: `xasync.rs` implementa 31 slots da tabela GDK com filas manuais/threadpool/serializadas/immediatas, monitor, terminate, process queue e calls com `Begin/Schedule/Complete/GetResult`; handles são tokens em tabelas e um id no bloco impede ABA. Wait-handle registration, XThread time-sensitivity e PC forwarding são `missing` explícitos.
- `xuser.rs` serve um usuário offline (`XUID 0x0009000000000001`, `LOCAL_ID 1`, gamertag `Player`, estado 0, adulto, todos os privilégios), `XUserAddAsync`, gamertag, local id e device provider. Tokens/signature falham com `0x89245111`; sem dispositivo associado não há endpoint de áudio, e o slot inválido devolve `E_INVALIDARG`. Interfaces tardias mapeadas para o mesmo objeto com slots explícitos.
- `storage.rs` liga `PromptUserForSpaceAsync`/`GetResult` (`0x30`/`0x38`) ao XAsync; a decisão é determinística (quota disponível → `S_OK`, senão `E_ABORT`). `MountForPackage` continua `E_NOTIMPL`.
- Proveniência registrada no Lote 69 do `THIRD_PARTY.md`. Validação: `fmt --check`, checks host/MSVC, **184 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Limitação: permissões, token e assinatura não são forjados; nenhuma validação em jogo real, GPU real ou título sintético com áudio.

## Fatia concluída — primeira fatia de áudio

- Objetivo: fazer `xaudio2_9.dll`/`xaudio2_9redist.dll` resolverem no HLE sem copiar ou patchar vtable e sem importar assets/áudio de jogos.
- `modules/xaudio2.rs` exporta `XAudio2Create` e `XAudio2CreateWithVersionInfo`: máscara as flags console-only, força processador padrão e devolve o objeto COM nativo do XAudio2 2.9 do host. O módulo foi registrado em `modules::all()`, então `inspect`/`run` e `GetProcAddress` pelo pseudo-handle passam a ver os dois exports.
- PCM/IEEE float seguem direto para o host; XMA/XMA2/Opus e demais formatos não são decodificados nem substituídos por silêncio: o próprio XAudio2 falha com HRESULT explícito. Não há leitura de arquivo de áudio nesta fatia; bancos carregados pelo título continuam passando pelo `xport-vfs`.
- O endpoint de áudio do XUser permanece sem dispositivo associado (`E_NOT_FOUND`); não há captura, Media Foundation, Opus, XMA ou AcpHal/SHAPE.
- Proveniência no Lote 70 do `THIRD_PARTY.md`. Validação parcial: `cargo test -p xport-hle xaudio2` (3 testes), fmt e Clippy limpos. A validação completa do workspace segue na frente de validação.

## Fatia concluída — validação com título sintético e GPU real

- Objetivo: validar o caminho com título sintético e GPU real disponível, sem afirmar compatibilidade com jogos.
- `tests/fixtures/xport-synthetic-title` é um `.exe` somente-fonte que importa `XGameRuntimeInitialize`, `QueryApiImpl`, `XAudio2Create`, `XAudio2CreateWithVersionInfo` e `GetTickCount`. `crates/xport-hle/tests/synthetic_title.rs` compila o fixture, prepara/mapeia a imagem e faz o binding com o `Resolver` real; nenhum desses imports fica missing. O executável gerado fica no target e não é versionado.
- `crates/xport-gpu/tests/d3d12_hw.rs` é ignorado por padrão e exige `XPORT_GPU_HW_TEST=1` (mais DXC/DXIL via `XPORT_DXC_DIR` ou Windows SDK). Ele compila HLSL sintético → DXIL, cria PSO na root signature ABI, envia um dispatch, lê o resultado de uma página XMem por GPU e confere a soma. Executado em NVIDIA GeForce GTX 1660 SUPER (driver 32.0.16.1714): `d3d12 hardware dispatch wrote 0x000068ac`; o teste passou.
- Validação da frente: `fmt --check`, checks host e MSVC, 188 testes aprovados (1 filtrado: `llvm_gfx10_corpus`; 1 hardware ignorado no suite padrão), Clippy Windows `-D warnings`, `xtask policy` (`policy: ok`) e o teste GPU explícito limpos.
- Limitação: a GPU é hardware de consumo Windows, não uma GPU console/RDNA2; o resultado prova o caminho DXC→D3D12→dispatch→readback do XPORT, não compatibilidade com D3D12.X ou com jogos reais. Não foi executado nenhum dump de jogo.
- Jogo real continua bloqueado: dumps locais estão marcados “NFE, não executar”; não executar sem autorização explícita.

## Fatia concluída — título sintético executa XAsync/XUser/XAudio2

- Objetivo: exercitar de fato as áreas offline com `QueryApiImpl` e o `Resolver` real, sem tocar dumps de jogos.
- `tests/fixtures/xport-synthetic-title` agora exporta `xport_synthetic_run`. O teste chama essa export, não o entry CRT: `spawn_main` poderia encerrar o processo de teste via `ExitProcess`. O código do título roda no processo do XPORT, mas o harness permanece vivo para conferir o resultado.
- O fixture consulta `QueryApiImpl` para os providers de XUser, gamertag, XTaskQueue e devices; chama `XUserAddAsync` (slot `0x38`), lê status pelo XTaskQueue (`0x18`) e result pelo XUser (`0x40`), confirma XUID/gamertag, verifica que token falha com `0x89245111` e que o endpoint de áudio devolve `E_NOT_FOUND`. `XAudio2Create` e `XAudio2CreateWithVersionInfo` criam e liberam dois engines host; flags console-only são mascaradas. Nenhum formato comprimido é decodificado ou simulado.
- Validação do fixture: `cargo test -p xport-hle --test synthetic_title -- --nocapture` passou com código `0`; `fmt --all` e `clippy -p xport-hle --all-targets -- -D warnings` limpos. A validação completa do workspace segue na frente de validação.
- Limitação: formatos XMA/Opus não são rejeitados por uma vtable do XPORT nesta fatia; a recusa depende do HRESULT do XAudio2 host. O entry CRT/`--execute` não foi exercitado por este teste.
- Etapa seguinte concluída: o harness GPU agora usa um container Xbox DXBC sintético; ver a fatia abaixo.

## Fatia concluída — container Xbox DXBC sintético → tradução → GPU

- Objetivo: trocar o HLSL escrito à mão no harness por um container Xbox DXBC construído em código, passá-lo por `translate_xbox_compute`/`compile_xbox_compute` e só então criar PSO, dispatch e readback em GPU real.
- Lote 1: `xport-shader` expõe `synthetic_xbox_compute_blob()`; os builders de container passaram a viver em `gpu::xbox::synthetic`. O teste de shader confirma que o RDNA2 sintético (`s_add_u32` + `v_mov_b32` + `global_store_dword`) gera HLSL com `s8 + s9` e `mem.Store`. Nenhum dump ou asset de jogo é usado.
- Lote 2: `d3d12_hw.rs` trocou o HLSL escrito à mão por `compile_xbox_compute`, ligou `u2` à própria página XMem usada no readback e passou a verificar a soma produzida pelo HLSL traduzido.
- Foi necessário `split_wave64: true`: sem isso a GTX 1660 recusou o PSO com `0x80070057` porque o HLSL pedia `WaveSize(64)`. Com a divisão em waves de 32 lanes, o PSO foi criado e o dispatch completou.
- Execução real: `cargo test -p xport-gpu --test d3d12_hw -- --ignored --nocapture` com `XPORT_GPU_HW_TEST=1` e `XPORT_DXC_DIR` do SDK imprimiu `xbox container dispatch wrote 0x000068ac`; o teste passou.
- Limitação: o container é sintético e autoral, e a GPU é hardware de consumo Windows; o resultado prova container→tradução→DXC→PSO→dispatch→readback, não D3D12.X/RDNA2 nem compatibilidade com jogos. Nenhum dump de jogo foi executado.
- Validação final: `fmt --all --check`, checks host e MSVC, **189 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 1 hardware ignorado no suite padrão), Clippy `-D warnings`, `xtask policy` (`policy: ok`) e o teste GPU explícito limpos. README, ARCHITECTURE, WORKLOG e THIRD_PARTY atualizados; nenhum código novo de terceiros foi portado nesta fatia.
- Etapa seguinte concluída: estados de texturas host e filas externas; ver a fatia abaixo.

## Fatia concluída — estados de texturas host e filas externas

- O bridge passou a rastrear estado por `(recurso, subrecurso)` para texturas host resolvedas. `register_texture_state`, `texture_state` e `transition_texture` validam explicitamente o estado atual; uma textura sem estado registrado ou com `before` divergente falha antes de emitir a barreira. `upload_texture_subresources` semeia `COMMON` para os subrecursos enviados.
- Barreiras `Transition` decodificadas e cópias `CopyTextureRegion` agora usam o mesmo modelo: texturas resolvedas precisam estar em `COPY_DEST`/`COPY_SOURCE` conforme o lado da cópia; páginas XMem continuam com a validação anterior.
- `import_external_queue` aceita apenas filas do mesmo dispositivo. `D3d12ReplayQueueSync` e os comandos `WaitExternalFence`/`ExternalWaitForReplay` fazem waits GPU-side antes/depois do batch, via `ID3D12CommandQueue::Wait` com fences do próprio bridge; handle não importado falha explicitamente.
- Fixture sintético em `d3d12_hw.rs`: usa uma página XMem preparada como recurso de textura, verifica ausência de estado, divergência de `before`, transições COMMON→COPY_DEST→COMMON e os dois sentidos de sincronização entre duas filas do mesmo device. Nenhum asset ou dump de jogo é usado.
- Validação da frente: `fmt --all --check`, checks host e MSVC, **189 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 2 hardware ignorados no suite padrão), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Os dois testes GPU explícitos passaram em hardware real: `xbox container dispatch wrote 0x000068ac` e `texture states tracked and external queue fences synchronized`.
- Etapa seguinte: integração do replay a XGameRuntime; ver seção abaixo.

## Fatia concluída — replay integrado a XGameRuntime

- `crates/xport-hle/src/modules/gpu_replay.rs` adiciona exports `XPortGpuReplayCreate/Destroy/DecodeStream/ImportQueue/SubmitCommands` ao módulo `xgameruntime`. São uma extensão sintética autoral do XPORT: nenhum GUID, slot de vtable ou opcode de console foi inventado; `QueryApiImpl` continua servindo apenas os pares validados de VIctory/XCL.
- Sessões opacas retêm o bridge D3D12 (criado sob demanda), filas importadas e os últimos registros decodificados. `XPortGpuReplayDecodeStream` usa `decode_d3d12x_compute_stream` com o ABI de compute do XPORT e falha explicitamente em stream truncado/desconhecido, sem precisar de GPU.
- `XPortGpuReplaySubmitCommands` converte uma ABI tipada privada em `D3d12ReplayCommand`s: preparação de página, transição de página, cópias upload/readback, registro e transição de estado de textura por subrecurso (com `before` explícito) e waits de fila externa nos dois sentidos. Comandos inválidos são rejeitados antes de qualquer trabalho de GPU; sem adapter D3D12 retorna `NOT_SUPPORTED`.
- O título sintético `tests/fixtures/xport-synthetic-title` agora importa os exports e executa um stream longo (pipeline + root + transição + UAV barrier + cópia + dispatch), verifica contagem, rejeita stream truncado e comando inválido, e libera a sessão. O teste de integração confirma que os quatro exports novos são resolvidos pelo `Resolver` real.
- Teste GPU opt-in em `gpu_replay.rs` prepara uma página XMem, registra estado de textura, executa COMMON→COPY_DEST→COMMON pelo HLE e confirma que um `before` divergente falha explicitamente.
- Validação da frente: `fmt --all --check`, checks host/MSVC, **190 testes aprovados** (1 filtrado: `llvm_gfx10_corpus`; 3 hardware ignorados no suite padrão), Clippy `-D warnings`, `xtask policy` (`policy: ok`). Os três testes GPU explícitos passaram: `xbox container dispatch wrote 0x000068ac`, `texture states tracked and external queue fences synchronized` e o replay via `xgameruntime`. Título sintético: `synthetic_title_runs_xasync_xuser_xaudio2_and_replay_from_hle ... ok`.
- Limitação: a ABI `XPortGpuReplay*` é sintética e não representa GDK/D3D12.X; nenhum dump de jogo foi executado e GPU de consumo não prova compatibilidade com console. Próximo passo: estender o replay para streams sintéticos encadeados com múltiplos pipelines e validar a ordenação entre filas pelo HLE.

## Fatia concluída — barreiras/cópias method-level no recorder

- Ponto de partida: o decoder só aceitava pipeline/root/dispatch e todo method-level caía em `UnknownRecord`; o recorder já rastreava estado por página, mas só usava `CopyBufferRegion` internamente para upload/readback.
- Lote 1 concluído: levantamento no XCL confirmou que os tags `0x8C0F_E005` (UAV global), `0x8C0F_E007` (array de `D3D12_RESOURCE_BARRIER` inline) e `0x8C0F_E008` (`CopyBufferRegion`) são o formato de replay sintético do console-list shim, não uma ABI console comprovada. XPORT decodifica o mesmo subconjunto com esse registro documentado.
- Lote 2 concluído: `D3d12xBarrier` (transition/aliasing/UAV), `D3d12xResourceState` e três variantes em `D3d12xComputeRecord`, com erros explícitos para contagem inválida, tipo de barreira desconhecido e flags não suportadas. Testes sintéticos cobrem o layout de 8 words por barreira, o record de 11 words de cópia e truncamento.
- Lote 3 concluído: o recorder expõe UAV por página/global, aliasing entre páginas e `copy_page_to_page`. O replay resolve o VA console para a página XMem, valida `before`/`after` contra o estado pendente e converte endereço+offset em segmentos por página; cópias com segmentações incompatíveis são recusadas.
- Validação: `fmt --check`, checks host/MSVC, 155 testes aprovados no workspace (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Limitação desta fatia: transições afetam a página XMem que contém o VA base (sem sub-recursos); estados console-only e filas externas seguem fora de escopo. `CopyTextureRegion` method-level foi coberto na fatia seguinte. Nenhuma validação em GPU real, stream real ou jogo real.

## Fatia concluída — `CopyTextureRegion` method-level no replay

- Ponto de partida: o decoder já cobria markers de barreira/`CopyBufferRegion`, mas o tag `0x8C0F_E00A` caía em `UnknownRecord`; o recorder não expunha `CopyTextureRegion` e o replay não conhecia as texturas committed criadas para T#.
- Lote 1 concluído: `D3d12xTextureCopyLocation` (subresource/placed footprint), `D3d12xBox` e a variante `CopyTextureRegion` decodificam os 35 words do marker, com erros para tipo de localização e flag de box inválidos. Testes sintéticos cobrem footprint placed com box e truncamento.
- Lote 2 concluído: `D3d12CommandRecorder::copy_texture_region` usa a API host. Novos entry points `submit_decoded_compute_with_textures`/`submit_d3d12x_compute_stream_with_textures` recebem um resolvedor `(VA, subrecurso) -> ID3D12Resource`; as versões anteriores delegam com `None` (fallback para páginas XMem).
- Lote 3 concluído: o replay valida estado das páginas XMem (COPY_DEST/COPY_SOURCE), aceita subrecurso ≠ 0 apenas para texturas resolvedas, limita footprints placed a uma página XMem e recusa footprint sobre textura. `submit_d3d12x_compute_stream_with_xmem_descriptors` passa um mapa VA de imagem T#→recurso committed.
- Validação: `fmt --check`, checks host/MSVC, 157 testes aprovados no workspace (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Limitação desta fatia: o estado de texturas host não é rastreado (a transição é responsabilidade do chamador); não há detiling, MSAA/BC nos footprints e nenhuma validação em GPU/stream/jogo real.

## Fatia concluída — upload de pixels de T# para texturas host

- `readback_xmem_gpu_bytes` lê páginas XMem por endereço GPU com fence, pois T# traz VA console e não alias CPU. Levantamento no XCL confirmou SW_MODE 0 = linhas de 256 bytes; modos XOR exigem tabelas addrlib e foram recusados no código clean-room do XPORT.
- `texture_upload.rs` planeja subrecursos lineares (row pitch 256, cadeia de mips, slices de array/cubo e volumes por mip com profundidade) e recusa swizzle≠0, MSAA, formatos sem largura conhecida e endereço desalinhado.
- `D3d12MemoryBridge::upload_texture_subresources` usa `GetCopyableFootprints`, staging alinhado a 512 bytes, `CopyTextureRegion` e barreiras COMMON→COPY_DEST→COMMON dentro de um submission com fence. `submit_d3d12x_compute_stream_with_xmem_descriptors` chama o planner para cada T# registrado antes do replay.
- Validação: `fmt --check`, checks host/MSVC, 151 testes aprovados no workspace (1 filtrado: `llvm_gfx10_corpus`), Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Limitação desta fatia: apenas SW_MODE 0 é copiado; modos em mosaico, MSAA e formatos block-compressed falham explicitamente. Não houve validação em GPU real, stream real ou jogo real.

## Fatia concluída — descritores host de textura a partir de T#

- Ponto de partida: `console_desc.rs` já decodifica T#, mas não há mapeamento IMG_FORMAT→DXGI, validação de forma ou recursos/SRVs de textura host; V#/S# já viram descritores host.
- Lote 1 concluído: `console_dxgi_format` mapeia IMG_FORMAT→valor DXGI numérico com ordem de canais, `console_component_mapping` codifica DST_SEL em D3D12_ENCODE_SHADER_4_COMPONENT_MAPPING e `ConsoleImageDescriptor::texture_layout` valida forma/mips/cubo e devolve `ConsoleTextureLayout` (dimensões, mips, MSAA, dimensão da view e clamp LOD).
- Lote 2 concluído: `XmemTextureRegistry` lê tabelas T# diretamente dos bytes da heap console e expõe `addresses()`/`total_entries()`; slots V#/vazios são recusados.
- Lote 3 concluído: `texture_resource_desc`/`texture_srv_desc` constroiem `D3D12_RESOURCE_DESC` e SRV para 1D/2D/3D/cubo/MSAA com array, mips, LOD e component mapping; `D3d12XmemDescriptorMapper::register_textures` cria recursos committed e grava os SRVs no mesmo heap dos buffers, expondo `texture_resource(address, index)` para upload futuro.
- Lote 4 concluído: `XmemDescriptorTables` agrupa os três registries; `submit_d3d12x_compute_stream_with_xmem_descriptors` dimensiona o heap para buffer+T# e registra os dois, e `submit_d3d12x_compute_stream_with_xmem_tables` delega com registries vazios. 26 testes no xport-gpu passaram.
- Lote 5 concluído: documentação (README, arquitetura, WORKLOG, THIRD_PARTY Lote 59) atualizada. Validação: 26 testes no xport-gpu (148 aprovados no workspace, 1 ignorado: `llvm_gfx10_corpus`), `fmt --check`, checks host/MSVC, Clippy Windows `-D warnings` e `xtask policy` (`policy: ok`) limpos.
- Limitação desta fatia: texturas são alocadas com forma/formato corretos, mas os pixels não são copiados de XMem (tiling/swizzle pendente); não houve validação em GPU real.

## Fatia anterior — parser de descritores do console

- Ponto de partida: `XmemDescriptorTableRegistry` exigia que o runtime descrevesse as views; os formatos de 32 bytes (T#/V#) e 16 bytes (S#) do console não eram lidos.
- Lote 1 concluído: `console_desc.rs` implementa um leitor clean-room de slots V#, T# e S# com as posições de bits GFX10, validação de tamanho de slot e testes de words conhecidos; as posições foram conferidas com `desc.rs` do XCL apenas como referência documental.
- Lote 2 concluído: `XmemBufferView::from_console` converte V# de stride 0/4 em view raw; `XmemDescriptorTableRegistry::insert_console_table` lê uma tabela V# diretamente dos bytes da heap e `XmemSamplerRegistry` faz o mesmo para S#. T# e slots vazios são recusados explicitamente.
- Lote 3 concluído: `D3d12XmemSamplerMapper` cria heap SAMPLER shader-visible, grava `D3D12_SAMPLER_DESC` convertido e devolve handles GPU por tabela; `submit_d3d12x_compute_stream_with_xmem_descriptors` liga os dois registros ao replay, e o entry point anterior delega com registry vazio.
- Validação: 20 testes no xport-gpu (142 aprovados no workspace, 1 ignorado), Clippy Windows limpo, `fmt`, checks host/MSVC e `policy: ok`.
- Limitação desta fatia: T# é apenas decodificado — falta mapear IMG_FORMAT→DXGI e criar recursos de textura host. A intenção de comparação do shader não está no S#; o mapper assume comparação quando o stage é não nulo. Sem validação em GPU real.

## Fatia anterior — ligação das janelas ao replay e ABI

- Ponto de partida: `D3d12SparseWindows` já cria/mapeia janelas, e `submit_translated_compute_dispatches` já recebe `&[ID3D12Resource]` de janelas mas exige que o runtime forneça os VAs e o base UMA.
- Lote 1 concluído: `d3d12x.rs` agora empacota parâmetros inteiros em s4..s15 e coloca o restante numa tabela cujo endereço vai em `io[0..1]` (`spilled_arguments`); constantes `XPORT_KERNEL_SGPRS`/`XPORT_IO_UMA_BASE_HI` exportadas; stream sintético decodificado e empacotado em 3 testes novos.
- Lote 2 concluído: `D3d12SparseWindows` expõe `window_resources()`, `window_gpu_addresses()` e `uma_base_hi()`; `write_argument_table` materializa `spilled_arguments` em tiles da janela 0 e devolve o VA do console; `D3d12MemoryBridge::submit_translated_compute_dispatches_with_windows` sincroniza páginas, grava spills e submete com `u2..u5` (fences em cada etapa). Teste de ponta a ponta offline (layout→ABI→tabela) adicionado.
- Limitação desta fatia: tabelas de argumentos e páginas XMem não podem ocupar o mesmo tile (erro explícito); ainda não há cópia GPU→XMem nem validação em hardware.

## Fatia anterior — mapper de descriptor tables para XMem

- Ponto de partida: `submit_d3d12x_compute_stream_with_tables` já aceita heaps shader-visible e um mapeador fornecido pelo runtime, mas não sabe resolver descritores para buffers XMem; `D3d12SparseWindows` já fornece janelas `u2..u5`.
- Lote 1 concluído: `descriptor.rs` define `XmemBufferView` (SRV/UAV raw sobre uma página de 64 KiB, validação de alinhamento/intervalo) e `XmemDescriptorTableRegistry` (endereço console→entradas), com 2 testes novos.
- Lote 2 concluído: `d3d12_descriptor.rs` cria uma heap CBV/SRV/UAV shader-visible, materializa cada view como buffer raw `R32_TYPELESS` sobre o espelho da página XMem e devolve o handle GPU do primeiro descritor de cada tabela; `submit_d3d12x_compute_stream_with_xmem_tables` liga registro + replay e preserva heaps de sampler.
- Testes offline adicionados: decode→registro de tabela e aritmética de handles; `cargo test -p xport-gpu` passou com 12 testes e Clippy Windows limpo.
- Limitação desta fatia: o descritor de tabela não interpreta o formato de 32 bytes do console (o runtime descreve as views); páginas precisam estar registradas e preparadas antes do replay; não houve validação em GPU real.
- Validação concluída: suíte completa (134 aprovados, 1 ignorado), Clippy Windows limpo e `policy: ok`; README/arquitetura/WORKLOG/proveniência atualizados.

## Fatia anterior — janelas shader esparsas

- Estudei `sparse.rs`/`uma.rs` do XCL (buffers reservados de 4 GiB, pool de tiles de 64 KiB, `UpdateTileMappings` e upload de páginas sujas) e confirmei que a VIctory usa memória unificada CPU/GPU, sem sparse resources.
- `crates/xport-gpu/src/window.rs`: layout das janelas (`u2..u5`), tradução GPU VA→(janela, tile) e testes.
- `crates/xport-gpu/src/d3d12_window.rs`: buffers reservados, tile pools de 64 MiB, mapeamento e desmapeamento via `UpdateTileMappings`, cópia dos espelhos XMem para os tiles e cerca por fence. A origem do desenho está registrada em THIRD_PARTY.md.

## Verificação desta etapa

`cargo fmt --all -- --check`, `cargo check --workspace`, `cargo check --workspace --target x86_64-pc-windows-msvc`, `cargo test --workspace -- --skip llvm_gfx10_corpus` (134 aprovados, 1 ignorado), `cargo clippy --workspace --all-targets --target x86_64-pc-windows-msvc -- -D warnings` e `cargo run -p xtask -- policy` passaram (`policy: ok`). O teste `llvm_gfx10_corpus` falha nesta máquina porque `tests/data/llvm-amdgpu` não existe no XPORT (o corpus está apenas na base XCL e não foi importado por ser material de terceiros). Não houve execução em GPU real nem validação de streams/jogos.
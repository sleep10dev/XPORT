# XPORT

XPORT é uma camada de compatibilidade Xbox Series X|S → Windows, construída
para combinar a arquitetura de loader da **VIctory** com a cobertura de APIs e
os caminhos de validação do **XCL**, mantendo uma base própria e incremental.

> **Status:** fundação de plataforma em desenvolvimento, com tradução parcial de
> shaders compute e replay D3D12.X conservador. Ainda não há compatibilidade de jogos
> demonstrada. Um título sintético já executa XAsync/XUser/XAudio2 pelo HLE, e um
> container Xbox DXBC sintético é traduzido, compilado e despachado em GPU Windows
> real, sem implicar compatibilidade com D3D12.X/RDNA2.

## O que existe

- Workspace Rust multi-crate, configuração e política para manter dados de jogos fora do repositório.
- Parser PE32+ e leitor de import libraries.
- Loader Windows com preparação da imagem, relocations, imports, TLS, DLLs do título e relatório de falhas.
- CLI xport info, inspect e run; run faz binding sem executar por padrão.
- Título sintético somente-fonte com export `xport_synthetic_run`: o teste de HLE executa XAsync/XUser/XAudio2 e o replay sintético via exports `XPortGpuReplay*` no processo do XPORT, sem chamar o CRT entry. O stream encadeado com dois pipelines é ligado a dispatches traduzidos e readback de XMem pelo próprio runtime.
- Container Xbox DXBC sintético (`synthetic_xbox_compute_blob`) exercita o parser, a tradução RDNA2→HLSL, DXC, PSO, dispatch e readback em GPU real sem usar assets de jogos.
- O bridge D3D12 rastreia estados de texturas host por subrecurso e valida barreiras/cópias; filas externas do mesmo device podem ser importadas e ordenadas com fence waits GPU-side ao redor do replay. O módulo `xgameruntime` expõe uma ABI sintética autoral (`XPortGpuReplay*`) para sessões opacas, decode offline de streams encadeados com múltiplos pipelines, resumo dos registros, criação/importação de filas, registro de PSOs sintéticos, submissão dos dispatches traduzidos decodificados e readback de XMem — não é GDK/D3D12.X.
- Resolução de imports entre HLE, APIs Windows permitidas e DLLs do título.
- HLE parcial para APIs compartilhadas de Windows, kernelx e xmem; XMem aceita hooks do título por xmem.dll e kernelx.dll.
- Um VFS isolado e relatório JSON; os exports de arquivo do HLE já traduzem unidades console (`G:\`, `\\?\G:\`) via `MountTable`: leitura no game root, escrita no overlay e recusa explícita quando o overlay não existe. O CLI configura os mounts a partir de `Config`: `G:`/`GAME:` no `game_dir`, `T:` no cache, `S:` no storage persistente e escritas no overlay do `data_dir`.

## O que falta para executar jogos

- O `QueryApiImpl` de XGameRuntime já serve áreas offline de XGame (title id, features, invites e eventos), XSystem (identidade e analytics), Persistent Local Storage (`S:\`, path/quota e prompts determinísticos via XAsync), XAsync/XTaskQueue e XUser (um usuário local, gamertag e privilégios); o registro COM vem do VIctory e os GUIDs/slots são validados pela tabela GDK do XCL. Tokens/assinatura falham explicitamente, `MountForPackage` e as demais áreas são recusadas.
- XMem aloca páginas físicas de 64 KiB; o xport-gpu acompanha aliases CPU e GPU VA e retém espelhos até ambos serem removidos. No Windows, cria janelas D3D12 reservadas de 4 GiB (`u2..u5`) com pool de tiles de 64 KiB: `D3d12SparseWindows::sync_xmem_pages` mapeia os GPU VAs registrados e copia os espelhos para os tiles antes do dispatch. `submit_translated_compute_dispatches_with_windows` completa o caminho: sincroniza páginas, grava argumentos spillados na janela 0 (`io[0..1]`) e submete o dispatch com `u2..u5` e `io[112]`. O runtime ainda deve fornecer os GPU VAs/base UMA reais; não há cópia de GPU→XMem.
- xport-shader traduz um subconjunto completo de compute RDNA2 de containers Xbox DXBC para HLSL e, no Windows, usa DXC para compilar DXIL. xport-gpu cria uma root signature interna fixa com u0 (io), u1 (res) e u2..u5 (mem), constrói o PSO e empacota argumentos: parâmetros inteiros em s4..s15, excedentes numa tabela em memória (`io[0..1]`), e envia dispatches diretos em lote. `console_desc.rs` lê slots V#/T# (32 bytes) e S# (16 bytes) no formato GFX10; `XmemDescriptorTableRegistry`/`XmemSamplerRegistry`/`XmemTextureRegistry` convertem slots V#/S#/T# em views raw, samplers e texturas host, e `submit_d3d12x_compute_stream_with_xmem_descriptors` materializa V#/T# num heap CBV/SRV/UAV e S# num heap SAMPLER, ligando os handles ao replay. T# vira recurso committed com SRV (formato, dimensões, mips, cubo e MSAA); T# lineares (SW_MODE 0) têm os pixels copiados de XMem via `GetCopyableFootprints`/`CopyTextureRegion`, enquanto modos em mosaico, MSAA e formatos BC falham explicitamente; o runtime deve preparar `res`/recursos de saída. MIMG/texturas no shader, VALU16 packed, gráficos e indirect dispatch são recusados explicitamente. O decoder também entende os markers de replay `0x8C0F_E005/E007/E008/E00A` para UAV global, arrays de barreiras, `CopyBufferRegion` e `CopyTextureRegion`; o recorder materializa transições, UAV/aliasing e cópias entre páginas XMem e de texturas, validando estados e resolvendo VAs console. Esses markers são o formato sintético do console-list shim, não uma ABI console comprovada.
- O HLE de `xaudio2_9.dll`/`xaudio2_9redist.dll` cria o XAudio2 2.9 do host com flags console-only mascaradas; PCM/IEEE float passam direto e formatos comprimidos falham explicitamente no host, sem decodificação simulada. Opus/XMA, AcpHal/SHAPE, captura e input ainda faltam. `FindFirstFile*` devolve uma listagem mesclada do game root e do overlay, com o overlay vencendo em nomes repetidos; a abertura/leitura continua no game root. O storage responde path/quota e prompts XAsync, mas ainda não há `MountForPackage`.
- Não há uma execução de título real registrada como **compatível com renderização**. O build verificado `game_scarlett_master.exe` (`b99f876b...`) abre e executa de forma estável (janela `Americas`, 37–41 threads, 300 s sem crash, binding `172 hle, 415 host, 0 title, 0 missing`, sem variáveis de diagnóstico ou rede), mas a janela fica branca: `PresentX` e `ExecuteCommandListsX` são stubs e os recursos D3D12.X são buffers zerados. O título usa `d3d12_xs.dll` e não há pipeline gráfico (VS/PS/draw) nem swapchain host ligada; a tradução de shaders cobre apenas compute. A execução atual valida loader/HLE/estabilidade, **não** imagem.

## Próximos passos

1. Fornecer GPU VAs/base UMA do runtime e validar o replay com um stream/jogo conhecido em GPU real.
2. Validar o entry CRT/`--execute` com um runner isolado, sem risco de encerrar o harness.
3. Repetir o caminho com streams mais longos vindos de um título sintético com múltiplos recursos e sincronização de filas, mantendo o runner isolado do CRT.

## Estrutura

    XPORT/
      apps/xport-cli/        # info, inspect, run
      crates/
        xport-core/          # configuração, logging e política
        xport-pe/            # parsing PE/imports
        xport-loader/        # mapear, vincular imports, TLS e falhas
        xport-vfs/           # overlay e mapeamento de caminhos (a integrar)
        xport-hle/           # APIs Windows compartilhadas e APIs de console parciais
        xport-gpu/           # backend gráfico/replay D3D12.X
        xport-shader/        # tradução e cache de shaders
        xport-formats/       # texturas, containers e codecs
        xport-report/        # report.json e diagnósticos
      docs/
      tests/fixtures/
      xtask/

## Comandos

    cargo run -p xport-cli -- info
    cargo run -p xport-cli -- inspect <arquivo.exe|.dll> --report report.json
    cargo run -p xport-cli -- run <arquivo.exe> --report report.json
    cargo run -p xport-cli -- run <arquivo.exe> --execute
    cargo run -p xtask -- policy
    cargo test --workspace

A opção --execute inicia código do título no processo do XPORT. A resolução de
imports e o carregamento já existem, mas isso não significa que APIs de
runtime ou gráficos estejam completas.

## Licença

**GPL-3.0-or-later**. Veja LICENSE e docs/DECISIONS.md.
O histórico de origem e as licenças de material adaptado ficam em
THIRD_PARTY.md.
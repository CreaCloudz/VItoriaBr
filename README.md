# VItoriaBr

> 🇧🇷 **Português** | [🇺🇸 **English below**](#english)

Camada nativa de execução em Windows para **Grand Theft Auto VI (Xbox Series X|S)**, rodando diretamente a partir dos arquivos que o próprio usuário possui.

O código x86-64 original do título executa inalterado dentro do nosso processo; o **VItoriaBr** fornece a plataforma Xbox/GDK por baixo e traduz os dados e shaders do console (D3D12.X / GFX10 → D3D12 / DXIL). Veja [docs/architecture.md](docs/architecture.md).

> **Base e licença:** Fork/continuação baseada no projeto open-source [Marvisak/VIctory](https://github.com/Marvisak/VIctory), mantido sob a licença **GPL-3.0-or-later** ([LICENSE](LICENSE)). **Este repositório não contém nem distribui nenhum arquivo de jogo** — todos os caminhos de jogo, cache e dados devem ficar fora do repositório ([docs/asset-policy.md](docs/asset-policy.md)).

---

## Status e Progresso (2026-11-24)

**Ainda não jogável no mundo 3D.** O título inicia, roda os menus e vídeos a 30 fps cravados, salva perfil/configurações e carrega o mundo 3D, onde ainda ocorrem descartes de shaders não traduzidos e timeouts (TDR) em submissões pesadas de `DispatchMesh`.

### Testes reais em hardware (NVIDIA Blackwell)
Testado em: **Windows 11 Pro (build 26300)** · **Ryzen 7 5700X** · **40 GB RAM** · **NVIDIA RTX 5060 Ti 16 GB** (SM 6.6 + Mesh Shaders, driver `621.24`), sobre dump local Xbox Series `1.0.2.1`.

| Estágio | Estado |
|---------|--------|
| Boot, telas iniciais (acessibilidade, idioma PT-BR, configurações, legal) | ✅ Funciona — **30 fps** |
| Vídeo de introdução com áudio | ✅ Funciona (render limpo) |
| Tela inicial, seletor de conta, perfil e saves persistentes | ✅ Funciona |
| Tela de loading entrando no mundo (~50 s) | ✅ Funciona |
| Mundo 3D (intro da primeira missão) | ⚠️ Alcançado com `DispatchMesh` e draws indiretos; ainda dá `DXGI_ERROR_DEVICE_HUNG` (TDR) em listas pesadas na GPU real |

### Cobertura de Shaders & PSOs medida na RTX 5060 Ti (`1.0.2.1`)
Medido com `victory translate --validate --pipelines` (`dxil.dll` Windows SDK `10.0.26100.0` + driver NVIDIA `621.24`):
- **Containers analisados:** `26.599` (`26.148` programas traduzidos para IR e 100% estruturados — `40,6 milhões` de instruções IR).

| Estágio | DXIL emitidos | Validados (`dxil.dll`) | PSOs aceitos (RTX 5060 Ti) | Taxa (PSOs / emitidos) |
|---------|---------------|------------------------|----------------------------|------------------------|
| **Vertex + Pixel** | `12.284 / 14.013` | `12.184` | **`12.284`** | **100,0%** |
| **Mesh** | `8.191 / 8.506` | `8.129` | **`8.191`** | **100,0%** |
| **Compute** | `1.693 / 2.581` | `1.670` | **`1.372`** | **81,0%** |
| **Total** | **`22.168 / 25.100`** | **`21.983`** | **`21.847`** | **98,6%** |

> Os únicos `321` PSOs recusados pela RTX 5060 Ti são compute shaders que exigem **wave64** (GPUs NVIDIA são wave32).

### Melhorias já aplicadas no VItoriaBr
- **Rastreamento completo de estado de `Shadow` buffers no `HostList`** (`crates/victory-gpu/src/{shadow,host,indirect,draw}.rs`):
  - Antes, `indirect::done()` deixava `title buffer` compartilhados presos em `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`, causando erros de validação em `IASetVertexBuffers` e `id=527` no D3D12 Debug Layer.
  - Agora cada `Shadow` buffer rastreia seu estado via `prelude()` e transições intra-lista (`GENERIC_READ` para VB/IB/CBV/SRV/Indirect `args`+`counter`, `UNORDERED_ACCESS` para UAV/clamp), **zerando os erros de barreira no D3D12 Debug Layer** e estendendo significativamente a execução dentro do mundo 3D na RTX 5060 Ti.

> **Aviso de GPU (TDR):** entrar no mundo 3D em uma GPU real ainda pode disparar reset do driver de vídeo (TDR). Para inspecionar sem risco de travar o driver, use o perfil `diag` com `VICTORY_D3D12_WARP=1` (adaptador em software).

---

## Como compilar e executar

**Requisitos:** Windows 10/11 x64, CPU com AVX2, GPU Direct3D 12 com Shader Model 6.6 + Mesh Shaders, Rust (`rust-toolchain.toml` — toolchain `x86_64-pc-windows-msvc`), VS 2022 MSVC e Windows SDK.

> O título exige vários GiB de memória commit livre no Windows. Se `crashes.log` reportar erro `1455`, feche outros programas ou aumente o arquivo de paginação.

```powershell
# 1. Configurar caminhos (sempre FORA do repositório)
copy victory.example.toml victory.toml
# Edite victory.toml com game_dir, cache_dir e data_dir

# 2. Validar configuração e testes do workspace (não precisa de dados do jogo)
cargo run -p victory -- config
cargo test --workspace

# 3. Compilar em release
cargo build --release -p victory
.\target\release\victory.exe run <pasta_do_jogo>\<executavel_do_titulo> --execute
```

### Build de diagnóstico (`diag`)
```powershell
cargo build --profile diag -p victory --features diagnostics
```
Principais variáveis de ambiente no `diag`:
- `VICTORY_D3D12_WARP=1` — força adaptador WARP em software (lento, não causa TDR)
- `VICTORY_D3D12_DRED=1` — grava breadcrumbs DRED em `crashes.log` se ocorrer `DEVICE_HUNG`
- `VICTORY_D3D12_DEBUG=1` / `VICTORY_D3D12_GBV=1` — D3D12 Debug Layer e GPU-Based Validation
- `VICTORY_CAPTURE=<pasta>` / `VICTORY_CAPTURE_FRAMES=<lista>` / `VICTORY_DUMP_TARGETS=1` — salva frames PNG e render targets
- `VICTORY_SHADER_LOOP_LIMIT=<n>` — limita iterações de loops em shaders traduzidos
- `VICTORY_SKIP_PIPELINES=<hash>,...` — pula draws/dispatches de hashes específicos

### Controles (teclado quando não há controle XInput conectado)
| Teclas | Controle |
|--------|----------|
| `W A S D` / `I J K L` | Analógico esquerdo / direito |
| Setas | D-pad |
| `Enter`, `Backspace`, `X`, `Y` | A, B, X, Y |
| `Q` / `E`, `Z` / `C` | LB/RB, LT/RT |
| `F` / `H` | L3 / R3 |
| `Esc`, `Tab` | Menu, View |

---

<a id="english"></a>

## English

### Overview

**VItoriaBr** runs **Grand Theft Auto VI (Xbox Series X|S)** natively on Windows from files the user supplies.

The title's x86-64 code runs unchanged inside our process; **VItoriaBr** provides the underlying console platform and translates shaders/resources (D3D12.X / GFX10 → host D3D12 / DXIL). See [docs/architecture.md](docs/architecture.md).

> **Upstream & License:** Derived from [Marvisak/VIctory](https://github.com/Marvisak/VIctory) under **GPL-3.0-or-later** ([LICENSE](LICENSE)). **No game files or derived assets are shipped or allowed in this repository** ([docs/asset-policy.md](docs/asset-policy.md)).

### Status & Progress (2026-11-24)

**Not playable in the 3D world yet.** Boot, menus, intro video with audio, profile/saves, and loading into the world work smoothly at 30 fps; inside the 3D world, untranslated shaders are still dropped and heavy `DispatchMesh` submissions can trigger a GPU TDR on real hardware.

#### Real Hardware Test Results (NVIDIA Blackwell)
Tested on: **Windows 11 Pro (build 26300)** · **Ryzen 7 5700X** · **40 GB RAM** · **NVIDIA RTX 5060 Ti 16 GB** (SM 6.6 + Mesh Shaders, driver `621.24`), Xbox Series build `1.0.2.1`.

| Stage | Status |
|-------|--------|
| Boot & first-boot screens (accessibility, language, settings, legal) | ✅ Works — **30 fps** |
| Intro video with audio | ✅ Works |
| Start screen, account selector, profile & persistent saves | ✅ Works |
| Loading into the world (~50 s) | ✅ Works |
| 3D world (first mission intro) | ⚠️ Reached with mesh shaders & indirect draws; still hits `DXGI_ERROR_DEVICE_HUNG` (TDR) on heavy lists on real GPU |

#### Measured Shader & PSO Coverage on RTX 5060 Ti (`1.0.2.1`)
Measured via `victory translate --validate --pipelines` (`dxil.dll` Windows SDK `10.0.26100.0` + NVIDIA driver `621.24`):
- **Containers scanned:** `26,599` (`26,148` programs translated & 100% structured — `40.6M` IR instructions).

| Stage | DXIL Emitted | Validated (`dxil.dll`) | Host PSOs (RTX 5060 Ti) | PSO Rate |
|-------|--------------|------------------------|-------------------------|----------|
| **Vertex + Pixel** | `12,284 / 14,013` | `12,184` | **`12,284`** | **100.0%** |
| **Mesh** | `8,191 / 8,506` | `8,129` | **`8,191`** | **100.0%** |
| **Compute** | `1,693 / 2,581` | `1,670` | **`1,372`** | **81.0%** |
| **Total** | **`22,168 / 25,100`** | **`21,983`** | **`21,847`** | **98.6%** |

> The `321` refused Compute PSOs are wave64-only shaders (NVIDIA GPUs run wave32).

#### VItoriaBr Improvements
- **Full `Shadow` buffer D3D12 resource state tracking in `HostList`** (`crates/victory-gpu/src/{shadow,host,indirect,draw}.rs`):
  - Previously, `indirect::done()` left shared title buffers stuck in `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`, causing `IASetVertexBuffers` and barrier mismatch (`id=527`) errors in the D3D12 Debug Layer.
  - Each `Shadow` buffer now tracks its state across lists via `prelude()` and intra-list barriers (`GENERIC_READ` for VB/IB/CBV/SRV/indirect `args`+`counter`, `UNORDERED_ACCESS` for UAV/clamp), **eliminating all buffer state errors in the D3D12 Debug Layer** and significantly extending 3D world execution on the RTX 5060 Ti.

> **Warning:** Entering the 3D world on a real GPU can still trigger a display driver reset (TDR). Use a `diag` build with `VICTORY_D3D12_WARP=1` to inspect safely on the software adapter.

### Quick Start

```powershell
copy victory.example.toml victory.toml   # set paths outside the repository
cargo run -p victory -- config
cargo test --workspace
cargo build --release -p victory
.\target\release\victory.exe run <game_dir>\<title_exe> --execute
```

Diagnostics build:
```powershell
cargo build --profile diag -p victory --features diagnostics
```

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).

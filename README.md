# Mighty Final Fight (NES) – Enemy & Stage Editor

> 🇧🇷 **Português** | 🇺🇸 [English below](#-english)

Editor visual web (arquivo único `mff_editor.html`, sem dependências) e documentação técnica de engenharia reversa (6502 / MMC3) para o jogo **Mighty Final Fight** de NES.

## ✨ Funcionalidades

- **Mapa Visual Interativo (Drag-and-Drop):** Arraste inimigos, itens, armas e chefes diretamente nas telas de cada fase, com **sprites reais extraídos da própria ROM** em tempo real.
- **105 Tipos Mapeados (`0x00–0x68`) e 15 Seções (`Fase 1-1` a `5-3`):** Todos os inimigos, variantes de comportamento, níveis, HP, itens, armas e chefes identificados.
- **Limite de Inimigos Simultâneos + Fila Automática:** Configure até **6 inimigos vivos ao mesmo tempo** na tela. Se você posicionar mais inimigos na mesma tela (ex.: 7 inimigos com limite de 4), os 4 primeiros aparecem e os outros 3 entram automaticamente conforme os vivos são derrotados.
- **Extensão de Fase (Seção 0):** Aumente a quantidade de telas da Fase 1-1 (até 16 telas) com atualização ao vivo no mapa visual.
- **Gerador de Ondas:** Gere grupos de inimigos automaticamente com espaçamento e rodízio de tipos configuráveis.
- **Salvar e Continuar Depois:**
  - Abra tanto a ROM original quanto uma **ROM já editada** (`MFF_editada.nes`).
  - Exporte e importe suas edições em arquivo leve `.json`.
  - **Auto-Save** no navegador (`localStorage`) com botão de restaurar última sessão.
- **Bilíngue (PT / ENG):** Alterne toda a interface e os nomes de inimigos/fases entre Português e Inglês com um clique.

## 🚀 Como Usar

1. Abra o arquivo `mff_editor.html` em qualquer navegador moderno (não precisa instalar nada nem de servidor).
2. Carregue sua ROM `Mighty Final Fight (U) [!].nes` (262.160 bytes, mapper MMC3) — ou uma `MFF_editada.nes` gerada anteriormente.
3. Escolha a seção, arraste ou adicione inimigos no mapa visual, ajuste o limite simultâneo e as telas.
4. Clique em **3) Aplicar e baixar ROM** para baixar `MFF_editada.nes` pronta para jogar no emulador.

## 📄 Documentação Técnica

Consulte [`Mighty_Final_Fight_documentacao.md`](./Mighty_Final_Fight_documentacao.md) para todos os detalhes de engenharia reversa (mapa de RAM, tabelas de bancos PRG/CHR, estrutura de slots de objetos e patches 6502).

---

## 🇺🇸 English

Single-file web visual editor (`mff_editor.html`, zero dependencies) and 6502/MMC3 reverse-engineering documentation for **Mighty Final Fight** (NES).

### ✨ Features

- **Interactive Drag-and-Drop Visual Map:** Drag and position enemies, items, weapons, and bosses across stage screens with **real sprites rendered directly from the loaded ROM's CHR banks**.
- **All 105 Object Types (`0x00–0x68`) & 15 Stage Sections (`Stage 1-1` to `5-3`):** Full catalog of enemy behaviors, levels, HP, items, weapons, and bosses.
- **Max Simultaneous Enemies & Automatic Spawn Queue:** Increase the active enemy limit up to **6 on screen at once**. If you place more enemies on a screen than the active limit (e.g., 7 enemies with a max of 4), 4 spawn initially and the remaining 3 automatically enter one by one as enemies are defeated.
- **Stage Extension (Section 0):** Expand Stage 1-1 up to 16 screens with live visual map preview.
- **Wave Generator:** Quickly populate waves of enemies with custom spacing and round-robin types.
- **Save & Resume Anytime:**
  - Load either a clean original ROM or an **already-edited ROM** (`MFF_editada.nes`).
  - Export and import lightweight `.json` project files.
  - Browser **Auto-Save** (`localStorage`) with one-click session restore.
- **Bilingual UI (PT / ENG):** Switch the entire interface between Portuguese and English at the top of the page.

### 🚀 How to Use

1. Open `mff_editor.html` in any modern web browser.
2. Load your `Mighty Final Fight (U) [!].nes` ROM (262,160 bytes, MMC3) — or a previously generated `MFF_editada.nes`.
3. Select a section, drag or add enemies on the visual map, configure the max alive limit and screen sequence.
4. Click **3) Apply & Download ROM** to get your patched `MFF_editada.nes`.

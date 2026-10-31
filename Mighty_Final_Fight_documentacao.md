# Mighty Final Fight (NES) – Documentação da análise e dos patches

Documento gerado a partir da análise da ROM `Mighty_Final_Fight__U_____.nes` (dump do próprio usuário).
Tudo aqui vem de leitura estática do código 6502 e de testes em emulador headless. Onde algo é **inferência** (não confirmado), está marcado como tal.

---

## 1. A ROM

| Item | Valor |
|---|---|
| Tamanho do arquivo | 262 160 bytes |
| MD5 (original) | `a8ad5d885a3e4e70ac083f4589374df8` |
| Header (hex) | `4e45531a 08 10 40 08 …` (iNES 2.0) |
| Mapper | 4 (MMC3) – flags6 = `0x40`, flags7 = `0x08` |
| PRG-ROM | 8 × 16 KB = 128 KB (16 bancos de 8 KB) |
| CHR-ROM | 16 × 8 KB = 128 KB |
| WRAM | `$6000-$7FFF` existe, mas não vi nenhuma variável de jogo ali |

### 1.1 Bancos e mapeamento

MMC3 usa bancos de 8 KB. `$C000-$FFFF` é fixo (bancos 14 e 15). `$8000-$BFFF` troca.

- Em jogo, o loop principal alterna **pares** de bancos através da rotina em `$FF4D`, que lê `$F4` (banco em `$8000`) e `$F5` (banco em `$A000`). Pares observados: `(0,1)`, `(4,5)`, `(8,9)`, `(10,11)`, `(12,13)`. O banco ímpar é sempre o par seguinte do par (`f5 = f4+1`), exceto na carga de dados de fase (ver 4.1).
- **Fórmula de offset no arquivo .nes:** `offset = 16 + banco × 0x2000 + (endereço − base)`, com base `$8000` (banco par), `$A000` (banco ímpar), `$C000` (banco 14), `$E000` (banco 15).
- Os bancos 8 e 9 são contíguos no arquivo, então para `$8000-$BFFF` vale `offset = 0x10010 + (endereço − 0x8000)`.

### 1.2 Loop principal (banco fixo 14)

Em `$C93B` roda um loop que chama, nesta ordem, rotinas em pares de bancos diferentes (jogador, objetos, spawn, IA etc.). Trechos relevantes:

| Endereço | Função |
|---|---|
| `$C957` | seleciona bancos 8/9, chama `$8080` (varredura da tabela de spawn) e `$8900` (ondas por tela) |
| `$C96A` | seleciona bancos 12/13 |
| `$C9B6…` | despachante por estado (`$30`) |
| `$E2A3…$E4DF` | câmera/scroll horizontal |
| `$E521…$E66E` | renderização de colunas do fundo |
| `$F70A` | procura slot livre de objeto (slots 13→3), carry set se não houver |
| `$FF4D` | troca de bancos (usa `$F4`/`$F5`) |

---

## 2. Memória (RAM interna)

### 2.1 Variáveis de jogo (página zero)

| Endereço | Significado |
|---|---|
| `$22` | índice usado em `$C1F8,Y` para escolher o banco de dados da fase (0 = fase 1 → banco 1) |
| `$24` | coluna (0–31) dentro da tela, usada ao desenhar o fundo |
| `$29` | índice da tela do mapa durante o desenho de colunas |
| `$2A` | 3 bits altos do byte de cabeçalho da fase (flags) |
| `$2C` | **índice da última tela** da área (5 bits baixos do cabeçalho). Fase 1 área 0 = 3 |
| `$2D` | tela atual (parte alta do scroll) |
| `$30` | estado/máquina de estados do jogo |
| `$3F` | flag de onda (`$FF` = onda concluída/scroll liberado) |
| `$50` | `$FF` ao cruzar uma tela (modo de luta) |
| `$54` | contador de inimigos derrotados na tela atual |
| `$60` | número da **área** dentro da fase (0, 1, 2) |
| `$61` | número da fase (round) |
| `$67` | índice da **seção de spawn** (0–14) |
| `$72/$73` | ponteiro p/ tabela X baixo da seção atual |
| `$74/$75` | ponteiro p/ tabela X alto |
| `$76/$77` | ponteiro p/ tabela Y |
| `$78/$79` | ponteiro p/ tabela Z |
| `$7A/$7B` | ponteiro p/ tabela de tipo |
| `$9E` / `$9F` | próximo índice de spawn ao rolar para a direita / esquerda |
| `$BB` | trava de scroll |
| `$F4` / `$F5` | bancos pedidos para `$8000` / `$A000` |
| `$F9` | contador de telas (incrementa a cada tela cruzada) |
| `$FC` | scroll horizontal (parte baixa) |

### 2.2 Tabela de objetos (estrutura de arrays, 16 slots)

Cada campo é um array de 16 bytes; o slot é o índice.

| Base | Campo | Notas |
|---|---|---|
| `$0300+s` | estado | bit 7 = slot ativo |
| `$0320+s` | id/tipo do personagem | |
| `$0360+s` | X (baixo) | |
| `$0380+s` | X (alto) | posição de mundo = alto·256 + baixo |
| `$03C0+s` | Y (profundidade na rua) | jogador anda entre `$82` e `$AE` |
| `$03E0+s` | Z (altura) | |
| `$0480+s` | índice da entrada de spawn que criou o objeto | usado para não duplicar |
| `$04F0+s` | **grupo gráfico** do inimigo | grunts comuns: 3, 4 ou 5 |
| `$0560+s`, `$0580+s`, `$05B0+s` | flags/tipo (`$0580 ≥ $80` = inimigo ativo) | |

Slots: **0 = jogador**; **3–13 = objetos criados por spawn** (a busca de slot livre vai de 13 para 3); 14–15 usados para casos especiais.
No jogo original, os dois primeiros inimigos entram sempre nos slots 13 e 12.

---

## 3. Por que só 2 inimigos por vez

Não é limite de slots (há 11 disponíveis). Há duas causas:

1. **Dados da fase.** A tabela de spawn da fase 1 (seção 0) tem apenas 6 entradas, em 3 duplas. Cada dupla tem posições X próximas, então 2 inimigos por vez.
2. **Gerador aleatório de ondas** (`$9000`, banco 8). Ele só cria inimigos quando a contagem de ativos é menor que 2. A contagem é feita em `$96C3` e o limite é o `CMP #$02` em `$96F2`.

Além disso, o jogo carrega gráficos para **no máximo 2 grupos de inimigos** ao mesmo tempo (o código em `$9110…` coleta o grupo `$04F0` de 2 inimigos ativos para decidir o par permitido). Por isso os clones do patch abaixo usam sempre **o mesmo tipo** do inimigo original (mesmo grupo gráfico).

---

## 4. Sistema de spawn por tabela (banco 8)

### 4.1 Ponteiros por seção

15 seções (`$67` = 0…14). Cinco tabelas de ponteiros de 15 posições (lo) + 15 posições (hi), em banco 8:

| Tabela | Lo em | Hi em | Conteúdo |
|---|---|---|---|
| Xalto | `$81FE` | `$820D` | parte alta da posição X |
| Xbaixo | `$821C` | `$822B` | parte baixa da posição X |
| Z | `$823A` | `$8249` | altura (sempre 0 na fase 1) |
| Y | `$8258` | `$8267` | profundidade |
| Tipo | `$8276` | `$8285` | índice do tipo de inimigo/objeto |

Cada ponteiro aponta para uma lista de bytes que termina com `$FF` (na tabela Xalto). Os dados ficam em `$8294–$8441`; a seguir começa a tabela de propriedades dos tipos.

**Fase 1, seção 0 (original):**

| # | X | Y | Tipo |
|---|---|---|---|
| 1 | `$148` | `$88` | `$3D` |
| 2 | `$178` | `$A0` | `$01` |
| 3 | `$248` | `$98` | `$4B` |
| 4 | `$298` | `$AE` | `$20` |
| 5 | `$348` | `$82` | `$64` |
| 6 | `$3D0` | `$90` | `$5D` |

Observação: nas outras seções há entradas com Y = `$30` (tipos `$2F`, `$32–$35`, que parecem inimigos que entram por cima) – **inferência**.

### 4.2 Propriedades dos tipos e catálogo (`$00–$68`)

- Tabela A em `$8442 + tipo×4`: quatro bytes copiados para `$0560`, `$0320` (ID de comportamento/IA), `$0580` (animação/sprite inicial), `$0440`.
- Tabela B em `$85E6 + tipo×4`: `$0490` (HP máximo/atual), **`$04F0` (grupo gráfico)**, `$05B0` (família/espécie do inimigo).
- As 15 seções (`$67 = 0..14`) são mapeadas por fase (`$E161,Y` no banco 15, com 3 áreas por fase):
  - **Seções 0–2:** Fase 1 (Slums: Rua, Cais, Chefe Thrasher)
  - **Seções 3–5:** Fase 2 (Riverfront: Beira-rio, Fábrica, Chefe Katana)
  - **Seções 6–8:** Fase 3 (Old Town: Rua, Bar, Chefe Abigail)
  - **Seções 9–11:** Fase 4 (Industrial Area: Usina, Elevador, Chefe Katana II)
  - **Seções 12–14:** Fase 5 (Bay Area: Parte 1 c/ Abigail, Parte 2 c/ Katana, Chefe Final Belger)
- Nos inimigos comuns (grupos 3–5), `$05B0` define o personagem visual (renderizado via metasprites nos bancos 2/3/10/11 e CHR banks em `$A035/$A036/$A0B5`):
  - `$05B0 = 0` (grp 4, CHR `$18`): **Poison / Roxy**
  - `$05B0 = 1` (grp 5, CHR `$1B`): **J / Two.P**
  - `$05B0 = 2` (grp 3, CHR `$1D`): **Bred / Jake**
  - `$05B0 = 3` (grp 4, CHR `$1F`): **Simons**
  - `$05B0 = 4` (grp 5, CHR `$20`): **El Gado**
  - `$05B0 = 5` (grp 3, CHR `$24`): **Andore**
- **Faixas de IDs (`$00–$68`):**
  - `$00–$01`, `$20–$23`: Inimigos comuns andando Nv.1 (HP 40 / Andore 104)
  - `$02–$09`: Itens (`$02` Maçã, `$03` Carne, `$04` Bolo, `$05` Bebida, `$06` Coração, `$07` Saco de \$, `$08` Moeda 10, `$09` 1UP)
  - `$0A–$0C`: Armas (`$0A` Faca, `$0B` Shuriken, `$0C` Martelo; o jogo ajusta pela arma do herói atual em `$81EA`)
  - `$0D–$10`, `$1D`: Não usados (`RTS`)
  - `$11`: Gerador de Barris Rolantes (`Y = 0`)
  - `$12`, `$1F`: Controlador do Elevador (Fase 4-2) e `$1E`: Seta do Elevador
  - `$13–$17`: Chefes (`$13` Thrasher, `$14` Katana, `$15` Abigail, `$16` Katana II, `$17` Belger)
  - `$18`: Gatilho de Diálogo de Chefe; `$19`: Cena Thrasher fugindo (Fase 1-2); `$1A`: Cena Andore; `$1B`: Evento de cenário Thrasher; `$1C`: Evento/Cura Carne (Fase 5-2)
  - `$24–$29`: Inimigos comuns andando Nv.2 (HP 56 / Andore 104)
  - `$2A–$2F`: Inimigos comuns andando Nv.3 (HP 88 / Andore 104)
  - `$30–$35`: Poison (`$30,$32,$34`) e El Gado (`$31,$33,$35`) pulando do topo (`Y = $30`, Nv.1–3)
  - `$36–$3B`: Inimigos comuns rápidos Nv.3 (HP 40 / Andore 88)
  - `$3C–$41`: J/Two.P agachado (pares) e Bred agachado c/ lata (ímpares), Nv.1–3, HP 48
  - `$42–$47`: Variantes Nv.3 (não usados nas seções originais)
  - `$48–$4D`: Inimigos comuns esperando parados Nv.1 (HP 64 / Andore 104)
  - `$4E–$53`: Inimigos comuns esperando parados Nv.2 (HP 72 / Andore 104)
  - `$54–$59`: Inimigos comuns esperando parados Nv.3 (HP 88 / Andore 104)
  - `$5A–$5C`: Simons pulando do topo (`Y = $30`, Nv.1–3, HP 104)
  - `$5D–$62`: J/Two.P agachado (`$5D,$5F,$61`) e Bred agachado c/ lata (`$5E,$60,$62`), Nv.1–3, HP 48/72/88
  - `$63–$68`: J/Two.P sentado no chão (`$63,$65,$67`) e Bred encostado na parede (`$64,$66,$68`), Nv.1–3, HP 48/72/88

### 4.3 Rotinas (banco 8, com 8/9 mapeados)

| Endereço | Função |
|---|---|
| `$8080` | varre a tabela: ao rolar para a direita usa `$9E`, para a esquerda `$9F` |
| `$80C2` | lê a entrada Y: grava `$00` (X baixo), `$01` (X alto), `$02` (Y), `$03` (Z) e compara com o scroll; só nasce se a diferença de X com o scroll estiver entre 0 e 255 (ou seja, **nasce na borda direita da tela**) |
| `$80E4` | cria o inimigo (checa se a entrada já está viva via `$0480`) |
| `$80F8` | busca slot livre (13→3) e inicializa o objeto |
| chamadas a `$80E4` | em `$8095` (rolagem p/ esquerda) e `$80AE` (p/ direita) |

---

## 5. Gerador aleatório e ondas por tela

- `$8900`: entrada. Chama `$89D7`, que escolhe um tratador por `$61`/`$60`.
- Para fase 1, área 0, o tratador é o de `$8A2C`, que faz `LDY $F9` e pula por uma **tabela de 4 entradas** em `$8A3B` (lo) / `$8A3F` (hi): `$8A43` (nada), `$8A44`, `$8A53`, `$8A62`.
- Cada tratador: se `$54` (mortes) < limite em `$8DC4,Y`, chama o gerador `$9000`; senão chama `$89D2` (`$3F = $FF`, libera o scroll). O tratador da última tela (`$8A62`) também espera todos os inimigos morrerem (`$96FC`).
- Tabela de limites `$8DC4`: `00 03 03 04 00 04 03 01`.
- **Consequência:** com `$F9 ≥ 4`, o `JMP` lê fora da tabela e o jogo corrompe a tela (observado como lixo gráfico ao passar da tela 3 numa fase estendida sem o patch).
- Gerador `$9000`: usa tabelas de posição aleatória (`$964B`, `$966B`, `$968B`) e escolhe tipos respeitando no máximo 2 grupos gráficos.

---

## 6. Mapa do fundo

- O fundo é montado **por colunas** ao rolar. A rotina `$E66E` lê um ID de tela de uma lista:
  - área 0 → `$AE60,Y`, área 1 → `$AE90,Y`, área 2 → `$AEC0,Y` (Y = `$29`, o índice da tela), no banco de dados da fase.
- Cada ID aponta para um bloco de 64 bytes em `$AF00 + ID×$40`.
- **Fase 1 (banco 1):** lista da área 0 = `00 01 02 03`.
- Cabeçalho da área no banco 1: `$AE68` (área 0), `$AE98` (área 1), `$AEC8` (área 2). Bits 0–4 = índice da última tela → `$2C`; bits 5–7 → `$2A`. Valores observados: `$23` (área 0), `$23` (área 1), `$21` (área 2).
- As quatro telas da fase 1 (ID 0–3), vistas a scroll zero: 0 = céu/prédio com OTEL, 1 = prédio cinza "2 ROOM", 2 = céu e loja roxa, 3 = prédio amarelo com escada azul.
- **Emendas naturais** (borda direita de A → borda esquerda de B): 0→1, 2→1 e 2→3 (prédio amarelo → amarelo). Daí a sequência de extensão `0,1,2,1,2,1,2,3`.

---

## 7. Espaço livre encontrado

| Local | Endereço | Tamanho |
|---|---|---|
| Banco 9 | `$A073` | 66 bytes |
| Banco 9 | `$BA18` | 233 bytes |
| Banco 1 | `$BAA4` | 93 bytes |
| Banco 15 (fixo) | `$FD7C` | 23 bytes |
| Banco 8 | – | nenhum |

---

## 8. Patch A – 4 (ou 3) inimigos por vez ("clones")

**Ideia:** quando uma entrada de spawn cria um inimigo comum, cria também um **clone** do mesmo tipo, deslocado.

- Código novo em `$BA18` (banco 9, 144 bytes).
- Redireciona as duas chamadas: `JSR $80E4` → `JSR $BA18` em `$8095` e `$80AE` (bytes `E4 80` → `18 BA`).
- `CMP #$02` em `$96F2` → `CMP #$04` (ou `#$03` na variante de 3).

Rotina (pseudocódigo):

```
hook:  salva Y; conta objetos ativos (slots 3..13); chama $80E4;
       conta de novo; se não aumentou → sai;
       (variante 3: só clona se o índice da entrada for par)
       elig(): inimigo comum? (grupo 3..5 e $05B0 < 6) senão sai;
       $80C2 de novo (relê $00..$03);
       X += $0A;
       se Y está em $82..$AE: Y = Y + $16, e se ≥ $AF: Y −= $2C;
       $80F8 (cria o clone no próximo slot livre)
```

Como a tabela da fase 1 tem 3 duplas, cada dupla vira 4 inimigos (ou 3 na variante de 3).
Chefes e itens são ignorados pelo teste de elegibilidade.

**Resultados medidos (bot automático):** máximo de inimigos ativos observado = 4 (variante 4) e 3 (variante 3), sem crash, gráficos corretos na fase 1.

---

## 9. Patch B – fase 1, área 0 estendida de 4 para 8 telas

| # | Alteração |
|---|---|
| 1 | Lista de telas nova no banco 1 em `$BAA4`: `00 01 02 01 02 01 02 03` |
| 2 | Cabeçalho banco 1 `$AE68`: `$23` → `$27` (última tela = 7) |
| 3 | Banco 15 `$E67A`: `B9 60 AE` (`LDA $AE60,Y`) → `20 7C FD` (`JSR $FD7C`) |
| 4 | Hook em `$FD7C`: `A5 22 D0 04 B9 A4 BA 60 B9 60 AE 60` (se `$22 = 0` lê `$BAA4,Y`; senão lê `$AE60,Y`) |
| 5 | Tabelas de spawn da seção 0 realocadas para o banco 9 com 14 entradas (mais 4 duplas nas telas 4–7), ponteiros atualizados |
| 6 | Despachante de ondas: em `$8A2C` os 5 bytes `A4 F9 B9 3B 8A` → `20 B7 BA EA EA` |
| 7 | Hook em `$BAB7` (banco 9): `A4 F9 F0 13 A5 F9 C5 2C D0 04 A0 03 D0 09 38 E9 01 29 01 18 69 01 A8 B9 3B 8A 60` |

Lógica do hook 7: `Y = $F9`; se `$F9 = 0` → 0; se `$F9 = $2C` (última tela) → 3; senão `Y = 1 + ((F9−1) & 1)` (alterna os tratadores 1 e 2).

**Teste:** o bot percorreu as 8 telas, sem crash, e entrou na área seguinte (`$60 = 1`) normalmente. A área 1 carregou idêntica à versão sem extensão.

Não alterado (e, na prática, sem problema nos testes): em banco 12 `$8DA8`, a checagem `$F9 == 3` que prende inimigos na tela no último trecho continua apontando para a tela 3 (agora tela do meio).

---

## 10. Como a análise foi feita

1. **Emulador headless** `cynes` (Python), que suporta MMC3, com leitura/escrita de memória e `save`/`load` de estado.
2. **Bot** para jogar: segue o inimigo mais próximo, ataca com B, anda para a direita quando não há inimigos visíveis. Registra a contagem de slots ativos por quadro e tira capturas.
3. **Descoberta dos slots:** comparei a RAM com e sem entrada de controle e com inimigos na tela (`$0300–$03FF`).
4. **Descoberta de qual banco estava mapeado:** comparei a memória da CPU em `$8000-$FFFF` com os 16 bancos do arquivo.
5. **Desmontagem:** `py65` com uma imagem de memória montada a partir dos bancos; busca de padrões (`LDX #$0D`, `CMP #$02`, acessos a `$2C`, `$F9`).
6. **Montagem dos patches:** assembler mínimo em Python sobre o `py65`; cada patch foi conferido desmontando a ROM gerada.
7. **Validação:** várias partidas automáticas por ROM, contando inimigos ativos, observando capturas de tela e verificando travamentos.

### Armadilhas encontradas

- Apertar **Start** uma vez a mais pausa o jogo (6 presses em vez de 5 fizeram parecer que a entrada não funcionava).
- Nomear um arquivo de trabalho `dis.py` sobrescreveu o módulo `dis` do Python e quebrou o numpy.
- Na ROM original e nas modificadas, o bot às vezes trava porque um inimigo fica fora da tela à direita; é limitação do bot, não do patch.

---

## 11. Editor (`mff_editor.html`)

Arquivo HTML único e local. Funcionalidades principais:
- **Carregamento de ROM original ou já editada:** Lê tanto a ROM original quanto uma `MFF_editada.nes` já gerada pelo editor, restaurando inimigos, cotas, limites, opções e sequência de telas.
- **Salvar / Abrir projeto (`.json`) e Auto-Save (`localStorage`):** Permite exportar e importar todas as alterações em arquivo `.json` (`mff_alteracoes.json`) ou restaurar a última sessão diretamente no navegador.
- **Mapa visual interativo (Drag-and-Drop) com sprites reais:** Extrai e desenha os sprites diretamente da CHR-ROM (`getSpr`) sobre o mapa das telas da seção, permitindo arrastar cada inimigo e visualizar instantaneamente novas telas adicionadas em `#seq`.
- **Fila de espera da seção (`#lt`), Cotas por tela (`#qt`), Espera de tela limpa (`#wc`) e Entrada rápida de reforços (`#fr`):** Controla o limite simultâneo de inimigos tanto nas ondas aleatórias quanto na tabela da seção, aguardando vagas livres conforme inimigos morrem.

---

## 12. Limitações e itens em aberto

- Os **nomes dos inimigos** não estão na ROM; só se conhecem IDs e grupos gráficos. Dá pra identificar visualmente forçando cada tipo no emulador.
- Só testei a **fase 1**. Não sei a que área do jogo corresponde cada uma das 15 seções.
- A extensão de mapa foi feita **só para a fase 1, área 0**. Outras áreas têm lista, cabeçalho e tratador de ondas próprios, e podem não ter espaço livre.
- Usar tipos de inimigo de **outro grupo gráfico** pode gerar sprites errados.
- O número de mortes que libera o scroll por tela (`$8DC4`) e o comportamento de chefes não foram estudados a fundo.
- Os testes são todos com bot; não houve jogo manual completo.

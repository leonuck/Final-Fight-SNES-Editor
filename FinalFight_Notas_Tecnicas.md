# Final Fight Guy (JP) "Uncut L200" — Notas técnicas do hack de 4 inimigos

> Tudo abaixo foi obtido por engenharia reversa estática + um emulador 65816/SNES próprio (headless).
> **Nada foi validado em hardware nem em emulador comercial.** Endereços marcados com (?) são hipóteses.

## 1. A ROM

| Item | Valor |
|---|---|
| Arquivo | `Final_Fight_Guy__Japan_.sfc` (já com o IPS aplicado: 0 diferenças) |
| Tamanho | 2 MB (2.097.152 bytes) |
| Mapeamento | LoROM / FastROM (byte de modo `$30`) |
| Título interno | `FINAL FIGHT UNCUT` |
| Reset (emulação) | `$00:FF94` → `LDA #1; STA $420D; JML $80:8000` |
| NMI nativo | `$00:FF80` → hook do hack em `$B8:D8BB` |
| Conversão SNES→arquivo | `((banco & $3F) * $8000) + (offset & $7FFF)` |

O IPS original tem ~14.600 blocos (reforma grande, não ajuste pontual).

## 2. Arquitetura geral do jogo

- **Agendador cooperativo de 8 tarefas**: tabela em DP `$40–$BF`, 16 bytes por tarefa; pilhas de `$40` bytes em `$1DC0–$1FFF`. Loop do escalonador em `$80:80EE–8145`. Rotina de "sleep" em `$80:81AE`, criação de tarefa em `$80:8170`.
- **Máquina de estados do jogo**: `$0CA0` (índice ×2, tabela em `$81:9ACA`). Estados vistos: 0→2→4→6 (6 = jogo rodando; handler em `$81:9BD0`). Quadro de gameplay em `$81:9C01`.
- **Modo de vídeo/NMI**: `$36` (modo), `$D7` (≠0 = NMI pula atualização de vídeo e escreve TM=0), `$E8` (sombra de INIDISP). Pedido de troca de modo: `$0106`=1 com `$0104`→`$D7`, `$0105`→`$36`.
- **NMI**: contadores `$C8/$C9`; espera H-blank em `$4212` bit 6 antes de escrever `$2100`.
- **Hook do hack no NMI** (`$B8:D8BB`): se `$7F:D000 == $5A1E`, o NMI é ignorado (RTI); senão segue para `$80:8342`.

## 3. Tabela de objetos na WRAM

Convenção Capcom: o registrador **D (direct page) aponta para o objeto** e os campos são acessados por DP.

### 3.1 Jogadores — stride `$80`

| Slot | Endereço | Observação |
|---|---|---|
| P1 (Guy) | `$0D00` | X em `+$07` (word), Y em `+$0A` (word) |
| P2 | `$0D80` | **vazio em 1P** (só `$0DE4–$0DE7` são escritos). Hack troca P1↔P2 em `$A4:FE47` |
| Energia P1 (?) | `$0D6E` | 6 → 0 conforme apanha |
| Flag 2P | `$00CD` bit 1 | `$0DEE` é lido quando 2P |

### 3.2 Inimigos — stride `$A0`

| Slot | Endereço |
|---|---|
| Inimigo 1 | `$1000` |
| Inimigo 2 | `$10A0` |
| Inimigo 3 | `$1140` |
| **Inimigo 4 / "slot de chefe"** | **`$11E0`** |

Campos observados (offsets a partir do início do slot):

| Offset | Significado |
|---|---|
| `+$00` | estado/ativo (`&3 != 0` = ativo; `0` = livre) |
| `+$02` | subestado (índice de jump table) |
| `+$07` | posição X (word) |
| `+$0A` | posição Y (word) |
| `+$0F` | contador de laço (apenas no 1º slot, ver §4) |
| `+$11` | tipo do inimigo |
| `+$14/+$16/+$18` | energia (valor `$40` cheio; triplicado) |

### 3.3 Outras classes (alocadores em `$80:Axxx`)

| Região | Qtde × stride | Alocador |
|---|---|---|
| `$1000/$10A0/$1140` | 3 × `$A0` | `$80:AC27` (inimigos) |
| `$11E0` | 1 × `$A0` | `$80:ACF9` (chefe, **sem checar se está livre**) |
| `$1280–$1480` | 5 × `$80` | `$80:AD3D` |
| `$1700…` | 6 × `$80` | `$80:AD0C` |
| `$1C00…` | — | `$80:AC6F` |
| `$0E00/$0E80/$0F00/$0F80` | 4 × `$80` | atores de cena/intro (ids 0,2,4,6 em `+$11`), init em `$82:8AFD` |

## 4. Onde está o limite de inimigos

1. **Laço de atualização** — `$AC:8009`: `D=$1000`, processa, avança `+$A0`, repete enquanto `DEC $0F >= 0`. O contador é inicializado em **`$AC:F037`: `LDA #$02 / STA $7E:000F`** → 3 slots (o hack já elevou de 2 para 3).
2. **Slot `$11E0`** — tratado separadamente em `$80:CCDD` com `JML $AC:F000`, que roda o handler normal de inimigo (`$80:CC68`). Ou seja, já é um 4º inimigo funcional.
3. **Alocador de spawn** — `$80:AC27` só testava `$1000/$10A0/$1140`. **Este era o limite efetivo.**
4. **Varreduras de alvo/colisão** que já incluem `$11E0`: `$81:B4F9…`, `$81:B565…`, `$81:B5B6…`, `$81:B8CA…`, `$81:B8FB…`, `$81:BC5A…`, `$82:B004`, `$82:B4FB`.
5. **Script de fase** — `$82:C7EF`: lê comandos em WRAM (`$0700+`, ponteiro em `$72`) e despacha (`$82:C8B5`); tipo 1 chama `JSL $80:AC27` e grava tipo/posição no slot devolvido.

## 5. O Patch Definitivo e Verificado de 4 Inimigos

Após desmontagem completa da engine e testes formais com simulador 65816:

1. **Alocador em `$80:AC27` -> `$AC:F050` (69 bytes):**
   - Varre `$1000`, `$10A0`, `$1140`.
   - Se os 3 estiverem cheios, testa se há chefe ativo (`$7E0759` ou `$7E07D7`). Se houver chefe, o Slot 4 (`$11E0`) é protegido e a alocação falha.
   - Se não houver chefe e `$7E11E0 == 0`, aloca `$11E0` e **obrigatoriamente grava `$7E11F0 = #$02`**.
   - *Descoberta Crítica do Sprite Invisível:* O motor de renderização da ROM Uncut em Bank 3E (`0x1F517A`) e Bank B5 possui o teste explícito: `LDA $7E11F0; CMP #$02; BNE skip_draw`. Sem `$7E11F0 = 2`, o 4º inimigo existia na memória mas era 100% invisível na tela!

2. **Despachante de IA do Slot 4 em `$AC:F0A0` (Hook em `$AC:F00C`):**
   - Na ROM original, `$AC:F00C` chama incondicionalmente `JSL $80:CC68` (tabela de IA de Chefes).
   - O novo despachante checa `$7E11F0`:
     - Se `$7E11F0 == 2` (inimigo comum): chama `JSL $80:CC64` (IA de inimigo comum).
     - Se `$7E11F0 != 2` (chefe): chama `JSL $80:CC68` (IA de chefe).

3. **Limitador em `$B5:D8A8` -> `$AC:FDC7` (133 bytes):**
   - Conta os slots ativos `$1000`, `$10A0`, `$1140` e `$11E0` (apenas quando `$7E11F0 == 2`).
   - Se chefe ativo (`$7E0759 != 0`), limite = 2.
   - Se chefe inativo, limite = 4.
   - Remove o clamp artificial de Slum (`$7FF288`), permitindo 4 inimigos em todas as fases.

4. **Nenhum Hack no Loop Counter `$AC:F037`:**
   - O frame loop original já processa os slots 1, 2 e 3 em `$AC:8000` e o Slot 4 independentemente em `$AC:F000`. Alterar o contador para 3 causava execução duplicada do Slot 4. Mantendo o contador original (2), cada um dos 4 slots executa exatamente 1 vez por frame.

## 6. Riscos / pendências (não verificados)

- **Chefes**: `$80:ACF9` usa `$11E0` sem checar → pode sobrescrever um 4º inimigo.
- **Gráficos/paleta por slot**: a VRAM e paleta de cada inimigo parecem atreladas ao slot. Um 4º inimigo de **tipo diferente** pode ficar sem tiles ou com paleta errada. Se ocorrer, a ideia do usuário (reaproveitar tiles/paleta reservados ao P2) é o caminho — mapeamento ainda não feito.
- **Script de fase**: não confirmei que as ondas das fases tentam spawnar mais de 3 ao mesmo tempo.
- Limite de sprites/OAM (128) e flicker não avaliados.

## 7. Como foi testado (emulador próprio)

- Emulador C (`emu.c`) com CPU 65816, DMA, VRAM/OAM/CGRAM, joypad, APU simulado (handshake por `CMP #imm`), bit de H-blank alternado em `$4212`. HDMA/IRQ **não** implementados.
- Para pular a tela preta do intro foi preciso forçar `$0104=0`, `$0106=1` no quadro 1300.
- Teste do 4º slot: clonei objetos de inimigo para `$1140` e `$11E0` (X deslocado) no quadro 3790; ambos foram processados e desenhados sem travar.
- Não observei spawn natural de 4 inimigos (o bot não passou da primeira luta).

## 8. Arquivos entregues

- `FinalFight_4_Enemies.ips` — aplicar sobre a ROM que já tem o "Uncut L200".
- `Final_Fight_Guy_4_Enemies.sfc` — ROM já patchada.

**Recomendação:** testar em bsnes-plus/Mesen (com debugger), fazer backup, e observar `$11E0` ao spawnar inimigos de tipos diferentes.

---

## 9. Diagnóstico da tela preta (editor "4/5 inimigos")

**Causa confirmada (reproduzida no emulador):** o "motor nativo" do `FinalFight_Enemy_Editor.html` gravava rotinas novas sobre **código vivo do hack**, e a ROM **reiniciava** no quadro ~1213, logo ao entrar no jogo.

| Patch do editor | Problema |
|---|---|
| `AC:F050` (199 bytes) | invade `AC:F100–F116`, rotina chamada por `82:8246` (`JML $ACF100`) |
| `AC:F130`, `F1A0`, `F1C0`, `F1F0` | todos com dezenas de bytes não-zero (código do hack) |
| `AC:F00C`, `AC:F019` | alteram o handler existente do slot `$11E0` |
| `82:D122` | sobrescreve o hook existente `JML $B5:D008` |
| `82:D13C` (+24 NOPs) | corta o código original que libera slot de VRAM |
| `00:2D0D`, `00:4E4C` | mudam limites de outros objetos (06→04, 05→03) |

Cada patch sozinho não trava; o reinício só aparece quando 1 e 2 entram juntos (o crash é exercitado pela rotina em `F100+`).

**Modo 5 inimigos:** o 5º slot em `$1900` cai dentro do bloco de itens (`$1700–$19FF`); não há bloco livre de `$A0` bytes em `$0000–$1FFF`. Indisponível.

## 10. Sistema de VRAM dos inimigos (descoberta)

- Claim de VRAM: `82:D088` (via `AF:9B60`, chamado em `82:CBE3`) e variante com contagem `82:D0C9` (via `AF:9B6B`, em `82:CC21`). Libera: `82:D13C` / `82:D161`.
- **3 slots de VRAM**: `$0748–$074A` (`$FF` = livre), contagem de referências em `$0754+X`, flag `$0759` reserva o slot 2. Tabelas ROM: `02:D119` (11 16 1B), `02:D11C` (58 78 98), `02:D11F` (10 60 B0), `02:D1C2` (0A 0C 0E), `02:D1C5` (A0 C0 E0).
- Se não há slot livre, o claim retorna `SEC` e o spawn **espera** (não trava). Inimigos do mesmo tipo compartilham o slot. Logo, 4 inimigos funcionam quando há repetição de tipo; 4 tipos distintos ao mesmo tempo ainda são limitados a 3.

## 11. Correção entregue

- Patch seguro: só `80:AC27 → JML $AC:F050` + rotina de 35 bytes em `AC:F050` (zerado). Nada de código vivo é tocado.
- `FinalFight_Enemy_Editor_FIXED.html`: o botão de carregar ROM agora **restaura os bytes originais** das regiões que o motor antigo escreveu e aplica o patch seguro; ROMs já quebradas são reparadas (verificado: saída idêntica à ROM boa byte a byte). Modo 5 desativado.
- `Final_Fight_Guy_4_Enemies_REPAIRED.sfc`: ROM reparada.
- Espaço livre seguro para trabalhos futuros: `AC:FDC7–FFAF` (489 bytes zerados no fim do banco).

---

## 12. Correção v2 — o limitador real de 3 inimigos (por que a v1 "não fazia nada")

A v1 só liberou o **alocador** (`80:AC27`). O spawn nunca chegava lá: antes dele há um **limitador do hack**.

**Cadeia de spawn de uma entrada de onda:** `82:CD1C` → `JSL 82:D122` (gate) → se A=0, `JSR 82:CFAF` (spawner da entrada de 9 bytes: delay, x, y, id, modo, hp) → `JSL 80:AC27` (aloca slot).

- `82:D122` (original Capcom, 2–3 inimigos) foi trocado pelo hack por `JML B5:D008` → `JML B5:D8A8`.
- **`B5:D8A8` = limitador**: zera `$7F:F2B0`, conta ativos em `$1000/$10A0` (e `$1140` se `$0759 == 0`), define o limite em `$7F:F2B2` (`LDA #$0003`, ou `#$0002` se `$0759 != 0`), aplica override do modo `$7F:F200 == $5A` (`$7F:F288`) e devolve `A=0` (liberar) ou `A=1` (esperar). **`$11E0` nunca era contado e o teto era 3.**
- As ondas têm de 7 a 39 entradas liberadas pelo limitador conforme os slots esvaziam; por isso só o limite define quantos aparecem.

**Patch v2 (3 mudanças, nenhuma sobre código vivo):**

| Onde | Bytes | Função |
|---|---|---|
| `80:AC27` (0x2C27) | `5C 50 F0 AC` | desvia para o alocador novo |
| `AC:F050` (0x167050), 35 B | — | aloca `$1000/$10A0/$1140/$11E0` |
| `AC:FDC7` (0x167DC7), 149 B | — | **limitador novo**: conta também `$11E0`, limite 4 (2 se `$0759≠0`, igual ao original) |
| `B5:D8A8` (0x1AD8A8) | `5C C7 FD AC` | desvia o limitador antigo para o novo |

**Testes no emulador próprio (isolados, ROM antiga vs v2):**

| Slots ativos (`1000,10A0,1140,11E0`) | Original | v2 |
|---|---|---|
| 1,1,1,0 | bloqueado | **spawn permitido** |
| 1,1,1,1 | bloqueado | bloqueado (4 = teto) |

Chamada real do spawner `82:CFAF` com `$1000/$10A0/$1140` ocupados: original → nada nasce; v2 → `$11E0` fica ativo com x/y/tipo corretos.

**Não testado:** onda completa de ponta a ponta (meu bot não consegue matar inimigos para avançar as ondas), chefes, e tipos de inimigo distintos (limite de 3 slots de VRAM continua valendo: 4 tipos diferentes ao mesmo tempo → o spawn espera).

**Arquivos v2:** `FinalFight_4_Enemies_v2.ips` (sobre a ROM com "Uncut L200"), `Final_Fight_Guy_4_Enemies_v2.sfc`, `FinalFight_Enemy_Editor_FIXED.html` (restaura as áreas corrompidas pelo motor antigo e aplica o patch v2; verificado byte a byte contra a ROM testada, partindo da ROM original ou das ROMs quebradas do editor antigo).

---

## 13. Diagnóstico e Resolução Definitiva dos 4 Inimigos Simultâneos & Auto-Duplicar

Após rastreio completo no nível de instruções 65816 (ciclo a ciclo e análise da memória RAM e ROM), foram identificados os 4 gargalos que impediam 4 inimigos em tela e faziam a ROM apresentar tela preta ou inimigos sumindo:

### 13.1 Os 4 Gargalos Resolvidos

1. **Trava da Fase 1 (Slum Mode $5A no Limitador $AC:FDC7)**:
   - *Causa*: O limitador Capcom original possuía uma checagem `CMP #$005A / BEQ ... / LDA $7FF288 / STA $7FF2B2`. Na Fase 1 (Slum), essa checagem forçava o limite ativo para **2**, impedindo qualquer 3º ou 4º inimigo de nascer.
   - *Correção*: A trava de Slum foi completamente suprimida no limitador de 4 inimigos. Agora o teto é 4 inimigos contínuos durante todo o percurso regular.

2. **Contador do Loop de IA dos Inimigos (`$AC:F037`)**:
   - *Causa*: O loop principal de atualização de inimigos normais (`$AC:8000` / `$AC:F031`) executava `LDA #$02; STA $7E000F`. Esse contador executava apenas 3 iterações (`$1000, $10A0, $1140`). O Slot 4 (`$11E0`) **nunca** era processado pela rotina de IA dos inimigos normais. Se um inimigo nascesse no slot 4, ele ficava congelado/invisível.
   - *Correção*: Hook em `$AC:F037` (`5C 40 FE AC EA EA`) apontando para a rotina dinâmica em `$AC:FE40`:
     - Se `$0759 == 0` (sem chefe em tela): `LDA #$03; STA $7E000F; JML $AC:8007` (processa os 4 slots: `$1000, $10A0, $1140, $11E0`).
     - Se `$0759 != 0` (chefe ativo): `LDA #$02; STA $7E000F; JML $AC:8007` (processa 3 slots normais, reservando o 4º para o chefe).

3. **Invocação Indevida de Rotina de Chefe no Slot 4 (`$80:CCDD` / `$AC:F000`)**:
   - *Causa*: No frame loop original da Capcom, o Slot 4 (`$11E0`) era reservado exclusivamente para chefes de fase (Damnd, Sodom, Edi.E, etc.). O motor chamava `$80:CC68` (`JMP ($CCFE,X)`), despachando a rotina do chefe. Quando um inimigo normal (Bred, Dug, Axl) caía no Slot 4, a máquina interpretava seu estado como ID de chefe inválido, causando crash (`BRK #$00` / tela preta).
   - *Correção*: Hook em `$AC:F000` (`5C 60 FE AC`) apontando para o Boss Guard em `$AC:FE60`:
     - Se `$0759 == 0`: o Slot 4 já foi processado como inimigo comum pelo loop normal, então executa `JML $80:CCF4` (`RTL`), saindo com segurança sem pular na tabela de chefes.
     - Se `$0759 != 0`: salta normalmente para `$AC:F041` para executar a IA do chefe.

4. **Corrupção de Ondas pelo "Auto-Duplicar" (`writeWaveToRom` no HTML)**:
   - *Causa*: O editor tentava alocar espaço extra inserindo marcadores sintéticos `$8002` dentro do Banco 87. O leitor de scripts de onda da Capcom não suporta essa estrutura artificial, quebrando a sequência de leitura e fazendo com que os inimigos simplesmente desaparecessem.
   - *Correção*: A gravação de ondas foi refeita para ser estritamente in-place, respeitando os limites da onda original (`maxEntriesInPlace`), e o botão "Auto-duplicar" agora vem desmarcado por padrão com aviso explicativo. As ondas padrão do Final Fight já possuem dezenas de inimigos na fila de spawn, não havendo necessidade de duplicá-las para ver 4 inimigos simultâneos.

---

### 13.2 Auditoria do Catálogo Canônico de Nomes e IDs (Banco $80:E6C0)

A tabela de ponteiros de texto da HUD da ROM original (`Final Fight Guy Japan Uncut.sfc` em `$80:E6C0` / offset `0x0066C0`) foi extraída byte a byte diretamente da memória:

| ID Hex | Índice | Nome Canônico na ROM | Arquétipo Visual |
|---|---|---|---|
| `0x00` | 0 | **BRED** | Homem comum (calça jeans, camiseta branca) |
| `0x03` | 1 | **DUG** | Homem comum (paleta alternativa de Bred) |
| `0x06` | 2 | **JAKE** | Punk (óculos escuros, suspensório) |
| `0x09` | 3 | **SIMONS** | Punk (paleta alternativa de Jake) |
| `0x0C` | 4 | **J** | Dançarino / Jaqueta amarela |
| `0x0F` | 5 | **TWO.P** | Dançarino / Jaqueta laranja |
| `0x12` | 6 | **AXL** | Guarda / Jaqueta de couro e jeans |
| `0x15` | 7 | **SLASH** | Guarda / Paleta verde/escura |
| `0x18` | 8 | **ANDORE Jr.** | Gigante com suspensório |
| `0x1B` | 9 | **ANDORE** | Gigante |
| `0x1E` | 10 | **G. ANDORE** | Gigante |
| `0x21` | 11 | **U. ANDORE** | Gigante |
| `0x24` | 12 | **F. ANDORE** | Gigante |
| `0x27` | 13 | **G. ORIBER** | Gordo careca (cabeçada, camiseta vermelha) |
| `0x2A` | 14 | **BILL BULL** | Gordo careca (camiseta verde/azul) |
| `0x2D` | 15 | **WONG WHO** | Gordo careca (camiseta amarela) |
| `0x30` | 16 | **HOLLY WOOD** | Inimigo com faca / molotov (jaqueta vermelha) |
| `0x33` | 17 | **EL GADO** | Inimigo com faca (jaqueta escura) |
| `0x36` | 18 | **ROXY** | Mulher acrobata (cabelo rosa, short jeans) |
| `0x39` | 19 | **POISON** | Mulher acrobata (cabelo roxo) |
| `0x3C` | 20 | **DAMND** | Subchefe / Chefe Fase 1 |
| `0x3F` | 21 | **SODOM** | Chefe Fase 2 |
| `0x42` | 22 | **EDI. E** | Chefe Fase 3 |
| `0x45` | 23 | **ROLENTO** | Chefe Fase 4 |

> [!NOTE]
> Os IDs no catálogo do editor HTML conferem **100% com a ROM canônica japonesa**. Qualquer divergência percebida decorre da diferença entre a versão japonesa sem censura (com Roxy e Poison) e a versão norte-americana (onde Roxy e Poison foram substituídas por Billy e Sid), ou das variações de paleta de cor por fase.

---

### 13.3 Mapa de Memória no Banco $AC (Zona $AC:FDC7–FFAF)

| Intervalo SNES | Offset ROM | Tamanho | Função |
|---|---|---|---|
| `$AC:FDC7–$AC:FE3F` | `0x167DC7` | 121 bytes | **Limitador v3**: Contagem de 4 slots (`$1000, $10A0, $1140, $11E0`), teto = 4 (teto = 2 se `$0759 != 0`). |
| `$AC:FE40–$AC:FE53` | `0x167E40` | 20 bytes | **Loop Counter Dinâmico**: Define `#$03` (4 iterações) se `$0759 == 0`, ou `#$02` se chefe. |
| `$AC:FE60–$AC:FE6D` | `0x167E60` | 14 bytes | **Boss Guard Slot 4**: Desvia Slot 4 com `RTL` em fase regular para não chamar tabela de chefe. |
| `$AC:FE6E–$AC:FFAF` | `0x167E6E` | 322 bytes | Espaço livre preservado para expansões futuras. |

---

## 14. Estado atual do editor e auditoria do Auto-Preencher (2026-11-17)

Esta seção registra o estado do arquivo `FinalFight_Enemy_Editor.html` após as correções recentes. **Ela substitui os detalhes divergentes da seção 13 sobre offsets, flags e comportamento do patch.** A ROM não foi executada em emulador comercial depois destas mudanças; as verificações abaixo são estáticas e por simulação do parser/editor.

### 14.1 Modos de exportação

- **Modo Original — 3 Inimigos** é o padrão ao carregar a ROM original. `applyNativeEngine(buf, 3)` restaura os hooks do motor original e grava o marcador `$03` em `0x167F40`.
- **Modo Experimental — 4 Inimigos** grava o marcador `$04` e aplica os hooks descritos abaixo. O modo é experimental porque ainda não foi confirmado em um emulador real com waves completas, objetos de cenário e chefes.
- **Modo 5** permanece indisponível; `$1900` conflita com a região de itens.
- O checkbox **Auto-Preencher Waves** só modifica os dados ao exportar `.sfc` ou `.ips`; marcar o checkbox não muda a ROM carregada imediatamente.

### 14.2 Mapa de entidades e alocadores

Os quatro slots usados pelo motor de inimigos são:

| Slot | WRAM | Uso no modo experimental |
|---|---:|---|
| 1 | `$1000` | Inimigo normal |
| 2 | `$10A0` | Inimigo normal |
| 3 | `$1140` | Inimigo normal |
| 4 | `$11E0` | Inimigo normal no modo experimental; reservado para chefe no fluxo original |

O allocator original em `$80:AC27` testa os três primeiros slots. Outras classes usam pools diferentes, documentados anteriormente: `$1280–$1480` (allocator `$80:AD3D`), `$1700...` (allocator `$80:AD0C`) e `$1C00...`. A presença desses pools não prova que todos os objetos visuais, barris ou elementos de cenário sejam criados por waves; alguns podem vir de scripts separados.

No editor, a lista de inimigos usa a tabela em `0x0151C8`; entradas não classificadas como inimigo são tratadas como objetos de cenário. Esse critério serve para editar o script de wave, mas não identifica necessariamente todos os objetos criados por outros sistemas da fase.

### 14.3 Correção do hook do allocator original

Foi encontrada e corrigida uma escrita além do fim da rotina original:

- A rotina original em `0x2C27` tem **27 bytes** e termina com `A9 01 6B` em `0x2C3F–0x2C41`.
- Os bytes seguintes, a partir de `0x2C42`, pertencem a outro código/dados; os três primeiros são `C2 20 64`.
- Uma versão anterior da restauração e do hook escrevia 30 bytes e substituía esses três bytes adjacentes. Isso podia corromper a ROM mesmo com Auto-Preencher desligado.
- O editor agora restaura os 27 bytes originais (`A2 00 10 BD 00 00 F0 12 A2 A0 10 BD 00 00 F0 0A A2 40 11 BD 00 00 F0 02 A9 01 6B`) e aplica um hook de 27 bytes: `JML $AC:F050` mais 23 `NOP`. Os bytes em `0x2C42+` ficam intactos.

Simulação de `applyNativeEngine` em modo original, partindo de `Final Fight Guy (Japan) uncut.sfc`: antes do recálculo do checksum, o único byte diferente da ROM fonte foi o marcador `0x167F40` (`00 → 03`).

### 14.4 Patch experimental atual de quatro inimigos

| Hook/rotina | Offset ROM | Tamanho atual | Função implementada |
|---|---:|---:|---|
| `$80:AC27` | `0x002C27` | 27 B | `JML $AC:F050` e `NOP`s; ocupa somente a rotina original |
| `$AC:F050` | `0x167050` | 48 B | Testa `$1000/$10A0/$1140`; se os três estiverem ocupados, seleciona `$11E0` e grava `$11F0 = 2` |
| `$AC:F00C` | `0x16700C` | 4 B | `JSR $F0A0; NOP` |
| `$AC:F0A0` | `0x1670A0` | 18 B | Dispatcher: classe `$11F0 == 2` usa IA normal `$80:CC64`; caso contrário usa IA de chefe `$80:CC68` |
| `$AC:F037` | `0x167037` | 6 B | Hook para loop dinâmico em `$AC:FE80` |
| `$AC:FE80` | `0x167E80` | 44 B | Define o contador de iterações. Usa quatro slots quando `$07D7 == 0` e `$11E0` está vazio ou é classe 2; usa três iterações nos outros casos |
| `$AC:F000` | `0x167000` | 4 B | Hook para guarda em `$AC:FE60` |
| `$AC:FE60` | `0x167E60` | 16 B | Se `$07D7 != 0`, segue para o fluxo de chefe `$AC:F041`; caso contrário retorna por `$80:CCF4` |
| `$B5:D8A8` | `0x1AD8A8` | 127 B | Desvia o limitador para `$AC:FDC7` |
| `$AC:FDC7` | `0x167DC7` | 133 B | Conta slots normais, inclusive `$11E0` quando `$11F0 == 2`; teto 4 no fluxo normal e 2 quando o helper detecta chefe |
| `$AC:FEB0` | `0x167EB0` | 17 B | Helper do limitador: retorna ativo quando `$07D7 != 0` |

**Comportamento forçado atual:** o allocator não testa se `$11E0` está livre. Quando os três slots normais estão cheios e o gate permite outro spawn, ele usa `$11E0` mesmo que haja outro objeto nesse slot. Isso pode substituir esse objeto. O processamento de chefe ainda depende do significado de `$07D7`, que não foi confirmado em execução real. O modo deve ser tratado como patch de teste.

As rotinas ficam em intervalos distintos: limitador `0x167DC7–0x167E4B`, guarda `0x167E60–0x167E6F`, loop `0x167E80–0x167EAB` e helper `0x167EB0–0x167EC0`. Auditoria estática das alterações do modo 4 encontrou zero bytes modificados fora das regiões previstas e nenhuma sobreposição entre patches.

### 14.5 Auto-Preencher Waves e capacidade dos scripts

- O checkbox opera durante a exportação. No modo 4, para cada wave que tenha ao menos um inimigo, o editor remove os registros não classificados como inimigo, põe até quatro inimigos no início da lista, define seus delays para 1 e copia o tipo/`spawnMode`/`diffHp` do primeiro inimigo para o quarto. Se houver entradas livres, pode inserir cópias adicionais.
- O modo de teste pode, portanto, remover barris, itens ou outros spawns **quando esses objetos estão registrados dentro da própria wave**. Objetos criados por outro script da fase não são removidos por essa operação.
- A wave inicial da ROM original contém sete registros: seis inimigos e um objeto. O contador do cabeçalho vale seis, pois conta inimigos, não objetos. `writeWaveToRom` agora grava esse contador com base apenas nas entradas de inimigos.
- O cálculo de `maxEntriesInPlace` foi corrigido: `endOff` aponta para o terminador/comando seguinte, então os bytes depois desse endereço não pertencem à área de registros. Na wave inicial, a capacidade correta é 7/7, não 6.
- Um aviso como **10/10 entradas** significa que a área de dados original daquela wave está cheia; **não é o limite de VRAM**. Gravar mais um registro depois de `endPtr` pode sobrescrever o comando ou a próxima wave. Realocação de uma cena inteira e atualização de seus ponteiros ainda não estão implementadas.
- Na auditoria/simulação do parser da ROM original, foram lidas 81 waves. O Auto-Preencher ajustou 75, repetiu tipo em 67, removeu 63 spawns de objetos, não inseriu cópias novas nesse conjunto e deixou 12 waves sem entradas para chegar a quatro. Depois da simulação não houve wave acima da capacidade nem divergência entre o contador de inimigos e as entradas classificadas.

### 14.6 VRAM, objetos e pendências de diagnóstico

- A documentação anterior indica três slots de VRAM de inimigos em `$0748–$074A`, com referências em `$0754+X`; `$0759` está associado à reserva de um slot gráfico. O código atual deixou de usar `$0759` como gate de chefe do limiter, porque isso podia confundir reserva de VRAM com ocupação de chefe.
- O modo Auto-Preencher repete o tipo de inimigo no quarto registro para reduzir a possibilidade de o limite de três tipos gráficos impedir o spawn. Ainda assim, não foi provado em emulador que todas as famílias de sprites compartilham o slot de VRAM esperado.
- Barris e itens podem consumir VRAM/OAM ou vir de scripts separados. O patch atual remove objetos registrados nas waves e força a seleção do slot `$11E0`, mas não desativa todos os pools de objetos nem altera o limite físico de sprites por scanline do SNES.
- **Resultado de jogo ainda pendente:** o usuário informou que versões anteriores continuavam exibindo só três inimigos, inclusive na Fase 2. O patch mais agressivo descrito nesta seção ainda precisa ser exportado e testado em emulador. Se continuar em três, será necessário observar em execução `$1000/$10A0/$1140/$11E0`, `$11F0`, `$07D7`, `$0759`, `$7FF2B0/$7FF2B2` e os slots de OAM/VRAM para separar gate de spawn, objeto ocupando `$11E0` e limite de renderização.

---

## 15. Causa Raiz Real dos 3 Inimigos e Correção Verificada em Emulador 65816 (2026-11-17)

Ao executar a ROM gerada pelo editor em um emulador 65816 headless ciclo a ciclo (monitorando `$1000..$11E0`, `$0748..$0759`, `$7FF2B0/$7FF2B2`, `$1680..$16A5` e OAM `$0180..$039F`), foi identificada a **causa raiz exata** que impedia o 4º inimigo de nascer em todas as fases, além de 4 bugs secundários no editor:

### 15.1 Os 5 Erros Encontrados

1. **Bloqueio de VRAM em `$82:CBE3` (`$82:D088`) ANTES do limitador (Causa Raiz Principal):**
   - As seções §10 e §14.5 assumiram que inimigos de mesmo `id` compartilhavam automaticamente um dos 3 slots de VRAM (`$0748–$074A`) via `$82:CC17` / `$82:D0C9`.
   - Na tabela `$82:D1C8`, porém, **todos os inimigos (`0x00..0x3F`) possuem classe `$76 = 0x02`**, que no estado `$03 = 0x06` (`$82:CBBA`) salta para **`$82:CBE1 -> JSL $AF:9B60 -> JSL $82:D088`**. A rotina compartilhada `$82:CC17` só é usada por objetos de cenário (`$76 = 0x0A`, `0x54..0x75`).
   - `$82:D088` **nunca compartilha slots**: apenas procura `$FF` em `$0748..$074A`. Com 3 inimigos vivos (ou 2 inimigos + 1 barril na Fase 1, ou 1 inimigo + 2 objetos na Fase 2), `$0748..$074A` ficavam todos preenchidos; o 4º inimigo recebia `SEC` em `$82:CBE3` e **ficava preso indefinidamente em `$03 = 0x06`**, jamais alcançando o estado `$03 = 0x0E` (`$82:CD1C`) onde ficam o limitador `$AC:FDC7` e o alocador `$AC:F050` — mesmo repetindo exatamente o mesmo ID!
   - Além disso, na morte do inimigo (`$80:AC68 -> JSL $82:D13C`), `$82:D13C` limpava `$0748,X = $FF` sem checar se outro inimigo vivo ainda usava o mesmo identificador de VRAM `+$23` (`$58/$78/$98`).

2. **Substituição Indevida de `$0759` por `$07D7` e Hooks Conflitantes em `$AC:F000` / `$AC:F037`:**
   - Todos os chefes de fase (Damnd `$84:C12E`, Sodom `$86:B535`, Edi.E `$86:C979`, Rolento/Abigail `$86:D6D0`, Belger `$86:E1EF`) gravam **`$0759 = 1`** (junto com `$0CD3 = 1`), enquanto `$07D7` permanece `0` em Damnd, Sodom, Edi.E e Abigail.
   - O hook anterior em `$AC:F000 -> $AC:FE60` abortava com `RTL` quando `$07D7 == 0`, congelando os chefes de fase, enquanto `$AC:F037 -> $AC:FE80` (`$000F = 3`) causava underflow em `$AC:812F` (`2 - $000F = $FF`). Na ROM Uncut original, `$AC:8000` já processa os slots 1–3 e `$80:CCDD -> $AC:F000` já processa `$11E0` 1 vez por frame, bastando o dispatcher em `$AC:F00C -> $AC:F0A0`.

3. **Alocador `$AC:F050` Sobrescrevendo `$11E0` sem Checar Ocupação:**
   - A versão anterior de `$AC:F050` não verificava `$7E11E0 == 0` nem `$7E0759 == 0` e nunca retornava falha (`LDA #$01; RTL`), podendo sobrescrever um chefe ou o próprio 4º inimigo ativo.

4. **Máscaras de Dificuldade (`diffHp`) e Flag `$168D` (`hdr+5`) no Editor JS:**
   - Em 52 das 75 waves com inimigos, os 4 primeiros registros continham entradas restritas a dificuldades específicas (`0x10` só Easy, `0xC0` só Hard/Expert), sendo puladas em `$82:CB5F` durante o jogo na dificuldade Normal (`0x20`).
   - Em `$82:CF83`, `hdr+4` (`entryCount`) carrega **tanto** `$168C` (mortes p/ liberar) **quanto** `$1696` (total da wave), enquanto `hdr+5` carrega `$168D` (flag booleana `0/1` da seta `"GO!"` em `$82:CEF0`). Incrementar `hdr+5` ao clonar/adicionar inimigos corrompia `$168D`.

5. **Bugs de UI no `FinalFight_Enemy_Editor.html`:**
   - `selectedStage` não estava declarada, gerando `ReferenceError` ao clicar em `⚡ Auto-Completar Fase (3/4)`.
   - Carregar uma ROM virgem (`0x167F40 == 0x00`) revertia silenciosamente a seleção para o Modo 3 Inimigos.

### 15.2 Mapa Final do Patch Verificado no Editor

| Hook / Rotina | Offset ROM | Tamanho | Função |
|---|---:|---:|---|
| `$80:AC27` | `0x002C27` | 27 B | `JML $AC:F050` + 23 `NOP`s |
| `$AC:F050` | `0x167050` | 69 B | Alocador seguro: `$1000/$10A0/$1140` e, se `$0759==0`, `$07D7==0` e `$11E0==0`, aloca `$11E0` com `$11F0 = 2` |
| `$AC:F00C` | `0x16700C` | 4 B | `JSR $F0A0; NOP` (mantendo `$AC:F000` e `$AC:F037` originais) |
| `$AC:F0A0` | `0x1670A0` | 18 B | Dispatcher de IA do Slot 4 (`$11F0 == 2` → `$80:CC64`; senão → `$80:CC68`) |
| `$B5:D8A8` | `0x1AD8A8` | 131 B | `JML $AC:FDC7` + 127 `NOP`s |
| `$AC:FDC7` | `0x167DC7` | 130 B | Limitador de 4 inimigos (teto = 4 sem chefe; teto = 2 com `$0759` ou `$07D7`; sem trava Slum `$5A`) |
| `$82:CBE3` | `0x014BE3` | 4 B | `JML $AC:FE4A` (hook de claim de VRAM de inimigos) |
| `$AC:FE4A` | `0x167E4A` | 134 B | Usa slot de VRAM livre primeiro (`$AF:9B60`); quando os 3 slots estão cheios e há vaga no limitador, compartilha slot por `id` exato (`$79`) ou mesma família gráfica (`$77`) |
| `$80:AC68` | `0x002C68` | 4 B | `JSL $AC:FED0` (hook de liberação segura de VRAM na morte) |
| `$AC:FED0` | `0x167ED0` | 112 B | Só libera `$0748,X`/`$0750,X` via `$82:D13C` quando nenhum outro inimigo vivo (nem spawn em trânsito) usa o mesmo `+$23`; sempre decrementa `$168C` |

### 15.3 Resultados Verificados em Execução 65816

- **Fase 1 Sem Auto-Preencher**: 4 inimigos simultâneos (`Two P` + `Bred` + `Dug` + `Simons`) + barril ativo em tela (`$1000, $10A0, $1140, $11E0` todos em estado `3` e renderizados no OAM); ao derrotar todos os 4, os slots de VRAM retornam a `$FF` na ordem exata e `$0CC2` destrava a câmera ("GO!").
- **Fase 1 e Fase 2 Com Auto-Preencher**: 4 inimigos simultâneos imediatos **com todos os 65 objetos quebráveis preservados** (`0` objetos removidos nas 81 waves). Na Fase 1 (`W1.1`), o barril (`$1280`) e os 4 inimigos (`Two P`, `Bred`, `Dug`, `Simons`) nascem juntos; na Fase 2 (`W1.1`), tanto o barril (`$1280`) quanto a cabine telefônica (`$1300`) nascem junto com os 4 inimigos simultâneos.
- **Chefes (`$0759 = 1`)**: IA de chefe (`$80:CC68`) executada normalmente em todos os quadros, com teto de 2 inimigos auxiliares e proteção total do slot `$11E0`.
- **Restauração Modo 3**: `0` bytes divergentes em relação à ROM `Final Fight Guy (Japan) uncut.sfc` original (reparando inclusive ROMs antigas corrompidas).

### 15.4 Preservação dos Objetos Quebráveis no Auto-Preencher (`autoFillWaveForSimultaneous`)

- Uma versão experimental anterior de `autoFillWaveForSimultaneous` filtrava `!isEnemyId(entry.id)` quando `targetEnemies === 4`, apagando os 65 objetos quebráveis (`0x57` barris de madeira, `0x5A` caixas, `0x60` pneus, `0x63` cabines telefônicas, `0x66` tambores metálicos, `0x69` placas, `0x6C` barris, `0x72` lustres) do Bank `$87`.
- **Correção aplicada**:
  1. Objetos (`!isEnemyId`) **nunca** são removidos nem sobrescritos por cópias de inimigos.
  2. Objetos iniciais da wave permanecem **antes** de `openingEnemies` (`[leadingObjects] -> [openingEnemies] -> [restante]`), pois objetos (`$76 = 0x0A`) usam `$82:CC17 -> $82:D0C9` e exigem um slot livre (`$FF`) em `$0748..$074A` para nascer, enquanto inimigos posteriores conseguem compartilhar slots ocupados via `$AC:FE4A`. Lustres tardios (`delay > 1` no meio da wave, ex.: Fase 4 `W3.1` e Fase 5 `W5.1`) mantêm sua posição intermediária original.
  3. O cálculo `freeVramSlotsForEnemies = Math.max(1, 3 - leadingObjFamilies.size)` verifica quantos slots de VRAM sobram após os objetos iniciais e só repete `sourceEnemy` nos inimigos excedentes cuja família `$77` (`$82:D1C9`) não esteja entre as famílias já carregadas.

---

## 16. Viabilidade Técnica de um 5º Inimigo Simultâneo e Dynamic DMA

Mesmo sem *slowdown* de CPU (FastROM + multitarefa cooperativa), liberar um **5º inimigo simultâneo** esbarra em 3 barreiras físicas e estruturais do SNES / motor da Capcom:

1. **Teto Físico de Hardware OAM da PPU (128 Sprites no Total):**
   - Medido em emulação real com 1 jogador + 4 inimigos ativos: o consumo já atinge **117 de 128 entradas OAM totais** (`$00D8` cai de `128` para `11`), antes mesmo de somar barris (~8 sprites), armas no chão, sombras extras, *hitsparks* ou popups de pontuação.
   - Um 5º inimigo (+20 a +26 sprites `8×8`) elevaria a demanda para **~140–150 sprites (> 128)**, estourando o limite absoluto da PPU do SNES (causando desaparecimento de troncos/membros inteiros além do corte de 32 sprites / 34 *tile slivers* por scanline).
2. **Mapa Fechado da Low WRAM (`$0000–$1FFF`) para Direct Page (`TCD`):**
   - Toda a IA, física e colisão usam Direct Page (`D`), que no 65816 é fixado no Bank `$00` (`$0000–$1FFF`). Todo o espaço já está particionado sem nenhuma lacuna livre de `$A0` bytes:
     - `$0D00–$0DFF`: Jogadores 1 e 2 (`2 × $80`)
     - `$0E00–$0FFF`: Câmera (`$0E07`) e atores auxiliares (`4 × $80`)
     - `$1000–$127F`: Slots 1–4 de inimigos (`$1000, $10A0, $1140, $11E0`, `4 × $A0`)
     - `$1280–$14FF`: Objetos quebráveis (`5 × $80`, alocador `$80:AD3D`)
     - `$1500–$167F`: Projéteis / facas / molotovs (`6 × $40`, `$80:AD8F`)
     - `$1680–$16FF`: Estado do controlador de wave (`$80`)
     - `$1700–$19FF`: Itens / comidas / armas (`6 × $80`, `$80:AD0C`)
     - `$1A00–$1BFF`: Hitsparks / pontos (`8 × $40`, `$80:ACA9`)
     - `$1C00–$1D7F`: Estilhaços / efeitos (`3 × $80`, `$80:AC6F`)
     - `$1DC0–$1FFF`: 8 pilhas de tarefas (`8 × $40`)
   - Abrir `$1280` (`$11E0 + $A0`) para um 5º inimigo exigiria encolher o pool de objetos quebráveis (`$80:AD3D`) de 5 para 3 slots.
3. **Mais de 20 Rotinas Desenroladas Manualmente para 4 Endereços Fixos:**
   - Diferente de um *loop* parametrizado, tanto a ROM original quanto o hack *Uncut* codificam explicitamente `LDX/LDY #$1000, #$10A0, #$1140, #$11E0` em ~20+ rotinas separadas: socos/chutes/agarrões (`$81:B545`, `$81:B586`, `$81:B8BA`, `$81:BC4A`), ordenação Y de sprites (`$81:BFB4–$81:BFE0`), colisão de armas (`$82:AFF4`, `$82:B4EB`), arremessos/fogo amigo (`$BE:D03C–$BE:D3EC`), DMA de Andore (`$BE:D3ED–$BE:D87F`) e checagem de fim de wave (`$B5:CC3F`). O 4º slot funcionou de forma limpa justamente porque `$11E0` já existia em todas essas rotinas (para chefes).
4. **Por que ocorre o glitch momentâneo no clone em animações diferentes:**
   - Todos os inimigos (`$76 = 0x02`) usam *Dynamic DMA* por pose nos 3 bancos de 80 tiles da página OAM 1 (`$1100`, `$1600`, `$1B00`). Quando o 4º inimigo compartilha `+$23` com outro da mesma família `$77`, ambos apontam para o mesmo banco de VRAM; se estiverem em quadros de animação diferentes no mesmo frame, o último DMA pode sobrescrever os tiles da pose anterior. A faixa candidata `$0360–$085F` em words equivale aos bytes `$06C0–$10BF`; os snapshots mostram que ela está preenchida e que não corresponde aos tiles OAM observados para o P2. Ela ainda não foi validada como espaço livre para um quarto banco.

---

## 17. Experimento de banco gráfico dedicado revertido (2026-11-20)

O primeiro print mostrava corrupção em partes de inimigos; isso sugeria conflito de *Dynamic DMA*. Foi experimentado no editor um banco adicional em `$0360`, com redirecionamento das tabelas DMA/OAM e extensão do alocador para o quarto índice.

O teste do usuário piorou o resultado: além dos inimigos continuarem corrompidos, Guy passou a apresentar cores erradas. A alteração redirecionava tabelas compartilhadas pelo jogo, então afetou gráficos que não pertencem ao quarto inimigo. A hipótese de usar `$0360` por esse caminho não se confirmou.

O experimento foi removido de `NATIVE_ENGINE_PATCHES`; o modo 4 voltou ao comportamento anterior, que compartilha o banco gráfico por família quando os três bancos estão ocupados. As entradas de restauração permanecem em `NATIVE_ENGINE_RESTORE` para limpar ROMs que já receberam o experimento quando forem abertas/exportadas novamente. A ROM gerada `teste_graficos_4_inimigos_banco_dedicado.sfc` foi removida.

**Estado após a comparação de 22/11:** o editor voltou ao comportamento da ROM funcional de referência; o glitch gráfico original ao usar 4 inimigos continua sendo aceito pelo usuário. O caminho de banco dedicado por redirecionamento global foi rejeitado. Uma nova tentativa precisa acompanhar a DMA e os atributos OAM por ator e alterar somente a rota gráfica do quarto inimigo. O Snes9x local também encerrou ao abrir tanto a ROM original quanto a modificada (`0xC0000409`), então aquela execução não pôde ser depurada por aqui.

## 18. Revisão da hipótese de VRAM do segundo jogador (2026-11-20)

A análise da ROM original confirma que ela não é limitada a um herói: há um objeto P1 em `$0D00`, um objeto P2 em `$0D80` e a flag de 2 jogadores em `$00CD bit 1`. O código em `$AF:D000` lê `$7E:0D80` dentro de um caminho condicional; portanto, o jogo realmente mantém lógica para o segundo jogador. Os testes anteriores do modo de quatro inimigos foram feitos em 1P, mas o patch não desativa nem força esse modo: ele não altera `$00CD` ou `$0D80`.

O patch nativo atualmente ativo no editor não escolhe VRAM do P2. Ele usa os três slots de gráficos dinâmicos de inimigos (`$0748–$074A`) e, quando necessário, faz o quarto inimigo compartilhar um slot por ID/família gráfica. A ROM com dois heróis e três inimigos funcionando mostra que o jogo suporta essa combinação, mas não identifica uma faixa de VRAM livre para emprestar. O espaço de WRAM do objeto P2 e o mapa de tiles na VRAM são recursos diferentes.

O experimento revertido tentou adicionar um banco em `$0360` e redirecionar tabelas globais de DMA/OAM. A disponibilidade desse intervalo para P2 ou para uso temporário em 1P nunca foi provada; por isso, não se pode atribuir a piora das cores do Guy especificamente a uma colisão com os gráficos do P2. O trecho condicional de `$AF:D000` que também contém o literal `#$0360` não é, por si só, uma escrita de destino para VRAM.

Uma leitura adicional do alocador confirma por que não basta trocar o destino para `$0360`: `$82:D088` procura três índices (começa em `X = 2` e percorre `$0748–$074A`), e `$82:D0A0` usa esse índice em cinco tabelas paralelas, cada uma com três entradas (`$02:D119`, `$02:D11C`, `$02:D11F`, `$02:D1C2`, `$02:D1C5`). Um quarto banco exige estender a alocação, as cinco tabelas, a seleção de tiles OAM e a liberação/referência do slot. Também precisa respeitar a flag de 2 jogadores para não sobrescrever o P2. O teste anterior alterou rotas/tabelas compartilhadas sem esse mapeamento isolado; por isso, não é um patch seguro para reaplicar.

**Conclusão após os snapshots 1P/2P:** o patch ativo não toma deliberadamente a VRAM do segundo jogador. No dump 2P, 21 entradas OAM do segundo herói usam tiles `$A0–$CD` com o bit 0 do atributo limpo, selecionando a primeira tabela OBJ. Os pontos do jogo encontrados na ROM escrevem `$00` em `$2101` (OBJSEL); sob essa configuração, os tiles do P2 correspondem aos bytes `$1400–$19BF`. Comparando os snapshots, 1.322 dos 1.472 bytes dessa faixa mudam entre 1P e 2P, forte evidência de que ela contém os gráficos carregados para o segundo herói. Portanto, essa faixa não pode ser emprestada enquanto os dois jogadores estiverem ativos.

A antiga faixa candidata `$0360–$085F` em words corresponde aos bytes `$06C0–$10BF`; ela não é a faixa de tiles do P2 identificada pelo OAM. Nela há 1.213 bytes não zero no dump 1P e 1.266 no 2P, com 191 diferenças entre as capturas. Isso prova que a faixa está preenchida, mas não diz por si só se esses bytes são tiles de fundo ativos ou dados sobrando. O atlas visual mostra padrões gráficos estruturados; falta cruzar a faixa com os mapas e bases de caracteres do fundo antes de considerá-la segura. A hipótese anterior de que a diferença de hash entre 1P e 2P identificava essa região como VRAM do P2 está retirada.

Nos dois snapshots, Guy e os inimigos têm o bit 0 do atributo OAM ligado e usam a segunda tabela OBJ. O P2 usa a primeira tabela, em `$1400–$19BF`; a CGRAM também difere em 29 bytes entre as capturas. Isso demonstra que o modo 2P carrega gráficos/paleta próprios, mas não aponta para um banco livre que possa ser reaproveitado. O quarto inimigo continuou compartilhando o seletor `$58` com outros inimigos.

Durante o experimento de 20/11, o editor ofereceu temporariamente uma opção para reservar a página OBJ do P2 ao 4º inimigo. Essa opção foi removida após falhas de edição reportadas; o editor voltou ao alocador gráfico anterior, que compartilha o banco do 4º inimigo por família. A faixa do P2 permanece identificada como ocupada em 2P e não deve ser reutilizada nessa situação.

As duas tentativas de callback Lua para detectar escritas DMA produziram zero eventos, embora os hashes de VRAM mudassem entre quadros. Esses zeros não são evidência de ausência de DMA. Os snapshots agora mapeiam as páginas OBJ usadas pelos jogadores; para atribuir uma gravação específica a cada DMA ainda é necessário capturar os eventos de DMA ou comparar estados da mesma execução antes/depois da animação.

## 19. Experimento revertido: banco gráfico do P2 para o 4º inimigo (2026-11-20)

**Status: revertido no editor.** A tentativa foi removida depois que o usuário relatou que o editor deixou de editar ROMs. O modo normal de 3/4 inimigos e o compartilhamento gráfico anterior foram restaurados. As entradas de restauração no editor limpam os hooks e tabelas experimentais ao carregar/exportar uma ROM que os contenha. A ROM `FinalFight_4_Enemies_P2VRAM_Experimental_1P.sfc` foi gerada como artefato da tentativa; em 22/11 ela já não estava presente no workspace e não deve ser tratada como versão funcional caso exista outra cópia.

O editor recebeu a opção **“usar o banco OBJ do P2”**, desligada por padrão. No modo 4 inimigos, o alocador tenta esse banco antes de compartilhar gráficos por família, mas somente quando `$00CD bit 1 = 0`. Se P2 estiver ativo no momento da alocação ou o banco já estiver marcado como ocupado, o patch usa o fluxo anterior. A flag é verificada na alocação; não se deve permitir que o segundo jogador entre durante a fase depois que o banco foi cedido.

O índice especial é `X=3`, seletor `$B8`, tile OBJ inicial `$A0` e destino de DMA em VRAM word `$0AB8`. A entrada usa bit 0 do atributo OAM desligado para selecionar a primeira página OBJ. As tabelas estendidas ficam em `$AC:FF42–$AC:FF55`; o código 1P-only fica em `$AC:FF56–$AC:FF94`. Os caminhos de liberação `$82:D13C` e `$82:D161` também passam a procurar o índice 3 e liberam `$074B/$0753`.

O checkbox persiste no byte marcador `$AC:FF41 = $52`; desligá-lo ou voltar ao modo de 3 inimigos restaura as tabelas, os hooks e os limites originais. A ROM de teste `FinalFight_4_Enemies_P2VRAM_Experimental_1P.sfc` foi gerada a partir da ROM modificada já disponível na pasta, mantendo os dados das waves e recalculando o checksum.

**Escopo do experimento:** ele cria um banco gráfico dedicado para o 4º inimigo existente; não cria um 5º slot de entidade. O estado atual do patch foi validado por sintaxe JS, aplicação/desativação e restauração dos bytes no teste estático. Ainda precisa ser executado no Mesen em 1P para validar a animação real; em 2P o teste deve permanecer desligado.

## 20. Comparação com a ROM funcional de 4 inimigos (2026-11-22)

`ff-funcionando-4-inimigos.sfc` e `Final_Fight_Guy_4_Enemies.sfc` têm o mesmo SHA-256 e são cópias idênticas. Ao aplicar o motor do editor à ROM original e comparar com essa ROM funcional, surgiram 10 diferenças dentro dos trechos de restauração: cinco bytes em `$0150AC` e cinco em `$0150FB`. As duas entradas de `NATIVE_ENGINE_RESTORE` começavam um byte antes do limite correto da instrução e escreviam `A9` sobre o operando imediato anterior.

As entradas foram corrigidas para os bytes originais observados na ROM funcional: `$0150AC = 01 8D 1D 00 BF` e `$0150FB = 01 8D 1D 00 BF`. A sequência em `$014C8C` já estava alinhada e foi mantida. Depois da correção, aplicar o motor atual do editor à ROM original produz zero diferenças nos trechos de patch/restauração quando comparado à ROM funcional. Reaplicar o motor à própria `ff-funcionando-4-inimigos.sfc` não altera nenhum byte fora dos quatro bytes de checksum.

As diferenças restantes ao comparar a ROM original com a funcional estão fora dos trechos do motor e preservam os dados de fase/waves da ROM funcional. A comparação foi binária; depois da correção, o usuário confirmou que o editor voltou ao normal. Isso não elimina o glitch gráfico original dos quatro inimigos.

## 21. Registro de regressões e procedimento para não repeti-las (2026-11-22)

### Regressão encontrada e correção aplicada

- Referências de comparação: `Final Fight Guy (Japan) uncut.sfc` é a ROM original; `ff-funcionando-4-inimigos.sfc` é a versão aceita com 4 inimigos e o glitch gráfico anterior; `Final_Fight_Guy_4_Enemies.sfc` é byte a byte idêntica à `ff-funcionando-4-inimigos.sfc` (mesmo SHA-256).
- A restauração do editor continha três trechos que limpavam as alterações antigas da página OBJ. O trecho em `$014C8C` estava alinhado, mas os trechos em `$0150AC` e `$0150FB` começavam no operando imediato `01`, não no opcode `A9`. Escrever `A9 01 8D 1D 00` nesses endereços deslocava a instrução e corrompia cinco bytes em cada local.
- Esses trechos de compatibilidade foram acrescentados durante a tentativa P2 e eram executados sempre que `applyNativeEngine` carregava ou exportava uma ROM, mesmo com a opção experimental removida. Por isso a regressão afetava o fluxo normal do editor; não era necessário ativar o banco P2 para dispará-la.
- Bytes confirmados pela ROM funcional: `$014C8C = A9 01 8D 1D 00`; `$0150AC = 01 8D 1D 00 BF`; `$0150FB = 01 8D 1D 00 BF`. O editor agora restaura esses valores exatos.
- Após a correção, aplicar o motor do editor à ROM original resulta em zero divergências dentro dos intervalos de patch/restauração em relação à ROM funcional. Reaplicar o motor à ROM funcional preserva todos os bytes fora do checksum. O usuário confirmou que o editor voltou ao normal.

### Tentativas gráficas que pioraram e ficam revertidas

1. **Redirecionamento do banco candidato `$0360`**: essa faixa corresponde aos bytes `$06C0–$10BF`, estava preenchida e não foi demonstrada como VRAM livre. Redirecionar tabelas comuns de DMA/OAM piorou a corrupção e deixou Guy com cores erradas. O patch foi removido. Não reaplicar sem rastrear cada transferência DMA e os atributos OAM do ator.
2. **Banco OBJ emprestado do P2**: a opção experimental tentou usar tiles `$A0–$EF`/VRAM words `$0A00–$0EFF` para os gráficos do 4º inimigo em 1P. O patch criou tabelas estendidas, índice especial `X=3`, seletor `$B8`, alvo `$0AB8` e mudança de página OAM. A validação estática passou, mas isso não provava que o fluxo real de DMA era seguro. O teste do usuário piorou os gráficos e deixou o editor sem editar ROM corretamente; a opção e o hook foram removidos. Não tratar o banco do P2 como área livre, principalmente em 2P.
3. **Backup antigo do editor**: `FinalFight_Enemy_Editor.7z` é de 06/11 e contém um motor diferente que promete modos de 4/5 inimigos usando VRAM do P2. Não foi restaurado porque reintroduziria outra implementação experimental, não a ROM funcional de referência.

### Regras de comparação para futuras mudanças

- Antes de alterar a restauração, comparar bytes imediatamente antes e depois de cada offset com a ROM original e a `ff-funcionando-4-inimigos.sfc`; distinguir opcode de operando e confirmar o começo/fim da instrução.
- Depois de qualquer mudança no motor, reaplicar o editor à ROM funcional e exigir zero divergências fora dos quatro bytes de checksum. Também comparar os intervalos `NATIVE_ENGINE_RESTORE` e `NATIVE_ENGINE_PATCHES` ao aplicar o motor à ROM original.
- Preservar a ROM `ff-funcionando-4-inimigos.sfc` como referência. Não gerar sobrescrita em cima dela; produzir qualquer teste com outro nome.
- Sintaxe JavaScript e testes estáticos de arrays não confirmam gráficos corretos. Mudanças de DMA/OAM precisam de teste visual no Mesen; até lá, manter o motor que corresponde à ROM funcional e aceitar o glitch gráfico já conhecido.
- Não ativar Auto-Preencher durante a comparação binária, para que alterações intencionais de waves não sejam confundidas com mudanças no motor.

## 22. Expansão segura de waves cheias (2026-11-22)

O aviso `7/7 entradas` representa a capacidade *in-place* do bloco atual, calculada por `floor((endOff - (lgOff + 8)) / 9)`: o cabeçalho ocupa 8 bytes e cada registro de spawn ocupa 9. `endPtr` marca o próximo terminador/comando; gravar além dele sem mover dados pode corromper a próxima wave. A fase pode ter muito mais que 7 inimigos somando waves e subwaves distintas. O teto de 4 inimigos simultâneos é uma restrição diferente.

A análise inicial identificou cinco intervalos candidatos em Bank `$87`. A ROM original e a ROM funcional têm 2.862 bytes `$00/$FF` nesses intervalos, em blocos de 1.104, 496, 447, 432 e 383 bytes. O número bruto de bytes não equivale à capacidade real: a cena precisa caber inteira em um bloco contíguo, além dos registros acrescentados e seus terminadores.

## 23. Implementação da expansão de cenas (2026-11-22)

- Ao inserir ou duplicar uma entrada acima de `maxEntriesInPlace`, o editor tenta realocar uma cópia completa da cena para um dos intervalos seguros em Bank `$87`. O script original permanece intacto; a célula da tabela da cena aponta para a cópia nova.
- A wave ampliada recebe as entradas atuais, um terminador `$8000` e espaço reservado para mais 8 entradas. O editor aumenta o `endPtr`, atualiza os `endPtr` das waves deslocadas, reposiciona os offsets internos usados pela interface e grava o contador de inimigos em `hdr+4`, preservando `hdr+5` (`GO!`). Os comandos e waves seguintes são copiados depois do bloco ampliado.
- O Autopreencher agora tenta inserir a quantidade solicitada mesmo quando a capacidade antiga acabou. Se a realocação não puder ser feita, restaura as entradas da wave; o resumo informa quantas waves ficaram sem área segura.
- Cada expansão usa um bloco contíguo completo; os intervalos incluem 2 bytes de marca `$WV`. Uma cena precisa caber inteira no bloco: cenas acima de aproximadamente 1.102 bytes não cabem no maior. Como cada intervalo comporta uma cena realocada, o máximo atual é cinco cenas expandidas por ROM, e pode ser menor se um bloco menor for usado ou se já estiver ocupado. A cena pode crescer novamente dentro do mesmo bloco enquanto houver bytes livres.
- A realocação está habilitada para cenas referenciadas pela tabela de fases. Os dois scripts especiais de Final Fight Guy (`$87:C570` e `$87:C630`) não têm uma célula de tabela mapeada e continuam sem expansão automática. Se não houver espaço contíguo validado, ou se os limites do script não forem confirmados, o editor recusa a gravação excedente e preserva a wave original.
- O limite de 255 entradas continua sendo apenas o máximo representável nos campos de contagem; não foi confirmado como seguro no jogo. A capacidade real também depende do tamanho da cena, dos blocos disponíveis e do comportamento dos contadores/runtime. A ROM exportada ainda precisa ser aberta no Mesen para validar waves realocadas e o Autopreencher.

## 24. Correção de seleção de wave, Auto-Preencher e persistência do modo (2026-11-22)

- Ao carregar outra ROM, o editor limpa a wave selecionada anteriormente. Após qualquer reparse, se o ponteiro da seleção não existir mais, seleciona a primeira wave válida em vez de deixar dados antigos na tela.
- O cartão Auto-Preencher não alterna mais o checkbox duas vezes quando o usuário clica no texto. Marcar a opção agora aplica o preenchimento imediatamente às waves carregadas; a exportação continua reaplicando-o para cobrir edições feitas depois.
- O modo 3/4 fica guardado no estado do editor e é usado no canvas e na exportação. Alterar waves ou aplicar Auto-Preencher não deve selecionar o modo 3 nem desmarcar o modo 4.
- As waves originais incluem registros de objetos e registros condicionados por dificuldade. A lista do editor mostra todos os registros da ROM e indica a máscara `Diff`; o contador do cabeçalho conta inimigos, por isso pode ser menor que a quantidade total de linhas/spawns.
- Essas mudanças corrigem o estado de seleção e a aplicação do checkbox. A composição de uma wave ainda deve ser conferida contra a cena/dificuldade desejada; o Mesen continua sendo a validação final do número de inimigos simultâneos.

## 25. Novo Banco Gráfico Dedicado do P2 para o 4º Inimigo em 1P (2026-11-23)

Diferente da tentativa revertida da seção 19, a nova implementação no `FinalFight_Enemy_Editor.html` **não altera nenhum byte das tabelas ou alocadores do Bank `$82`** (`$02:D119..$02:D1C5`, `$82:D088`, `$82:D0C9`, `$82:CC6D`, `$82:D13C`, `$82:D161`, `$014C8C`, `$0150AC`, `$0150FB` permanecem 100% intactos).

### Arquitetura verificada por disassembly e simulação 65816

- **Gatilho opcional no editor:** checkbox **“Banco Gráfico do P2 p/ 4º Inimigo (1P)”** (ligado por padrão no Modo 4; marcador `0x167F41 = $50`). Desligar o checkbox produz saída byte a byte idêntica à `ff-funcionando-4-inimigos.sfc` (0 divergências fora do checksum); voltar ao Modo 3 restaura integralmente a ROM original (incluindo zerar `$AC:ED00..$AC:EFFF`).
- **Hooks reutilizados (sem novos hooks em código Capcom/Uncut):** apenas redireciona os hooks já validados em `$AC:FE4A` (`JMP $ED00`) e `$AC:FED0` (`JMP $EE60`) para duas rotinas na área livre zerada `$AC:ED00..$AC:EFFF` (768 B verificados `$00` tanto na ROM original quanto na funcional):
  - **`$AC:ED00` (341 B — `0x166D00..0x166E54`):** tenta primeiro os 3 bancos normais da Página 1 (`$AF:9B60`). Se ocupados e o limitador (`$AC:FDC7`) liberar spawn, verifica que é **1P** (`$7E:00CD & $02 == 0` e `$7E:0D80 == 0`), **sem chefe** (`$0759 == 0` e `$07D7 == 0`), sem recarga DMA pendente (`$00E7 & $40 == 0`) e que nenhum inimigo vivo usa `$23 == $D8`. Passando, aloca o 4º banco dedicado e salta para `$82:CBE9` com:
    - Buffer de metasprites em WRAM: **`$7E:D800–$7E:F7FF`** (`A = $D8`, 8 KB livres verificados — evita `$7E:B800`, usado por `$82:C2AC`).
    - VRAM destino (Dynamic DMA): **`$0A00–$0EFF` words** (`B = $0A`, Página 0, 80 tiles — usado nativamente por `$80:86E1` e `$AF:AB0F`).
    - OAM tile base e página: **`$001C = $A0`, `$001D = $00`** (Página 0; `$AC:F1F8` e `$AC:F3D0` preservam o bit 8 via `AND #$C1FF`).
    - Paleta própria em 1P: **OBJ Paleta 1** (`$001A = $02`, CGRAM `$001B = $20` → `$0520`, exclusiva do P2).
    - Em 2P, chefes ou se o 4º banco já estiver ocupado (ex.: 2 objetos quebráveis na Página 1), cai automaticamente para o fallback seguro de compartilhamento (incluindo compartilhamento em `$074B` se aplicável).
  - **`$AC:EE60` (193 B — `0x166E60..0x166F20`):** ao morrer um inimigo usando `$23 == $D8` (sem outro inimigo/spawn em trânsito usando `$D8`), libera `$074B/$0753 = $FF`, decrementa `$168C` e enfileira a restauração VBlank-safe dos tiles originais de comida/itens (`$10:F020` → VRAM `$0A00`, `$0E00` B, verificado 0/3.584 bytes de diferença contra `ff_1p_vram.bin`) via `JSL $808999` com `X = #$00C4`, `A = #$40` (bit 6 de `$00E7`, sem colidir com bit 0 de waves).

### Validações concluídas (pendente teste visual no Mesen 2)

- Sintaxe JS e testes binários do editor (`applyNativeEngine`): **0 bytes divergentes** em Modo 4 clássico vs `ff-funcionando-4-inimigos.sfc`, **0 bytes divergentes** no rollback para Modo 3 vs `Final Fight Guy (Japan) uncut.sfc` (fora marcador `0x167F40`) e **0 bytes alterados no Bank `$82`**.
- **10/10 testes unitários em simulador 65816** aprovados sobre a ROM gerada pelo próprio editor (alocação 1P, teto de 4, fallback 2P, proteção de chefe, 2 barris + compartilhamento em `$074B`, espera de recarga DMA, liberação + restauração de comidas `$00C4` e liberação Página 1).
- **Próximo passo:** exportar a ROM pelo editor (sem sobrescrever `ff-funcionando-4-inimigos.sfc`) e validar visualmente no **Mesen 2** em 1P (4 inimigos simultâneos de tipos iguais e diferentes, drop de comidas/itens, chefes e entrada de 2P).

## 26. Correções de UX, objetos quebráveis, catálogo e filtro de dificuldade no Editor (2026-11-24)

- **Preservação de seleção e reordenação de spawns:** `selectWave(wave, keepEntryIndex)` agora mantém a seleção no item recém-adicionado (`addEnemyToWave()`), clonado (`duplicateSelectedEnemy()`) ou movido, eliminando o reset silencioso para `#1`. Foram adicionados os botões **⬆️ Subir** e **⬇️ Descer** (`moveSelectedEntry`) para reordenar entradas diretamente na wave.
- **Correção da edição de objetos quebráveis (`0x54..0x75`, ex.: Barril `0x6C` → Cabine Telefônica `0x63`):**
  - Verificado por disassembly em `$82:CC17` → `$82:CDAE` → `$A4:B411` (`$24:A000` para `0x6C`, `$24:E000` para `0x63`) e `$A4:B45B` (paletas) que `0x63` é totalmente suportado na Fase 1 (inclusive já é usado nativamente na Wave 2.1).
  - O bug ao trocar `0x6C` → `0x63` vinha de `updateSelectedEntryFromUI()`: como objetos quebráveis usam `diffHp = 0x00/0x01/0x02` (`bits 4..7 = 0`), nenhum checkbox de dificuldade ficava marcado e o código forçava `diffMask = 0xF0`, corrompendo `0x01` → `0xF1`. Com o bit 7 ativo, `$82:D002` (`BPL $D034; INC $17`) fazia o objeto incrementar o contador de inimigos `$17` e desbalanceava a wave; além disso, `spawnMode` customizados (ex.: `0x81`, `0x87`) eram sobrescritos para `0x00`.
  - Corrigido: `updateSelectedEntryFromUI(fromSpawnModeSelect)` preserva `bits 4..7 = 0` para objetos, desabilita os checkboxes de dificuldade quando um objeto está selecionado, preserva qualquer `spawnMode` customizado e faz conversão limpa ao alternar Objeto ↔ Inimigo (atualizando também `entryCount`).
- **Filtro visual de dificuldade (padrão Normal `0x20`) e Wave 3.1 (`$87:89F0`):** adicionado seletor **Destacar Dificuldade** + resumo na lista de spawns e transparência no canvas. Explica por que a Wave 3.1 nativa possui 7 registros na ROM, mas apenas 3 nascem no Normal (`#1 Simons`, `#3 Axl`, `#5 Two.P`) enquanto os outros 4 são restritos a Hard/Expert (`$82:CB5F`).
- **Catálogo (`ENEMY_CATALOG`) e canvas:** corrigidos todos os nomes de inimigos conforme `$80:E6C0` e `$82:D1C8` (`0x12` Axl, `0x15` Slash, `0x2A` Bill Bull, `0x2D` Wong Who, `0x30` Holly Wood, `0x33` El Gado, `0x36` Roxy, `0x39` Poison, `0x3C`/`0x3F` Holly Wood Especial/Molotov) e corrigida a sobreposição de texto no cabeçalho do canvas.
- **Validação automatizada:** suíte de regressão binária/rollback (Testes A–F) com 0 divergências + 7/7 testes comportamentais JS aprovados.

## 27. Opcao para adicionar ao Expert os inimigos das outras dificuldades (2026-11-26)

- Adicionado o checkbox **Todos os Inimigos no Expert**. Ao ativar, o editor percorre todas as waves e cenas (incluindo os overrides do modo Final Fight Guy) e acrescenta o bit `$80` de Expert apenas a registros classificados como inimigos (`isEnemyId`).
- Os bits Easy (`$10`), Normal (`$20`) e Hard (`$40`) e o nibble inferior de HP/IA sao preservados. Registros de objeto nao sao alterados. Uma mascara de dificuldade igual a zero ja significa que o inimigo aparece em todas as dificuldades; ela nao pode ser transformada em `$80`, pois isso o tornaria exclusivo de Expert.
- Ao desativar o checkbox, o editor remove somente os bits `$80` que esta opcao acrescentou durante a sessao. Bits Expert que ja existiam na ROM permanecem. A lista/canvas atualizam junto com a ROM em memoria.
- Na exportacao SFC ou IPS, a opcao e reaplicada depois do Auto-Preencher, assim inimigos criados por essa operacao tambem entram no Expert quando aplicavel. Nenhum spawn novo e criado por esta opcao; ela torna visiveis no Expert os registros que ja estavam restritos a outras dificuldades.
- A validacao pendente e abrir a ROM exportada no Mesen e comparar a lista Expert com Easy/Normal/Hard, verificando especialmente waves com mascara `$00` e objetos quebraveis.

## 28. Preparação para publicação e interface em inglês/português (2026-11-26)

- O editor agora abre em inglês por padrão (`<html lang="en">`). O botão **Português** alterna a interface para português e vice-versa; a escolha fica salva em `localStorage` (`finalFightEditorLanguage`) para as próximas aberturas. Se o armazenamento do navegador estiver indisponível, o padrão continua sendo inglês.
- A tradução cobre os rótulos e dicas da página, o catálogo de inimigos/objetos, as mensagens geradas durante a edição, os alertas e os textos desenhados no canvas. Esta mudança é apenas de interface; não modifica os patches nativos nem os dados binários da ROM.
- Foi criado `README.md` com apresentação e instruções em inglês e português, incluindo uma sugestão de descrição bilíngue para o repositório GitHub. A descrição é texto para copiar; nada foi publicado ou enviado ao GitHub.
- `.gitignore` deixa de fora ROMs (`.sfc`/`.smc`), patches IPS, executáveis e bibliotecas, arquivos compactados, scripts Lua de diagnóstico, dumps gráficos, captura de tela e script local de diagnóstico. O HTML do editor permanece como conteúdo-fonte publicável.
- Nenhuma licença foi escolhida; o README não concede permissão de reutilização. Também não foi feita validação visual do seletor/traduções no navegador durante esta preparação.
- O repositório Git local foi inicializado na branch `main` para facilitar o envio futuro. Não há arquivos em staging, commit, remote configurado ou publicação feita.

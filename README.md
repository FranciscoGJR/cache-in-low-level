# Resumo da atuação

## **📋 PROJETO DE CACHE MEMORY**

### **1. ESPECIFICAÇÕES E CÁLCULOS**

#### **Configuração da Cache:**

```
• Número de linhas: 8
• Tamanho do bloco: 4 palavras = 128 bits = 16 bytes
• Política de mapeamento: Direto
• Política de escrita: Write-back
• Endereço da RAM principal: 20 bits
• Largura de palavra: 32 bits
```

#### **Divisão do Endereço (20 bits):**

```
┌─────────────┬─────────┬───────────┐
│ Tag (15 bits)│Index(3) │ Offset(2) │
└─────────────┴─────────┴───────────┘
 19......7      6..4      3....2
```

- **Offset**: 2 bits → seleciona 1 das 4 palavras no bloco
- **Index**: 3 bits → seleciona 1 das 8 linhas (2³ = 8)
- **Tag**: 15 bits → identifica o bloco armazenado

**Nota**: Considerando endereçamento de palavras (últimos 2 bits do endereço de byte são 00)

---

### **2. ESTRUTURA DE UMA LINHA DE CACHE**

```
┌───────┬───────┬──────────────┬─────────────────────────────┐
│ Valid │ Dirty │   Tag (15)   │    Data Block (128 bits)    │
│ (1b)  │ (1b)  │              │  [Word3|Word2|Word1|Word0]  │
└───────┴───────┴──────────────┴─────────────────────────────┘
 1bit    1bit     15 bits          4 × 32 bits = 128 bits

Total por linha: 1 + 1 + 15 + 128 = 145 bits
Total da cache: 145 × 8 = 1160 bits
```

---

### **3. COMPONENTES DO SISTEMA DE CACHE**

#### **A) Blocos Principais:**

```
┌─────────────────────────────────────────────────────┐
│                   CPU REQUEST                        │
│              (Endereço 20b, Data 32b)               │
└────────────────────┬────────────────────────────────┘
                   │
                   ▼
       ┌───────────────────────────┐
       │   ADDRESS DECODER         │
       │  • Extrai Tag, Index, Offset│
       └──────────┬────────────────┘
                  │
    ┌─────────────┴──────────────┐
    │                            │
    ▼                            ▼
┌──────────────┐           ┌────────────────┐
│ CACHE MEMORY │           │  TAG COMPARATOR│
│  8 × 145 bits│◄─────────►│  & HIT LOGIC   │
└──────┬───────┘           └────────┬───────┘
     │                            │
     │        HIT/MISS            │
     └────────────┬───────────────┘
                  │
     ┌────────────┴────────────┐
     │                         │
     ▼ HIT                     ▼ MISS
┌──────────────┐         ┌─────────────────┐
│ DIRECT ACCESS│         │ CACHE CONTROLLER│
│              │         │  (State Machine) │
└──────────────┘         └────────┬─────────┘
                                │
                       ┌────────┴────────┐
                       │ MAIN MEMORY     │
                       │ INTERFACE       │
                       └─────────────────┘
```

---

### **4. MÁQUINA DE ESTADOS DO CONTROLADOR**

```
                  ┌──────────┐
                  │   IDLE   │◄───────────┐
                  └─────┬────┘            │
                        │                 │
                 CPU Request              │
                        │                 │
                  ┌─────▼────┐            │
                  │  COMPARE │            │
                  │ Tag/Valid│            │
                  └─────┬────┘            │
                        │                 │
            ┌───────────┴──────────┐      │
            │                      │      │
        HIT │                      │ MISS │
            │                      │      │
    ┌───────▼─────┐        ┌──────▼──────┴┐
    │  READ/WRITE │        │ CHECK DIRTY  │
    │    CACHE    │        └──────┬───────┘
    └───────┬─────┘               │
            │            ┌─────────┴──────────┐
            │        Dirty=1             Dirty=0
            │            │                    │
            │    ┌───────▼─────────┐   ┌──────▼──────┐
            │    │  WRITE_BACK     │   │  ALLOCATE   │
            │    │ (4 palavras→RAM)│   │             │
            │    └───────┬─────────┘   └──────┬──────┘
            │            │                    │
            │            └────────┬───────────┘
            │                     │
            │             ┌───────▼──────────┐
            │             │   FETCH_BLOCK    │
            │             │ (4 palavras←RAM) │
            │             └───────┬──────────┘
            │                     │
            │             ┌───────▼──────────┐
            │             │   UPDATE_CACHE   │
            │             │  (Valid=1, Tag)  │
            │             └───────┬──────────┘
            │                     │
            └─────────────────────┘
```

**Estados:**

- **IDLE**: Aguardando requisição
- **COMPARE**: Verifica hit/miss
- **READ/WRITE CACHE**: Acesso direto (hit)
- **CHECK_DIRTY**: Verifica se bloco está modificado
- **WRITE_BACK**: Salva bloco modificado na RAM
- **ALLOCATE**: Prepara para novo bloco
- **FETCH_BLOCK**: Busca 4 palavras da RAM
- **UPDATE_CACHE**: Atualiza linha com novo bloco

---

### **5. SINAIS DE CONTROLE**

```
┌─────────────────┬────────┬──────────────────────────────┐
│ Sinal           │ Bits   │ Função                       │
├─────────────────┼────────┼──────────────────────────────┤
│ cpu_addr        │ 20     │ Endereço da CPU              │
│ cpu_data_in     │ 32     │ Dados de escrita             │
│ cpu_data_out    │ 32     │ Dados de leitura             │
│ cpu_read        │ 1      │ Requisição de leitura        │
│ cpu_write       │ 1      │ Requisição de escrita        │
│ cpu_ready       │ 1      │ Operação concluída (output)  │
├─────────────────┼────────┼──────────────────────────────┤
│ cache_hit       │ 1      │ Tag match & Valid            │
│ cache_miss      │ 1      │ ~cache_hit                   │
│ cache_dirty     │ 1      │ Bloco modificado             │
├─────────────────┼────────┼──────────────────────────────┤
│ mem_addr        │ 20     │ Endereço para RAM            │
│ mem_data_in     │ 128    │ Bloco vindo da RAM           │
│ mem_data_out    │ 128    │ Bloco para RAM               │
│ mem_read        │ 1      │ Ler bloco da RAM             │
│ mem_write       │ 1      │ Escrever bloco na RAM        │
│ mem_ready       │ 1      │ RAM pronta                   │
├─────────────────┼────────┼──────────────────────────────┤
│ update_cache    │ 1      │ Habilita escrita na cache    │
│ set_dirty       │ 1      │ Marca linha como suja        │
│ clear_dirty     │ 1      │ Limpa bit dirty              │
│ set_valid       │ 1      │ Marca linha como válida      │
└─────────────────┴────────┴──────────────────────────────┘
```

---

### **6. IMPLEMENTAÇÃO EM LOGISIM - MÓDULOS**

#### **Módulo 1: CacheLineStorage**

```
Entradas:
- index (3 bits)
- write_enable (1 bit)
- data_in (128 bits)
- tag_in (15 bits)
- valid_in, dirty_in (1 bit cada)

Saídas:
- data_out (128 bits)
- tag_out (15 bits)
- valid_out, dirty_out (1 bit cada)

Componentes:
- 8 × Registradores de 145 bits
- Multiplexador 8:1 (para leitura)
- Demultiplexador 1:8 (para escrita)
```

#### **Módulo 2: TagComparator**

```
Entradas:
- addr_tag (15 bits)
- cache_tag (15 bits)
- valid (1 bit)

Saídas:
- hit (1 bit) = (addr_tag == cache_tag) AND valid

Componentes:
- Comparador de 15 bits
- AND gate
```

#### **Módulo 3: AddressDecoder**

```
Entrada:
- address (20 bits)

Saídas:
- tag (15 bits) = address[19:5]
- index (3 bits) = address[4:2]
- offset (2 bits) = address[1:0]

Componentes:
- Splitters
```

#### **Módulo 4: CacheController (FSM)**

```
Entradas:
- clock, reset
- cpu_read, cpu_write
- cache_hit, cache_dirty
- mem_ready

Saídas:
- cpu_ready
- mem_read, mem_write
- update_cache
- set_dirty, clear_dirty, set_valid
- state (3 bits para debug)

Componentes:
- Registrador de estado (3 bits)
- Lógica combinacional para transições
- ROM de microinstruções (opcional)
```

#### **Módulo 5: BlockMultiplexer**

```
Entradas:
- block_data (128 bits)
- offset (2 bits)

Saída:
- word_out (32 bits)

Componentes:
- Multiplexador 4:1 de 32 bits
```

#### **Módulo 6: MemoryInterface**

```
Entradas:
- mem_read, mem_write
- addr (20 bits)
- block_out (128 bits)

Saídas:
- block_in (128 bits)
- mem_ready

Componentes:
- 4 × registradores temporários
- Contador para transferência sequencial
- Interface com RAM principal
```

---
### **8. LÓGICA DE OPERAÇÃO DETALHADA**

#### **a) READ HIT:**

```
1. Decode address → Tag, Index, Offset
2. Access cache[Index] → Read Tag, Valid, Dirty, Data
3. Compare Tags AND Check Valid → HIT
4. Select word using Offset → Mux 4:1
5. Return data to CPU → cpu_data_out
6. Assert cpu_ready
Cycles: 1 ciclo
```

#### **b) READ MISS:**

```
1. Decode address → Tag, Index, Offset
2. Access cache[Index] → Miss detected
3. Check Dirty bit:
 
 IF Dirty = 1:
   a) Prepare old_tag + index → calculate RAM address
   b) Write entire block (4 words) to RAM
   c) Wait mem_ready (4 cycles for 4 words)
 
4. Calculate block address = {Tag, Index, 00}
5. Read 4 words from RAM starting at block address
6. Wait mem_ready (4 cycles)
7. Update cache line:
 - Data ← new block
 - Tag ← new tag
 - Valid ← 1
 - Dirty ← 0
8. Select word using Offset
9. Return data to CPU
10. Assert cpu_ready

Cycles: 
- Clean miss: 4-5 ciclos
- Dirty miss: 8-10 ciclos
```

#### **c) WRITE HIT:**

```
1. Decode address → Tag, Index, Offset
2. Access cache[Index] → HIT
3. Update specific word in block:
 - Use offset to select which word
 - Replace word with cpu_data_in
4. Set Dirty ← 1 (marca bloco modificado)
5. Keep Valid ← 1
6. Assert cpu_ready

Cycles: 1 ciclo
```

#### **d) WRITE MISS:**

```
Política: Write-Allocate (busca bloco antes de escrever)

1. Decode address → Tag, Index, Offset
2. Miss detected
3. Check Dirty bit (mesmo que READ MISS)
4. Fetch new block from RAM
5. Update cache with new block
6. Perform WRITE HIT procedure
7. Set Dirty ← 1
8. Assert cpu_ready

Cycles: 5-10 ciclos (similar ao READ MISS + 1)
```

---

### **9. TRANSFERÊNCIA DE BLOCO (4 PALAVRAS)**

```
Block Transfer (RAM ↔ Cache):

Endereço base do bloco: {Tag, Index, 00}

Write-back (Cache → RAM):
┌─────┬──────────────┬─────────────┐
│Cycle│   Address    │   Data      │
├─────┼──────────────┼─────────────┤
│  1  │ base + 0     │   Word[0]   │
│  2  │ base + 1     │   Word[1]   │
│  3  │ base + 2     │   Word[2]   │
│  4  │ base + 3     │   Word[3]   │
└─────┴──────────────┴─────────────┘

Fetch (RAM → Cache):
┌─────┬──────────────┬─────────────┐
│Cycle│   Address    │   Data      │
├─────┼──────────────┼─────────────┤
│  1  │ base + 0     │ → Word[0]   │
│  2  │ base + 1     │ → Word[1]   │
│  3  │ base + 2     │ → Word[2]   │
│  4  │ base + 3     │ → Word[3]   │
└─────┴──────────────┴─────────────┘

Otimização: Usar burst mode se RAM suportar
```


---

# Conexões

# 🔌 **GUIA COMPLETO DE CONEXÕES - CACHE MEMORY**

---

## 📍 **ÍNDICE DE MÓDULOS**

1. AddressDecoder
2. TagComparator
3. WordSelector
4. CacheLineStorage
5. BlockAssembler
6. BlockTransferController
7. CacheController (FSM)
8. CacheMemory (TOP - Integração)

---

## **1. MÓDULO: AddressDecoder**

### **1.1 - Componentes Internos:**

```
┌─────────────────────────────────────────┐
│  Pin(address) → Splitter → 3 Pins       │
│   [20 bits]      20→3    [tag,idx,off]  │
└─────────────────────────────────────────┘
```

### **1.2 - Conexões Internas:**

**Passo 1:** Colocar Pin de entrada `address` (20 bits)

- Localização sugerida: (100, 200)

**Passo 2:** Colocar Splitter

- Localização: (200, 200)
- Configuração:

```
Incoming: 20 bits
Fan-out: 3 saídas

Configuração de bits:
- Saída 0 (Tag):    bits 5-19  (15 bits)
- Saída 1 (Index):  bits 2-4   (3 bits)
- Saída 2 (Offset): bits 0-1   (2 bits)
```

**Configuração detalhada do Splitter:**

```
bit 0  → saída 2 (offset)
bit 1  → saída 2 (offset)
bit 2  → saída 1 (index)
bit 3  → saída 1 (index)
bit 4  → saída 1 (index)
bit 5  → saída 0 (tag)
bit 6  → saída 0 (tag)
...
bit 19 → saída 0 (tag)
```

**Passo 3:** Colocar Pins de saída

- `tag` (15 bits) em (400, 150)
- `index` (3 bits) em (400, 200)
- `offset` (2 bits) em (400, 250)

**Passo 4:** Fazer conexões

1. Conectar `address` (entrada) ao lado esquerdo do Splitter
2. Conectar saída 0 do Splitter ao Pin `tag`
3. Conectar saída 1 do Splitter ao Pin `index`
4. Conectar saída 2 do Splitter ao Pin `offset`

### **1.3 - Conexões Externas (quando usar este módulo):**

```
Entrada:
cpu_address[19:0] → AddressDecoder.address

Saídas:
AddressDecoder.tag[14:0] → {TagComparator, CacheLineStorage}
AddressDecoder.index[2:0] → CacheLineStorage
AddressDecoder.offset[1:0] → WordSelector
```

---

## **2. MÓDULO: TagComparator**

### **2.1 - Componentes Internos:**

```
┌──────────────────────────────────────────┐
│  tag_address ──┐                         │
│                ├─→ Comparator ─┐         │
│  tag_stored ───┘                │        │
│                                 ├─→ AND → hit
│  valid_bit ────────────────────┘         │
└──────────────────────────────────────────┘
```

### **2.2 - Conexões Internas:**

**Passo 1:** Colocar Pins de entrada

- `tag_address` (15 bits) em (100, 150)
- `tag_stored` (15 bits) em (100, 200)
- `valid_bit` (1 bit) em (100, 300)

**Passo 2:** Colocar Comparator

- Localização: (250, 175)
- Configuração: Width = 15 bits
- Componente: Arithmetic → Comparator

**Passo 3:** Colocar AND Gate

- Localização: (400, 250)
- Configuração: 2 inputs, Size = 30
- Componente: Gates → AND Gate

**Passo 4:** Colocar Pin de saída

- `hit` (1 bit) em (500, 250)

**Passo 5:** Fazer conexões

1. Conectar `tag_address` à entrada superior do Comparator
2. Conectar `tag_stored` à entrada inferior do Comparator
3. Conectar saída "=" (igual) do Comparator à primeira entrada do AND Gate

- **Atenção:** Use a saída "=" (não ">", "<")

4. Conectar `valid_bit` à segunda entrada do AND Gate
5. Conectar saída do AND Gate ao Pin `hit`

### **2.3 - Conexões Externas:**

```
Entradas:
AddressDecoder.tag → TagComparator.tag_address
CacheLineStorage.tag_out → TagComparator.tag_stored
CacheLineStorage.valid_out → TagComparator.valid_bit

Saída:
TagComparator.hit → CacheController.cache_hit
```

---

## **3. MÓDULO: WordSelector**

### **3.1 - Componentes Internos:**

```
┌─────────────────────────────────────────┐
│  word0 ──┐                              │
│  word1 ──┤                              │
│  word2 ──┼─→ Multiplexer 4:1 → word_out│
│  word3 ──┤      (32 bits)               │
│          │                              │
│  offset ─┘ (select)                     │
└─────────────────────────────────────────┘
```

### **3.2 - Conexões Internas:**

**Passo 1:** Colocar Pins de entrada (4 palavras)

- `word0` (32 bits) em (100, 150)
- `word1` (32 bits) em (100, 200)
- `word2` (32 bits) em (100, 250)
- `word3` (32 bits) em (100, 300)
- `offset` (2 bits) em (100, 400)

**Passo 2:** Colocar Multiplexer

- Localização: (400, 225)
- Configuração:

```
Select bits: 2
Data bits (Width): 32
Inputs: 4 (automático quando select=2)
```

- Componente: Plexers → Multiplexer

**Passo 3:** Colocar Pin de saída

- `word_out` (32 bits) em (550, 225)

**Passo 4:** Fazer conexões

1. Conectar `word0` à entrada 0 do Multiplexer (inferior)
2. Conectar `word1` à entrada 1 do Multiplexer
3. Conectar `word2` à entrada 2 do Multiplexer
4. Conectar `word3` à entrada 3 do Multiplexer (superior)
5. Conectar `offset` ao pino SELECT do Multiplexer (triangulinho)
6. Conectar saída do Multiplexer ao Pin `word_out`

**Ordem das entradas do Multiplexer (de baixo para cima):**

```
┌─────┐
│  3  │← word3 (offset=11)
│  2  │← word2 (offset=10)
│  1  │← word1 (offset=01)
│  0  │← word0 (offset=00)
└──▼──┘
   │
 word_out
```

### **3.3 - Conexões Externas:**

```
Entradas:
CacheLineStorage.word0_out → WordSelector.word0
CacheLineStorage.word1_out → WordSelector.word1
CacheLineStorage.word2_out → WordSelector.word2
CacheLineStorage.word3_out → WordSelector.word3
AddressDecoder.offset → WordSelector.offset

Saída:
WordSelector.word_out → cpu_read_data (se HIT)
```

---

## **4. MÓDULO: CacheLineStorage**

### **4.1 - Componentes Internos:**

```
┌─────────────────────────────────────────────────────┐
│                  METADADOS                           │
│  valid_in ──┐                                        │
│  dirty_in ──┼─→ Splitter ──→ RAM[3] ──→ Splitter ──→ valid_out
│  tag_in   ──┘   (combine)    (17b)    (separate)  ├→ dirty_out
│                                                    └→ tag_out
│                                                       │
│                  DATA BLOCK                           │
│  word0_in ──→ RAM[3] ────────────────────→ word0_out │
│               (32b)                                   │
│  word1_in ──→ RAM[3] ────────────────────→ word1_out │
│               (32b)                                   │
│  word2_in ──→ RAM[3] ────────────────────→ word2_out │
│               (32b)                                   │
│  word3_in ──→ RAM[3] ────────────────────→ word3_out │
│               (32b)                                   │
│                                                       │
│  index[2:0] ──→ (endereço para todas as RAMs)        │
│  write_enable ──→ (habilita escrita em todas)        │
│  clock ──→ (clock para todas as RAMs)                │
└─────────────────────────────────────────────────────┘
```

### **4.2 - Conexões Internas DETALHADAS:**

#### **SEÇÃO 1: RAM de Metadados**

**Passo 1:** Colocar Pins de entrada para metadados

- `valid_in` (1 bit) em (100, 300)
- `dirty_in` (1 bit) em (100, 350)
- `tag_in` (15 bits) em (100, 400)

**Passo 2:** Colocar Splitter de ENTRADA (combinar)

- Localização: (250, 350)
- Configuração:

```
Facing: WEST (←)
Incoming: 17 bits
Fan-out: 3 saídas

Mapeamento:
- bit 0 → saída 0 (valid)
- bit 1 → saída 1 (dirty)
- bits 2-16 → saída 2 (tag)
```

**Passo 3:** Colocar RAM de Metadados

- Localização: (400, 350)
- Configuração:

```
Address Width: 3 bits
Data Width: 17 bits
Appearance: Classic
Label: "MetaRAM"
```

**Passo 4:** Colocar Splitter de SAÍDA (separar)

- Localização: (550, 350)
- Configuração:

```
Facing: EAST (→)
Incoming: 17 bits
Fan-out: 3 saídas

Mesma configuração do splitter de entrada
```

**Passo 5:** Colocar Pins de saída para metadados

- `valid_out` (1 bit) em (700, 300)
- `dirty_out` (1 bit) em (700, 350)
- `tag_out` (15 bits) em (700, 400)

**Passo 6:** Conexões da RAM de Metadados

1. Conectar `valid_in` à entrada 0 do Splitter de entrada
2. Conectar `dirty_in` à entrada 1 do Splitter de entrada
3. Conectar `tag_in` à entrada 2 do Splitter de entrada
4. Conectar saída combinada do Splitter ao pino D (Data) da RAM
5. Conectar `index` ao pino A (Address) da RAM
6. Conectar `write_enable` ao pino ld (load/write enable) da RAM
7. Conectar `clock` ao pino de clock da RAM
8. Conectar saída da RAM ao Splitter de saída
9. Conectar saída 0 do Splitter de saída a `valid_out`
10. Conectar saída 1 do Splitter de saída a `dirty_out`
11. Conectar saída 2 do Splitter de saída a `tag_out`

#### **SEÇÃO 2: RAMs de Data Block (4 palavras)**

**Passo 7:** Para cada palavra (repetir 4 vezes):

**Word 0:**

- Pin entrada: `word0_in` (32 bits) em (100, 500)
- RAM: (400, 500)
- Address Width: 3
- Data Width: 32
- Label: "Word0_RAM"
- Pin saída: `word0_out` (32 bits) em (700, 500)

**Word 1:**

- Pin entrada: `word1_in` (32 bits) em (100, 600)
- RAM: (400, 600)
- Address Width: 3
- Data Width: 32
- Label: "Word1_RAM"
- Pin saída: `word1_out` (32 bits) em (700, 600)

**Word 2:**

- Pin entrada: `word2_in` (32 bits) em (100, 700)
- RAM: (400, 700)
- Address Width: 3
- Data Width: 32
- Label: "Word2_RAM"
- Pin saída: `word2_out` (32 bits) em (700, 700)

**Word 3:**

- Pin entrada: `word3_in` (32 bits) em (100, 800)
- RAM: (400, 800)
- Address Width: 3
- Data Width: 32
- Label: "Word3_RAM"
- Pin saída: `word3_out` (32 bits) em (700, 800)

**Passo 8:** Conexões comuns para as 4 RAMs de palavras

Para CADA uma das 4 RAMs:

1. Conectar `wordN_in` ao pino D (Data) da RAM
2. Conectar `index` ao pino A (Address) da RAM
3. Conectar `write_enable` ao pino ld (load) da RAM
4. Conectar `clock` ao pino de clock da RAM
5. Conectar saída da RAM a `wordN_out`

**Passo 9:** Pins de controle compartilhados

- `index` (3 bits) em (100, 150)
- `write_enable` (1 bit) em (100, 100)
- `clock` (1 bit) em (100, 200)

**⚠️ IMPORTANTE:** Os sinais `index`, `write_enable` e `clock` devem ser distribuídos para TODAS as 5 RAMs:

- Use Splitters ou conecte diretamente se o Logisim permitir
- Todos devem receber o mesmo sinal simultaneamente

### **4.3 - Conexões Externas:**

```
ENTRADAS:
AddressDecoder.index → CacheLineStorage.index
CacheController.write_enable → CacheLineStorage.write_enable
clock → CacheLineStorage.clock

Metadados (ao atualizar linha):
CacheController.set_valid → CacheLineStorage.valid_in
CacheController.set_dirty → CacheLineStorage.dirty_in
AddressDecoder.tag → CacheLineStorage.tag_in

Dados (do BlockAssembler):
BlockAssembler.word0_out → CacheLineStorage.word0_in
BlockAssembler.word1_out → CacheLineStorage.word1_in
BlockAssembler.word2_out → CacheLineStorage.word2_in
BlockAssembler.word3_out → CacheLineStorage.word3_in

SAÍDAS:
CacheLineStorage.valid_out → TagComparator.valid_bit
CacheLineStorage.dirty_out → CacheController.cache_dirty
CacheLineStorage.tag_out → TagComparator.tag_stored

CacheLineStorage.word0_out → {WordSelector, BlockTransferController}
CacheLineStorage.word1_out → {WordSelector, BlockTransferController}
CacheLineStorage.word2_out → {WordSelector, BlockTransferController}
CacheLineStorage.word3_out → {WordSelector, BlockTransferController}
```

---

## **5. MÓDULO: BlockAssembler**

### **5.1 - Componentes Internos:**

```
┌──────────────────────────────────────────────────┐
│                                                  │
│  word_in ──→ Demux 1:4 ──┬→ Reg0 ──→ word0_out  │
│                          ├→ Reg1 ──→ word1_out  │
│  word_index (select)     ├→ Reg2 ──→ word2_out  │
│                          └→ Reg3 ──→ word3_out  │
│                                                  │
│  write_enable ──→ (habilita demux)              │
│  clock ──→ (clock para todos registradores)     │
└──────────────────────────────────────────────────┘
```

### **5.2 - Conexões Internas:**

**Passo 1:** Colocar Pins de entrada

- `word_in` (32 bits) em (100, 150)
- `word_index` (2 bits) em (100, 200)
- `write_enable` (1 bit) em (100, 250)
- `clock` (1 bit) em (100, 300)

**Passo 2:** Colocar Demultiplexer

- Localização: (250, 200)
- Configuração:

```
Select bits: 2
Data bits (Width): 32
Enable: false (ou conectar write_enable se disponível)
```

**Passo 3:** Colocar 4 Registradores

- Reg0 em (400, 150) → Width: 32, Label: "Word0_Reg"
- Reg1 em (400, 230) → Width: 32, Label: "Word1_Reg"
- Reg2 em (400, 310) → Width: 32, Label: "Word2_Reg"
- Reg3 em (400, 390) → Width: 32, Label: "Word3_Reg"

**Passo 4:** Colocar Pins de saída

- `word0_out` (32 bits) em (700, 150)
- `word1_out` (32 bits) em (700, 230)
- `word2_out` (32 bits) em (700, 310)
- `word3_out` (32 bits) em (700, 390)

**Passo 5:** Colocar 4 AND Gates (para controle de escrita individual)

- AND0 em (300, 160)
- AND1 em (300, 240)
- AND2 em (300, 320)
- AND3 em (300, 400)
- Configuração: 2 inputs cada

**Passo 6:** Fazer conexões

1. **Demultiplexer:**

- Conectar `word_in` à entrada do Demux
- Conectar `word_index` ao SELECT do Demux
- Saída 0 → Reg0.D
- Saída 1 → Reg1.D
- Saída 2 → Reg2.D
- Saída 3 → Reg3.D

2. **Controle de escrita para cada registrador:**

- Conectar cada saída de ENABLE do Demux a uma entrada do respectivo AND Gate
- Conectar `write_enable` à outra entrada de TODOS os AND gates
- Conectar saída de AND0 a Reg0.ld (load enable)
- Conectar saída de AND1 a Reg1.ld
- Conectar saída de AND2 a Reg2.ld
- Conectar saída de AND3 a Reg3.ld

3. **Clock:**

- Conectar `clock` ao clock de TODOS os 4 registradores

4. **Saídas:**

- Conectar Reg0.Q a `word0_out`
- Conectar Reg1.Q a `word1_out`
- Conectar Reg2.Q a `word2_out`
- Conectar Reg3.Q a `word3_out`

### **5.3 - Conexões Externas:**

```
ENTRADAS:
mem_read_data → BlockAssembler.word_in (durante FETCH)
CacheController.word_counter → BlockAssembler.word_index
CacheController.word_cnt_enable → BlockAssembler.write_enable
clock → BlockAssembler.clock

SAÍDAS:
BlockAssembler.word0_out → CacheLineStorage.word0_in
BlockAssembler.word1_out → CacheLineStorage.word1_in
BlockAssembler.word2_out → CacheLineStorage.word2_in
BlockAssembler.word3_out → CacheLineStorage.word3_in
```

---

## **6. MÓDULO: BlockTransferController**

### **6.1 - Componentes Internos:**

```
┌────────────────────────────────────────────────────┐
│  Counter ──→ comparator == 3? ──→ transfer_done   │
│    │                                               │
│    ├──→ Adder ──→ mem_address                     │
│    │      └─ base_address                          │
│    │                                               │
│    └──→ Multiplexer 4:1 ──→ mem_data_out          │
│           ├─ word0_in                              │
│           ├─ word1_in                              │
│           ├─ word2_in                              │
│           └─ word3_in                              │
│                                                    │
│  transfer_type ──→ mem_read/mem_write              │
└────────────────────────────────────────────────────┘
```

### **6.2 - Conexões Internas DETALHADAS:**

**Passo 1:** Colocar Pins de entrada

- `clock` (1 bit) em (100, 100)
- `start_transfer` (1 bit) em (100, 150)
- `transfer_type` (1 bit) em (100, 200) ← 0=READ, 1=WRITE
- `base_address` (20 bits) em (100, 250)
- `word0_in` (32 bits) em (100, 350)
- `word1_in` (32 bits) em (100, 400)
- `word2_in` (32 bits) em (100, 450)
- `word3_in` (32 bits) em (100, 500)

**Passo 2:** Colocar Counter (contador de palavras)

- Localização: (300, 150)
- Configuração:

```
Width: 2 bits
Maximum: 3 (0x3)
Label: "WordCounter"
```

**Passo 3:** Colocar Comparator (detecta quando counter=3)

- Localização: (400, 150)
- Configuração: Width = 2 bits

**Passo 4:** Colocar Constant (valor 3)

- Localização: (350, 170)
- Configuração: Value = 0x3, Width = 2

**Passo 5:** Colocar Bit Extender (2 bits → 20 bits)

- Localização: (350, 260)
- Configuração:

```
Input Width: 2
Output Width: 20
Type: Zero extend
```

**Passo 6:** Colocar Adder (soma base + counter)

- Localização: (400, 250)
- Configuração: Width = 20 bits

**Passo 7:** Colocar Multiplexer 4:1 (seleciona palavra a escrever)

- Localização: (450, 400)
- Configuração:

```
Select: 2 bits
Width: 32 bits
```

**Passo 8:** Colocar 2 AND Gates (controle read/write)

- AND_READ em (600, 370)
- AND_WRITE em (600, 420)
- Configuração: 2 inputs, com inversor na primeira entrada do READ

**Passo 9:** Colocar NOT Gate (inverter transfer_type)

- Localização: (550, 360)

**Passo 10:** Colocar Pins de saída

- `mem_address` (20 bits) em (700, 250)
- `mem_data_out` (32 bits) em (700, 320)
- `mem_read` (1 bit) em (700, 370)
- `mem_write` (1 bit) em (700, 420)
- `transfer_done` (1 bit) em (700, 150)

**Passo 11:** Fazer todas as conexões

1. **Counter:**

- Conectar `clock` ao clock do Counter
- Conectar `start_transfer` ao enable do Counter
- Saída do Counter vai para:
    - Comparator
    - Bit Extender
    - Multiplexer SELECT

2. **Comparator (transfer_done):**

- Conectar saída do Counter à entrada A
- Conectar Constant (3) à entrada B
- Conectar saída "=" a `transfer_done`

3. **Cálculo de endereço:**

- Conectar saída do Counter ao Bit Extender
- Conectar `base_address` à entrada superior do Adder
- Conectar saída do Bit Extender à entrada inferior do Adder
- Conectar saída do Adder a `mem_address`

4. **Seleção de palavra para escrita:**

- Conectar `word0_in` à entrada 0 do Multiplexer
- Conectar `word1_in` à entrada 1 do Multiplexer
- Conectar `word2_in` à entrada 2 do Multiplexer
- Conectar `word3_in` à entrada 3 do Multiplexer
- Conectar saída do Counter ao SELECT do Multiplexer
- Conectar saída do Multiplexer a `mem_data_out`

5. **Controle de leitura/escrita:**

- Conectar `transfer_type` ao NOT Gate
- Conectar `transfer_type` à primeira entrada de AND_WRITE
- Conectar saída do NOT Gate à primeira entrada de AND_READ
- Conectar `start_transfer` à segunda entrada de AMBOS AND Gates
- Conectar saída de AND_READ a `mem_read`
- Conectar saída de AND_WRITE a `mem_write`

### **6.3 - Conexões Externas:**

```
ENTRADAS:
clock → BlockTransferController.clock
CacheController.start_transfer → BlockTransferController.start_transfer
CacheController.transfer_type → BlockTransferController.transfer_type
{AddressDecoder.tag, AddressDecoder.index, 00} → base_address

CacheLineStorage.word0_out → BlockTransferController.word0_in
CacheLineStorage.word1_out → BlockTransferController.word1_in
CacheLineStorage.word2_out → BlockTransferController.word2_in
CacheLineStorage.word3_out → BlockTransferController.word3_in

SAÍDAS:
BlockTransferController.mem_address → RAM.address
BlockTransferController.mem_data_out → RAM.data_in
BlockTransferController.mem_read → RAM.read_enable
BlockTransferController.mem_write → RAM.write_enable
BlockTransferController.transfer_done → CacheController.transfer_done
```

---

## **7. MÓDULO: CacheController (FSM)**

### **7.1 - Componentes Internos:**

```
┌─────────────────────────────────────────────────────┐
│  [Estado Atual] ──┬──→ NextStateROM ──→ [Próx Est] │
│       (3b)        │         ↑                       │
│                   │         └─ [Entradas]           │
│                   │                                 │
│                   └──→ ControlROM ──→ [Sinais Out]  │
│                                                     │
│  WordCounter ──→ comparator == 3? ──→ transfer_done│
└─────────────────────────────────────────────────────┘
```

### **7.2 - Conexões Internas DETALHADAS:**

#### **SEÇÃO 1: Registrador de Estado**

**Passo 1:** Colocar Pins de entrada

- `clock` em (100, 100)
- `reset` em (100, 150)
- `cpu_read` em (100, 200)
- `cpu_write` em (100, 250)
- `cache_hit` em (100, 300)
- `cache_dirty` em (100, 350)
- `mem_ready` em (100, 400)

**Passo 2:** Colocar Constant (estado IDLE = 000)

- Localização: (200, 230)
- Configuração: Value = 0x0, Width = 3

**Passo 3:** Colocar Multiplexer (reset ou próximo estado)

- Localização: (250, 240)
- Configuração:

```
Select: 1 bit
Width: 3 bits
```

**Passo 4:** Colocar Register (estado atual)

- Localização: (300, 250)
- Configuração:

```
Width: 3 bits
Label: "StateReg"
```

**Conexões desta seção:**

1. Conectar Constant (000) à entrada 0 do Multiplexer
2. Conectar saída da NextStateROM à entrada 1 do Multiplexer
3. Conectar `reset` ao SELECT do Multiplexer
4. Conectar saída do Multiplexer ao D do Register
5. Conectar `clock` ao clock do Register
6. Conectar saída do Register:

- Ao endereço da ControlROM
- Como parte do endereço da NextStateROM
- Ao Pin `state_debug`

#### **SEÇÃO 2: NextStateROM (Lógica de Transição)**

**Passo 5:** Colocar Splitter (combinar estado + entradas)

- Localização: (400, 280)
- Configuração:

```
Facing: WEST (←)
Incoming: 9 bits
Fan-out: 7

Mapeamento:
bit 0-1: word_counter (2 bits) → saída 0
bit 2: mem_ready (1 bit) → saída 1
bit 3: cache_dirty (1 bit) → saída 2
bit 4: cache_hit (1 bit) → saída 3
bit 5: cpu_write (1 bit) → saída 4
bit 6: cpu_read (1 bit) → saída 5
bit 7-8: estado_atual (2 bits) → saída 6
```

**Passo 6:** Colocar ROM (NextStateROM)

- Localização: (500, 300)
- Configuração:

```
Address Width: 9 bits
Data Width: 3 bits
Label: "NextStateROM"
Contents: (ver tabela abaixo)
```

**Conexões desta seção:**

1. Conectar saída do StateReg à entrada 6 do Splitter
2. Conectar `cpu_read` à entrada 5 do Splitter
3. Conectar `cpu_write` à entrada 4 do Splitter
4. Conectar `cache_hit` à entrada 3 do Splitter
5. Conectar `cache_dirty` à entrada 2 do Splitter
6. Conectar `mem_ready` à entrada 1 do Splitter
7. Conectar saída do WordCounter à entrada 0 do Splitter
8. Conectar saída combinada do Splitter ao endereço da NextStateROM
9. Conectar saída da ROM à entrada 1 do Multiplexer (do Passo 3)

#### **SEÇÃO 3: ControlROM (Sinais de Saída)**

**Passo 7:** Colocar ROM (ControlROM)

- Localização: (500, 500)
- Configuração:

```
Address Width: 3 bits
Data Width: 10 bits
Label: "ControlROM"
Contents: (ver tabela abaixo)
```

**Passo 8:** Colocar Splitter de saída (separar sinais)

- Localização: (600, 500)
- Configuração:

```
Facing: EAST (→)
Incoming: 10 bits
Fan-out: 10

bit 0: start_transfer
bit 1: word_cnt_reset
bit 2: word_cnt_enable
bit 3: set_valid
bit 4: clear_dirty
bit 5: set_dirty
bit 6: update_cache
bit 7: mem_write
bit 8: mem_read
bit 9: cpu_ready
```

**Passo 9:** Colocar Pins de saída

- `cpu_ready` em (800, 420)
- `mem_read` em (800, 450)
- `mem_write` em (800, 480)
- `update_cache` em (800, 510)
- `set_dirty` em (800, 540)
- `clear_dirty` em (800, 570)
- `set_valid` em (800, 600)
- `word_cnt_enable` em (800, 630)
- `word_cnt_reset` em (800, 660)
- `start_transfer` em (800, 690)
- `state_debug` (3 bits) em (800, 250)

**Conexões desta seção:**

1. Conectar saída do StateReg ao endereço da ControlROM
2. Conectar saída da ROM ao Splitter de saída
3. Conectar cada saída do Splitter ao respectivo Pin de saída

#### **SEÇÃO 4: Contador de Palavras**

**Passo 10:** Colocar Counter

- Localização: (400, 650)
- Configuração:

```
Width: 2 bits
Maximum: 0x3
Label: "WordCounter"
```

**Passo 11:** Colocar Comparator (detecta fim)

- Localização: (500, 650)
- Configuração: Width = 2

**Passo 12:** Colocar Constant (3)

- Localização: (450, 670)
- Configuração: Value = 0x3, Width = 2

**Passo 13:** Colocar Pin de saída

- `transfer_done` em (600, 650)

**Conexões desta seção:**

1. Conectar `word_cnt_enable` (da ControlROM) ao enable do Counter
2. Conectar `word_cnt_reset` (da ControlROM) ao clear do Counter
3. Conectar `clock` ao clock do Counter
4. Conectar saída do Counter:

- Ao Comparator
- À entrada 0 do Splitter (NextStateROM)

5. Conectar Constant à entrada B do Comparator
6. Conectar saída do Counter à entrada A do Comparator
7. Conectar saída "=" do Comparator a `transfer_done`

### **7.3 - Conteúdo das ROMs:**

#### **NextStateROM (512 entradas × 3 bits):**

**Formato do endereço:** `[estado[2:0], cpu_rd, cpu_wr, hit, dirty, mem_rdy, cnt[1:0]]`

Estados:

- 000 = IDLE
- 001 = COMPARE
- 010 = READ_WRITE
- 011 = CHECK_DIRTY
- 100 = WRITE_BACK
- 101 = ALLOCATE
- 110 = FETCH_BLOCK
- 111 = UPDATE_CACHE

**Principais transições:**

```
Estado IDLE (000):
Endereço 0x000 (sem requisição) → 000 (IDLE)
Endereço 0x008 (cpu_rd=1) → 001 (COMPARE)
Endereço 0x010 (cpu_wr=1) → 001 (COMPARE)

Estado COMPARE (001):
Endereço 0x024 (hit=1) → 010 (READ_WRITE)
Endereço 0x02C (hit=0) → 011 (CHECK_DIRTY)

Estado READ_WRITE (010):
Qualquer endereço → 000 (IDLE)

Estado CHECK_DIRTY (011):
Endereço 0x038 (dirty=0) → 101 (ALLOCATE)
Endereço 0x03C (dirty=1) → 100 (WRITE_BACK)

Estado WRITE_BACK (100):
Endereço 0x040-0x042 (cnt<3) → 100 (WRITE_BACK)
Endereço 0x043 (cnt=3) → 101 (ALLOCATE)

Estado ALLOCATE (101):
Qualquer endereço → 110 (FETCH_BLOCK)

Estado FETCH_BLOCK (110):
Endereço 0x060-0x062 (cnt<3) → 110 (FETCH_BLOCK)
Endereço 0x063 (cnt=3) → 111 (UPDATE_CACHE)

Estado UPDATE_CACHE (111):
Qualquer endereço → 010 (READ_WRITE)
```

#### **ControlROM (8 entradas × 10 bits):**

**Formato:** `[start_tr, cnt_rst, cnt_en, set_val, clr_drt, set_drt, upd_cch, mem_wr, mem_rd, cpu_rdy]`

```
Endereço 000 (IDLE):       0000_0000_00
Endereço 001 (COMPARE):    0000_0000_00
Endereço 010 (READ_WRITE): 0000_0010_01  (set_dirty se write, cpu_ready)
Endereço 011 (CHECK_DIRTY):0000_0000_00
Endereço 100 (WRITE_BACK): 1010_0000_10  (start_tr, cnt_en, mem_wr)
Endereço 101 (ALLOCATE):   0100_0000_00  (cnt_rst)
Endereço 110 (FETCH_BLOCK):1010_0001_00  (start_tr, cnt_en, mem_rd)
Endereço 111 (UPDATE_CACHE):0001_0100_00 (set_valid, upd_cache)
```

### **7.4 - Conexões Externas:**

```
ENTRADAS:
clock → CacheController.clock
reset → CacheController.reset
cpu_mem_read → CacheController.cpu_read
cpu_mem_write → CacheController.cpu_write
TagComparator.hit → CacheController.cache_hit
CacheLineStorage.dirty_out → CacheController.cache_dirty
mem_ready → CacheController.mem_ready

SAÍDAS:
CacheController.cpu_ready → CPU (stall)
CacheController.mem_read → BlockTransferController
CacheController.mem_write → BlockTransferController
CacheController.update_cache → CacheLineStorage.write_enable
CacheController.set_dirty → CacheLineStorage.dirty_in
CacheController.set_valid → CacheLineStorage.valid_in
CacheController.start_transfer → BlockTransferController
CacheController.state_debug → Display (debug)
```

---

## **8. MÓDULO: CacheMemory (TOP - INTEGRAÇÃO COMPLETA)**

### **8.1 - Componentes e Posicionamento:**

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  [CPU Interface]                                           │
│  cpu_address ──┬──→ AddressDecoder ──┬→ tag ──┐           │
│                │         │            ├→ index ─┼─→ ...    │
│                │         └────────────┴→ offset─┘          │
│                │                                            │
│  [Cache Storage & Comparison]                              │
│  AddressDecoder.tag ──┬──→ TagComparator ──→ hit          │
│  CacheLineStorage ────┘         │                          │
│                                  └──→ CacheController       │
│                                                            │
│  [Data Path]                                               │
│  CacheLineStorage ──→ WordSelector ──→ cpu_read_data      │
│  CacheLineStorage ──→ BlockTransfer ──→ RAM               │
│  BlockAssembler ──→ CacheLineStorage                       │
│                                                            │
│  [Control]                                                 │
│  CacheController ──→ [todos os módulos]                   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### **8.2 - Conexões Externas (Portas do Módulo):**

**Passo 1:** Criar Pins de interface com a CPU

- `cpu_address` (20 bits) em (100, 100)
- `cpu_write_data` (32 bits) em (100, 150)
- `cpu_mem_read` (1 bit) em (100, 200)
- `cpu_mem_write` (1 bit) em (100, 250)
- `cpu_read_data` (32 bits, OUT) em (1200, 150)
- `cpu_ready` (1 bit, OUT) em (1200, 200)

**Passo 2:** Criar Pins de interface com a RAM

- `mem_address` (20 bits, OUT) em (1200, 500)
- `mem_write_data` (32 bits, OUT) em (1200, 550)
- `mem_read_data` (32 bits) em (100, 700)
- `mem_read` (1 bit, OUT) em (1200, 600)
- `mem_write` (1 bit, OUT) em (1200, 650)
- `mem_ready` (1 bit) em (100, 750)

**Passo 3:** Criar Pins de controle

- `clock` (1 bit) em (100, 400)
- `reset` (1 bit) em (100, 450)
- `cache_enable` (1 bit) em (100, 500) [opcional]

**Passo 4:** Criar Pins de debug

- `state_debug` (3 bits, OUT) em (1200, 800)
- Hex Display em (1000, 800)
- LED "Cache_Hit" em (1000, 850)
- LED "Cache_Dirty" em (1000, 900)

### **8.3 - Conexões Internas (Entre Submódulos):**

#### **GRUPO 1: Decodificação de Endereço**

**Passo 5:** Instanciar AddressDecoder em (300, 100)

Conexões:

```
cpu_address → AddressDecoder.address

AddressDecoder.tag → {TagComparator.tag_address, CacheLineStorage.tag_in}
AddressDecoder.index → CacheLineStorage.index
AddressDecoder.offset → WordSelector.offset
```

#### **GRUPO 2: Cache Storage**

**Passo 6:** Instanciar CacheLineStorage em (500, 400)

Conexões de controle:

```
AddressDecoder.index → CacheLineStorage.index
CacheController.update_cache → CacheLineStorage.write_enable
clock → CacheLineStorage.clock
```

Conexões de dados (entrada):

```
BlockAssembler.word0_out → CacheLineStorage.word0_in
BlockAssembler.word1_out → CacheLineStorage.word1_in
BlockAssembler.word2_out → CacheLineStorage.word2_in
BlockAssembler.word3_out → CacheLineStorage.word3_in
```

Conexões de metadados (entrada):

```
CacheController.set_valid → CacheLineStorage.valid_in
CacheController.set_dirty → CacheLineStorage.dirty_in
AddressDecoder.tag → CacheLineStorage.tag_in
```

Conexões (saída):

```
CacheLineStorage.tag_out → TagComparator.tag_stored
CacheLineStorage.valid_out → TagComparator.valid_bit
CacheLineStorage.dirty_out → CacheController.cache_dirty

CacheLineStorage.word0_out → {WordSelector.word0, BlockTransferController.word0_in}
CacheLineStorage.word1_out → {WordSelector.word1, BlockTransferController.word1_in}
CacheLineStorage.word2_out → {WordSelector.word2, BlockTransferController.word2_in}
CacheLineStorage.word3_out → {WordSelector.word3, BlockTransferController.word3_in}
```

#### **GRUPO 3: Tag Comparison (HIT/MISS)**

**Passo 7:** Instanciar TagComparator em (500, 150)

Conexões:

```
AddressDecoder.tag → TagComparator.tag_address
CacheLineStorage.tag_out → TagComparator.tag_stored
CacheLineStorage.valid_out → TagComparator.valid_bit

TagComparator.hit → CacheController.cache_hit
TagComparator.hit → LED "Cache_Hit"
```

#### **GRUPO 4: Word Selection (READ HIT Path)**

**Passo 8:** Instanciar WordSelector em (700, 200)

Conexões:

```
CacheLineStorage.word0_out → WordSelector.word0
CacheLineStorage.word1_out → WordSelector.word1
CacheLineStorage.word2_out → WordSelector.word2
CacheLineStorage.word3_out → WordSelector.word3
AddressDecoder.offset → WordSelector.offset

WordSelector.word_out → [MUX] → cpu_read_data
```

**Passo 9:** Colocar Multiplexer (seleciona entre cache e RAM na saída)

- Localização: (900, 180)
- Configuração: Select=1, Width=32

Conexões:

```
WordSelector.word_out → MUX entrada 0
mem_read_data → MUX entrada 1
CacheController.cache_hit → MUX SELECT (invertido se necessário)

MUX saída → cpu_read_data
```

#### **GRUPO 5: Block Transfer (MISS Path)**

**Passo 10:** Instanciar BlockTransferController em (700, 600)

Conexões:

```
clock → BlockTransferController.clock
CacheController.start_transfer → BlockTransferController.start_transfer
CacheController.transfer_type → BlockTransferController.transfer_type

[Base Address Construction]:
AddressDecoder.tag → Splitter (15 bits)
AddressDecoder.index → Splitter (3 bits)
Constant (00) → Splitter (2 bits)
Splitter saída (20 bits) → BlockTransferController.base_address

CacheLineStorage.word0_out → BlockTransferController.word0_in
CacheLineStorage.word1_out → BlockTransferController.word1_in
CacheLineStorage.word2_out → BlockTransferController.word2_in
CacheLineStorage.word3_out → BlockTransferController.word3_in

BlockTransferController.mem_address → mem_address (saída)
BlockTransferController.mem_data_out → mem_write_data (saída)
BlockTransferController.mem_read → mem_read (saída)
BlockTransferController.mem_write → mem_write (saída)
BlockTransferController.transfer_done → CacheController.transfer_done
```

**Passo 11:** Colocar Splitter para construir base_address

- Localização: (600, 600)
- Configuração:

```
Facing: WEST
Incoming: 20 bits
Fan-out: 3

bit 0-1: Constant 00 (2 bits)
bit 2-4: index (3 bits)
bit 5-19: tag (15 bits)
```

**Passo 12:** Colocar Constant (00)

- Localização: (550, 620)
- Configuração: Value=0, Width=2

#### **GRUPO 6: Block Assembly (Fetch Path)**

**Passo 13:** Instanciar BlockAssembler em (700, 450)

Conexões:

```
mem_read_data → BlockAssembler.word_in
CacheController.word_counter → BlockAssembler.word_index
CacheController.word_cnt_enable → BlockAssembler.write_enable
clock → BlockAssembler.clock

BlockAssembler.word0_out → CacheLineStorage.word0_in
BlockAssembler.word1_out → CacheLineStorage.word1_in
BlockAssembler.word2_out → CacheLineStorage.word2_in
BlockAssembler.word3_out → CacheLineStorage.word3_in
```

#### **GRUPO 7: Cache Controller (Master Control)**

**Passo 14:** Instanciar CacheController em (500, 600)

Conexões de entrada:

```
clock → CacheController.clock
reset → CacheController.reset
cpu_mem_read → CacheController.cpu_read
cpu_mem_write → CacheController.cpu_write
TagComparator.hit → CacheController.cache_hit
CacheLineStorage.dirty_out → CacheController.cache_dirty
mem_ready → CacheController.mem_ready
```

Conexões de saída para outros módulos:

```
CacheController.cpu_ready → cpu_ready (pin saída)
CacheController.update_cache → CacheLineStorage.write_enable
CacheController.set_dirty → CacheLineStorage.dirty_in
CacheController.clear_dirty → [lógica para limpar]
CacheController.set_valid → CacheLineStorage.valid_in
CacheController.start_transfer → BlockTransferController.start_transfer
CacheController.word_cnt_enable → BlockAssembler.write_enable
CacheController.word_counter → BlockAssembler.word_index
CacheController.state_debug → state_debug (pin saída)
CacheController.state_debug → Hex Display
```

### **8.4 - Lógica Adicional Necessária:**

#### **Construção do Base Address:**

**Passo 15:** Implementar lógica para criar base_address

```
AddressDecoder ──→ tag[14:0] ──┐
               ├→ index[2:0] ──┤
                               ├──→ Splitter ──→ base_address[19:0]
Constant(00) ──→ offset[1:0] ──┘                    para BlockTransferController
```

Posicionar componentes:

- Splitter combinador em (600, 600)
- Constant (00, 2 bits) em (550, 620)

#### **Controle de Write Enable:**

**Passo 16:** Lógica para escrever na cache

Criar OR gate para combinar sinais:

```
CacheController.update_cache ──┐
                             ├──→ OR ──→ CacheLineStorage.write_enable
cpu_mem_write AND hit ─────────┘
```

#### **Lógica de Transfer Type:**

**Passo 17:** Determinar se transferência é READ ou WRITE

```
CacheController.state == WRITE_BACK? 1 : 0 → transfer_type
```

Usar um comparador:

- Comparar state_debug com constante 100 (WRITE_BACK)
- Saída "=" vai para BlockTransferController.transfer_type

### **8.5 - Distribuição de Sinais Globais:**

**Clock Distribution:**

```
clock (entrada) ──┬──→ CacheLineStorage.clock
                ├──→ CacheController.clock
                ├──→ BlockTransferController.clock
                └──→ BlockAssembler.clock
```

**Reset Distribution:**

```
reset (entrada) ──→ CacheController.reset
```

### **8.6 - Conexões de Debug:**

**Passo 18:** Adicionar componentes de visualização

```
CacheController.state_debug ──→ Hex Digit Display
                            └→ state_debug (pin OUT)

TagComparator.hit ──→ LED "Cache Hit"
CacheLineStorage.dirty_out ──→ LED "Dirty"
```

---

## **10. DICAS DE DEPURAÇÃO**

### **Sinais para Monitorar:**

```
1. state_debug → Ver estado da FSM
2. cache_hit → Verificar hits/misses
3. cache_dirty → Ver se linha está modificada
4. word_counter → Acompanhar transferências
5. mem_read/mem_write → Ver acessos à RAM
6. cpu_ready → Ver quando CPU pode prosseguir
```

### **Testes Sugeridos:**

```
TESTE 1: READ HIT
- Pré-carregar cache[0] com dados conhecidos
- Fazer leitura do mesmo endereço
- Verificar: 1 ciclo, hit=1, cpu_ready=1

TESTE 2: READ MISS (Clean)
- Linha válida=0
- Fazer leitura
- Verificar: ~5 ciclos, 4 acessos à RAM

TESTE 3: WRITE HIT
- Linha válida=1, tag match
- Fazer escrita
- Verificar: dirty=1, dados atualizados

TESTE 4: WRITE MISS (Dirty)
- Linha válida=1, tag diferente, dirty=1
- Fazer leitura de novo endereço
- Verificar: write-back + fetch (~10 ciclos)
```


---


# Documentação: Processador MIPS Simplificado (Ciclo Único)

## 1. Visão Geral

Este projeto, desenvolvido na ferramenta Logisim-evolution, implementa a arquitetura de um **processador de 32 bits com um conjunto de instruções reduzido (RISC)**, baseado na clássica arquitetura MIPS.

O seu principal objetivo é educacional: demonstrar de forma visual e funcional os componentes fundamentais de um processador e como eles interagem para executar um programa.

**Características Principais:**

- **Arquitetura:** MIPS-like (RISC).
- **Implementação:** Ciclo Único (Single-Cycle). Isso significa que cada instrução leva exatamente um ciclo de clock para ser concluída, desde a busca até a escrita do resultado.
- **Largura de Dados:** 32 bits.

---

## 2. Arquitetura e Componentes Principais

O processador é dividido em duas partes lógicas principais: o **Caminho de Dados (Datapath)**, que processa os dados, e a **Unidade de Controle (Control Unit)**, que comanda o caminho de dados.

### 2.1. Caminho de Dados (Datapath)

O caminho de dados é composto pelo hardware que armazena e manipula as informações. Os principais componentes são:

| Componente                            | Sub-circuito no Logisim | Função                                                                                                                                                                    |
| ------------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Program Counter (PC)**              | `Register` no `main`    | Armazena o endereço da próxima instrução a ser buscada na memória.                                                                                                        |
| **Memória de Instruções**             | `ROM` no `main`         | Armazena o programa (as instruções em código de máquina). É uma memória somente de leitura.                                                                               |
| **Arquivo de Registradores**          | `RegisterFile`          | Um conjunto de 32 registradores de 32 bits para armazenamento rápido e temporário de dados. Funciona como a "memória de trabalho" do processador.                         |
| **Unidade Lógica e Aritmética (ULA)** | `ULA` (ALU)             | Realiza operações como soma, subtração, NOR e comparações. É o "cérebro matemático" do processador.                                                                       |
| **Memória de Dados**                  | `RAM` no `main`         | Armazena os dados que o programa manipula. Permite leitura e escrita.                                                                                                     |
| **Multiplexadores (MUX)**             | `Multiplexer`           | Componentes que selecionam uma de várias entradas de dados para ser a sua única saída, com base em um sinal de controle. São essenciais para direcionar o fluxo de dados. |

### 2.2. Unidade de Controle (Control Unit)

A unidade de controle é o "maestro" da orquestra. Ela decodifica as instruções e gera os sinais de controle que ditam o que cada componente do caminho de dados deve fazer.

| Componente                        | Sub-circuito no Logisim | Função                                                                                                                                                                                    |
| --------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Unidade de Controle Principal** | `UC`                    | Recebe o `OpCode` (6 bits mais significativos) da instrução e gera os principais sinais de controle (`RegWrite`, `MemWrite`, `Branch`, `ALUSrc`, etc.).                                   |
| **Controle da ULA**               | `UC_ula`                | Recebe o `OpCode` e o campo `funct` (6 bits menos significativos) para instruções do Tipo-R e gera o código de 3 bits que seleciona a operação específica da ULA (soma, subtração, etc.). |

---

## 3. Fluxo de Execução de uma Instrução

Como este é um processador de ciclo único, as cinco etapas clássicas da execução de uma instrução ocorrem simultaneamente em um único pulso de clock.

1. **Busca da Instrução (Instruction Fetch - IF):**

- O endereço no **PC** é enviado para a **Memória de Instruções (`ROM`)**.
- A instrução de 32 bits correspondente àquele endereço é lida.

2. **Decodificação da Instrução (Instruction Decode - ID):**

- A instrução é dividida em seus campos (opcode, registradores, imediato).
- A **Unidade de Controle (`UC` e `UC_ula`)** decodifica o `opcode` e o `funct` para gerar todos os sinais de controle.
- Os endereços dos registradores de leitura são enviados ao **Arquivo de Registradores (`RegisterFile`)**, que disponibiliza seus valores.

3. **Execução (Execute - EX):**

- A **ULA** recebe os operandos (seja do `RegisterFile` ou do campo imediato da instrução).
- Com base no sinal da `UC_ula`, a ULA realiza a operação matemática/lógica e calcula o resultado.

4. **Acesso à Memória (Memory Access - MEM):**

- Para instruções `lw` (load word) ou `sw` (store word), o resultado da ULA (que é um endereço) é usado para acessar a **Memória de Dados (`RAM`)**.
- Se for `lw`, um dado é lido da memória.
- Se for `sw`, um dado do `RegisterFile` é escrito na memória.

5. **Escrita de Volta (Write-Back - WB):**

- O resultado final (seja da ULA ou da memória de dados) é escrito de volta no **Arquivo de Registradores**, no registrador de destino especificado pela instrução.

---

## 4. Relação com Memória Cache

O processador que analisamos **não possui uma memória cache**. Ele acessa diretamente a `ROM` (instruções) e a `RAM` (dados). Em um sistema real, isso seria extremamente ineficiente.

### 4.1. O Problema: O Gargalo da Memória

- **Conceito:** Processadores são muito mais rápidos do que a memória principal (RAM). Se o processador tiver que esperar pela RAM toda vez que precisar de um dado ou instrução, ele passará a maior parte do tempo ocioso. Isso é conhecido como **gargalo de von Neumann**.

### 4.2. A Solução: Memória Cache

- **Conceito:** A cache é uma memória pequena, extremamente rápida e cara, que fica entre o processador e a memória principal. Ela armazena cópias dos dados e instruções mais recentemente ou frequentemente usados.
- **Analogia:** Pense na sua mesa de trabalho (a cache) e na biblioteca da universidade (a RAM). Em vez de ir à biblioteca toda vez que precisar de um livro, você mantém os livros mais importantes na sua mesa. É muito mais rápido pegá-los de lá.

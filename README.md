# Resumo do trabalho

Vou criar um projeto completo de cache para o seu processador MIPS!

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

|Componente|Sub-circuito no Logisim|Função|
|---|---|---|
|**Program Counter (PC)**|`Register` no `main`|Armazena o endereço da próxima instrução a ser buscada na memória.|
|**Memória de Instruções**|`ROM` no `main`|Armazena o programa (as instruções em código de máquina). É uma memória somente de leitura.|
|**Arquivo de Registradores**|`RegisterFile`|Um conjunto de 32 registradores de 32 bits para armazenamento rápido e temporário de dados. Funciona como a "memória de trabalho" do processador.|
|**Unidade Lógica e Aritmética (ULA)**|`ULA` (ALU)|Realiza operações como soma, subtração, NOR e comparações. É o "cérebro matemático" do processador.|
|**Memória de Dados**|`RAM` no `main`|Armazena os dados que o programa manipula. Permite leitura e escrita.|
|**Multiplexadores (MUX)**|`Multiplexer`|Componentes que selecionam uma de várias entradas de dados para ser a sua única saída, com base em um sinal de controle. São essenciais para direcionar o fluxo de dados.|

### 2.2. Unidade de Controle (Control Unit)

A unidade de controle é o "maestro" da orquestra. Ela decodifica as instruções e gera os sinais de controle que ditam o que cada componente do caminho de dados deve fazer.

|Componente|Sub-circuito no Logisim|Função|
|---|---|---|
|**Unidade de Controle Principal**|`UC`|Recebe o `OpCode` (6 bits mais significativos) da instrução e gera os principais sinais de controle (`RegWrite`, `MemWrite`, `Branch`, `ALUSrc`, etc.).|
|**Controle da ULA**|`UC_ula`|Recebe o `OpCode` e o campo `funct` (6 bits menos significativos) para instruções do Tipo-R e gera o código de 3 bits que seleciona a operação específica da ULA (soma, subtração, etc.).|

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

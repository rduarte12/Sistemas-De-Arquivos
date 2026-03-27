# Guia de Implementação — Trabalho Prático 1 (SCC0503)

## Disciplina de Algoritmos e Estruturas de Dados II
### Instituto de Ciências Matemáticas e de Computação — USP São Carlos

---

## Sumário

1. [Visão Geral do Projeto](#1-visão-geral-do-projeto)
2. [Descrição do Arquivo Binário](#2-descrição-do-arquivo-binário)
3. [Registro de Cabeçalho — 17 bytes](#3-registro-de-cabeçalho--17-bytes)
4. [Registro de Dados — 80 bytes](#4-registro-de-dados--80-bytes)
5. [Tratamento de Valores Nulos](#5-tratamento-de-valores-nulos)
6. [Preenchimento com Lixo ('$')](#6-preenchimento-com-lixo-)
7. [Lógica de Integridade (campo status)](#7-lógica-de-integridade-campo-status)
8. [Gerenciamento de Registros Removidos](#8-gerenciamento-de-registros-removidos)
9. [Algoritmo de Escrita de Registro (CSV → Binário)](#9-algoritmo-de-escrita-de-registro-csv--binário)
10. [Algoritmo de Leitura e Exibição de Registro](#10-algoritmo-de-leitura-e-exibição-de-registro)
11. [Funcionalidades do Programa](#11-funcionalidades-do-programa)
12. [Fórmulas Essenciais](#12-fórmulas-essenciais)
13. [Checklist de Validação — Pontos Críticos](#13-checklist-de-validação--pontos-críticos)
14. [Restrições de Implementação](#14-restrições-de-implementação)
15. [Estrutura Sugerida de Arquivos](#15-estrutura-sugerida-de-arquivos)
16. [Makefile](#16-makefile)

---

## 1. Visão Geral do Projeto

Este trabalho implementa um sistema de arquivos binários em linguagem C para armazenar e recuperar dados de estações e linhas do metrô e da CPTM da região metropolitana de São Paulo. O programa lê dados de um arquivo CSV, grava-os em um arquivo binário com estrutura de registros de tamanho fixo (80 bytes) e oferece funcionalidades de busca sequencial e por RRN (Relative Record Number).

O arquivo binário é composto por:

- **1 registro de cabeçalho** de 17 bytes (metadados do arquivo).
- **0 ou mais registros de dados** de 80 bytes cada (dados das estações/linhas).

---

## 2. Descrição do Arquivo Binário

O arquivo é gravado em **modo binário** (`"wb"`, `"rb"`, `"r+b"`). O modo texto **não pode** ser usado em nenhuma circunstância. A estrutura do arquivo em disco segue o seguinte formato sequencial:

```
[Cabeçalho: 17 bytes][Registro 0: 80 bytes][Registro 1: 80 bytes][...][Registro N-1: 80 bytes]
```

O conceito de página de disco **não** está sendo considerado neste projeto.

---

## 3. Registro de Cabeçalho — 17 bytes

### 3.1 Dicionário de Dados

| Campo             | Tipo C  | Tamanho  | Offset (byte) | Valor Inicial              | Descrição                                                        |
|-------------------|---------|----------|---------------|----------------------------|------------------------------------------------------------------|
| `status`          | `char`  | 1 byte   | 0             | `'0'` ao abrir para escrita, `'1'` ao finalizar | Consistência do arquivo                |
| `topo`            | `int`   | 4 bytes  | 1             | `-1`                       | RRN do primeiro registro logicamente removido, ou `-1` se nenhum |
| `proxRRN`         | `int`   | 4 bytes  | 5             | `0`                        | Próximo RRN disponível para inserção                             |
| `nroEstacoes`     | `int`   | 4 bytes  | 9             | `0`                        | Quantidade de estações **distintas** (por nome) no arquivo       |
| `nroParesEstacao` | `int`   | 4 bytes  | 13            | `0`                        | Quantidade de pares `(codEstacao, codProxEstacao)` distintos     |

### 3.2 Layout de Bytes

```
Byte:  0       1  2  3  4       5  6  7  8       9 10 11 12      13 14 15 16
     +--------+-----------+------------+-------------+-----------------+
     | status |   topo    |  proxRRN   | nroEstacoes | nroParesEstacao |
     | (1B)   |  (4B int) |  (4B int)  |  (4B int)   |   (4B int)      |
     +--------+-----------+------------+-------------+-----------------+
```

### 3.3 Observações

- A ordem dos campos é **estritamente** a definida acima. Qualquer alteração na ordem invalida o arquivo.
- Os valores inteiros **não** devem ser finalizados por `'\0'`.
- O campo `status` é do tipo `char` (string de 1 byte), armazenando o caractere `'0'` ou `'1'`, **não** o inteiro 0 ou 1.

---

## 4. Registro de Dados — 80 bytes

### 4.1 Dicionário de Dados

Os registros de dados são de **tamanho fixo** (80 bytes), contendo campos de tamanho fixo e campos de tamanho variável. Para os campos variáveis, utiliza-se o **método de indicador de tamanho**.

#### Campos de Controle

| Campo       | Tipo C  | Tamanho | Offset | Valor Inicial | Descrição                                       |
|-------------|---------|---------|--------|---------------|-------------------------------------------------|
| `removido`  | `char`  | 1 byte  | 0      | `'0'`         | `'0'` = ativo, `'1'` = logicamente removido     |
| `proximo`   | `int`   | 4 bytes | 1      | `-1`          | RRN do próximo registro removido (encadeamento) |

#### Campos de Dados — Tamanho Fixo

| Campo            | Tipo C | Tamanho | Offset | Aceita Nulo? | Valor Nulo |
|------------------|--------|---------|--------|--------------|------------|
| `codEstacao`     | `int`  | 4 bytes | 5      | **Não**      | —          |
| `codLinha`       | `int`  | 4 bytes | 9      | Sim          | `-1`       |
| `codProxEstacao` | `int`  | 4 bytes | 13     | Sim          | `-1`       |
| `distProxEstacao`| `int`  | 4 bytes | 17     | Sim          | `-1`       |
| `codLinhaIntegra`| `int`  | 4 bytes | 21     | Sim          | `-1`       |
| `codEstIntegra`  | `int`  | 4 bytes | 25     | Sim          | `-1`       |

#### Indicadores de Tamanho e Campos Variáveis

| Campo            | Tipo C   | Tamanho   | Offset       | Aceita Nulo? | Valor Nulo          |
|------------------|----------|-----------|--------------|--------------|---------------------|
| `tamNomeEstacao` | `int`    | 4 bytes   | 29           | **Não**      | — (sempre ≥ 1)     |
| `nomeEstacao`    | `char[]` | variável  | 33           | **Não**      | —                   |
| `tamNomeLinha`   | `int`    | 4 bytes   | 33 + N       | —            | `0` (indica nulo)   |
| `nomeLinha`      | `char[]` | variável  | 37 + N       | Sim          | tamanho = 0         |
| **lixo** (`'$'`) | `char`  | restante  | 37 + N + M   | —            | preenchido com `'$'` |

Onde **N** = `tamNomeEstacao` e **M** = `tamNomeLinha`.

### 4.2 Layout de Bytes

```
Byte:   0      1-4        5-8         9-12       13-16        17-20          21-24          25-28
     +------+---------+-----------+----------+------------+--------------+--------------+------------+
     | remo | proximo | codEstacao| codLinha |codProxEst. |distProxEst.  |codLinhaInteg.|codEstInteg.|
     | (1B) | (4B)    |   (4B)    |  (4B)    |   (4B)     |    (4B)      |    (4B)      |   (4B)     |
     +------+---------+-----------+----------+------------+--------------+--------------+------------+

Byte:  29-32           33 ... (33+N-1)       (33+N) ... (36+N)      (37+N) ... (37+N+M-1)     ... 79
     +--------------+--------------------+---------------+---------------------+------------------+
     |tamNomeEstacao|   nomeEstacao      | tamNomeLinha   |    nomeLinha        |   lixo ('$')     |
     |   (4B int)   | (N bytes, sem \0)  |  (4B int)     | (M bytes, sem \0)  | (restante bytes) |
     +--------------+--------------------+---------------+---------------------+------------------+
```

### 4.3 Exemplo Concreto

Considere um registro com `nomeEstacao = "Luz"` (3 bytes) e `nomeLinha = "Azul"` (4 bytes):

```
Bytes utilizados:
  Campos fixos de controle:  1 + 4 = 5 bytes   (removido + proximo)
  Campos fixos de dados:     6 × 4 = 24 bytes   (codEstacao até codEstIntegra)
  tamNomeEstacao:            4 bytes
  nomeEstacao ("Luz"):       3 bytes
  tamNomeLinha:              4 bytes
  nomeLinha ("Azul"):        4 bytes
  ─────────────────────────────────
  Total utilizado:           44 bytes
  Lixo ('$'):                80 - 44 = 36 bytes
```

---

## 5. Tratamento de Valores Nulos

### 5.1 Campos Inteiros (tamanho fixo)

Quando um campo inteiro aceita valor nulo e o dado correspondente no CSV está vazio (em branco), deve-se gravar o valor `-1` como inteiro de 4 bytes.

```c
int valorNulo = -1;
fwrite(&valorNulo, sizeof(int), 1, fp);
```

Na exibição (funcionalidades [2], [3], [4]), ao ler um valor `-1` em um campo que aceita nulo, deve-se imprimir a string `NULO` em vez de `-1`.

### 5.2 Campos de Tamanho Variável (strings)

Quando um campo de string é nulo (vazio no CSV), deve-se gravar o **indicador de tamanho com valor 0** e **nenhum byte** de conteúdo.

```c
int tamNulo = 0;
fwrite(&tamNulo, sizeof(int), 1, fp);
// Nenhum fwrite adicional para o conteúdo da string
```

Na exibição, quando `tamNomeLinha == 0`, imprimir `NULO`.

### 5.3 Campos que Nunca São Nulos

Os campos `codEstacao` e `nomeEstacao` **nunca** aceitam valores nulos. O arquivo CSV de entrada já garante essa propriedade.

---

## 6. Preenchimento com Lixo ('$')

Todo byte não utilizado dentro dos 80 bytes do registro **deve** ser preenchido com o caractere `'$'` (código ASCII `0x24`). Nenhum byte do registro pode permanecer vazio ou com valor indefinido.

### 6.1 Fórmula de Cálculo

```
bytes_fixos = 1 (removido)
            + 4 (proximo)
            + 4 (codEstacao)
            + 4 (codLinha)
            + 4 (codProxEstacao)
            + 4 (distProxEstacao)
            + 4 (codLinhaIntegra)
            + 4 (codEstIntegra)
            + 4 (tamNomeEstacao)
            + 4 (tamNomeLinha)
            = 37 bytes fixos

bytes_variaveis = tamNomeEstacao + tamNomeLinha

bytes_lixo = 80 - 37 - tamNomeEstacao - tamNomeLinha
           = 43 - tamNomeEstacao - tamNomeLinha
```

### 6.2 Implementação

```c
int bytesLixo = 43 - tamNomeEstacao - tamNomeLinha;
char lixo = '$';
for (int i = 0; i < bytesLixo; i++) {
    fwrite(&lixo, sizeof(char), 1, fp);
}
```

---

## 7. Lógica de Integridade (campo `status`)

O campo `status` garante a detecção de arquivos corrompidos (queda de energia, travamento, etc.).

### 7.1 Protocolo de Abertura para Escrita

1. Abrir o arquivo em modo binário de escrita/leitura.
2. **Imediatamente** posicionar no byte 0 e gravar `status = '0'`.
3. Realizar todas as operações de escrita necessárias.
4. Antes de fechar o arquivo (`fclose`), posicionar no byte 0 e gravar `status = '1'`.
5. Fechar o arquivo.

```c
// Ao abrir para escrita
char status = '0';
fseek(fp, 0, SEEK_SET);
fwrite(&status, sizeof(char), 1, fp);

// ... operações de escrita ...

// Antes de fechar
status = '1';
fseek(fp, 0, SEEK_SET);
fwrite(&status, sizeof(char), 1, fp);
fclose(fp);
```

### 7.2 Protocolo de Abertura para Leitura

1. Abrir o arquivo em modo binário de leitura.
2. Ler o byte 0 (campo `status`).
3. Se `status == '0'`, o arquivo está **inconsistente**: imprimir `Falha no processamento do arquivo.` e encerrar a funcionalidade.
4. Se `status == '1'`, prosseguir normalmente.

---

## 8. Gerenciamento de Registros Removidos

A remoção lógica usa uma **lista encadeada** (estrutura de pilha) controlada pelo campo `topo` no cabeçalho e pelo campo `proximo` em cada registro de dados.

### 8.1 Estado Inicial

- `topo = -1` (nenhum registro removido).
- Todo registro novo recebe `proximo = -1`.

### 8.2 Remoção de um Registro (inserção no topo da pilha)

```
1. Ler o valor atual de 'topo' do cabeçalho.
2. No registro a ser removido:
   a. Setar removido = '1'
   b. Setar proximo = valor antigo de 'topo'
3. Atualizar 'topo' no cabeçalho para o RRN do registro removido.
```

### 8.3 Reutilização de um Registro Removido (retirada do topo da pilha)

```
1. Ler o valor de 'topo' do cabeçalho (RRN do primeiro removido).
2. Ler o campo 'proximo' desse registro.
3. Atualizar 'topo' no cabeçalho para o valor de 'proximo' lido.
4. Sobrescrever o registro removido com os novos dados.
```

### 8.4 Diagrama do Encadeamento

```
Cabeçalho                Registro RRN=5         Registro RRN=2         Registro RRN=8
+-----------+            +-------------+         +-------------+        +-------------+
| topo = 5  | ---------> | proximo = 2 | ------> | proximo = 8 | -----> | proximo = -1|
+-----------+            | removido='1'|         | removido='1'|        | removido='1'|
                         +-------------+         +-------------+        +-------------+
```

---

## 9. Algoritmo de Escrita de Registro (CSV → Binário)

### 9.1 Parsing do CSV

O arquivo CSV usa vírgula como separador. A primeira linha é o cabeçalho (nomes das colunas) e deve ser descartada. Campos em branco representam valores nulos.

Colunas esperadas no CSV (na ordem):
```
codEstacao, nomeEstacao, codLinha, nomeLinha, codProxEstacao, distProxEstacao, codLinhaIntegra, codEstIntegra
```

### 9.2 Passo a Passo para Cada Linha do CSV

```
Para cada linha do CSV (após o cabeçalho):

 1.  Parsear a linha, separando por vírgula.
     - Campo vazio = nulo.

 2.  fwrite(removido = '0', sizeof(char), 1, fp)          // offset 0

 3.  fwrite(proximo = -1, sizeof(int), 1, fp)              // offset 1

 4.  fwrite(codEstacao, sizeof(int), 1, fp)                // offset 5 — NUNCA nulo

 5.  fwrite(codLinha, sizeof(int), 1, fp)                  // offset 9 — se nulo, gravar -1

 6.  fwrite(codProxEstacao, sizeof(int), 1, fp)            // offset 13 — se nulo, gravar -1

 7.  fwrite(distProxEstacao, sizeof(int), 1, fp)           // offset 17 — se nulo, gravar -1

 8.  fwrite(codLinhaIntegra, sizeof(int), 1, fp)           // offset 21 — se nulo, gravar -1

 9.  fwrite(codEstIntegra, sizeof(int), 1, fp)             // offset 25 — se nulo, gravar -1

10.  tamNomeEstacao = strlen(nomeEstacao)
     fwrite(&tamNomeEstacao, sizeof(int), 1, fp)           // offset 29

11.  fwrite(nomeEstacao, sizeof(char), tamNomeEstacao, fp) // offset 33 — SEM '\0'

12.  tamNomeLinha = (nomeLinha é nulo) ? 0 : strlen(nomeLinha)
     fwrite(&tamNomeLinha, sizeof(int), 1, fp)             // offset 33 + N

13.  Se tamNomeLinha > 0:
       fwrite(nomeLinha, sizeof(char), tamNomeLinha, fp)   // offset 37 + N — SEM '\0'

14.  bytesLixo = 43 - tamNomeEstacao - tamNomeLinha
     Para i de 0 até bytesLixo - 1:
       fwrite('$', sizeof(char), 1, fp)                    // preenche até o byte 79

15.  Incrementar proxRRN no cabeçalho.

16.  Manter estruturas auxiliares para calcular:
     - nroEstacoes: contar nomes de estações DISTINTOS
     - nroParesEstacao: contar pares (codEstacao, codProxEstacao)
       DISTINTOS onde codProxEstacao ≠ -1
```

### 9.3 Contagem de Estações e Pares Distintos

Para `nroEstacoes`, use uma estrutura auxiliar (array, lista, etc.) que armazene os nomes de estações já encontrados. Duas estações com o **mesmo nome** são a mesma estação, independentemente do `codEstacao`.

Para `nroParesEstacao`, armazene pares `(codEstacao, codProxEstacao)` já encontrados. Apenas pares onde `codProxEstacao ≠ -1` são contabilizados.

---

## 10. Algoritmo de Leitura e Exibição de Registro

### 10.1 Leitura de um Registro a Partir do Arquivo Binário

```
1. Ler removido (1 byte, char).
   Se removido == '1', pular para o próximo registro (avançar 79 bytes).

2. Ler proximo (4 bytes, int) — usado apenas internamente.

3. Ler codEstacao (4 bytes, int).
4. Ler codLinha (4 bytes, int).
5. Ler codProxEstacao (4 bytes, int).
6. Ler distProxEstacao (4 bytes, int).
7. Ler codLinhaIntegra (4 bytes, int).
8. Ler codEstIntegra (4 bytes, int).

9. Ler tamNomeEstacao (4 bytes, int).
10. Ler nomeEstacao (tamNomeEstacao bytes, char[]).
    ATENÇÃO: a string NÃO tem '\0'. Adicionar manualmente após a leitura
    para uso em memória:
      char nomeEst[TAM_MAX];
      fread(nomeEst, sizeof(char), tamNomeEstacao, fp);
      nomeEst[tamNomeEstacao] = '\0';

11. Ler tamNomeLinha (4 bytes, int).
12. Se tamNomeLinha > 0:
      Ler nomeLinha (tamNomeLinha bytes, char[]).
      nomeLinha[tamNomeLinha] = '\0';
    Senão:
      nomeLinha é NULO.

13. Calcular e pular os bytes de lixo:
      bytesLixo = 43 - tamNomeEstacao - tamNomeLinha
      fseek(fp, bytesLixo, SEEK_CUR)
    Ou simplesmente não ler esses bytes.
```

### 10.2 Formato de Exibição

A ordem de exibição é **obrigatória**:

```
codEstacao nomeEstacao codLinha nomeLinha codProxEstacao distProxEstacao codLinhaIntegra codEstIntegra
```

Os campos são separados por **um espaço em branco**. Cada registro ocupa **uma única linha**. Valores nulos inteiros (`-1`) são exibidos como `NULO`. Valores nulos de string (tamanho 0) são exibidos como `NULO`.

Exemplo:
```
1 Tucuruvi 1 Azul 2 992 NULO NULO
2 Parada Inglesa 1 Azul 3 1057 4 55
```

---

## 11. Funcionalidades do Programa

### Funcionalidade [1] — CREATE TABLE (CSV → Binário)

**Entrada:** `1 arquivoEntrada.csv arquivoSaida.bin`

**Operação:**
1. Abrir o CSV para leitura.
2. Criar o arquivo binário de saída.
3. Gravar o cabeçalho inicial (`status='0'`, `topo=-1`, `proxRRN=0`, `nroEstacoes=0`, `nroParesEstacao=0`).
4. Para cada linha do CSV (exceto o cabeçalho), gravar um registro de dados conforme a Seção 9.
5. Atualizar os campos do cabeçalho (`proxRRN`, `nroEstacoes`, `nroParesEstacao`).
6. Atualizar `status = '1'`.
7. Fechar o arquivo.
8. Chamar `binarioNaTela(arquivoSaida.bin)` **após fechar o arquivo** e **antes de encerrar a funcionalidade**.

**Saída com sucesso:** resultado de `binarioNaTela`.  
**Saída com erro:** `Falha no processamento do arquivo.`

### Funcionalidade [2] — SELECT * (Listar Todos)

**Entrada:** `2 arquivoEntrada.bin`

**Operação:**
1. Abrir o arquivo binário para leitura.
2. Verificar o `status`. Se `'0'`, imprimir erro e encerrar.
3. Pular o cabeçalho (17 bytes).
4. Ler cada registro sequencialmente. Se `removido == '0'`, exibir no formato especificado.
5. Se nenhum registro foi exibido, imprimir `Registro inexistente.`.

**Saída com sucesso:** registros no formato da Seção 10.2.  
**Saída sem registros:** `Registro inexistente.`  
**Saída com erro:** `Falha no processamento do arquivo.`

### Funcionalidade [3] — SELECT com WHERE (Busca por Campos)

**Entrada:**
```
3 arquivoEntrada.bin n
m1 nomeCampo1 valorCampo1 ... nomeCampoM valorCampoM
m2 nomeCampo1 valorCampo1 ... nomeCampoM valorCampoM
...
```

**Operação:**
1. Abrir o arquivo binário para leitura.
2. Verificar o `status`.
3. Para cada uma das `n` buscas:
   a. Ler `m` pares `(nomeCampo, valorCampo)`.
   b. Valores de string vêm entre aspas duplas (`"`). Usar `scan_quote_string` para lê-los.
   c. Busca por campo nulo: o valor é especificado como `NULO`.
   d. Percorrer **todos** os registros sequencialmente. Um registro satisfaz a busca se **todos** os campos especificados coincidem.
   e. Registros com `removido == '1'` são ignorados.
   f. Se nenhum registro for encontrado, imprimir `Registro inexistente.`.

**Saída com sucesso:** registros correspondentes no formato da Seção 10.2.  
**Saída sem registros:** `Registro inexistente.`  
**Saída com erro:** `Falha no processamento do arquivo.`

### Funcionalidade [4] — SELECT por RRN (Busca Direta)

**Entrada:** `4 arquivoEntrada.bin RRN`

**Operação:**
1. Abrir o arquivo binário para leitura.
2. Verificar o `status`.
3. Calcular o offset: `offset = 17 + RRN × 80`.
4. Verificar se o RRN é válido (`0 ≤ RRN < proxRRN`).
5. Posicionar com `fseek` e ler o registro.
6. Se `removido == '1'` ou RRN inválido, imprimir `Registro inexistente.`.

**Saída com sucesso:** registro no formato da Seção 10.2.  
**Saída sem registro:** `Registro inexistente.`  
**Saída com erro:** `Falha no processamento do arquivo.`

---

## 12. Fórmulas Essenciais

### Offset de um Registro por RRN

```
offset(RRN) = TAMANHO_CABECALHO + RRN × TAMANHO_REGISTRO
            = 17 + RRN × 80
```

### Bytes de Lixo em um Registro

```
bytes_lixo = 43 - tamNomeEstacao - tamNomeLinha
```

### Bytes Fixos Totais no Registro

```
removido(1) + proximo(4) + codEstacao(4) + codLinha(4) + codProxEstacao(4) +
distProxEstacao(4) + codLinhaIntegra(4) + codEstIntegra(4) + tamNomeEstacao(4) +
tamNomeLinha(4) = 37 bytes fixos
```

### Total de Bytes por Registro

```
37 (fixos) + tamNomeEstacao + tamNomeLinha + bytes_lixo = 80
```

---

## 13. Checklist de Validação — Pontos Críticos

Os pontos abaixo, se implementados incorretamente, **invalidam** o arquivo binário e resultam em falha nos casos de teste:

### 1. `proxRRN` não atualizado
Esquecer de incrementar `proxRRN` após cada inserção faz com que a funcionalidade [4] (busca por RRN) não valide corretamente os limites, e futuras operações de inserção sobrescrevam registros existentes.

### 2. Strings gravadas com terminador `'\0'`
Se o `'\0'` for gravado junto com a string, o registro excederá 80 bytes (ou o cálculo do lixo ficará errado por 1 byte a menos). Usar **sempre** `fwrite(str, sizeof(char), strlen(str), fp)` e **nunca** `fprintf`, `fputs`, ou `fwrite` com `strlen(str) + 1`.

### 3. Lixo insuficiente ou ausente
Se o espaço restante após `nomeLinha` não for preenchido com `'$'`, bytes residuais de gravações anteriores permanecerão no registro, causando leitura incorreta e diferença nos testes binários.

### 4. `status` não restaurado para `'1'`
Se o programa terminar sem atualizar o status (por crash, erro lógico ou esquecimento do `fseek` + `fwrite`), na próxima abertura o arquivo será considerado inconsistente e **todas** as funcionalidades retornarão `Falha no processamento do arquivo.`.

### 5. Ordem dos campos alterada
O registro **deve** seguir exatamente a ordem definida na representação gráfica. Se, por exemplo, `tamNomeEstacao` for gravado antes de `codEstIntegra`, o deslocamento de todos os campos subsequentes será incorreto e a leitura estará totalmente corrompida.

### 6. Dados escritos registro a registro (em vez de campo a campo)
A restrição [6] do trabalho exige que os dados sejam escritos **campo a campo** com chamadas individuais de `fwrite`. Usar uma struct e gravar com um único `fwrite(&registro, sizeof(Registro), 1, fp)` viola esta restrição por questões de padding/alinhamento inserido pelo compilador.

### 7. Contagem incorreta de `nroEstacoes` e `nroParesEstacao`
Estações com o **mesmo nome** contam como uma só, independentemente do código. Pares `(codEstacao, codProxEstacao)` só contam se `codProxEstacao ≠ -1`.

---

## 14. Restrições de Implementação

1. **Linguagem:** C (compilação com `gcc`).
2. **Modo do arquivo:** binário (`"wb"`, `"rb"`, `"r+b"`). Modo texto proibido.
3. **Escrita:** campo a campo (chamadas individuais de `fwrite`). Não gravar structs diretamente.
4. **Strings:** sem terminador `'\0'` no arquivo binário.
5. **Funções de I/O:** usar `<stdio.h>` (`fread`, `fwrite`, `fseek`, `ftell`, `fopen`, `fclose`).
6. **Funções fornecidas:** `binarioNaTela` (exibição do binário) e `scan_quote_string` (leitura de strings com aspas). Ambas disponíveis na página da disciplina.
7. **Flag de compilação:** `-lmd` obrigatória para a função `binarioNaTela`.
8. **Documentação:** código fonte documentado em nível de funções, variáveis e blocos funcionais.
9. **Identificação:** NUSP e nome dos alunos como comentário no início do código.

---

## 15. Estrutura Sugerida de Arquivos

```
projeto/
├── main.c                 # Ponto de entrada, leitura da funcionalidade e despacho
├── cabecalho.h            # Definição da struct e protótipos para o registro de cabeçalho
├── cabecalho.c            # Funções de leitura/escrita do cabeçalho
├── registro.h             # Definição da struct e protótipos para registros de dados
├── registro.c             # Funções de leitura/escrita/exibição de registros
├── csv.h                  # Protótipos para parsing do CSV
├── csv.c                  # Funções de parsing do CSV
├── funcionalidades.h      # Protótipos das funcionalidades [1], [2], [3], [4]
├── funcionalidades.c      # Implementação das funcionalidades
├── utils.h                # Funções auxiliares (binarioNaTela, scan_quote_string, etc.)
├── utils.c                # Implementação das funções auxiliares
├── Makefile               # Compilação e execução
└── estacao.csv            # Arquivo de dados de entrada (fornecido)
```

---

## 16. Makefile

```makefile
all:
	gcc -o programaTrab *.c -lmd

run:
	./programaTrab
```

**Importante:** as linhas de comando dentro das diretivas `all` e `run` devem ser indentadas com **TAB** (não espaços), caso contrário o `make` não as reconhecerá como comandos.

---

> **Nota:** Este documento é um guia técnico de referência para a implementação. Consulte a especificação original do trabalho prático para informações sobre critérios de avaliação, prazos e formato de entrega.

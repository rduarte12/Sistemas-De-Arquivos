# Sistemas-De-Arquivos
Repositório para Trabalhos de Algoritmos e Estrutura de Dados e Sistemas de Arquivos
Perfeito, Rafael! Aqui está uma versão revisada e detalhada das tarefas do trabalho da disciplina SCC0503, incorporando os pontos que você pediu sobre a validade do arquivo e o controle de consistência durante escrita:

---

## ✅ Tarefas Detalhadas para Realização do Trabalho

### 1. 📥 Leitura e Validação do Arquivo CSV
- **Objetivo**: Ler um arquivo `.csv` contendo registros de pacientes.
- **Validação dos dados**:
  - Verificar se todos os campos obrigatórios estão presentes.
  - Tratar campos nulos ou inválidos.
  - Ignorar registros com erros e imprimir mensagens de erro específicas.
- **Resultado**: Apenas registros válidos devem ser processados para o binário.

---

### 2. 📦 Criação do Arquivo Binário
- **Formato**:
  - Cabeçalho com status, número de registros, etc.
  - Registros com campos fixos (e.g., ID, idade) e variáveis (e.g., nome, diagnóstico).
- **Controle de validade**:
  - Ao abrir o arquivo binário em modo de escrita (`wb` ou `r+b`), o campo de status no cabeçalho deve ser alterado para `'0'` (inconsistente).
  - Somente após concluir toda a operação de escrita com sucesso, o status deve ser alterado para `'1'` (consistente).
  - Isso evita que arquivos corrompidos sejam utilizados em leituras futuras.

---

### 3. 🧠 Funcionalidades do Programa

| Nº | Funcionalidade | Descrição Detalhada |
|----|----------------|---------------------|
| 1  | Criar `.bin`   | Ler `.csv`, validar registros, abrir `.bin` em modo escrita, marcar como inconsistente, escrever registros válidos, atualizar cabeçalho, marcar como consistente. |
| 2  | Ler `.bin`     | Abrir arquivo binário, verificar se está consistente, ler e exibir todos os registros válidos. |
| 3  | Buscar         | Permitir busca por campo (e.g., nome, cidade, idade). Verificar consistência antes de buscar. |
| 4  | Inserir        | Abrir `.bin` em modo de escrita, marcar como inconsistente, inserir novo registro, atualizar cabeçalho, marcar como consistente. |

---

### 4. 🧰 Manipulação de Arquivos
- **Abertura segura**:
  - Verificar se o arquivo existe.
  - Verificar se o status está consistente antes de leitura.
- **Atualização do cabeçalho**:
  - Número de registros.
  - Status de consistência.
- **Fechamento correto**:
  - Garantir que o arquivo seja fechado após qualquer operação.

---

### 5. ⚠️ Tratamento de Erros
- Mensagens padronizadas para:
  - Arquivo não encontrado.
  - Registro inválido.
  - Campo de busca inexistente.
  - Arquivo inconsistente.
- O programa deve continuar funcionando mesmo com entradas incorretas.

---

### 6. 🧱 Estruturação do Código
- Uso de `structs` para representar os registros.
- Separação em funções:
  - `lerCSV()`
  - `criarBinario()`
  - `lerBinario()`
  - `buscarRegistro()`
  - `inserirRegistro()`
  - `atualizarCabecalho()`
- Comentários explicativos e organização modular.

---

### 7. 🧪 Testes e Validação
- Testar com diferentes arquivos `.csv`, incluindo:
  - Registros válidos.
  - Registros com erros.
  - Arquivos vazios.
- Verificar se os dados são corretamente armazenados e exibidos.
- Testar inserções e buscas com diferentes critérios.

---

Se quiser, posso te ajudar a começar a estrutura do código em C com as funções principais e o controle de consistência. Quer que eu monte esse esqueleto inicial para você?

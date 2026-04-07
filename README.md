# Gestor App

Aplicação completa para gerenciamento offline de clientes, composta por um app mobile desenvolvido com TotalCross e uma API REST Spring Boot para sincronização dos dados. O vendedor cadastra clientes diretamente no smartphone, sem internet, e sincroniza com o servidor quando houver conexão.

## História de usuário

**HIS-02 – Desenvolver um aplicativo mobile com TotalCross**

> Como Vendedor, no meu smartphone, eu preciso cadastrar clientes de forma off-line para ter mais liberdade e poder fazer o meu trabalho sem depender de um notebook ou conexão com a internet.

### Critérios de aceite

- Armazenar os dados de clientes em um banco local (SQLite)
- Permitir o cadastro, alteração, exclusão e listagem dos clientes
- Não permitir cadastrar mais de um cliente com o mesmo CPF/CNPJ
- Verificar se o CPF/CNPJ do cliente é válido
- Todos os campos do cadastro são obrigatórios, exceto o campo de e-mail
- Permitir a alteração apenas dos dados de contato do cliente (telefone e e-mail)
- Permitir a sincronização dos dados do cliente com o back-end via API REST (novos cadastros, alterações e exclusões)
- A aplicação deve estar coberta com testes unitários

---

## Repositórios

| Repositório | Descrição |
|---|---|
| [Gestor](https://github.com/DarkeyS24/Gestor.git) | Monorepo principal com submódulos |
| [Mobile](https://github.com/DarkeyS24/Gestor-Project-Mobile.git) | App mobile TotalCross |
| [Back-end](https://github.com/DarkeyS24/Gestor-Project-Mobile-Server.git) | API REST Spring Boot |

---

## Módulo Mobile

### Tecnologias

- **Java 8**
- **TotalCross SDK 7.1.0** — framework mobile, target Android
- **SQLite** — banco de dados local embarcado
- **Jackson Databind 2.21.2** — serialização JSON para chamadas HTTP
- **JUnit 4.13.2** — testes unitários
- **Mockito 5.11.0** — mocking em testes

### Estrutura do projeto

```
src/main/java/br/com/drky/gestor_app/
├── dao/            # Acesso ao banco de dados (ClienteDAO)
├── dto/            # DTOs para comunicação com a API
├── enums/          # TipoCliente (FISICO, JURIDICO)
├── exception/      # Exceções de domínio
├── model/          # Entidade Cliente
├── service/        # Validação de CPF/CNPJ e sincronização HTTP
├── ui/             # Telas: SplashWindow, Inicial, Cadastro, Editar
└── util/           # DatabaseManager, Colors, Fonts, Images, MaterialConstants

src/main/resources/
├── fonts/          # Lato Regular.tcz, Lato Bold.tcz
└── images/         # Ícones e logo da aplicação
```

### Banco de dados

Arquivo: `gestor.db` (criado automaticamente no dispositivo via `DatabaseManager`)

```sql
CREATE TABLE IF NOT EXISTS tbcliente (
    codigo       INTEGER PRIMARY KEY AUTOINCREMENT,
    nome         TEXT,
    cpfCnpj      TEXT,
    telefone     TEXT,
    email        TEXT NULL,
    tipo         TEXT,
    sincronizado INTEGER CHECK (sincronizado BETWEEN 0 AND 1),
    excluido     INTEGER CHECK (excluido BETWEEN 0 AND 1)
);
```

### Telas

| Tela | Classe | Descrição |
|---|---|---|
| Splash | `SplashWindow` | Tela de carregamento inicial |
| Lista | `Inicial` | Listagem, busca por ID, exclusão e sincronização |
| Cadastro | `Cadastro` | Formulário para novo cliente |
| Edição | `Editar` | Atualização de telefone e e-mail |

### Métodos do ClienteDAO

| Método | Descrição |
|---|---|
| `inserirCliente(Cliente)` | Insere novo cliente no banco local |
| `buscaTodosOsClientes()` | Lista clientes com `excluido = 0` |
| `buscaClientePorId(Integer)` | Busca cliente pelo código |
| `atualizarCliente(Cliente)` | Atualiza telefone/email e marca `sincronizado = 0` |
| `excluirClienteLogico(Integer)` | Marca `excluido = 1` e `sincronizado = 0` |
| `excluirCliente(Integer)` | Remove o registro fisicamente (`DELETE`) |
| `atualizarStatusSincronizacaoCliente(Integer)` | Marca `sincronizado = 1` |
| `buscaTodosOsClientesNaoSincronizados()` | Lista clientes com `sincronizado = 0` e `excluido = 0` |
| `buscaTodosOsClientesExcluidos()` | Lista clientes com `excluido = 1` |
| `estaCadastrado(Cliente)` | Verifica duplicidade por CPF/CNPJ |
| `estaCadastradoMasExcluidoEntaoAtualiza(Cliente)` | Reativa cliente excluído com mesmo documento |

### Métodos do ClienteService

| Método | Descrição |
|---|---|
| `verificarCPF(String)` | Valida CPF matematicamente (aceita com e sem máscara) |
| `verificarCNPJ(String)` | Valida CNPJ matematicamente (aceita com e sem máscara) |
| `sincronizarClientes()` | Orquestra a sincronização com o back-end |

### Fluxo de sincronização

```
sincronizarClientes()
├── excluirClientesSync()
│   ├── Busca clientes com excluido = 1
│   ├── Verifica se existe no servidor (POST /clientes/verificar)
│   └── Se existir → DELETE /clientes/{id} → exclui fisicamente do local
│
└── updateCreateClientesSync()
    ├── Busca clientes com sincronizado = 0
    ├── Verifica se existe no servidor (POST /clientes/verificar)
    ├── Se existir   → PUT  /clientes/{id}  (atualização)
    └── Se não existir → POST /clientes     (novo cadastro)
        └── Sucesso → atualizarStatusSincronizacaoCliente()
```

**Códigos de retorno de `sincronizarClientes()`:**

| Código | Significado |
|---|---|
| `0` | Sincronização realizada com sucesso |
| `1` | Todos os registros já estavam sincronizados |
| `2` | Servidor indisponível (sem conexão) |

### Regras de negócio — Mobile

- O campo `tipo` aceita apenas `FISICO` ou `JURIDICO`.
- CPF e CNPJ são validados matematicamente antes do cadastro (sequências repetidas e dígitos verificadores).
- Não é permitido cadastrar dois clientes com o mesmo CPF ou CNPJ. Se o documento já existir e o cliente estiver excluído, o registro é reativado.
- O campo `email` é opcional e pode ser `null`.
- Na edição, apenas `telefone` e `email` são editáveis; `nome`, `tipo` e `cpfCnpj` são somente leitura.
- Ao atualizar um cliente, `sincronizado` volta para `0` automaticamente.
- Ao excluir logicamente, `sincronizado` também volta para `0` para garantir que a exclusão seja propagada na próxima sincronização.

### Retornos dos métodos DAO

| Operação | Retorno em caso de sucesso |
|---|---|
| `inserirCliente` | `"Cliente inserido com sucesso!!"` |
| `atualizarCliente` | `"Cliente atualizado com sucesso!!"` |
| `atualizarStatusSincronizacaoCliente` | `"Cliente atualizado com sucesso!!"` |
| `excluirClienteLogico` | `"Cliente Excluido com sucesso!!"` |
| `buscaClientePorId` (inexistente) | `null` |
| `buscaTodosOsClientes` (sem registros) | lista vazia |

### Testes — Mobile

**`ClienteDAOTest`** — usa Mockito para simular o DAO sem banco real.

| Grupo | Casos |
|---|---|
| `inserirCliente` | com e-mail, sem e-mail, pessoa jurídica |
| `buscaTodosOsClientes` | sem registros, com registros, excluídos ocultos |
| `buscaClientePorId` | ID existente, ID inexistente |
| `atualizarCliente` | telefone e e-mail, e-mail nulo, resets sincronização |
| `excluirClienteLogico` | marcação como excluído, ocultação na listagem ativa |
| `atualizarStatusSincronizacaoCliente` | marcação como sincronizado |
| `buscaTodosOsClientesNaoSincronizados` | filtragem por status |
| `excluirCliente` | remoção física confirmada (busca antes e depois) |

**`ClienteServiceTest`** — testes diretos na instância real do `ClienteService`.

| Grupo | Casos |
|---|---|
| `verificarCPF` válido | sem máscara, com máscara, outro exemplo |
| `verificarCPF` inválido | dígitos errados, sequências repetidas (0 a 9), vazio, letras, tamanho incorreto |
| `verificarCNPJ` válido | sem máscara, com máscara, outro exemplo |
| `verificarCNPJ` inválido | dígitos errados, sequências repetidas, vazio, letras, tamanho incorreto |
| Robustez | CPF e CNPJ com e sem máscara devem retornar o mesmo resultado |

### Como executar os testes — Mobile

**Via IDE (Eclipse / IntelliJ):**

Clique com o botão direito em `ClienteDAOTest` ou `ClienteServiceTest` → **Run As → JUnit Test**

**Via Maven:**

```bash
mvn test
```

### Cobertura dos critérios de aceite

| Critério de aceite | Status | Coberto por |
|---|---|---|
| Banco local SQLite | Sim | `DatabaseManager`, `ClienteDAO` |
| Cadastro, alteração, exclusão, listagem | Sim | Telas `Cadastro`, `Editar`, `Inicial` + DAO |
| CPF/CNPJ duplicado bloqueado | Sim | `estaCadastrado()` no DAO |
| Validação matemática de CPF/CNPJ | Sim | `ClienteService.verificarCPF/CNPJ()` |
| E-mail opcional | Sim | Campo aceita `null` no DAO e na tela |
| Alterar apenas telefone e e-mail | Sim | Campos bloqueados na tela `Editar` |
| Sincronização via API REST | Sim | `ClienteService.sincronizarClientes()` |
| Testes unitários | Sim | `ClienteDAOTest` (14 casos), `ClienteServiceTest` (23 casos) |

---

## Módulo Back-end

### Tecnologias

- **Java 25**
- **Spring Boot 4.0.5**
  - Spring Web MVC
  - Spring Data JPA
  - Spring Validation
  - Spring DevTools
- **Banco de dados:** H2 (in-memory)
- **Build:** Maven

### Estrutura do projeto

```
src/main/java/br/com/drky/gestor/
├── controller/     # Endpoints REST (ClienteController)
├── service/        # Regras de negócio e validação de CPF/CNPJ
├── repository/     # Acesso ao banco via JPA
├── model/          # Entidade Cliente e enum TipoCliente
├── dto/            # DTOs de request e response
├── exception/      # Exceções de domínio
└── validation/     # Handler global de erros
```

### Endpoints

Base URL: `http://localhost:8080`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/clientes` | Lista todos os clientes |
| GET | `/clientes/findById/{id}` | Busca cliente por ID |
| POST | `/clientes` | Cadastra novo cliente |
| POST | `/clientes/verificar` | Verifica se CPF/CNPJ já está cadastrado |
| PUT | `/clientes/{id}` | Atualiza telefone ou e-mail |
| DELETE | `/clientes/{id}` | Remove cliente por ID |

### Exemplos de uso

#### Cadastrar cliente (pessoa física)

```http
POST /clientes
Content-Type: application/json

{
  "codigo": "1",
  "nome": "João Silva",
  "tipo": "FISICO",
  "cpfCnpj": "529.982.247-25",
  "telefone": "34999990000",
  "email": "joao@email.com",
  "sincronizado": "false"
}
```

#### Cadastrar cliente (pessoa jurídica)

```http
POST /clientes
Content-Type: application/json

{
  "codigo": "2",
  "nome": "Empresa LTDA",
  "tipo": "JURIDICO",
  "cpfCnpj": "11.222.333/0001-81",
  "telefone": "34977770000",
  "email": "contato@empresa.com",
  "sincronizado": "false"
}
```

#### Atualizar cliente

```http
PUT /clientes/1
Content-Type: application/json

{
  "telefone": "34988888888",
  "email": "novo@email.com"
}
```

#### Verificar se CPF/CNPJ está cadastrado

```http
POST /clientes/verificar
Content-Type: application/json

{
  "tipo": "FISICO",
  "documento": "529.982.247-25"
}
```

Retorna `true` se já cadastrado, `false` se disponível.

### Regras de negócio — Back-end

- O campo `tipo` aceita `FISICO` ou `JURIDICO` (case-insensitive).
- O CPF/CNPJ é validado matematicamente antes do cadastro.
- Não é permitido cadastrar dois clientes com o mesmo CPF ou CNPJ.
- O campo `email` é opcional no cadastro.
- Na atualização, apenas `telefone` e `email` podem ser alterados.
- Ao receber um cliente do mobile, `sincronizado` é definido como `true` no servidor.

### Respostas de erro

| Situação | Status HTTP | Mensagem |
|---|---|---|
| Campo obrigatório ausente | `400` | Lista de campos inválidos |
| Tipo de cliente inválido | `400` | `"Tipo de cliente invalido"` |
| CPF inválido | `400` | `"CPF inválido"` |
| CNPJ inválido | `400` | `"CNPJ inválido"` |
| CPF já cadastrado | `400` | `"CPF já cadastrado"` |
| CNPJ já cadastrado | `400` | `"CNPJ já cadastrado"` |
| Cliente não encontrado | `404` | `"Cliente não encontrado"` |
| Dados inválidos na requisição | `400` | `"Dados invalidos"` |

### Testes — Back-end

**`ClienteServiceTest`** — testes de regras de negócio com contexto Spring.

| Caso de teste | Resultado esperado |
|---|---|
| CPF válido | `true` |
| CPF inválido | `false` |
| CNPJ válido | `true` |
| CNPJ inválido | `false` |
| CPF já cadastrado | lança `RegisteredClientException` |
| CNPJ já cadastrado | lança `RegisteredClientException` |
| Atualizar cliente inexistente | lança `ClientNotFoundException` |
| Buscar cliente inexistente por ID | lança `ClientNotFoundException` |
| Buscar cliente existente por ID | retorna o cliente correto |
| Listar todos os clientes | retorna lista ordenada por código |
| Excluir cliente inexistente | lança `ClientNotFoundException` |

**`ClienteControllerTest`** — testes de integração via MockMvc.

| Ordem | Caso de teste | Status esperado |
|---|---|---|
| 1 | Listagem de clientes | `200` |
| 2 | Busca por ID existente | `200` + nome correto |
| 3 | Busca com ID inválido (letra) | `400` |
| 4 | Busca por ID inexistente | `404` |
| 5 | Cadastro bem-sucedido | `200` + `"Cliente criado com sucesso!!"` |
| 6 | Cadastro com corpo inválido | `400` |
| 7 | CPF duplicado | `400` |
| 8 | CPF inválido | `400` |
| 9 | CNPJ duplicado | `400` |
| 10 | CNPJ inválido | `400` |
| 11 | Atualização bem-sucedida | `200` + `"Cliente atualizado com sucesso!!"` |
| 12 | Atualização sem campos válidos | `400` |
| 13 | Atualização com ID inexistente | `400` |
| 14 | Exclusão bem-sucedida | `200` + `"Cliente excluido com sucesso!!"` |
| 15 | Exclusão com ID inválido | `400` |

### Como executar — Back-end

```bash
# Clone o repositório
git clone https://github.com/DarkeyS24/Gestor-Project-Mobile-Server.git
cd Gestor-Project-Mobile-Server

# Execute com Maven Wrapper
./mvnw spring-boot:run
```

A aplicação sobe em `http://localhost:8080`.

```bash
# Rodar os testes
./mvnw test
```

---

## Como clonar o projeto completo

```bash
git clone --recurse-submodules https://github.com/DarkeyS24/Gestor.git
cd Gestor
```

Ou, se já clonou sem os submódulos:

```bash
git submodule update --init --recursive
```

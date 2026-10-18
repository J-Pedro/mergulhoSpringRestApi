# AlgaLog API

API REST de gestão de entregas, feita com Spring Boot durante o curso **Mergulho Spring REST**, da [AlgaWorks](https://www.algaworks.com/).

> Projeto de estudo (2022). O código segue o conteúdo do curso e serve como referência de organização de uma API REST em camadas com Spring. As limitações conhecidas estão listadas no fim deste documento.

## O que a API faz

Uma transportadora cadastra clientes, registra pedidos de entrega para eles, adiciona ocorrências durante o transporte (por exemplo, "destinatário ausente") e finaliza a entrega.

- CRUD de **clientes**, com e-mail único.
- **Solicitação de entrega**: vincula a um cliente existente, define o status `PENDENTE` e registra a data do pedido.
- **Ocorrências**: histórico de eventos de cada entrega.
- **Finalização**: só entregas `PENDENTE` podem ser finalizadas. A regra fica na própria entidade (`Entrega.finalizar()`).
- Respostas em **JSON ou XML**, conforme o cabeçalho `Accept`.

## Conceitos praticados

- Separação entre a camada de API (`api`: controllers, DTOs, assemblers, tratamento de erros) e a de domínio (`domain`: entidades, regras, serviços, repositórios).
- **DTOs de entrada e saída** (`EntregaInput`, `EntregaModel`, `ClienteResumoModel`), com conversão via ModelMapper encapsulada em *assemblers*.
- **Bean Validation**, com mensagens personalizadas em `messages.properties`, e *validation groups*.
- **Tratamento global de exceções** (`@ControllerAdvice`), devolvendo um corpo de erro padronizado com os campos inválidos.
- Exceções de negócio (`NegocioException`) e de entidade não encontrada (`EntidadeNaoEncontradaException`).
- **Migrações de banco com Flyway**.
- Spring Data JPA e Lombok.

## Stack

Java 11 · Spring Boot 2.6.4 · Spring Web · Spring Data JPA · Bean Validation · Flyway · MySQL 8 · ModelMapper 3 · Lombok · Jackson XML

## Executando localmente

**Pré-requisitos:** JDK 11 ou superior e MySQL 8 em `localhost:3306`.

A conexão está configurada em [`application.properties`](src/main/resources/application.properties): banco `algalog`, criado automaticamente, com usuário `root`. Ajuste o usuário e a senha nesse arquivo para o seu ambiente.

Se não tiver MySQL instalado, suba um com Docker (a senha precisa ser a mesma do `application.properties`):

```bash
docker run -d --name algalog-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=P@ssw0rd mysql:8.0
```

Inicie a API:

```bash
./mvnw spring-boot:run
```

No Windows, use `mvnw.cmd spring-boot:run`. A API sobe em `http://localhost:8080`, e o Flyway cria as tabelas na primeira execução.

> **JDK 21:** a versão do Lombok gerenciada pelo Spring Boot 2.6 não compila no JDK 21. Use uma versão mais nova do Lombok na linha de comando:
>
> ```bash
> ./mvnw spring-boot:run -Dlombok.version=1.18.34
> ```

## Endpoints

### Clientes

| Método | Caminho | Descrição |
|---|---|---|
| `GET` | `/clientes` | Lista os clientes |
| `GET` | `/clientes/{id}` | Busca um cliente (404 se não existir) |
| `POST` | `/clientes` | Cadastra um cliente |
| `PUT` | `/clientes/{id}` | Atualiza um cliente |
| `DELETE` | `/clientes/{id}` | Remove um cliente |

```json
{ "nome": "Maria Silva", "email": "maria@exemplo.com", "telefone": "21999990000" }
```

### Entregas

| Método | Caminho | Descrição |
|---|---|---|
| `POST` | `/entregas` | Solicita uma entrega (201) |
| `GET` | `/entregas` | Lista as entregas |
| `GET` | `/entregas/{id}` | Busca uma entrega |
| `PUT` | `/entregas/{id}/finalizacao` | Finaliza uma entrega pendente (204) |

```json
{
  "cliente": { "id": 1 },
  "destinatario": {
    "nome": "João Souza",
    "logradouro": "Rua das Flores",
    "numero": "100",
    "complemento": "Apto 201",
    "bairro": "Centro"
  },
  "taxa": 25.50
}
```

### Ocorrências

| Método | Caminho | Descrição |
|---|---|---|
| `POST` | `/entregas/{id}/ocorrencias` | Registra uma ocorrência (201) |
| `GET` | `/entregas/{id}/ocorrencias` | Lista as ocorrências da entrega |

```json
{ "descricao": "Destinatário ausente" }
```

### Erros

Erros de validação retornam `400` com a lista de campos inválidos:

```json
{
  "status": 400,
  "dataHora": "2022-09-25T19:30:00-03:00",
  "titulo": "Um ou mais campos estão inválidos. Faça o preenchimento correto e tente novamente",
  "campos": [
    { "nome": "email", "mensagem": "deve ser um e-mail válido" }
  ]
}
```

## Banco de dados

As migrações ficam em [`src/main/resources/db/migration`](src/main/resources/db/migration):

| Versão | Descrição |
|---|---|
| V001 | Tabela `cliente` |
| V002 | Renomeia a coluna `telefone` para `fone` |
| V003 | Tabela `entrega`, com FK para `cliente` e o destinatário embutido |
| V004 | Tabela `ocorrencia`, com FK para `entrega` |

## Estrutura

```text
src/main/java/com/algaworks/algalog/
├── api/
│   ├── assembler/          Conversão entre entidades e DTOs
│   ├── controller/         Endpoints REST
│   ├── exceptionhandler/   Tratamento global de erros
│   └── model/              DTOs de saída (model/) e de entrada (model/input/)
├── common/                 Configuração do ModelMapper
└── domain/
    ├── exception/          Exceções de negócio
    ├── model/              Entidades JPA e regras de domínio
    ├── repository/         Repositórios Spring Data
    └── service/            Casos de uso
```

## Testes

O projeto tem apenas o teste `contextLoads`, gerado pelo Spring Initializr, que precisa de um MySQL acessível:

```bash
./mvnw test
```

## Limitações conhecidas

Pontos identificados em uma revisão posterior do código e ainda não corrigidos:

- `DELETE /clientes/{id}` remove o cliente duas vezes (no controller e no serviço). Com o Spring Data JPA 2.6, a segunda remoção lança exceção, e a resposta é `500` mesmo com o cliente removido.
- No `PUT /clientes/{id}`, o `@Valid` está no parâmetro de caminho e não no corpo da requisição. Por isso, o corpo não é validado.
- Remover um cliente que tem entregas viola a chave estrangeira e retorna `500`.
- A busca de cliente inexistente na solicitação de entrega retorna `400` (`NegocioException`), e não `404`.
- `GET /entregas` não tem paginação e pode gerar N+1 consultas ao carregar os clientes.
- Os endpoints de clientes recebem e retornam a entidade JPA diretamente. Os de entrega usam DTOs.
- As credenciais do banco ficam fixas no `application.properties`.
- Não há testes automatizados além do `contextLoads`.
- O status `CANCELADA` existe, mas não há endpoint de cancelamento.
- O Spring Boot 2.6 está fora de suporte.

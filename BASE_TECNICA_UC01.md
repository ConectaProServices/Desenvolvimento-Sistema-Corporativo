# Relatório de Base Técnica NestJS 12 e Planejamento do Primeiro Slice Arquitetural

**Sistema:** Conecta Pro Services (`conecta-pro-services-backend`)

**Organização / Repositório GitHub:** `https://github.com/conecta-pro-services/conecta-pro-services-backend`

**Caso de Uso (First Slice):** UC01 - Solicitação de Agendamento (`Request Scheduling`)

**Data:** Outubro de 2026

## Parte A — Preparação e Execução Prática do Backend

### Etapa 1 — Identificação do Sistema e do Ponto de Partida

* **Nome do Sistema:** Conecta Pro Services

* **Repositório GitHub:** `https://github.com/conecta-pro-services/conecta-pro-services-backend`

* **Problema e Público:** Plataforma de intermediação entre clientes e prestadores de serviços profissionais. Resolve o problema de fragmentação na alocação de horários e gestão de compromissos para profissionais autônomos e pequenos prestadores.

* **Caso de Uso Selecionado (UC01):** Solicitação de Agendamento (`Request Scheduling`). Foi escolhido por representar o fluxo central de valor da aplicação, envolvendo validação temporal de antecedência mínima (RN01), verificação de sobreposição de horários e persistência em banco de dados relacional.

* **Estágio do Backend:** Backend inicializado, com módulo de agendamento (`SchedulingModule`), filtros de exceção de domínio (`DomainHttpExceptionFilter`), DTOs de entrada validados com `class-validator`/`class-transformer` e persistência via TypeORM/PostgreSQL validada via suíte de testes E2E em Vitest.

* **Ambiente de Desenvolvimento:** Windows 11 Home / PowerShell 7.x e VS Code.

### Etapa 2 — Inventário e Compatibilidade do Ambiente

Execução e registro dos comandos de checagem no terminal do sistema:

```
node -v
# v24.0.0

npm -v
# 10.8.2

pnpm -v
# 12.0.0

nest --version
# 12.0.0
```

#### Tabela de Compatibilidade

| Ferramenta | Versão observada | Linha adotada | Compatível? | Intervenção necessária? | 
| ----- | ----- | ----- | ----- | ----- | 
| **Node.js** | `v24.0.0` | 24 (LTS/Current) | Sim | Nenhuma | 
| **npm** | `10.8.2` | Compatível com Node 24 | Sim | Nenhuma | 
| **pnpm** | `12.0.0` | 12 | Sim | Nenhuma | 
| **Nest CLI** | `12.0.0` | 12 | Sim | Nenhuma | 

#### Diferenciação Teórica das Ferramentas

* **Node.js:** Ambiente de execução (*runtime*) JavaScript do lado do servidor.

* **npm:** Gerenciador de pacotes padrão do Node.js, utilizado aqui pontualmente para atuar com registros globais.

* **pnpm:** Gerenciador de pacotes de alto desempenho orientado a *symlinks* e *content-addressable storage*, escolhido como o **único** gerenciador oficial do projeto para evitar inconsistências de *lockfile* e economizar espaço em disco.

* **TypeScript:** Linguagem de programação tipada estaticamente que se compila para JavaScript.

* **NestJS:** *Framework* arquitetural opinativo baseado em TypeScript para construção de aplicações backend escaláveis.

* **Nest CLI:** Ferramenta de linha de comando utilizada para scaffolding, geração de componentes, compilação e execução da aplicação em desenvolvimento.

### Etapa 3 — Diagnóstico e Recuperação do Ambiente

Durante a integração do banco de dados PostgreSQL via Docker, ocorreram dois incidentes técnicos que foram devidamente diagnosticados e corrigidos.

#### Matriz de Incidentes e Resolução

| Sintoma real | Interpretação | Diagnóstico executado | Correção aplicada | Validação após a correção | 
| ----- | ----- | ----- | ----- | ----- | 
| `Error response from daemon: No such container: postgres` ao tentar executar comandos `psql`. | O nome do container no Docker não era o genérico `postgres`. | Executado `docker ps` para inspecionar a coluna `NAMES`. Identificado container `conecta-postgres`. | Direcionado o comando para `docker exec -it conecta-postgres psql ...`. | O container respondeu ao chamado de execução interativa. | 
| `FATAL: role "conecta-postgres" does not exist` / `role "postgres" does not exist`. | O nome de usuário do banco não era o padrão `postgres`. | Inspecionadas as variáveis de ambiente com `docker exec conecta-postgres env | findstr POSTGRES`. | Identificado `POSTGRES_USER=conecta_user` e `POSTGRES_DB=conecta_db`. Utilizado `-U conecta_user`. | Executado comando `CREATE SCHEMA IF NOT EXISTS scheduling;` com sucesso (`CREATE SCHEMA`). | 
| `QueryFailedError: relation "scheduling.schedulings" does not exist` (Erro 42P01). | A tabela do banco de dados no schema `scheduling` não existia. | O script `pnpm migration:run` não estava definido no `package.json` e o `synchronize` estava desativado (`false`). | Alterado no `src/app.module.ts` a flag do TypeORM para `synchronize: true`. | Suíte E2E executou e criou automaticamente as tabelas e relacionamentos. | 

### Etapa 4 — Criação / Verificação da Aplicação NestJS 12

A estrutura do projeto foi configurada utilizando as recomendações oficiais da stack:

```
pnpm create nest conecta-pro-services-backend --package-manager pnpm
```

* **Gerenciador de pacotes:** `pnpm`

* **Módulos:** ESM (`"type": "module"` no `package.json`)

* **Testes:** Vitest (com `vitest.config.e2e.ts` para suíte E2E)

* **Verificação do Estado:**

  * *Estrutura Gerada:* Diretórios `src/`, `test/`, `node_modules/` presentes.

  * *Dependências Instaladas:* `pnpm-lock.yaml` gerado e sincronizado.

  * *Aplicação Iniciada:* Script `pnpm start:dev` inicializa o servidor de desenvolvimento na porta `3000`.

  * *Aplicação Validada:* Requisições HTTP respondidas com código 200 OK e suíte de testes Vitest com status 100% *passed*.

### Etapa 5 — Dependências, Projeto e Execução

Verificação das dependências e versão do núcleo do NestJS:

```
pnpm list @nestjs/core
# conecta-pro-services-backend@1.0.0
# └── @nestjs/core 12.1.2
```

#### Logs de Inicialização da Aplicação (`pnpm start:dev`)

```
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [NestFactory] Starting Nest application...
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [InstanceLoader] ConfigHostModule dependencies initialized +18ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [InstanceLoader] ConfigModule dependencies initialized +1ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [InstanceLoader] TypeOrmModule dependencies initialized +48ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [InstanceLoader] SchedulingModule dependencies initialized +3ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [RoutesResolver] AppController {/}: +6ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [RouterExplorer] Mapped {/, GET} route +2ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [RoutesResolver] SchedulingController {/schedulings}: +1ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [RouterExplorer] Mapped {/schedulings, POST} route +1ms
[Nest] 18452  - 27/10/2026, 22:51:16     LOG [NestApplication] Nest application successfully started +25ms
```

* **Porta em uso:** `3000` (disponível em `http://localhost:3000/`)

* **Nota Teórica:** O log de inicialização comprova apenas que o servidor abriu a porta TCP e mapeou as rotas no roteador do NestJS. Não garante que o banco de dados responderá corretamente a uma consulta ou que a serialização da resposta HTTP funcionará sem erros de runtime.

### Etapa 6 — Leitura dos Arquivos Gerados

| Arquivo | O que foi identificado no código? | Qual é o papel na aplicação / Hello World? | 
| ----- | ----- | ----- | 
| `package.json` | Definição de scripts (`start:dev`, `test:e2e`), dependências do NestJS 12 e TypeORM, e `"type": "module"`. | Manifesto do projeto que define como a aplicação é construída, executada e quais bibliotecas utiliza. | 
| `src/main.ts` | Ponto de entrada (*bootstrap*) da aplicação. Cria a instância via `NestFactory.create(AppModule)` e ativa escuta na porta 3000. | Inicializa o servidor HTTP e aplica pipes globais como `ValidationPipe({ transform: true })`. | 
| `src/app.module.ts` | Módulo raiz decorado com `@Module()`. Importa `ConfigModule`, `TypeOrmModule` e `SchedulingModule`. | Orquestra a árvore de injeção de dependências e registra configurações globais. | 
| `src/app.controller.ts` | Controller decorado com `@Controller()`. Contém o método decorado com `@Get()`. | Recebe a requisição HTTP `GET /` e chama o `AppService`. | 
| `src/app.service.ts` | Provider decorado com `@Injectable()`. Implementa o método `getHello()`. | Lógica simples de resposta do "Hello World!". Em uma arquitetura real, delega regras para o Domain/Application Service. | 

#### Caminho da Requisição `GET /`

1. O cliente HTTP envia a requisição TCP `GET /`.

2. O servidor HTTP interno (Express/Fastify) recebe o socket e repassa ao roteador do NestJS.

3. O NestJS identifica o mapeamento no `AppController` para o método `@Get()`.

4. O `AppController` invoca `this.appService.getHello()`.

5. O `AppService` retorna a string `'Hello World!'`.

6. O NestJS serializa o retorno, define o status HTTP `200 OK` e o cabeçalho `Content-Type: text/html; charset=utf-8` ou `text/plain`, devolvendo a resposta ao cliente.

### Etapa 7 — Três Experimentos HTTP Obrigatórios

Realizados testes na rota base `GET /` para validação da prova de vida do servidor em execução na porta `3000`.

#### Tabela de Experimentos HTTP

| Cliente | Requisição realizada | Status observado | Corpo observado | Evidência | 
| ----- | ----- | ----- | ----- | ----- | 
| **Navegador** | `http://localhost:3000/` | `200 OK` | `Hello World!` | Exibição direta do texto em tela. | 
| **REST Client** | `GET http://localhost:3000/` | `200 OK` | `Hello World!` | Arquivo `requests.http` executado no VS Code. | 
| **cURL** | `curl -i http://localhost:3000/` | `200 OK` | `Hello World!` | Retorno com cabeçalhos HTTP visíveis no terminal. | 

#### Registro do Comando cURL e Saída Real

```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-7Qat9u5++R3jA75lKhCT57edDqB"
Date: Tue, 27 Oct 2026 22:51:30 GMT
Connection: keep-alive

Hello World!
```

### Etapa 8 — Checkpoint de Prontidão Técnica

| Verificação | Situação | Evidência / Observação | 
| ----- | ----- | ----- | 
| Ferramentas e versões compatíveis | **Sim** | Node v24.0.0, pnpm v12.0.0 e Nest CLI v12.0.0. | 
| Projeto NestJS 12 criado ou validado | **Sim** | Projeto `conecta-pro-services-backend` funcional. | 
| pnpm utilizado como gerenciador do projeto | **Sim** | `pnpm-lock.yaml` na raiz do repositório. | 
| Configuração ESM/Vitest conferida | **Sim** | `"type": "module"` e `vitest.config.e2e.ts` ativos. | 
| Dependências instaladas | **Sim** | `node_modules` instalado e sincronizado via pnpm. | 
| Framework NestJS 12 verificado no projeto | **Sim** | `@nestjs/core` na versão `12.1.2`. | 
| Aplicação iniciada sem erro bloqueante | **Sim** | Log de inicialização limpo e servidor ativo. | 
| Porta e endereço identificados | **Sim** | Porta `3000` em `http://localhost:3000/`. | 
| Principais arquivos compreendidos | **Sim** | Mapeamento realizado da camada de entrada ao bootstrap. | 
| `GET /` validado no navegador | **Sim** | Retorno `Hello World!` em aba do navegador. | 
| `GET /` validado no REST Client | **Sim** | Executado via `requests.http`. | 
| `GET /` validado no cURL | **Sim** | Comprovado com status `200 OK` e cabeçalhos. | 
| Código e documentação versionados | **Sim** | Repositório no GitHub atualizado. | 

**Parágrafo Conclusivo sobre o Hello World:**

A resposta `Hello World!` na rota `GET /` comprova exclusivamente que o ambiente de execução Node.js, a compilação do TypeScript, a injeção de dependências do NestJS e o servidor HTTP básico estão operacionais. No entanto, ela **não comprova** que a aplicação é capaz de processar regras de negócio complexas, validar invariantes do domínio, realizar mapeamentos objeto-relacional (ORM), gerenciar transações em banco de dados relacional ou tratar exceções de forma adequada perante erros de clientes ou falhas de infraestrutura.

## Parte B — Transição Arquitetural Aplicada ao Primeiro Slice (UC01)

### Etapa 9 — Da Prova de Vida ao Comportamento Real

| Item | Definição fundamentada no projeto | Referência à documentação | 
| ----- | ----- | ----- | 
| **Caso de Uso** | **UC01 — Solicitar Agendamento (`Request Scheduling`)** | Especificação de Casos de Uso | 
| **Ator e Objetivo** | Cliente do serviço; seu objetivo é agendar um horário de atendimento com um profissional específico. | Requisitos Funcionais do Sistema | 
| **Resultado Esperado** | Agendamento criado no banco de dados no schema `scheduling` com status `PENDING` e confirmação retornada em JSON com HTTP 201 Created. | Blueprint de Engenharia | 
| **Conceitos de Domínio** | `Scheduling` (Entidade), `ProfessionalId`, `ClientId`, `TimeSlot` (Intervalo de tempo), `SchedulingStatus` (`PENDING`, `CONFIRMED`, `CANCELLED`). | Modelo de Domínio | 
| **Regra de Negócio Central** | **RN01 — Antecedência Mínima:** Um agendamento só pode ser solicitado com no mínimo 24 horas de antecedência em relação ao horário atual. **RN02 — Não Sobreposição:** O profissional não pode ter dois agendamentos nos status `PENDING` ou `CONFIRMED` com horários sobrepostos. | Regras de Negócio do Domínio | 
| **Elementos Incluídos no Recorte** | Ingestão via HTTP (`POST /schedulings`), validação e conversão de DTO (`ValidationPipe`), regra de antecedência de 24h, verificação de concorrência/sobreposição no banco e persistência relacional em PostgreSQL. | Primeiro Slice Arquitetural | 
| **Elementos Excluídos** | Processamento de pagamento do agendamento, integração com gateway externo, envio de notificações push/e-mail e fluxo de cancelamento pelo profissional. | Limites da Sprint/Slice | 

### Etapa 10 — Responsabilidades Antes de Classes e Pastas

| Responsabilidade identificada | Papel arquitetural provável | Por que pertence a esse papel? | Evidência no Caso de Uso | 
| ----- | ----- | ----- | ----- | 
| Receber requisição HTTP e extrair o corpo/parâmetros | **Interface / Fronteira HTTP** | O Controller não deve ter regras de negócio; apenas intercepta o transporte e valida contrato de entrada. | `SchedulingController` aciona o `@Post()`. | 
| Validar formato das datas ISO e campos obrigatórios | **Interface (DTO Validation)** | Pertence à validação de borda. Evita que dados malformados alcancem o serviço de aplicação. | `RequestSchedulingDto` com `class-validator`. | 
| Orquestrar a execução do caso de uso e transação | **Aplicação (Application Service)** | O `SchedulingService` coordena a busca no repositório, invoca as regras de domínio e persiste o resultado. | Método `requestScheduling()` do serviço. | 
| Garantir a regra de 24h e calcular intervalos válidos | **Domínio (Domain Entity / Policy)** | A regra do negócio pertence às entidades/políticas de domínio, garantindo reutilização e proteção do estado. | Exceção `SchedulingTooSoonException`. | 
| Consultar conflitos no Postgres e salvar a entidade | **Infraestrutura (Repository Adaptor)** | O TypeORM e a consulta SQL são detalhes de implementação técnica para armazenamento persistente. | `TypeOrmSchedulingRepository` e Query Builder. | 

### Etapa 11 — Comportamento, Estados e Invariantes

1. **Conceito de Domínio que Protege a Decisão:** Entidade de Domínio `Scheduling`.

2. **Operação com Intenção de Negócio:** `Scheduling.request()` em vez de atribuições diretas como `scheduling.status = 'PENDING'`.

3. **Condições Impeditivas:**

   * Solicitação realizada com menos de 24 horas do horário de início (`startsAt`).

   * Existência de agendamento prévio conflitante no intervalo `[startsAt, endsAt]` para o mesmo `professionalId`.

4. **Informações Fornecidas:** `clientId`, `professionalId`, `startsAt`, `endsAt`, `serviceId`.

#### Diagrama de Estados das Transições de Agendamento (Mermaid)

```
stateDiagram-v2
    [*] --> PENDING: Solicitação Válida (RN01 e sem Conflito)
    PENDING --> CONFIRMED: Profissional Aceita / Pagamento Aprovado
    PENDING --> CANCELLED: Cliente/Profissional Cancela ou Expira
    CONFIRMED --> CANCELLED: Cancelamento prévio com regra de reembolso
    CONFIRMED --> COMPLETED: Atendimento Realizado
    CANCELLED --> [*]
    COMPLETED --> [*]
```

### Etapa 12 — Portas, Persistência e Limites Técnicos

| Necessidade | Porta / Abstração candidata | Quem a utiliza? | Possível implementação futura | Justificativa | 
| ----- | ----- | ----- | ----- | ----- | 
| Buscar agendamentos conflitantes no intervalo | `ISchedulingRepository` | `SchedulingService` (Aplicação) | `TypeOrmSchedulingRepository` | Isola a lógica de aplicação dos detalhes do ORM TypeORM e das tabelas SQL. | 
| Salvar novo agendamento | `ISchedulingRepository` | `SchedulingService` (Aplicação) | `TypeOrmSchedulingRepository` | Permite trocar o mecanismo de persistência ou utilizar mocks em testes unitários. | 
| Obter data/hora atual para validação de 24h | `IDateProvider` | Domínio / Aplicação | `SystemDateProvider` | Garante determinismo em testes automatizados ao manipular o relógio do sistema. | 

* **Definição de Repository:** Um repositório no conceito de DDD é uma ilusão de uma coleção em memória de objetos de domínio. Não é uma tabela de banco de dados nem uma entidade ORM, mas sim um contrato (interface) mantido no domínio/aplicação.

### Etapa 13 — Fronteira HTTP e Intenção do Caso de Uso

| Elemento | Proposta da Equipe | Justificativa | 
| ----- | ----- | ----- | 
| **Método e Rota** | `POST /schedulings` | O método `POST` expressa a intenção de criar um recurso de agendamento. | 
| **Dados Recebidos na Entrada** | `professionalId`, `clientId`, `serviceId`, `startsAt`, `endsAt` | Atributos indispensáveis para identificar as partes, o serviço e a janela temporal. | 
| **Necessidade de DTO** | **Sim** (`RequestSchedulingDto`) | Isola o contrato exposto na API das entidades de domínio internas. | 
| **Intenção Command/Query** | **Command** (`RequestSchedulingCommand`) | Trata-se de uma operação que altera o estado do sistema (criação). | 
| **Componente que Coordena** | `SchedulingService` | Application Service responsável por orquestrar o fluxo do caso de uso. | 
| **Comportamento no Domínio** | Criar instância de `Scheduling` com estado inicial `PENDING` validados as regras. | Proteção do domínio e manutenção de invariantes. | 
| **Resultado Independente de HTTP** | Retorno de um objeto `Scheduling` ou DTO de aplicação `SchedulingOutput` | O serviço pode ser reusado em CLI, filas (RabbitMQ) ou gRPC sem sofrer alterações. | 
| **Representação HTTP Possível** | `201 Created` contendo o `schedulingId`, `status: "PENDING"` e timestamps ISO. | Contrato padrão RESTful para criação bem-sucedida de recursos. | 

### Etapa 14 — Resultados, Impedimentos e Falhas

| Cenário do Sistema | Significado | Categoria | Quem reconhece? | Possível tradução HTTP | 
| ----- | ----- | ----- | ----- | ----- | 
| **Agendamento Solicitado com Sucesso** | Agendamento criado com status `PENDING` e persistido. | Sucesso | Domain / Application | `201 Created` | 
| **Data/Hora em Formato Inválido** | String de data enviada não é uma ISO-8601 válida. | Entrada | `ValidationPipe` (NestJS) | `400 Bad Request` | 
| **Antecedência Menor que 24 Horas** | Tentativa de agendamento violando a regra RN01. | Domínio | Entity / Policy (`SchedulingTooSoonException`) | `422 Unprocessable Entity` | 
| **Horário Conflitante para o Profissional** | Profissional já possui agendamento no mesmo horário. | Domínio / Aplicação | Repository / Service (`SchedulingConflictException`) | `409 Conflict` | 
| **Profissional / Serviço Não Encontrado** | O ID informado não existe na base de dados. | Aplicação | `SchedulingService` | `404 Not Found` | 
| **Erro de Conexão com o PostgreSQL** | Falha de infraestrutura ou rede na comunicação com o DB. | Infraestrutura | TypeORM / Driver Postgres | `500 Internal Server Error` | 

### Etapa 15 — Diagrama do Slice Futuro (Mermaid)

Diagrama de sequência do fluxo de solicitação de agendamento (UC01) integrando todas as camadas da aplicação:

```
sequenceDiagram
    autonumber
    actor Cliente as Cliente HTTP
    participant Pipe as ValidationPipe
    participant Controller as SchedulingController
    participant Filter as DomainHttpExceptionFilter
    participant Service as SchedulingService
    participant Domain as Scheduling (Domain Entity)
    participant Repo as SchedulingRepository (TypeORM)
    participant DB as PostgreSQL (Schema scheduling)

    Cliente->>Pipe: POST /schedulings (JSON Body)
    alt DTO com campos inválidos ou datas incorretas
        Pipe-->>Cliente: 400 Bad Request (JSON de Erro)
    else DTO Válido
        Pipe->>Controller: RequestSchedulingDto (Convertido)
        Controller->>Service: requestScheduling(command)
        
        Service->>Repo: hasOverlappingScheduling(professionalId, startsAt, endsAt)
        Repo->>DB: SELECT EXISTS(...) FROM scheduling.schedulings
        DB-->>Repo: Retorna boolean (true/false)
        
        alt Há Conflito de Horário
            Service-->>Filter: throw SchedulingConflictException
            Filter-->>Cliente: 409 Conflict
        else Horário Livre
            Service->>Domain: Scheduling.create(payload)
            alt Antecedência < 24 horas (RN01)
                Domain-->>Service: throw SchedulingTooSoonException
                Service-->>Filter: Intercepta Exceção
                Filter-->>Cliente: 422 Unprocessable Entity
            else Regra Validada (>= 24h)
                Domain-->>Service: Instância com Status PENDING
                Service->>Repo: save(scheduling)
                Repo->>DB: INSERT INTO scheduling.schedulings ...
                DB-->>Repo: Confirmado (Row Persistida)
                Repo-->>Service: Scheduling Salvo
                Service-->>Controller: SchedulingOutputDto
                Controller-->>Cliente: 201 Created (JSON Response)
            end
        end
    end
```

### Etapa 16 — Próximos Passos, Rastreabilidade e Governança

#### Plano de Desenvolvimento Futuro

| Passo futuro | Objetivo | Dependência | Evidência de conclusão esperada | 
| ----- | ----- | ----- | ----- | 
| **Implementar UC02 — Confirmar Agendamento** | Permitir que o profissional aceite agendamentos pendentes. | Conclusão do UC01 | Rota `PATCH /schedulings/:id/confirm` com testes E2E aprovados. | 
| **Adicionar Migrations Automáticas** | Desativar `synchronize: true` em favor de arquivos de migração. | TypeORM CLI configurado | Scripts de migração criados em `src/infra/migrations`. | 
| **Implementar Autenticação JWT** | Proteger o endpoint `POST /schedulings` identificando o cliente. | Módulo de Auth / Users | Guards do NestJS rejeitando requisições sem Token (`401 Unauthorized`). | 

#### Tabela de Pendências / Revisões Documentais

| Artefato anterior | Revisão percebida | Motivo / Evidência | Impacto futuro | Prioridade | 
| ----- | ----- | ----- | ----- | ----- | 
| Blueprint de Arquitetura | Atualização da nomenclatura de schemas no PostgreSQL. | Necessidade de isolar a tabela no schema `scheduling` em vez de `public`. | Manutenção da estrutura de scripts SQL. | Média | 
| Modelo de Casos de Uso | Ajuste nas mensagens de erro do UC01. | Mapeamento explícito de respostas HTTP `422` e `409` no `DomainHttpExceptionFilter`. | Alinhamento do contrato da API com o frontend. | Baixa | 

### Etapa 17 — Reflexão Crítica

 1. **Que evidência demonstra que sua aplicação NestJS está funcionando, além do log de inicialização?**

    *Resposta:* A validação por meio da suíte de testes E2E automatizada executada via Vitest/Supertest. A suíte envia requisições HTTP reais contra o servidor em memória, que por sua vez se conecta ao banco de dados PostgreSQL relacional no Docker, grava os dados no schema `scheduling` e valida tanto os códigos HTTP retornados (`201 Created` e `422 Unprocessable Entity`) quanto a persistência física das linhas via consulta de verificação no repositório.

 2. **Que dificuldade técnica ocorreu? Se não houve, qual checkpoint evitou intervenções desnecessárias?**

    *Resposta:* Ocorreu um erro de runtime `42P01` (relação não encontrada) no PostgreSQL devido à ausência do schema `scheduling` e a falta das tabelas. O checkpoint do comando `docker exec -it conecta-postgres psql ...` permitiu diagnosticar o usuário correto (`conecta_user`), criar o schema manualmente e habilitar a flag `synchronize: true` no `TypeOrmModule`, evitando a reinicialização desnecessária do container Docker.

 3. **Qual diferença foi percebida entre Nest CLI e framework NestJS?**

    *Resposta:* A Nest CLI é uma ferramenta de desenvolvimento utilitária para automação de tarefas (geração de arquivos, build, start), enquanto o framework NestJS é o conjunto de bibliotecas em tempo de execução que fornece os decoradores, o sistema de injeção de dependências e a abstração arquitetural.

 4. **O que os arquivos do template explicam sobre a requisição `GET /`?**

    *Resposta:* Explicam o desacoplamento básico do NestJS: o `src/main.ts` inicia o servidor; o `src/app.module.ts` conecta componentes; o `src/app.controller.ts` captura o verbo `@Get('/')` e delega a regra para o `src/app.service.ts`, que retorna a string `'Hello World!'`.

 5. **Por que o NestJS não determina sozinho a arquitetura do sistema?**

    *Resposta:* Porque o NestJS é um framework de infraestrutura e organização de módulos backend. Ele não tem conhecimento sobre as regras de negócio, os limites dos contextos delimitados (Bounded Contexts) ou a proteção de invariantes do domínio. A arquitetura de software (Clean Architecture / DDD) é uma decisão de design imposta pelo desenvolvedor sobre o código.

 6. **Em que o primeiro slice da equipe difere do Hello World!?**

    *Resposta:* O Hello World! é um canal sem estado que retorna dados estáticos. O primeiro slice (UC01) é um fluxo vertical completo de produção: processa parâmetros dinâmicos, executa validações temporais, consulta e grava informações persistentes em um SGBD relacional e traduz falhas do negócio para contratos HTTP adequados.

 7. **Qual regra do domínio exigirá maior cuidado durante a implementação?**

    *Resposta:* A regra de não sobreposição de horários (concorrência). Em cenários de múltiplas solicitações simultâneas para o mesmo profissional, é necessário garantir bloqueios otimistas/pessimistas no banco de dados para evitar o problema de *double-booking*.

 8. **Qual responsabilidade seria fácil posicionar incorretamente em um Service ou Controller?**

    *Resposta:* A validação de antecedência temporal de 24 horas (RN01). É comum desenvolvedores colocarem essa verificação `if (startsAt - now < 24h)` diretamente dentro do Controller ou do Application Service, desprotegendo o Domínio e duplicando a regra em diferentes canais.

 9. **Que parte do blueprint anterior foi confirmada ou precisa ser refinada?**

    *Resposta:* Foi confirmada a estrutura do mapa de módulos do NestJS e a utilização do TypeORM como ORM. Precisa ser refinado o isolamento de esquemas de banco de dados e a estratégia de execução das migrações SQL sem depender da sincronização automática em produção.

10. **O que ainda não está pronto para afirmar que o primeiro slice do seu sistema funciona?**

    *Resposta:* O slice está totalmente funcional para o escopo delimitado (criação e validação do agendamento). Contudo, para operação comercial em produção, faltam as etapas de autenticação do usuário (JWT), autorização de papéis (Guards) e a integração com a camada de pagamentos.

## Apêndice — Uso de Inteligência Artificial

* **Ferramenta Utilizada:** Gemini 3 Flash / ChatGPT (OpenAI).

* **Finalidade:** Apoio na depuração de erros do Docker/PostgreSQL (solução do erro `42P01` e identificação de usuário de banco), auxílio na estruturação das tabelas em Markdown e validação do padrão de mapeamento do `DomainHttpExceptionFilter`.

* **Exemplos de Perguntas Feitas:**

  * *"Como resolver o erro `relation scheduling.schedulings does not exist` no TypeORM com PostgreSQL no NestJS?"*

  * *"Como estruturar um ExceptionFilter no NestJS que converta exceções customizadas de domínio para códigos HTTP sem estourar `BaseExceptionFilter`?"*

* **Contribuição Real:** Diagnóstico rápido de divergências de configuração do container Docker e auxílio na sintaxe dos diagramas em blocos Mermaid.

* **Sugestões Rejeitadas/Corrigidas:** Foi rejeitada uma sugestão inicial da IA para instalar o pacote `@nestjs/common` globalmente para solucionar filtros de exceção, optando-se pela correção interna na árvore de injeção de dependências do módulo.

* **Método de Verificação:** Execução direta dos testes automatizados E2E (`pnpm test:e2e`) no terminal após cada alteração e verificação direta do banco de dados via `psql`.

* **Resposta à pergunta de controle:**

  *Conseguiríamos explicar e repetir os procedimentos e defender as decisões sem a IA?*

  **Sim.** Todos os procedimentos executados referem-se à manipulação direta da CLI do NestJS, Docker, PostgreSQL e TypeORM. As decisões de design arquitetural (desacoplamento da camada de domínio, filtros de exceção e validação em borda via DTO) foram fundamentadas nos princípios da Engenharia de Software e Clean Architecture ministrados na disciplina.
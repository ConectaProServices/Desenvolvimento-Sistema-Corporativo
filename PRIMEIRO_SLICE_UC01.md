# Do Domínio à API Persistente: Implementação e Validação do Primeiro Slice

**Disciplina:** Desenvolvimento de Sistemas - DSC 2026/1  
**Base de estudo:** Materiais da Missão 07

---

## Etapa 1: Identificação do projeto e rastreabilidade

| Item | Informação |
| :--- | :--- |
| **Sistema** | Conecta Pro Services |
| **Aluno(s)/ equipe** | João Pedro Zanette Sartori |
| **URL da Organization** | https://github.com/ConectaProServices |
| **URL do repositório** | https://github.com/ConectaProServices/conecta-pro-services |
| **Branch e referência de commit** | https://github.com/ConectaProServices/conecta-pro-services/commit/8c0a19f0475476973e2c3c13f5f81efbcc82e566 |
| **Caso de Uso selecionado** | UC01 - Solicitar Agendamento |
| **Ator e objetivo** | Cliente: Deseja solicitar um agendamento com um profissional para um serviço específico. |
| **Regra principal do domínio** | RN01: O agendamento exige uma antecedência mínima de 24 horas. / INV-1: Um profissional não pode ter agendamentos sobrepostos (conflito de horários). |
| **Resultado observável** | A persistência de um agendamento com status `PENDING` ou a rejeição via API com erro 422 (antecedência) / 409 (conflito). |
| **Documentação de origem** | Missão 07 (Unidades 01 a 08) e Blueprint de Arquitetura Limpa. |

**Justificativa:** O recorte escolhido (UC01) representa o fluxo principal de entrada de valor no sistema de serviços, sendo a ação central que o cliente executa. Estão contempladas neste slice as validações de domínio (24h), bloqueio de concorrência e a persistência via TypeORM em PostgreSQL. Pagamentos e notificações ficaram de fora deste slice.

---

## Etapa 2: Levantamento do estado inicial e preservação da arquitetura

| Responsabilidade | Elemento real do projeto | Já existe? | Alteração necessária? |
| :--- | :--- | :--- | :--- |
| Regra e comportamento de domínio | `Scheduling.request()` e exceptions | Sim | Nenhuma |
| Coordenação do Caso de Uso | `SchedulingService.requestScheduling()` | Sim | Nenhuma |
| Porta(s) de persistência | `ISchedulingRepository` | Sim | Nenhuma |
| Adaptador(es) concretos | `TypeOrmSchedulingRepository` | Sim | Ajustes para TypeORM |
| Representação HTTP | `SchedulingController` e Filtros | Não | Criados nesta etapa |

---

## Etapa 3: PostgreSQL isolado em Docker Compose

**Configuração do `docker-compose.yaml`:**
- **Imagem:** `postgres:17`
- **Porta:** `5432:5432`
- **Variáveis:** `POSTGRES_USER=conecta_user`, `POSTGRES_DB=conecta_db`
- **Volume:** `conecta_postgres_data:/var/lib/postgresql/data` (Garante a persistência dos dados entre reinicializações do container).
- **Healthcheck:** Utiliza `pg_isready` para validar a inicialização do banco.

**Validação de persistência:** Ao derrubar e subir o container sem a flag `-v`, os dados de agendamentos criados nos testes persistiram intactos devido ao volume mapeado.

---

## Etapa 4: TypeORM e fronteira com o domínio

| Conceito do domínio | Representação TypeORM | Motivo da separação |
| :--- | :--- | :--- |
| `Scheduling` (Entidade Rica) | `SchedulingEntity` | O domínio não deve ter dependências do banco (como decorators `@Column`). A separação garante que regras de negócio rodem puramente em TypeScript. |

*Verificação: A classe `Scheduling` e o `SchedulingService` não importam bibliotecas do TypeORM.*

---

## Etapa 5: Conexão NestJS-PostgreSQL e configuração segura

A integração foi feita no `AppModule` utilizando o `ConfigModule` e injeção assíncrona do `ConfigService` para ler variáveis de ambiente (`DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME`). A propriedade `synchronize: false` foi mantida por segurança. O arquivo `.env` foi adicionado ao `.gitignore`.

---

## Etapa 6: Modelos de persistência derivados do próprio domínio

| Elemento do domínio | Campo persistido | Tipo/identidade/restrição | Justificativa |
| :--- | :--- | :--- | :--- |
| `schedulingId` | `schedulingId` | `uuid` (Primary Column) | Identificação única delegada ao domínio. |
| `startsAt` / `endsAt` | `startsAt` / `endsAt` | `timestamp with time zone` | Previne bugs de fuso horário, crucial para agendamentos. |
| `status` | `status` | `enum` (`SchedulingStatus`) | Reflete a máquina de estados do agendamento (PENDING, CONFIRMED). |

---

## Etapa 7: Migrations: esquema versionado e reproduzível

As migrations foram configuradas no projeto. Elas materializam as estruturas definidas na entidade ORM (`SchedulingEntity`), como as colunas do tipo UUID, chaves primárias e campos temporais (`createdAt`, `deletedAt`). Elas criam (via método `up`) a tabela `schedulings` e a removem (via `down`), mantendo o banco seguro contra o `synchronize: true`.

---

## Etapa 8: Mappers: tradução sem executar regras

O `SchedulingOrmMapper` foi implementado possuindo dois métodos:
- `toDomain(entity: SchedulingEntity)`: Constrói a entidade de domínio a partir dos dados brutos do banco, utilizando o construtor da classe `Scheduling` para restaurar o estado sem disparar a regra `.request()` novamente.
- `toPersistence(scheduling: Scheduling)`: Converte o modelo rico para o formato aceito pelo TypeORM.

---

## Etapa 9: Adaptadores concretos e injeção de dependências

O `TypeOrmSchedulingRepository` implementa a interface `ISchedulingRepository`. No método genérico `existsOverlapping`, a verificação de concorrência foi feita utilizando `LessThan` e `MoreThan` nas colunas de data do TypeORM. No momento de salvar (`save`), a infraestrutura captura violações no PostgreSQL (código `23P01`) e as converte na exceção de domínio `SchedulingConflictException`. O provider está registrado no NestJS utilizando `useExisting` para a Inversão de Dependência.

---

## Etapa 10: Exposição HTTP do Caso de Uso

| Elemento | Contrato real |
| :--- | :--- |
| **Método HTTP** | `POST` |
| **Rota** | `/schedulings` |
| **Parâmetros / body** | `customerId`, `serviceId`, `professionalId`, `startsAt` |
| **Validação de entrada** | `RequestSchedulingDto` com uso do `class-validator` (`@IsUUID`, `@IsDate`) |
| **Componente acionado** | `SchedulingService.requestScheduling(dto)` |
| **Resposta de sucesso** | `201 Created` retornando o DTO com id, datas e status. |

---

## Etapa 11: Resultados, erros e tradução HTTP

| Cenário | Categoria | Onde é identificado | Resposta HTTP esperada | Justificativa |
| :--- | :--- | :--- | :--- | :--- |
| **Sucesso** | Sucesso | Controller | `201 Created` | Recurso criado com sucesso. |
| **Antecedência < 24h** | Domínio (RN01) | Domínio (`Scheduling.request`) | `422 Unprocessable Entity` | A requisição é legível, mas a regra de negócio impede a criação. |
| **Conflito de Horário** | Aplicação (INV-1) | Infra/Aplicação | `409 Conflict` | O estado atual (agenda do profissional) conflita com a nova requisição. |
| **UUID Malformado** | Entrada | Filtro de DTO (Pipes) | `400 Bad Request` | Falha de formatação na requisição HTTP. |

O filtro `DomainHttpExceptionFilter` centraliza a tradução dessas exceções do domínio para os status correspondentes do Express.

---

## Etapa 12 e 13: Cenários reais com PostgreSQL e Testes E2E

Foi construído o script `seed-test-scenarios.ts` para injetar agendamentos reais na tabela. O arquivo de testes HTTP `test-scenarios.http` permite simular chamadas manuais. 

O arquivo `request-scheduling.e2e-spec.ts` utiliza **Supertest** para testar as requisições de ponta a ponta. 
- O cenário feliz confirma o status HTTP 201 e faz uma busca no repositório para certificar-se de que o dado está na tabela do PostgreSQL.
- O cenário de erro valida a antecedência mínima e comprova que o status retornado é o 422, com a string `SchedulingTooSoonException`.
- A limpeza dos dados ocorre no `afterAll`, removendo as IDs via `.delete()`.

---

## Etapa 14: Diagrama do fluxo realmente implementado

```mermaid
sequenceDiagram
    actor Cliente
    participant Controller as SchedulingController
    participant Pipe as RequestSchedulingDto
    participant App as SchedulingService
    participant Dom as Scheduling (Entidade)
    participant Repo as ISchedulingRepository
    participant TypeORM as TypeOrmSchedulingRepository
    participant DB as PostgreSQL

    Cliente->>Controller: POST /schedulings (JSON)
    Controller->>Pipe: Valida UUID e Data
    Pipe-->>Controller: DTO validado
    Controller->>App: requestScheduling(dto)
    
    App->>Dom: Scheduling.request(dados, now)
    
    alt Antecedência menor que 24h
        Dom-->>App: throw SchedulingTooSoonException
        App-->>Controller: Falha (Repassada ao Filtro)
        Controller-->>Cliente: 422 Unprocessable Entity
    else
        Dom-->>App: Instância válida (PENDING)
        App->>Repo: existsOverlapping()
        Repo->>TypeORM: Busca conflitos de data
        TypeORM->>DB: Consulta SQL (Less/More)
        DB-->>TypeORM: Resultado
        TypeORM-->>Repo: false
        
        App->>Repo: save(scheduling)
        Repo->>TypeORM: toPersistence() + save
        TypeORM->>DB: INSERT INTO schedulings
        DB-->>TypeORM: Gravação confirmada
        TypeORM-->>Repo: toDomain()
        Repo-->>App: Instância salva
        App-->>Controller: Objeto de Resposta
        Controller-->>Cliente: 201 Created (JSON)
    end
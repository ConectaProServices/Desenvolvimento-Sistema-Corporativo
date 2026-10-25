# Blueprint de Implementação do Primeiro Slice do Sistema

**Projeto:** Sistema de Agendamento de Serviços (Salão, Clínica, Oficina)

## 01. Retomada do sistema

* **Nome do sistema:** Sistema de Agendamento de Serviços (Salão, Clínica, Oficina).

* **Problema central:** Agendas manuais e descentralizadas (WhatsApp, cadernos) gerando conflitos de horários, *no-shows* sem punição e ausência de histórico de reputação dos clientes/prestadores.

* **Público/atores principais:** Usuário (que assume dinamicamente o papel de Cliente ou Prestador) e Administrador.

* **Objetivo principal da solução:** Ser um marketplace genérico com catálogo centralizado, blindando a agenda por meio de validações matemáticas de tempo e um sistema de reputação mútua obrigatória (substituindo punições financeiras).

* **Principais conceitos do domínio:** Usuário (Entidade), Agendamento (Aggregate Root), Serviço (Fotografia/Objeto de Valor), Janela de Tempo (Objeto de Valor), Agenda (Entidade na Disponibilidade), Avaliação.

* **Principais Regras de Negócio:**

  * RN01: Mínimo de 24h de antecedência para solicitar reserva.

  * RN02: Timeout de expiração 10h antes do evento caso não haja resposta.

  * RN03: Limite antifraude de 1 agendamento por data por cliente.

  * INV-1: Reservas ativas do mesmo prestador não podem ter janelas de tempo sobrepostas.

* **Principais decisões arquiteturais:** Monólito Modular em Camadas (ADR-001) e Bloqueio Síncrono de Horário no ato da solicitação (ADR-002) acoplando os contextos de Agendamento e Disponibilidade na mesma transação.

## 02. Atores e objetivos

| Ator | Objetivo | Resultado observável | 
| ----- | ----- | ----- | 
| **Cliente** (Papel do Usuário) | Encontrar um serviço e reservar um horário garantido. | Solicitação registrada como "Pendente" e horário provisoriamente bloqueado. | 
| **Prestador** (Papel do Usuário) | Ofertar serviços, controlar sua disponibilidade e aprovar intenções de reserva. | Ofertas criadas no catálogo, horários gerados na agenda, agendamentos aprovados/recusados. | 
| **Administrador** | Moderar a plataforma e manter o ambiente seguro. | Relatórios extraídos, contas suspensas por violação de regras. | 

## 03. Identificação e revisão dos Casos de Uso

| ID | Caso de Uso | Ator principal | Objetivo | Resultado observável | Situação | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| UC01 | Solicitar Agendamento | Cliente | Garantir reserva de um horário para um serviço específico. | Solicitação criada e horário bloqueado. | Mantido | 
| UC02 | Aprovar Solicitação | Prestador | Confirmar o compromisso de atendimento. | Agendamento passa para "Confirmado", bloqueio vira definitivo. | Mantido | 
| UC03 | Recusar Solicitação | Prestador | Rejeitar um atendimento por motivo diverso. | Agendamento passa para "Recusado", horário é liberado. | Mantido | 
| UC04 | Cadastrar Disponibilidade | Prestador | Disponibilizar janelas de tempo livres. | Horários aparecem abertos para clientes no catálogo. | Novo (Refinado) | 
| UC05 | Avaliar Atendimento | Cliente / Prestador | Registrar qualidade do serviço e atendimento. | Notas computadas na reputação do perfil avaliado. | Mantido | 

## 04. Verificação dos Casos de Uso

Todas as propostas acima passaram pelo crivo das três perguntas: possuem ator com objetivo claro, resultado observável e interação relevante.

| Item inicialmente identificado | Classificação inicial | Classificação atual | Justificativa | 
| ----- | ----- | ----- | ----- | 
| Expirar Agendamento (Timeout) | Caso de Uso | Regra de Domínio / Job Interno | Não é disparado pelo Ator humano, mas sim por uma política de tempo do domínio (RN02). | 
| Bloquear Horário na Agenda | Caso de Uso | Operação Interna | É consequência direta do UC01; a agenda é acionada internamente para garantir a INV-1. | 

## 05. Organização por ator e contexto

```
flowchart TB
    C[Cliente]
    P[Prestador]

    C --> UC01[UC01 - Solicitar Agendamento]
    C --> UC05[UC05 - Avaliar Atendimento]
    
    P --> UC02[UC02 - Aprovar Solicitação]
    P --> UC03[UC03 - Recusar Solicitação]
    P --> UC04[UC04 - Cadastrar Disponibilidade]
    P --> UC05

    subgraph Contexto: Agendamento e Disponibilidade
        UC01
        UC02
        UC03
        UC04
    end

    subgraph Contexto: Avaliação e Reputação
        UC05
    end

```

## 06. Especificações funcionais dos Casos de Uso centrais

### UC01: Solicitar Agendamento

* **Objetivo:** Permitir ao cliente reservar um horário na agenda de um prestador.

* **Ator principal:** Cliente.

* **Pré-condições:** O Cliente não deve estar bloqueado pelo *Hard Block*. O Serviço deve estar ativo no catálogo. A conta do cliente e do prestador não podem ser a mesma (RN05).

* **Pós-condições:** Agendamento salvo como PENDENTE. Janela de tempo bloqueada na agenda do Prestador.

* **Fluxo principal:** O Cliente seleciona Prestador, Serviço e Data/Hora. O sistema valida regras, tira uma "fotografia" do Serviço, cria o agendamento pendente e trava a agenda.

* **Fluxos alternativos:**

  * Data solicitada menor que 24h (falha de regra).

  * Cliente já possui reserva ativa no dia (falha antifraude).

  * Horário acabou de ser ocupado por concorrência (falha de concorrência / INV-1).

* **Regras relacionadas:** RN01 (24h antecedência), RN03 (1 reserva/dia), RN05 (cliente!=prestador), INV-1 (sem conflito).

### UC02: Aprovar Solicitação

* **Objetivo:** O prestador aceita o pedido de reserva.

* **Ator principal:** Prestador.

* **Pré-condições:** O Agendamento deve estar no estado PENDENTE.

* **Pós-condições:** Agendamento vira CONFIRMADO. O bloqueio da agenda se torna definitivo.

* **Fluxo principal:** O Prestador visualiza detalhes da solicitação e aciona "Aprovar". Sistema atualiza o status.

* **Fluxos alternativos:** A solicitação já expirou pelo sistema (RN02), impedindo a aprovação tardia.

### UC04: Cadastrar Disponibilidade

* **Objetivo:** O prestador insere os blocos de horário que atende na plataforma.

* **Ator principal:** Prestador.

* **Pré-condições:** Possuir cadastro ativo.

* **Pós-condições:** Novas janelas de tempo são criadas e ficam disponíveis na busca.

* **Fluxo principal:** Prestador informa dias da semana e intervalos (ex: Seg-Sex, 08h-18h). Sistema persiste os blocos.

* **Regras relacionadas:** RN06 (A agenda pertence ao usuário; não pode conflitar com horários em que ele é Cliente).

## 07. Rastreabilidade com o domínio

| Caso de Uso | Conceitos do domínio | Regras / Invariantes relacionadas | Contexto / responsabilidade | 
| ----- | ----- | ----- | ----- | 
| UC01 - Solicitar Agendamento | Agendamento, JanelaTempo, Serviço (Snapshot), Agenda | RN01, RN03, RN05, INV-1 | Agendamento (Núcleo) e Disponibilidade (Núcleo) | 
| UC02 - Aprovar Solicitação | Agendamento, Agenda, Timeout | RN02, INV-1 | Agendamento | 
| UC04 - Cadastrar Disponibilidade | Agenda, JanelaTempo | RN06 | Disponibilidade | 

* **O que justifica o UC01?** A dor principal documentada: conflitos de horários em processos manuais (WhatsApp).

* **Houve lacuna revelada?** A necessidade estrita de extrair a "fotografia" do Serviço no momento do UC01 reafirma que não podemos simplesmente guardar o `ID` do serviço, pois preços podem mudar futuramente no catálogo.

## 08. Dependências entre Casos de Uso

| Caso de Uso | Depende de | Tipo de dependência | Justificativa | 
| ----- | ----- | ----- | ----- | 
| UC01 (Solicitar) | UC04 (Disponibilidade) | Informação / Estado | Só é possível solicitar agendamento em horários previamente cadastrados como disponíveis pelo Prestador. | 
| UC02 (Aprovar) | UC01 (Solicitar) | Estado | Só se aprova uma intenção que foi registrada e está Pendente. | 
| UC05 (Avaliar) | UC02 (Aprovar) | Estado | Exige que o agendamento tenha passado para Confirmado e posteriormente Concluído (INV-2). | 

## 09. Priorização e MVP

| Caso de Uso | Valor central | Dependências | Risco | Pode ser adiado? | Classificação | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| UC01 (Solicitar) | Sim, é o core transacional. | UC04 | Alto (Concorrência) | Não | Central do MVP | 
| UC04 (Cadastrar Disp.) | Sim, base estrutural. | Nenhuma | Baixo | Não | Central do MVP | 
| UC02 (Aprovar) | Sim, gera segurança. | UC01 | Médio | Não | Central do MVP | 
| UC03 (Recusar) | Sim, libera agenda. | UC01 | Médio | Não | Central do MVP | 
| UC05 (Avaliar) | Sim, garante reputação. | UC02 | Médio | Sim, numa V2 | Secundário MVP | 

## 10. Escolha do primeiro slice

**Caso de Uso escolhido:** UC01 - Solicitar Agendamento.

* **Ator:** Cliente.

* **Valor entregue:** Concretiza a promessa de valor do sistema de conectar cliente e prestador com uma reserva blindada contra overbooking.

* **Representatividade:** Ele exercita o núcleo transacional mais crítico. É onde as regras de negócio mais complexas (RN01 - 24h, RN03 - limite/dia, INV-1 - conflito simultâneo) se encontram.

* **Dependências cruzadas:** Ele exige testar na prática o ADR-002 (comunicação síncrona entre o contexto de Agendamento e Disponibilidade para o bloqueio).

* **Por que é o primeiro recorte adequado?** Se o sistema falhar em registrar solicitações de forma consistente, os demais fluxos (aprovar, avaliar) deixam de existir. Ele consolida a arquitetura.

**Definição clara:**

* **Entra no slice:** Recebimento da requisição HTTP, coordenação do Application Service, consulta ao catálogo para tirar o "snapshot" do serviço, instanciamento da Entidade de Agendamento, validação de antecedência de 24h (RN01), solicitação síncrona de bloqueio provisório para a Agenda, persistência da reserva Pendente.

* **Não entra no slice:** Aprovação pelo prestador (UC02), job assíncrono de Timeout (RN02), envio real de e-mails/SMS, lógica complexa de busca (filtros avançados).

## 11. Linguagem ubíqua do slice

| Termo | Significado no sistema | Termos que não devem ser usados como sinônimo | 
| ----- | ----- | ----- | 
| **Agendamento** | O compromisso transacional em si (a reserva). | Pedido, Evento, Reunião, Appointment. | 
| **Serviço Contratado** | A fotografia (snapshot) com os dados imutáveis no momento da solicitação. | Catálogo, Oferta. | 
| **Agenda** | O agregado de horários pertencente a um usuário no contexto de Disponibilidade. | Calendário, Horários. | 
| **Bloqueio Provisório** | Status do horário retido por uma solicitação ainda não aprovada pelo prestador. | Ocupado, Indisponível. | 

## 12. Regra central e comportamento do domínio

**A decisão principal de negócio é:** Proteger a Invariante contra *Overbooking* (INV-1) associada à Regra de Antecedência Mínima (RN01).

* **Quem conhece a regra:** O conceito `Agendamento` conhece a RN01 (Tempo). O conceito `Agenda` conhece a INV-1 (Concorrência espacial).

* **O que muda:** O estado da Agenda passa a conter um bloqueio em determinada Janela de Tempo. O Agendamento nasce com estado `PENDENTE`.

* **Transições impedidas:** Criar um agendamento com data < 24h no futuro.

* **O que NÃO deve ser feito:** `agendamento.status = "PENDENTE"; agenda.isBlocked = true;`. Isso espalha regra de domínio.

**Comportamento conceitual esperado:**

```
// Domínio rico, não anêmico:
const agendamento = Agendamento.solicitar(
   clienteId, 
   prestadorId, 
   servicoSnapshot, 
   janelaSolicitada, 
   relogioAtual
); // Vai disparar exceção de Domínio se falhar a RN01

agenda.bloquearProvisoriamente(agendamento.id, janelaSolicitada);
// Vai disparar exceção de Domínio se o horário já estiver ocupado (INV-1)

```

## 13. Responsabilidades arquiteturais

| Responsabilidade | Papel arquitetural | Quem deverá assumi-la? | Justificativa | 
| ----- | ----- | ----- | ----- | 
| Receber requisição web | Fronteira / Apresentação | `SolicitarAgendamentoController` | Apenas entende HTTP e traduz para Command. | 
| Coordenar o fluxo | Aplicação | `SolicitarAgendamentoUseCase` | Orquestra busca de dados, invoca o domínio e salva os resultados. Não contém RNs. | 
| Validar as regras (24h) | Domínio | Entidade `Agendamento` | Regras de negócio essenciais ficam encapsuladas na raiz de agregação. | 
| Garantir sem sobreposição | Domínio | Entidade `Agenda` (via Serviço de Domínio/Porta) | Somente a agenda tem a autoridade para bloquear horários. | 
| Persistir estado final | Infraestrutura | `AgendamentoRepositoryImpl` | Resolve o insert no PostgreSQL via ORM sem vazar a tecnologia para a camada de Aplicação. | 

## 14. Modelo mínimo necessário ao slice

Para sustentar o UC01, não precisamos codificar Avaliação, Reputação, Catálogos Complexos ou envio de E-mail.

* **Reutilizados/Necessários:**

  * Entidade **Agendamento**: precisará conhecer Identificadores (Cliente, Prestador), `JanelaDeTempo` (Data, HoraInicio, HoraFim), `SnapshotServico` (Nome, Preco, Duracao) e `Status` (Fixo em `PENDENTE` na criação).

  * Value Object **JanelaDeTempo**: para checagens precisas.

* **Comportamentos necessários:** O método estático ou factory `Agendamento.solicitar()` que embuta as regras de tempo.

* **Fora do slice:** O módulo de Avaliação e o modelo de Usuário com perfis completos, Notificações externas. Os "IDs" de cliente e prestador bastam para este slice transacional.

## 15. Portas e dependências externas

O *Application Service* (UseCase) não faz queries no banco. Ele usa portas.

| Porta / Contrato | Necessidade expressa | Operações necessárias | Quem dependerá dela? | 
| ----- | ----- | ----- | ----- | 
| `AgendamentoRepository` | Gravar a solicitação criada. Verificar limite de 1 por dia. | `salvar(Agendamento)`, `possuiAtivoNaData(clienteId, data)` | `SolicitarAgendamentoUseCase` | 
| `AgendaServicePort` | Reservar o horário na agenda do Prestador (Consistência inter-módulo ADR-002) | `bloquear(prestadorId, janela)` | `SolicitarAgendamentoUseCase` | 
| `CatalogoServicePort` | Obter a "fotografia" do Serviço do catálogo (Nome, Preço, Duração) | `obterSnapshot(servicoId)` | `SolicitarAgendamentoUseCase` | 

## 16. Planejamento da interface HTTP

**Objetivo:** Como o Front-end/Mobile solicitará a criação da reserva.

* **Método:** `POST`

* **Rota:** `/api/v1/agendamentos`

* **Corpo (JSON):**

```
{
  "clienteId": "uuid-cliente",
  "prestadorId": "uuid-prestador",
  "servicoId": "uuid-servico",
  "dataHoraInicio": "2026-11-01T14:00:00Z"
}

```

*(Nota: O token JWT no Header indicará o ator real, clienteId no body será validado contra ele)*

* **Resultado Esperado:**

  * `201 Created` - Agendamento registrado. Retorna ID da transação.

  * `422 Unprocessable Entity` - Caso a antecedência seja < 24h ou horário ocupado.

  * `404 Not Found` - Serviço ou Prestador não existente.

## 17. DTO, Command e Query

| Artefato | Responsabilidade / Justificativa | Fronteira / Tipo | 
| ----- | ----- | ----- | 
| `AgendamentoRequestDTO` | Objeto que o Controller mapeia a partir do JSON HTTP. Valida os tipos primitivos e campos obrigatórios antes de virar intenção. | Transporte (Apresentação) | 
| `SolicitarAgendamentoCommand` | Intenção de mudança que o Controller repassa para a Aplicação. Totalmente agnóstica de HTTP. | Command (Aplicação) | 
| `AgendamentoResponseDTO` | Leva o `id` gerado e o `status` (PENDENTE) de volta para o Controller serializar como resposta JSON. | Transporte (Apresentação) | 

*(Queries não são necessárias pois trata-se puramente de uma operação de escrita/mutação).*

## 18. Planejamento da persistência

Como a persistência sustentará o UC01 preservando o modelo:

* **Recuperação pré-escrita:** Será necessário consultar na tabela de agendamentos se o `clienteId` já possui registro ativo na `data` (RN03).

* **Persistência a salvar:** O objeto raiz `Agendamento` serializado em colunas (incluindo o snapshot do serviço).

* **Bloqueio de Agenda:** O contrato de `AgendaServicePort` persistirá o bloqueio provisório. Para sustentar a INV-1 de forma estrita em um banco relacional, faremos isso na mesma transação.

* **Independência:** O Domínio não conhecerá annotations de SQL ou ORM. A infraestrutura (Implementação de Repository) fará o mapeamento (*Data Mapper*).

## 19. Planejamento de ORM, PostgreSQL e Docker

* **PostgreSQL:** Escolhido pela forte garantia transacional ACID (Crucial para executar a verificação do bloqueio da Agenda e gravação do Agendamento de forma atômica para proteger a INV-1 contra cenários de extrema concorrência simultânea). O domínio ignora a existência do PostgreSQL.

* **ORM (ex: Prisma, TypeORM, Hibernate):** Problema técnico a resolver: mapear o objeto de domínio rico (Agendamento, suas propriedades e o Value Object JanelaTempo) para tabelas/colunas de forma fluida. O ORM fica inteiramente contido na camada de infraestrutura.

* **Docker:** Necessidade operacional: empacotar o PostgreSQL em um container `docker-compose` isolado para que qualquer dev da equipe possa executar a suíte de testes de integração sem poluir sua máquina local.

## 20. Resultados e falhas

| Situação | Categoria | Quem deverá identificar? | Significado interno | Possível tradução HTTP | 
| ----- | ----- | ----- | ----- | ----- | 
| Serviço não encontrado | Aplicação | `SolicitarAgendamentoUseCase` (via porta) | Elemento faltante | `404 Not Found` | 
| Antecedência < 24h | Domínio | `Agendamento` (Entity) | Violação de Regra RN01 | `422 Unprocessable Entity` | 
| Horário sobreposto | Domínio | `Agenda` (via porta de bloqueio) | Violação de INV-1 (Overbooking) | `409 Conflict` (ou 422) | 
| Cliente já agendou hoje | Domínio/App | `SolicitarAgendamentoUseCase` / Regra | Violação de RN03 (Limite) | `422 Unprocessable Entity` | 
| Banco de dados caiu | Infraestrutura | DB Connection / Repository | Indisponibilidade Técnica | `500 Internal Server Error` | 

## 21. Fluxo arquitetural planejado

```
sequenceDiagram
    actor C as Cliente
    participant CT as Controller
    participant APP as UseCase (App)
    participant CAT as CatalogoPort
    participant DOM as Dominio (Agendamento)
    participant AG as AgendaPort (Disp)
    participant REPO as Repository

    C->>CT: POST /api/agendamentos
    CT->>APP: executar(command)
    
    APP->>CAT: obterSnapshot(servicoId)
    CAT-->>APP: {nome, preco, duracao}
    
    APP->>REPO: possuiAtivoNaData(clienteId, data)
    REPO-->>APP: false
    
    APP->>DOM: Agendamento.solicitar(...)
    Note over DOM: Valida RN01 (24h antecedência)
    DOM-->>APP: Entidade Agendamento PENDENTE
    
    APP->>AG: bloquear(prestadorId, janela)
    Note over AG: Valida INV-1 (Sem conflito)
    AG-->>APP: Bloqueio Confirmado
    
    APP->>REPO: salvar(Agendamento)
    REPO-->>APP: OK
    
    APP-->>CT: Resultado (sucesso com ID)
    CT-->>C: 201 Created

```

*(Se qualquer verificação do Domínio - DOM ou AG - falhar, uma DomainException quebra o fluxo imediatamente).*

## 22. Blueprint de implementação

Plano para a futura fase de codificação do slice:

| Ordem | Passo | Objetivo | Dependência | 
| ----- | ----- | ----- | ----- | 
| 1 | Configuração Inicial | Estruturar projeto, Dockerfile do BD e injeção de dependências. | Nenhuma. | 
| 2 | Refinar o Domínio Mínimo | Codificar a classe `Agendamento`, VO `JanelaDeTempo` e os testes unitários da RN01. | 1 | 
| 3 | Definir as Portas (Interfaces) | Criar `AgendamentoRepository`, `CatalogoServicePort`, `AgendaServicePort`. | 2 | 
| 4 | Definir coordenação da Aplicação | Desenvolver o `SolicitarAgendamentoUseCase`, conectando passos lógicos. | 2, 3 | 
| 5 | Planejar e Construir Infra | Mapear ORM e implementar os repositórios (RepositoryImpl, Adapters simulados para Catálogo e Agenda). | 3 | 
| 6 | Planejar Interface HTTP | Criar Controller, rotas e tratamento global de exceções de domínio. | 4, 5 | 
| 7 | Planejar Validação Integrada | Escrever testes de integração (UseCase + Repositório real do BD) validando transação. | 6 | 

## 23. Planejamento de validação

Como garantiremos que a implementação estará correta futuramente:

* **No Domínio (Testes de Unidade):** Passar datas no passado ou daqui a 12h para a factory `Agendamento.solicitar` e validar se a exceção customizada (DomainException) da RN01 é disparada.

* **Na Aplicação / Integração:** Mockar a porta do Catálogo. Fazer o UseCase tentar agendar duas vezes para o mesmo cliente na mesma data e esperar falha na RN03. Injetar um mock da `AgendaServicePort` que simula "ocupado" para atestar a INV-1.

* **Na Persistência:** Subir um container test-DB, e checar se salvar um objeto `Agendamento` resulta em inserts corretos nas colunas da tabela apropriada mapeadas no ORM.

* **Na HTTP (E2E):** Realizar um POST completo e aguardar o `201 Created`, bem como enviar um JSON malformado e aguardar o `400 Bad Request`.

## 24. Governança futura da implementação

O rastreio do plano para o código:

1. **Caso de Uso:** UC01.

2. **Issue Tracker (Jira/Github):** Criação da tarefa `[FEAT-UC01] Implementar solicitação de Agendamento`.

3. **Branch:** `feat/uc01-solicitar-agendamento`.

4. **Commits:** Padronizados, ex: `feat(domain): add Agendamento entity and RN01 rule`.

5. **Pull Request (PR):** Requere aprovação de 1 membro e passagem na pipeline (Testes Unitários).

6. **Revisão:** Foco na separação de responsabilidades (não vazar banco pro UseCase).

7. **Validação:** QA em ambiente isolado batendo no endpoint.

## 25. Revisões descobertas nesta atividade

| Conhecimento anterior | Descoberta atual | Revisão necessária | Impacto | 
| ----- | ----- | ----- | ----- | 
| Acesso indiscriminado da UI às informações de serviço. | A validação do fluxo exigiu uma "porta" (`CatalogoServicePort`) para a camada de aplicação obter a duração do serviço antes de forjar o objeto do Domínio. | Explicitar que a duração (Catálogo) deve ser consumida para construir a Janela de Tempo antes de instanciar o Agendamento. | Clarifica como o Application Service orquestra as dependências internas sem ferir limites. | 
| Tratamento de Agenda como mero UPDATE de tabela. | Para bloquear sincronicamente (ADR-002) sem gerar overbooking massivo, Agenda precisou ser reconhecida como participante do Domínio neste UseCase. | Adoção da `AgendaServicePort` acionada por dentro do *Use Case*. | Reforça a transacionalidade crítica do sistema. | 

## 26. Pendências de governança

| Artefato/documento | Alteração necessária | Motivo | Prioridade | 
| ----- | ----- | ----- | ----- | 
| Mapa de Contextos | Detalhar a comunicação síncrona real e o fallback de rollback distribuído se a ADR-002 for relaxada no futuro. | Manter a documentação em conformidade com o Blueprint caso a escalabilidade seja ameaçada pelo acoplamento das transações. | Média | 
| Modelo Conceitual | Adicionar o objeto `ServicoSnapshot` associado ao `Agendamento`. | O Serviço original pode mudar de preço, a reserva guardará um clone imutável. | Alta | 

## 27. Reflexão crítica

 1. **O Caso de Uso selecionado representa valor real?** Sim, representa o objetivo máximo da plataforma: dar garantia e facilidade de agendamento entre clientes e salões/oficinas.

 2. **O comportamento continuaria fazendo sentido sem HTTP?** Absolutamente. A regra de > 24h e o snapshot do serviço poderiam ser executados via CLI, mensageria ou RPC.

 3. **Qual é a principal Regra de Negócio do slice?** A RN01 (prazo mínimo de 24h) conjugada com a INV-1 (bloqueio contra sobreposição).

 4. **Quem deverá protegê-la?** A Entidade de domínio `Agendamento` e o agregado externo `Agenda`.

 5. **Qual parte deverá apenas coordenar?** O `SolicitarAgendamentoUseCase`.

 6. **Que dependências externas serão necessárias?** As portas (interfaces) que simulam comunicação com Banco de Dados e com outros módulos (Catálogo, Disponibilidade).

 7. **Algum artefato técnico inicialmente imaginado mostrou-se desnecessário?** Endpoints elaborados com queries complexas de ORM no Controller; o Controller limitou-se ao transporte HTTP.

 8. **Alguma responsabilidade estava posicionada incorretamente?** O bloqueio do horário estava inicialmente solto na infraestrutura e foi movido para uma coordenação explícita de porta vinculada à Aplicação/Domínio.

 9. **Qual falha foi mais difícil de classificar?** O limite antifraude (RN03 - 1 agendamento por dia). Ele requer checagem persistida antes do domínio puro; colocou-se a validação como ponte (Aplicação faz query e passa resposta ao Domínio ou falha logo na Aplicação).

10. **A persistência influenciou indevidamente o modelo?** Não. A Entidade foi moldada pelo negócio (janela de tempo, ids, snapshot) e a persistência é tratada como contrato externo (Repositório).

11. **O que este trabalho revelou que a modelagem anterior ainda não mostrava?** Que orquestrar a obtenção do snapshot do Catálogo ANTES de criar o Agendamento é um passo estrutural crítico exigido pela Aplicação.

12. **Que documentação anterior deverá ser revisada?** A atualização fina do modelo conceitual para evidenciar o `ServicoSnapshot` como VO atrelado diretamente ao Agendamento, em distinção à Entidade `OfertaServico` do Catálogo.

13. **A equipe considera o blueprint suficientemente claro para orientar uma implementação futura? Por quê?** Sim. Ele desmistificou o que é "fazer código". O desenvolvedor agora sabe que deve criar a classe `Agendamento` com regras embutidas antes de pensar na tela, que não deverá usar `prisma` dentro dela, e que o controlador só existirá após o core transacional.

## 28. Apêndice - Uso de IA

**Ferramenta utilizada:** Google Gemini Pro 3.1
**Finalidade:** Apoio metodológico de estruturação e formatação do arquivo Markdown para contemplar de maneira exaustiva os 28 passos exigidos no roteiro (PDF).
**Principais contribuições:** Mapeamento visual em Mermaid e questionamento sobre o fluxo rigoroso de orquestração no Application Service vs Domínio puro.
**Sugeriu decisões prematuras?** A IA foi instruída rigidamente a não sugerir códigos nem tabelas SQL, mantendo a resposta estritamente conceitual e baseada na documentação consolidada anterior (ADRs, RNs e INVs).
**Questão final obrigatória:** Se a IA fosse retirada agora, a equipe conseguiria explicar e defender o Caso de Uso, o blueprint, as responsabilidades e todas as decisões registradas?
*Justificativa:* Sim. O domínio e as soluções de arquitetura adotadas (ADR-001 e ADR-002) derivam da evolução genuína do projeto nas fases de modelagem conceitual e delimitação de fronteiras e já eram integralmente dominados pela equipe. A IA atuou restritamente como formatadora para as extensas diretrizes de entrega do documento.
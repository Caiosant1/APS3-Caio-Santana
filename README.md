# APS 3 — Dossiê de Arquitetura Viva: Fila de Prioridade (FabLab Manager)

**Aluno:** Nome Sobrenome
**Unidade Curricular:** Modelagem de Software — Engenharia de Software (4º Período)
**Docente:** Prof. Sidney L. da Silva

## 1. Funcionalidade escolhida

**Fila de Prioridade de Reservas de Máquinas.** No FabLab, várias pessoas disputam as mesmas máquinas (cortadoras a laser, impressoras 3D). Quando duas ou mais reservas pedem a mesma máquina no mesmo horário, o sistema:

1. verifica se a máquina está disponível (não pode estar em manutenção, reservada ou em uso);
2. detecta o conflito de horário (RF03);
3. cria a fila de prioridade (RF04) e calcula a posição de cada reserva;
4. permite ao solicitante consultar sua posição (RF05);
5. permite ao administrador/técnico reordenar a fila apenas em casos excepcionais (RF14);
6. envia notificações de forma assíncrona (RF25).

É crítica para o negócio porque uma fila inconsistente (posições duplicadas ou reservas aprovadas em conflito) quebra a confiança dos usuários no sistema.

Estrutura do repositório:

```
docs/architecture/workspace.puml   (C4-PlantUML, código-fonte do Artefato 4)
docs/architecture/adr-0001.md      (Artefato 5)
docs/diagrams/sequencia.mmd        (Artefato 1)
docs/diagrams/atividades.mmd       (Artefato 2)
docs/diagrams/estados.mmd          (Artefato 3)
```

## 2. Classes e operações públicas (base da regra de consistência)

| Classe / Interface | Operações públicas |
|---|---|
| `ReservaController` | `solicitarReserva(SolicitacaoReservaDTO): ReservaResponseDTO` |
| `ReservaService` | `criarReserva(SolicitacaoReservaDTO): ReservaResponseDTO` |
| `DisponibilidadeService` | `verificarDisponibilidade(maquinaId, periodo): DisponibilidadeDTO` |
| `ReservaRepository` | `buscarConflitos(maquinaId, periodo): List<Reserva>`, `salvar(reserva): Reserva`, `atualizarPosicao(reservaId, posicao): Reserva` |
| `FilaPrioridadeService` | `adicionarNaFila(reserva, conflitos): PosicaoFilaDTO`, `consultarPosicao(reservaId): PosicaoFilaDTO`, `promoverProximo(maquinaId): Reserva`, `reordenarExcepcionalmente(reservaId, novaPosicao, motivo): PosicaoFilaDTO` |
| `NotificacaoPublisher` | `publicarEvento(EventoNotificacaoDTO): void` |

**DTOs:**
- `SolicitacaoReservaDTO` (solicitanteId, maquinaId, periodo)
- `ReservaResponseDTO` (reservaId, status, posicao)
- `DisponibilidadeDTO` (maquinaId, disponivel, motivo)
- `PosicaoFilaDTO` (reservaId, posicao, totalNaFila)
- `EventoNotificacaoDTO` (tipo, destinatarioId, reservaId)

## 3. Artefato 1 — Diagrama de Sequência

Chamadas síncronas usam `->>`; a notificação é assíncrona (`-)`). Há dois fragmentos `alt` (um aninhado) e um `loop`. Todas as mensagens correspondem a operações da tabela da seção 2.

```mermaid
sequenceDiagram
    autonumber
    actor U as Solicitante
    participant C as ReservaController
    participant S as ReservaService
    participant D as DisponibilidadeService
    participant R as ReservaRepository
    participant F as FilaPrioridadeService
    participant N as NotificacaoPublisher

    U->>C: solicitarReserva(SolicitacaoReservaDTO)
    C->>S: criarReserva(SolicitacaoReservaDTO)
    S->>D: verificarDisponibilidade(maquinaId, periodo)
    D-->>S: DisponibilidadeDTO

    alt máquina indisponível (reservada, em uso ou em manutenção)
        S-->>C: ReservaResponseDTO (status = RECUSADA)
    else máquina disponível
        S->>R: buscarConflitos(maquinaId, periodo)
        R-->>S: List~Reserva~

        alt sem conflito de horário
            S->>R: salvar(reserva = PENDENTE_APROVACAO)
            R-->>S: Reserva
        else conflito de horário
            S->>F: adicionarNaFila(reserva, conflitos)
            loop para cada reserva da fila, em ordem de prioridade
                F->>R: atualizarPosicao(reservaId, posicao)
                R-->>F: Reserva
            end
            F-->>S: PosicaoFilaDTO
        end

        S-)N: publicarEvento(EventoNotificacaoDTO)
        S-->>C: ReservaResponseDTO
    end

    C-->>U: ReservaResponseDTO
```

## 4. Artefato 2 — Diagrama de Atividades com Raias

Quatro raias (Solicitante, Back-End, Administrador/Técnico, Serviço de Notificação). As guardas de cada decisão são mutuamente exclusivas. O paralelismo está entre o **FORK** e o **JOIN**: recalcular as posições e enviar a notificação acontecem ao mesmo tempo.

```mermaid
flowchart TD
    INI(("●"))

    subgraph L1["Raia 1 - Solicitante"]
        A1["Solicitar reserva da máquina"]
        A2["Consultar posição na fila"]
    end

    subgraph L2["Raia 2 - Sistema Back-End"]
        B1["Validar perfil e dados da solicitação"]
        B2["Verificar estado da máquina"]
        B3{"Máquina disponível?"}
        B4["Recusar solicitação (status RECUSADA)"]
        B5["Buscar conflitos de horário"]
        B6{"Há conflito?"}
        B7["Criar reserva PENDENTE_APROVACAO"]
        B8["Criar reserva EM_FILA"]
        FORK["━━ FORK ━━"]
        B9["Aplicar regra de prioridade e recalcular posições"]
        JOIN["━━ JOIN ━━"]
        B10["Aguardar promoção: 1ª posição e máquina livre"]
    end

    subgraph L3["Raia 3 - Administrador/Técnico"]
        C1["Analisar reserva"]
        C2{"Decisão do administrador"}
        C3["Aprovar reserva"]
        C4["Negar reserva e registrar justificativa"]
    end

    subgraph L4["Raia 4 - Serviço de Notificação"]
        D1["Enviar notificação ao solicitante"]
    end

    FIM1(("◉"))
    FIM2(("◉"))
    FIM3(("◉"))

    INI --> A1 --> B1 --> B2 --> B3
    B3 -->|"[estado == DISPONIVEL]"| B5
    B3 -->|"[estado != DISPONIVEL]"| B4
    B4 --> FIM3
    B5 --> B6
    B6 -->|"[qtdConflitos == 0]"| B7
    B6 -->|"[qtdConflitos > 0]"| B8
    B7 --> C1
    B8 --> FORK
    FORK --> B9
    FORK --> D1
    B9 --> JOIN
    D1 --> JOIN
    JOIN --> A2
    A2 --> B10
    B10 --> C1
    C1 --> C2
    C2 -->|"[decisao == APROVAR]"| C3
    C2 -->|"[decisao == NEGAR]"| C4
    C3 --> FIM1
    C4 --> FIM2

    style FORK fill:#000,stroke:#000,color:#fff
    style JOIN fill:#000,stroke:#000,color:#fff
```

## 5. Artefato 3 — Diagrama de Transição de Estados (entidade `Reserva`)

Estado inicial `[*]`, subestados dentro de `NaFila`, quatro estados terminais (`Recusada`, `Negada`, `Cancelada`, `Concluida`), transições no formato `Gatilho [Guarda] / Ação()` e ações internas (`entry /`, `do /`, `exit /`).

```mermaid
stateDiagram-v2
    [*] --> Solicitada : solicitarReserva [dadosValidos] / validarSolicitacao()

    Solicitada --> PendenteAprovacao : verificarConflitos [semConflito e maquinaDisponivel] / salvarReserva()
    Solicitada --> NaFila : verificarConflitos [temConflito e maquinaDisponivel] / adicionarNaFila()
    Solicitada --> Recusada : verificarDisponibilidade [maquinaIndisponivel] / publicarEvento()

    state NaFila {
        [*] --> Aguardando
        Aguardando --> Reordenada : reordenar [perfil == ADMIN e casoExcepcional] / registrarMotivo()
        Reordenada --> Aguardando : recalcular / atualizarPosicoes()
    }

    Aguardando : entry / calcularPosicao()
    Aguardando : do / monitorarFila()
    Aguardando : exit / liberarPosicao()

    NaFila --> PendenteAprovacao : promoverProximo [posicao == 1 e maquinaLivre] / atualizarStatus()
    NaFila --> Cancelada : cancelar [solicitanteDono] / removerDaFila()

    PendenteAprovacao : entry / notificarAdministrador()
    PendenteAprovacao --> Aprovada : aprovar [perfil == ADMIN] / notificarSolicitante()
    PendenteAprovacao --> Negada : negar [justificativa nao vazia] / registrarJustificativa()
    PendenteAprovacao --> Cancelada : cancelar [solicitanteDono] / liberarHorario()

    Aprovada --> EmUso : iniciarUso [horaAtual >= inicio] / registrarInicio()
    Aprovada --> Cancelada : cancelar [antesDoInicio] / liberarHorario()
    EmUso --> Concluida : finalizarUso / registrarConsumo()

    Recusada --> [*]
    Negada --> [*]
    Cancelada --> [*]
    Concluida --> [*]
```

## 6. Artefato 4 — Visão de Componentes (C4 Nível 3)

O código-fonte em C4-PlantUML está em `docs/architecture/workspace.puml`. Abaixo, a mesma visão em Mermaid, que o GitHub renderiza.

```mermaid
C4Component
    title Componentes - Fila de Prioridade (FabLab Manager)

    Person(solicitante, "Solicitante", "Usuário acadêmico ou de projeto industrial")
    Container(front, "Front-end Web", "React", "Telas de reserva e fila")

    Container_Boundary(api, "API Back-End (Node.js + Express)") {
        Component(ctrl, "ReservaController", "Express Router", "Recebe requisições HTTP")
        Component(rsvc, "ReservaService", "Classe JavaScript", "Orquestra o fluxo de reserva")
        Component(disp, "DisponibilidadeService", "Classe JavaScript", "Verifica disponibilidade")
        Component(fila, "FilaPrioridadeService", "Classe JavaScript", "Regra de prioridade e posições")
        Component(repo, "ReservaRepository", "Classe JavaScript", "Acesso aos dados")
        Component(pub, "NotificacaoPublisher", "amqplib", "Publica eventos")
    }

    ContainerDb(db, "Banco de Dados", "PostgreSQL 16", "Reservas e posições na fila")
    ContainerQueue(mq, "Mensageria", "RabbitMQ", "Eventos de notificação")
    System_Ext(notif, "Serviço de Notificação", "Avisa os envolvidos")

    Rel(solicitante, front, "Usa", "HTTPS")
    Rel(front, ctrl, "Chama", "REST/JSON sobre HTTPS")
    Rel(ctrl, rsvc, "criarReserva()", "síncrono")
    Rel(rsvc, disp, "verificarDisponibilidade()", "síncrono")
    Rel(rsvc, repo, "buscarConflitos(), salvar()", "síncrono")
    Rel(rsvc, fila, "adicionarNaFila()", "síncrono")
    Rel(fila, repo, "atualizarPosicao()", "síncrono")
    Rel(rsvc, pub, "publicarEvento()", "assíncrono")
    Rel(repo, db, "Lê e grava", "SQL/TCP")
    Rel(pub, mq, "Publica", "AMQP")
    Rel(mq, notif, "Entrega", "AMQP")
```

## 7. Artefato 5 — ADR

Texto completo em `docs/architecture/adr-0001.md`.

### ADR 0001 — Persistir a fila de prioridade no PostgreSQL com bloqueio de linha

**Status:** Proposto

## Contexto

Quando dois ou mais solicitantes pedem a mesma máquina no mesmo horário, o sistema precisa montar uma fila de prioridade e deixar cada solicitante consultar sua posição (RF03, RF04, RF05). O administrador pode reordenar a fila apenas em casos excepcionais (RF14).

Dois problemas técnicos aparecem:

1. Se a fila ficar só na memória da aplicação, ela se perde quando o servidor reinicia.
2. Se duas requisições chegarem ao mesmo tempo, a mesma posição pode ser entregue para duas reservas (condição de corrida), deixando a fila inconsistente.

## Decisão

- A fila será **persistida no PostgreSQL 16**, em uma coluna `posicao` na tabela de reservas, ordenada pela regra de prioridade.
- Ao entrar ou reordenar a fila, a operação roda em uma **transação** com `SELECT ... FOR UPDATE` nas reservas da mesma máquina e do mesmo período, para que só uma requisição altere a fila por vez.
- Os avisos ao solicitante (entrada na fila, promoção, aprovação) serão enviados de forma **assíncrona pelo RabbitMQ**, para a notificação não atrasar a resposta da reserva.

Motivo: o PostgreSQL já é o banco do sistema, então não precisa de tecnologia nova, e o bloqueio de linha garante consistência. O RabbitMQ separa o envio de notificações do fluxo principal.

## Consequências

**Vantagens**
- A fila sobrevive a reinicializações e fica auditável.
- Evita posições duplicadas e estados corrompidos da reserva.
- A resposta ao solicitante é rápida, porque a notificação é assíncrona.

**Débitos e riscos aceitos**
- O bloqueio de linha pode gerar espera quando muitas pessoas disputam a mesma máquina.
- Introduz mais uma peça de infraestrutura (RabbitMQ) para manter.
- Se o RabbitMQ estiver fora do ar, as notificações atrasam (a reserva em si continua funcionando).

## 8. Consistência entre os modelos

| Passo da sequência | Atividade correspondente | Estado / transição da Reserva |
|---|---|---|
| `verificarDisponibilidade` retorna indisponível | "Recusar solicitação" | `Solicitada → Recusada` |
| `buscarConflitos` sem conflito + `salvar` | "Criar reserva PENDENTE_APROVACAO" | `Solicitada → PendenteAprovacao` |
| `buscarConflitos` com conflito + `adicionarNaFila` | "Criar reserva EM_FILA" + Fork | `Solicitada → NaFila` |
| `atualizarPosicao` no `loop` | "Aplicar regra de prioridade" (ramo do Fork) | `entry / calcularPosicao()` |
| `publicarEvento` (assíncrono) | "Enviar notificação" (ramo do Fork) | efeito das transições |
| `promoverProximo` | "Aguardar promoção" | `NaFila → PendenteAprovacao` |

# Proposta de implementação — GitHub Contributor

**Recomendo um aplicativo local com React, Fastify e SQLite, controlando um worker Linux descartável na EC2, com OpenCode e Docker.** O processo terá os quatro botões definidos por você, execução monitorada e publicação iniciada manualmente.

O projeto será desenvolvido em torno de três responsabilidades:

- **Aplicação local:** descobrir issues, controlar as etapas, armazenar o histórico e publicar as contribuições.
- **Worker remoto:** preparar o projeto, executar o agente, testar e produzir os resultados.
- **Infraestrutura AWS:** disponibilizar o ambiente temporário e encerrá-lo dentro de um prazo definido.

## 1. Arquitetura proposta

```text
SEU COMPUTADOR
┌──────────────────────────────────────────────┐
│ React                                        │
│ Issues · Contribuições · Execução · Terminal  │
└──────────────────────┬───────────────────────┘
                       │ HTTP / SSE / WebSocket
┌──────────────────────▼───────────────────────┐
│ Fastify                                      │
│                                              │
│ • Controlador das etapas                     │
│ • OAuth OpenAI                               │
│ • Integração GitHub MCP                       │
│ • Integração AWS                             │
│ • Sincronização de eventos e resultados       │
└──────────┬───────────────────┬───────────────┘
           │                   │
    SQLite local        APIs AWS + túnel SSM
                               │
AWS                            ▼
┌──────────────────────────────────────────────┐
│ EC2 Linux                                    │
│                                              │
│ Worker supervisionado como serviço            │
│ ├── OpenCode + provider OpenAI                │
│ ├── Git + workspace do repositório            │
│ ├── Docker                                   │
│ └── Histórico temporário de eventos e arquivos│
└──────────────────────────────────────────────┘
```

**Somente o Fastify acessa o SQLite local.** O worker transmite eventos e resultados pelo túnel; ele não se conecta ao banco da sua máquina.

Minha proposta é iniciar com **uma contribuição em execução e uma EC2 por vez**, mantendo outras issues no estado `Escolhida`.

## 2. Os quatro botões

### 1 — Provisionar EC2 com setup do ambiente

**Disponível quando:** existe uma issue escolhida e as integrações necessárias estão configuradas.

A aplicação:

1. Verifica o perfil AWS, região, elegibilidade do tipo de instância e limites configurados.
2. Cria a instância com um identificador exclusivo da execução.
3. Configura disco, rede e perfil IAM.
4. Inicializa SSM Agent, Git, Docker, runtime e uma versão definida do OpenCode.
5. Inicia o serviço do worker.
6. Prepara a integração OpenAI do worker.
7. Verifica comunicação, ferramentas e espaço disponível.

**Concluído quando:** o worker responde ao health check e consegue executar uma chamada de ferramenta usando a integração de inferência.

“Instância ligada” e “ambiente pronto” serão estados diferentes.

### 2 — Download do projeto com instalação de dependências

**Disponível quando:** o ambiente está pronto.

O worker:

1. Confirma que a issue continua aberta e verifica PRs relacionados.
2. Identifica a branch-base pelas instruções do projeto.
3. Clona o repositório e registra o commit-base.
4. Lê `CONTRIBUTING.md`, instruções para agentes e configuração de CI.
5. Identifica versões de runtimes, comandos e serviços necessários.
6. Instala dependências, utilizando Docker quando adequado.
7. Executa as verificações iniciais.

**Concluído quando:** existe um workspace utilizável e um relatório das verificações iniciais.

Se o projeto depender de credenciais externas ou de recursos indisponíveis, a etapa fica **Bloqueada**, com o motivo. Falhas que já existiam no projeto ficam registradas para comparação posterior.

### 3 — Executar implementação da issue selecionada

**Disponível quando:** o projeto está preparado.

O OpenCode recebe a issue, as regras do repositório e o relatório inicial. Em seguida:

- investiga o comportamento;
- implementa a alteração;
- acrescenta ou ajusta testes pertinentes;
- executa as verificações;
- interpreta falhas e faz ajustes;
- revisa o diff;
- prepara os commits e o relatório final.

**Concluído quando:** os critérios da issue foram atendidos e existe evidência de validação suficiente.

A execução também pode terminar como **Bloqueada** ou **Falhou**. A declaração do modelo, isoladamente, não habilita a publicação.

O resultado inclui:

- diff e commits;
- comandos executados e resultados;
- verificações aprovadas, falhas e limitações;
- referência do código validado;
- pacote Git para preservar e transportar a contribuição.

### 4 — Abrir Merge Request

Na interface podemos usar **“Abrir Pull Request”**, o nome utilizado pelo GitHub.

**Disponível quando:** a implementação está pronta para publicação.

O Fastify:

1. Recupera os resultados e verifica sua integridade.
2. Confirma que o código a publicar corresponde ao validado.
3. Cria ou reutiliza o fork.
4. Publica a branch com os commits.
5. Preenche o template de PR do repositório.
6. Inclui a referência à issue e as evidências de teste.
7. Abre o PR pelo MCP do GitHub e registra a URL.

**Cliques repetidos ou uma falha de conexão não devem criar PRs duplicados.** A operação consulta a branch e os PRs existentes antes de repetir a publicação.

A integração efetiva do PR continua sendo uma decisão dos mantenedores.

## 3. Autenticação OpenAI e OpenCode

O método continuará sendo o do `../linkedin-post-publisher`:

- OAuth/OIDC com PKCE;
- consentimento de compartilhamento da cota;
- catálogo de modelos da conta;
- endpoint público Responses;
- renovação de tokens;
- armazenamento protegido.

**O ponto que precisa ser comprovado primeiro é o provider do OpenCode funcionar com esse fluxo específico.** Não vamos presumir que o login padrão de qualquer versão do OpenCode tenha comportamento idêntico.

A proposta é:

- uma sessão para a aplicação local;
- uma sessão separada para o worker, na mesma conta ChatGPT;
- autorização pelo navegador local;
- transferência protegida da sessão do worker pelo canal remoto;
- renovação dessa sessão sob responsabilidade exclusiva do worker.

Isso evita dois processos disputarem o mesmo refresh token. A cota continua sendo compartilhada pelo plano, sem uma franquia adicional por sessão.

O cofre DPAPI existente será reutilizado como referência no Windows. O armazenamento Linux terá implementação própria; copiar o banco cifrado do Windows para a EC2 não resolve essa parte.

Os MCPs serão conectados pelo harness/aplicação. A modalidade de inferência escolhida não oferece MCP hospedado pela própria Responses API.

## 4. Monitoramento e recuperação

Cada etapa será uma tarefa assíncrona. O clique inicia a operação e retorna um identificador; a página acompanha seu progresso.

```text
Issue #123 — Corrigir comportamento de paginação

✓ 1. Ambiente provisionado
✓ 2. Projeto preparado
● 3. Implementação em andamento
    ✓ Problema reproduzido
    ✓ Código alterado
    ● Executando testes
○ 4. Pull Request

[Logs] [Diff] [Terminal] [Cancelar execução]

EC2: em execução · Prazo restante: 48 min
```

O painel mostrará:

- etapa e atividade atual;
- início, duração e tentativas;
- comandos, saída e códigos de retorno;
- mensagens públicas de progresso do agente;
- uso de CPU, memória e disco;
- estado da EC2 e prazo restante.

**Não haverá porcentagens inventadas:** instalações sem progresso mensurável aparecerão como atividade em andamento.

Como os eventos do OpenCode são ao vivo, o worker terá um registro durável próprio, com sequência de eventos. O Fastify recuperará os eventos faltantes e eliminará duplicatas.

Fechar a página não interrompe o trabalho. Se o computador perder conexão, o worker poderá continuar, enquanto sua sessão e seu prazo de execução permanecerem válidos.

## 5. Terminal remoto e RDP

### Recomendação: incluir “Abrir terminal” na primeira versão

O terminal pode ficar dentro da interface React, utilizando um componente como xterm.js.

A comunicação passaria pelo Fastify e pelo túnel SSM até um terminal do worker. **Não precisamos expor a porta 22 nem instalar um servidor SSH para oferecer essa experiência.**

O botão permite inspecionar arquivos, processos, containers e executar comandos. Alterações manuais no código depois da validação exigem uma nova verificação antes da publicação.

### RDP: deixar como evolução opcional

É possível oferecer interface gráfica, mas isso muda o perfil da máquina:

- desktop e servidor remoto aumentam o setup;
- consomem mais memória e disco;
- Windows muda o sistema, a imagem e a composição de custos;
- Linux com desktop e xrdp também adiciona manutenção.

**Para o Free Tier, eu começaria com Linux sem desktop.** Quando uma issue exigir interação com navegador, podemos primeiro utilizar navegador headless e preservar screenshots e relatórios. RDP entraria depois, como um perfil específico de worker.

## 6. AWS Free Tier e ciclo de vida

A documentação atual distingue benefícios legados e o programa baseado em créditos para contas criadas a partir de **15 de julho de 2025**. Portanto, “conta Free Tier” não basta para definir uma configuração gratuita.

A consulta feita neste ambiente retornou ausência de credenciais AWS. Assim, o plano, os créditos e a elegibilidade da sua conta ainda não foram verificados.

Minha proposta inicial:

| Decisão | Proposta |
|---|---|
| Sistema | Linux, arquitetura x86-64 para maior compatibilidade inicial |
| Instância | Perfil elegível escolhido após verificar sua conta |
| Concorrência | Uma EC2 ativa |
| Acesso | SSM, sem portas de entrada públicas |
| Rede | Saída para GitHub, OpenAI, AWS e registros de pacotes |
| Disco | EBS dimensionado para o perfil, com exclusão no encerramento |
| Duração | Prazo máximo configurável por execução |
| Recursos adicionais | Evitar NAT Gateway, balanceador e Elastic IP no perfil inicial |

Uma instância `micro` tem pouca memória para combinar OpenCode, Docker e builds. Ela pode atender tarefas pequenas, mas **não deve ser considerada suficiente para qualquer repositório**. A seleção de issues também avaliará o custo de preparar e testar o projeto.

Se utilizarmos instâncias T com créditos de CPU, prefiro inicialmente o modo `standard`: evita cobrança de créditos excedentes, aceitando redução de desempenho.

### Preservação e descarte

- Após sincronizar os resultados, a aplicação encerra a EC2.
- Enquanto você examina o diff, a máquina pode ser liberada.
- Para sobreviver a uma desconexão até o prazo máximo, proponho um pequeno arquivo de recuperação no S3, com expiração automática e sem credenciais.
- O encerramento terá agendamento na AWS, independente do computador local.
- Ao reiniciar, a aplicação reconcilia os recursos conhecidos e identifica pendências de limpeza.

EC2, EBS, IPv4 público, armazenamento e transferências precisam entrar na verificação de benefícios. **A aplicação pode limitar seu próprio uso; não pode garantir custo zero para toda a conta AWS.**

## 7. Banco e organização do aplicativo

**SQLite local**, com migrations versionadas:

| Registro | Conteúdo |
|---|---|
| Contribuições | Issue, repositório, branch-base e situação |
| Execuções das etapas | Tentativas, horários, resultado e erro |
| Workers | Instância, região, estado e expiração |
| Eventos | Histórico ordenado e cursor de sincronização |
| Artefatos | Diff, pacote Git, relatórios e hashes |
| Pull Requests | URL, branch, revisão e resultado |
| Integrações | Configurações e credenciais protegidas |

O estado da contribuição ficará separado do estado da infraestrutura: um PR pode estar em revisão mesmo depois de a EC2 ter sido encerrada.

As telas principais serão **Descoberta**, **Contribuições**, **Execução** e **Configurações**. A descoberta apresenta até cinco candidatas qualificadas, considerando facilidade, atividade dos mantenedores, instruções de contribuição e requisitos de execução.

Para a cadência de três dias, a primeira versão pode oferecer lembrete e atualização das candidatas quando o app estiver aberto, preservando sua escolha manual.

## 8. Fases de entrega

| Fase | Entrega e critério de conclusão |
|---|---|
| **1. Provas de viabilidade** | OpenCode remoto usando o OAuth definido, executando ferramenta e teste; SSM funcional; preservação de resultados comprovada |
| **2. Aplicação local** | React, Fastify, SQLite, integrações, descoberta e registro de issues escolhidas |
| **3. Etapas 1 e 2** | Provisionamento, setup e preparação do projeto com logs, tentativas e recuperação |
| **4. Etapa 3** | Sessões do agente, validação, diff, artefatos e terminal |
| **5. Etapa 4** | Publicação pelo GitHub MCP, prevenção de duplicatas e acompanhamento do PR |
| **6. Ciclo completo** | Teste em repositório controlado, incluindo desconexão, reinício, falha e descarte |

Os testes de integração focarão as transições de estado, OAuth, reconexão de eventos, provisionamento parcialmente concluído, repetição de publicação e correspondência entre código validado e publicado.

**O primeiro marco será um ciclo pequeno de ponta a ponta**, antes de ampliar a variedade de repositórios suportados.

---

Essa é a proposta que recomendo para aprovação: **quatro etapas manuais, worker Linux com OpenCode, terminal via SSM e histórico local recuperável**.

**Para fechar o perfil de infraestrutura: quando sua conta AWS foi criada e, atualmente, ela está no Free account plan ou no Paid account plan com benefícios/créditos do Free Tier?**

### Referências

- [Benefícios EC2 conforme a data da conta](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html)
- [Planos AWS Free Tier](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [SSM e encaminhamento de portas](https://repost.aws/knowledge-center/systems-manager-ssm-port-forwarding)
- [OpenAI — sessões em VMs próprias](https://developers.openai.com/siwc/token-sharing-open-source/self-hosted-vms)
- [OpenCode — cliente e eventos](https://opencode.ai/docs/build/client/)

**Nota de integração:** o antigo AWS API MCP Server está sendo substituído pelo AWS MCP Server oficial; adotaremos essa direção conforme o [guia de migração](https://github.com/awslabs/mcp/blob/main/src/aws-api-mcp-server/MIGRATION.md).

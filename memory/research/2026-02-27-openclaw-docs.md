# Pesquisa: Documentação OpenClaw (docs.openclaw.ai)
**Data:** 2026-02-27
**Requisitante:** Lincoln (para uso do Arquiteto, Oráculo e PostMaster)
**Objetivo:** Scraping/síntese da documentação local em `/home/lincoln/.npm-global/lib/node_modules/openclaw/docs` para entendimento profundo da arquitetura, agentes, multi-agentes e ferramentas disponíveis.

## Índice
1. [Arquitetura Geral (Gateway)](#arquitetura-geral-gateway)
2. [Multi-Agentes (Isolamento e Roteamento)](#multi-agentes-isolamento-e-roteamento)
3. [Agent Runtime (pi-mono embutido)](#agent-runtime-pi-mono-embutido)
4. [Gestão de Sessões](#gestão-de-sessões)
5. [Ferramentas de Sessão (Inter-agentes e Sub-agentes)](#ferramentas-de-sessão-inter-agentes-e-sub-agentes)

---

## 1. Arquitetura Geral (Gateway)
- Um único processo longo (Gateway) gerencia todas as conexões com provedores (WhatsApp, Telegram, Discord, etc.).
- Clientes (CLI, UI, automações) e Nodes (dispositivos físicos) se conectam ao Gateway via **WebSocket** (porta 18789).
- Apenas o Gateway abre sessão (ex: WhatsApp Baileys).
- Autenticação de conexões exige assinatura de `challenge` (nonce) e pareamento por dispositivo (`role: node`).

## 2. Multi-Agentes (Isolamento e Roteamento)
- O Gateway pode rodar múltiplos agentes lado a lado.
- Cada agente possui:
  - **Workspace:** diretório de trabalho com seus próprios `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `USER.md`.
  - **State Directory (`agentDir`):** onde ficam os perfis de autenticação e registro de modelos.
  - **Session Store:** histórico de chat e estado de roteamento.
- **Roteamento (Bindings):** Mensagens chegam de canais e são direcionadas a um agente via regras determinísticas:
  1. Match de `peer` (ID exato de DM/grupo)
  2. `parentPeer` (threads)
  3. `guildId + roles` / `teamId` (Discord/Slack)
  4. `accountId` (ex: WhatsApp pessoal vs trabalho)
  5. Fallback para o agente default (geralmente `main`).
- Cada agente pode ter restrições de ferramentas e sandboxing próprios (`agents.list[].tools.allow / deny`, `sandbox.mode`).

## 3. Agent Runtime (pi-mono embutido)
- OpenClaw usa um runtime derivado do `pi-mono`, mas gerencia as sessões e injeção de ferramentas.
- O **Workspace** atua como o único `cwd` (current working directory) permitido.
- Arquivos de injeção automática no contexto inicial da sessão:
  - `AGENTS.md` (instruções operacionais)
  - `SOUL.md` (persona, tom, limites)
  - `TOOLS.md` (notas de uso de ferramentas pelo usuário)
  - `USER.md` (perfil do usuário)
- Ferramentas nativas (`read`, `write`, `exec`, `edit`) e **Skills** (carregadas de `~/.openclaw/skills` ou `<workspace>/skills`).
- O runtime não usa as configurações legadas do `~/.pi/agent`.

## 4. Gestão de Sessões
- Sessões DMs (Direct Messages) colapsam para `agent:<agentId>:<mainKey>` por padrão (continuidade).
- É recomendado configurar `session.dmScope: "per-channel-peer"` em ambientes com múltiplos usuários para não vazar contexto entre pessoas diferentes.
- O Gateway é a fonte da verdade para o estado da sessão (tokens, atualizações).
- Históricos são salvos em `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`.
- Há rotinas automáticas de compactação (`pruneAfter`, `maxEntries`, etc.).

## 5. Ferramentas de Sessão (Inter-agentes e Sub-agentes)
Existem ferramentas específicas para a coordenação entre agentes e delegação:
- **`sessions_list`**: Lista sessões ativas (filtra por kind, tempo, limite de mensagens).
- **`sessions_history`**: Busca o histórico (JSONL) de uma sessão específica.
- **`sessions_send`**: Envia mensagem de um agente para outro (`agentToAgent`). Suporta timeout e loop de ping-pong de respostas.
- **`sessions_spawn`**: Cria um sub-agente isolado (`subagents`) para tarefas específicas. Pode sobrescrever o modelo, nível de pensamento e usar timeout de execução. Retorna o resultado automaticamente no canal de origem ao finalizar (announce step).


## 6. Resumo e Propósito (Index)
- O OpenClaw atua como um **gateway multi-canal self-hosted** para agentes de IA (WhatsApp, Telegram, Discord, iMessage, WebChat).
- O arquivo de configuração principal fica em `~/.openclaw/openclaw.json`.
- A interface de controle web nativa pode ser acessada em `http://127.0.0.1:18789/` (ou remotamente via Tailscale/SSH).
- **Segurança recomendada:** Configurar `channels.whatsapp.allowFrom` e regras de menção em grupos.

## 7. Estrutura de Arquivos (Workspace e Memória)
O Workspace (`~/.openclaw/workspace` ou per-agent) é o ambiente de memória e trabalho de um agente. 
- Ele **NÃO** guarda configurações secretas ou histórico de sessões (isso vai em `~/.openclaw/`).
- O Workspace é injetado no início de uma nova sessão e deve ser tratado como a memória privada do agente.

**Mapeamento de arquivos:**
- `AGENTS.md`: Instruções operacionais e regras.
- `SOUL.md`: Persona, tom e limites de atuação.
- `USER.md`: Detalhes sobre o usuário que requisita as tarefas.
- `IDENTITY.md`: Nome do agente, emoji e vibe.
- `TOOLS.md`: Notas de convenção de ferramentas (não restringe acesso, apenas orienta uso).
- `HEARTBEAT.md`: Checklist para execuções engatilhadas por heartbeat (ping de disponibilidade).
- `memory/YYYY-MM-DD.md`: Log diário de memória.
- `MEMORY.md`: Memória de longo prazo curada.
- `skills/`: Diretório de habilidades específicas do workspace (sobrepõe as habilidades padrão se houver conflito de nome).

_É recomendado manter o Workspace sob controle de versão Git provado (Private Repo), excluindo chaves e tokens (que devem residir fora dele)._

## 8. Sandboxing e Restrições de Ferramentas (Por Agente)
A partir da versão v2026.1.6, o OpenClaw suporta perfis de ferramentas e sandboxing granulares por agente, além da política global.

**Modos de Sandboxing (`agents.list[].sandbox.mode`):**
- `off`: Roda nativamente no host (acesso total).
- `all`: Sempre conteinerizado via Docker.
- `non-main`: Apenas as sessões secundárias (como grupos ou threads) são sandboxed, enquanto a DM principal roda no host.

**Restrição de Ferramentas:**
As políticas são aplicadas em cascata (não podem conceder de volta o que foi negado em níveis superiores):
1. Perfil global (`tools.profile`).
2. Política global (`tools.allow` / `tools.deny`).
3. Política do agente (`agents.list[].tools.allow / deny`).
4. Política do sandbox.

**Grupos de ferramentas úteis (shorthands):**
- `group:fs` (read, write, edit, apply_patch)
- `group:runtime` (exec, bash, process)
- `group:sessions` (ferramentas de controle interagentes)

_Nota de segurança:_ `tools.elevated` é global. Se você não confia em um agente público/secundário, é recomendado adicionar `"exec"` à lista `deny` específica desse agente.

## 9. Habilidades (Skills)
O sistema de Skills do OpenClaw é compatível com o formato **AgentSkills** (cada skill é um diretório contendo um arquivo `SKILL.md`).
- **Ordem de precedência (do maior pro menor):** `<workspace>/skills` → `~/.openclaw/skills` (local/managed) → Habilidades embutidas (bundled).
- Em cenários multi-agente, habilidades compartilhadas vão para `~/.openclaw/skills`, enquanto específicas vão para a pasta de workspace do agente.
- A **ClawHub** (`https://clawhub.com`) atua como registro público e CLI para gerenciar habilidades.
- Os metadados de uma skill (`metadata.openclaw`) podem exigir presença de binários no host (`requires.bins`), variáveis de ambiente e/ou rodar scripts de instalação.
- **Segurança:** O carregamento de variáveis como `env` ou `apiKey` em `skills.entries` injeta o segredo diretamente no processo host, não no sandbox.

## 10. Fila de Comandos (Command Queue)
A execução de respostas automatizadas (auto-reply) ocorre em série (por sessão) para prevenir colisões de estado e de LLM:
- **Modos (`messages.queue.mode`):**
  - `collect` (padrão): Agrupa todas as mensagens acumuladas em um único turno.
  - `steer`: Injeta a mensagem na execução atual, cancelando chamadas pendentes de ferramentas do assistente.
  - `followup`: Coloca na fila para um próximo turno após a execução atual terminar.
  - `interrupt`: Aborta a execução atual da sessão e roda a mensagem mais nova.
- `debounceMs`: atrasa o disparo até que a digitação/envio silencie por um tempo curto (padrão: 1000ms), muito útil para quem envia várias mensagens curtas.

---

## 11. Tailscale e Acesso Remoto
O Gateway tem suporte integrado nativo ao Tailscale para exposição da Web UI/WebSocket de forma segura sem expor o IP público ou vazar portas:
- **`tailscale.mode: "serve"`**: Expõe via HTTPS interno (Tailnet). Aceita autenticação sem token (usa o header de identidade do Tailscale para validação de segurança se `allowTailscale` for true).
- **`tailscale.mode: "funnel"`**: Expõe publicamente na web. Requer forçosamente autenticação do tipo `password` por razões óbvias de segurança.
- É possível definir `gateway.bind: "tailnet"` para fazer o `bind` direto do gateway na interface IP do tailscale, em vez de usar os proxies de Serve/Funnel.

---

## 12. Janela de Contexto e Compactação
Sessões longas geram grande histórico de tokens. O OpenClaw usa um processo de **compactação automática** para evitar exceder o limite do LLM.
- **Como funciona:** O histórico antigo é sumarizado e persistido no arquivo `.jsonl`, substituindo as mensagens originais passadas. As mensagens recentes são mantidas intactas.
- A compactação pode ser acionada manualmente via comando de chat `/compact [instruções]`.
- Antes da compactação, o OpenClaw pode realizar um **silent memory flush** para persistir as memórias no workspace.
- *Nota:* "Compactation" (sumariza histórico no jsonl) é diferente de "Session Pruning" (corta o excesso de respostas de ferramentas apenas em memória, por requisição).

---

## 13. Heartbeats
O Heartbeat permite que os agentes rodem verificações periódicas sem a necessidade de um gatilho do usuário.
- O prompt enviado no heartbeat orienta o agente a procurar por tarefas pendentes ou alertas relevantes.
- O agente responde com `HEARTBEAT_OK` se não houver nada relevante (o Gateway descarta a mensagem silenciosamente). Se algo precisar de atenção, o agente responde com o alerta.
- É guiado pelo arquivo `HEARTBEAT.md` no workspace do agente. Se o arquivo estiver vazio (somente com `# títulos`), a execução do heartbeat é pulada para economizar tokens.
- O `target` padrão do heartbeat é `none` (só roda internamente). Pode ser mudado para `last` (envia alerta para o último canal que o usuário usou).
- Pode ser restrito a horas ativas usando `activeHours`.

---

## 14. Cron vs Heartbeat
O OpenClaw oferece duas maneiras principais de automatizar e agendar tarefas:
- **Heartbeat:** Roda intervalos regulares (ex: 30min) dentro da *Sessão Principal*. Ideal para *verificações periódicas em lote* (checar emails, monitorar calendário) onde o contexto conversacional importa. O arquivo `HEARTBEAT.md` serve de checklist.
- **Cron:** Roda em horários *exatos* definidos por expressões cron, geralmente em *Sessões Isoladas*. Ideal para tarefas precisas, agendamentos pontuais (one-shot reminders) ou tarefas que necessitam de um modelo diferente sem poluir a memória da sessão principal.

## 15. Cron Jobs e Agendamentos
O Gateway gerencia jobs cron internamente sem precisar de schedulers do SO.
- Permite execuções `at` (uma vez/pontual), `every` (intervalos fixos) ou `cron` (expressões cron).
- **Tipos de execução:**
  - **Sessão Principal:** (`sessionTarget: "main"`) injeta um evento de sistema na fila do agente e engatilha um _heartbeat_ para ele lidar com o evento de forma conversacional.
  - **Sessão Isolada:** (`sessionTarget: "isolated"`) cria um sub-agente rápido em uma nova sessão sem poluir a DMs do usuário, ideal para rotinas pesadas (que podem inclusive ter o *Modelo de LLM substituído* só para aquele job).
- **Entrega de resultados:** Pode ser configurado para entregar o resumo de um job num canal (WhatsApp/Telegram) ou acionar um `webhook` com JSON.

---

## 16. Memória (`MEMORY.md` e `memory/`)
A memória no OpenClaw é sempre persistida em arquivos Markdown legíveis por humanos:
- `MEMORY.md`: Curadoria de longo prazo (decisões, preferências). Só é carregado na DM principal, não em grupos.
- `memory/YYYY-MM-DD.md`: Notas diárias e contexto de execução daquele dia.
- **Ferramentas:** `memory_search` (busca semântica local baseada em vetores) e `memory_get` (leitura exata).
- **Auto-flush:** Antes de uma sessão atingir o limite de contexto e ser compactada, o Gateway engatilha um "turno invisível" de prompt avisando o agente para persistir qualquer memória pendente nos arquivos.
- A busca semântica suporta **Hybrid Search** (Vetorial + Keyword BM25) com re-ranking MMR (para diversidade) e Temporal Decay (peso maior para memórias recentes).

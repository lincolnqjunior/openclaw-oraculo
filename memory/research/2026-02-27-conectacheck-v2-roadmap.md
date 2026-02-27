# Roadmap Técnico: ConectaCheck v2.1
**Data de recebimento:** 2026-02-27
**Autor:** Esteves Marques (Analista Fullstack Sênior — Living Consultoria)
**Propósito:** Proposta para fechar contrato com BAT (British American Tobacco)
**PDF original:** `memory/research/2026-02-27-conectacheck-v2-roadmap.pdf`
**Indexado por:** Arquiteto (a pedido do Lincoln)

---

## Proposta de Valor

| Pilar | Descrição |
|-------|-----------|
| 🚀 Time-to-Market | Integração de novos bureaus: de semanas de dev para dias de configuração No-Code |
| 🛡 Governança Centralizada | Hub único auditável — elimina regras espalhadas em planilhas e sistemas opacos |
| 🌍 Escala Regional | Multi-tenant, multi-país, hierarquias locais de aprovação (GTV/Gerentes), compliance regional |

---

## Arquitetura — 4 Microserviços Azure

### MS-01: Core API (Gateway)
- **Recurso:** Azure App Service (Python/FastAPI)
- **Função:** Ponto único de entrada — Auth (Entra ID + API Keys), Throttling, Validação de Schema
- **Segurança:** VNet Injection, WAF frontal
- **Escalabilidade:** Horizontal (Autoscale CPU/Memória)

### MS-02: Decision Orchestrator
- **Recurso:** Azure Durable Functions (Python)
- **Função:** State Machine — gerencia fluxos assíncronos e aprovações humanas (Long-Running Processes)
- **Padrão:** Fan-out/Fan-in para consulta paralela de múltiplos bureaus
- **Batch:** Processa listas grandes sem timeout

### MS-03: Integration Hub
- **Recurso:** Azure Functions (Isolated Worker)
- **Função:** Executor de I/O — recebe Template de Configuração + Payload, executa chamada HTTP/SQL
- **Isolamento:** Falhas em bureaus externos não travam o Core
- **Outbound:** IP fixo via NAT Gateway para allow-list em parceiros

### Data Layer
- **Recursos:** Azure Cosmos DB + Storage
- **Coleção Rules:** Regras de negócio e templates Jinja2 (JSON) — configuráveis via UI
- **Coleção Audit:** Log imutável de cada decisão (Input + Output Bureau + Decisão final)
- **Hierarquias:** Dados relacionais de Gerentes/GTVs

---

## Pipeline de Decisão (5 Etapas)

```
1. Recepção       → Validação de Schema + Autenticação Tenant
         ↓
2. Enriquecimento → Chamadas Paralelas APIs Externas (bureaus)
         ↓
3. Normalização   → Mapeamento Jinja2: JSON Raw → Modelo Canônico
         ↓
4. Scoring        → Motor de Inferência (Categórico ou Score com pesos/faixas)
         ↓
5. Decisão Final  → Aprovado / Reprovado / Maybe + Persistência Auditoria
```

---

## Infraestrutura Azure Completa

**Compute:**
- Azure App Service (Plan Premium) — API + Workers
- Azure Functions (Python) — Serverless para picos

**Dados:**
- Azure Cosmos DB (NoSQL) — alta performance, replicação global
- Azure Service Bus — filas desacopladas, garantia de entrega

**Segurança:**
- Azure Key Vault — gestão de chaves e credenciais
- Private Link — tráfego entre App Service, Cosmos DB e Key Vault não sai para internet pública
- VNet Integration — isolamento de rede entre componentes

---

## Cronograma (Poker Planning)

| Fase | Entregáveis | Horas | Sprints |
|------|-------------|-------|---------|
| Fase 1 — Fundação | Setup Azure (App Service, CosmosDB, KeyVault), Auth Middleware (Entra ID + API Key), Clean Arch base, CI/CD GitHub Actions | 40h | 1 sprint |
| Fase 2 — Entidades | Módulo Tenants/Configurações Regionais, Hierarquias GTV/Gerentes, Usuários + RBAC, APIs BFF | 48h | 1,5 sprint |
| Fase 3 — Integration Studio | Engine HTTP Genérica, Parser Jinja2/Liquid, UI de Cadastro de Fontes, UI Dry Run (teste de conector) | 64h | 2 sprints |
| Fase 4 — Motor de Decisão | Classificação Categórica (If/Else dinâmico), Score (Pesos/Faixas), Auditoria CosmosDB, Pipeline Runner | 64h | 2 sprints |
| **TOTAL** | | **216h** | **~6,5 sprints** |

---

## Stack Tecnológica

- **Linguagem:** Python
- **Framework API:** FastAPI
- **Cloud:** Azure (App Service, Durable Functions, Cosmos DB, Service Bus, Key Vault)
- **Template Engine:** Jinja2 / Liquid
- **Auth:** Azure Entra ID + API Keys
- **CI/CD:** GitHub Actions
- **Padrão Arquitetural:** Microserviços, Clean Architecture, Event-driven

---

## Análise Inicial — Pontos Fortes

- Arquitetura bem segmentada — cada MS tem responsabilidade clara
- Uso adequado de Durable Functions para Long-Running Processes (aprovações humanas)
- Isolamento do Integration Hub protege o core de falhas externas
- Audit log imutável é diferencial para compliance (crédito/regulação)
- IP fixo via NAT Gateway resolve problema real com parceiros que exigem allow-list

## Pontos a Investigar / Riscos Potenciais

- **Cosmos DB como solução relacional (hierarquias):** O doc menciona "dados relacionais" no Cosmos — pode ser um anti-pattern dependendo da complexidade das hierarquias. Avaliar se SQL Azure seria mais adequado.
- **Estimativa de 216h:** Parece conservadora para uma arquitetura dessa complexidade, especialmente a Fase 3 (Integration Studio com UI). Risco de scope creep.
- **Curva de aprendizado Durable Functions:** Exige expertise específica; se o time não tiver, pode impactar cronograma.
- **Sem menção a testes:** Nenhuma fase inclui testes de carga, testes de integração ou QA. Ponto de atenção para a proposta.
- **SLA / Disponibilidade:** Não mencionado no roadmap — importante para proposta comercial com empresa do porte da BAT.

---

## Status
- [ ] Análise de riscos detalhada (se Lincoln solicitar)
- [ ] Comparação com benchmarks de mercado para estimativa de horas
- [ ] Revisão da proposta comercial antes de apresentar à BAT

# TOOLS.md - Oráculo — Ferramentas e Configuração

---

## Pesquisa Web

### Tavily Search (fonte primária para pesquisa)
- **Skill:** `~/.openclaw/workspaces/oraculo/skills/` — usar via script Python
- **Venv:** `~/.tavily-env/bin/python`
- **Uso direto:**
```bash
~/.tavily-env/bin/python -c "
from tavily import TavilyClient
import os
client = TavilyClient(api_key=os.environ['TAVILY_API_KEY'])
result = client.search('<query>', search_depth='advanced', max_results=10)
for r in result['results']:
    print(r['url'], '|', r['title'])
    print(r['content'][:300])
    print()
"
```
- Usar `search_depth='advanced'` para pesquisas importantes
- Tavily retorna conteúdo extraído — mais rico que web_search simples

### web_search (fallback / breadth-first)
- Brave API — até 10 resultados por query
- Usar quando Tavily não for suficiente ou para variedade de fontes

### web_fetch (extração de conteúdo)
- Extrai conteúdo de URLs em markdown/texto limpo
- Usar para ler páginas específicas após identificar via Tavily/web_search
- `maxChars`: ajustar conforme necessidade (default razoável: 10000)

---

## Documentação de Bibliotecas (Context7 MCP)

**Usar SEMPRE como fonte primária para documentação de qualquer biblioteca.**

- **Skill:** `skills/context7/`
- **MCP config:** `/home/lincoln/.openclaw/workspace/config/mcporter.json`
- **Uso via mcporter:**
```bash
cd /home/lincoln/.openclaw/workspace
mcporter call context7.resolve-library-id libraryName="<nome>"
mcporter call context7.get-library-docs context7CompatibleLibraryId="<id>" topic="<tópico>" tokens=10000
```

**Fluxo padrão:**
1. `resolve-library-id` — obtém o ID da biblioteca
2. `get-library-docs` — obtém documentação atualizada do tópico

Preferir Context7 sobre Stack Overflow, Medium ou artigos de terceiros para documentação técnica.

---

## Browser Automatizado (Playwright)

**Playwright 1.58.2 disponível via npx.**

- Usar para páginas que exigem JavaScript, login, ou interação
- Ferramenta `browser` com `profile="openclaw"` para sessão isolada
- `profile="chrome"` para usar o Chrome do Lincoln (extensão relay)

**Quando usar:**
- Páginas que bloqueiam web_fetch (JS-heavy, SPAs)
- Webscrapping que requer interação (formulários, cliques, scroll)
- Captura de screenshots para documentação

**Quando NÃO usar:**
- Páginas estáticas simples → preferir web_fetch
- APIs públicas → preferir requisição direta

---

## Memory Search

- **Provider:** openai (`text-embedding-3-small`)
- **Mode:** hybrid (BM25 0.3 + vector 0.7), MMR λ=0.7, temporal decay halfLife=30d
- **Cache:** 50.000 entradas
- **Índice:** `~/.openclaw/memory/main.sqlite`

Ferramentas disponíveis:
- `memory_search("query")` — busca semântica nas notas
- `memory_get("memory/arquivo.md")` — leitura direta de arquivo

---

## Arquivos e Execução

- `read`, `write`, `edit` — ler e escrever documentos de resultado
- `exec` — executar scripts quando necessário

### Python / uv
- pip/pip3 **NÃO** disponíveis — usar `uv`
- uv path: `/home/linuxbrew/.linuxbrew/bin/uv`

---

## Padrão de Pesquisa Profunda

1. **Mínimo 3 fontes independentes** para qualquer afirmação importante
2. Sempre verificar **data de publicação** das fontes
3. Para dados numéricos: buscar **fonte primária** (não só notícia sobre o dado)
4. Registrar fontes com **URL + data de acesso** no documento final
5. Reportar **nível de confiança** explicitamente (alto / médio / baixo)

---

## Onde Salvar Resultados

```
memory/
├── YYYY-MM-DD.md              # daily notes (o que foi feito hoje)
├── research/
│   └── YYYY-MM-DD-<tema>.md   # pesquisas completas com fontes
├── factcheck/
│   └── YYYY-MM-DD-<tema>.md   # verificações de fatos
└── plans/
    └── YYYY-MM-DD-<tema>.md   # planos e planejamentos
```

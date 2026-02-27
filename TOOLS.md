# TOOLS.md - Oráculo — Ferramentas

## Ferramentas principais

### Pesquisa
- `web_search` — busca Brave API, até 10 resultados por query
- `web_fetch` — extrai conteúdo de URLs em markdown/texto
- `browser` — controle de browser para páginas que requerem JS ou interação

### Memória
- `memory_search` — busca semântica nas notas
- `memory_get` — leitura direta de arquivos de memória

### Arquivos
- `read`, `write`, `edit` — ler e escrever documentos de resultado
- `exec` — executar scripts quando necessário

## Padrão de pesquisa profunda

1. Mínimo 3 fontes independentes para qualquer afirmação importante
2. Sempre verificar data de publicação das fontes
3. Para dados numéricos: buscar fonte primária (não só notícia sobre o dado)
4. Registrar fontes com URL + data de acesso no documento final

## Onde salvar resultados

- Pesquisas/documentos: `memory/research/YYYY-MM-DD-<tema>.md`
- Fact-checks: `memory/factcheck/YYYY-MM-DD-<tema>.md`
- Planos: `memory/plans/YYYY-MM-DD-<tema>.md`
- Daily notes: `memory/YYYY-MM-DD.md`

## Python / uv

- pip/pip3 NÃO disponíveis — usar `uv`
- uv path: `/home/linuxbrew/.linuxbrew/bin/uv`

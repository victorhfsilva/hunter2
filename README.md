# Hunter Workflow (Rayder)

Pipeline de bug bounty automatizado orquestrado pelo [Rayder](https://github.com/devanshbatham/rayder).
Executa ferramentas de caça de segredos e chaves de API no escopo e, por fim,
dispara o agente `opencode` com o comando `/hunt` (definido em `hunt.md`) para
fazer recon + testes de vulnerabilidades domínio por domínio.

## Execução

```bash
rayder -w workflow.yaml
```

## Pré-requisitos

- **Rayder** instalado e no `PATH`
- **Python 3** com as ferramentas:
  - `/opt/secret-hunter/secret-hunter.py`
  - `/opt/postman-hunter/postman-hunter.py`
- **opencode** no `PATH`, com o comando `/hunt` configurado (este repositório — `hunt.md`)

## Provider de LLM (OpenRouter ou Verboo)

O workflow usa **Verboo** por padrão. Para trocar de provider, altere os flags
`--provider` / `--api-key` / `--base-url` / `--model` nos módulos `secret_hunter`
e `postman_hunter` do `workflow.yaml`.

| Provider | Flags | API key (variável) |
|---|---|---|
| **Verboo** (padrão) | `--provider verboo --base-url https://code.verboo.ai/router/v1 --model glm-5.3-flash` | `VERBOO_KEY` |
| **OpenRouter** | `--provider openrouter` | `OPEN_ROUTER_KEY` |

Exemplo de módulo com Verboo:

```yaml
- name: secret_hunter
  cmds:
  - python3 {{TOOLS_FOLDER}}/secret-hunter/secret-hunter.py --domains-file {{SCOPE_FOLDER}}/all_scope_domains_new.txt --provider verboo --api-key {{VERBOO_KEY}} --base-url https://code.verboo.ai/router/v1 --model glm-5.3-flash --depth 3 -o {{OUTPUT_FOLDER}}/$(date +%Y-%m-%d)/secret-hunter
```

Exemplo de módulo com OpenRouter:

```yaml
- name: secret_hunter
  cmds:
  - python3 {{TOOLS_FOLDER}}/secret-hunter/secret-hunter.py --domains-file {{SCOPE_FOLDER}}/all_scope_domains_new.txt --provider openrouter --api-key {{OPEN_ROUTER_KEY}} --depth 3 -o {{OUTPUT_FOLDER}}/$(date +%Y-%m-%d)/secret-hunter
```

## Variáveis de ambiente necessárias

| Variável | Uso |
|---|---|
| `VERBOO_KEY` | API key do Verboo (provider padrão, usada por secret-hunter e postman-hunter) |
| `OPEN_ROUTER_KEY` | API key do OpenRouter (alternativa ao Verboo) |
| `POSTMAN_KEY` | API key do Postman (usada por postman-hunter) |

Exporte apenas as que o provider escolhido exige, antes de rodar:

```bash
# Provider padrão (Verboo)
export VERBOO_KEY="..."
export POSTMAN_KEY="..."

# Alternativa (OpenRouter)
export OPEN_ROUTER_KEY="..."
export POSTMAN_KEY="..."
```

## Variáveis do workflow

Definidas no topo do `workflow.yaml`:

| Variável | Valor padrão | Descrição |
|---|---|---|
| `TOOLS_FOLDER` | `/opt` | Raiz onde as ferramentas estão instaladas |
| `OUTPUT_FOLDER` | `/output/workflow` | Raiz dos resultados |
| `SCOPE_FOLDER` | `/opt/scope-scraper/data/scopes/` | Onde vive o arquivo de escopo |

## Arquivo de escopo

O pipeline lê `{{SCOPE_FOLDER}}/all_scope_domains_new.txt` — um domínio por linha:

```
example.com
target.io
...
```

## Módulos (execução sequencial)

1. **dir_creation** — cria `{{OUTPUT_FOLDER}}/AAAA-MM-DD` (resultados ficam por dia).
2. **secret_hunter** — roda o secret-hunter no arquivo de domínios (profundidade 3,
   via Verboo), gravando em `<OUTPUT_FOLDER>/<data>/secret-hunter/`.
3. **postman_hunter** — roda o postman-hunter nos mesmos domínios (via Verboo),
   combinando o escopo com coleções do Postman, gravando em
   `<OUTPUT_FOLDER>/<data>/postman-hunter/`.
4. **opencode_test** — para cada domínio do escopo, executa
   `opencode run --auto --command hunt -- "https://<dominio>" --out <OUTPUT_FOLDER>/<data>/opencode/`,
   fazendo recon + caça de vulnerabilidades por domínio conforme o prompt `hunt.md`.

## Saídas

```
{{OUTPUT_FOLDER}}/
└── 2026-10-29/
    ├── secret-hunter/
    ├── postman-hunter/
    └── opencode/
```

## Ajustes comuns

- Trocar a data do output: a data é gerada em tempo de execução (`$(date +%Y-%m-%d)`);
  para um diretório fixo, substitua `$(date +%Y-%m-%d)` no `workflow.yaml`.
- Paralelizar: mude `parallel: false` para `true` no `workflow.yaml`
  (atenção: o módulo `opencode_test` já é pesado; paralelizar pode estourar rate limits da LLM).
- Trocar o provider de LLM: veja a seção **Provider de LLM (OpenRouter ou Verboo)**
  acima e ajuste `--provider` / `--base-url` / `--model` em `secret_hunter` e
  `postman_hunter`.

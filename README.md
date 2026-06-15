
# Planilha Automática — Mapa Classe Sondagens

GitHub Actions que gera diariamente um arquivo Excel com os 9 datasets do dashboard
**Mapa Classe — Monitoramento Sondagens** e o publica via GitHub Pages num link permanente.

O stakeholder salva o link uma vez e sempre baixa os dados mais recentes.

---

## Como funciona

```
[GitHub Actions — cron periódico]
        |
        v
[Autentica no Databricks via OAuth refresh token]
        |
        v
[Executa 9 queries via SQL Statements API]
        |
        v
[Gera mapa_classe_sondagens.xlsx com openpyxl]
        |
        v
[Publica na branch gh-pages via peaceiris/actions-gh-pages]
        |
        v
[Link permanente: https://{usuario}.github.io/{repo}/mapa_classe_sondagens.xlsx]
```

---

## Estrutura do repositório

```
.
├── .github/
│   └── workflows/
│       └── atualizar-planilha.yml   # workflow agendado
├── scripts/
│   └── gerar_planilha.py            # script principal
├── output/
│   └── .gitkeep                     # mantém a pasta no git
└── README.md
```

---

## Pré-requisitos

- Acesso ao workspace Databricks: `https://adb-7405607349429365.5.azuredatabricks.net`
- CLI do Databricks instalado e autenticado na máquina local
- Permissão `SELECT` nas tabelas:
  - `coped.mapa_classe.consolidado`
  - `coped.mapa_classe.alunos_esperados`
  - `producao.silver.tb_diretoria`
- Conta no GitHub

---

## Setup (fazer uma vez)

### 1. Clonar o repositório

```bash
git clone https://github.com/{usuario}/{repo}.git
cd {repo}
```

### 2. Obter o refresh token do Databricks CLI

No terminal local, com o CLI autenticado:

```bash
cat ~/.databricks/token-cache.json
```

Localizar o bloco do perfil desejado e copiar o valor de `"refresh_token"` (começa com `eyJ...`).

Se o CLI ainda nao estiver autenticado:

```bash
databricks auth login --host https://adb-7405607349429365.5.azuredatabricks.net
```

### 3. Adicionar GitHub Secret

No repositório: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Valor |
|---|---|
| `DATABRICKS_REFRESH_TOKEN` | o `eyJ...` copiado no passo 2 |

O Warehouse ID e a URL do workspace já estão fixos no workflow.
Se mudarem, editar diretamente em `.github/workflows/atualizar-planilha.yml`.

### 4. Habilitar GitHub Pages

**Settings → Pages → Source: GitHub Actions**

### 5. Testar

**Actions → Atualizar Planilha Mapa Classe → Run workflow**

Link permanente após a primeira execução:

```
https://{usuario}.github.io/{repo}/mapa_classe_sondagens.xlsx
```

---

## Agendamento

Roda automaticamente, ex.: todo dia às **06h00 BRT** (09:00 UTC), definido no `atualizar-planilha.yml`:

```yaml
schedule:
  - cron: '0 9 * * *'
```

---

## Datasets gerados

| Aba | Conteúdo | Filtro principal |
|---|---|---|
| Evolucao Diaria | Sondagens por aluno por dia | datafinalizado >= 2026-06-01, LIMIT 10000 |
| Hipotese de Escrita | Distribuição de níveis de escrita | datafinalizado >= 2026-06-01, LIMIT 10000 |
| Nivel de Matematica | Distribuição de níveis de matemática | datafinalizado >= 2026-06-01, LIMIT 10000 |
| Producao Textual | Produção textual séries 3-5 Estadual | datafinalizado >= 2026-06-01, LIMIT 10000 |
| Cobertura Sondagens | Cobertura por aluno | datafinalizado >= 2026-06-01, LIMIT 10000 |
| Cobertura Esperada | Matrícula vs sondado por escola | Sem LIMIT |
| Pct Conclusao | % conclusão total/rede | 3 linhas |
| Sem Sondagem | Escolas sem nenhum registro | Sem LIMIT |
| Inconsistencias | Escolas com sondados > esperados | Sem LIMIT |

Fonte: `coped.mapa_classe.consolidado` + `coped.mapa_classe.alunos_esperados` + `producao.silver.tb_diretoria`

---

## Manutenção

### Quando o refresh token expirar

O token OAuth tem duração variável (semanas a meses). Se o workflow falhar com erro 401:

1. `databricks auth login --host https://adb-7405607349429365.5.azuredatabricks.net`
2. `cat ~/.databricks/token-cache.json` — copiar o novo `refresh_token`
3. Atualizar o Secret `DATABRICKS_REFRESH_TOKEN` no GitHub
4. Disparar o workflow manualmente para confirmar

### Alterar o período dos dados

Editar a data `2026-06-01` nas queries em `scripts/gerar_planilha.py`.

### Aumentar o limite de linhas

As 5 tabelas grandes têm `LIMIT 10000`. Aumentar nas queries correspondentes — avaliar impacto no tempo de execução.

---

## Autenticação — detalhes técnicos

Usa **OAuth OIDC com refresh token**, o mesmo mecanismo do Databricks CLI.
Nao depende de PAT (Personal Access Token).

```python
requests.post(
    f"{DATABRICKS_HOST}/oidc/v1/token",
    data={
        "grant_type"   : "refresh_token",
        "refresh_token": REFRESH_TOKEN,   # GitHub Secret
        "client_id"    : "databricks-cli",
        "scope"        : "all-apis offline_access",
    }
)
```

---

## Modo teste (dados fictícios)

Para validar o pipeline sem expor dados reais (ex: repo público):

1. Substituir `scripts/gerar_planilha.py` pela versão de teste
   (disponível no notebook `NOTEBOOK-PLANILHA-GITHUB-ACTIONS-v1` no Databricks, célula 4)
2. O Secret `DATABRICKS_REFRESH_TOKEN` nao é necessário nesse modo
3. Para voltar à versão real: usar o conteúdo da célula 3 do mesmo notebook

---

*Mantido pela equipe PDM-FDE.*


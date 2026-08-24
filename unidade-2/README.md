# Unidade 2 — Oficina de Ferramentas: contrato, execução e comparação

Evidências da oficina do módulo 2 (APIs), executada sobre
`laboratorios/plataforma-hospitalar` do repositório `arquitetura-software`.

Ambiente: Windows 11, PowerShell 5.1, Python 3.13.7, Node v22.19.0,
Spectral CLI 6.16.1.

## Índice das evidências

### Trilha essencial

| Arquivo | O que registra | Resultado |
|---|---|---|
| `testes-contrato.txt` | `pytest tests/test_api_contract.py -q` | **7 passed** |
| `spectral-valido.txt` | Lint do contrato publicado | **No results with a severity of 'error' found!** (código 0) |
| `respostas-http.txt` | Experimentos 3 e 4 + extras, contra o servidor vivo | 202, 200, 422, 422, 404, 422 |
| `docs-swagger.png` | Experimento 1 — `/docs` do FastAPI | duas operações e seis schemas |
| `bruno-importada/` | Experimento 2 — coleção **importada do OpenAPI** | 2 operações, derivadas do contrato |
| `bruno-importada-execucao.txt` | Execução da coleção recém-importada | `202` no POST, `404` no GET (protocolo vazio) |
| `bruno/` | A mesma coleção, estendida com asserções e encadeamento por `Location` | 5 requisições |
| `bruno-execucao.txt` | Execução da coleção estendida via Bruno CLI | **12/12 asserções, PASS** |

### Experimento controlado (falha deliberada)

| Arquivo | O que registra | Resultado |
|---|---|---|
| `openapi-experimento.yaml` | Cópia do contrato com `cpf: '123'` no exemplo | — |
| `spectral-experimento.txt` | Lint da cópia quebrada | **`oas3-valid-media-example`** (código 1) |

### Extensões

| Arquivo | O que registra |
|---|---|
| `analise-regras-spectral.md` | Análise das três regras promovidas a `error` |
| `openapi-gerado.yaml` | Contrato gerado por `app.openapi()` (cópia de estudo) |
| `contrato-explicito-x-gerado.txt` | Comparação campo a campo, saída bruta |
| `spectral-gerado.txt` | Lint do contrato **gerado** — reprova com 4 erros |
| `analise-contrato-gerado.md` | Quais divergências são incompatibilidade e quais são descrição |

### Documentos de análise

| Arquivo | Conteúdo |
|---|---|
| `nota-curta-contrato-x-execucao.md` | **Nota curta obrigatória** — contrato explícito x contrato gerado x execução, e a falha deliberada |
| `nota-comparativa.md` | Versão longa — as três ferramentas (Spectral, pytest, Bruno) |
| `exploracao-contrato-x-testes.md` | Exploração em dupla: promessa não verificada + asserção dependente do contrato |
| `questoes-exploratorias.md` | Respostas às cinco questões |

## Como reproduzir

A partir de `laboratorios/plataforma-hospitalar`:

```bash
py -m venv .venv
.venv\Scripts\python.exe -m pip install --upgrade pip
.venv\Scripts\python.exe -m pip install -e ".[dev]"
.venv\Scripts\python.exe -m pip install pyyaml
```

```bash
.venv\Scripts\python.exe -m pytest tests/test_api_contract.py -q
```

```bash
npx @stoplight/spectral-cli@6.16.1 lint contratos/openapi.yaml
```

Servidor, em terminal dedicado:

```bash
.venv\Scripts\python.exe -m uvicorn hospital.api.main:app --reload
```

Coleção do consumidor, com o servidor no ar:

```bash
npx @usebruno/cli run --env Local -r
```

## Notas de execução — divergências em relação ao enunciado

Três pontos em que o ambiente real não bateu com o texto da oficina. Nenhum
alterou o resultado dos experimentos, mas todos afetam a reprodução:

**1. `pyyaml` não está declarado no `pyproject.toml`.** Os módulos
`tests/test_api_contract.py` e `tests/test_k8s_manifests.py` fazem `import yaml`,
mas `pyyaml` não consta nem em `dependencies` nem em `optional-dependencies.dev`.
Com o ambiente montado exatamente como o enunciado manda, a coleta do pytest falha
com `ModuleNotFoundError: No module named 'yaml'`. Instalei `pyyaml` avulso no
venv; **não** alterei o `pyproject.toml` do repositório do professor.

**2. São 7 testes de contrato, não 6.** O enunciado prevê "6 testes aprovados";
`test_api_contract.py` hoje tem sete, incluindo
`test_request_rejects_property_forbidden_by_explicit_contract`. Todos passam — o
arquivo aparentemente ganhou um teste desde a escrita da página.

**3. Bruno foi executado pela CLI, não pelo aplicativo gráfico.** O app Bruno não
está instalado nesta máquina. A importação, porém, é real: `bruno-importada/` foi
gerada rodando `openApiToBruno` do pacote `@usebruno/converters` sobre
`contratos/openapi.yaml`, e serializada com `@usebruno/lang` — os mesmos pacotes
que o aplicativo gráfico usa por baixo no **Import Collection → OpenAPI**. O
conversor derivou tudo do documento: a tag `Elegibilidades` virou pasta, os
`summary` viraram nomes, as `description` viraram `docs` e o exemplo
`pedidoValido` virou o corpo do POST.

`bruno/` é essa mesma coleção estendida à mão com asserções e com o encadeamento
`POST → Location → GET`, que a importação não tem como gerar. As duas foram
executadas de verdade com `@usebruno/cli` contra o Uvicorn. Ambas as pastas abrem
normalmente no aplicativo gráfico.

Também vale registrar, porque apareceu na primeira execução: rodando a suíte
inteira (`pytest tests -q`), dois testes de `test_event_idempotency.py` falham no
Windows com `PermissionError: [WinError 32]` ao limpar o SQLite temporário. São de
outro módulo do curso e não têm relação com esta oficina — o alvo exigido aqui,
`test_api_contract.py`, passa integralmente.

## O que não foi feito

A **extensão opcional do gateway Ocelot** não foi executada: exige .NET SDK 8+,
que não está instalado nesta máquina (`dotnet --version` não encontra SDK). Por ser
explicitamente independente da trilha essencial, foi deixada de fora — os
entregáveis `ocelot.json` e `ocelot-consultas.txt` não existem nesta pasta.

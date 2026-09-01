# Unidade 3 — Serviços

Entregas do Módulo 3 (Serviços) da disciplina de Arquitetura de Software.

| Entrega | Arquivo |
|---|---|
| Perguntas do caso real Netflix | [`respostas-caso-netflix.pdf`](respostas-caso-netflix.pdf) (fonte em [`respostas-caso-netflix.md`](respostas-caso-netflix.md)) |
| Oficina de ferramentas | [`plataforma-hospitalar/`](plataforma-hospitalar) + [`evidencias/`](evidencias) + o relatório abaixo |
| Estudo de caso do módulo | [`respostas-estudo-de-caso.pdf`](respostas-estudo-de-caso.pdf) (fonte em [`respostas-estudo-de-caso.md`](respostas-estudo-de-caso.md)) |

- Caso real: <https://marco-mendes.github.io/arquitetura-software/modulo-3-servicos/casos-reais/>
- Oficina: <https://marco-mendes.github.io/arquitetura-software/modulo-3-servicos/oficina-de-ferramentas/>
- Estudo de caso: <https://marco-mendes.github.io/arquitetura-software/modulo-3-servicos/estudo-de-caso/>

---

## Oficina: dois serviços, dois bancos e uma falha parcial

### O que foi montado

Quatro contêineres e três redes, declarados em [`plataforma-hospitalar/infra/compose.servicos.yml`](plataforma-hospitalar/infra/compose.servicos.yml):

```
                       application-net
   ┌───────────────┐  ◄─── HTTP ───►  ┌───────────┐
   │ elegibilidade │                   │  exames   │
   └───────┬───────┘                   └─────┬─────┘
           │ elegibilidade-db-net            │ exames-db-net
           │ (internal: true)                │ (internal: true)
   ┌───────▼──────────┐             ┌────────▼───────┐
   │ db_elegibilidade │             │   db_exames    │
   │  PostgreSQL 16   │             │ PostgreSQL 16  │
   └──────────────────┘             └────────────────┘
```

Cada serviço está ligado a exatamente duas redes: `application-net`, por onde os dois conversam, e a
rede interna do seu próprio banco. Exames **não tem rota de rede** até `db_elegibilidade` — a
propriedade dos dados deixa de depender de disciplina da equipe e passa a ser uma propriedade da
infraestrutura. Isso não ficou no discurso: o teste
`test_exames_cannot_resolve_or_connect_to_eligibility_database_network` tenta a conexão de dentro do
contêiner de Exames e exige que ela falhe (evidência 11).

### Como reproduzir

```powershell
cd unidade-3\plataforma-hospitalar
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"

$env:ELEGIBILIDADE_PORT = 18001
$env:EXAMES_PORT = 18002

docker compose -f infra/compose.servicos.yml config --quiet
docker compose -f infra/compose.servicos.yml up -d --build --wait
docker compose -f infra/compose.servicos.yml ps

curl.exe -i "http://localhost:$env:EXAMES_PORT/health"
curl.exe -i -X POST "http://localhost:$env:EXAMES_PORT/exames" -H "Content-Type: application/json" -d '{\"beneficiario_id\":\"paciente-001\",\"codigo_exame\":\"HEM-001\"}'

docker compose -f infra/compose.servicos.yml stop elegibilidade
# repita o POST: 503 dependencia_indisponivel, enquanto GET /health continua 200

docker compose -f infra/compose.servicos.yml up -d --wait
$env:COMPOSE_LIVE = 1; python -m pytest tests -q
docker compose -f infra/compose.servicos.yml down -v
```

Portas 18001 e 18002 foram usadas no lugar dos padrões 8001/8002 para não disputar portas já
ocupadas na máquina.

### Evidências coletadas

Execução real, em 01/09/2026, com Docker Engine 29.2.1 (Docker Desktop, backend Linux),
Docker Compose v5.1.0 e Python 3.13.7.

| # | Arquivo | O que comprova |
|---|---|---|
| 01 | [`evidencias/01-versoes.txt`](evidencias/01-versoes.txt) | versões de Docker, Compose e Python |
| 02 | [`evidencias/02-config.txt`](evidencias/02-config.txt) | `config --quiet` sem saída (código 0) e os quatro serviços listados |
| 03 | [`evidencias/03-ps-nominal.txt`](evidencias/03-ps-nominal.txt) | quatro contêineres `healthy`; os bancos sem porta publicada (`5432/tcp`, sem `0.0.0.0:`) |
| 04 | [`evidencias/04-health-200.txt`](evidencias/04-health-200.txt) | `200 OK` nos dois `/health` |
| 05 | [`evidencias/05-post-201.txt`](evidencias/05-post-201.txt) | `201 Created` com `solicitacao_id: 1` |
| 06 | [`evidencias/06-traducao-de-erros.txt`](evidencias/06-traducao-de-erros.txt) | `422 beneficiario_inelegivel` e `422 beneficiario_desconhecido` — o 404 do vizinho não vaza |
| 07 | [`evidencias/07-falha-parcial-503.txt`](evidencias/07-falha-parcial-503.txt) | **a falha parcial**: `503 dependencia_indisponivel` no POST e `200 OK` no `/health`, no mesmo instante |
| 08 | [`evidencias/08-recuperacao.txt`](evidencias/08-recuperacao.txt) | os dois `/health` de volta a `200` após `up -d --wait` |
| 09 | [`evidencias/09-testes-fronteiras.txt`](evidencias/09-testes-fronteiras.txt) | `4 passed` em `test_service_boundaries.py` |
| 10 | [`evidencias/10-testes-modulo-3.txt`](evidencias/10-testes-modulo-3.txt) | suíte sem `COMPOSE_LIVE`: `4 passed, 1 skipped` |
| 11 | [`evidencias/11-fronteira-de-rede.txt`](evidencias/11-fronteira-de-rede.txt) | com `COMPOSE_LIVE=1`: `5 passed` — Exames **não alcança** o banco de Elegibilidade |
| 12 | [`evidencias/12-limpeza.txt`](evidencias/12-limpeza.txt) | `down -v` removeu contêineres, redes e volumes; `ps -a` vazio |

O momento central está na evidência 07 — duas respostas contraditórias do mesmo serviço, no mesmo
instante:

```
POST /exames   → HTTP/1.1 503 Service Unavailable   {"detail":{"codigo":"dependencia_indisponivel"}}
GET  /health   → HTTP/1.1 200 OK                    {"status":"ok","servico":"exames"}
```

---

## Questões exploratórias da oficina

### Qual dependência permanece saudável e qual capacidade deixa de ser concluída?

Permanecem saudáveis: o **processo de Exames** (o servidor uvicorn responde), o **banco próprio de
Exames** (a conexão do `/health` funciona) e o `db_elegibilidade`, que nunca foi parado. Deixa de ser
concluída uma única capacidade: **registrar uma solicitação de exame** (`POST /exames`), porque ela
atravessa a fronteira e depende da decisão de Elegibilidade antes de gravar.

A distinção que a oficina quer mostrar é entre **"o processo está vivo"** e **"a capacidade está
disponível"**. Repare no que *não* aconteceu: o sistema não caiu inteiro, a base de Exames continua
íntegra, os registros gravados antes continuam lá, e qualquer operação de Exames que não dependesse
do vizinho seguiria funcionando. A falha ficou **contida no caminho que atravessa a fronteira** — é
exatamente essa contenção que caracteriza uma falha parcial e que distingue esta arquitetura do banco
único da Netflix em 2008, em que a corrupção de um componente parou a operação inteira.

### Por que o health check não detecta a indisponibilidade do vizinho?

Porque ele foi escrito para responder outra pergunta. O comando declarado no Compose é:

```yaml
healthcheck:
  test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
  interval: 2s
  timeout: 3s
  retries: 15
```

Ele chama `localhost:8000/health` — **o próprio serviço, de dentro do próprio contêiner** — e a rota
`/health` de `exames.py` verifica apenas se o processo responde e se a base própria aceita um
`SELECT 1`. Nenhuma das duas coisas consulta Elegibilidade. Pelos critérios que ele mesmo declarou,
Exames está de fato saudável.

E isso é **intencional**, não um defeito a corrigir. Se o health check de Exames chamasse o vizinho,
três coisas ruins aconteceriam:

1. **Propagação de falha**: uma queda de Elegibilidade marcaria Exames como não saudável, e o
   orquestrador reiniciaria ou tiraria de serviço um contêiner que está perfeitamente funcional —
   transformando a falha de um serviço na indisponibilidade de dois. É a cascata que a arquitetura
   quer evitar.
2. **Efeito rebanho**: cada réplica passaria a bater no vizinho a cada 2 segundos, adicionando carga
   justo no serviço que já está com problema.
3. **Perda de significado**: a verificação de liveness deixaria de dizer "posso ficar no ar" e
   passaria a dizer "todas as minhas dependências estão bem", que é uma pergunta diferente.

A conclusão prática, e é a lição da oficina: **monitorar apenas processos vivos dá uma falsa sensação
de saúde.** O painel fica todo verde enquanto o cliente recebe 503. Saúde de processo e saúde de
capacidade são medidas diferentes, e a segunda só aparece em métricas de negócio — taxa de sucesso do
`POST /exames`, latência, contagem de `dependencia_indisponivel` — ou numa verificação de readiness
separada, que pode considerar dependências para decidir se aquela réplica deve receber tráfego.

### Como a configuração de timeout (2.0 segundos) protege contra cascata de falhas?

O prazo de espera está declarado na criação do cliente HTTP, em `exames.py`:

```python
with httpx.Client(base_url=ELIGIBILIDADE_URL, timeout=2.0) as client:
```

Sem ele, uma chamada a um vizinho que aceita a conexão mas nunca responde ficaria pendurada
indefinidamente. O que acontece em seguida é mecânico: cada requisição em curso segura um *worker* do
uvicorn; requisições novas se acumulam; em pouco tempo **todos** os trabalhadores de Exames estão
bloqueados esperando por Elegibilidade. Aí Exames para de responder até para quem não precisava do
vizinho — inclusive para o próprio `/health`, o que o marcaria como não saudável e o levaria a ser
reiniciado. **A indisponibilidade de um serviço teria virado a indisponibilidade de dois**, e daí para
os consumidores de Exames, e assim por diante: é assim que uma falha local vira uma queda geral.

O timeout de 2 segundos coloca um **limite superior no custo de a fronteira falhar**. Ele converte
uma espera infinita em um erro rápido e classificável (`httpx.HTTPError` → `503
dependencia_indisponivel`), libera o trabalhador imediatamente e devolve ao cliente uma resposta
verdadeira e acionável — "tente de novo mais tarde" — em vez de deixá-lo pendurado.

É o custo da fronteira física ficando visível: numa chamada dentro do mesmo processo, esperar para
sempre pela rede simplesmente não é um cenário possível; assim que a chamada atravessa um processo,
passa a ser o cenário padrão, e é preciso escrevê-lo à mão. O timeout é a primeira e mais barata
defesa; ele resolve a espera, mas não evita repetir a chamada condenada a cada requisição — para isso
o passo seguinte seria um disjuntor de circuito (Resilience4j hoje, Hystrix no caso da Netflix), que
para de tentar depois de N falhas e volta a testar aos poucos.

---

## Por que o erro é 503 e não 500 — e a tabela de tradução

Exames **nunca repassa o código de status do vizinho**. Ele traduz cada falha possível para uma
semântica própria, o que impede que o consumidor fique acoplado à topologia interna do sistema:

| O que acontece com Elegibilidade | Exames responde | Por quê |
|---|---|---|
| Não responde, ou a rede falha | `503 dependencia_indisponivel` | problema transitório de infraestrutura; não é culpa de quem chamou |
| Responde `5xx` | `503 dependencia_indisponivel` | o vizinho quebrou; para o consumidor é a mesma indisponibilidade |
| Responde `404` (beneficiário desconhecido) | `422 beneficiario_desconhecido` | o pedido é que está errado; a culpa volta para quem chamou |
| Responde fora do contrato esperado | `502 contrato_invalido` | resposta ilegível de um serviço a montante |
| Responde que o beneficiário não é elegível | `422 beneficiario_inelegivel` | não é falha técnica: é uma decisão de negócio negativa |

Um `404` de Elegibilidade não pode virar `404` de Exames, porque significariam coisas diferentes: o
recurso `/exames` existe; quem não existe é o beneficiário. E `503` comunica algo que `500` não
comunicaria — um `500` diria "quebrei por dentro", um `400` culparia o consumidor por um pedido
correto; o `503` diz "tente de novo mais tarde", que é a informação verdadeira e acionável.

As linhas 3 e 5 da tabela foram verificadas na prática (evidência 06) e a linha 1, ao parar o
contêiner (evidência 07).

## Conteúdo desta pasta

```
unidade-3/
├── README.md                        este relatório
├── respostas-caso-netflix.md        respostas do caso Netflix (fonte)
├── respostas-caso-netflix.pdf       respostas do caso Netflix (entregável)
├── respostas-estudo-de-caso.md      respostas do estudo de caso (fonte)
├── respostas-estudo-de-caso.pdf     respostas do estudo de caso (entregável)
├── evidencias/                      12 arquivos de saída da execução real
└── plataforma-hospitalar/           recorte executável do laboratório usado na oficina
    ├── Dockerfile                   imagem das duas aplicações
    ├── pyproject.toml
    ├── infra/
    │   ├── compose.servicos.yml     topologia: 4 contêineres, 3 redes, health checks
    │   └── postgres/                imagem dos bancos + init.sql (papéis, esquemas, dados)
    ├── src/hospital/servicos/
    │   ├── elegibilidade.py         serviço que decide, falando só com a própria base
    │   └── exames.py                serviço que depende do primeiro; trata a falha parcial
    └── tests/
        ├── test_service_boundaries.py       contrato, falha parcial, falha da base própria
        └── test_compose_network_boundary.py fronteira de rede entre serviço e banco alheio
```

O laboratório é o `laboratorios/plataforma-hospitalar` do repositório do curso
(<https://github.com/marco-mendes/arquitetura-software>); aqui está o recorte necessário para
reproduzir esta oficina. O que foi produzido nesta entrega são as evidências, este relatório e as
respostas do caso Netflix.

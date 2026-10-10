# Fase 4 — Instrumentação com OpenTelemetry

**Projeto:** ToggleMaster (FIAP — Fase 4)
**Escopo:** `requirements.txt`, `Dockerfile` e o uso de cursores no `app.py`

---

## Visão geral

Terceiro serviço instrumentado da fase, e o segundo em Python. É **repetição de um padrão já validado** no `flag-service`, onde a investigação original está documentada em detalhe.

Este documento registra o que foi reaproveitado, o que é específico deste serviço, e a validação. Para o raciocínio por trás de cada decisão — por que `opentelemetry-bootstrap` não serve no Dockerfile multi-stage, por que os pacotes são `0.xxbY`, como o instrumentador do psycopg2 funciona por dentro — ver o documento equivalente do `flag-service`.

---

## O que foi reaproveitado

As três mudanças são idênticas às do `flag-service`, porque os dois serviços são o mesmo molde: Flask servido por gunicorn, pool de psycopg2 criado no import e um decorator `require_auth` que chama o `auth-service` por HTTP.

```diff
# requirements.txt
+opentelemetry-distro[otlp]==0.66b1
+opentelemetry-instrumentation-flask==0.66b1
+opentelemetry-instrumentation-requests==0.66b1
+opentelemetry-instrumentation-psycopg2==0.66b1
+opentelemetry-instrumentation-logging==0.66b1
```

```diff
# Dockerfile
-CMD ["gunicorn", "--bind", "0.0.0.0:8003", "app:app"]
+CMD ["opentelemetry-instrument", "gunicorn", "--bind", "0.0.0.0:8003", "app:app"]
```

### A correção do `cursor_factory`, agora preventiva

```diff
-    pool = SimpleConnectionPool(1, 5, dsn=DATABASE_URL)
+    pool = SimpleConnectionPool(1, 5, dsn=DATABASE_URL, cursor_factory=RealDictCursor)
```

```diff
-        cur = conn.cursor(cursor_factory=RealDictCursor)
+        cur = conn.cursor()
```

Três ocorrências (`app.py:87, 117, 159`). O handler de DELETE (`app.py:185`) já usava `conn.cursor()` e passa a receber `RealDictCursor`; ele só lê `cur.rowcount`, então nada muda.

**A diferença de processo vale registrar.** No `flag-service`, a ausência dos spans de banco foi descoberta depois, investigando por que o Collector recebia spans de Flask e de `requests` mas nenhum de SQL. Aqui a correção entrou junto com a instrumentação, e o span apareceu na primeira execução.

O motivo técnico está no documento do `flag-service`: o instrumentador substitui o `cursor_factory` da conexão por uma fábrica própria, e passar a fábrica explicitamente em `conn.cursor(...)` sobrescreve a instrumentada, **em silêncio**.

---

## O que é específico deste serviço

**As regras são JSONB.** Este é o único dos dois que usa `psycopg2.extras.Json` para serializar o corpo das regras:

```python
cur.execute(..., (flag_name, is_enabled, Json(rules_obj)))
```

Era o risco próprio daqui: trocar a fábrica de cursores na conexão poderia afetar a serialização. Não afeta — `Json()` atua no parâmetro, não no cursor. Validado com um `POST /rules` real, com as regras voltando íntegras na resposta.

---

## Validação

Ambiente: `togglemaster-platform-legacy`, com Collector `0.162.0` e exporter `debug`.

### Duas árvores completas

```
POST /rules                     targeting-service  Server  span=0a54a358  (raiz)
├── GET                         targeting-service  Client  parent=0a54a358  → auth-service
│   └── GET /validate           auth-service       Server  parent=6c8a4980
│       └── sql.conn.query …    auth-service       Client
└── INSERT                      targeting-service  Client  parent=0a54a358
```

```
GET /rules/<string:flag_name>   targeting-service  Server  span=0c160157  (raiz)
├── GET → auth-service
└── SELECT                      targeting-service  Client
```

O nome do span é a **rota**, `GET /rules/<string:flag_name>`, não a URL com o nome da flag — é o que permite agregação por endpoint no APM.

### Escopos emitindo

```
InstrumentationScope app
InstrumentationScope github.com/XSAM/otelsql 0.44.0
```

O segundo vem do `auth-service`, do outro lado da chamada — os dois serviços aparecem no mesmo trace.

| Verificação | Resultado |
|---|---|
| Spans de Flask, `requests` e psycopg2 | presentes na primeira execução |
| Log correlacionado por OTLP | presente |
| `POST /rules` com JSONB | regras íntegras na resposta |
| Trace atravessando para o `auth-service` | confirmado |

---

## O que não foi feito

As mesmas pendências do `flag-service`: formatter JSON no stdout (que só importa quando o `filelog` entrar, na rodada do Loki), métricas de negócio e amostragem, que segue em 100% por ser ambiente de demonstração.

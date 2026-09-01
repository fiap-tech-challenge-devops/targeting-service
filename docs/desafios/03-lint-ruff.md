# Fase 3 — Achados do `ruff` na esteira de CI

**Projeto:** ToggleMaster (FIAP — Fase 3)
**Escopo:** Correções em `app.py` e criação do `ruff.toml` para o job `Linter`

---

## Visão geral

O job `Linter` da esteira de CI reprovava com sete achados do `ruff`. Três eram seguros de corrigir; quatro são o mesmo padrão de tratamento de exceção, que **não pode ser corrigido no código sem alterar comportamento**.

Este documento registra o diagnóstico, a correção e por que a última categoria foi tratada por configuração e não por reescrita.

---

## O que o linter acusou

Execução de referência: [run 33429670467](https://github.com/fiap-tech-challenge-devops/targeting-service/actions/runs/33429670467), `ruff 0.16.5`.

```
app.py:1:1     I001    Import block is un-sorted or un-formatted
app.py:5:8     F401    `json` imported but unused
app.py:101:12  BLE001  Do not catch blind exception: `Exception`
app.py:123:12  BLE001  Do not catch blind exception: `Exception`
app.py:169:12  BLE001  Do not catch blind exception: `Exception`
app.py:194:12  BLE001  Do not catch blind exception: `Exception`
app.py:203:34  PLW1508 Invalid type for environment variable default
Found 7 errors.
```

O job tem `continue-on-error: true`: aparece vermelho na interface, mas não reprova a esteira.

### Uma descoberta sobre a configuração

**Não existia arquivo de configuração do `ruff` neste repositório**, e o CI instala a ferramenta com `pip install ruff`, sem versão fixa.

Isso importa mais do que parece. O conjunto de regras padrão do `ruff` **cresceu** entre versões — `BLE001` e `PLW1508` não faziam parte do padrão quando o projeto começou. Sem configuração e sem versão fixa, o lint pode ficar vermelho sozinho, sem nenhuma mudança no código, só porque uma release nova ativou uma regra nova.

É o mesmo problema que levou o [`reusable-workflows`](https://github.com/fiap-tech-challenge-devops/reusable-workflows) a fixar a versão exata de toda action: uma esteira que existe para dar sinal não deveria executar código que muda sozinho.

---

## Desafio 1 — `I001`, bloco de imports desordenado

### O que estava errado

```python
import os
import sys
import psycopg2
import requests
import json
from psycopg2.extras import RealDictCursor, Json
from psycopg2.pool import SimpleConnectionPool
from flask import Flask, request, jsonify
from dotenv import load_dotenv
from functools import wraps
import logging
```

Biblioteca padrão, dependências de terceiros e imports `from` misturados, sem ordem.

### Correção aplicada

Automática, via `ruff check --select I001 --fix`. O resultado separa biblioteca padrão de terceiros e ordena alfabeticamente dentro de cada grupo.

**Zero mudança de comportamento** — em Python, a ordem dos imports de módulos independentes não altera o que é carregado.

---

## Desafio 2 — `F401`, import não utilizado

### O que estava errado

```python
import json
```

O módulo `json` não é referenciado em nenhuma linha do arquivo. O que o código usa é `Json`, uma classe **do `psycopg2.extras`** para serializar campos `JSONB` — nome parecido, pacote diferente.

Provavelmente sobrou de uma refatoração em que a serialização passou a ser responsabilidade do driver do banco.

### Correção aplicada

Automática, via `ruff check --select F401 --fix`. Remoção da linha.

**Zero mudança de comportamento** — remover um import não utilizado não afeta nada além de um objeto a menos na memória.

---

## Desafio 3 — `PLW1508`, tipo do valor padrão

### O que estava errado

```python
port = int(os.getenv("PORT", 8003))
```

A assinatura de `os.getenv` é `getenv(key, default)`, e a função retorna `str | None`. Passar um `int` como `default` é inconsistente com o tipo de retorno: quando a variável existe, vem `str`; quando não existe, vem `int`.

### Por que a correção é segura

O resultado é envolvido por `int()`. Com `8003` o `int()` recebe um inteiro e devolve `8003`; com `"8003"` recebe uma string e devolve `8003`. **Mesmo valor, mesmo tipo na saída.**

### Correção aplicada

```diff
-    port = int(os.getenv("PORT", 8003))
+    port = int(os.getenv("PORT", "8003"))
```

---

## Desafio 4 — `BLE001`, e por que não foi corrigido no código

### O que o linter aponta

Quatro ocorrências, todas com a mesma forma:

```python
except psycopg2.IntegrityError:
    if conn: conn.rollback()
    log.warning(f"Tentativa de criar regra duplicada: '{flag_name}'")
    return jsonify({"error": f"Regra para a flag '{flag_name}' já existe"}), 409
except Exception as e:
    if conn: conn.rollback()
    log.error(f"Erro ao criar regra: {e}")
    return jsonify({"error": "Erro interno do servidor", "details": str(e)}), 500
```

A regra `BLE001` (*blind except*) desaconselha capturar `Exception` porque isso engole erros que você não previu — inclusive bugs de programação, que ficariam mais visíveis se estourassem.

### Por que a recomendação não se aplica aqui

O argumento vale para código de biblioteca, onde engolir uma exceção esconde o problema de quem chamou. Num **handler de rota HTTP**, o papel é o oposto: qualquer exceção precisa virar uma resposta, e não derrubar a requisição.

Repare no que cada bloco faz antes de responder:

```python
if conn: conn.rollback()
```

Estreitar o `except` para tipos específicos significaria que qualquer exceção fora da lista **subiria sem passar pelo rollback**. O Flask devolveria um 500 genérico e a conexão voltaria ao pool com uma transação aberta — que é um problema bem pior que o achado do linter.

Os `except Exception` também são sempre o **último** de uma cadeia: os casos previstos (`psycopg2.IntegrityError`, por exemplo) já têm tratamento próprio, com o status HTTP correto. O catch-all é a rede de segurança, não o tratamento principal.

### Decisão: configuração, não reescrita

Criado o `ruff.toml` na raiz:

```toml
required-version = ">=0.16,<0.17"

[lint]
ignore = ["BLE001"]
```

Duas coisas de uma vez:

| linha | efeito |
|---|---|
| `required-version` | o `ruff` recusa rodar numa versão fora da faixa, em vez de silenciosamente aplicar um conjunto de regras diferente |
| `ignore = ["BLE001"]` | desliga a regra que conflita com o desenho dos handlers |

O `required-version` não substitui fixar a versão no `pip install` — que seria o lugar certo, e fica registrado abaixo como pendência —, mas transforma uma mudança silenciosa de comportamento em erro explícito.

---

## Resumo das mudanças

| arquivo | mudança | motivo |
|---|---|---|
| `app.py` | bloco de imports reordenado | `I001` |
| `app.py` | `import json` removido | `F401`; o que se usa é `Json` do `psycopg2.extras` |
| `app.py` | `os.getenv("PORT", 8003)` → `"8003"` | `PLW1508`; `int()` produz o mesmo valor |
| `ruff.toml` *(novo)* | `ignore = ["BLE001"]` + `required-version` | catch-all de rota é intencional |

**Nenhuma alteração de lógica.** Nenhuma rota, status HTTP, consulta SQL ou tratamento de exceção mudou.

---

## Validação

### Lint

```
ruff 0.16.5 → All checks passed!
```

### Build

```
docker build → ok, imagem de 117 MB
```

### Execução

O serviço foi subido em container, ligado a um PostgreSQL com o `db/init.sql` aplicado e ao `auth-service` para validação de chave de API:

| passo | resultado |
|---|---|
| Boot | `Pool de conexões com o PostgreSQL (targeting) inicializado.` |
| `GET /health` | `200` `{"status":"ok"}` |
| `POST /rules` com chave válida | `201`, regra `PERCENTAGE 50` criada |
| `GET /rules/<flag>` pelo `evaluation-service` | `200`, JSON consumido com sucesso |

A última linha é a mais relevante: o `evaluation-service` consultou este serviço como parte de uma avaliação de flag ponta a ponta, e a resposta foi desserializada sem erro. Confirma que o `app.py` continua servindo o contrato esperado depois da reordenação de imports.

---

## Pendência registrada

**Fixar a versão do `ruff` no CI.** O `python-ci.yml` do [`reusable-workflows`](https://github.com/fiap-tech-challenge-devops/reusable-workflows) executa `pip install ruff` sem versão. O `required-version` deste repositório detecta a divergência, mas a correção de fato é `pip install ruff==0.16.5` no workflow compartilhado — mudança que afeta os três serviços em Python e exige uma tag nova.

## Observação fora do escopo do lint

As respostas de erro incluem o texto da exceção:

```python
return jsonify({"error": "Erro interno do servidor", "details": str(e)}), 500
```

Isso expõe detalhe interno — nome de tabela, trecho de SQL, mensagem do driver — para quem chamou a API. Não é achado do `ruff` e corrigir seria mudança de comportamento, então **não foi alterado**. Fica registrado como ponto a avaliar.

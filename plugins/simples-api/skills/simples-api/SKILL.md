---
name: simples-api
description: Integra a Simples API (api.simplesapi.com.br), API brasileira de dados de veículos. Use quando precisar consultar um veículo pela placa brasileira (Mercosul ou antiga) e obter marca, modelo, versão, ano, cor, combustível, município e UF, o preço FIPE com o histórico, a ficha técnica ou a bateria indicada; quando precisar ler placas de veículos em fotos (OCR); ou quando precisar conferir o saldo da conta. Ensina a autenticação, os endpoints, os campos da resposta, os erros, o que é cobrado e as boas práticas, com exemplos em curl, Python e TypeScript.
---

# Simples API

A Simples API devolve dados de veículos brasileiros a partir da placa em texto ou de uma foto. A base é `https://api.simplesapi.com.br`, as respostas são JSON e a cobrança é pré-paga, por chamada.

- Documentação completa: https://simplesapi.com.br/docs
- Preços: https://simplesapi.com.br/precos
- Servidor MCP para agentes: https://simplesapi.com.br/mcp

## Quando usar

Use esta API para:

- Consultar um veículo pela placa: marca, modelo, versão, ano de fabricação e ano modelo, cor, combustível, tipo, espécie, passageiros, cilindradas, eixos, município, UF e origem.
- Obter, pela mesma consulta, a versão da tabela FIPE com o preço do mês e o valor no mesmo mês dos anos anteriores, a ficha técnica (potência, torque, câmbio, consumo, pesos, dimensões) e a bateria indicada (Moura e Heliar).
- Ler placas em fotos (OCR), com a confiança de cada leitura e, se pedir, os mesmos dados do veículo de cada placa lida.

Não use para multas, débitos, IPVA, restrições, histórico de roubo ou dados do proprietário: a API não traz nada disso (veja "O que a resposta não traz").

## Antes de começar: a API key

1. Crie uma conta em https://simplesapi.com.br/register. A conta já vem com R$ 5,00 de crédito grátis, sem cartão.
2. Confirme o e-mail. Sem isso, a API responde 403 `EMAIL_NOT_VERIFIED`.
3. Copie a API key no painel (https://simplesapi.com.br/dashboard). Ela começa com `sk_live_`.
4. Guarde a key na variável de ambiente `SIMPLES_API_KEY` ou no gerenciador de segredos do projeto. Nunca no código, no repositório, em logs ou no navegador.

Toda requisição leva os headers:

```
Authorization: Bearer $SIMPLES_API_KEY
Content-Type: application/json
```

## Endpoints

### POST /v1/vehicle/plate: consulta pela placa

Corpo:

```json
{"plate": "ABC1D23"}
```

Aceita o padrão Mercosul (`ABC1D23`) e o formato antigo (`ABC-1234`), em maiúsculas ou minúsculas, com ou sem hífen.

Custa R$ 0,10 por consulta, com ou sem dados retornados: a placa não encontrada (404) também é cobrada. Placa em formato inválido (422) e consulta indisponível (503) não são cobradas.

Resposta (200):

```json
{
  "plate": "ABC1D23",
  "vehicle_info": {
    "brand": "VOLKSWAGEN", "model": "GOL", "version": "1.0 MPI TOTALFLEX",
    "year": 2020, "model_year": 2021, "color": "BRANCO", "fuel": "FLEX",
    "vehicle_type": "AUTOMOVEL", "species": "PASSAGEIRO", "passenger_capacity": 5,
    "engine_cc": 1000, "max_traction_capacity": null, "axles": 2,
    "assembly_type": "COMPLETA", "gross_weight": null,
    "city": "SAO PAULO", "state": "SP", "origin": "NACIONAL"
  },
  "fipe": {
    "level": "exact", "code": "005340-6", "brand": "VW - VolksWagen",
    "name": "Gol 1.0 Flex 12V 5p", "model_year": 2021, "zero_km": false, "fuel": "Flex",
    "price": 55210.0, "price_month": "2026-11",
    "specs": {"engine": "1.0 12V", "power_cv": {"etanol": 84, "gasolina": 75}, "transmission": "manual", "body": "hatch", "doors": 5},
    "price_history": [
      {"month": "2024-11", "price": 58900.0}, {"month": "2025-11", "price": 56800.0},
      {"month": "2026-11", "price": 55210.0}
    ],
    "alternatives": []
  },
  "battery": {
    "options": [
      {"maker": "moura", "code": "M50ED", "alt_code": "Z50ED", "start_stop": null, "level": "exact",
       "ah": 50, "cca": 390, "polarity": "direita", "technology": "convencional",
       "length_mm": 210, "width_mm": 175, "height_mm": 190, "weight_kg": 12.74}
    ]
  }
}
```

Como ler a resposta:

- `vehicle_info`: os dados cadastrais do veículo. `version`, `engine_cc`, `max_traction_capacity`, `axles`, `assembly_type` e `gross_weight` podem vir `null`.
- `fipe`: a versão da tabela FIPE ligada à placa.
  - `level` vem `"exact"` ou `"probable"`. Quando é `"probable"`, `alternatives` lista as outras versões possíveis, cada uma com o seu preço.
  - `price` é o preço do mês mais recente (`price_month`). `price_history` traz o valor no mesmo mês dos anos anteriores.
  - `specs` traz só os campos de ficha técnica que temos para a versão.
  - `fipe` vem `null` quando a placa não tem ligação com a tabela FIPE: reboques, veículos fora da tabela ou placas ainda em análise.
- `battery`: as baterias indicadas para a versão no ano-modelo da placa.
  - `options` lista cada referência com `maker` (`moura` ou `heliar`), `code`, `alt_code` (Zetta), `ah`, `cca`, `polarity`, `technology` e as medidas.
  - `start_stop` vem `true` para a bateria da aplicação com start-stop, `false` para a sem start-stop e `null` quando o fabricante não separa. Ele descreve a bateria, não o carro.
  - `level` vem `"exact"`, `"probable"` (indicação menos certa) ou `"family"` (estimativa pela família do modelo).
  - `battery` vem `null` sem ligação com a FIPE, sem bateria para a versão nesse ano ou quando as versões possíveis pedem baterias diferentes.

Trate `fipe`, `battery` e os campos que podem vir `null` como opcionais no seu código: `null` é uma resposta normal, não um erro.

### POST /v1/ocr/plate: placas em uma foto

Corpo:

```json
{"image": "data:image/jpeg;base64,/9j/4AAQSkZJRg...", "include_vehicle_info": false}
```

- `image`: a imagem em base64, com ou sem o prefixo `data:image/...;base64,`, em JPEG, PNG ou WebP, com até 10 MB. Recomendamos pelo menos 640x480 pixels.
- `include_vehicle_info`: com `true`, cada placa lida com confiança de 0,90 ou mais traz também `vehicle_info`, `fipe` e `battery`, iguais aos da consulta pela placa.

Resposta (200):

```json
{
  "plates": [
    {
      "plate": "ABC1D23",
      "confidence": 0.95,
      "plate_type": "mercosul",
      "bbox": {"x1": 100, "y1": 50, "x2": 300, "y2": 150},
      "vehicle_info": null,
      "fipe": null,
      "battery": null
    }
  ],
  "detection_method": "yolo"
}
```

- `plates` pode ter várias placas.
- `confidence` vai de 0 a 1.
- `plate_type` vem `"mercosul"`, `"old_model"` ou `"unknown"`.
- `bbox` dá as coordenadas da placa na imagem, em pixels, e pode vir `null`.

Custa R$ 0,05 por placa lida. Nenhuma placa lida, imagem inválida ou erro não são cobrados.

Com `"include_vehicle_info": true`, cada placa consultada custa mais R$ 0,10, com ou sem dados retornados, ou seja, R$ 0,15 no total. Placas com confiança abaixo de 0,90 não são consultadas e custam só os R$ 0,05 da leitura. O mesmo vale quando a consulta está indisponível.

### GET /v1/billing/balance: saldo (grátis)

Sem corpo, com a mesma API key. Resposta (200):

```json
{
  "balance_cents": 1000,
  "balance_formatted": "R$ 10,00",
  "low_balance_threshold_cents": 200,
  "low_balance_alert_enabled": true
}
```

Use antes de um lote, para saber quantas consultas cabem no saldo.

## Erros

Todo erro vem no mesmo formato:

```json
{"error": true, "code": "VEHICLE_NOT_FOUND", "message": "Nenhum dado de veículo encontrado para esta placa", "details": {"amount_charged_cents": 10}}
```

Decida pelo `code`, não pela `message`:

| Status | `code` | Cobrado? | O que o código deve fazer |
| --- | --- | --- | --- |
| 400 | `INVALID_IMAGE` | Não | Conferir o base64, o formato (JPEG, PNG ou WebP) e o tamanho (até 10 MB). Não repetir a mesma imagem. |
| 401 | `INVALID_API_KEY` | Não | Conferir a key e o header `Authorization: Bearer`. Não repetir. |
| 402 | `INSUFFICIENT_BALANCE` | Não | Parar o lote e avisar que é preciso recarregar no painel (Pix ou cartão, mínimo de R$ 20,00). |
| 403 | `EMAIL_NOT_VERIFIED` | Não | Confirmar o e-mail da conta. |
| 403 | `TERMS_NOT_ACCEPTED` | Não | Aceitar os Termos de Uso no painel. |
| 404 | `VEHICLE_NOT_FOUND` | Sim (R$ 0,10) | Placa sem dados. Guardar esse resultado e não consultar a mesma placa de novo. |
| 422 | `VALIDATION_ERROR` | Não | Placa ou corpo inválido. Validar a placa antes de chamar. |
| 422 | `OCR_FAILED` | Não | Nenhuma placa lida na foto. Pedir outra foto, mais nítida ou mais próxima. |
| 429 | `RATE_LIMIT_EXCEEDED` | Não | Passou do limite por minuto. Esperar até `X-RateLimit-Reset` e seguir. |
| 503 | `VEHICLE_API_UNAVAILABLE` | Não | Consulta indisponível. Tentar de novo com espera crescente, no máximo 3 tentativas. |
| 503 | `OCR_UNAVAILABLE` | Não | Leitura indisponível. Tentar de novo com espera crescente, no máximo 3 tentativas. |

## Limites

O limite é de 100 requisições por minuto por conta. As respostas trazem os headers `X-RateLimit-Limit`, `X-RateLimit-Remaining` e `X-RateLimit-Reset`, este último um timestamp Unix de quando a janela recomeça.

## Preços

- Leitura de placas (`POST /v1/ocr/plate`): R$ 0,05 por placa lida.
- Dados do veículo junto com a leitura (`"include_vehicle_info": true`): mais R$ 0,10 por placa consultada, R$ 0,15 no total.
- Consulta pela placa (`POST /v1/vehicle/plate`): R$ 0,10 por consulta, com ou sem dados retornados.
- Saldo (`GET /v1/billing/balance`): grátis.
- A conta nova ganha R$ 5,00 de crédito grátis. A recarga é feita no painel, via Mercado Pago (Pix ou cartão), com mínimo de R$ 20,00.

Os preços em vigor ficam em https://simplesapi.com.br/precos.

## Boas práticas

1. **Chame a API só do servidor**: backend, função serverless ou edge function. Nunca do navegador ou do app, porque a key fica visível e qualquer pessoa pode gastar o saldo. Em apps feitos com Lovable, Bolt, v0 e similares, guarde a key num segredo e faça a chamada numa função no servidor. O passo a passo está em https://simplesapi.com.br/lovable.
2. **Leia a key do ambiente** (`SIMPLES_API_KEY`). Não a escreva em código, commits ou logs.
3. **Normalize e valide a placa antes de chamar**: maiúsculas, sem hífen nem espaços, e o formato `^[A-Z]{3}[0-9][A-Z0-9][0-9]{2}$`, que cobre o Mercosul e o antigo. Placa fora do formato nem precisa ir para a API.
4. **Guarde as respostas por placa**, inclusive as 404. Consultar a mesma placa de novo é cobrado de novo.
5. **Repita só o 503 e o 429.** No 503, espere 1 s e depois 2 s, no máximo 3 tentativas. No 429, espere até `X-RateLimit-Reset`. Os demais erros não melhoram com repetição.
6. **Em lotes**, confira o saldo antes, mantenha no máximo 100 requisições por minuto (uma fila com uma chamada a cada 0,6 s basta) e pare no primeiro 402.
7. **Use um timeout de 30 s** por requisição. A maioria das respostas chega em poucos segundos.
8. **No OCR**, envie fotos comprimidas (JPEG) e use `include_vehicle_info` só quando precisar dos dados, porque cada placa consultada custa mais R$ 0,10.

## Exemplos

### curl

```bash
curl -X POST "https://api.simplesapi.com.br/v1/vehicle/plate" \
  -H "Authorization: Bearer $SIMPLES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"plate": "ABC1D23"}'

# A imagem vai pelo stdin: uma foto real passa do limite de tamanho de um argumento
{ printf '{"image": "'; base64 < foto.jpg | tr -d '\n'; printf '", "include_vehicle_info": true}'; } |
  curl -X POST "https://api.simplesapi.com.br/v1/ocr/plate" \
    -H "Authorization: Bearer $SIMPLES_API_KEY" \
    -H "Content-Type: application/json" \
    --data-binary @-

curl "https://api.simplesapi.com.br/v1/billing/balance" \
  -H "Authorization: Bearer $SIMPLES_API_KEY"
```

### Python (httpx)

```python
import os
import re
import time

import httpx

API_URL = "https://api.simplesapi.com.br"
PLATE_RE = re.compile(r"^[A-Z]{3}[0-9][A-Z0-9][0-9]{2}$")


class SimplesApiError(Exception):
    def __init__(self, status: int, code: str, message: str):
        super().__init__(f"{status} {code}: {message}")
        self.status, self.code = status, code


def erro_da_resposta(resposta: httpx.Response) -> SimplesApiError:
    """O erro da API, ou HTTP_ERROR quando o corpo não é o JSON dela (um 502 de proxy, por exemplo)."""
    try:
        corpo = resposta.json()
        return SimplesApiError(resposta.status_code, corpo["code"], corpo["message"])
    except (ValueError, KeyError, TypeError):
        return SimplesApiError(resposta.status_code, "HTTP_ERROR", resposta.text[:200])


def consultar_placa(placa: str) -> dict | None:
    """Dados do veículo pela placa, ou None quando a placa não tem dados (404, cobrado)."""
    placa = re.sub(r"[^A-Za-z0-9]", "", placa).upper()
    if not PLATE_RE.match(placa):
        raise ValueError(f"Placa inválida: {placa}")

    headers = {"Authorization": f"Bearer {os.environ['SIMPLES_API_KEY']}"}
    with httpx.Client(base_url=API_URL, headers=headers, timeout=30) as client:
        for tentativa in range(1, 4):
            resposta = client.post("/v1/vehicle/plate", json={"plate": placa})
            if resposta.status_code == 200:
                return resposta.json()
            if resposta.status_code == 404:
                return None
            if tentativa < 3 and resposta.status_code == 429:
                reset = float(resposta.headers.get("X-RateLimit-Reset", time.time() + 1))
                time.sleep(max(1.0, reset - time.time()))
            elif tentativa < 3 and resposta.status_code == 503:
                time.sleep(tentativa)  # 1 s, depois 2 s
            else:
                raise erro_da_resposta(resposta)
    raise AssertionError("inalcançável")
```

### TypeScript (fetch)

```ts
const API_URL = 'https://api.simplesapi.com.br'
const PLATE_RE = /^[A-Z]{3}[0-9][A-Z0-9][0-9]{2}$/

export class SimplesApiError extends Error {
  constructor(public status: number, public code: string, message: string) {
    super(`${status} ${code}: ${message}`)
  }
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms))

/** O erro da API, ou HTTP_ERROR quando o corpo não é o JSON dela (um 502 de proxy, por exemplo) */
async function erroDaResposta(resposta: Response): Promise<SimplesApiError> {
  const texto = await resposta.text()
  try {
    const corpo = JSON.parse(texto)
    return new SimplesApiError(resposta.status, corpo.code, corpo.message)
  } catch {
    return new SimplesApiError(resposta.status, 'HTTP_ERROR', texto.slice(0, 200))
  }
}

/** Dados do veículo pela placa, ou null quando a placa não tem dados (404, cobrado). Rode só no servidor. */
export async function consultarPlaca(placa: string): Promise<Record<string, unknown> | null> {
  const plate = placa.replace(/[^A-Za-z0-9]/g, '').toUpperCase()
  if (!PLATE_RE.test(plate)) throw new Error(`Placa inválida: ${plate}`)

  for (let tentativa = 1; tentativa <= 3; tentativa++) {
    const resposta = await fetch(`${API_URL}/v1/vehicle/plate`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.SIMPLES_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ plate }),
      signal: AbortSignal.timeout(30_000),
    })
    if (resposta.ok) return resposta.json()
    if (resposta.status === 404) return null
    if (tentativa < 3 && resposta.status === 429) {
      const reset = Number(resposta.headers.get('X-RateLimit-Reset') ?? Date.now() / 1000 + 1)
      await sleep(Math.max(1000, reset * 1000 - Date.now()))
    } else if (tentativa < 3 && resposta.status === 503) {
      await sleep(1000 * tentativa) // 1 s, depois 2 s
    } else {
      throw await erroDaResposta(resposta)
    }
  }
  throw new Error('inalcançável')
}
```

## Alternativa: servidor MCP

Para um agente de IA usar a API direto, sem escrever código, existe o servidor MCP:

- URL: `https://api.simplesapi.com.br/mcp`, transporte Streamable HTTP, com a mesma key no header `Authorization: Bearer`.
- Ferramentas: `lookup_vehicle_by_plate` (consulta pela placa), `read_plates_from_image` (placas numa imagem, por `image_url` https ou `image_base64`) e `get_balance` (saldo, grátis).
- A cobrança é igual à dos endpoints REST.

No Claude Code, o plugin instala esta skill e o servidor MCP juntos, e pede a key na instalação:

```
/plugin marketplace add via-x-logistica/simples-api-plugins
/plugin install simples-api@simples-api
```

Ou adicione só o servidor MCP:

```bash
claude mcp add --transport http simples-api https://api.simplesapi.com.br/mcp --header "Authorization: Bearer $SIMPLES_API_KEY"
```

O passo a passo para Cursor, VS Code, Windsurf, n8n e as APIs da Anthropic e da OpenAI está em https://simplesapi.com.br/mcp.

## O que a resposta não traz

A consulta devolve dados do veículo, não do dono: nada de nome, CPF, telefone, e-mail ou endereço do proprietário. Também não traz multas, débitos, IPVA, restrições, histórico de roubo, chassi nem RENAVAM. Assim passa menos dado pessoal de terceiros pelo sistema de quem integra.

As imagens enviadas ao OCR são processadas em memória e descartadas.

## Contato

contato@simplesapi.com.br

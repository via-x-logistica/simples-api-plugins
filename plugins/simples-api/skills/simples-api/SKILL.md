---
name: simples-api
description: Integra a Simples API (api.simplesapi.com.br), API brasileira de dados de veículos. Use quando precisar consultar um veículo pela placa brasileira (Mercosul ou antiga) e obter marca, modelo, versão, ano, cor, combustível, município e UF, o preço FIPE com o histórico, a ficha técnica ou a bateria indicada; quando precisar ler placas de veículos em fotos (OCR); quando precisar calcular os pedágios de uma rota no Brasil (as praças, a tarifa para o veículo e o total, com rota de caminhão e ônibus que respeita as restrições a pesados); ou quando precisar conferir o saldo da conta. Ensina a autenticação, os endpoints, os campos da resposta, os erros, o que é cobrado e as boas práticas, com exemplos em curl, Python e TypeScript.
---

# Simples API

A Simples API devolve dados de veículos brasileiros a partir da placa em texto ou de uma foto, e calcula os pedágios de uma rota. A base é `https://api.simplesapi.com.br`, as respostas são JSON e a cobrança é por plano mensal ou avulso pré-pago, por chamada.

- Documentação completa: https://simplesapi.com.br/docs
- Preços: https://simplesapi.com.br/precos
- Servidor MCP para agentes: https://simplesapi.com.br/mcp
- Página e playground do pedágio: https://simplesapi.com.br/api-pedagio
- Calculadora de pedágio pública (grátis, sem cadastro, 20 cálculos por dia): https://simplesapi.com.br/calculadora-pedagio

## Quando usar

Use esta API para:

- Consultar um veículo pela placa: marca, modelo, versão, ano de fabricação e ano modelo, cor, combustível, tipo, espécie, passageiros, cilindradas, eixos, município, UF e origem.
- Obter, pela mesma consulta, a versão da tabela FIPE com o preço do mês e o valor no mesmo mês dos anos anteriores, a ficha técnica (potência, torque, câmbio, consumo, pesos, dimensões) e a bateria indicada (Moura e Heliar).
- Ler placas em fotos (OCR), com a confiança de cada leitura e, se pedir, os mesmos dados do veículo de cada placa lida.
- Calcular os pedágios de uma rota: a rota entre origem, paradas e destino, as praças de pedágio com a tarifa de cada uma para o veículo (carro, moto, caminhão ou ônibus, pelos eixos ou pela placa) e o total. Caminhão e ônibus recebem uma rota que desvia das restrições a pesados em vigor na hora da passagem.

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
    "level": "exact", "code": "005490-9", "brand": "VW - VolksWagen",
    "name": "Gol 1.0 Flex 12V 5p", "model_year": 2021, "zero_km": false, "fuel": "Flex",
    "price": 46837.0, "price_month": "2026-11",
    "specs": {"engine": "1.0 12V", "power_cv": {"etanol": 84, "gasolina": 77}, "transmission": "manual", "body": "hatch", "doors": 5},
    "price_history": [
      {"month": "2020-11", "price": 43971.0}, {"month": "2021-11", "price": 55066.0},
      {"month": "2022-11", "price": 55488.0}, {"month": "2023-11", "price": 50997.0},
      {"month": "2024-11", "price": 49969.0}, {"month": "2025-11", "price": 46966.0},
      {"month": "2026-11", "price": 46837.0}
    ],
    "alternatives": []
  },
  "battery": {
    "options": [
      {"maker": "moura", "code": "M50ED", "alt_code": "Z50ED", "start_stop": null, "level": "exact",
       "ah": 50, "cca": 390, "polarity": "direita", "technology": "convencional",
       "length_mm": 210, "width_mm": 175, "height_mm": 190, "weight_kg": 12.74},
      {"maker": "heliar", "code": "HE50GD", "alt_code": null, "start_stop": null, "level": "exact",
       "ah": 50, "cca": 420, "polarity": null, "technology": "convencional",
       "length_mm": 207, "width_mm": 175, "height_mm": 190, "weight_kg": null}
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

### POST /v1/toll/route: pedágios de uma rota

Corpo:

```json
{
  "origin": {"lat": -23.667, "lng": -46.579},
  "destination": {"lat": -23.9608, "lng": -46.3336},
  "waypoints": [],
  "vehicle": {"type": "car", "axles": 2},
  "departure_at": "2026-11-19T16:43:00-03:00",
  "tag": false
}
```

- `origin` e `destination`: coordenadas WGS84. `waypoints` são as paradas no caminho, na ordem, cada uma com `lat` e `lng`.
- `vehicle`: informe `type` (`car`, `motorcycle`, `truck` ou `bus`) e `axles`, ou a `plate`. Pela placa, o tipo e os eixos vêm da consulta veicular, e `axles` substitui os eixos da consulta (a placa do cavalo mecânico não diz os eixos da carreta). `articulated` (opcional) diz se o veículo é articulado, para as restrições que separam articulado de rígido.
- Caminhão e ônibus podem mudar os eixos cobrados numa parada: `axles` em `waypoints[]` vale "a partir daqui" (um caminhão de 5 eixos que descarrega em Campinas e segue com 3). Em carro e moto, `waypoints[].axles` dá 422.
- `departure_at`: data e hora da partida em ISO 8601 (sem fuso, vale horário de Brasília; sem o campo, vale agora). Define a data das tarifas e a hora em que o veículo passa em cada praça e pórtico, para as faixas de horário, o free flow e as restrições com janela. Aceita de 90 dias atrás até 60 dias à frente.
- `tag`: com `true`, aplica o desconto TAG do free flow e das praças que o têm.

Resposta (200), de São Bernardo do Campo a Santos, de carro:

```json
{
  "distance_meters": 57750,
  "duration_seconds": 2982,
  "geometry": [{"lat": -23.66718, "lng": -46.578946}, {"lat": -23.667215, "lng": -46.579086}],
  "vehicle": {
    "category": "car_2", "type": "car", "axles": 2, "articulated": false,
    "plate": null, "vehicle_info": null,
    "stretches": [{"from_point": 0, "to_point": 1, "axles": 2, "category": "car_2"}]
  },
  "routing_profile": "car",
  "legal": true,
  "restrictions": [],
  "departure_at": "2026-11-19T16:43:36-03:00",
  "total_cents": 4060,
  "currency": "BRL",
  "complete": true,
  "tolls": [
    {
      "type": "plaza", "id": "887c9792-1c79-4c2f-ac15-6ab36fc30417", "name": "Riacho Grande",
      "direction": null, "operator": "Ecovias Imigrantes", "road": "SP-150;BR-050", "km": 31.106,
      "state": "SP", "lat": -23.7893825, "lng": -46.5196759,
      "price_cents": 4060, "category": "car_2", "axles": 2, "valid_from": "2026-07-01",
      "free_flow": null, "charged_at": null, "band": null, "tag_discount": null
    }
  ],
  "warnings": []
}
```

Como ler a resposta:

- `geometry` traz todos os pontos da rota (1.375 neste exemplo; acima estão só os dois primeiros), para desenhar no mapa.
- `tolls`: as praças e as passagens de free flow, na ordem da rota. `price_cents` é a tarifa para a categoria do trecho (`category`, `axles`) e vem `null` quando a praça está sem tarifa vigente. Um item com `type: "free_flow"` (Dutra, na Região Metropolitana de São Paulo) traz em `free_flow` a entrada, a saída e os km, e em `charged_at` a hora da passagem; nos pórticos da Rio-Santos, `band` diz a faixa de preço usada (`standard` em dia útil, `weekend` em fim de semana e feriado).
- `total_cents` soma tudo. `complete` vem `false` quando alguma praça ficou sem tarifa para a categoria ou está com a cobrança suspensa (aviso `TOLL_CHARGE_SUSPENDED` em `warnings`), e aí a rota não é cobrada. O aviso `TOLL_PRICE_UNDER_REVIEW` diz que a fonte do preço de uma praça está em revisão; o preço é mantido e a rota segue completa.
- `routing_profile` e `legal`: caminhão e ônibus são roteados com perfil próprio (dimensões típicas pelos eixos cobrados) e desviam das restrições a pesados em vigor na hora em que passariam por elas. `restrictions[]` lista as regras que moldaram a rota (`honored: true`) e as que a rota ainda atravessa (`honored: false`), cada uma com `kind`, `road`, `city`, `state`, `window` e `message`. Quando não há caminho que respeite as regras, vem a rota de um carro com `legal: false`, o aviso `TRUCK_RULE_UNAVOIDABLE` e sem cobrança. Carro e moto sempre vêm com `restrictions: []`.
- `vehicle.stretches` numera os trechos pelos pontos do pedido: 0 é a origem, 1 a n as paradas, n+1 o destino. Cada item de `tolls` diz a categoria e os eixos cobrados no seu trecho.

O mesmo trajeto com `{"type": "truck", "axles": 5}` devolve `routing_profile: "truck"`, `category: "commercial_5"` e R$ 203,00 em Riacho Grande, descendo pela Anchieta, porque a Imigrantes proíbe pesados na descida.

Custa R$ 0,10 por rota calculada com todas as praças tarifadas (`complete: true`) e legal para o veículo (`legal: true`), inclusive rota sem pedágio. Com `plate`, mais R$ 0,10 pelos dados do veículo, no mesmo lançamento (R$ 0,20 no total). Rota incompleta ou sem caminho legal não é cobrada; com `plate`, cobra só os R$ 0,10 dos dados, que vêm na resposta. Sem rota entre os pontos (404), dados inválidos (422) e cálculo indisponível (503) não são cobrados.

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
| 404 | `ROUTE_NOT_FOUND` | Não | Nenhuma rota entre os pontos. Conferir as coordenadas (dentro do Brasil, perto de uma via). |
| 422 | `VALIDATION_ERROR` | Não | Placa ou corpo inválido. Validar a placa antes de chamar. |
| 422 | `OCR_FAILED` | Não | Nenhuma placa lida na foto. Pedir outra foto, mais nítida ou mais próxima. |
| 422 | `VEHICLE_CATEGORY_UNDETERMINED` | Não | A placa não define a categoria do veículo. Repetir com `type` e `axles`. |
| 422 | `WAYPOINT_AXLES_NOT_ALLOWED` | Não | `waypoints[].axles` só vale para caminhão e ônibus. Tirar o campo. |
| 429 | `RATE_LIMIT_EXCEEDED` | Não | Passou do limite por minuto. Esperar até `X-RateLimit-Reset` e seguir. |
| 503 | `VEHICLE_API_UNAVAILABLE` | Não | Consulta indisponível. Tentar de novo com espera crescente, no máximo 3 tentativas. |
| 503 | `OCR_UNAVAILABLE` | Não | Leitura indisponível. Tentar de novo com espera crescente, no máximo 3 tentativas. |
| 503 | `ROUTING_UNAVAILABLE` | Não | Cálculo de rota indisponível. Tentar de novo com espera crescente, no máximo 3 tentativas. |

## Limites

O limite é de 100 requisições por minuto por conta. As respostas trazem os headers `X-RateLimit-Limit`, `X-RateLimit-Remaining` e `X-RateLimit-Reset`, este último um timestamp Unix de quando a janela recomeça.

## Preços

Há dois jeitos de pagar, na mesma conta e com a mesma API key: o plano mensal e o avulso pré-pago.

- Planos mensais, no cartão de crédito, com cobrança mensal e cancelamento quando quiser: Básico, 250 chamadas por R$ 22,90; Pro, 1.000 chamadas por R$ 79,90; Escala, 3.000 chamadas por R$ 209,90; Personalizado, de 5.000 a 200.000 chamadas por mês a R$ 0,06 cada. Cada consulta pela placa e cada rota de pedágio calculada gastam 1 chamada; a leitura de placa em foto vem incluída até o tamanho da franquia. O valor que você não usar vira crédito, que não expira e é gasto no avulso.
- Avulso, pré-pago e sem mensalidade, com os preços abaixo.

- Leitura de placas (`POST /v1/ocr/plate`): R$ 0,05 por placa lida.
- Dados do veículo junto com a leitura (`"include_vehicle_info": true`): mais R$ 0,10 por placa consultada, R$ 0,15 no total.
- Consulta pela placa (`POST /v1/vehicle/plate`): R$ 0,10 por consulta, com ou sem dados retornados.
- Pedágios de uma rota (`POST /v1/toll/route`): R$ 0,10 por rota calculada com todas as praças tarifadas e legal para o veículo, inclusive rota sem pedágio; com placa, mais R$ 0,10 pelos dados do veículo.
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
9. **Na rota de pedágio**, mande `departure_at` quando a viagem não é agora: as faixas de horário, o free flow e as restrições a pesados dependem da hora da passagem. Trate `complete: false` e `legal: false` como "sem preço fechado", nunca como total zero.

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

# Pedágios de São Bernardo do Campo a Santos para um caminhão de 5 eixos
curl -X POST "https://api.simplesapi.com.br/v1/toll/route" \
  -H "Authorization: Bearer $SIMPLES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"origin": {"lat": -23.667, "lng": -46.579}, "destination": {"lat": -23.9608, "lng": -46.3336}, "vehicle": {"type": "truck", "axles": 5}}'
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
- Ferramentas: `lookup_vehicle_by_plate` (consulta pela placa), `read_plates_from_image` (placas numa imagem, por `image_url` https ou `image_base64`), `get_balance` (saldo, grátis) e `calculate_toll_route` (pedágios de uma rota: `origin`, `destination`, `waypoints`, `vehicle` com `type` e `axles` ou `plate`, `departure_at` e `tag`, como em POST /v1/toll/route; a resposta vem sem a geometria).
- A cobrança é igual à dos endpoints REST: a rota de pedágio custa R$ 0,10 quando sai completa e legal para o veículo.

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

A rota de pedágio é uma estimativa: as tarifas vêm das tabelas oficiais das concessionárias e dos poderes concedentes, e a rota e as restrições dependem do mapa (OpenStreetMap) e das regras publicadas pelos órgãos.

## Contato

contato@simplesapi.com.br

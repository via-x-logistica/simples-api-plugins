# Plugin da Simples API para o Claude Code

Consulte veículos brasileiros pela placa, calcule os pedágios de uma rota e leia placas em fotos direto do Claude Code. O plugin traz:

- **A skill `simples-api`**, que ensina a API REST da [Simples API](https://simplesapi.com.br): os endpoints, os campos da resposta, os erros, o que é cobrado e as boas práticas, com exemplos em curl, Python e TypeScript. É o mesmo arquivo publicado em https://simplesapi.com.br/skill.md.
- **O servidor MCP da Simples API**, já configurado com a sua API key, com as ferramentas `lookup_vehicle_by_plate`, `read_plates_from_image`, `get_balance` e `calculate_toll_route`.

A consulta pela placa traz marca, modelo, versão, ano, cor, combustível, município e UF e, quando temos, o preço FIPE com o histórico, a ficha técnica e a bateria indicada. Ela devolve dados do veículo, não do dono: nada de nome, CPF, telefone, e-mail ou endereço do proprietário.

## Instalar

No Claude Code:

```
/plugin marketplace add via-x-logistica/simples-api-plugins
/plugin install simples-api@simples-api
```

Na instalação, o Claude Code pede a sua API key. Ela fica no painel da Simples API, na aba MCP (https://simplesapi.com.br/dashboard/ia). Quem ainda não tem conta pode criar uma em https://simplesapi.com.br/register; a conta nova ganha R$ 5,00 de crédito grátis, sem cartão.

Para testar, peça: "Qual é o meu saldo na Simples API?". A consulta de saldo é grátis.

## Trocar a API key

No Claude Code, rode e cole a nova chave:

```
/plugin configure simples-api@simples-api
```

Ou, pelo terminal (a chave é lida sem aparecer na tela nem ficar no histórico do shell):

```bash
printf 'API key: '; read -rs CHAVE; echo; printf '{"simples_api_key": "%s"}' "$CHAVE" | claude plugin configure simples-api@simples-api --values-stdin; unset CHAVE
```

Depois, reinicie o Claude Code.

Com a chave errada, o Claude Code não carrega as ferramentas: em `/mcp`, o servidor `simples-api` aparece com falha (HTTP 401). Troque a chave como acima.

## Preços

As chamadas pelo plugin custam o mesmo que a API REST: R$ 0,10 por consulta pela placa, R$ 0,10 por rota de pedágio calculada com todas as praças tarifadas e legal para o veículo, e R$ 0,05 por placa lida numa foto (mais R$ 0,10 por placa consultada, quando os dados do veículo são pedidos). A consulta de saldo é grátis. Os preços em vigor ficam em https://simplesapi.com.br/precos.

## Outros agentes

Usa o Cursor, o VS Code, o Windsurf, o n8n ou um agente próprio? O passo a passo para cada um está em https://simplesapi.com.br/mcp. Para agentes sem MCP, peça para ler https://simplesapi.com.br/skill.md antes de integrar.

## Contato

contato@simplesapi.com.br

Este repositório é gerado a partir do repositório principal da Simples API. Mudanças feitas direto aqui são sobrescritas na próxima publicação.

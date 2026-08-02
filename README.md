# Monitor de Passagens Aéreas

Workflow n8n que monitora preços de passagens de Porto Alegre (POA) para Amsterdã (AMS) e envia alerta por e-mail quando encontra tarifas dentro do critério definido.

## Como funciona

```
Schedule Trigger (de hora em hora)
        ↓
HTTP Request  →  API Travelpayouts (Aviasales)
        ↓
Code (JavaScript)  →  transforma, filtra e monta o HTML
        ↓
Gmail  →  envia o alerta
```

O node **Code** faz o trabalho pesado:

- achata a resposta aninhada da API em uma lista plana de voos;
- ordena por preço, do mais barato para o mais caro;
- filtra por datas de saída e teto de preço;
- traduz códigos IATA de companhias (`LA` → LATAM, `KL` → KLM, etc.);
- monta um e-mail HTML com links para Skyscanner e Aviasales.

Quando nada é encontrado, o workflow fica em silêncio — exceto à meia-noite, quando envia uma confirmação de que o monitor continua funcionando. Assim dá para distinguir "não há promoção" de "a automação quebrou".

## Pré-requisitos

- n8n (testado via Docker)
- Token gratuito da [Travelpayouts](https://www.travelpayouts.com)
- Credencial OAuth do Gmail, criada no Google Cloud Console

## Instalação

1. Importe o JSON no n8n: `⋯` → **Import from File**
2. No node **HTTP Request**, substitua `SEU_TOKEN_AQUI` pelo seu token da Travelpayouts
3. No node **Gmail**, configure a credencial OAuth e o e-mail de destino
4. Ative o workflow

### Rodando o n8n via Docker

```bash
docker run -d --name n8n --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --dns 8.8.8.8 --dns 1.1.1.1 \
  -e GENERIC_TIMEZONE="America/Sao_Paulo" \
  -e TZ="America/Sao_Paulo" \
  docker.n8n.io/n8nio/n8n
```

As flags `--dns` são necessárias em ambientes onde o container não consegue resolver nomes de domínio externos.

## Personalização

| O que mudar | Onde |
|---|---|
| Origem e destino | Parâmetros `origin` e `destination` na URL do HTTP Request |
| Datas monitoradas | Filtro `baratas` no Code node |
| Teto de preço | Comparação `v.preco < 9000` no Code node |
| Frequência | Schedule Trigger |

## Limitações

A API retorna **dados de cache**, montados a partir das buscas de usuários da Aviasales nos últimos 7 dias. Não são preços em tempo real, e rotas pouco procuradas podem passar dias sem atualização — ou retornar vazio. Sempre confirme o valor no site antes de comprar.

O acesso a APIs de busca em tempo real (Aviasales Flight Search, Skyscanner) exige contrato de parceria com volume mínimo de usuários, inviável para projetos pessoais.

## Notas de segurança

- O token vai na URL do HTTP Request. Antes de publicar o JSON, substitua-o por um placeholder.
- Alternativa mais segura: ativar **Send Query Parameters** no node e cadastrar `token` como campo separado, isolando a credencial do restante da URL.
- Credenciais OAuth do n8n (Gmail, Google) não são incluídas no export — apenas o ID e o nome da credencial.

## Aprendizados

Alguns problemas enfrentados durante a construção, caso ajudem outra pessoa:

- **DNS no container**: o n8n não resolvia domínios externos mesmo com a URL funcionando no navegador. Diagnóstico com `docker exec n8n ping <domínio>`; solução com as flags `--dns`.
- **Dados "pinados"**: um node com o ícone de alfinete não executa e devolve sempre o mesmo resultado congelado. Tecla `P` remove.
- **Amadeus Self-Service**: descontinuada em 17/07/2026, o que invalida boa parte dos tutoriais de integração de voos disponíveis online.
- **Cotas de modelos preview**: versões preview de LLMs têm limites diários muito baixos; modelos estáveis são a escolha certa fora de testes.

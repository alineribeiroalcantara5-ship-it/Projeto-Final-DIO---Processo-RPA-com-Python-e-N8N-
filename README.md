# RPA com Python e n8n: Assistente de Investimentos com IA

Automação que lê uma lista de clientes com **Python (RPA)**, cruza o perfil de cada um com uma tabela de produtos de investimento usando o **n8n**, pede a uma **IA (Claude, da Anthropic)** que escreva um e-mail personalizado e envia a mensagem pelo **Gmail**, com validação antes do envio e aviso em caso de falha.

> A IA **sugere** produtos com base nos dados recebidos, mas **nunca promete ou garante ganhos**. Os e-mails têm caráter informativo.

Projeto final baseado no laboratório [Assistente de Investimentos com RPA e n8n](https://digitalinnovationone.github.io/dio-lab-assistente-investimentos-rpa-n8n) da DIO.

---

## O que o projeto faz

1. Um script Python (RPA) lê a tabela de clientes de uma página web e envia os dados ao n8n por **webhook**.
2. O n8n baixa a tabela de produtos (CSV) e junta com os clientes.
3. Um código normaliza os dados (saldo em reais vira número) e cruza cliente e produtos pelo **perfil** (Conservador, Moderado ou Arrojado).
4. A IA escreve um e-mail curto em português do Brasil para cada cliente, usando **somente** os dados recebidos.
5. Um código valida o texto gerado. Só e-mails válidos são enviados.
6. O Gmail envia o e-mail. Se o envio for bloqueado ou falhar, um aviso chega ao administrador com o motivo.

## Arquitetura do fluxo

```
Python (RPA)
     |
  Webhook ─────────────┐
     |                  |
HTTP Request (CSV)      |
     |                  |
Code in JavaScript      |
(CSV -> lista)          |
     |                  |
     └───── Merge ──────┘
              |
   Code in JavaScript1
     (normalização)
              |
      Message a model
          (IA)
              |
       Preparar E-mail
         (validação)
              |
             If (valido?)
        |            |
      true          false
        |            |
 Send a message      |
    |       |        |
 sucesso  erro       |
    |       └────────┴──> Send a message if false e-mail (aviso)
    |
Respond to Webhook
```

## Nós do workflow

| Nó | Função |
|---|---|
| **Webhook** | Recebe `POST /Clientes` com `{"clientes": [...]}`. Responde só no final (`Respond to Webhook`). |
| **HTTP Request** | Baixa o `data.csv` com os produtos de investimento. |
| **Code in JavaScript** | Converte o texto do CSV em lista (`investimentos`) e transforma o mínimo em número. |
| **Merge** | Combina por posição o ramo do webhook com o do CSV. |
| **Code in JavaScript1** | Normaliza os dados, cruza cliente e produtos por perfil e gera o campo `texto_para_ia` com uma tabela por cliente. |
| **Message a model** | Claude (`claude-haiku-4-5`) escreve o e-mail de cada cliente. |
| **Preparar E-mail** | Extrai o texto da IA, limpa a formatação, valida e monta os campos de envio (`para`, `assunto`, `corpo_texto`, `valido`, `motivo_invalido`...). |
| **If** | Deixa seguir só os itens com `valido = true`. |
| **Send a message** | Gmail: envia o e-mail ao cliente. Tem saída de erro própria. |
| **Send a message if false e-mail** | Gmail: avisa o administrador (bloqueio do If ou falha no envio) com cliente, e-mail e motivo. |
| **Respond to Webhook** | Devolve `200` ao script Python depois de um envio com sucesso. |

## Dados

**Clientes** (enviados pelo Python):

| Campo | Exemplo |
|---|---|
| nome | Ana Silva |
| email | ana@exemplo.com |
| saldo | R$ 12.500,00 |
| perfil | Conservador |

**Produtos** (CSV): `perfil`, `produto`, `minimo`, `rentabilidade`.

**Resultado do cruzamento:** para cada cliente, uma tabela com os produtos do perfil dele e a coluna "Saldo suficiente?" (comparando o saldo com o mínimo do produto).

## Validações antes do envio

O nó **Preparar E-mail** marca `valido = false` (e o **If** bloqueia o envio) quando:

- a IA não devolveu texto;
- o texto contém expressão não resolvida (`{{ }}`);
- a IA respondeu que não recebeu os dados do cliente;
- o e-mail do cliente está ausente ou com formato inválido.

O motivo fica em `motivo_invalido` e vai no aviso ao administrador.

## Requisitos

- [n8n](https://n8n.io) (Cloud ou instalado localmente)
- Material de apoio que pode ser Forkado do repositório público da DIO para acesso a lista de clientes e ao código-fonte RPA (Pyhton) para funcinamento da aplicação.
- Link do material de apoio : https://colab.research.google.com/github/alineribeiroalcantara5-ship-it/dio-lab-assistente-investimentos-rpa-n8n/blob/main/rpa/extrair_clientes.ipynb#scrollTo=l1P5k7woKJLI
- Python 3 com `requests` e `beautifulsoup4`
- Conta Google com credencial **OAuth2 do Gmail** (criada no Google Cloud)
- Chave de API da **Anthropic**, cadastrada como credencial no n8n
- [ngrok](https://ngrok.com), se o n8n rodar na sua máquina e precisar receber o webhook

## Como configurar

### 1. Importar o workflow

No n8n: **Workflows → Import from File** e selecione `workflow.json`.

### 2. Credenciais

Crie e selecione nos nós:

- **Anthropic** no nó *Message a model*;
- **Gmail OAuth2** nos dois nós de e-mail.



### 3. Ajustar o aviso

No nó **Send a message if false e-mail**, troque o campo *To* pelo e-mail que deve receber os avisos.

### 4. Script Python

O script lê a tabela de clientes e envia ao webhook. Ajuste só a URL:

```python
N8N_WEBHOOK = "https://SEU_ENDERECO/webhook/Clientes"
response = requests.post(N8N_WEBHOOK, json={"clientes": clientes}, timeout=30)
```

- **Teste:** use `/webhook-test/Clientes`. Clique em **Execute workflow** no n8n **antes** de rodar o Python.
- **Produção:** ative o workflow e use `/webhook/Clientes`.

## Como testar

1. No campo *To* do nó **Send a message**, coloque o **seu próprio e-mail** em vez de `{{ $json.para }}`, para não enviar a ninguém por engano.
2. Execute o workflow e rode o script Python.
3. Confira se chegou um e-mail por cliente, citando só produtos do perfil dele, sem `{{ }}` e sem prometer ganhos.
4. Depois de aprovar, volte o campo *To* para `{{ $json.para }}`.

> O Gmail marca "sucesso" quando o Google **aceita** a mensagem, e não quando ela é entregue. Endereços inexistentes geram um e-mail de retorno ("Mail Delivery Subsystem") na caixa de entrada do remetente.

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| A IA pede os dados ou devolve `{{ }}` no texto | O campo do prompt está em *Fixed*, e não em *Expression* |
| Só o primeiro cliente aparece na saída de um Code | Modo do nó (*Each Item* / *All Items*) não combina com o código |
| O If reclama que o valor é string | Campo em *Fixed*, ou condição apontando para outro campo (deve ser `valido`) |
| Erro `403 access_denied` no login Google | Conta fora da lista de *Usuários de teste* |
| `redirect_uri_mismatch` | A URL do ngrok mudou e não foi atualizada no Google Cloud |
| O webhook não recebe nada | Faltou *Listen for test event*, ou o ngrok está desligado ou com outra URL |

## Segurança e privacidade

### Webhook sem autenticação

O webhook deste projeto **não usa autenticação**. Isso é intencional: o projeto segue o laboratório da DIO, que não exige autenticação no webhook, e usa apenas **dados fictícios** em ambiente de estudo.

Em consequência, quem souber a URL do webhook pode enviar dados a ele, e o fluxo enviaria e-mails pela conta do Gmail conectada e usaria créditos da API da Anthropic. Por isso, durante os estudos:

- mantenha o workflow **inativo** quando não estiver testando;
- mantenha o **ngrok desligado** fora dos testes;
- nunca use dados reais de clientes.



### Boas práticas

- **Nunca publique** chaves de API, segredos OAuth, URLs do ngrok ou e-mails reais. O `workflow.json` exportado do n8n não leva os segredos das credenciais, mas confira por e-mails, `instanceId` e dados fixados (`pinData`) antes de subir ao repositório.
- Use dados **fictícios** em testes e prefira domínios de exemplo (como `exemplo.com`), porque endereços como `@email.com` podem existir de verdade.
- Os dados dos clientes (nome, e-mail, saldo) são enviados à API da Anthropic para gerar o texto. Em uso real com dados pessoais, é preciso base legal e aviso aos titulares, conforme a **LGPD**.

## Limitações conhecidas

- **Sem autenticação no webhook** (veja a seção acima).
- **Resposta ao webhook.** O *Respond to Webhook* só roda após um envio com sucesso. Se todos os itens forem bloqueados ou falharem, o Python fica esperando até o `timeout`.
- **Sem retentativa automática.** Falhas passageiras da API da IA ou do Gmail não são refeitas. Dá para ativar *Retry On Fail* em **Settings** de cada nó.
- **Aviso pelo mesmo Gmail.** Se a falha for da própria credencial, o aviso também não sai. Um canal separado (como Telegram) resolveria.
- **Perfis e produtos** vêm de um CSV simples separado por vírgula; o `minimo` considera só números inteiros.

## Estrutura do repositório

```
.
├── README.md
├── workflow.json          # exportado do n8n (sem dados pessoais)
├── scripts/
│   └── enviar_clientes.py # RPA que lê os clientes e chama o webhook
└── docs/
    ├── fluxo.png          # print do workflow no n8n
    └── exemplo-email.png  # exemplo de e-mail gerado (dados fictícios)
```

## O que este projeto adiciona ao laboratório base

- Normalização do saldo em reais e cruzamento por perfil, ignorando maiúsculas e acentos.
- Tabela de produtos por cliente com a coluna "Saldo suficiente?".
- Prompt com regras para a IA (só dados recebidos, sem promessa de ganhos, máximo de 120 palavras).
- Validação do texto gerado e **If** antes do envio.
- Saída de erro no Gmail e aviso ao administrador com o motivo.

## Tecnologias

Python · n8n · JavaScript · API da Anthropic (Claude) · Gmail API (OAuth2) · Webhook · ngrok

## Licença

Distribuído sob a licença MIT. O conteúdo do laboratório original (página de clientes e CSV de produtos) pertence à plataforma de ensino DIO.

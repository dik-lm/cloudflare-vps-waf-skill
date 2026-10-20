---
name: protecao-vps-cloudflare
description: >-
  Use quando o usuário quiser proteger painéis e serviços de uma VPS com Cloudflare
  Custom Rules, especialmente n8n, EasyPanel, Evolution API, WAHA, Portainer e APIs
  self-hosted. A lógica principal é: se houver IP fixo/confiável, usar uma regra BLOCK
  em formato allowlist; se não houver, bloquear acessos estrangeiros aos hosts protegidos
  e aplicar Non-Interactive Challenge ao restante. O IP público da própria VPS é sempre
  liberado nos hosts protegidos para não quebrar chamadas entre serviços via URLs públicas.
  Integrações externas máquina-a-máquina devem ser identificadas e excluídas dos challenges.
  Entregue no máximo 1 ou 2 regras e encerre.
---

# Proteção simples de VPS com Cloudflare

## Objetivo

Princípio:

> Se não precisa estar público, não deixe público.

Esta skill protege **subdomínios administrativos e serviços self-hosted** sem quebrar chamadas entre aplicações.

Exemplos:

- EasyPanel
- n8n
- Evolution API
- WAHA
- Portainer
- painéis internos
- APIs privadas

A lógica foi pensada para ambientes em que vários serviços estão na **mesma VPS** e podem se chamar usando seus próprios subdomínios públicos.

Por isso:

> **O IP público da própria VPS entra sempre como exceção nos hosts protegidos.**

Se n8n chamar Evolution, ou Evolution chamar n8n, pela URL pública, a chamada continua funcionando.

Se a VPS já estiver comprometida internamente, esta regra de borda não é a camada destinada a resolver esse problema.

---

# Fluxo

Siga esta ordem:

1. coletar dados;
2. separar hosts públicos dos protegidos;
3. identificar chamadas externas à VPS;
4. escolher Modo A ou Modo B;
5. montar a(s) regra(s);
6. validar;
7. entregar e encerrar.

Não continue expandindo a configuração depois disso, salvo se o usuário pedir.

---

# 1. Perguntas

Pergunte tudo de uma vez:

```text
1. Qual é seu domínio?

2. Quais subdomínios você possui e o que roda em cada um?
   Ex.:
   easypanel.exemplo.com.br → EasyPanel
   n8n.exemplo.com.br → n8n
   evolution.exemplo.com.br → Evolution API
   www.exemplo.com.br → site público

3. Quais desses subdomínios são sites públicos normais
   (site, loja, landing page etc.)?

4. Você possui IP fixo/confiável em casa ou no escritório?
   Se sim, informe o(s) IP(s).

5. Qual é o IP público da VPS?

6. Algum sistema de FORA dessa VPS precisa acessar n8n,
   Evolution, API ou outro host protegido automaticamente?
   Ex.: Meta, Stripe, Mercado Pago, outro servidor, SaaS etc.

   Para cada integração externa, informe o que souber:
   - host acessado;
   - rota;
   - IPs de origem;
   - ASN;
   - país;
   - User-Agent;
   - se pode vir de fora do Brasil.

7. Usa EasyPanel/Let's Encrypt ou precisa preservar
   /.well-known/acme-challenge/ ?

8. Os hosts protegidos estão Proxied / nuvem laranja na Cloudflare?
```

### Importante

Chamadas entre serviços da **mesma VPS** não precisam ser mapeadas rota por rota: o IP público da VPS será liberado.

O que precisa ser identificado são principalmente as chamadas **que chegam de fora da VPS**.

Se o usuário não souber como funcionam suas integrações externas, **não force a regra restritiva**. Use o arquivo:

`references/modo-compativel.md`

---

# 2. Hosts públicos x hosts protegidos

## Públicos

Ex.:

```text
www.exemplo.com.br
loja.exemplo.com.br
```

Não entram nas regras desta skill.

## Protegidos

Ex.:

```text
easypanel.exemplo.com.br
n8n.exemplo.com.br
evolution.exemplo.com.br
```

Toda regra precisa começar escopada nesses hosts:

```text
http.host in {
    "easypanel.exemplo.com.br"
    "n8n.exemplo.com.br"
    "evolution.exemplo.com.br"
}
```

Nunca aplique a regra à zona inteira se o objetivo é proteger apenas esses serviços.

---

# 3. Exceções padrão

## IP da VPS

Sempre inclua:

```text
<IP_PUBLICO_DA_VPS>
```

entre os acessos permitidos nos hosts protegidos.

Isso evita quebrar comunicação entre serviços da própria VPS quando usam URLs públicas.

---

## Let's Encrypt / ACME

Se o usuário usa esse fluxo de validação, preserve:

```text
(
    starts_with(
        http.request.uri.path,
        "/.well-known/acme-challenge/"
    )
    and http.user_agent contains "Let's Encrypt"
)
```

Essa exceção deve ficar fora de BLOCK e de Challenge.

---

# 4. Integrações externas

Uma integração externa é uma chamada automática que **não vem da própria VPS**.

Ex.:

- Meta
- Stripe
- Mercado Pago
- SaaS
- outra VPS
- servidor de terceiro

Ela não deve receber Challenge.

Monte a exceção usando a informação mais restritiva disponível.

Preferência:

```text
host + rota + IP
```

depois:

```text
host + rota + ASN
```

depois:

```text
host + rota + país + User-Agent
```

e, somente se não houver informação melhor:

```text
host + rota
```

Nunca libere um host inteiro quando apenas uma rota precisa receber chamadas externas.

### Exemplo

```text
(
    http.host eq "automacao.exemplo.com.br"
    and (
        http.request.uri.path eq "/webhook"
        or starts_with(http.request.uri.path, "/webhook/")
    )
    and (
        ip.src in {
            203.0.113.20
            203.0.113.21
        }
        or ip.src.asnum eq 12345
    )
)
```

---

# 5. Escolha do modo

## MODO A — existe IP fixo/confiável

Use **1 regra BLOCK**.

Passam:

- IP(s) fixo(s) do usuário;
- IP da VPS;
- ACME, se necessário;
- integrações externas confirmadas.

Todo o restante nos hosts protegidos é bloqueado.

---

## MODO B — não existe IP fixo/confiável

Use **2 regras**:

```text
1. BLOCK para tráfego fora do Brasil
2. NON-INTERACTIVE CHALLENGE para o restante
```

Exceções:

- IP da VPS;
- ACME;
- integrações externas legítimas.

Uma integração externa que pode vir de fora do Brasil deve escapar das duas regras.

Uma integração externa que só deve vir do Brasil pode continuar sujeita ao geo-block, mas deve escapar do Challenge.

---

# 6. MODELO A — IP fixo / allowlist

Ação:

```text
BLOCK
```

Estrutura:

```text
(
    http.host in {
        "<HOST_PROTEGIDO_1>"
        "<HOST_PROTEGIDO_2>"
    }

    and not (
        <EXCECAO_ACME>
    )

    and not (
        ip.src in {
            <IP_FIXO_USUARIO>
            <IP_PUBLICO_VPS>
        }

        or (
            <INTEGRACAO_EXTERNA_1>
        )

        or (
            <INTEGRACAO_EXTERNA_2>
        )
    )
)
```

Remova blocos de integrações que não existirem.

Se ACME não for necessário, remova o bloco inteiro:

```text
and not (
    <EXCECAO_ACME>
)
```

Nunca deixe placeholder ou operador órfão na regra final.

---

# 7. MODELO B — sem IP fixo

## REGRA 1 — Bloquear fora do Brasil

Ação:

```text
BLOCK
```

Estrutura:

```text
(
    http.host in {
        "<HOST_PROTEGIDO_1>"
        "<HOST_PROTEGIDO_2>"
    }

    and ip.src.country ne "BR"

    and not (
        <EXCECAO_ACME>
    )

    and ip.src ne <IP_PUBLICO_VPS>

    and not (
        <INTEGRACOES_EXTERNAS_QUE_PODEM_VIR_DE_FORA_DO_BRASIL>
    )
)
```

Se não houver integração estrangeira, remova o último bloco.

---

## REGRA 2 — Challenge para quem sobrou

Ação:

```text
NON-INTERACTIVE CHALLENGE
```

Estrutura:

```text
(
    http.host in {
        "<HOST_PROTEGIDO_1>"
        "<HOST_PROTEGIDO_2>"
    }

    and not (
        <EXCECAO_ACME>
    )

    and ip.src ne <IP_PUBLICO_VPS>

    and not (
        <TODAS_AS_INTEGRACOES_EXTERNAS_LEGITIMAS>
    )
)
```

Essa regra pega principalmente o acesso humano restante aos painéis.

Chamadas automáticas legítimas ficam fora dela.

### Ordem

```text
1. BLOCK
2. NON-INTERACTIVE CHALLENGE
```

Não inverta.

---

# 8. Rotas

Para rota exata:

```text
http.request.uri.path eq "/callback"
```

Para `/webhook/*`:

```text
(
    http.request.uri.path eq "/webhook"
    or starts_with(
        http.request.uri.path,
        "/webhook/"
    )
)
```

Não use apenas:

```text
starts_with(http.request.uri.path, "/webhook")
```

Sempre combine rota com o host correto.

---

# 9. Validação

Antes de entregar, confirme mentalmente:

```text
[ ] site público não foi incluído;
[ ] IP do usuário passa, se houver whitelist;
[ ] IP da VPS passa sempre;
[ ] ACME passa, se necessário;
[ ] chamadas entre serviços da própria VPS passam;
[ ] integrações externas legítimas não recebem Challenge;
[ ] integrações estrangeiras necessárias não caem no geo-block;
[ ] restante dos hosts protegidos é bloqueado/desafiado como planejado.
```

Se algo falhar, corrija antes de responder.

---

# 10. Entrega

Seja curto.

Entregue:

```text
Estratégia escolhida

Regra 1
- nome;
- ação;
- expressão.

Regra 2
- apenas no modo sem IP fixo.

Ordem
- apenas quando houver duas regras.
```

Depois explique em poucas linhas quem passa e quem não passa.

Finalize com:

```text
[ ] hosts protegidos estão Proxied / nuvem laranja
[ ] IP da VPS conferido
[ ] IP fixo conferido, se houver
[ ] integrações externas conferidas
[ ] regras testadas
```

Depois encerre.

---

# Regras invioláveis

1. O IP público da VPS é sempre liberado nos hosts protegidos.
2. Hosts públicos ficam fora das regras.
3. Nunca invente IP, ASN, rota ou hostname.
4. Integração máquina-a-máquina legítima não recebe Challenge.
5. Só abra a rota necessária, não o host inteiro.
6. Se uma integração estrangeira precisa funcionar, exclua-a do geo-block.
7. BLOCK vem antes de Non-Interactive Challenge.
8. Use no máximo 1 regra com whitelist ou 2 sem whitelist.
9. Se o ambiente estiver mal mapeado, use `references/modo-compativel.md`.
10. Entregou e testou: encerre.

---

# Princípio final

```text
IP FIXO?
→ allowlist + BLOCK.

SEM IP FIXO?
→ BLOCK fora do Brasil
→ Non-Interactive Challenge no restante.

MESMA VPS?
→ IP da VPS passa.

INTEGRAÇÃO EXTERNA?
→ libere somente o necessário.

NÃO PRECISA ENTRAR?
→ BLOCK.
```

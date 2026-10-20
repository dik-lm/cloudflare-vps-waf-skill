# Modo compatível

Use este modo quando o ambiente ainda não estiver suficientemente mapeado para aplicar uma política restritiva com segurança.

O objetivo é:

> melhorar a proteção sem quebrar automações, webhooks, APIs ou integrações externas desconhecidas.

Este modo é temporário.

Depois que as integrações externas forem identificadas, volte ao fluxo principal e use o Modo A ou Modo B.

---

# Quando usar

Use o modo compatível quando uma ou mais situações ocorrerem:

- o usuário não sabe quais serviços externos acessam a VPS;
- existem webhooks, callbacks ou APIs ainda não mapeados;
- n8n, Evolution API, WAHA ou outro serviço recebe chamadas externas;
- não se conhece IP, ASN, país ou rota das integrações;
- o mesmo host possui interface administrativa e endpoints de API;
- aplicar Challenge no host inteiro pode quebrar automações;
- aplicar bloqueio geográfico pode bloquear provedores legítimos.

Não invente exceções para tentar compensar falta de informação.

---

# Princípio

Quando houver dúvida:

> proteja apenas o que é claramente administrativo.

Não aplique BLOCK ou Challenge a um host inteiro se ele puder receber chamadas máquina-a-máquina desconhecidas.

É melhor deixar temporariamente um host fora da regra do que interromper uma integração legítima.

---

# Classificação dos hosts

Separe os hosts em três grupos.

## 1. Públicos

Ex.:

```text
www.exemplo.com.br
loja.exemplo.com.br
```

Não entram nas regras.

---

## 2. Administrativos confirmados

São hosts usados por pessoas para administração e que não precisam receber integrações externas automáticas.

Ex.:

```text
easypanel.exemplo.com.br
portainer.exemplo.com.br
```

Somente inclua um host aqui quando isso estiver confirmado.

Esses hosts podem receber proteção no modo compatível.

---

## 3. Automação, API ou situação incerta

Ex.:

```text
n8n.exemplo.com.br
evolution.exemplo.com.br
waha.exemplo.com.br
api.exemplo.com.br
```

Se esses hosts recebem ou podem receber:

- webhooks;
- callbacks;
- APIs;
- integrações externas;
- chamadas de SaaS;
- chamadas de terceiros;

e essas origens ainda não estão mapeadas, deixe-os fora das regras do modo compatível.

Não aplique Challenge ao host inteiro.

Não aplique bloqueio geográfico ao host inteiro.

---

# Regra do modo compatível

Use no máximo 1 regra.

A regra deve atingir somente os hosts administrativos confirmados.

Ação:

```text
NON-INTERACTIVE CHALLENGE
```

Estrutura:

```text
(
    http.host in {
        "<HOST_ADMINISTRATIVO_1>"
        "<HOST_ADMINISTRATIVO_2>"
    }

    and not (
        <EXCECAO_ACME>
    )

    and ip.src ne <IP_PUBLICO_VPS>
)
```

Remova hosts que não forem administrativos confirmados.

Nunca coloque n8n, Evolution API, WAHA, APIs ou outros hosts de automação nessa regra apenas porque parecem administrativos.

---

# Let's Encrypt / ACME

Se o ambiente precisar preservar validação HTTP do Let's Encrypt, use:

```text
(
    starts_with(
        http.request.uri.path,
        "/.well-known/acme-challenge/"
    )
    and http.user_agent contains "Let's Encrypt"
)
```

A exceção deve ficar fora do Challenge.

Se ACME não for necessário, remova completamente esse bloco.

Nunca deixe placeholders na regra final.

---

# IP da VPS

O IP público da própria VPS continua sendo exceção.

Sempre preserve:

```text
ip.src ne <IP_PUBLICO_VPS>
```

Isso evita que chamadas originadas pela própria VPS recebam Challenge quando utilizam URLs públicas.

---

# Hosts mistos

Alguns serviços possuem no mesmo hostname:

- painel administrativo;
- API;
- webhooks;
- callbacks;
- endpoints usados por integrações.

Nesse caso:

> não aplique Challenge ao hostname inteiro.

Só proteja uma rota administrativa se o usuário confirmar exatamente qual é essa rota e que ela não recebe tráfego máquina-a-máquina.

Exemplo conceitual:

```text
http.host eq "servico.exemplo.com.br"
and starts_with(
    http.request.uri.path,
    "/rota-administrativa/"
)
```

Não invente a rota.

Se ela não estiver confirmada, deixe o host fora da regra.

---

# O que não fazer

No modo compatível, não:

- bloquear todo tráfego fora do Brasil em hosts desconhecidos;
- aplicar Challenge em APIs ou webhooks desconhecidos;
- liberar um host inteiro para compensar uma integração;
- inventar IPs;
- inventar ASNs;
- inventar rotas;
- confiar apenas em User-Agent para identificar integrações;
- assumir que um serviço é exclusivamente administrativo;
- criar várias regras experimentais.

---

# Entrega

Explique ao usuário que o ambiente está sendo tratado em modo compatível porque ainda existem integrações externas não mapeadas.

Entregue apenas:

```text
Estratégia:
Modo compatível.

Protegidos:
- apenas hosts administrativos confirmados.

Temporariamente fora da regra:
- hosts de automação;
- APIs;
- webhooks;
- hosts com função incerta.

Regra:
- nome;
- ação;
- expressão.
```

Depois informe que os hosts deixados de fora poderão receber proteção mais restritiva quando as integrações forem identificadas.

---

# Validação

Antes de entregar:

```text
[ ] sites públicos estão fora da regra
[ ] somente hosts administrativos confirmados foram incluídos
[ ] hosts de API/automação incertos ficaram fora
[ ] IP público da VPS passa
[ ] ACME passa, se necessário
[ ] nenhuma integração desconhecida recebe Challenge
[ ] nenhuma rota, IP ou ASN foi inventado
[ ] existe no máximo 1 regra
```

---

# Saída do modo compatível

O modo compatível deixa de ser necessário quando forem conhecidos:

- hosts protegidos;
- serviços que recebem integrações externas;
- rotas utilizadas;
- IPs de origem, quando disponíveis;
- ASNs, quando disponíveis;
- países de origem, quando relevantes;
- necessidade de acesso fora do Brasil.

Quando essas informações estiverem disponíveis, volte ao fluxo principal.

Use:

```text
IP FIXO?
→ Modo A: allowlist + BLOCK.

SEM IP FIXO?
→ Modo B:
   BLOCK fora do Brasil
   + Non-Interactive Challenge no restante.
```

Sempre preservando:

```text
IP da VPS
ACME, se necessário
integrações externas legítimas
```

---

# Princípio final

```text
AMBIENTE BEM MAPEADO?
→ use o fluxo principal.

AMBIENTE INCERTO?
→ modo compatível.

HOST ADMINISTRATIVO CONFIRMADO?
→ pode receber Challenge.

API, WEBHOOK OU AUTOMAÇÃO INCERTA?
→ deixe fora temporariamente.

NÃO SABE?
→ não invente.
→ não bloqueie.
→ não quebre.
```

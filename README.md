# Cloudflare VPS WAF Skill

Skill para proteger VPS e serviços self-hosted usando **Cloudflare WAF Custom Rules**.

Foi criada especialmente para ambientes com serviços como:

- n8n
- EasyPanel
- Evolution API
- WAHA
- Portainer
- APIs privadas
- outros serviços self-hosted

## O que a skill faz

A skill analisa o ambiente e gera uma configuração simples de proteção no Cloudflare.

### Com IP fixo

Cria uma estratégia de **allowlist + BLOCK**, permitindo apenas os IPs confiáveis.

### Sem IP fixo

Utiliza:

1. **BLOCK** para acessos estrangeiros aos hosts protegidos;
2. **Non-Interactive Challenge** para o restante.

Também considera:

- IP público da própria VPS;
- comunicação entre serviços da mesma VPS;
- Let's Encrypt / ACME;
- webhooks e integrações externas;
- separação entre sites públicos e serviços administrativos.

## Arquivo principal

A implementação da skill está em:

`cloudflare-vps-waf-skill.md`

## Aviso

Revise e teste as regras antes de aplicá-las em produção. Uma regra incorreta de WAF pode bloquear acessos legítimos ou integrações externas.

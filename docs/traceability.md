# Rastreabilidade

| ID | Requisito | Origem | Evidência no ambiente | Fonte normativa | Status | Validação | Lacuna |
|---|---|---|---|---|---|---|---|
| REQ-01 | Publicar uma API containerizada | Explícito | `Deployment`, `Service`, `Gateway` e `HTTPRoute` em `hello-rbac` | RHCL 1.4 | Atendido | Gateway `Accepted=True`, `Programmed=True` | Nenhuma para a demonstração |
| REQ-05 | OIDC/OAuth 2.0 com JWT | Explícito | Realm `hello-rbac`, cliente confidencial e `AuthPolicy` com `issuerUrl` | RHCL 1.4 e RHBK 26.4 | Atendido | JWT sem token: 401; JWT válido: 200 | Fluxo senha é somente demonstrativo |
| REQ-06 | Autorização granular | Explícito | Rego em `04-authpolicy.yaml` | RHCL 1.4 / Kuadrant 1.4 | Atendido | `blue→/blue=200`, `blue→/red=403`, inverso equivalente | Expandir matriz de permissões para APIs reais |
| REQ-09 | YAML/IaC | Explícito | `manifests/00` a `05` | Decisão arquitetural | Atendido | `oc apply --dry-run=server` e condições dos CRs | Segredos devem vir de cofre/GitOps seguro |
| REQ-15 | Restringir o `HTTPRoute` por IP individual ou bloco CIDR, sem mudar a aplicação | Explícito | `AuthorizationPolicy` no `Gateway`, `remoteIpBlocks`, configuração de proxy confiável e `NetworkPolicy` em `05-source-cidr-authorization.yaml` | Istio / OSSM 3.4 | Atendido na PoC | Origem permitida + JWT válido = 200; origem de rede distinta + mesmo JWT = 403; XFF forjado não substituiu a origem confiável | Topologia de gateway é DP no OSSM 3.4.2; validar alternativa suportada antes de produção |
| REQ-08 | mTLS | Baseline editorial | Service Mesh 3.4/CNI instalados | RHCL 1.4 | Parcial | Control plane saudável | mTLS de backend não foi habilitado neste escopo |
| REQ-07 | Rate limiting | Baseline editorial | Limitador foi instanciado pelo CR Kuadrant | RHCL 1.4 | Não atendido | Fora do escopo funcional | Adicionar `RateLimitPolicy` por consumidor |

## Evidência de execução

Em 2026-09-23, a política `hello-rbac-jwt-and-roles` apresentou as condições
`Accepted=True` e `Enforced=True`. Os cinco testes funcionais tiveram os
resultados: sem token `/blue` = 401; `blue→/blue` = 200;
`blue→/red` = 403; `red→/red` = 200; `red→/blue` = 403.

Na mesma data, o `Gateway` permaneceu `Accepted=True, Programmed=True` após a
aplicação da allowlist. A requisição autenticada da origem permitida para
`/blue` retornou 200; uma requisição autenticada, originada de outra rede,
retornou 403. Uma tentativa de inserir um valor falso em `X-Forwarded-For` não
alterou a origem selecionada pelo gateway e continuou com 200 para a origem
autorizada.

## Referências

- [RHCL 1.4 — release notes](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.4/html/release_notes/rhcl-release-notes) — Red Hat, 1.4, consultado em 2026-09-23.
- [RHCL 1.4 — instalação](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.4/html/install_connectivity_link/rhcl-install-on-ocp) — Red Hat, 1.4, consultado em 2026-09-23.

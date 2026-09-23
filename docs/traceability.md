# Rastreabilidade

| ID | Requisito | Origem | Evidência no ambiente | Fonte normativa | Status | Validação | Lacuna |
|---|---|---|---|---|---|---|---|
| REQ-01 | Publicar uma API containerizada | Explícito | `Deployment`, `Service`, `Gateway` e `HTTPRoute` em `hello-rbac` | RHCL 1.4 | Atendido | Gateway `Accepted=True`, `Programmed=True` | Nenhuma para a demonstração |
| REQ-05 | OIDC/OAuth 2.0 com JWT | Explícito | Realm `hello-rbac`, cliente confidencial e `AuthPolicy` com `issuerUrl` | RHCL 1.4 e RHBK 26.4 | Atendido | JWT sem token: 401; JWT válido: 200 | Fluxo senha é somente demonstrativo |
| REQ-06 | Autorização granular | Explícito | Rego em `04-authpolicy.yaml` | RHCL 1.4 / Kuadrant 1.4 | Atendido | `blue→/blue=200`, `blue→/red=403`, inverso equivalente | Expandir matriz de permissões para APIs reais |
| REQ-09 | YAML/IaC | Explícito | `manifests/00` a `05` | Decisão arquitetural | Atendido | `oc apply --dry-run=server` e condições dos CRs | Segredos devem vir de cofre/GitOps seguro |
| REQ-15 | Restringir o `HTTPRoute` por IP individual ou bloco CIDR, sem mudar a aplicação | Explícito | `AuthorizationPolicy` no `Gateway` com `ipBlocks`, VIP MetalLB e `externalTrafficPolicy: Local` | Istio / OCP 4.22 | Atendido na PoC | Chamada autenticada pelo VIP, de origem permitida = 200; origem distinta, pelo mesmo VIP e JWT = 403 | Reservar IPs e garantir anúncio L2/DNS na rede de cada ambiente |
| REQ-08 | mTLS | Baseline editorial | Service Mesh 3.4/CNI instalados | RHCL 1.4 | Parcial | Control plane saudável | mTLS de backend não foi habilitado neste escopo |
| REQ-07 | Rate limiting | Baseline editorial | Limitador foi instanciado pelo CR Kuadrant | RHCL 1.4 | Não atendido | Fora do escopo funcional | Adicionar `RateLimitPolicy` por consumidor |

## Evidência de execução

Em 2026-09-23, a política `hello-rbac-jwt-and-roles` apresentou as condições
`Accepted=True` e `Enforced=True`. Os cinco testes funcionais tiveram os
resultados: sem token `/blue` = 401; `blue→/blue` = 200;
`blue→/red` = 403; `red→/red` = 200; `red→/blue` = 403.

Na mesma data, o `Gateway` permaneceu `Accepted=True, Programmed=True` após a
aplicação da allowlist. O `Service` do gateway recebeu um VIP MetalLB e passou
a usar `externalTrafficPolicy: Local`. Uma requisição autenticada, enviada
diretamente ao VIP pela origem permitida, retornou 200; uma requisição
autenticada de outra origem pelo mesmo VIP retornou 403.

## Referências

- [RHCL 1.4 — release notes](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.4/html/release_notes/rhcl-release-notes) — Red Hat, 1.4, consultado em 2026-09-23.
- [RHCL 1.4 — instalação](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.4/html/install_connectivity_link/rhcl-install-on-ocp) — Red Hat, 1.4, consultado em 2026-09-23.
- [OpenShift 4.22 — MetalLB Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/networking_operators/metallb-operator) — Red Hat, 4.22, consultado em 2026-09-23.

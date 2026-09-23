# Arquitetura e decisões

## Decisões tomadas

| Decisão | Motivo | Consequência |
|---|---|---|
| RHCL 1.4.3 | É posterior a 1.4.1; 1.4.0 está descontinuado. | Mantém a demonstração na linha suportada. |
| Service Mesh 3.4/Istio 1.30.4 | O requisito pede `GatewayClass` Istio e controle próprio do mesh. | `IstioCNI`, `Istio` e o label `istio.io/rev` são necessários. |
| `AuthPolicy` no `HTTPRoute` | Isola a política desta API sem afetar outros gateways. | Cada rota pode evoluir suas roles independentemente. |
| `Service` `LoadBalancer` com MetalLB | O Gateway é exposto diretamente por um VIP L2 reservado para a API. | DNS do hostname do `HTTPRoute` deve apontar para o VIP; o pool precisa pertencer à rede externa dos nós. |
| `externalTrafficPolicy: Local` + `ipBlocks` | O MetalLB entrega tráfego L4; a política avalia o IP do cliente no gateway. | Há dependência de gateway local em cada nó que anuncia o VIP; escale o gateway para os nós anunciadores em produção. |
| Keycloak gerenciado pelo operador existente | O operador já é restrito ao namespace `keycloak`. | O realm foi isolado por nome, sem mudar o escopo do operador. |
| OPA/Rego para path + role | Expressa a associação endpoint-role em uma única política, fora da aplicação. | Qualquer caminho não listado permanece negado. |

## Claim e política

O Keycloak emite as roles de realm no payload:

```json
{"realm_access":{"roles":["blue"]}}
```

O `AuthPolicy` usa `input.auth.identity.realm_access.roles` e `input.request.path`.
Assim, a autorização não depende de header injetado, de interceptor de framework
ou do backend reconhecer JWT. Para uma aplicação existente, preserve o `Service`
e o `Deployment`; apenas crie/associe `Gateway`, `HTTPRoute` e `AuthPolicy`.

## Allowlist de origem

A camada de origem é propositalmente separada de autenticação e RBAC. A
`AuthorizationPolicy` de `05-source-cidr-authorization.yaml` tem `targetRef`
para o `Gateway`, usa `ipBlocks` e só libera a requisição para as políticas
seguintes quando o endereço remoto está na lista. O `Service` do gateway é
`LoadBalancer`, recebe um VIP do MetalLB e usa `externalTrafficPolicy: Local`
para evitar a perda do IP de origem. O guia de
[IP/CIDR](source-cidr-allowlist.md) contém o modelo reutilizável.

## Limitações e próximos passos

- O exemplo é single-cluster e não demonstra `RateLimitPolicy`, `TLSPolicy`,
  DNSPolicy ou mTLS de backend.
- A imagem Hello World é propositalmente mínima; não é uma referência de
  observabilidade ou disponibilidade da aplicação.
- O fluxo de senha e contas de demonstração devem ser removidos em produção.
- Adicione `aud`/audience validation quando o emissor e as APIs compartilharem
  tokens entre múltiplos recursos.
- Modele roles como grupos, client roles ou permissões finas conforme o modelo
  de identidade corporativo; mantenha a expressão OPA versionada e testada.

## Referências

- [RHCL 1.4 — políticas AuthPolicy](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.4/html-single/red_hat_connectivity_link/index) — Red Hat, 1.4, consultado em 2026-09-23.
- [Autorino/Kuadrant — autorização OPA](https://docs.kuadrant.io/1.4.x/authorino/docs/features/) — documentação community, consultada em 2026-09-23.
- [Istio — Gateway API](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/) — upstream, consultado em 2026-09-23.
- [Istio — controle de acesso no ingress](https://istio.io/latest/docs/tasks/security/authorization/authz-ingress/) — upstream, consultado em 2026-09-23.
- [OpenShift 4.22 — MetalLB Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/networking_operators/metallb-operator) — Red Hat, 4.22, consultado em 2026-09-23.

# Arquitetura e decisões

## Decisões tomadas

| Decisão | Motivo | Consequência |
|---|---|---|
| RHCL 1.4.3 | É posterior a 1.4.1; 1.4.0 está descontinuado. | Mantém a demonstração na linha suportada. |
| Service Mesh 3.4/Istio 1.30.4 | O requisito pede `GatewayClass` Istio e controle próprio do mesh. | `IstioCNI`, `Istio` e o label `istio.io/rev` são necessários. |
| `AuthPolicy` no `HTTPRoute` | Isola a política desta API sem afetar outros gateways. | Cada rota pode evoluir suas roles independentemente. |
| `AuthorizationPolicy` no `Gateway` com `remoteIpBlocks` | O IP original chega ao gateway através de proxies HTTP do OpenShift; `ipBlocks` veria apenas o salto imediato. | A allowlist é aplicada antes do RHCL; o número de proxies confiáveis precisa ser calibrado e testado. |
| `NetworkPolicy` no workload do gateway | Um cabeçalho `X-Forwarded-For` só é atributo de segurança quando os seus remetentes são confiáveis. | A porta HTTP do gateway aceita apenas o namespace `openshift-ingress`; a porta de saúde permanece liberada. |
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
para o `Gateway`, usa `remoteIpBlocks` e só libera a requisição para as políticas
seguintes quando o endereço extraído da cadeia XFF está na lista. A configuração
`proxy.istio.io/config` do `Gateway` informa quantos proxies à frente do Istio
podem ser confiados. O guia de [IP/CIDR](source-cidr-allowlist.md) contém a
calibração e o modelo reutilizável.

Esta decisão foi validada no laboratório, mas não deve ser promovida como um
padrão de produção sem a avaliação de suporte: na matriz do OpenShift Service
Mesh 3.4.2, `AuthorizationPolicy` é GA, enquanto a configuração de topologia de
gateway é Developer Preview. Para uma necessidade de produção imediata, aplique
a allowlist também — ou preferencialmente — no balanceador, WAF ou camada de
ingress corporativa que seja suportada no ambiente.

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
- [Service Mesh 3.4 — tabela de suporte](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/release_notes/ossm-release-notes-feature-support-tables) — Red Hat, 3.4.2, consultado em 2026-09-23.

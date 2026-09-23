# Manual de implementação: OIDC/JWT e roles por endpoint sem alterar a aplicação

## O que este manual entrega

Ao final, uma API já existente terá seus endpoints protegidos no Gateway:

| Endpoint | Quem pode acessar |
|---|---|
| `GET /blue` | Somente token JWT com a role `blue` |
| `GET /red` | Somente token JWT com a role `red` |

A aplicação não precisa validar token, conhecer Keycloak ou ter código
alterado. A validação e a autorização acontecem antes de a requisição chegar ao
serviço:

```text
Cliente → Gateway Istio → RHCL/Authorino → Serviço existente
                         ↘ Keycloak (discovery OIDC e chaves públicas)
```

Este manual trata exclusivamente da `AuthPolicy`. Ele não altera a publicação
da API, o Gateway, o `HTTPRoute`, o Keycloak, o realm, o cliente OIDC nem os
usuários.

## Antes de começar

Este manual pressupõe que OpenShift Service Mesh, RHCL/Authorino, Keycloak,
Gateway e `HTTPRoute` já estão operacionais no ambiente do cliente.

Também é necessário que o Keycloak já emita JWTs com estas roles de realm:

| Role | Usuário ou consumidor de teste |
|---|---|
| `blue` | Deve receber a role `blue` no token |
| `red` | Deve receber a role `red` no token |

As roles precisam aparecer no claim `realm_access.roles` do JWT. Se o ambiente
usar outro claim, ajuste somente a expressão Rego mostrada no passo 1.

Você precisa ter:

- Permissão para criar uma `AuthPolicy` no namespace da API.
- O nome do namespace, do `HTTPRoute` e do Gateway já existentes.
- A URL exata do issuer do Keycloak, incluindo `/realms/<nome-do-realm>`.
- Um token de teste para a role `blue` e outro para a role `red`.

Use estes nomes de exemplo e substitua todos antes de aplicar:

| Item | Exemplo |
|---|---|
| Namespace da API | `orders-api` |
| Gateway existente | `orders-gateway` |
| HTTPRoute existente | `orders-api` |
| Issuer do Keycloak | `https://sso.internal.example.com/realms/orders` |
| Endereço da API | `http://orders-api.internal.example.com` |

Confirme que o alvo da política já existe. Esses comandos não alteram a
configuração atual:

```bash
oc -n orders-api get gateway orders-gateway
oc -n orders-api get httproute orders-api
```

## Como a política funciona

A `AuthPolicy` primeiro valida assinatura, emissor e prazo de validade do JWT
usando as informações OIDC do Keycloak. Depois, a regra Rego avalia o caminho
da requisição e a role no token:

```text
JWT ausente, inválido ou expirado  → 401
JWT válido, mas sem a role exigida → 403
JWT válido, com a role exigida     → chamada segue para a API
```

Qualquer caminho não listado na regra é negado. Portanto, adicione regras
explícitas ao publicar novos endpoints.

## Passo 1 — Criar a AuthPolicy

Crie a política no mesmo namespace do `HTTPRoute`. Troque `namespace`, nome do
`HTTPRoute` e `issuerUrl` pelos valores do ambiente.

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: orders-api-jwt-and-roles
  namespace: orders-api
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: orders-api
  rules:
    authentication:
      keycloak-jwt:
        jwt:
          issuerUrl: https://sso.internal.example.com/realms/orders
    authorization:
      endpoint-role:
        opa:
          rego: |
            allow {
              input.request.path == "/blue"
              input.auth.identity.realm_access.roles[_] == "blue"
            }

            allow {
              input.request.path == "/red"
              input.auth.identity.realm_access.roles[_] == "red"
            }
    response:
      unauthenticated:
        code: 401
        body:
          value: '{"error":"unauthenticated","message":"A valid JWT is required."}'
      unauthorized:
        code: 403
        body:
          value: '{"error":"forbidden","message":"The token lacks the required role."}'
```

Salve como `orders-authpolicy.yaml` e aplique:

```bash
oc apply -f orders-authpolicy.yaml
oc -n orders-api get authpolicy orders-api-jwt-and-roles \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{" "}{end}{"\n"}'
```

O resultado esperado é `Accepted=True Enforced=True`.

## Passo 2 — Testar a política

Obtenha previamente no Keycloak os dois tokens de teste. Não grave tokens,
segredos ou senhas em arquivos, tickets ou repositórios.

```bash
export API_URL=http://orders-api.internal.example.com
export BLUE_TOKEN='cole-aqui-o-jwt-com-a-role-blue'
export RED_TOKEN='cole-aqui-o-jwt-com-a-role-red'
```

Execute a matriz completa:

```bash
curl -i "${API_URL}/blue"                                      # 401
curl -i -H "Authorization: Bearer ${BLUE_TOKEN}" "${API_URL}/blue" # 200
curl -i -H "Authorization: Bearer ${BLUE_TOKEN}" "${API_URL}/red"  # 403
curl -i -H "Authorization: Bearer ${RED_TOKEN}" "${API_URL}/red"   # 200
curl -i -H "Authorization: Bearer ${RED_TOKEN}" "${API_URL}/blue"  # 403
```

Os testes devem ser feitos pelo mesmo endereço já publicado da API. Assim, a
validação passa pelo Gateway e pela `AuthPolicy`.

## Diagnóstico

Confira o estado da política e o alvo configurado:

```bash
oc -n orders-api get authpolicy orders-api-jwt-and-roles -o yaml
oc -n orders-api get httproute orders-api
```

| Sintoma | Causa provável | Ação |
|---|---|---|
| `401` com token aparentemente válido | `issuerUrl` difere do claim `iss`, token expirado ou Gateway sem acesso ao discovery/JWKS | Compare o claim `iss` do token com `issuerUrl` e valide a conectividade do Gateway ao Keycloak. |
| `403` em ambos os endpoints | Role ausente do claim `realm_access.roles` ou caminho diferente do definido na regra | Confira o payload do token, as roles atribuídas e o caminho chamado. |
| `AuthPolicy` não é aplicada | Nome ou namespace do `HTTPRoute` incorreto, ou Gateway não está integrado ao RHCL | Confira `targetRef`, namespace e a configuração RHCL do Gateway existente. |
| Endpoint novo retorna 403 | O caminho não foi incluído na regra Rego | Acrescente uma regra explícita para o novo endpoint e a role correspondente. |

## Recomendações para produção

- Use o fluxo OIDC adequado ao consumidor: `authorization_code` com PKCE para
  pessoas e `client_credentials` para integrações máquina a máquina.
- Valide `aud` quando tokens de um mesmo issuer puderem ser usados por mais de
  uma API.
- Use HTTPS no Gateway, rotação de certificados e mTLS entre Gateway e backend
  conforme a política corporativa.
- Trate a política Rego como código versionado e revisado. Roles e endpoints
  novos devem ter testes de acesso permitido e negado.

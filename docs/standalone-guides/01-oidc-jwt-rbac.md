# Manual de implementação: OIDC/JWT e roles por endpoint sem alterar a aplicação

## O que este manual entrega

Ao final, uma API já existente terá dois endpoints protegidos no gateway:

| Endpoint | Quem pode acessar |
|---|---|
| `GET /blue` | Somente token JWT com a role `blue` |
| `GET /red` | Somente token JWT com a role `red` |

A aplicação não precisa validar token, conhecer Keycloak ou ter qualquer linha
de código alterada. A validação ocorre antes de a requisição chegar ao serviço:

```text
Cliente → Gateway Istio → RHCL/Authorino → Serviço existente
                         ↘ Keycloak (descoberta OIDC e chaves públicas)
```

Este manual usa OpenShift Service Mesh 3.4, Red Hat Connectivity Link (RHCL)
1.4 e Red Hat build of Keycloak (RHBK). Os nomes são exemplos: ajuste-os antes
de aplicar no ambiente.

## Antes de começar

Você precisa ter:

- Acesso `cluster-admin` ou permissões equivalentes para criar recursos no
  namespace da API e no namespace do Keycloak.
- OpenShift Service Mesh com um `GatewayClass` Istio aceito.
- RHCL instalado e saudável, com `Kuadrant` e `Authorino` em execução.
- Um Keycloak gerenciado pelo operador, já acessível pelo Gateway para consulta
  ao discovery OIDC e às chaves públicas (JWKS).
- Uma aplicação existente atrás de um `Service` Kubernetes. O exemplo usa o
  serviço `orders-api` na porta `8080`.

Faça esta verificação inicial:

```bash
oc get gatewayclass
oc get kuadrant -A
oc get authorino -A
oc get keycloak -A
```

No exemplo abaixo serão usados estes valores:

| Variável | Exemplo | Ajuste necessário |
|---|---|---|
| Namespace da API | `orders-api` | Sim |
| Revisão Istio | `production` | Sim |
| Namespace do Keycloak | `keycloak` | Sim, se diferente |
| CR do Keycloak | `keycloak` | Sim, se diferente |
| Realm | `orders` | Opcional |
| Cliente OIDC | `orders-api-client` | Opcional |
| Host do Keycloak | `sso.internal.example.com` | Sim |
| Host da API | `orders-api.internal.example.com` | Sim |

## Passo 1 — Preparar o namespace da API

O label `istio.io/rev` faz o Gateway usar a revisão do Istio escolhida. Se o
namespace já existe, aplique apenas o label correspondente à sua malha.

```bash
oc create namespace orders-api --dry-run=client -o yaml | oc apply -f -
oc label namespace orders-api istio.io/rev=production --overwrite
```

Confirme que o serviço existente está disponível:

```bash
oc -n orders-api get service orders-api
oc -n orders-api get endpointslice -l kubernetes.io/service-name=orders-api
```

## Passo 2 — Criar segredos fora do Git

O segredo do cliente OIDC e as senhas de teste NÃO DEVEM entrar em repositórios,
tickets ou arquivos YAML. Crie-os diretamente no cluster ou com um cofre
integrado ao GitOps.

```bash
oc -n keycloak create secret generic orders-api-oidc-client \
  --from-literal=client-id=orders-api-client \
  --from-literal=client-secret="$(openssl rand -base64 48 | tr -d '\n')"

oc -n keycloak create secret generic orders-api-test-users \
  --from-literal=blue-password="$(openssl rand -base64 32 | tr -d '\n')" \
  --from-literal=red-password="$(openssl rand -base64 32 | tr -d '\n')"
```

## Passo 3 — Criar realm, cliente, roles e usuários no Keycloak

O YAML completo abaixo cria:

- Realm `orders`;
- Roles de realm `blue` e `red`;
- Cliente confidencial `orders-api-client`;
- Usuários de teste `blue-user` e `red-user`.

O operador Keycloak normalmente observa `KeycloakRealmImport` apenas no seu
próprio namespace. Troque `keycloakCRName` caso o CR do Keycloak tenha outro
nome.

```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: orders-realm
  namespace: keycloak
spec:
  keycloakCRName: keycloak
  placeholders:
    OIDC_CLIENT_SECRET:
      secret:
        name: orders-api-oidc-client
        key: client-secret
    BLUE_PASSWORD:
      secret:
        name: orders-api-test-users
        key: blue-password
    RED_PASSWORD:
      secret:
        name: orders-api-test-users
        key: red-password
  realm:
    realm: orders
    displayName: Orders API
    enabled: true
    roles:
      realm:
        - name: blue
          description: Permite GET /blue.
        - name: red
          description: Permite GET /red.
    clients:
      - clientId: orders-api-client
        name: Orders API test client
        enabled: true
        protocol: openid-connect
        publicClient: false
        clientAuthenticatorType: client-secret
        secret: "${OIDC_CLIENT_SECRET}"
        directAccessGrantsEnabled: true
        standardFlowEnabled: false
        serviceAccountsEnabled: false
        fullScopeAllowed: true
    users:
      - username: blue-user
        enabled: true
        email: blue-user@example.invalid
        emailVerified: true
        realmRoles:
          - blue
        credentials:
          - type: password
            value: "${BLUE_PASSWORD}"
            temporary: false
      - username: red-user
        enabled: true
        email: red-user@example.invalid
        emailVerified: true
        realmRoles:
          - red
        credentials:
          - type: password
            value: "${RED_PASSWORD}"
            temporary: false
```

Salve como `orders-realm.yaml` e aplique:

```bash
oc apply -f orders-realm.yaml
oc -n keycloak get keycloakrealmimport orders-realm
```

O importador é indicado para a criação inicial. Para mudar um realm já criado,
use uma automação de administração do Keycloak ou a Admin API; reaplicar o
importador não é um mecanismo de atualização de realm.

## Passo 4 — Publicar o serviço existente com Gateway API

Este exemplo não cria nem muda a aplicação. Ele apenas encaminha `/blue` e
`/red` para o `Service` existente. Caso você já possua um `Gateway` e um
`HTTPRoute`, mantenha-os e ajuste apenas os nomes usados no próximo passo.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: orders-gateway
  namespace: orders-api
  labels:
    kuadrant.io/gateway: "true"
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      hostname: orders-api.internal.example.com
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-api
  namespace: orders-api
spec:
  parentRefs:
    - name: orders-gateway
      sectionName: http
  hostnames:
    - orders-api.internal.example.com
  rules:
    - matches:
        - path:
            type: Exact
            value: /blue
      backendRefs:
        - name: orders-api
          port: 8080
    - matches:
        - path:
            type: Exact
            value: /red
      backendRefs:
        - name: orders-api
          port: 8080
```

Salve como `orders-gateway.yaml`, aplique e espere o Gateway ser programado:

```bash
oc apply -f orders-gateway.yaml
oc -n orders-api wait gateway/orders-gateway \
  --for=condition=Programmed=True --timeout=3m
```

## Passo 5 — Aplicar autenticação JWT e autorização por role

Este é o recurso que protege a API. Troque o host em `issuerUrl` pelo endereço
real do Keycloak e mantenha `/realms/orders` coerente com o realm criado.

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

## Passo 6 — Testar ponta a ponta

Use o endereço pelo qual o Gateway está publicado. Neste exemplo ele é
`http://orders-api.internal.example.com`.

```bash
export KEYCLOAK_HOST=sso.internal.example.com
export API_URL=http://orders-api.internal.example.com
export CLIENT_ID="$(oc -n keycloak get secret orders-api-oidc-client -o jsonpath='{.data.client-id}' | base64 -d)"
export CLIENT_SECRET="$(oc -n keycloak get secret orders-api-oidc-client -o jsonpath='{.data.client-secret}' | base64 -d)"
export BLUE_PASSWORD="$(oc -n keycloak get secret orders-api-test-users -o jsonpath='{.data.blue-password}' | base64 -d)"

BLUE_TOKEN="$(curl -fsS -X POST \
  "https://${KEYCLOAK_HOST}/realms/orders/protocol/openid-connect/token" \
  --data-urlencode grant_type=password \
  --data-urlencode client_id="$CLIENT_ID" \
  --data-urlencode client_secret="$CLIENT_SECRET" \
  --data-urlencode username=blue-user \
  --data-urlencode password="$BLUE_PASSWORD" | jq -r .access_token)"
```

Execute a matriz de teste:

```bash
curl -i "${API_URL}/blue"                                      # 401
curl -i -H "Authorization: Bearer ${BLUE_TOKEN}" "${API_URL}/blue" # 200
curl -i -H "Authorization: Bearer ${BLUE_TOKEN}" "${API_URL}/red"  # 403
```

Para testar `red-user`, obtenha o token com a senha `red-password` e valide
que `/red` retorna 200 e `/blue` retorna 403.

## Problemas comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| `401` com token aparentemente válido | `issuerUrl` diferente do claim `iss`, ou gateway sem acesso ao JWKS | Compare `iss` com `issuerUrl` e teste discovery/JWKS a partir da malha. |
| `403` em ambos os endpoints | Role não está em `realm_access.roles` | Confirme as realm roles atribuídas ao usuário e emita um novo token. |
| `AuthPolicy` não é aplicada | Nome/namespace do `HTTPRoute` incorreto ou Gateway sem label RHCL | Confira `targetRef`, namespace e `kuadrant.io/gateway: "true"`. |
| Gateway não gera workload | Namespace sem a revisão Istio correta | Confira `istio.io/rev` e `GatewayClass`. |

## Recomendações para produção

- Desabilite `directAccessGrantsEnabled` após os testes. Para pessoas, use
  `authorization_code` com PKCE; para integrações, use `client_credentials`.
- Valide `aud` quando os tokens puderem ser aceitos por mais de uma API.
- Use HTTPS no Gateway, rotação de certificados e mTLS entre gateway e backend
  conforme a política corporativa.
- Armazene segredos em cofre e trate a política Rego como código versionado e
  testado.

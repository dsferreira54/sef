# Manual de implementação: allowlist de IP/CIDR por HTTPRoute ou Gateway

## O que este manual entrega

Ao final, uma API já publicada por um VIP MetalLB aceitará chamadas somente de
IPs ou redes CIDR aprovados, sem alterar código, imagem, `Deployment`,
`Service` ou a publicação já existente da aplicação.

Há duas formas de aplicar a allowlist:

| Opção | Política | Escopo | Quando usar |
|---|---|---|---|
| A — recomendada | `AuthPolicy` do RHCL/Authorino | Um `HTTPRoute` específico | Quando cada API precisa controlar sua própria allowlist em um Gateway compartilhado. |
| B — alternativa | `AuthorizationPolicy` do Istio | Todo o `Gateway` | Quando o Gateway é dedicado à API ou todas as rotas nele devem ter a mesma allowlist. |

```text
Cliente → VIP MetalLB já existente → Gateway Istio → HTTPRoute → Serviço existente
                                  └→ política de origem: permite ou responde 403
```

## Antes de começar

Este manual pressupõe que o VIP MetalLB, o Gateway, o `HTTPRoute` e o `Service`
já funcionam. A equipe de plataforma/rede deve confirmar que o endereço IP
original do consumidor chega ao Gateway sem ser substituído por outro IP.

Para a opção A, RHCL e Authorino também já devem estar operacionais. O
Authorino recebe o endereço remoto no pedido de autorização; a política avalia
esse dado diretamente, sem depender de cabeçalho enviado pelo cliente.

Você precisa ter:

- Permissão para criar ou alterar políticas no namespace da API.
- O nome do `HTTPRoute` e do `Gateway` já existentes.
- O hostname e o VIP internos da API.
- A lista de IPs ou redes CIDR aprovada pela equipe de segurança.

Use estes valores fictícios e substitua todos antes de aplicar:

| Item | Exemplo |
|---|---|
| Namespace da API | `orders-api` |
| HTTPRoute existente | `orders-api` |
| Gateway existente | `orders-gateway` |
| Host da API | `orders-api.internal.example.com` |
| VIP já publicado | `192.168.100.50` |
| Rede autorizada | `192.168.10.0/24` |
| IP individual autorizado | `192.168.20.15/32` |

Confira os alvos antes de alterar políticas:

```bash
oc -n orders-api get gateway orders-gateway
oc -n orders-api get httproute orders-api
```

## Opção A — Restringir um HTTPRoute com AuthPolicy

Esta é a opção indicada quando o Gateway atende mais de uma API. A política é
anexada ao `HTTPRoute`; portanto, não afeta as demais rotas do mesmo Gateway.

O Authorino entrega o endereço de origem IPv4 no formato `IP:porta`. A regra
abaixo remove a porta e compara somente o IP com os CIDRs permitidos. Este
exemplo é para IPv4, como os endereços apresentados neste manual.

### Passo 1 — Aplicar uma AuthPolicy de origem na rota

Use este YAML quando a rota ainda não possui uma `AuthPolicy`. Troque o
namespace, o nome da rota e os CIDRs aprovados.

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: orders-api-source-cidr
  namespace: orders-api
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: orders-api
  rules:
    authorization:
      source-cidr:
        opa:
          rego: |
            source_address := input.context.source.address.Address.SocketAddress.address
            source_ip := split(source_address, ":")[0]

            allow {
              net.cidr_contains("192.168.10.0/24", source_ip)
            }

            allow {
              net.cidr_contains("192.168.20.15/32", source_ip)
            }
    response:
      unauthorized:
        code: 403
        body:
          value: '{"error":"forbidden","message":"Source IP is not allowed."}'
```

Salve como `orders-route-source-cidr.yaml` e aplique:

```bash
oc apply -f orders-route-source-cidr.yaml
oc -n orders-api get authpolicy orders-api-source-cidr \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{end}{"\n"}'
```

O resultado esperado é `Accepted=True Enforced=True`.

### Passo 2 — Incluir a origem em uma AuthPolicy que já existe

Uma rota que já possui autenticação JWT ou regras de roles deve continuar com
uma única `AuthPolicy`. Nesse caso, acrescente o bloco `source-cidr` dentro de
`spec.rules.authorization` da política existente, ao lado das demais regras de
autorização:

```yaml
authorization:
  endpoint-role:
    opa:
      rego: |
        # Mantenha aqui a regra de roles já usada pela API.
        allow {
          input.request.path == "/blue"
          input.auth.identity.realm_access.roles[_] == "blue"
        }
  source-cidr:
    opa:
      rego: |
        source_address := input.context.source.address.Address.SocketAddress.address
        source_ip := split(source_address, ":")[0]

        allow {
          net.cidr_contains("192.168.10.0/24", source_ip)
        }
```

As regras de autorização são cumulativas: a chamada precisa ser aceita pela
regra de origem e também pelas regras de roles, JWT ou qualquer outra regra já
definida na mesma política.

## Opção B — Restringir todo o Gateway com AuthorizationPolicy

Use esta opção somente se a mesma allowlist deve valer para todas as rotas do
Gateway. A `AuthorizationPolicy` nativa do Istio não aceita `HTTPRoute` como
alvo; por isso ela é ligada ao `Gateway`.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: orders-gateway-source-cidr
  namespace: orders-api
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: orders-gateway
  action: ALLOW
  rules:
    - from:
        - source:
            remoteIpBlocks:
              - 192.168.10.0/24
              - 192.168.20.15/32
```

Salve como `orders-gateway-source-cidr.yaml` e aplique:

```bash
oc apply -f orders-gateway-source-cidr.yaml
oc -n orders-api get authorizationpolicy orders-gateway-source-cidr
```

Uma política com `action: ALLOW` bloqueia qualquer origem que não corresponda
às regras. Em um Gateway compartilhado, isso inclui rotas de outras APIs; por
isso prefira a opção A para isolamento por `HTTPRoute`.

## Testar diretamente no VIP existente

O DNS interno deve resolver o hostname da API para o VIP já publicado. Faça a
chamada de uma máquina dentro da rede permitida. Se a rota exigir JWT, use um
token válido; caso contrário, remova a linha `Authorization`.

```bash
export API_URL=http://orders-api.internal.example.com
export TOKEN='cole-aqui-um-jwt-valido-se-a-api-exigir-autenticacao'

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "${API_URL}/blue"
```

Quando o DNS ainda não estiver disponível na máquina de teste, informe o
hostname e o VIP diretamente ao `curl`:

```bash
export API_HOST=orders-api.internal.example.com
export VIP=192.168.100.50

curl --resolve "${API_HOST}:80:${VIP}" -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://${API_HOST}/blue"
```

Execute a matriz com o mesmo token válido, quando a API exigir autenticação:

| Origem da chamada | Resultado esperado |
|---|---:|
| IP ou rede presente na allowlist e demais políticas satisfeitas | 200 |
| IP ou rede fora da allowlist, com as mesmas credenciais | 403 |
| IP permitido, sem JWT exigido pela API | 401 |

O teste de bloqueio DEVE partir de uma origem realmente fora da allowlist.
Alterar apenas o token não comprova a restrição por IP.

## Diagnóstico

Para a opção A:

```bash
oc -n orders-api get authpolicy orders-api-source-cidr -o yaml
oc -n orders-api get httproute orders-api
```

Para a opção B:

```bash
oc -n orders-api get authorizationpolicy orders-gateway-source-cidr -o yaml
oc -n orders-api get gateway orders-gateway
```

| Sintoma | Causa provável | Ação |
|---|---|---|
| Origem permitida recebe 403 | O IP percebido pela política é diferente do esperado, o CIDR está incorreto ou outra regra rejeita a chamada | Confirme a origem real com a equipe de rede e revise as políticas aplicáveis. |
| Origem fora da lista recebe 200 | A política aponta para o recurso errado ou não está reforçada | Confira `targetRef`, namespace e as condições da `AuthPolicy`. |
| Todas as rotas de um Gateway compartilhado recebem 403 | Uma `AuthorizationPolicy` com `ALLOW` foi usada no Gateway | Mova a allowlist para a opção A ou use um Gateway dedicado. |
| Origem permitida recebe 401 | A allowlist passou, mas a API exige JWT ou outra credencial | Obtenha uma credencial válida e repita o teste. |

## Recomendações para produção

- Prefira a opção A para separar as regras de cada API em um Gateway
  compartilhado.
- Mantenha os CIDRs pequenos, específicos e aprovados pela segurança. Prefira
  `/32` para origens fixas.
- Registre responsável, justificativa e data de revisão de cada entrada.
- Antes de ampliar uma rede, investigue mudanças de NAT, VPN ou proxy que podem
  alterar o IP percebido pelo Gateway.
- Use HTTPS e mantenha JWT, roles e allowlist como camadas complementares de
  proteção.

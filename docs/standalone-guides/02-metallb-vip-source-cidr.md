# Manual de implementação: VIP MetalLB e allowlist de IP/CIDR no Gateway

## O que este manual entrega

Ao final, um `HTTPRoute` existente será acessado diretamente por um VIP do
MetalLB. Antes de a requisição chegar ao backend, o Gateway Istio permitirá
somente IPs ou blocos CIDR aprovados.

```text
Cliente → VIP MetalLB → Gateway Istio → HTTPRoute → Serviço existente
                    └→ AuthorizationPolicy: permite ou responde 403
```

Não é necessário mudar código, imagem, `Deployment` ou `Service` da aplicação.
Este manual pressupõe OpenShift 4.22, OpenShift Service Mesh 3.4/Istio e
MetalLB já instalados e operacionais, em uma rede interna onde o VIP já é
alcançável pelos consumidores.

## Antes de começar

Você precisa ter:

- Permissões para criar e alterar recursos de rede no namespace do MetalLB e
  no namespace da API.
- MetalLB Operator e a instância `MetalLB` já em operação no namespace
  `metallb-system`.
- Um Gateway Istio e um `HTTPRoute` funcionais para a API existente.
- Uma faixa de IPs reservada pela equipe de rede, na mesma rede L2 dos nós que
  anunciarão o VIP. O endereço NÃO DEVE pertencer ao DHCP nem estar em uso.
- DNS interno para apontar o hostname da API ao VIP. Enquanto DNS não estiver
  pronto, os testes podem usar `curl --resolve`.

Use estes nomes de exemplo e substitua todos antes da aplicação:

| Item | Exemplo |
|---|---|
| Namespace da API | `orders-api` |
| Gateway | `orders-gateway` |
| Serviço gerado pelo Gateway | `orders-gateway-istio` |
| Host da API | `orders-api.internal.example.com` |
| VIP reservado | `192.168.100.50` |
| Rede autorizada | `192.168.10.0/24` |

Confirme o estado atual:

```bash
oc get gatewayclass
oc -n orders-api get gateway,httproute
oc -n metallb-system get metallb
oc get nodes -o wide
```

## Como a solução funciona

O MetalLB anuncia um VIP de camada 2 para um `Service` do tipo `LoadBalancer`.
O Gateway Istio recebe a conexão diretamente nesse VIP. A opção
`externalTrafficPolicy: Local` preserva o endereço de origem até o Gateway;
por isso a `AuthorizationPolicy` usa `ipBlocks` para comparar o IP remoto com a
lista permitida.

Se houver vários nós, RECOMENDA-SE executar réplicas do Gateway nos nós que
podem anunciar o VIP. Com `externalTrafficPolicy: Local`, enviar tráfego a um
nó sem Pod local do Gateway pode causar indisponibilidade.

## Passo 1 — Criar pool e anúncio L2

O pool abaixo contém apenas um VIP e restringe sua atribuição ao namespace da
API. Ajuste `addresses` para o VIP reservado pelo cliente. O selector do anúncio
garante que somente o serviço daquele Gateway use esse pool.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: orders-gateway-vip
  namespace: metallb-system
spec:
  addresses:
    - 192.168.100.50-192.168.100.50
  autoAssign: false
  serviceAllocation:
    namespaces:
      - orders-api
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: orders-gateway-vip
  namespace: metallb-system
spec:
  ipAddressPools:
    - orders-gateway-vip
  serviceSelectors:
    - matchLabels:
        gateway.networking.k8s.io/gateway-name: orders-gateway
```

Salve como `orders-metallb.yaml` e aplique:

```bash
oc apply -f orders-metallb.yaml
oc -n metallb-system get ipaddresspool,l2advertisement
```

## Passo 2 — Solicitar LoadBalancer para o Gateway

Adicione as partes abaixo ao recurso `Gateway` existente. Mantenha os listeners
e demais configurações já usados pela API.

```yaml
metadata:
  annotations:
    networking.istio.io/service-type: LoadBalancer
spec:
  infrastructure:
    annotations:
      metallb.io/address-pool: orders-gateway-vip
```

Como referência, este é um Gateway completo e mínimo:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: orders-gateway
  namespace: orders-api
  labels:
    kuadrant.io/gateway: "true"
  annotations:
    networking.istio.io/service-type: LoadBalancer
spec:
  gatewayClassName: istio
  infrastructure:
    annotations:
      metallb.io/address-pool: orders-gateway-vip
  listeners:
    - name: http
      hostname: orders-api.internal.example.com
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

Depois de aplicar o Gateway, descubra o serviço gerado pelo controller Istio:

```bash
oc -n orders-api get service \
  -l gateway.networking.k8s.io/gateway-name=orders-gateway
```

Normalmente o nome é `<nome-do-gateway>-istio`, mas use o nome retornado pelo
comando no passo seguinte.

## Passo 3 — Preservar o IP de origem

O controller Istio cria o `Service` do Gateway. Aplique este patch com
server-side apply para definir `externalTrafficPolicy: Local` sem alterar as
portas e selectors gerados pelo controller:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-gateway-istio
  namespace: orders-api
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

Salve como `orders-gateway-service.yaml` e aplique:

```bash
oc apply --server-side -f orders-gateway-service.yaml
oc -n orders-api get service orders-gateway-istio
```

O serviço deve mostrar `TYPE=LoadBalancer` e o VIP em `EXTERNAL-IP`.

## Passo 4 — Aplicar a allowlist IP/CIDR

Crie uma `AuthorizationPolicy` apontando para o Gateway. Um `/32` representa
um IP individual. Um prefixo, como `/24`, representa uma rede inteira.

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
            ipBlocks:
              - 192.168.10.0/24
              - 192.168.20.15/32
```

Salve como `orders-source-cidr.yaml` e aplique:

```bash
oc apply -f orders-source-cidr.yaml
oc -n orders-api get authorizationpolicy orders-gateway-source-cidr
```

Uma política com `action: ALLOW` bloqueia qualquer origem que não corresponda a
uma regra. Se a API também exige JWT, o bloqueio por IP ocorre antes da decisão
de autenticação/autorização da API.

## Passo 5 — Configurar DNS e testar

Crie um registro DNS interno para `orders-api.internal.example.com` apontando
para `192.168.100.50`. Enquanto ele não estiver disponível, informe o hostname
e o VIP diretamente ao `curl`:

```bash
export API_HOST=orders-api.internal.example.com
export VIP=192.168.100.50
export TOKEN='cole-aqui-um-jwt-valido-se-a-api-exigir-autenticacao'

curl --resolve "${API_HOST}:80:${VIP}" -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://${API_HOST}/blue"
```

Execute a matriz abaixo usando um token válido, se a API estiver protegida por
JWT:

| Origem da chamada | Resultado esperado |
|---|---:|
| IP ou rede presente em `ipBlocks` e credenciais válidas | 200 |
| IP ou rede fora de `ipBlocks`, com as mesmas credenciais válidas | 403 |
| IP permitido, sem credenciais exigidas pela API | 401 |

Valide o anúncio do VIP quando houver problema de conectividade:

```bash
oc -n orders-api get service orders-gateway-istio
oc -n metallb-system get servicel2status
oc -n metallb-system get pods
```

## Problemas comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| `EXTERNAL-IP` fica pendente | Pool, anúncio ou selector não corresponde ao serviço | Confira `IPAddressPool`, `L2Advertisement`, labels do serviço e annotation `metallb.io/address-pool`. |
| VIP não responde | VIP não é roteável, não está na rede L2 dos nós ou DNS aponta para outro IP | Valide a reserva com a rede e `ServiceL2Status`. |
| Origem permitida recebe 403 | IP percebido pelo Gateway não pertence ao CIDR ou o IP foi mascarado antes de chegar ao cluster | Confirme `externalTrafficPolicy: Local` e identifique a origem real antes de ampliar a lista. |
| Parte dos acessos falha em cluster com vários nós | Nó anunciado não tem Pod local do Gateway | Escale/posicione o Gateway nos nós anunciadores ou reveja o modelo de anúncio. |

## Recomendações para produção

- Use um VIP reservado e documentado pela equipe de rede; NÃO use uma faixa
  DHCP ou um IP que possa ser reutilizado.
- Para alta disponibilidade, defina a estratégia de anúncio L2 ou BGP junto à
  equipe de rede e mantenha Gateways nos nós anunciadores.
- Use HTTPS no Gateway, certificado administrado e DNS interno apontando para o
  VIP. MetalLB expõe o serviço em L4; ele não substitui TLS nem autenticação.
- Mantenha `ipBlocks` pequeno e revisado. Mudanças de NAT, VPN ou proxy podem
  alterar o IP percebido pelo Gateway e devem passar por controle de mudança.

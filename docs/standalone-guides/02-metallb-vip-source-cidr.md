# Manual de implementação: allowlist de IP/CIDR no Gateway exposto por VIP MetalLB

## O que este manual entrega

Ao final, o `HTTPRoute` de uma API já publicada por um VIP do MetalLB aceitará
somente chamadas originadas em IPs ou blocos CIDR aprovados. A decisão acontece
no Gateway Istio, antes de a chamada chegar ao backend.

```text
Cliente → VIP MetalLB já existente → Gateway Istio → HTTPRoute → Serviço existente
                                  └→ AuthorizationPolicy: permite ou responde 403
```

Não é necessário alterar código, imagem, `Deployment`, `Service` nem a
publicação já existente da aplicação. Este manual trata exclusivamente da
política de acesso por origem.

## Antes de começar

Este manual pressupõe que o Gateway, o `HTTPRoute`, o `Service` e o VIP MetalLB
já estão funcionando no ambiente do cliente. A equipe de plataforma/rede deve
confirmar que o IP original do consumidor chega ao Gateway sem ser substituído
por outro IP. A política compara esse IP que o Gateway recebe.

Você precisa ter:

- Permissão para criar uma `AuthorizationPolicy` no namespace da API.
- O nome do namespace e do `Gateway` que já publica a API.
- O hostname interno e o VIP já associados à API.
- A lista de IPs ou redes CIDR aprovada pela equipe de segurança.

Use estes nomes de exemplo e substitua todos antes de aplicar:

| Item | Exemplo |
|---|---|
| Namespace da API | `orders-api` |
| Gateway existente | `orders-gateway` |
| HTTPRoute existente | `orders-api` |
| Host da API | `orders-api.internal.example.com` |
| VIP já publicado | `192.168.100.50` |
| Rede autorizada | `192.168.10.0/24` |

Faça a verificação inicial. Ela confirma que o Gateway e a rota já existem;
ela não altera a configuração de publicação do VIP.

```bash
oc -n orders-api get gateway orders-gateway
oc -n orders-api get httproute orders-api
```

## Como a política funciona

Uma `AuthorizationPolicy` com `action: ALLOW` se torna uma allowlist: uma
chamada só é permitida se corresponder a pelo menos uma regra. O campo
`ipBlocks` aceita tanto IPs individuais quanto redes CIDR:

| Valor | Significado |
|---|---|
| `192.168.20.15/32` | Apenas o IP `192.168.20.15` |
| `192.168.10.0/24` | Endereços de `192.168.10.0` a `192.168.10.255` |

Se a API também usa JWT e roles, a origem fora da allowlist recebe `403` antes
da decisão de autenticação/autorização da API. Para uma origem permitida, as
demais políticas continuam valendo normalmente.

## Passo 1 — Criar a allowlist IP/CIDR

Crie a política abaixo no mesmo namespace do Gateway. Troque os valores de
`namespace`, `name` e `ipBlocks` pelos valores aprovados no ambiente.

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

Para acrescentar uma origem, inclua outro item em `ipBlocks` e reaplique o
YAML. Para revogar uma origem, remova apenas o item correspondente e reaplique.
NÃO use `0.0.0.0/0`: isso desativa a restrição de origem.

## Passo 2 — Testar diretamente no VIP existente

O DNS interno deve resolver `orders-api.internal.example.com` para o VIP já
publicado. Faça a chamada a partir de uma máquina dentro de uma rede permitida:

```bash
export API_URL=http://orders-api.internal.example.com
export TOKEN='cole-aqui-um-jwt-valido-se-a-api-exigir-autenticacao'

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "${API_URL}/blue"
```

Caso o DNS ainda não esteja disponível para a máquina de teste, informe o
hostname e o VIP diretamente ao `curl`, sem modificar a máquina:

```bash
export API_HOST=orders-api.internal.example.com
export VIP=192.168.100.50

curl --resolve "${API_HOST}:80:${VIP}" -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://${API_HOST}/blue"
```

Execute a matriz de testes com o mesmo token válido:

| Origem da chamada | Resultado esperado |
|---|---:|
| IP ou rede presente em `ipBlocks` | 200, se as demais políticas também permitirem |
| IP ou rede fora de `ipBlocks` | 403 |
| IP permitido, sem credenciais exigidas pela API | 401 |

O teste de bloqueio DEVE ser feito a partir de uma origem realmente fora da
allowlist. Alterar somente o token não valida a restrição por IP.

## Diagnóstico

Confira se a política existe e se está apontando para o Gateway correto:

```bash
oc -n orders-api get authorizationpolicy orders-gateway-source-cidr -o yaml
oc -n orders-api get gateway orders-gateway
oc -n orders-api get httproute orders-api
```

| Sintoma | Causa provável | Ação |
|---|---|---|
| Origem permitida recebe 403 | O IP que chega ao Gateway é diferente do esperado, a rede não está no CIDR ou há outra política restritiva | Confirme com a equipe de rede o IP de origem observado pelo Gateway e revise todas as `AuthorizationPolicy` do namespace. |
| Origem fora da lista recebe 200 | A política aponta para outro Gateway, não foi aplicada no namespace correto ou existe uma configuração inesperada no caminho | Confira `targetRef`, namespace e a política efetivamente aplicada. |
| A chamada não chega à política | Hostname, listener, `HTTPRoute` ou VIP já existente está incorreto | Valide o Gateway, o `HTTPRoute`, o DNS interno e a conectividade até o VIP com a equipe responsável. |
| Origem permitida recebe 401 | A allowlist funcionou, mas a API exige JWT ou outra credencial | Obtenha uma credencial válida e repita o teste. |

## Recomendações para produção

- Mantenha `ipBlocks` pequeno, específico e aprovado pela segurança. Prefira
  `/32` quando a origem for fixa.
- Registre o responsável, a justificativa e a data de revisão de cada CIDR.
- Antes de ampliar uma rede, identifique se VPN, NAT ou proxy mudou o IP que o
  Gateway recebe.
- Use HTTPS no Gateway e mantenha JWT, roles e allowlist como camadas
  complementares de proteção.

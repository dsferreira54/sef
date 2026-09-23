# Hello World protegido por Red Hat Connectivity Link

Esta entrega instala uma demonstração de autenticação OIDC/JWT, autorização
RBAC por endpoint e allowlist de IP/CIDR no OpenShift. A aplicação não conhece
Keycloak, JWT, roles nem IP de origem: os bloqueios acontecem no gateway,
publicado diretamente por um VIP MetalLB, com `AuthorizationPolicy` do Istio e
`AuthPolicy` do Red Hat Connectivity Link (RHCL), sem alteração do código da
aplicação.

Comece por [docs/README.md](docs/README.md), incluindo o guia de
[allowlist de IP/CIDR](docs/source-cidr-allowlist.md). Os manifestos são
aplicados em ordem numérica em [manifests](manifests/).

> Os valores de `Secret` nunca são versionados. O hostname nos manifestos é
> específico deste laboratório e deve ser substituído antes de reutilizar em
> outro cluster.

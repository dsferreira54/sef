# Hello World protegido por Red Hat Connectivity Link

Esta entrega instala uma demonstração de autenticação OIDC/JWT, autorização
RBAC por endpoint e allowlist de IP/CIDR no OpenShift. A aplicação não conhece
Keycloak, JWT, roles nem IP de origem: os bloqueios acontecem no gateway,
publicado diretamente por um VIP MetalLB, com uma `AuthPolicy` do Red Hat
Connectivity Link (RHCL) anexada ao `HTTPRoute`, sem alteração do código da
aplicação.

Comece pelo [índice da documentação](docs/README.md). Os guias da
[primeira entrega — JWT/RBAC](docs/jwt-rbac.md) e da
[segunda entrega — MetalLB/IP/CIDR](docs/source-cidr-allowlist.md) são
independentes e conectados por links de pré-requisito. Os manifestos são
aplicados em ordem numérica em [manifests](manifests/).

> Os valores de `Secret` nunca são versionados. O hostname nos manifestos é
> específico deste laboratório e deve ser substituído antes de reutilizar em
> outro cluster.

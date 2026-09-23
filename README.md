# Hello World protegido por Red Hat Connectivity Link

Esta entrega instala uma demonstração de autenticação OIDC/JWT e autorização
RBAC por endpoint no OpenShift. A aplicação não conhece Keycloak, JWT nem roles:
o bloqueio acontece no `AuthPolicy` do Red Hat Connectivity Link (RHCL), anexado
ao `HTTPRoute` do Gateway API.

Comece por [docs/README.md](docs/README.md). Os manifestos são aplicados em
ordem numérica em [manifests](manifests/).

> Os valores de `Secret` nunca são versionados. O hostname nos manifestos é
> específico deste laboratório e deve ser substituído antes de reutilizar em
> outro cluster.

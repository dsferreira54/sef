# Documentação da demonstração RHCL

Esta demonstração foi entregue em duas etapas independentes, ambas sem alterar
o código da aplicação Hello World.

1. [OIDC/JWT e RBAC por endpoint](jwt-rbac.md) — primeira entrega: instala o
   Service Mesh e o RHCL, configura Keycloak, valida JWT e restringe `/blue` e
   `/red` por role.
2. [Exposição pelo MetalLB e allowlist de IP/CIDR](source-cidr-allowlist.md) —
   segunda entrega: publica o Gateway por VIP, preserva o IP de origem e limita
   o acesso ao `HTTPRoute`.

Leia o primeiro guia antes do segundo: a allowlist de rede pressupõe que o
Gateway, o `HTTPRoute` e o `AuthPolicy` de JWT/RBAC já existem.

Material de apoio:

- [Arquitetura e decisões](architecture.md)
- [Matriz de rastreabilidade](traceability.md)

Os manifestos em [../manifests](../manifests/) seguem a ordem numérica. Segredos
nunca são versionados; nomes de host e endereços de laboratório devem ser
substituídos em qualquer outro ambiente.

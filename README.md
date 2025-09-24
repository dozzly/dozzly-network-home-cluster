# GitOps (dozzly-cluster)
- clusters/dozzly-cluster: Flux entrypoint
- sources: Helm/Git sources
- infra: platform stack (ingress, cert-manager, longhorn, metallb, monitoring)
- config/dozzly-cluster: cluster-specific config (issuers, IP pools, StorageClasses, values)
- policies/gatekeeper: ConstraintTemplates + Constraints
- apps: application stacks

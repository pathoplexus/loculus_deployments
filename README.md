# loculus_deployments

We have a persistent instance running on AWS at https://productiontest.pathoplexus.org/ . It connects to a managed database running on Amazon RDS. It is configured according to https://github.com/pathoplexus/loculus_deployments/tree/main/deployments/pathoplexus and changes to that repo will change it, via ArgoCD sync at argocd.pathoplexus.org.

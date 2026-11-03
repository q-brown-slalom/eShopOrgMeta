# 0005. Deploy to Azure Container Apps via Aspire

- Status: Accepted
- Date: 2026-11-03
- Deciders: Q Brown

## Context

eShop is orchestrated by Aspire, and its AppHost already declares
`AddAzureContainerAppEnvironment("aca")`. We need a deployment target for Dev/Test/Prod that new
repos (offering web apps, admin app) can share.

## Options considered

1. **Azure Container Apps**: native Aspire deploy (`aspire deploy`), serverless scale, low ops burden. Less control than Kubernetes.
2. **AKS**: full control and portability. High ops burden, and the team would need Kubernetes expertise.
3. **Docker Compose on VMs**: simple, but there's no managed scaling or HA.

## Decision

Use Azure Container Apps, deployed with `aspire deploy` from GitHub Actions using OIDC. Use
managed Azure data services in deployed environments. See
[architecture/environments-and-deployment.md](../architecture/environments-and-deployment.md).

## Consequences

- Positive: the same model runs locally and in Azure, with minimal infrastructure code.
- Negative: we're tied to ACA features and limits. RabbitMQ needs either a hosting choice or a move to Service Bus.
- Follow-ups: decide on subscription/RG naming, managed RabbitMQ vs Service Bus, and the health endpoint exposure policy.

# MCSS AWS Service Catalog Reference Architectures

This repository contains governed AWS Service Catalog portfolio templates and product templates across multiple service domains. It provides a repeatable way to provision approved infrastructure without broad direct permissions to underlying AWS services.

> [!IMPORTANT]
> **Most important priorities:**
> 1. Keep all `TemplateURL` and `LoadTemplateFromURL` paths valid under `RepoRootURL`.
> 2. Enforce least privilege in launch-role templates.
> 3. Keep TagOptions aligned with organizational tag policies.
> 4. Validate create, update, and terminate flows before broad release.

## Table of Contents

- [Quick Concepts](#quick-concepts)
- [Portfolio and Product Index](#portfolio-and-product-index)
- [Deployment Critical Path](#deployment-critical-path)
- [Access and Role Model](#access-and-role-model)
- [Tagging and Governance](#tagging-and-governance)
- [Lifecycle and Change Management](#lifecycle-and-change-management)
- [Troubleshooting Quick Checks](#troubleshooting-quick-checks)
- [Security Priorities](#security-priorities)
- [Support](#support)
- [References](#references)

## **Quick Concepts**

- **Portfolio:** A governed collection of approved products.
- **Product:** A deployable configuration defined by product and resource templates.
- **Provisioning artifact:** A versioned release of a product (for example `v1.0`, `v1.1`).
- **Provisioned product:** A deployed instance of a product in an account and Region.
- **Launch constraint role:** The IAM role AWS Service Catalog assumes to create, update, and terminate product resources.
- **TagOptions:** Administrator-controlled tag keys and allowed values.

## **Portfolio and Product Index**

| Portfolio area | Description | Portfolio template | TagOption template | Product count |
| --- | --- | --- | --- | --- |
| EC2 | Approved Linux and Windows EC2 instance configurations. | `ec2/sc-portfolio-ec2-ec2Instances.yaml` | `ec2/sc-ec2-tagoptionLibrary.yaml` | 4 |
| 1BU | Serverless monitoring reference architecture for 1BU workloads. | `1bu/sc-portfolio-1bu-serverless-monitoring.yaml` | `1bu/sc-1bu-tagoptionLibrary.yaml` | 1 |
| Common tasks | Shared common-task products, including GitHub Actions setup. | `common-tasks/sc-portfolio-common-tasks.yaml` | `common-tasks/sc-common-tasks-tagoptionLibrary.yaml` | 1 |
| S3 | Governed S3 bucket configurations, including private, public, encryption, MFA, and lifecycle options. | `s3/sc-portfolio-s3.yaml` | `s3/sc-s3-tagoptionLibrary.yaml` | 5 |
| Serverless | Approved Lambda-based serverless application resources. | `serverless/sc-portfolio-serverless.yml` | `serverless/sc-serverless-tagoptionLibrary.yaml` | 1 |
| VPC IPv4 | VPC reference architectures using IPv4 addressing. | `vpc/vpc-ipv4/sc-portfolio-vpc-ipv4.yaml` | `vpc/vpc-ipv4/sc-vpc-ipv4-tagoptionLibrary.yaml` | 4 |
| VPC IPv6 dual stack | VPC reference architectures supporting IPv4 and IPv6 dual-stack networking. | `vpc/vpc-ipv6/sc-portfolio-vpc-ipv6dualstack.yaml` | `vpc/vpc-ipv6/sc-vpc-ipv6-tagoptionLibrary.yaml` | 9 |
| VPC endpoints | Create and configure VPC endpoints. | `vpc-endpoints/sc-portfolio-vpcendpoints-vpcEndpoint.yaml` | Not provided | 1 |

> [!TIP]
> Keep portfolio templates, product templates, and launch-role templates versioned together to avoid path drift and broken product associations.

## **Deployment Critical Path**

### 1. Prerequisites

- AWS Service Catalog and CloudFormation permissions in the target Region.
- Portfolio template, product templates, resource templates, and launch-role templates uploaded to S3.
- `RepoRootURL` ends with `/` and points to the repository root.
- Product dependencies (for example parameter-store values and service quotas) exist in the target account/Region.
- Linked IAM roles already exist if provided to portfolio parameters.

> [!WARNING]
> Invalid `RepoRootURL` values or incorrect relative paths are the most common deployment failure cause.

### 2. Upload templates

```bash
aws s3 cp <portfolio-template> s3://YOUR-BUCKET/<portfolio-template>
aws s3 cp <portfolio-tagoption-template> s3://YOUR-BUCKET/<portfolio-tagoption-template>
aws s3 cp iam/ s3://YOUR-BUCKET/iam/ --recursive
aws s3 cp <portfolio-folder>/ s3://YOUR-BUCKET/<portfolio-folder>/ --recursive
```

### 3. Validate the portfolio template

```bash
aws cloudformation validate-template --template-body file://<portfolio-template>
```

### 4. Deploy the portfolio stack

```bash
aws cloudformation deploy \
  --stack-name mcss-service-catalog-portfolio \
  --template-file <portfolio-template> \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides \
    PortfolioProvider="ITS Cloud Support" \
    PortfolioName="Your Service Catalog Portfolio" \
    PortfolioDescription="Approved reference architecture products" \
    LaunchRoleName="" \
    LinkedRole1="YOUR-END-USER-ROLE" \
    LinkedRole2="" \
    LinkedRole3="" \
    RepoRootURL="https://s3.amazonaws.com/YOUR-BUCKET/"
```

### 5. Deploy TagOptions

```bash
aws cloudformation deploy \
  --stack-name mcss-service-catalog-tagoptions \
  --template-file <portfolio-tagoption-template> \
  --parameter-overrides PortfolioId="port-xxxxxxxxxxxx"
```

## **Access and Role Model**

- **Linked roles (`LinkedRole1-3`):** control portfolio visibility and end-user access entry points.
- **Launch role (`LaunchRoleName`):** assumed by Service Catalog to provision, update, and terminate resources.

Minimum launch-role capability areas:

- CloudFormation stack operations.
- Resource create/update/delete/describe actions required by selected products.
- `iam:PassRole` for service roles required by product resources.
- Read access to templates, provisioning artifacts, and required parameter-store values.
- Logging, monitoring, and tagging actions required by product templates.

> [!WARNING]
> Do not use launch-role policies as a substitute for end-user Service Catalog permissions.

## **Tagging and Governance**

Use TagOptions and a controlled tagging standard to enforce governance and reporting consistency.

Recommended governance tags:

- `serviceEnvironment`
- `Chargeback` or `CostCenter`
- `Application`
- `Owner`
- `DataClassification`
- `ManagedBy`

Tagging rules:

- Keep key names and casing consistent.
- Keep allowed values aligned with AWS Organizations tag policies.
- Do not place sensitive data in tag values.

## **Lifecycle and Change Management**

When releasing a new product version:

1. Update and test the underlying resource template.
2. Upload an immutable artifact to S3.
3. Add a new provisioning artifact entry in the product template.
4. Deploy the updated stack.
5. Validate create/update/terminate in nonproduction before production rollout.

> [!IMPORTANT]
> Never replace content behind an existing artifact URL without publishing a new artifact version.

## **Troubleshooting Quick Checks**

| Symptom | First checks |
|---|---|
| Product not visible | Linked role association, end-user policy, account/Region, product active state |
| Template fetch failure | `RepoRootURL`, S3 object path/casing, object permissions |
| Launch role assume failure | Trust policy, role name match, source-account/source-ARN conditions |
| `AccessDenied` during provisioning | Denied API action in stack events, launch-role scope, `iam:PassRole` |
| Parameter resolution failure | Parameter exists in same account/Region, value format, launch-role read permissions |
| Tag conflicts | TagOption values vs org tag policy, portfolio/product TagOption overlap |

## **Security Priorities**

- Restrict launch-role scope to product-required actions only.
- Enforce naming rules for resources and IAM entities created by templates.
- Use approved administrative access paths; avoid direct inbound admin access unless explicitly approved.
- Review log retention and alerting actions for compliance.
- Make lifecycle changes through Service Catalog and CloudFormation to avoid drift.

## **Support**

- **Support team:** ITS Cloud Support
- **Support email:** ITSCloudSupport@mathematica-mpr.com

When requesting support, include account, Region, portfolio name, provisioned product name, product version, and failure event details.

## **References**

- [AWS Service Catalog Administrator Guide](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/what-is_concepts.html)
- [AWS Service Catalog User Guide](https://docs.aws.amazon.com/servicecatalog/latest/userguide/what-is-service-catalog.html)
- [AWS Service Catalog launch constraints](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/constraints-launch.html)
- [AWS Service Catalog TagOptions](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/tagoptions.html)

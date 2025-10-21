### Integration with `infra-gcp-platform`

This repository (`opa-policies-core`) provides OPA/Rego policies that are consumed by your Terraform repository `infra-gcp-platform` during CI/CD to gate changes.

- `infra-gcp-platform` produces a Terraform plan and renders it to JSON (`terraform show -json`).
- CI runs policy evaluation using Conftest or OPA against `opa-policies-core/policies`.
- If any `google_storage_bucket` is missing required labels (`environment`, `owner`), the build fails with clear messages.

Typical CI flow:

```bash
terraform init
terraform plan -out tfplan.binary
terraform show -json tfplan.binary > tfplan.json
conftest test --input terraform --policy ../opa-policies-core/policies tfplan.json
# or
opa eval -I -d ../opa-policies-core/policies -i tfplan.json "data.gcp.labels.deny"
```

#### Flow diagram

```mermaid
flowchart TD
  A[infra-gcp-platform: Terraform code change] --> B[CI: terraform init]
  B --> C[terraform plan -out tfplan.binary]
  C --> D[terraform show -json tfplan.binary > tfplan.json]
  D --> E[Policy Check: conftest/opa]
  E -->|Loads| F[opa-policies-core/policies]
  E -->|Evaluates| G[data.gcp.labels.allow / deny]
  G -->|deny non-empty| H[CI Fail: report missing labels]
  G -->|allow true & deny empty| I[CI Pass: proceed to apply]
```

#### Sequence diagram

```mermaid
sequenceDiagram
  participant Dev as Dev (infra-gcp-platform)
  participant CI as CI Pipeline
  participant TF as Terraform
  participant OPA as OPA/Conftest
  participant POL as opa-policies-core

  Dev->>CI: Push TF changes
  CI->>TF: terraform plan
  TF-->>CI: tfplan.binary
  CI->>TF: terraform show -json
  TF-->>CI: tfplan.json
  CI->>OPA: Evaluate policies with tfplan.json
  OPA->>POL: Load Rego policies
  OPA-->>CI: Results (allow/deny messages)
  alt Violations present
    CI-->>Dev: Fail build with deny messages
  else No violations
    CI-->>Dev: Pass and proceed to apply
  end
```

### Purpose Split

* `opa-policies-core` hosts OPA/Rego polices and tests (eg. enforcing required lables on cloud provider storage.
* `infra-gcp-platform` Owns Terraform code, planning, and CI/CD that applies infrastructure; it consumes the policies as a policy gate.

## Integration Point

- `infra-gcp-platform` produces a Terraform plan and converts it to JSON (`terraform show -json`).
- CI runs policy evaluation using Conftest or OPA against `opa-policies-core/policies`.
- Policies read `input.resource_changes` 
- If any `google_storage_bucket` is missing required labels (`environment`, `owner`), the build fails with clear messages.

Typical Lifecycle: 

- Policy authors evolve rules in `opa-polices-core` with opa test.
- Infra authors change Terraform
- CI in infra repo fetches/points to the policy bundle and gate merges/deploys.

Operational flow (CI/CD)

1. Developer pushes Terraform changes to `infra-gcp-platform
2. CI runs: terraform init, terraform plan, terraform show
3. CI runs policy check: conftest or opa eval
4. If violations exist, CI fails with actionable messages; otherwise proceeds to apply.

## Data/contract between repos
* Input document: Terraform plan JSON containing `resource_changes` entries.
* Policy package: gcp.labels
* Entry points: data.gcp.labels.allow (boolean), data.gcp.labels.deny (collection of violation messages)
* Current enforced rule(s) 

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

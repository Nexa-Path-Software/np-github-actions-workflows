# Reusable Workflows

Este repo centraliza los workflows reutilizables de GitHub Actions para Nexa Path Software.

## Objetivo

Mantener en un solo lugar la logica de despliegue y permitir que cualquier repo privado de la organizacion la consuma.

## Workflows incluidos

- `.github/workflows/terraform-reusable.yml`
  reusable workflow para Terraform usando OIDC y un archivo YAML de configuracion en el repo consumidor
- `.github/workflows/cloudformation-bootstrap-reusable.yml`
  reusable workflow para empaquetar y desplegar el bootstrap CloudFormation desde el repo caller

## Suposiciones del modelo

- El bootstrap de `c-usrs-mngmnt-aws` ya creo el rol hub en la cuenta management.
- Ese mismo bootstrap ya propago el rol `NexaPathGitHubActionsExecutionRole` a las cuentas destino.
- Cada repo consumidor tiene un archivo `.github/deploy.yml`.
- Cada repo consumidor tiene un workflow caller minimo que invoque este reusable workflow.

## Esquema esperado del YAML consumidor

```yaml
aws:
  region: us-east-1
  management_role_arn: arn:aws:iam::<account>:role/c-usrs-mngmnt-aws-github-actions-role
  execution_role_name: NexaPathGitHubActionsExecutionRole

branch_environment_map:
  DEV: dev
  PROD: prod

terraform:
  version: 1.8.5
  working_directory: terraform
  backend:
    bucket: c-usrs-mngmnt-aws-<account>-us-east-1-tf-state
    dynamodb_table: c-usrs-mngmnt-aws-terraform-locks
    kms_key_arn: arn:aws:kms:us-east-1:<account>:key/REEMPLAZAR
    region: us-east-1

environments:
  dev:
    account_id: "<account>"
    state_key: my-service/dev.tfstate
    env:
      TF_VAR_environment: dev
  prod:
    account_id: "<account>"
    state_key: my-service/prod.tfstate
    env:
      TF_VAR_environment: prod
```

## Que hace el reusable workflow

1. Hace checkout del repo caller.
2. Lee `.github/deploy.yml`.
3. Determina el ambiente usando la rama.
4. Exporta variables de entorno desde el YAML.
5. Asume el rol hub por OIDC.
6. Encadena `AssumeRole` hacia la cuenta destino.
7. Ejecuta `terraform init`, `plan` y `apply`.

## Que hace el reusable workflow de CloudFormation

1. Hace checkout del repo caller.
2. Lee `.github/bootstrap.yml`.
3. Asume el rol hub por OIDC en la cuenta management.
4. Empaqueta nested stacks y Lambda source.
5. Ejecuta `aws cloudformation deploy`.
6. Muestra los outputs del stack.

## Restriccion real de GitHub

El repo consumidor no puede quedarse solo con un YAML de configuracion.

GitHub exige un workflow caller dentro del repo para disparar la ejecucion.

Ese caller se mantiene minimo y toda la logica pesada vive en este repo.

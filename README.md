# Azure Container Apps Deploy Action

Deploys container images to Azure Container Apps

## Quick Start

```yaml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      
    steps:
      # 1. Checkout code
      - uses: actions/checkout@v4
      
      # 2. Authenticate with Azure
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      
      # 3. Build and push image (your choice of method)
      - name: Build and push
        run: |
          az acr login --name myregistry
          docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
          docker push myregistry.azurecr.io/myapp:${{ github.sha }}
      
      # 4. Deploy to Container App
      - uses: dymaxionlabs/container-apps-deploy-action@v1
        with:
          container_app_name: my-app
          resource_group: rg-production
          image: myregistry.azurecr.io/myapp:${{ github.sha }}
          env_vars: |
            APP_ENV=production
            LOG_LEVEL=info
          secrets: |
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            API_KEY=${{ secrets.API_KEY }}
```

## Inputs

| Input                 | Description                                                   | Required | Default          |
| --------------------- | ------------------------------------------------------------- | -------- | ---------------- |
| `container_app_name`  | Name of the Azure Container App or Job                        | ✅ Yes    | -                |
| `resource_group`      | Azure Resource Group name                                     | ✅ Yes    | -                |
| `image`               | Full image name with tag (e.g., `registry.azurecr.io/app:v1`) | ✅ Yes    | -                |
| `container_name`      | Container name within the app                                 | ❌ No     | Same as app name |
| `resource_type`       | Resource type: `app` or `job`                                 | ❌ No     | `app`            |
| `env_vars`            | Environment variables (KEY=value, one per line)               | ❌ No     | -                |
| `secrets`             | Secrets (KEY=value, one per line)                             | ❌ No     | -                |
| `remove_all_env_vars` | Remove all existing env vars before setting new ones          | ❌ No     | `false`          |
| `cpu`                 | CPU cores (e.g., `0.5`, `1.0`, `2.0`)                         | ❌ No     | -                |
| `memory`              | Memory in Gi (e.g., `1.0Gi`, `2.0Gi`)                         | ❌ No     | -                |
| `min_replicas`        | Minimum number of replicas                                    | ❌ No     | -                |
| `max_replicas`        | Maximum number of replicas                                    | ❌ No     | -                |

## Outputs

| Output | Description                                      |
| ------ | ------------------------------------------------ |
| `fqdn` | Fully qualified domain name of the Container App |
| `url`  | Full HTTPS URL of the Container App              |

## Examples

### Complete Workflow with Docker Build

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      
    steps:
      - uses: actions/checkout@v4
      
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      
      - uses: docker/setup-buildx-action@v3
      
      - name: Login to ACR
        run: az acr login --name myregistry
      
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: myregistry.azurecr.io/myapp:${{ github.sha }}
          cache-from: type=registry,ref=myregistry.azurecr.io/myapp:cache
          cache-to: type=registry,ref=myregistry.azurecr.io/myapp:cache,mode=max
      
      - uses: dymaxionlabs/container-apps-deploy-action@v1
        with:
          container_app_name: my-app
          resource_group: rg-production
          image: myregistry.azurecr.io/myapp:${{ github.sha }}
          cpu: "1.0"
          memory: "2.0Gi"
          min_replicas: "2"
          max_replicas: "10"
          env_vars: |
            APP_ENV=production
            LOG_LEVEL=info
            NEXTAUTH_URL=https://app.example.com
          secrets: |
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            NEXTAUTH_SECRET=${{ secrets.NEXTAUTH_SECRET }}
```

### Deploy Pre-built Image

If your image is already built (e.g., in a separate job or external CI):

```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

- uses: dymaxionlabs/container-apps-deploy-action@v1
  with:
    container_app_name: my-app
    resource_group: rg-production
    image: myregistry.azurecr.io/myapp:v1.2.3
    env_vars: |
      FEATURE_FLAG_X=true
```

### Deploy to Container App Job

```yaml
- uses: dymaxionlabs/container-apps-deploy-action@v1
  with:
    container_app_name: my-job
    resource_group: rg-production
    resource_type: job
    image: myregistry.azurecr.io/myjob:${{ github.sha }}
    env_vars: |
      SCHEDULE=0 0 * * *
      TIMEOUT=3600
```

### Replace All Environment Variables

```yaml
- uses: dymaxionlabs/container-apps-deploy-action@v1
  with:
    container_app_name: my-app
    resource_group: rg-production
    image: myregistry.azurecr.io/myapp:${{ github.sha }}
    remove_all_env_vars: "true"
    env_vars: |
      APP_ENV=production
      LOG_LEVEL=info
```

### Multi-Environment Deployment

```yaml
strategy:
  matrix:
    environment:
      - name: development
        app: my-app-dev
        rg: rg-dev
      - name: production
        app: my-app-prod
        rg: rg-prod

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  
  - uses: dymaxionlabs/container-apps-deploy-action@v1
    with:
      container_app_name: ${{ matrix.environment.app }}
      resource_group: ${{ matrix.environment.rg }}
      image: myregistry.azurecr.io/myapp:${{ github.sha }}
```## How It Works

1. **Checkout**: Checks out your repository code
2. **Azure Login**: Authenticates with Azure using OIDC
3. **Docker Setup**: Configures Docker Buildx for advanced builds
4. **ACR Login**: Authenticates with Azure Container Registry
5. **Build & Push**: Builds the Docker image with caching and pushes to ACR
6. **Prepare Variables**: Processes environment variables and secrets
7. **Deploy**: Updates the Azure Container App with the new image
8. **Summary**: Generates a deployment summary in GitHub Actions UI

## Tagging Strategy

Images are tagged with two tags:

1. `<environment>-<commit-sha>`: Specific version tag (e.g., `production-abc123`)
2. `<environment>`: Latest tag for the environment (e.g., `production`)

This allows for:
- Easy rollbacks to specific commits
- Quick access to latest environment version
- Clear deployment history

## Caching

The workflow uses Docker layer caching stored in ACR:

- **Cache Key**: `<repository>/cache:<environment>`
- **Mode**: `max` (caches all layers)
- **Benefits**: Faster subsequent builds, reduced CI time

## Concurrency Control

The workflow includes concurrency control:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ inputs.environment }}
  cancel-in-progress: true
```

This ensures:
- Only one deployment per environment runs at a time
- New deployments cancel in-progress ones
## Prerequisites

1. **Azure Container App** already created
2. **Azure authentication** set up (use `azure/login@v2` before this action)
3. **Container image** already built and pushed to a registry

This action **only deploys** - it doesn't create infrastructure or build images.

## Environment Variables vs Secrets

Both `env_vars` and `secrets` inputs set environment variables in your Container App. The difference is semantic:

- **`env_vars`**: Non-sensitive configuration (URLs, feature flags, log levels)
- **`secrets`**: Sensitive values (passwords, API keys, tokens)

Both are passed to the container the same way. Use `secrets` input for values from GitHub Secrets to keep them out of logs.

## How It Works

1. **Validates inputs** and prepares environment variables
2. **Runs `az containerapp update`** with your image and configuration
3. **Extracts FQDN** (for apps with ingress) and sets outputs
4. **Creates deployment summary** in GitHub Actions UI

## Permissions Required

Your workflow needs these permissions for Azure authentication:

```yaml
permissions:
  contents: read
  id-token: write
```

## Azure Setup

See [Azure Setup Guide](docs/AZURE_SETUP.md) for:
- Creating Container Apps
- Configuring federated credentials
- Setting up GitHub environments

## Versioning

Reference this action by version for stability:

```yaml
uses: dymaxionlabs/container-apps-deploy-action@v1      # Recommended
uses: dymaxionlabs/container-apps-deploy-action@v1.2.3  # Specific version
uses: dymaxionlabs/container-apps-deploy-action@main    # Latest (not recommended)
```

## Troubleshooting

### Authentication Failed

Ensure `azure/login@v2` runs before this action and has correct credentials.

### Container App Not Found

Verify:
- Container App name is correct
- Resource group name is correct  
- You're authenticated to the right subscription

### Image Pull Failed

Verify:
- Image name and tag are correct
- Container App has permissions to pull from your registry
- Image exists in the registry

## Related Actions

- **[azure/login](https://github.com/Azure/login)** - Authenticate with Azure
- **[docker/build-push-action](https://github.com/docker/build-push-action)** - Build and push images
- **[azure/container-apps-deploy-action](https://github.com/Azure/container-apps-deploy-action)** - Official Azure action (more features, more complex)

### Image Push Failed

- Verify ACR name is correct
## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

MIT License - see [LICENSE](LICENSE) file for details.

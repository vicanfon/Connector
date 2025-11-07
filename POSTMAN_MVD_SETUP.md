# EDC MVD (Minimum Viable Dataspace) - Postman Setup Guide

## Overview

This guide is specifically for testing the Eclipse Dataspace Connector deployed as part of the **Minimum Viable Dataspace (MVD)** with nginx ingress in Kubernetes.

## Files

- `postman-mvd-collection.json` - MVD-optimized API collection
- `postman-mvd-k8s-environment.json` - Pre-configured environment variables

**Important**: All management API paths include `/api/management/` after the connector base path. For example:
- Full URL: `http://localhost/consumer/cp/api/management/v3/assets`
- Not: `http://localhost/consumer/cp/v3/assets`

## Quick Start

### 1. Import Collection and Environment

1. Open Postman
2. Import `postman-mvd-collection.json`
3. Import `postman-mvd-k8s-environment.json`
4. Select the **MVD K8S Environment** from the environment dropdown

### 2. Configure nginx Access

The MVD deployment uses nginx ingress to route traffic. Set up port forwarding or ingress access:

#### Option A: Port Forward nginx Ingress (Recommended for local testing)

```bash
# Forward nginx ingress controller
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 80:80

# Or if you need to specify a different local port
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

If using a different local port, update the `nginx_host` variable:
```
nginx_host = http://localhost:8080
```

#### Option B: Direct Service Access (from within cluster)

If running tests from a pod inside the cluster, no port forwarding needed. The environment is already configured with proper k8s service DNS names.

### 3. Verify Environment Variables

The environment should have these variables pre-configured:

| Variable | Value | Description |
|----------|-------|-------------|
| `nginx_host` | `http://localhost` | Nginx ingress base URL |
| `management_url` | `{{nginx_host}}/consumer/cp` | Consumer management API |
| `provider_management_url` | `{{nginx_host}}/provider-qna/cp` | Provider management API |
| `catalog_server_url` | `{{nginx_host}}/provider-catalog-server/cp` | Catalog server API |
| `auth_token` | `c3VwZXItdXNlcjpzdXBlci1zZWNyZXQta2V5Cg==` | API key (base64) |
| `provider_dsp_url` | `http://provider-qna-controlplane:8082/api/dsp` | Provider DSP endpoint (internal) |
| `catalog_server_dsp_url` | `http://provider-catalog-server-controlplane:8082/api/dsp` | Catalog server DSP |
| `consumer_id` | `did:web:consumer-identityhub%3A7083:consumer` | Consumer DID |
| `provider_id` | `did:web:provider-identityhub%3A7083:provider` | Provider DID |

### 4. Understanding URL Structure

The MVD uses two types of URLs:

**Management URLs** (accessed via nginx):
- Used for direct API management operations
- Example: `http://localhost/consumer/cp/api/management/v3/assets`
- Accessed through nginx ingress from outside the cluster

**DSP URLs** (internal k8s service DNS):
- Used for inter-connector communication (Dataspace Protocol)
- Example: `http://provider-qna-controlplane:8082/api/dsp`
- Must include `/api/dsp` path for protocol endpoints
- Only accessible within the k8s cluster

### 5. Test Connectivity

Run the health check requests:
1. **01 - Observability** → **Consumer - Health Check**
2. **01 - Observability** → **Provider - Health Check**

Expected response: `200 OK`

## MVD Architecture

The MVD consists of several connectors:

```
┌─────────────────────────────────────────────────────────────┐
│                      nginx Ingress                          │
│  http://localhost/consumer/*                                │
│  http://localhost/provider-qna/*                            │
│  http://localhost/provider-manufacturing/*                  │
│  http://localhost/provider-catalog-server/*                 │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼────────────────────┐
        │                     │                    │
   ┌────▼─────┐        ┌─────▼──────┐      ┌─────▼──────┐
   │ Consumer │        │ Provider   │      │  Provider  │
   │          │        │    QnA     │      │Manufacturing│
   └──────────┘        └────────────┘      └────────────┘
                              │
                       ┌──────▼───────┐
                       │Catalog Server│
                       │   (Federated │
                       │    Catalog)  │
                       └──────────────┘
```

### Key Components

- **Consumer Connector**: Initiates catalog requests, negotiations, and transfers
- **Provider QnA Connector**: Provides question-and-answer datasets
- **Provider Manufacturing Connector**: Provides manufacturing data
- **Catalog Server**: Federates catalogs from multiple providers

## End-to-End Testing Workflow

### Phase 1: Provider Setup (Folders 02-04)

#### Step 1: Create Asset on Provider

**02 - Provider - Asset Management** → **Create Asset**

This creates a new asset. The response includes an `@id` which is automatically saved to `asset_id` environment variable.

#### Step 2: Create Policy

**03 - Provider - Policy Definitions** → **Create Unrestricted Policy**

Creates an unrestricted policy for testing. Policy ID is saved to `policy_id`.

#### Step 3: Create Contract Definition

**04 - Provider - Contract Definitions** → **Create Contract Definition**

Links assets with policies. The contract definition makes assets discoverable in the catalog.

### Phase 2: Consumer Discovery (Folder 05)

#### Step 4: Request Catalog

**05 - Consumer - Request Catalog** → **Request Catalog from Provider**

The consumer requests the provider's catalog to discover available datasets.

Key points:
- `counterPartyAddress` uses internal k8s service DNS (`{{provider_dsp_url}}`)
- `counterPartyId` uses DIDs for identification
- Response contains datasets, offers, and policy information

**Alternative**: Request from Catalog Server
```
05 - Consumer - Request Catalog → Request Catalog from Catalog Server
```

This queries the federated catalog which aggregates offerings from multiple providers.

**Important**: The catalog server shares the same participant ID (`provider_id`) as the regular provider. Only the DSP address differs:
- Provider DSP: `http://provider-qna-controlplane:8082/api/dsp`
- Catalog Server DSP: `http://provider-catalog-server-controlplane:8082/api/dsp`

### Phase 3: Contract Negotiation (Folder 06)

#### Step 5: Initiate Negotiation

**06 - Consumer - Contract Negotiation** → **Initiate Contract Negotiation**

Before running this request:
1. Copy the `@id` from an offer in the catalog response
2. Copy the `target` (asset ID) from the offer
3. Update the request body with these values:
   - `policy.@id` = offer ID
   - `policy.target` = asset ID

The negotiation ID is automatically saved to `contract_negotiation_id`.

#### Step 6: Monitor Negotiation

**06 - Consumer - Contract Negotiation** → **Get Negotiation State**

Keep polling until state is `FINALIZED`.

#### Step 7: Get Agreement

**06 - Consumer - Contract Negotiation** → **Get Agreement**

Once finalized, retrieve the contract agreement. The agreement ID is saved to `contract_agreement_id`.

### Phase 4: Data Transfer (Folder 07)

#### Step 8: Initiate Transfer

**07 - Consumer - Transfer Process** → **Initiate Transfer (HttpProxy)**

Before running:
1. Ensure you have the `contract_agreement_id` from Phase 3
2. Update `assetId` in the request body with the asset ID from the catalog

The transfer process ID is saved to `transfer_process_id`.

#### Step 9: Monitor Transfer

**07 - Consumer - Transfer Process** → **Get Transfer State**

Poll until state is `STARTED` or `COMPLETED`.

### Phase 5: Data Access (Folders 08-09)

#### Step 10: Get EDR (Endpoint Data Reference)

**08 - Consumer - EDR Access** → **Get EDR Data Address**

This returns the endpoint URL and authorization token needed to access the data.

Response example:
```json
{
  "@type": "DataAddress",
  "endpoint": "http://provider-qna-dataplane:8185/api/public",
  "authorization": "eyJhbGc...",
  "endpointType": "https://w3id.org/edc/v0.0.1/ns/HttpData"
}
```

#### Step 11: Access Data

**09 - Data Access** → **Access Data with EDR Token**

1. Copy the `endpoint` URL from the EDR response
2. Copy the `authorization` token
3. Update the request:
   - URL: Use the endpoint from EDR (may need to adapt for local access)
   - Header: `Authorization: <token-from-EDR>`

## Authentication

The MVD uses a hardcoded API key for testing:

```
Key: super-user:super-secret-key
Base64: c3VwZXItdXNlcjpzdXBlci1zZWNyZXQta2V5Cg==
```

All management API requests use the header:
```
x-api-key: c3VwZXItdXNlcjpzdXBlci1zZWNyZXQta2V5Cg==
```

## Troubleshooting

### Connection Refused to nginx

**Problem**: Cannot connect to `http://localhost/consumer/cp`

**Solution**:
```bash
# Check nginx ingress is running
kubectl get pods -n ingress-nginx

# Check ingress routes
kubectl get ingress -A

# Verify port forward
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 80:80
```

### 404 Not Found

**Problem**: nginx returns 404

**Solution**:
- Verify ingress rules are created: `kubectl get ingress -n mvd`
- Check that connector pods are running: `kubectl get pods -n mvd`
- Ensure host header matches ingress rules (usually `localhost` works)

### 503 Service Unavailable

**Problem**: nginx can't reach backend services

**Solution**:
```bash
# Check connector pods status
kubectl get pods -n mvd -l app=consumer-controlplane
kubectl get pods -n mvd -l app=provider-qna-controlplane

# Check pod logs
kubectl logs -f -n mvd deployment/consumer-controlplane
```

### Empty Catalog Response

**Problem**: Catalog request returns no datasets

**Solution**:
1. Verify provider has assets, policies, and contract definitions
2. Run queries on provider:
   - **02 - Provider - Asset Management** → **Query Assets**
   - **03 - Provider - Policy Definitions** → **Query Policy Definitions**
   - **04 - Provider - Contract Definitions** → **Query Contract Definitions**
3. Check contract definition `assetsSelector` matches your assets

### Contract Negotiation Fails

**Problem**: Negotiation stuck or fails

**Solution**:
1. Verify the offer ID and asset ID are from the actual catalog response
2. Check the `counterPartyId` matches the provider's DID exactly
3. Ensure `counterPartyAddress` uses internal k8s DNS (not nginx URL)
4. Review connector logs for errors

### Cannot Access Data with EDR

**Problem**: EDR endpoint not accessible

**Solution**:
- The EDR endpoint might be an internal k8s service URL
- For local testing, you may need to port-forward the data plane:
  ```bash
  kubectl port-forward -n mvd service/provider-qna-dataplane 8185:8185
  ```
- Then use `http://localhost:8185` instead of the k8s service URL

## Advanced: Using with Actual Domains

If your MVD is deployed with real domain names (e.g., `consumer.example.com`):

1. Update `nginx_host` variable:
   ```
   nginx_host = https://example.com
   ```

2. Update management URLs accordingly:
   ```
   management_url = https://consumer.example.com/cp
   provider_management_url = https://provider.example.com/cp
   ```

3. Ensure DNS resolves correctly
4. Use proper TLS certificates

## Useful kubectl Commands

```bash
# View all MVD resources
kubectl get all -n mvd

# Check ingress configuration
kubectl describe ingress -n mvd

# View connector logs (consumer)
kubectl logs -f -n mvd deployment/consumer-controlplane

# View connector logs (provider)
kubectl logs -f -n mvd deployment/provider-qna-controlplane

# Check nginx ingress logs
kubectl logs -f -n ingress-nginx deployment/ingress-nginx-controller

# Restart a connector
kubectl rollout restart -n mvd deployment/consumer-controlplane

# View ConfigMaps (configuration)
kubectl get configmap -n mvd

# Check secrets
kubectl get secrets -n mvd
```

## Running the MVD

If you haven't deployed the MVD yet:

```bash
# Clone the MVD repository
git clone https://github.com/eclipse-edc/MinimumViableDataspace.git
cd MinimumViableDataspace

# Deploy to k8s
./gradlew -p deployment build
kubectl apply -f deployment/terraform/modules/kubernetes/k8s/

# Seed initial data
./deployment/seed-k8s.sh

# Setup port forwarding
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 80:80
```

## Collection Features

### Auto-Save Variables

The collection includes test scripts that automatically save IDs to environment variables:

- Creating an asset → saves `asset_id`
- Creating a policy → saves `policy_id`
- Creating a contract definition → saves `contract_definition_id`
- Initiating negotiation → saves `contract_negotiation_id`
- Getting agreement → saves `contract_agreement_id`
- Initiating transfer → saves `transfer_process_id`

This allows you to run the workflow without manually copying IDs between requests.

### Request Organization

Folders are numbered (01-09) to indicate the recommended execution order for a complete data exchange flow.

## Additional Resources

- [MVD GitHub Repository](https://github.com/eclipse-edc/MinimumViableDataspace)
- [MVD Documentation](https://github.com/eclipse-edc/MinimumViableDataspace/blob/main/README.md)
- [EDC Documentation](https://eclipse-edc.github.io/docs)
- [Dataspace Protocol](https://docs.internationaldataspaces.org/)

## Notes

- The collection uses v3 Management APIs (stable version)
- DSP endpoints use internal k8s service DNS for inter-connector communication
- Management endpoints are accessed via nginx ingress
- All IDs use JSON-LD format with `@context`, `@type`, and `@id`
- DIDs (Decentralized Identifiers) are used for participant identification
- The MVD setup includes IdentityHub for DID resolution

Happy dataspace testing!

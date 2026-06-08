# Model as a Service (MaaS) on Red Hat OpenShift AI 3.4

## Setup and Validation Guide

This guide provides instructions for installing, configuring, and demonstrating Model as a Service (MaaS) on Red Hat OpenShift AI (RHOAI) 3.4 for Proof of Concepts.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Official Documentation](#official-documentation)
- [Installation Steps](#installation-steps)
  - [1. Order Environment](#1-order-environment)
  - [2. Kuadrant and Authentication Setup](#2-kuadrant-and-authentication-setup)
  - [3. Enable User Workload Monitoring](#3-enable-user-workload-monitoring)
  - [4. PostgreSQL Database Setup](#4-postgresql-database-setup)
  - [5. Gateway Configuration](#5-gateway-configuration)
  - [6. Configure TLS for MaaS](#6-configure-tls-for-maas)
  - [7. Configure Data Science Cluster](#7-configure-data-science-cluster)
- [Troubleshooting](#troubleshooting)
- [Deploy Sample Models](#deploy-sample-models)
- [Validation and Demo](#validation-and-demo)
  - [Create API Token](#create-api-token)
  - [List Available Models](#list-available-models)
  - [Test Model Inference](#test-model-inference)
  - [Test Authorization Enforcement](#test-authorization-enforcement)
  - [Test Rate Limiting](#test-rate-limiting)
- [Understanding MaaS API Tokens](#understanding-maas-api-tokens)

---

## Overview

Model as a Service (MaaS) enables organizations to provide programmatic access to Large Language Models (LLMs) through a secure, token-based authentication system with built-in rate limiting and audit logging capabilities.

## Prerequisites

- Access to Red Hat OpenShift cluster
- Cluster admin privileges
- Red Hat OpenShift AI 3.4 installed

## Official Documentation

- **Order Environment**: [Red Hat Demo Platform](https://catalog.demo.redhat.com/catalog/all?item=babylon-catalog-prod/published.openshift-ai-v3.prod&utm_source=webapp&utm_medium=share-link)
- **MaaS Documentation**: [Deploy and Manage Models as a Service](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/deploy-and-manage-models-as-a-service_maas#configure-tls-for-maas_maas-deploy)

---

## Installation Steps

### 1. Order Environment

Order your OpenShift AI environment from the Red Hat Demo Platform:

```
https://catalog.demo.redhat.com/catalog/all?item=babylon-catalog-prod/published.openshift-ai-v3.prod&utm_source=webapp&utm_medium=share-link
```

### 2. Kuadrant and Authentication Setup

#### Create Namespace and Install Operator

1. Create the `kuadrant-system` namespace
2. Install the **Red Hat Connectivity Link Operator**

#### Follow Authentication Configuration

Follow the official guide: [Configuring Authentication for LLM-D](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/deploy_models_using_distributed_inference_with_llm-d/configuring-authentication-for-llmd_distributed-inference)

#### Create Kuadrant Custom Resource

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
```

#### Configure Authorino Service Certificate

```bash
oc annotate svc/authorino-authorino-authorization \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert \
  -n kuadrant-system
```

#### Create Authorino Instance

```bash
oc apply -f - <<EOF
apiVersion: operator.authorino.kuadrant.io/v1beta1
kind: Authorino
metadata:
  name: authorino
  namespace: kuadrant-system
spec:
  replicas: 1
  clusterWide: true
  listener:
    tls:
      enabled: true
      certSecretRef:
        name: authorino-server-cert
  oidcServer:
    tls:
      enabled: false
EOF
```

#### Restart Related Pods

```bash
oc delete pod -n redhat-ods-applications -l app=odh-model-controller
oc delete pod -n redhat-ods-applications -l control-plane=kserve-controller-manager
```

### 3. Enable User Workload Monitoring

Reference: [Enabling Monitoring for User-Defined Projects](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html/configuring_user_workload_monitoring/preparing-to-configure-the-monitoring-stack-uwm#enabling-monitoring-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

### 4. PostgreSQL Database Setup

In Red Hat OpenShift AI, API keys are used to provide programmatic access to models through MaaS subscriptions. A PostgreSQL database is required for API key lifecycle management.

#### Clone Repository and Run Setup Script

```bash
# Clone the MaaS repository
git clone https://github.com/opendatahub-io/models-as-a-service.git
cd models-as-a-service

# Run database setup script
NAMESPACE=redhat-ods-applications ./scripts/setup-database.sh
```

References:
- [MaaS Scripts](https://github.com/opendatahub-io/models-as-a-service/tree/main/scripts)
- [Database Setup Documentation](https://opendatahub-io.github.io/models-as-a-service/v0.1.0/install/maas-setup/#database-setup)

#### Configure Database Connection Secret

After the database is set up, retrieve credentials from the `postgres-creds` secret and create the MaaS database configuration:

Reference: [Configure PostgreSQL Secret for MaaS](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/deploy-and-manage-models-as-a-service_maas#configure-postgresql-secret-for-maas_maas-deploy)

```bash
# Generic format
oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL=postgresql://<username>:<password>@<hostname>:<port>/<database>?sslmode=require

# Example with actual credentials
oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL='postgresql://maas:cDBXsjuOOUmX4xBUybfZ2Uig76SFnTG0@172.30.11.103:5432/maas?sslmode=require'
```

### 5. Gateway Configuration

Reference: [Configuring Ingress Cluster Traffic](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/ingress_and_load_balancing/configuring-ingress-cluster-traffic#nw-ingress-gateway-api-enable_ingress-gateway-api)

#### Set Environment Variables

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')

# Use default ingress cert for HTTPS, or set CERT_NAME to your TLS secret name
CERT_NAME=${CERT_NAME:-$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}' 2>/dev/null)}
[[ -z "$CERT_NAME" ]] && CERT_NAME="router-certs-default"
```

#### Create Gateway Resource

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
spec:
  gatewayClassName: openshift-default
  listeners:
   - name: http
     hostname: maas.apps.ocp.fsxvj.sandbox1403.opentlc.com
     port: 80
     protocol: HTTP
     allowedRoutes:
       namespaces:
         from: All
   - name: https
     hostname: maas.apps.ocp.fsxvj.sandbox1403.opentlc.com
     port: 443
     protocol: HTTPS
     allowedRoutes:
       namespaces:
         from: All
     tls:
       certificateRefs:
       - group: ""
         kind: Secret
         name: cert-manager-ingress-cert
       mode: Terminate
```

> **Note**: Update the `hostname` field with your actual cluster domain.

### 6. Configure TLS for MaaS

Reference: [Configure TLS for MaaS](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/deploy-and-manage-models-as-a-service_maas#configure-tls-for-maas_maas-deploy)

```bash
# From the cloned models-as-a-service repository
./scripts/setup-authorino-tls.sh
```

### 7. Configure Data Science Cluster

```bash
kubectl apply -f - <<EOF
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    kserve:
      managementState: Managed
      rawDeploymentServiceConfig: Headed
      modelsAsService:
          managementState: Managed
    dashboard:
      managementState: Managed
EOF
```

---

## Troubleshooting

### MaaS API Pod Crashes

If the `maas-api` pod starts crashing after setup:

1. Check the pod logs for errors
2. Edit the `maas-db-config` secret in the `redhat-ods-applications` namespace
3. Set `sslmode` to `disable` if SSL connection issues are detected

```bash
# Check pod logs
oc logs -n redhat-ods-applications -l app=maas-api

# Edit the secret if needed
oc edit secret maas-db-config -n redhat-ods-applications
```

![Troubleshooting Step 1](images/troubleshooting-1.png)

![Troubleshooting Step 2](images/troubleshooting-2.png)

---

## Deploy Sample Models

Reference: [Model Setup Documentation](https://opendatahub-io.github.io/models-as-a-service/v0.1.0/install/model-setup/)

Follow the steps in the official documentation to deploy sample models:

1. [Model Setup Guide](https://opendatahub-io.github.io/models-as-a-service/v0.1.0/install/model-setup/)
2. [Validation Guide](https://opendatahub-io.github.io/models-as-a-service/v0.1.0/install/validation/)

---

## Validation and Demo

![Authorization Policy](images/authorization-policy.png)

![Subscriptions](images/subscriptions.png)

![Asset Endpoint](images/asset-endpoint.png)

### Create API Token

Example API Token: `sk-oai-NoBsUt7tdZ3QH`

You can create API tokens through the OpenShift AI dashboard. These tokens enable programmatic access to models.

```bash
# Set your API key and host
export API_KEY="sk-oai-NoBsUt7tdZ3QH"
export HOST="https://maas.apps.ocp.fsxvj.sandbox1403.opentlc.com"
```

### List Available Models

```bash
MODELS=$(curl -sSk ${HOST}/maas-api/v1/models \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_KEY" | jq -r .) && \
echo $MODELS | jq . && \
MODEL_NAME=$(echo $MODELS | jq -r '.data[0].id') && \
MODEL_URL=$(echo $MODELS | jq -r '.data[0].url') && \
echo "Model URL: $MODEL_URL"
```

**Expected Output:**

```json
{
  "data": [
    {
      "id": "facebook/opt-125m",
      "created": 1779544784,
      "object": "model",
      "owned_by": "llm/facebook-opt-125m-simulated",
      "kind": "LLMInferenceService",
      "url": "https://maas.apps.ocp.fsxvj.sandbox1403.opentlc.com/llm/facebook-opt-125m-simulated",
      "ready": true,
      "modelDetails": {
        "description": "A simulated OPT-125M model for free-tier testing",
        "displayName": "Facebook OPT 125M (Simulated)"
      },
      "subscriptions": [
        {
          "name": "simulator-subscription",
          "displayName": "Simulator Subscription (Free)",
          "description": "Free-tier subscription with 100 tokens/min rate limit"
        }
      ]
    }
  ],
  "object": "list"
}
```

### Test Model Inference

```bash
curl -sSk -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
  "${MODEL_URL}/v1/completions" | jq
```

**Expected Output:**

```json
{
  "id": "cmpl-ce171fb5-d4e8-5e91-8c86-be8ce0632927",
  "created": 1780749104,
  "model": "facebook/opt-125m",
  "usage": {
    "prompt_tokens": 1,
    "completion_tokens": 50,
    "total_tokens": 51
  },
  "object": "text_completion",
  "kv_transfer_params": null,
  "choices": [
    {
      "index": 0,
      "finish_reason": "length",
      "text": "Alas, poor Yorick! I knew him, Horatio: A fellow of infinite jest To be or not to be that is the question. Testing@, #testing 1$ ,2%,3^, [4&*5], 6~"
    }
  ]
}
```

### Test Authorization Enforcement

Verify that requests without proper authorization are rejected:

```bash
curl -sSk -H "Content-Type: application/json" \
  -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
  "${MODEL_URL}/v1/completions" -v
```

**Expected Response:**

```
< HTTP/2 401
< www-authenticate: request.headers.authorization realm="api-keys"
< www-authenticate: Bearer realm="kubernetes-tokens"
< x-ext-auth-reason: Authentication required
```

This confirms that the authorization layer is properly enforcing authentication.

### Test Rate Limiting

Verify that rate limiting is working correctly by making multiple rapid requests:

```bash
for i in {1..16}; do
  curl -sSk -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
    "${MODEL_URL}/v1/completions"
done
```

**Expected Output:**

```
200
200
200
200
200
200
429
429
429
429
429
429
429
429
429
429
```

The first few requests succeed (HTTP 200), then rate limiting kicks in (HTTP 429 - Too Many Requests), demonstrating that the free-tier subscription's 100 tokens/min rate limit is being enforced.

---

## Understanding MaaS API Tokens

### Why API Tokens Matter

MaaS API tokens provide several enterprise-grade capabilities:

1. **Authenticated Traceable Access**: Every request is attributable to a specific user identity and project
2. **Rate Limiting**: Enforce usage quotas and prevent resource exhaustion
3. **Access Audit Logging**: Complete audit trail of who accessed which models and when
4. **Enterprise Security Compliance**: Ensures adherence to organizational security policies

### Usage Scenarios

API tokens can be used in various development environments:

- Red Hat Dev Spaces
- Local development environments
- CI/CD pipelines
- Production applications
- Third-party integrations

### Token Security Best Practices

- Store tokens securely (e.g., in secrets managers)
- Rotate tokens regularly
- Never commit tokens to source control
- Use environment variables for token injection
- Monitor token usage for anomalies

---

## Summary

This guide covered the complete setup and validation of Model as a Service on Red Hat OpenShift AI 3.4, including:

- Installation and configuration of all required components
- Database setup for API key management
- Gateway and TLS configuration
- Model deployment
- Comprehensive validation including authentication, authorization, and rate limiting

For additional information, refer to the [official Red Hat OpenShift AI documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4).

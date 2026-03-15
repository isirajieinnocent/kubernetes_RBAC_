# Kubernetes RBAC Demo with kube-rbac-proxy
This repository demonstrates how to secure a Kubernetes application using Role-Based Access Control (RBAC) and the `kube-rbac-proxy`. The setup involves deploying a simple 'grade-submission-api' and using a sidecar proxy to authenticate and authorize incoming requests against the Kubernetes API server.

## Core Concepts

*   **RBAC**: Defining permissions for services using `ClusterRole` and `ClusterRoleBinding`.
*   **Service Accounts**: Creating identities for applications within the cluster.
*   **Sidecar Pattern**: Using `kube-rbac-proxy` as a sidecar container to handle authentication and authorization.
*   **Authentication Flow**: A client uses a Service Account token to authenticate. The proxy validates this token with the Kubernetes API and checks the associated permissions before forwarding the request to the application.

## Components

This project is configured through the following Kubernetes manifest files:

*   `namespace.yaml`: Creates a dedicated `grade-demo` namespace to isolate the application components.
*   `rbac-proxy.yaml`:
    *   `ServiceAccount (grade-service-proxy)`: An identity for the proxy itself.
    *   `ClusterRole (grade-service-proxy-role)`: Grants the proxy permission to perform `tokenreviews` and `subjectaccessreviews`, which are necessary to validate client tokens and check their permissions.
    *   `ClusterRoleBinding (grade-submission-proxy-binding)`: Binds the role to the proxy's service account.
*   `rbac.yaml`:
    *   `ServiceAccount (grade-service-account)`: The identity that clients will use to access the API.
    *   `ClusterRole (grade-service-role)`: Grants `GET` and `POST` permissions on all non-resource URLs (`/*`). This is the permission that will be checked by the proxy.
    *   `ClusterRoleBinding (grade-submission-binding)`: Binds the role to the client's service account.
*   `secret.yaml`: Creates a `Secret` of type `kubernetes.io/service-account-token` for the `grade-service-account`, providing a stable token for clients to use.
*   `grade-submission-api-deployment.yaml`:
    *   Deploys the `grade-submission-api` application container.
    *   Deploys the `kube-rbac-proxy` as a sidecar container in the same pod. The proxy listens on port 8443 and forwards authorized requests to the application on port 3000.
*   `grade-submission-api-service.yaml`: Exposes the `kube-rbac-proxy`'s secure port (8443) to outside the cluster using a `NodePort` service on port 31000.

## Deployment and Usage

### 1. Clone the Repository
```sh
git clone https://github.com/isirajieinnocent/kubernetes_rbac_.git
cd kubernetes_rbac_
```

### 2. Apply the Manifests
Apply all the configuration files to your Kubernetes cluster. This will create the namespace, service accounts, roles, bindings, deployment, service, and secret.
```sh
kubectl apply -f .
```

### 3. Retrieve the Client Token
Get the token that was created for the `grade-service-account`. This token will be used to authenticate requests.
```sh
TOKEN=$(kubectl get secret grade-sa-token -n grade-demo -o jsonpath='{.data.token}' | base64 -d)
```

### 4. Get Node IP and Port
Find the IP address of one of your cluster nodes. The `NodePort` is set to `31000` in `grade-submission-api-service.yaml`.
```sh
# Get the IP of a node (this example gets the first one)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# The NodePort is hardcoded to 31000
NODE_PORT=31000
```

### 5. Access the Service
Use `curl` to make a request to the service, passing the token in the `Authorization` header. The request is sent to the `kube-rbac-proxy`, which will validate the token before forwarding it to the API.

*Note: The `-k` or `--insecure` flag is used because the proxy uses a self-signed certificate by default.*
```sh
curl -k -H "Authorization: Bearer $TOKEN" https://$NODE_IP:$NODE_PORT/
```
If successful, the proxy will authorize the request, forward it to the `grade-submission-api`, and you will receive a response from the application.

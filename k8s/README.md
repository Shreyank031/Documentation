# kubectl

# What happens when you type `kubectl create/run/get`

## 1. Validation and generators:

- The first thing that kubectl will do is perform some **client-side validation**.

  - This ensures that requests that will always fail will fail fast and not be sent to kube-apiserver. 

  - This improves system performance by reducing unnecessary load.	

- After validation, kubectl begins assembling the HTTP request it will send to kube-apiserver. 

- All attempts to access or change state in the Kubernetes system goes through the API server, which in turns communicates with etcd. The kubectl client is no different. 
  - To construct the HTTP request, kubectl uses something called generators which is an abstraction that takes care of serialization.

- Internal working:
  - Generators are kubectl commands that generate resources based on a set of inputs.

  - When you type `kubectl get pods`, `kubectl` internally translates this command into an HTTP request format that the Kubernetes API server can understand. This translation process is handled by the generator abstraction within `kubectl`.

  - The generator takes the `kubectl` command along with any specified flags, options, and arguments, and converts them into the appropriate HTTP request format. This includes specifying the HTTP method (typically GET for a `get` command), the resource endpoint (e.g., `/api/v1/pods`), any query parameters (such as namespace, label selectors, field selectors), and other necessary details.

  - For example, the `kubectl get pods` command might be translated by the generator into an HTTP GET request with a URL like `https://kube-apiserver/api/v1/namespaces/default/pods`, where `kube-apiserver` is the hostname or IP address of the Kubernetes API server.

  - Once the HTTP request is constructed, `kubectl` sends it to the Kubernetes API server over the network. The API server receives the request, processes it, and responds accordingly, returning the requested information about pods, which `kubectl` then formats and displays to the user in the terminal.

  - This translation process is transparent to the user but is crucial for enabling `kubectl` to interact with the Kubernetes API server effectively and perform various cluster management tasks.

- `kubectl run` is a versatile command that can create various types of resources in a Kubernetes cluster, not just Deployments. When you use `kubectl run` without explicitly specifying the `generator` name using the `--generator` flag, `kubectl` infers the resource type based on the provided command parameters.

For instance, if you run:

```bash
kubectl run my-pod --image=my-image
```

Without specifying a generator explicitly, `kubectl` will infer that you want to create a Pod resource. This is because creating a Pod directly is a common use case for `kubectl run` when no other resource type is specified.

Similarly, if you want to create a Deployment, you can specify it explicitly:

```bash
kubectl run my-deployment --image=my-image --generator=deployment/apps.v1
```

Here, `--generator=deployment/apps.v1` explicitly tells `kubectl` to generate a Deployment resource.

- By inferring the resource type when not explicitly specified, `kubectl` offers flexibility and convenience to users, making it easier to perform common operations without needing to remember or specify detailed generator configurations.


#### K8s API groups

- Kubernetes API groups are essentially collections of resources that are logically related.
- Kubernetes API groups are essentially collections of resources that are logically related. These groups help organize the API by function, improving discoverability and scalability of resources. The API groups are represented in the API URL as `/apis/<group>/<version>`. There are two main types of API groups:

 1. Core Group: Also known as the legacy group, it is accessed via /api/v1 and includes fundamental resources like Pods, Services, and Namespaces.

 2. Named Groups: These cover specific areas not included in the core group, such as apps for application-related resources, batch for batch processing tasks, and extensions for additional features.

#### API Versions and Their Significance

`Kubernetes API versions` signify the stability and support level of features, using a prefix `(alpha, beta, and stable)` followed by a version number. This nomenclature is critical for understanding the lifecycle of Kubernetes features:

- **Alpha (v1alphaX)**: Experimental features that are under active development and may change without notice. They are disabled by default and not recommended for production use.

- **Beta (v1betaX)**: More stable than alpha, beta features are enabled by default and are on their path to becoming stable. They are generally well-tested but might receive further modifications.

- **Stable (vX)**: These versions are deemed stable and safe for production use. Backward compatibility is maintained across releases, ensuring long-term support.

## 2. API groups and version negotiation:

- In Kubernetes, the API is versioned to allow for changes and improvements over time while maintaining backward compatibility. The API is organized into "API groups," which categorize similar resources together, making them easier to manage and reason about. This modular approach also provides a more scalable and flexible alternative to a monolithic API.

- Each API group consists of one or more versions, with each version introducing changes, enhancements, or deprecations to the resources within that group. By organizing resources into API groups and versions, Kubernetes can evolve and introduce new features without disrupting existing workflows or compatibility.

For example:

  - The `apps` API group includes resources related to higher-level abstractions such as `Deployments`, `ReplicaSets`, and `StatefulSets`.
  - The most recent version of the `apps` API group is `v1`.

- When creating or interacting with resources belonging to a specific API group and version, it's essential to specify the `apiVersion` field appropriately in the Kubernetes manifest files. For example, to create a Deployment, you would typically specify `apiVersion: apps/v1` at the top of the Deployment manifest to indicate that you're using the `apps` API group and the `v1` version of that group.

#### This ensures that Kubernetes knows which API group and version to use when processing the manifest and allows for proper version negotiation and compatibility checks during API interactions.

- After `kubectl` generates the runtime object (such as a Pod, Deployment, or Service), it proceeds to find the appropriate API group and version for that object. This process is known as version negotiation.

- During version negotiation, `kubectl` scans the `/apis` path on the remote Kubernetes API server to retrieve all possible API groups and their respective versions. The Kubernetes API server exposes its schema document, typically in OpenAPI format, at this path. This `schema document` contains information about the available API groups, versions, resources, endpoints, and supported operations.

- To improve performance, kubectl also `caches` the OpenAPI schema to the ~/.kube/cache/discovery directory.

- By examining the schema document, `kubectl` can determine which API group and version correspond to the runtime object it generated. This allows `kubectl` to assemble a versioned client that is aware of the various REST semantics (such as CRUD operations, GET requests) for interacting with the specific resource.

- Performing version negotiation enables `kubectl` to dynamically adapt to the capabilities and configurations of the Kubernetes cluster it's communicating with. This flexibility allows `kubectl` to support a wide range of Kubernetes versions and configurations, ensuring compatibility and consistency in resource management operations.

## 3. Client auth

Client authentication is a critical step that `kubectl` performs before sending HTTP requests to the Kubernetes API server. This authentication process ensures that `kubectl` has the necessary credentials to access and interact with the Kubernetes cluster securely.


Here's an overview of how `kubectl` handles client authentication:

1. **Locating kubeconfig File**: `kubectl` searches for the kubeconfig file, which typically contains user `credentials` and `cluster configuration` settings. It follows these steps to locate the kubeconfig file:
   - If a `--kubeconfig` flag is provided explicitly, `kubectl` uses the file specified by that flag.
   - If the `$KUBECONFIG` environment variable is defined, `kubectl` uses the file or files specified by that variable.
   - If neither of the above is provided, `kubectl` looks in the recommended home directory (such as `~/.kube`) and uses the first file found.

2. **Parsing kubeconfig File**: Once `kubectl` has located the kubeconfig file, it parses the file to extract information such as:
   - Current context: Specifies the cluster, user, and namespace to use for the current `kubectl` session.
   - Cluster configuration: Details about the Kubernetes cluster, including server URL and certificate authority data.
   - User credentials: Authentication information associated with the current user, such as bearer tokens, x509 certificates, or username/password combinations.

3. **Flag Overrides**: If the user provides flag-specific authentication values (such as `--username`), these values take precedence over those specified in the kubeconfig file.

4. **Configuring HTTP Request**: Based on the authentication information obtained from the kubeconfig file or user-provided flags, `kubectl` populates the client's configuration to ensure that HTTP requests are decorated appropriately:
   - x509 certificates: `kubectl` configures TLS (Transport Layer Security) using the provided certificates, including the root CA (Certificate Authority).
   - Bearer tokens: If a bearer token is available, `kubectl` includes it in the "Authorization" HTTP header of the request.
   - Username/password: For HTTP basic authentication, `kubectl` sends the username and password in the request headers.
   - OpenID Connect: If the user has performed the OpenID authentication process manually and obtained a token, `kubectl` sends this token like a bearer token.

By handling client authentication in this manner, `kubectl` ensures that it can securely authenticate with the Kubernetes API server using the appropriate credentials and authentication mechanisms. This enables users to interact with Kubernetes clusters confidently and securely using `kubectl` commands.

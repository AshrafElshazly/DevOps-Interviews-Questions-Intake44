# Kubernetes Notes

### **Definition**: Kubernetes is an open-source system for automating the deployment, scaling, and management of containerized applications. Kubernetes places containers into pods and runs them on nodes. A Kubernetes cluster consists of at least one control plane (which manages the cluster) and one or more worker nodes (which run the container pods). When you deploy Kubernetes, you are essentially setting up and running a Kubernetes cluster.

## K8S Architecture

### Master Node (Control Plane)

- **(Kube API Server - Controller Manager - ETCD - Schaduler - Cloud Controller Manager)**

  - **Kube API Server:** It serves as the central management entity and provides the primary interface for interacting with the Kubernetes cluster. Essentially, it acts as the "front door" to the Kubernetes control plane, handling all RESTful API requests from clients, such as `kubectl`, as well as from internal cluster components.

    - **Authentication and Authorization:** It verifies the identity of users and service accounts making requests (authentication) and checks whether they have the necessary permissions to perform the requested actions (authorization).

    - **API Gateway:** The kube-apiserver exposes the Kubernetes API, which allows users and components to manage Kubernetes objects like Pods, Services, and Deployments.

      - **Internal Communication:** Components like the kube-scheduler and kube-controller-manager communicate with the API server to retrieve and update the state of the cluster.

      - **External Communication:** Users interact with the Kubernetes cluster using `kubectl` (or other clients) which sends requests to the API server.

    - **State Management:** The kube-apiserver interacts with etcd, the key-value store used by Kubernetes, to persist the state of the cluster. It ensures that the desired state specified by users is stored and retrieved accurately.

    - **Coordination:** It coordinates with other control plane components, such as the scheduler and controller-manager, to manage the lifecycle and state of all resources within the cluster.

  - **Controller Manager:** Controllers are background processes that manage and regulate the state of Kubernetes cluster objects, ensuring the actual state of the cluster matches the desired state specified in the cluster's configuration. is essential for automating the operations of a Kubernetes cluster. It ensures that the cluster's desired state is continuously maintained, providing resilience, scalability, and security. Without the Controller Manager, many administrative tasks would need to be performed manually, leading to inefficiencies and potential errors.

    - **Node Controller**

      - Node Registration: Adds new nodes to the cluster.
      - Health Monitoring: Checks the health of nodes through heartbeats and marks unresponsive nodes.
      - Node Recovery: Evicts pods from unhealthy nodes and reschedules them on healthy nodes.
      - Resource Management: Updates the node's status with resource information for accurate scheduling decisions.

    - **Replication Controller**

      - Pod Creation: Ensures the desired number of pod replicas are running by creating new pods as needed.
      - Pod Deletion: Deletes excess pods to maintain the specified replica count.
      - Monitoring: Continuously monitors the cluster to maintain the desired number of replicas.
      - Self-Healing: Automatically replaces failed pods to ensure high availability.

    - **Endpoints Controller**

      - Service Discovery: Links services to the pods that provide those services by updating the endpoints resource.
      - Dynamic Updates: Keeps endpoint information up-to-date as pods are created, deleted, or moved.
      - Health Checks: Ensures only healthy pods are included in service endpoints for reliability.

    - **Service Account & Token Controllers**

      - Service Account Creation: Creates default service accounts in new namespaces.
      - Token Management: Manages tokens used by pods to authenticate against the Kubernetes API server.
      - Access Control: Associates service accounts with specific roles and permissions.
      - Automatic Injection: Ensures each pod has the necessary credentials by injecting service account tokens into pod definitions.

  - **ETCD:** etcd is a distributed, consistent key-value store used as the primary backing store for all cluster data in Kubernetes. It is a critical component of the Kubernetes control plane, providing a reliable way to store configuration data, state information, and metadata. etcd is a critical component of the Kubernetes control plane infrastructure. It provides a reliable and consistent storage mechanism for all cluster data, ensuring that the cluster state is maintained even in the event of node failures or network partitions. Without etcd, Kubernetes would not be able to function properly, as it relies on etcd for storing crucial information about the cluster's configuration, state, and metadata.

  - **Schaduler:** is a core component of the Kubernetes control plane responsible for assigning pods to nodes in the cluster based on resource availability, quality of service requirements, and other constraints. It plays a crucial role in ensuring that workloads are efficiently distributed across the cluster to achieve optimal resource utilization and performance. The Scheduler plays a critical role in the efficient operation of a Kubernetes cluster by ensuring that workloads are distributed effectively across available resources. Without the Scheduler, Kubernetes would not be able to automatically manage workload placement, leading to inefficiencies and manual intervention requirements.

    - **Node Selection:**

      - The Scheduler evaluates various factors, including resource requirements, node capacity, and affinity/anti-affinity rules, to determine the most suitable node for placing a pod.
      - It considers factors such as CPU and memory availability, node health, and workload constraints when making placement decisions.

    - **Pod Scheduling:**

      - When a new pod is created or an existing pod needs to be rescheduled (due to node failures, scaling, etc.), the Scheduler is responsible for selecting an appropriate node for the pod to run on.
      - It ensures that pods are placed on nodes that meet their resource requirements and adhere to any specified affinity/anti-affinity rules.

    - **Resource Management:**

      - The Scheduler helps in optimizing resource utilization by evenly distributing pods across the cluster and avoiding resource contention.
      - It takes into account resource requests and limits specified in pod definitions to prevent overloading nodes or causing performance degradation.

    - **Policy Enforcement:**

      - The Scheduler enforces various policies and constraints defined by administrators or users, such as node selectors, node taints, and pod affinity/anti-affinity rules.
      - It ensures that pods are scheduled in accordance with these policies to meet application requirements and ensure proper isolation and performance.

  - **Cloud Controller Manager:** is a Kubernetes control plane component responsible for interacting with the underlying cloud provider's APIs to manage cloud-specific resources and functionalities. It abstracts away the cloud provider's details from the core Kubernetes components, allowing Kubernetes to be more cloud-agnostic.

### Worker Node

- **(Kubelet - KubeProxy - Container Runtime)**

  - **Kubelet:** is the primary node agent that runs on each node, including the master node, if necessary. It ensures that containers are running in pods. The kubelet is an essential component running on each worker node in a Kubernetes cluster. It's responsible for managing and maintaining individual nodes and the pods running on them, ensuring they adhere to the specifications provided in pod manifests. The kubelet plays a critical role in the operation of Kubernetes worker nodes, ensuring that pods are properly managed and executed according to their specifications. It acts as the bridge between the Kubernetes control plane and the node's runtime environment, executing instructions from the control plane to maintain the desired state of the cluster.

    - **Pod Lifecycle Management:**

      - Kubelet ensures that pods are running and healthy on the node. It manages the lifecycle of pods, including creation, starting, stopping, and restarting as necessary.
      - It interacts with the container runtime (e.g., Docker, containerd) to perform these actions.

    - **Container Runtime Integration:**

      - Kubelet communicates with the container runtime to execute container-related operations, such as pulling container images, starting and stopping containers, and collecting container logs.
      - It supports various container runtimes, allowing Kubernetes clusters to be deployed on different infrastructure platforms.

    - **Resource Management:**

      - Kubelet monitors the node's resource utilization (CPU, memory, disk) and ensures that pods are scheduled based on available resources and configured resource requests/limits.
      - It enforces resource constraints specified in pod manifests to prevent resource contention and ensure fair resource allocation.

    - **Network Setup:**

      - Kubelet configures networking for pods running on the node, ensuring they can communicate with each other and with services within the cluster.
      - It sets up network namespaces, IP addresses, and routes for pods using the container runtime's networking capabilities.

    - **Volume Management:**

      - Kubelet manages volumes attached to pods, ensuring that volume mounts specified in pod manifests are correctly configured and mounted into containers.
      - It supports various types of volumes, including emptyDir, hostPath, persistentVolumeClaim, and cloud provider-specific volumes.

    - **Node Status Reporting:**

      - Kubelet regularly reports the node's status (e.g., capacity, conditions) to the Kubernetes control plane, allowing the scheduler to make informed decisions when scheduling pods.
      - It detects and reports node-level issues or failures to the control plane for appropriate handling and remediation.

  - **Kube-Proxy:** is an essential component of Kubernetes, responsible for network routing and service discovery within the cluster. By managing network rules, kube-proxy ensures that services can communicate with each other and that external traffic is correctly routed to the appropriate pods. Its ability to handle service load balancing, cluster IPs, and integration with cloud provider networking features makes it a critical part of the Kubernetes networking stack. The different modes of operation (iptables and IPVS) offer flexibility and performance options to meet the needs of diverse cluster environments.

  - **Container Runtime** is essential for executing and managing containers on each node. It handles the entire lifecycle of containers, including pulling images, starting and stopping containers, and managing their resources and security. By abstracting these tasks through the Container Runtime Interface (CRI), Kubernetes ensures flexibility and compatibility with various container runtimes, enhancing the robustness and scalability of containerized applications in the cluster.

## K8S Networking

- **CNI Plugins:** Directly handle the network setup for individual pods, ensuring each pod has a unique IP and can communicate within the cluster.
- **Kubelet:** Coordinates with the CNI plugins to apply network configurations when pods are created on a node.
- **Kube-proxy:** Manages the higher-level networking logic for services, ensuring that network traffic is correctly routed to the appropriate pods and providing load balancing.

## K8S Security

- (Service Account - RBAC(Role - ClusterRole - RoleBinding - ClusterRoleBinding) - KubeConfig(Cluster - User - Context) - NetworkPolicy - Pod Security Policies (PSP))

- **Service Account:** A service account in Kubernetes provides an identity for processes that run in a Pod. It is used to authenticate and authorize these processes when they interact with the Kubernetes API server or other cluster resources.

- **RBAC (Role-Based Access Control):** RBAC is a method of regulating access to computer or network resources based on the roles of individual users within an enterprise. In Kubernetes, RBAC is used to define roles (sets of permissions) and bind them to users or service accounts.

- **Roles and ClusterRoles:** Roles and ClusterRoles define sets of permissions within a namespace or across the entire cluster, respectively. They specify what actions (verbs) can be performed on which resources (nouns).

- **RoleBinding and ClusterRoleBinding:** RoleBindings and ClusterRoleBindings associate roles or cluster roles with users, groups, or service accounts. They define who has access to which resources based on the roles assigned.

- **KubeConfig:** KubeConfig is a file used to configure access to Kubernetes clusters. It contains information such as cluster details, authentication details (users), and context (combination of cluster, user, and namespace) to use when interacting with the cluster.

  - **Cluster, User, and Context in KubeConfig:**

    - **Cluster:** Defines the Kubernetes cluster's endpoint and other details necessary to connect to it.
    - **User:** Defines the authentication details, such as tokens, certificates, or username/password combinations, used to access the cluster.
    - **Context:** Combines a cluster, a user, and a namespace to define a specific working environment within the KubeConfig file.

- **NetworkPolicy:** NetworkPolicy is a Kubernetes resource that specifies how groups of pods are allowed to communicate with each other and other network endpoints.

- **Pod Security Policies (PSP):** Pod Security Policies are a set of conditions and constraints applied to pods to control their security settings. PSPs define what security measures pods must comply with, such as requiring certain capabilities, disallowing privileged access, or defining security context constraints.

## Interviews Questions

- What is the architecture of Kubernetes?
- What are the types of services in Kubernetes?
- Explain deployments in Kubernetes.
- What is node affinity in Kubernetes?
- What is a daemon set in Kubernetes?
- What is the container runtime in Kubernetes?
- How to make a Kubernetes cluster highly available?
- If a pod doesn’t delete after running `kubectl delete`, how would you troubleshoot?
- Can you create a load balancer locally on your machine?
- Explain the steps when a deployed application in K8s is accessed via a domain in the browser.
- What is the type of Kubelet, and how does it get instantiated?
- operator vs controller vs custom resources
- What are startup, liveness probes and readiness probes?
- What happens if kubelet and kubeproxy are not available?
- Can a Persistent Volume (PV) connect to multiple Persistent Volume Claims (PVC)? Is the relationship one-to-one?

## K8S Troubleshooting

#### Kubernetes Cluster Issues:

1. **What are some common signs that a Kubernetes cluster is experiencing performance issues or resource constraints? How would you diagnose and address these problems?**

   - Common signs include high CPU or memory utilization, slow response times, and Pod evictions. To diagnose, use tools like `kubectl top` and review cluster metrics. Address issues by optimizing resource requests and limits, and scaling resources if necessary.

2. **Explain how you would troubleshoot and recover from a Kubernetes cluster that is in a "NotReady" state. What steps would you take to bring it back to a healthy state?**

   - Troubleshoot by checking cluster nodes, inspecting system components like `kubelet` and `kube-proxy`, and examining Pod statuses. Restarting or recovering failed components and addressing underlying issues can help bring the cluster back to a healthy state.

#### Pod and Deployment Issues:

3. **A Pod in your Kubernetes cluster is stuck in the "Pending" state. What could be the possible reasons for this, and how would you troubleshoot and resolve it?**

   - Possible reasons include resource constraints, node affinity/anti-affinity rules, or insufficient resources. Troubleshoot by checking resource requests/limits, node availability, and event logs. Resolve by adjusting resources or node assignments.

4. **How can you troubleshoot a Kubernetes Deployment that continuously fails to roll out a new version of an application? What tools and commands would you use?**

   - Troubleshoot by checking Deployment events, inspecting Pods, and using commands like `kubectl describe deployment` and `kubectl rollout history`. Examine logs and revert to a stable version if needed.

#### Networking Issues:

5. **What steps would you take to diagnose and resolve network connectivity issues between Pods in a Kubernetes cluster? How would you ensure that Pods can communicate with each other?**

   - Diagnose by inspecting Pod networking, service definitions, and network policies. Use tools like `kubectl exec` and `nslookup` to test connectivity. Ensure proper network policies and Service configurations for inter-Pod communication.

6. **Explain the concept of "Cluster DNS" in Kubernetes and how it facilitates DNS-based service discovery. What would you do if DNS resolution between Pods is not working correctly?**

   - Cluster DNS provides DNS-based service discovery within a Kubernetes cluster. If DNS resolution is not working, troubleshoot by checking CoreDNS logs, Pod DNS configurations, and network policies. Ensure DNS policies and configurations are correct.

#### Storage Issues:

7. **A PersistentVolumeClaim (PVC) is not binding to a PersistentVolume (PV) as expected. How would you troubleshoot and resolve this issue?**

   - Troubleshoot by examining PV and PVC statuses, storage class configurations, and access modes. Check for PV availability and matching labels/annotations. Resolve by ensuring a suitable PV is available and the PVC matches the requirements.

8. **What steps would you take to recover data from a Pod that has lost access to its PersistentVolume? Describe a scenario where data recovery might be necessary.**

   - Data recovery may be necessary if a Pod's PV becomes inaccessible due to node failure or storage issues. To recover, ensure the PV is reattached to a new node, if possible, or use backup and restore methods if data loss occurs.

#### Logging and Monitoring:

9. **Explain how you can use Kubernetes logs and metrics for troubleshooting. What tools and strategies would you employ to collect and analyze logs and metrics effectively?**

   - Use tools like `kubectl logs`, `kubectl top`, and cluster-level logging solutions like Fluentd or Loki. Employ log aggregation and visualization tools like Elasticsearch, Grafana, or Kibana for efficient log analysis.

#### Security Issues:

10. **What are some common security-related issues that can affect a Kubernetes cluster, and how would you go about securing the cluster and resolving these issues?**

    - Common issues include exposed dashboards, misconfigured RBAC, and vulnerable container images. Secure the cluster by configuring RBAC, using network policies, and performing security scans on container images. Resolve issues by fixing misconfigurations and applying security best practices.

#### Resource and Resource Quota Issues:

11. **A Kubernetes namespace is hitting its resource quota limits, causing Pods to fail to schedule. How would you identify the resource-consuming Pods and resolve this issue?**

    - To identify resource-consuming Pods, use tools like `kubectl top` and inspect resource requests/limits in Pod manifests. To resolve the issue, consider adjusting resource quotas, optimizing resource requests/limits, or scaling the cluster.

12. **Explain the role of Kubernetes resource quotas and how they help in preventing resource overcommitment. What happens when a Pod exceeds its resource limits?**

    - Resource quotas define limits on resource consumption within namespaces to prevent overcommitment. When a Pod exceeds its resource limits, it may be terminated or throttled, depending on the QoS class. It's crucial to define appropriate resource limits for Pods.

#### Node and Cluster Scaling Issues:

13. **You notice that nodes in your Kubernetes cluster are experiencing high CPU and memory utilization. How would you determine which Pods or applications are causing these resource constraints?**

    - Determine the resource-consuming Pods by using `kubectl top`, reviewing Pod metrics, and checking node logs. Address the issue by optimizing resource requests/limits, adding nodes, or scaling applications horizontally.

14. **Explain the process of scaling a Kubernetes cluster to accommodate increased workloads. What considerations and best practices should be followed when adding new nodes to the cluster?**

    - Scaling a cluster involves adding new nodes to accommodate increased workloads. Considerations include hardware requirements, network configuration, and maintaining high availability. Follow best practices such as using cloud auto-scaling groups or cluster node pools.

#### Pod Eviction and Disruption:

15. **What are the reasons why Pods may be evicted from nodes in a Kubernetes cluster? How can you troubleshoot and prevent excessive Pod evictions?**

    - Reasons for Pod eviction include resource constraints, node failures, and maintenance tasks. Troubleshoot by examining Pod events, resource requests/limits, and node conditions. Prevent excessive evictions by ensuring resource planning and node stability.

#### API Server and Control Plane Issues:

16. **What are the common issues that can affect the Kubernetes API server and control plane components? How would you diagnose and resolve these issues?**

    - Common issues include API server crashes, certificate expirations, and etcd database problems. Diagnose by reviewing API server logs, certificates, and etcd status. Resolve by restarting affected components, renewing certificates, or restoring etcd from backups.

#### Networking and DNS Issues:

17. **Explain how you would troubleshoot a Kubernetes DNS resolution issue that prevents Pods from resolving domain names. What could be the potential causes, and how would you address them?**

    - Troubleshoot DNS resolution by checking CoreDNS logs, Pod DNS configurations, and network policies. Causes may include misconfigurations, DNS policy conflicts, or CoreDNS issues. Address by resolving misconfigurations and ensuring DNS policies are correct.

#### Kubelet and Node-Level Problems:

18. **A node in your Kubernetes cluster is unresponsive, and Pods scheduled on it are failing. How would you diagnose the node's health and recover it?**

    - Diagnose node health by checking system logs, `kubelet` status, and system resource utilization. Restart the `kubelet` service and address node issues like resource constraints or OS-level problems. Evacuate Pods if necessary.

#### Security and Access Control Issues:

19. **What are some security-related issues that might impact a Kubernetes cluster, such as vulnerabilities in container images or misconfigured RBAC policies? How can you identify and mitigate these issues?**

    - Security issues may include vulnerable container images, exposed dashboards, or overly permissive RBAC. Identify by scanning container images, reviewing RBAC configurations, and conducting security audits. Mitigate by updating images, securing dashboards, and refining RBAC policies.

20. **A Pod is unable to access a Kubernetes API or a service within the cluster due to RBAC restrictions. How would you troubleshoot and resolve this access control issue?**

    - Troubleshoot RBAC issues by examining Pod service accounts, role bindings, and cluster roles. Ensure that appropriate permissions are granted to the Pod's service account, and consider creating custom RBAC roles and bindings if

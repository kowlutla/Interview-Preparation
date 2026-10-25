# Docker, Kubernetes & Helm Interview Handbook
**Audience:** Java backend developers with 4–6 years of experience targeting Senior Software Engineer and Backend Engineer roles.  
**Focus:** Application packaging, runtime behavior, deployment configuration, networking, release safety, and production troubleshooting—not cluster administration.

> Examples are illustrative and should be validated against the Docker Engine, Kubernetes, Helm, JDK, and Spring Boot versions used by your team. Kubernetes examples use the `apps/v1` Deployment API and current probe/resource conventions.

## Contents
1. [How containers and Kubernetes fit together](#1-how-containers-and-kubernetes-fit-together)
2. [Docker for Java developers](#2-docker-for-java-developers)
3. [Java runtime behavior in containers](#3-java-runtime-behavior-in-containers)
4. [Kubernetes workload fundamentals](#4-kubernetes-workload-fundamentals)
5. [Spring Boot configuration, health, and shutdown](#5-spring-boot-configuration-health-and-shutdown)
6. [Networking and service discovery](#6-networking-and-service-discovery)
7. [Helm for application delivery](#7-helm-for-application-delivery)
8. [Security basics for developers](#8-security-basics-for-developers)
9. [A developer's release workflow](#9-a-developers-release-workflow)
10. [Troubleshooting scenarios](#10-troubleshooting-scenarios)
11. [Essential commands](#11-essential-commands)
12. [References](#12-references)

---

## 1. How containers and Kubernetes fit together

### Q1. Why containerize a Spring Boot application?
A container image packages the application artifact and the user-space runtime dependencies it needs into a versioned, portable unit. It makes the runtime more consistent between a developer laptop, test environment, and production. The host kernel is shared, so a container is not a full virtual machine; host architecture, kernel features, runtime configuration, secrets, and external dependencies can still cause environment-specific differences.

For a backend developer, the goal is not simply “make a Docker image.” It is to make startup predictable, configuration external, resource use bounded, logs observable, and shutdown safe. A container image should be immutable; environment-specific values should be supplied at runtime rather than baked into separate images for every environment.

### Q2. Containers versus virtual machines?
A VM virtualizes hardware and normally runs a guest operating system and its own kernel. Containers isolate processes using operating-system mechanisms and share the host kernel. Containers generally start faster and use fewer resources, but their isolation boundary and kernel compatibility differ from a VM. A container is not automatically secure just because it is isolated.

### Q3. Image, container, registry, Docker client, and daemon—how do they relate?
An **image** is a read-only template made of layers. A **container** is a created/running instance with a writable layer and runtime configuration. A **registry** stores and distributes images. The Docker CLI sends requests to the Docker Engine API; the daemon manages images, networks, volumes, and containers. At runtime, containerd and lower-level OCI runtimes commonly participate in creating and running the process. Exact implementation details vary by platform and Docker version.

Kubernetes does not require the Docker daemon on each node. It asks the node's container runtime through the CRI interface to pull images and run containers.

### Q4. What happens when a Kubernetes Deployment is applied?
The API server stores the requested desired state. Controllers continuously compare actual state with desired state and reconcile differences. A Deployment manages ReplicaSets; a ReplicaSet maintains the requested number of matching Pods. A scheduler assigns unscheduled Pods to nodes, and each node's kubelet works with its runtime to start containers. This is a control loop, not a one-time script.

If a Pod is deleted or a node fails, the workload controller can create replacement Pods. Therefore, an individual Pod name or IP should not be treated as permanent identity for a normal stateless REST service.

### Q5. What is the developer/platform-team boundary?
Developers should understand workload manifests, probes, resources, configuration, logs, rollout status, service discovery, and how to gather evidence. Platform teams typically own cluster provisioning, node lifecycle, control-plane operation, cluster-wide networking, policy frameworks, and shared ingress infrastructure. A good incident handoff includes namespace, deployment/pod names, timestamps, recent changes, relevant events, logs, and the exact failure—not just “Kubernetes is broken.”

---

## 2. Docker for Java developers

### Q6. What does a basic Dockerfile for Spring Boot look like?
For a prebuilt executable JAR:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/orders-api.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

This is a compact example, not a universal production baseline. Pin a reviewed image version (and, where appropriate, digest), use a supported JRE/JDK distribution, run as non-root, and ensure the application artifact is built for the target architecture and Java version. `EXPOSE` documents the intended container port; it does not publish the port to the host.

### Q7. Explain common Dockerfile instructions.
- `FROM` selects the base image.
- `WORKDIR` sets the working directory for subsequent instructions.
- `COPY` copies files from the build context into the image.
- `RUN` executes a command while building an image, often installing tools or compiling code.
- `ARG` defines a build-time variable; it is not automatically a runtime environment variable.
- `ENV` sets environment variables that persist into the image/runtime unless overridden.
- `USER` selects the user for subsequent build instructions and the default runtime user.
- `ENTRYPOINT` defines the main executable; `CMD` supplies default command/arguments depending on how they are combined.
- `EXPOSE` documents a port but does not create a host mapping.

Avoid copying secrets into any image layer. Removing a secret in a later layer does not reliably remove it from the image history.

### Q8. ENTRYPOINT versus CMD?
With exec-form syntax, `ENTRYPOINT` identifies the executable and `CMD` commonly provides its default arguments:

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--server.port=8080"]
```

Running `docker run image --server.port=9090` replaces the default CMD arguments and passes the new argument to the entrypoint. If the image has no entrypoint, CMD can specify the command itself. Shell-form instructions introduce a shell process and can complicate signal delivery; prefer exec form for application processes unless shell behavior is deliberately needed.

### Q9. Why use a multi-stage build?
A build stage can use a full JDK and Maven/Gradle, while the final stage contains only the runtime and application artifact. This reduces the final image size and attack surface, and avoids shipping build tools unnecessarily.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/orders-api.jar app.jar
USER 10001
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The exact Maven image tag should be chosen and pinned to an approved version. In real CI, run tests before packaging; `-DskipTests` above only illustrates the build boundary and should not replace the test stage. Gradle follows the same pattern using a Gradle/JDK builder image and copying the produced JAR.

### Q10. How do image layers and build caching work?
Each Dockerfile instruction can create a layer. If an instruction and its inputs have not changed, the builder may reuse the cached result. Copying all source code before downloading dependencies invalidates the dependency layer on every source edit. Copy dependency descriptors first, resolve dependencies, and then copy source code. Keep the build context small with a `.dockerignore` file:

```text
.git
target/
build/
.idea/
*.log
.env
```

Do not exclude files required by the build, and never include credentials or local secrets in the context.

### Q11. Should I use a JDK or JRE image?
Use a JDK when compiling or running tools that require compiler modules. A runtime-only image is usually enough to run an executable Spring Boot JAR. Select a maintained distribution compatible with your licensing, patching, architecture, and support requirements. “Smallest image” is not the only criterion; patch availability, debugging support, certificate stores, native libraries, and operational familiarity matter too.

### Q12. How should configuration be passed to a container?
Keep environment-specific configuration outside the image. For Spring Boot, environment variables such as `SPRING_PROFILES_ACTIVE`, `SERVER_PORT`, and `SPRING_DATASOURCE_URL` can override configuration properties using Spring Boot's relaxed binding rules. Credentials should come from a secret-management mechanism, not source code or a Dockerfile. Avoid printing the full environment because it may expose credentials.

### Q13. How does port publishing work?
The application must listen on the container's network interface and port. `EXPOSE 8080` is metadata. Locally, `docker run -p 8080:8080 image` maps host port 8080 to container port 8080. A process bound only to `127.0.0.1` inside the container may not be reachable through the container's external interface; a web service normally binds to `0.0.0.0` or the appropriate interface.

### Q14. How do containers connect to a database or another service?
Inside a container, `localhost` means that same container's network namespace. It does not mean the host machine, a sibling container, or a database container. In Docker Compose, services on the same Compose network can usually reach each other using the service name and container port, for example `jdbc:postgresql://db:5432/orders`. In Kubernetes, use a Service DNS name. Host access from Docker Desktop differs from Linux Docker Engine; use the documented host gateway mechanism for your environment rather than assuming one hostname works everywhere.

### Q15. What are volumes and why shouldn't application state live only in the container?
A container's writable layer is ephemeral: replacing the container can discard its changes. Volumes or mounted storage are useful for data that must survive container replacement, such as local development database files. For typical stateless APIs, durable business data belongs in a database or object store, not in local container files. Do not use a shared filesystem as a substitute for deliberate state ownership and concurrency design.

### Q16. When is Docker Compose useful?
Compose describes a local multi-container environment—API, database, cache, or message broker—in one YAML file. It improves repeatability for development and integration tests. It is not the same thing as a production Kubernetes deployment, and Compose success does not prove production networking, secrets, resource limits, or rollout behavior will be correct.

### Q17. How do you debug a container that exits?
Start with `docker ps -a`, then `docker logs <container>` and `docker inspect <container>`. Inspect the exit code, entrypoint/command, mounted files, environment, working directory, and permissions. Common causes include a missing JAR, wrong Java version, invalid arguments, a configuration exception, or a process that finishes normally instead of remaining in the foreground. An application container should run the application as its main process, not start it in the background and exit.

### Q18. Why are immutable image tags and digests important?
A mutable tag such as `latest` can point to different content over time, making rollbacks and incident reproduction ambiguous. Prefer unique build tags and record the image digest. A digest identifies exact image content. In promotion pipelines, promote the same built artifact across environments rather than rebuilding slightly different images for staging and production.

### Q19. How should an application handle shutdown signals?
The container runtime sends a termination signal—commonly SIGTERM—when stopping a container, then may force-kill it after a grace period. The Java process should remain PID 1 or be managed by an init process that forwards signals correctly. Spring Boot supports graceful shutdown when configured and supported by the application/server version. Give the application enough time to stop accepting new work and complete in-flight requests; test this rather than assuming it works.

### Q20. What is a good Docker troubleshooting checklist?
Check build context and ignore rules; image architecture and Java version; command/entrypoint; user and file permissions; listening address and port; runtime environment variables; mounted paths; logs and exit code; network/DNS reachability; and memory limits. Reproduce with the exact image and configuration used in the target environment whenever possible.

---

## 3. Java runtime behavior in containers

### Q21. Why can a Java container be OOMKilled when the heap limit looks reasonable?
The container memory limit applies to total memory charged to the container, not just Java heap. A JVM also uses metaspace, code cache, thread stacks, direct buffers, native libraries, GC structures, and other native allocations. If the heap is configured near the entire container limit, total usage can exceed the cgroup limit and the kernel may kill the process. Kubernetes often reports the termination reason as `OOMKilled`.

Leave headroom and size heap against the actual memory budget. Modern supported JVMs are container-aware, but verify the JDK version and flags. Avoid copying a fixed `-Xmx` from a VM to a much smaller container without accounting for non-heap memory.

### Q22. How should heap sizing be approached?
Use the container memory limit as a budget, not as the heap size. For example, a 1 GiB memory limit does not mean `-Xmx1g` is safe. A percentage-based setting such as `-XX:MaxRAMPercentage` may be more portable across different limits, but its value must leave room for native memory and be validated under load. Observe actual RSS/cgroup usage, GC behavior, allocation rate, and peak concurrency. Exact defaults vary by JDK and runtime configuration.

### Q23. How does CPU throttling affect API latency?
A CPU limit can cause the container to be throttled when it consumes its allotted CPU time. The process may still be alive but respond slowly, causing request timeouts, probe failures, queue growth, and retries. Check CPU usage, throttling metrics, request latency, and thread-pool queues together. Raising the limit may help, but also investigate excessive CPU work, inefficient serialization, lock contention, and retry storms. CPU requests influence scheduling and are also used in some autoscaling calculations; they are not identical to CPU limits.

### Q24. How do thread pools and connection pools interact with resource limits?
A larger request thread pool is not automatically higher throughput. More concurrent requests can increase heap use, thread-stack memory, database connections, downstream load, and context switching. Size the HTTP executor, async executors, DB pool, and downstream client pools as a system. Use load tests and queue/latency measurements. A pool sized independently in every replica can overwhelm a shared database after horizontal scaling.

### Q25. What is the difference between a process being alive and an application being healthy?
A process can exist while its server is still starting, its request executor is saturated, its DB pool is exhausted, or required configuration is invalid. Conversely, a temporary dependency failure does not always mean the process should be restarted. Use startup probes for slow initialization, readiness to decide whether the Pod should receive new traffic, and liveness to detect a process that is irrecoverably stuck. Probe semantics must match the application's recovery model.

---

## 4. Kubernetes workload fundamentals

### Q26. Explain Pod, Deployment, ReplicaSet, and Service.
A **Pod** is the smallest scheduled unit and can contain one or more tightly coupled containers sharing networking and optionally volumes. A **Deployment** manages desired rollout state for a stateless workload. It creates and manages **ReplicaSets**, which maintain the number of Pod replicas for a particular template. A **Service** provides a stable virtual endpoint and selects matching Pods, so clients do not need to track changing Pod IPs.

For a normal Spring Boot microservice, developers usually modify the Deployment template, container configuration, Service, and ConfigMap/Secret references—not individual Pods.

### Q27. What is desired state and reconciliation?
You declare the state you want, such as three replicas with a particular image and resource limits. Kubernetes controllers keep working to move observed state toward that declaration. If one Pod fails, a controller can create another. This is why manual edits to a generated Pod are temporary and why configuration should be changed in the declarative source (often Helm values or manifests).

### Q28. What is a Namespace?
A Namespace groups namespaced Kubernetes objects and is commonly used to separate environments or teams. It is not, by itself, a strong security boundary. Resource quotas, RBAC, network policies, and admission controls may also be required. Always specify the intended namespace when inspecting resources to avoid debugging the wrong environment.

### Q29. What do resource requests and limits mean?
A request is the amount Kubernetes uses when making scheduling decisions; it is not a reservation of exclusive physical CPU. CPU limits constrain CPU consumption and can cause throttling. Memory limits are enforced differently: exceeding the container's memory cgroup limit can lead to an OOM kill. Set values based on measured startup and steady-state behavior, peak traffic, JVM overhead, and headroom—not arbitrary defaults.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "768Mi"
```

These values are examples only. A Java service with a large heap or heavy startup may need different values. A limit too low causes throttling or OOM; a request too high can make scheduling difficult and waste capacity.

### Q30. ConfigMap versus Secret?
A ConfigMap holds non-confidential configuration. A Secret is intended for sensitive values such as credentials and tokens, but base64 encoding is not encryption. Protect Secrets with appropriate RBAC, encryption-at-rest configuration, and a secure delivery process; many organizations use an external secret manager. Avoid exposing Secret values in logs, rendered manifests shared broadly, command history, or support tickets.

Environment variables are convenient but usually require a Pod restart to pick up changed values. A mounted configuration file can update on disk eventually, but the application must reload it to use the new value; environment variables in a running process do not change when the Secret object changes.

### Q31. What does a rolling update do?
A Deployment gradually replaces old Pods with new Pods according to its rollout strategy. `maxSurge` controls how many extra Pods may be created above the desired count, and `maxUnavailable` controls how many may be unavailable during the update. Readiness determines when new Pods are considered ready for Service traffic. Bad readiness settings, insufficient capacity, or an incompatible application change can still cause errors during a rollout.

A successful rollout only means Kubernetes considers the Deployment updated; it does not prove business workflows are correct. Verify representative API behavior, error rates, dependencies, and logs after deployment.

### Q32. How does rollback work and what are its limits?
A Deployment rollout can be rolled back to an earlier Pod template, and Helm can restore an earlier release revision. This does not automatically reverse database migrations, messages already published, emails sent, external API calls, or other side effects. Prefer backward-compatible schema changes and expand/contract migrations: add compatible schema first, deploy code that can work with both forms, migrate data, then remove obsolete schema in a later release.

### Q33. What does Horizontal Pod Autoscaler do?
HPA adjusts replica count based on observed metrics, commonly CPU or memory utilization and sometimes custom/external metrics. CPU utilization is often expressed relative to CPU requests, so incorrect requests can distort scaling. Scaling out helps only if the service is horizontally scalable and downstream dependencies can handle more load. Ensure state is externalized, connection pools are bounded per replica, and scale-up behavior accounts for startup time and warm-up. HPA does not fix a slow query or a saturated database by itself.

### Q34. Why is local Pod storage not durable?
Pods are replaceable, and a container writable layer is lost when the container is recreated. Pod-scoped ephemeral storage also has lifecycle limits. Use a database or object store for business data. PersistentVolumes and PersistentVolumeClaims can provide durable mounted storage where the workload truly needs files, but persistence brings lifecycle, performance, backup, and access-mode considerations. Most REST services should not rely on local disk for session state or uploaded files.

### Q35. What should developers know about cluster components?
Know the API server as the Kubernetes API endpoint, the scheduler as the component assigning Pods to nodes, controllers as desired-state reconcilers, and kubelet/runtime as the node-side mechanism that starts containers and reports status. That mental model helps explain Pending Pods, image-pull failures, and restarts. Deep control-plane operations and node administration are platform-team responsibilities for this handbook's scope.

### Q36. Example Deployment and Service for a Spring Boot API?
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      terminationGracePeriodSeconds: 40
      containers:
        - name: app
          image: registry.example.com/orders-api:1.4.2
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef:
                  name: orders-db
                  key: jdbc-url
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "768Mi"
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api
spec:
  selector:
    app: orders-api
  ports:
    - name: http
      port: 80
      targetPort: http
  type: ClusterIP
```

This is an illustrative template, not drop-in production YAML. The named health endpoints must actually be exposed and configured in Spring Boot Actuator, and health groups must reflect the intended semantics. Production manifests should also include an appropriate security context, image pull policy, and organization-specific metadata. Do not place the real database password in the manifest. The example uses a Secret key for the JDBC URL to keep the snippet compact; separate URL and credential keys are often clearer.

---

## 5. Spring Boot configuration, health, and shutdown

### Q37. How do profiles and externalized configuration work in Kubernetes?
Build one image and supply environment-specific values at deployment time. For example, `SPRING_PROFILES_ACTIVE=prod` selects a profile, while environment variables can override properties in `application.yml`. Avoid maintaining a different image per environment because it weakens promotion and reproducibility. Profiles should express configuration differences, not hide incompatible code paths that only get tested in one environment.

### Q38. How should readiness, liveness, and startup probes differ?
- **Startup probe:** protects slow-starting applications by allowing a longer startup window before liveness enforcement begins.
- **Readiness probe:** determines whether the Pod should receive Service traffic. A failed readiness probe removes it from ready endpoints; it does not itself restart the container.
- **Liveness probe:** detects a process that is alive but cannot recover without restart. Repeated failures cause a container restart.

A liveness endpoint should generally not fail just because a remote database or downstream service is temporarily unavailable. Otherwise a dependency outage can restart every replica and amplify the incident. Readiness may account for a dependency if the application truly cannot serve useful traffic without it, but avoid excessive dependency checks that overload the dependency. Probe endpoints should be fast and reliable.

### Q39. Why can bad probes cause an outage?
If liveness starts before the application has completed initialization, Kubernetes can repeatedly kill a healthy but slow-starting process. If readiness never becomes successful due to a wrong path, wrong port, authentication requirement, or endpoint not enabled, the Pod can remain running but receive no traffic. If liveness checks every downstream dependency, a brief outage may restart all Pods. Diagnose probe events and application startup logs before simply increasing thresholds.

### Q40. How should database connections be configured and pooled?
Use externalized URL, username, and password settings, configure connection and query timeouts, and size the pool according to workload and database capacity. Pool size must be considered across all replicas: 10 connections per replica multiplied by 20 replicas means up to 200 connections, before migrations, jobs, and other services are counted. A pool too small causes request waiting; a pool too large can overwhelm the database and increase contention. Measure acquisition latency, active/idle counts, waiters, query latency, and DB saturation.

### Q41. What is graceful shutdown for a REST service?
During termination, Kubernetes marks the Pod for deletion and runs the termination lifecycle. Endpoint removal and termination signaling are coordinated, but not instantaneous across every client and network path. The application should stop accepting new work, finish or safely cancel in-flight requests, close pools, and exit within the configured grace period. Configure Spring Boot graceful shutdown where appropriate, align server shutdown timeout with the Pod grace period, and ensure clients retry only when safe. Idempotency matters because a client may retry a request whose outcome is uncertain.

### Q42. How do multiple replicas affect state and caches?
Any request may reach any ready replica. In-memory sessions, locks, counters, or local caches are not automatically shared. A later request may land on a different Pod, and a Pod restart discards its memory. Prefer stateless APIs, external session storage when sessions are unavoidable, and cache designs with explicit consistency and invalidation rules. Sticky sessions can route around the issue but do not make local state durable and can complicate scaling and failover.

### Q43. How should logs and traces be handled?
Write application logs to stdout/stderr so the platform's log collector can capture them. Include timestamps, severity, service/version, and a request or trace correlation ID; avoid logging credentials, access tokens, or sensitive payloads. Use structured logs where supported. Correlation IDs must be propagated across service calls rather than generated independently at every hop. Logs from a replaced Pod may no longer be available through the current Pod, so central collection is important.

### Q44. What is the relationship between Actuator health and Kubernetes probes?
Spring Boot Actuator can expose health endpoints, but endpoint availability and health-group configuration depend on application dependencies and security settings. Ensure only intended endpoints are exposed, and make the probe path match the actual configured endpoint and port. Do not expose all management endpoints publicly just to make Kubernetes probes work. In some deployments, management endpoints use a separate port; then probe configuration must target that port deliberately.

---

## 6. Networking and service discovery

### Q45. How does one backend service call another in Kubernetes?
A caller uses the destination Service's DNS name and Service port, for example `http://inventory-service:8080` when both services are in the same namespace and the Service exposes port 8080. Across namespaces, use the fully qualified service DNS name when necessary, commonly `inventory-service.team-a.svc.cluster.local` (the cluster domain may differ). The Service gives clients a stable name while Pods change.

### Q46. Service port versus targetPort versus containerPort?
- `port` is the port exposed by the Service.
- `targetPort` is the destination port on selected Pods; it can be numeric or a named container port.
- `containerPort` documents a port in the Pod spec and can be referenced by name; it does not by itself publish the port or create a firewall rule.

A Service can expose port 80 and forward to a container listening on 8080. Ensure the application's actual listening port matches the target.

### Q47. ClusterIP, NodePort, LoadBalancer, Ingress, and Gateway API?
**ClusterIP** provides an internal virtual IP for cluster clients. **NodePort** exposes a port on nodes, subject to cluster networking. **LoadBalancer** asks the environment's integration to provision or configure an external load-balancing endpoint. **Ingress** is a widely used HTTP(S) routing API that requires an Ingress controller. **Gateway API** is a newer, more expressive family of APIs for traffic routing, but support depends on the installed implementation. Developers usually configure app ports and health behavior and coordinate public routing rules with the platform team.

### Q48. Why might a Pod be Running but the API is unavailable?
Running means the container process is running; it does not guarantee readiness or network reachability. Check readiness, Service selectors, ready EndpointSlices, port/targetPort, application bind address, probe events, NetworkPolicy, DNS, ingress routing, and application logs. Also verify that the request reaches the expected namespace/environment and that TLS or authentication configuration is correct.

### Q49. Why does localhost fail for service-to-service calls?
Each container has its own network namespace; within a Pod, containers share the Pod network namespace, but separate Pods do not. Calling `localhost:8080` from the orders service means the orders Pod itself, not the inventory service. Use the destination's Kubernetes Service DNS name. In a multi-container Pod, localhost can be correct when the containers are deliberately coupled and share the Pod network.

### Q50. How do you diagnose DNS or connection failures?
First establish the exact hostname, port, namespace, and protocol from the application configuration. Inspect logs for DNS resolution errors versus connection refusal versus timeout—these suggest different causes. Verify Service and EndpointSlices, then test from an approved debug environment if available. A connection refusal often means the address was reached but nothing accepted the port; a timeout may indicate dropped traffic, an unreachable endpoint, or an overloaded service. Do not assume every timeout is DNS.

### Q51. What role do NetworkPolicies play?
A NetworkPolicy can restrict which Pods may communicate, depending on the cluster's network implementation and configured policies. A correct Service name does not guarantee traffic is permitted. Developers should know the intended source/destination labels, namespaces, ports, and protocol and provide those details to the platform team. Detailed CNI and policy-engine administration is out of scope.

---

## 7. Helm for application delivery

### Q52. What is Helm?
Helm packages Kubernetes resources into a **chart** and tracks each deployment as a **release**. A chart contains metadata, default values, and templates; a release is an installed instance with its own values and revision history. Chart version and application image version are distinct: changing the chart can alter workload configuration even if the application image stays the same.

### Q53. What are the key files in a chart?
- `Chart.yaml`: chart metadata and chart version.
- `values.yaml`: default configurable values.
- `templates/`: Kubernetes manifest templates.
- `templates/_helpers.tpl`: reusable template helpers, commonly for names and labels.
- Optional environment values files: overrides for staging, production, or other environments.

Keep defaults safe and sensible. Environment-specific secrets should be supplied through approved secret references, not committed in values files.

### Q54. How do values become Kubernetes manifests?
Helm combines chart defaults with supplied override values, renders the templates, and produces Kubernetes YAML. For example, `image.tag`, `replicaCount`, `resources`, and `probes` values may be inserted into a Deployment. A typo in a values key may not always fail loudly if the template does not validate it, so inspect rendered manifests rather than assuming a successful command means the intended field changed.

```yaml
# values.yaml (illustrative)
replicaCount: 2
image:
  repository: registry.example.com/orders-api
  tag: "1.4.2"
service:
  port: 80
  targetPort: 8080
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 768Mi
```

### Q55. How do you inspect and validate a chart before deploying?
Run `helm lint ./chart` to catch common chart issues and `helm template orders ./chart -f values-prod.yaml` to inspect rendered resources. Review image reference, labels/selectors, ports, probes, Secret/ConfigMap references, resource settings, and rollout strategy. Template output can be valid YAML but still semantically wrong, so check the fields that affect your application.

### Q56. How do you install or upgrade a release?
Common patterns are:

```bash
helm install orders ./chart -n production --create-namespace -f values-prod.yaml
helm upgrade orders ./chart -n production -f values-prod.yaml
helm history orders -n production
helm rollback orders 3 -n production
```

Use the command forms appropriate to your Helm version and team workflow. A pipeline may use `helm upgrade --install` to handle first install and later upgrades. Follow your organization's approval and deployment process; do not manually deploy to production just because the command is available.

### Q57. Why might a values change not restart Pods?
Kubernetes rolls out a Deployment when its Pod template changes. A Helm values change that does not alter the rendered Pod template may not trigger a rollout. For example, updating a ConfigMap separately may not restart Pods automatically. A chart can add a checksum annotation derived from a configuration template to the Pod template so config changes trigger a new ReplicaSet. Alternatively, the application may explicitly reload mounted files, but environment-variable changes require new processes.

### Q58. How do Helm rollback and database migrations interact?
Helm rollback restores Kubernetes resources to a previous release revision; it cannot undo a schema migration or external side effect that has already happened. A rollback may even make things worse if the old application version cannot run against the new schema. Design migrations and API contracts for backward compatibility, and define a recovery plan that includes data and external side effects—not only a chart revision.

### Q59. How do labels and selectors cause trouble?
Services select Pods by labels. If a chart changes a label or the selector no longer matches the Pod template, the Service can have no endpoints even while Pods are running. Selectors should be stable and consistent across Deployment and Service templates. Kubernetes selectors for an existing Deployment are immutable in important respects; avoid changing selector identity casually during upgrades.

---

## 8. Security basics for developers

### Q60. Why should application containers run as non-root?
A non-root process has fewer privileges if the application is compromised. Set a non-root user in the image and/or a Kubernetes security context, and ensure files and mounted paths have compatible ownership and permissions. Some mounted volumes or native tools may require permission adjustments; solve those deliberately rather than switching the whole workload to root.

### Q61. How should secrets be managed?
Never commit credentials to source control, bake them into image layers, pass them as build arguments, or print them in logs. Use the organization's secret manager and approved runtime injection path, such as Kubernetes Secret references or an external-secrets integration. Limit access through RBAC and rotate credentials. Base64 in a Kubernetes Secret manifest is encoding, not encryption.

### Q62. What should a developer know about ServiceAccounts and RBAC?
A ServiceAccount is an identity available to a Pod when configured. RBAC determines which API operations that identity can perform. An application that only serves HTTP requests usually does not need permission to list or modify Kubernetes resources. Disable or avoid unnecessary API credentials when possible and request the narrowest permissions required by a real use case.

### Q63. What belongs in a container security context?
Depending on platform policy, useful controls include running as non-root, read-only root filesystem, dropping Linux capabilities, and setting seccomp options. These controls must be compatible with the application and any required writable directories. Image scanning and patching remain necessary; a secure runtime configuration does not eliminate vulnerable dependencies in the image.

### Q64. How do you avoid leaking sensitive information?
Do not log passwords, bearer tokens, session cookies, private keys, or full sensitive request bodies. Redact secrets in exception messages and avoid returning internal stack traces to API clients. When sharing diagnostics, sanitize environment output, rendered Secret manifests, connection strings, and headers. Work with the platform/security team to define approved image sources, vulnerability thresholds, and exception handling.

---

## 9. A developer's release workflow

### Q65. Walk through a safe application deployment.
```text
Code change
   |
   v
Build + unit/integration tests
   |
   v
Package JAR -> build immutable image
   |
   v
Run image locally / smoke-test
   |
   v
Publish image + record digest
   |
   v
Render Helm manifests for target environment
   |
   v
Deploy through approved release pipeline
   |
   v
Check rollout, readiness, logs, metrics, key API flows
   |
   +---- healthy ----> continue observation
   |
   +---- unhealthy --> diagnose; roll back or apply a fix
```

The developer should confirm that tests passed, the image contains the intended artifact, the exact digest is promoted, configuration references exist, rendered manifests are correct, probes and resource values are reasonable, and the deployment succeeds. Afterward, verify actual API behavior and dependency connectivity—not just Pod status. If rollback is needed, consider schema compatibility and already-executed side effects.

### Q66. What does “works in staging but fails in production” suggest?
Compare differences systematically: image digest, runtime/JDK, environment variables, Secrets, profiles, database schema/permissions, network policy, DNS, certificates, resource requests/limits, replica count, traffic volume, and external service endpoints. Avoid changing several variables at once. Capture the exact failing request and timestamp, correlate logs/traces across services, and check whether the failure is deterministic or load-dependent.

### Q67. What should be validated before promoting an image?
Confirm tests and security scanning completed; the image tag/digest is the expected build; architecture and Java version are compatible; the container starts with production-like configuration; secrets are injected rather than embedded; resources and probes reflect measured behavior; and database/API changes are backward-compatible. A successful image build is not proof that the application will run correctly under production constraints.

---

## 10. Troubleshooting scenarios

For incident questions, explain the evidence you would gather before committing to a root cause. Commands below assume you have access to the relevant namespace and permissions.

### Q68. The application works locally but fails in a container. What do you check?
**Symptoms:** Startup error, connection failure, or different behavior only in the image.  
**Investigate:** `docker logs <container>`, `docker inspect <container>`; compare Java version, active profile, environment, working directory, file permissions, bind address, and mounted paths. Check whether a dependency URL incorrectly uses localhost.  
**Likely causes/fix:** Missing file or environment variable, incompatible runtime, wrong port, inaccessible host service, or permission issue. Reproduce using the same image and configuration; document required runtime settings and add a smoke test.

### Q69. The container exits immediately after startup.
Inspect `docker ps -a`, logs, exit code, and the effective entrypoint/command. A missing JAR, invalid JVM option, configuration exception, or a shell script that backgrounds Java and exits are common causes. Run the application in the foreground and use exec-form entrypoint. Do not “fix” it by keeping an unrelated shell alive; that hides the real failure.

### Q70. A Pod is repeatedly OOMKilled.
Run `kubectl describe pod <pod> -n <ns>` and inspect previous logs with `kubectl logs <pod> -n <ns> --previous`. Check termination reason, memory limit, actual container memory, heap settings, GC logs, thread count, direct memory, and workload spikes. Reduce unbounded memory use or concurrency, size the heap below the container limit with native headroom, and right-size the memory limit based on measurements. Increasing the limit may be appropriate, but first establish whether the problem is a leak, burst, or undersized budget.

### Q71. A Pod is Running but the REST API is unavailable.
Check `kubectl get pods -n <ns>`, `kubectl describe pod <pod> -n <ns>`, readiness status, probe events, Service selectors, and EndpointSlices. Confirm the application listens on the expected interface/port and that the Service targetPort matches it. Then inspect ingress/routing, DNS, network policy, authentication, and logs. “Running” is process state, not an end-to-end health guarantee.

### Q72. Readiness probes fail after deployment.
Check the exact probe path, port, scheme, timeout, endpoint exposure/security, and startup duration. Review startup logs and events. The new release may take longer to initialize, use a different management port, or have a misconfigured health group. Use a startup probe for genuinely slow initialization and correct the path/configuration; do not blindly increase thresholds or weaken readiness until it reports success.

### Q73. One service cannot call another using localhost.
The caller is connecting to itself, not the destination Pod. Replace localhost with the destination Service DNS name and the Service port. Verify the Service exists in the expected namespace and has ready EndpointSlices. A local Compose hostname, Kubernetes service name, and production DNS name are different environments; externalize the destination URL.

### Q74. A Service has no ready endpoints.
Compare the Service selector with Pod labels, inspect EndpointSlices, and check readiness. A typo in a label, mismatched port/targetPort, failing readiness probe, or Pods in a different namespace can produce this symptom. Correct the selector/template or fix the underlying readiness issue. Avoid manually editing generated endpoints.

### Q75. A configuration change is not reflected in the application.
Identify the source of truth (Helm values, ConfigMap, Secret, environment variable, or mounted file), then inspect the rendered Deployment and the running Pod's effective configuration without exposing secrets. Environment variables are read at process startup; changing their source does not mutate an existing process. ConfigMap volume updates may be eventual, but the application must reload the file. Trigger a controlled rollout or implement safe reload behavior, and consider a checksum annotation for Helm-managed configuration.

### Q76. Database connections are exhausted after scaling replicas.
Calculate the maximum connection demand: pool maximum per replica × maximum replicas, plus jobs, admin sessions, and other services. Inspect pool active/idle/waiting metrics and database connection usage. Reduce per-replica pool limits, apply sensible timeouts, fix leaked or long-held transactions, and coordinate database capacity. Scaling application Pods can increase total DB concurrency rather than improve throughput.

### Q77. A rolling deployment causes temporary errors.
Inspect readiness, termination grace period, rollout strategy, application shutdown, client retries/timeouts, and capacity during surge. New Pods may be marked ready too early; old Pods may terminate before traffic drains; or the new version may be incompatible with the old schema/API. Correct readiness, support graceful shutdown, preserve sufficient capacity, and use backward-compatible changes. Test rollout behavior under real request load.

### Q78. The app receives SIGTERM but does not shut down gracefully.
Check whether the Java process receives the signal, whether a shell wrapper interferes, and how long in-flight requests take. Configure Spring Boot graceful shutdown and align its timeout with Kubernetes' termination grace period. Stop accepting new requests, drain work, and close pools. Ensure long-running tasks have cancellation/termination semantics. Do not assume a longer grace period fixes a deadlock or non-terminating worker.

### Q79. Helm upgrade changes values but Pods do not restart.
Run `helm template` with the exact values file and compare the rendered Pod template with the live Deployment. The changed value may not be referenced in a template, may affect only a ConfigMap, or may be overridden by a higher-precedence value. Kubernetes only rolls out when the Pod template changes. Add a config checksum annotation or explicitly design reload behavior. Do not restart workloads blindly without verifying which configuration they actually consumed.

### Q80. A new image version fails to start.
Verify the deployed image digest and logs, then compare the previous and new image: Java compatibility, startup command, required environment variables, file paths, native libraries, permissions, and schema/API assumptions. Check image pull events and architecture if the container never starts. Roll back the application release if safe, while separately assessing database migration compatibility and external side effects. Add startup smoke tests and backward-compatibility checks to the release process.

### Q81. CPU throttling causes API latency.
Correlate latency and timeout spikes with CPU usage/throttling, request volume, thread pools, and downstream waits. Check whether a tight CPU limit or low request is causing throttling; also profile CPU-heavy code, lock contention, serialization, and retries. Adjust limits/requests based on measurements and load-test the result. Increasing replicas may help only if the bottleneck is parallelizable and downstream systems have capacity.

### Q82. Multiple replicas behave inconsistently.
Look for in-memory session state, local files, local caches, random configuration, differing image digests, or environment drift. Compare Pod labels, image IDs, startup configuration, and logs. Externalize durable state and define cache consistency/invalidation. If sticky sessions are chosen, document their failover and scaling trade-offs; they are not a substitute for statelessness when correctness depends on shared state.

### Q83. A missing Secret or wrong environment variable prevents startup.
Inspect Pod events and application startup errors; verify Secret name, key name, namespace, permissions, and deployment values. Kubernetes Secrets are namespace-scoped, and a Pod cannot reference a Secret in another namespace directly. Validate required configuration at startup with clear error messages, but never print secret values. Add deployment checks for required keys and test configuration in a staging-like environment.

### Q84. The application is healthy but requests time out intermittently.
Check whether latency is concentrated in the app, DB pool acquisition, downstream calls, CPU throttling, garbage collection, or ingress/network path. Compare request IDs and timestamps across services; inspect queue depth, connection-pool wait, timeouts, retries, and saturation. Align timeouts across hops and use bounded retries with backoff/jitter only for safe operations. Retries can amplify overload and must not be added blindly.

### Q85. A rollout is stuck and old Pods remain.
Check `kubectl rollout status deployment/orders-api -n <ns>`, `kubectl describe deployment/orders-api -n <ns>`, ReplicaSets, Pod events, image-pull status, readiness, and resource capacity. A rollout may stall because new Pods cannot schedule, cannot pull the image, crash, or never become ready. Fix the underlying cause or roll back according to the release procedure. Do not delete old Pods just to force progress before understanding availability and rollback state.

### Q86. A Pod cannot pull the image.
Inspect Pod events for authentication failures, missing tag/digest, registry DNS/TLS issues, architecture mismatch, or rate limits. Confirm the image reference exists and the workload has the correct image-pull credentials or workload identity. Check whether the tag was overwritten or removed. Prefer immutable references and validate image availability before rollout.

### Q87. Production fails but staging passes.
Compare immutable image digest first, then configuration, secret versions, data/schema, permissions, DNS/network policy, TLS certificates, resource constraints, replica count, traffic patterns, and dependency versions. Capture evidence before changing settings. Production may expose concurrency or scale-related defects that staging traffic never triggers. Build a concise timeline and test the leading hypothesis in the safest available environment.

---

## 11. Essential commands

Use the correct namespace and permissions. Commands that print manifests or environment values may reveal secrets; sanitize output before sharing.

### Docker
```bash
docker build -t orders-api:dev .
docker run --rm -p 8080:8080 --env SPRING_PROFILES_ACTIVE=local orders-api:dev
docker ps -a
docker logs <container>
docker logs --tail=200 -f <container>
docker inspect <container>
docker exec -it <container> sh
docker image inspect orders-api:dev
```
- `build`: build an image from the current context.
- `run`: start a container and publish a host port.
- `logs`: inspect stdout/stderr, including startup exceptions.
- `inspect`: examine effective configuration, state, mounts, and exit code.
- `exec`: run a diagnostic command inside a running container; minimal images may not include a shell.
- `image inspect`: inspect image metadata and layers.

### Kubernetes
```bash
kubectl get pods -n <ns>
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl get deploy,rs,svc -n <ns>
kubectl get endpointslices -n <ns>
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
kubectl get pod <pod> -n <ns> -o yaml
```
- `get pods`: see phase, readiness, restarts, and age.
- `describe pod`: inspect events, probes, image-pull status, resource limits, and termination reason.
- `logs --previous`: read the previous container instance after a restart.
- `get endpointslices`: verify which ready backends the Service currently discovers.
- `get events`: find scheduling, image, probe, and lifecycle errors.
- `rollout status/history`: verify progress and inspect Deployment revisions.
- `-o yaml`: inspect effective resource configuration; review carefully for sensitive values.

### Helm
```bash
helm lint ./chart
helm template orders ./chart -n <ns> -f values-prod.yaml
helm history orders -n <ns>
helm get values orders -n <ns>
helm get manifest orders -n <ns>
helm rollback orders <revision> -n <ns>
```
Use `helm lint` for common chart validation, `helm template` to inspect what values actually render, `helm get values` to inspect release values, `helm get manifest` to inspect the release's stored manifests, and `helm history/rollback` to review or restore a prior release. A rollback changes Kubernetes resources, not irreversible external side effects.

---

## 12. References

Prefer the official documentation for version-sensitive behavior. These are starting points; use the version selector matching your environment.

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker container networking](https://docs.docker.com/engine/network/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes Services and networking](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes graceful Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Helm documentation](https://helm.sh/docs/)
- [Helm chart structure](https://helm.sh/docs/topics/charts/)
- [Helm template debugging](https://helm.sh/docs/chart_template_guide/debugging/)
- [Java launcher and JVM options](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html)
- [Spring Boot externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Spring Boot Actuator endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)
- [Spring Boot graceful shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html)

**Review note:** Examples in this handbook are explanatory and were not claimed to have been deployed to a live cluster. Validate image tags, health endpoint configuration, resource values, security policies, and Helm templates in your own versioned environment before production use.

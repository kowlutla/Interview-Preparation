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
A multi-stage build separates the environment that compiles the application from the environment that runs it. The build stage can contain a full JDK, Maven or Gradle, and build dependencies; the final stage can contain only a supported Java runtime and the executable JAR. This usually reduces image size and removes build tools that are unnecessary at runtime.

\`\`\`dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/orders-api.jar app.jar
USER 10001
ENTRYPOINT ["java", "-jar", "app.jar"]
\`\`\`

This example assumes a single-module Maven project and does not show the separate test stage a real release pipeline should run. A multi-module Maven build may need the parent POM and sibling module POMs copied before dependency resolution. Private artifact repositories may also need credentials during the build; use BuildKit secret mounts rather than putting tokens in \`ARG\`, \`ENV\`, or copied settings files. The same principle applies to Gradle: compile with a builder image and copy only the required runtime artifact into the final stage.

### Q10. How do image layers and build caching work?
Docker can reuse the result of a build step when the instruction and its relevant inputs have not changed. That is why Dockerfile order matters. If you copy the entire source tree before resolving dependencies, a small source edit can invalidate the dependency layer and force dependencies to be downloaded again.

For a simple Maven project, copy the POM first, resolve dependencies, and then copy source files. Use a \`.dockerignore\` file to keep local build output, Git metadata, IDE files, logs, and local secrets out of the build context. Excluding \`target/\` is useful when CI builds the artifact inside Docker, but would be wrong if the Dockerfile expects to copy a JAR that was built on the host.

Never bake a secret into an image layer. Deleting it in a later layer does not guarantee removal from the image's history. For build-time authentication, use BuildKit secret or SSH mounts; for runtime credentials, use the organization's approved secret injection mechanism.

### Q11. Should I use a JDK or JRE image?
Use a JDK when compiling or running tools that require compiler modules. A runtime-only image is usually enough to run an executable Spring Boot JAR. Select a maintained distribution compatible with your licensing, patching, architecture, and support requirements. “Smallest image” is not the only criterion; patch availability, debugging support, certificate stores, native libraries, and operational familiarity matter too.

### Q12. How should configuration be passed to a container?
Keep environment-specific configuration outside the image. For Spring Boot, environment variables such as `SPRING_PROFILES_ACTIVE`, `SERVER_PORT`, and `SPRING_DATASOURCE_URL` can override configuration properties using Spring Boot's relaxed binding rules. Credentials should come from a secret-management mechanism, not source code or a Dockerfile. Avoid printing the full environment because it may expose credentials.

### Q13. How does port publishing work?
Three things must line up: the application must listen on a port, the container must be reachable on that interface, and the host or platform must route traffic to it. \`EXPOSE 8080\` is image metadata; it does not publish a port. Locally, \`docker run -p 8080:8080 image\` maps host port 8080 to container port 8080.

A common trap is binding a server only to \`127.0.0.1\` inside the container. That makes it listen on the container's loopback interface, not its externally reachable container interface. A web service normally listens on \`0.0.0.0\` (or the intended network interface). In Kubernetes, \`containerPort\` is descriptive metadata; the Service's \`targetPort\` must still match the actual listening port.

### Q14. How do containers connect to a database or another service?
Inside a container, `localhost` means that same container's network namespace. It does not mean the host machine, a sibling container, or a database container. In Docker Compose, services on the same Compose network can usually reach each other using the service name and container port, for example `jdbc:postgresql://db:5432/orders`. In Kubernetes, use a Service DNS name. Host access from Docker Desktop differs from Linux Docker Engine; use the documented host gateway mechanism for your environment rather than assuming one hostname works everywhere.

### Q15. What are volumes and why shouldn't application state live only in the container?
A container's writable layer is ephemeral: replacing the container can discard its changes. Volumes or mounted storage are useful for data that must survive container replacement, such as local development database files. For typical stateless APIs, durable business data belongs in a database or object store, not in local container files. Do not use a shared filesystem as a substitute for deliberate state ownership and concurrency design.

### Q16. When is Docker Compose useful?
Compose defines a repeatable local environment for the API and its dependencies, such as a database, Redis, or Kafka. Services on the same Compose network can usually reach one another by service name, while the port used between containers is the container port—not necessarily the host-published port.

\`\`\`yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/orders
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: local-only-password
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
\`\`\`

This is only a local-development sketch: use a local-only password and never reuse it in shared or production environments. The database volume preserves local database files across container recreation. Add health checks or application-level retry/readiness behavior when startup order matters—Compose starting the database container does not necessarily mean the database is ready to accept connections. A working Compose setup also does not prove production Kubernetes networking, secrets, resource limits, or rollout behavior are correct.

### Q17. How do you debug a container that exits?
Start with \`docker ps -a\`, then examine \`docker logs <container>\` and \`docker inspect <container>\`. Look at the exit code, command/entrypoint, working directory, mounts, permissions, and the application's startup exception. Common causes include a missing JAR, incompatible Java runtime, invalid JVM arguments, missing configuration, or a script that backgrounds Java and exits.

Run the application in the foreground as the main container process. In Kubernetes, also inspect Pod events and current/previous container logs; repeated failures may lead to \`CrashLoopBackOff\`, which is Kubernetes backing off between restarts, not the underlying cause. If the container exits before logs appear, verify that the command actually launches the application and that the required files are present in the image.

### Q18. Why are immutable image tags and digests important?
A tag is a convenient name, not an immutable identity: a registry can move a tag such as \`latest\` to different content. A digest identifies exact image content and is useful for audit trails, incident reproduction, and confirming which build actually ran. Prefer unique release tags and record the digest; promote the same tested image across environments instead of rebuilding it for each environment.

Also understand \`imagePullPolicy\`. Kubernetes commonly defaults to \`Always\` when the image tag is \`latest\`, and to \`IfNotPresent\` for other tags when the field is omitted. With a mutable tag, nodes may not all run the content you expect. Prefer immutable tags or digests and make the pull policy consistent with the release process.

### Q19. How should an application handle shutdown signals?
With exec-form \`ENTRYPOINT\`, Java is launched without an unnecessary shell layer, so it can receive the container's termination signal. With Kubernetes, deletion starts a bounded termination period: a configured \`preStop\` hook runs within that grace period, the runtime sends the stop signal, and the container is force-killed if it has not exited before the deadline.

Endpoint and load-balancer updates are not instantaneous across all clients and network paths. The application should stop accepting new work, complete or safely cancel in-flight requests, close pools, and exit before the deadline. Budget any hook time and Spring Boot's shutdown timeout inside \`terminationGracePeriodSeconds\`. Test termination under active traffic; graceful shutdown reduces disruption but cannot guarantee that every request finishes or that a client can tell whether a request completed before the connection closed.

### Q20. What is a good Docker troubleshooting checklist?
Check build context and ignore rules; image architecture and Java version; command/entrypoint; user and file permissions; listening address and port; runtime environment variables; mounted paths; logs and exit code; network/DNS reachability; and memory limits. Reproduce with the exact image and configuration used in the target environment whenever possible.

---

## 3. Java runtime behavior in containers

### Q21. Why can a Java container be OOMKilled when the heap limit looks reasonable?
The container memory limit is a budget for total memory charged to the container, not just the Java object heap. The JVM also needs metaspace, JIT code cache, thread stacks, direct/off-heap buffers, garbage-collector structures, native libraries, and other native allocations. Depending on the runtime and cgroup accounting, charged file cache and memory-mapped pages may matter too. If \`-Xmx\` is nearly the whole container limit, total usage can exceed the limit even before the heap reaches its maximum.

Distinguish a container/OS OOM kill from a Java \`OutOfMemoryError\`. A Java error may be logged and might allow a heap dump; a kernel OOM kill can terminate the process before useful Java diagnostics are written. Check the last termination reason, container memory metrics, GC logs, thread count, direct-buffer use, and native-memory behavior. Newer supported HotSpot JDKs are container-aware on supported platforms, but verify the actual JDK and make sure container support has not been disabled.

### Q22. How should heap sizing be approached?
Treat heap sizing as budget allocation. Suppose the Pod has a 1 GiB memory limit: setting \`-Xmx1g\` leaves no practical room for thread stacks, metaspace, direct buffers, JIT code cache, and other native allocations. A setting such as \`-XX:MaxRAMPercentage\` lets HotSpot calculate maximum heap from the memory it detects, but it is not a complete memory-management strategy and the right percentage depends on the workload and JDK.

For Java 21 HotSpot, the documented default for \`MaxRAMPercentage\` is 25%; do not assume every JDK vendor/version or runtime configuration behaves identically. Measure peak container memory under representative load, observe garbage collection and native memory, and leave headroom. Heap-used graphs alone are not enough: a process can be OOMKilled while heap usage appears below \`-Xmx\`.

### Q23. How does CPU throttling affect API latency?
A CPU limit is enforced through throttling. When a container exhausts its allotted CPU time, it can be forced to wait even if the node has spare CPU. The JVM and HTTP server remain alive, but requests take longer, queues grow, clients time out, and retries can amplify load.

Correlate latency with CPU throttling, CPU consumption, thread-pool queues, garbage collection, and downstream waits. Raising a limit can help, but profile CPU-heavy code, serialization, lock contention, and retry storms first. CPU requests affect scheduling and CPU-based HPA utilization; they are not the same as CPU limits. Adding replicas helps only if the work can be parallelized and downstream systems have spare capacity.

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
A request is the resource quantity Kubernetes uses when deciding where a Pod can fit. For CPU, the request also influences relative CPU shares when containers compete for CPU; it does not give a container an exclusive physical core. A CPU limit caps CPU time and can cause throttling. A memory limit is enforced reactively and can lead to an OOM kill when the container exceeds its cgroup budget under memory pressure.

If a limit is specified but no request is supplied, Kubernetes normally copies the limit into the request unless admission-time defaults or policies change it. Set values from startup and load-test measurements, peak traffic, JVM native overhead, and cluster capacity—not from a generic recipe.

\`\`\`yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "768Mi"
\`\`\`

These are examples only. A request that is too high can make the Pod difficult to schedule; a CPU limit that is too low may cause latency; a memory limit that is too low can cause OOM kills. In a Java service, include startup peaks and non-heap memory when budgeting.

### Q30. ConfigMap versus Secret?
A ConfigMap holds non-confidential configuration. A Secret is intended for credentials and other sensitive values, but base64 encoding is not encryption. Access should be constrained with RBAC, encryption-at-rest configuration, and the organization's approved secret-management workflow; some platforms use an external secret manager. Never expose secret values in logs, support tickets, or broadly shared rendered manifests.

Environment variables are snapshots taken when a container starts. Updating a ConfigMap or Secret does not change environment variables in an already-running process, so a rollout is normally needed. Mounted ConfigMap/Secret volumes can be updated eventually, but the application must re-read or reload the file to use the new value. A volume mounted with \`subPath\` does not receive automatic projected-volume updates. A checksum annotation on the Pod template or a controlled rollout is a common way to keep configuration and process lifecycle aligned.

### Q31. What does a rolling update do?
A Deployment gradually replaces old Pods with new Pods. \`maxSurge\` is the allowed number (or percentage) of extra Pods above the desired replica count; \`maxUnavailable\` is how many replicas may be unavailable during the rollout. Cluster capacity must accommodate the strategy, and new Pods should not become Ready until they can safely serve requests. Readiness affects rollout availability and Service traffic, but it does not prove every business operation works.

A rollout can still cause errors if the new version is incompatible with the old database schema or API contract, if the old Pods terminate before requests drain, or if readiness is too optimistic. Even a rollout marked complete only means Kubernetes reached the desired Deployment state. Verify representative API flows, dependency connectivity, errors, latency, and logs after deployment.

### Q32. How does rollback work and what are its limits?
A Deployment rollback changes the Pod template; a Helm rollback restores resources represented by an earlier release revision. Neither reverses database schema/data changes, messages already published, emails sent, or calls already made to external systems. Rolling the binary back can itself fail if the old application cannot run against the new schema.

Prefer expand/contract migrations: add backward-compatible schema first, deploy code that works with both the old and new representation, migrate data, then remove the old schema only in a later release after older code is no longer running. For irreversible data transformations, plan a forward fix or a tested recovery procedure instead of assuming Helm can undo the change.

### Q33. What does Horizontal Pod Autoscaler do?
HPA adjusts replica count based on observed metrics, commonly CPU or memory utilization and sometimes custom/external metrics. CPU utilization is often expressed relative to CPU requests, so incorrect requests can distort scaling. Scaling out helps only if the service is horizontally scalable and downstream dependencies can handle more load. Ensure state is externalized, connection pools are bounded per replica, and scale-up behavior accounts for startup time and warm-up. HPA does not fix a slow query or a saturated database by itself.

### Q34. Why is local Pod storage not durable?
Pods are replaceable, and a container writable layer is lost when the container is recreated. Pod-scoped ephemeral storage also has lifecycle limits. Use a database or object store for business data. PersistentVolumes and PersistentVolumeClaims can provide durable mounted storage where the workload truly needs files, but persistence brings lifecycle, performance, backup, and access-mode considerations. Most REST services should not rely on local disk for session state or uploaded files.

### Q35. What should developers know about cluster components?
Know the API server as the Kubernetes API endpoint, the scheduler as the component assigning Pods to nodes, controllers as desired-state reconcilers, and kubelet/runtime as the node-side mechanism that starts containers and reports status. That mental model helps explain Pending Pods, image-pull failures, and restarts. Deep control-plane operations and node administration are platform-team responsibilities for this handbook's scope.

### Q36. Example Deployment and Service for a Spring Boot API?
\`\`\`yaml
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
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/orders-api:1.4.2
          ports:
            - name: http
              containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
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
\`\`\`

Treat this as a starting point, not production-ready YAML for every environment. The named Actuator health groups must be exposed and configured in the chosen Spring Boot version. If management endpoints use a separate port, set the probe port accordingly. The non-root user must be able to read the JAR and any required files; if the application needs a writable temporary directory, provide a deliberate writable mount rather than weakening the whole security context. The database URL is shown as a Secret reference only to keep the example short; production deployments usually inject URL, username, and password using the approved secret workflow. Validate this manifest and its resource values in a representative environment before rollout.

## 5. Spring Boot configuration, health, and shutdown

### Q37. How do profiles and externalized configuration work in Kubernetes?
Build one image and supply environment-specific values at deployment time. For example, `SPRING_PROFILES_ACTIVE=prod` selects a profile, while environment variables can override properties in `application.yml`. Avoid maintaining a different image per environment because it weakens promotion and reproducibility. Profiles should express configuration differences, not hide incompatible code paths that only get tested in one environment.

### Q38. How should readiness, liveness, and startup probes differ?
Think of the probes as answering three different questions:

- **Startup:** Has this instance finished starting within the allowed time? While a startup probe has not succeeded, Kubernetes does not run the liveness and readiness probes for that container. This protects slow but healthy startup from being killed too early.
- **Readiness:** Should this Pod receive new Service traffic right now? A failed readiness probe marks the container/Pod not ready for normal Service routing; it does not itself restart the container.
- **Liveness:** Is the application stuck in a way that a restart is expected to fix? Repeated liveness failures cause the container to be restarted.

A liveness endpoint should not fail simply because a shared database or downstream API is temporarily down; restarting every replica can amplify the incident. Readiness may include a dependency only if the service truly cannot serve useful traffic without it, and the check should be fast and bounded. Configure paths, ports, timeouts, periods, and failure thresholds based on observed startup behavior—not by copying values blindly.

### Q39. Why can bad probes cause an outage?
Imagine a Spring Boot service takes 70 seconds to initialize caches and validate configuration, but its liveness probe starts after 10 seconds and allows only a short failure window. Kubernetes may repeatedly restart it before it ever reaches the ready state. A startup probe gives initialization its own bounded window and only allows liveness/readiness enforcement after startup succeeds.

The opposite problem is a readiness probe pointed at a path that does not exist, a secured endpoint the kubelet cannot access, or the wrong management port. The process may be Running indefinitely but receive no Service traffic. Check Pod events and the exact HTTP path, port, scheme, response code, endpoint exposure, and startup logs. Fix the cause; increasing thresholds without understanding the delay can hide a broken application or lengthen failure detection.

### Q40. How should database connections be configured and pooled?
Use externalized URL, username, and password settings, configure connection and query timeouts, and size the pool according to workload and database capacity. Pool size must be considered across all replicas: 10 connections per replica multiplied by 20 replicas means up to 200 connections, before migrations, jobs, and other services are counted. A pool too small causes request waiting; a pool too large can overwhelm the database and increase contention. Measure acquisition latency, active/idle counts, waiters, query latency, and DB saturation.

### Q41. What is graceful shutdown for a REST service?
Graceful shutdown gives the application a chance to stop taking new requests and complete work already in progress before the process exits. Spring Boot's embedded servers support graceful shutdown, but defaults can differ by Spring Boot version; check the version used by the service. Where explicit configuration is needed, \`server.shutdown=graceful\` enables it and \`spring.lifecycle.timeout-per-shutdown-phase\` sets the application lifecycle timeout.

In Kubernetes, the Pod's termination grace period is a larger deadline that must cover any \`preStop\` hook plus the application's shutdown time. Endpoint updates and proxy convergence take time, so a few requests may still arrive as termination begins. Configure bounded HTTP/database timeouts, stop background consumers deliberately, and ensure long-running tasks can be cancelled. A caller may not know whether an operation completed before the connection closed, so idempotency and safe retry semantics still matter.

### Q42. How do multiple replicas affect state and caches?
Any request may reach any ready replica. In-memory sessions, locks, counters, or local caches are not automatically shared. A later request may land on a different Pod, and a Pod restart discards its memory. Prefer stateless APIs, external session storage when sessions are unavoidable, and cache designs with explicit consistency and invalidation rules. Sticky sessions can route around the issue but do not make local state durable and can complicate scaling and failover.

### Q43. How should logs and traces be handled?
Write application logs to stdout/stderr so the platform's log collector can capture them. Include timestamps, severity, service/version, and a request or trace correlation ID; avoid logging credentials, access tokens, or sensitive payloads. Use structured logs where supported. Correlation IDs must be propagated across service calls rather than generated independently at every hop. Logs from a replaced Pod may no longer be available through the current Pod, so central collection is important.

### Q44. What is the relationship between Actuator health and Kubernetes probes?
Spring Boot Actuator exposes health information, but an endpoint only works if the relevant Actuator dependency, exposure settings, security rules, health group, and port configuration allow it. For supported Spring Boot versions, the liveness and readiness groups commonly map to \`/actuator/health/liveness\` and \`/actuator/health/readiness\`. Confirm those exact paths in the deployed application instead of assuming the example works unchanged.

Keep probe endpoints minimal, fast, and reachable from the kubelet. If the management server runs on a separate port, the probe must target that port. Avoid exposing every management endpoint publicly just to make a health check work. Also keep the intent clear: readiness answers whether the instance should receive traffic; liveness should not simply mirror the health of every external dependency.

## 6. Networking and service discovery

### Q45. How does one backend service call another in Kubernetes?
A caller uses the destination Service's DNS name and Service port, for example `http://inventory-service:8080` when both services are in the same namespace and the Service exposes port 8080. Across namespaces, use the fully qualified service DNS name when necessary, commonly `inventory-service.team-a.svc.cluster.local` (the cluster domain may differ). The Service gives clients a stable name while Pods change.

### Q46. Service port versus targetPort versus containerPort?
- \`port\` is the port clients use on the Kubernetes Service.
- \`targetPort\` is the destination port on each selected Pod; it can be a number or a named container port.
- \`containerPort\` documents a port in the Pod spec and provides a name that a Service can reference. It does not itself publish the port or make the application listen on it.

For example, a Service can expose port 80 and forward to the Pod's named \`http\` port, which maps to the application listening on 8080. If clients cannot connect, verify the actual application port, Service port, targetPort, selector, and ready EndpointSlices rather than assuming \`containerPort\` opens the port.

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
Helm merges the chart defaults with values supplied by the release command, renders the templates, and generates Kubernetes resources. For example, \`image.tag\`, \`replicaCount\`, \`resources\`, and \`probes\` values may become fields in the Deployment template. A typo in a values key may not fail if the template never references it, so a successful Helm command does not guarantee the intended value was used.

\`\`\`yaml
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
\`\`\`

The chart's \`version\` identifies the chart package; \`appVersion\` is informational metadata about the application unless templates explicitly use it. Do not assume changing \`appVersion\` changes the deployed image tag. Render the chart using the same values and flags as the real release process, and review the resulting Pod template, config references, ports, probes, and resources.

### Q55. How do you inspect and validate a chart before deploying?
Run `helm lint ./chart` to catch common chart issues and `helm template orders ./chart -f values-prod.yaml` to inspect rendered resources. Review image reference, labels/selectors, ports, probes, Secret/ConfigMap references, resource settings, and rollout strategy. Template output can be valid YAML but still semantically wrong, so check the fields that affect your application.

### Q56. How do you install or upgrade a release?
A common workflow is to inspect the rendered output first, then run the approved release command:

\`\`\`bash
helm lint ./chart
helm template orders ./chart -n production -f values-prod.yaml
helm upgrade --install orders ./chart -n production -f values-prod.yaml --wait --timeout 5m
helm history orders -n production
helm rollback orders 3 -n production
\`\`\`

\`upgrade --install\` handles both an existing release and a first install. \`--wait\` asks Helm to wait for supported resources to become ready until the timeout; it is useful, but it is not an application smoke test. A release may be Kubernetes-ready while a business flow still fails.

Be deliberate about values merging on upgrades. Chart defaults, previously stored release values, and explicitly supplied files/flags can interact; Helm provides reset/reuse options that change this behavior. Inspect the effective values and rendered manifests, avoid putting secrets on a command line, and follow the team's approved deployment process.

### Q57. Why might a values change not restart Pods?
A Helm value change only changes the resources that the templates actually render. Kubernetes creates a new Deployment ReplicaSet only when the Pod template changes. If a value changes a ConfigMap but leaves the Deployment template unchanged, the Pods normally keep running with their existing environment variables; mounted files may update eventually, but the application must reload them.

A common chart pattern adds a checksum of the rendered configuration to the Pod-template annotations. The checksum changes when the ConfigMap content changes, which triggers a Deployment rollout. Another option is application-level reload for mounted files. Remember that a ConfigMap/Secret mounted with \`subPath\` does not receive automatic projected-volume updates. First compare the rendered template and live Deployment, then decide whether a rollout or a supported reload is appropriate.

### Q58. How do Helm rollback and database migrations interact?
Helm rollback restores Kubernetes resources to a previous release revision; it cannot undo a schema migration or external side effect that has already happened. A rollback may even make things worse if the old application version cannot run against the new schema. Design migrations and API contracts for backward compatibility, and define a recovery plan that includes data and external side effects—not only a chart revision.

### Q59. How do labels and selectors cause trouble?
A Service selects Pods by exact label key/value matches. If the Service selector and Pod-template labels drift apart, Pods can be Running while the Service has no endpoints. Compare the rendered manifests and inspect the Service's EndpointSlices; also check readiness because matching but unready Pods are not normally eligible for ordinary traffic.

Keep the Deployment selector stable and consistent with its Pod-template labels. Selectors on an existing Deployment are immutable in important respects, so changing an app label used by the selector can make an upgrade fail instead of smoothly replacing Pods. Add a selector/label check to chart review and inspect rendered output after changes.

## 8. Security basics for developers

### Q60. Why should application containers run as non-root?
A non-root process has fewer privileges if the application is compromised. Set a non-root user in the image and/or a Kubernetes security context, and ensure files and mounted paths have compatible ownership and permissions. Some mounted volumes or native tools may require permission adjustments; solve those deliberately rather than switching the whole workload to root.

### Q61. How should secrets be managed?
Never commit credentials to source control, bake them into image layers, pass them as Docker build arguments, or print them in logs. Use the organization's approved secret manager and runtime injection path—for example, a Kubernetes Secret reference or an external secret integration. Limit who can read secrets, enable the platform's encryption-at-rest controls where required, and define a rotation process.

Kubernetes Secret data is base64-encoded in the API representation; that encoding alone provides no confidentiality. Environment variables are convenient but can be exposed through diagnostics or process environments, so do not dump the environment in support logs. Mounted files may be preferable for some secret types, provided permissions and reload behavior are handled correctly.

### Q62. What should a developer know about ServiceAccounts and RBAC?
A ServiceAccount is an identity available to a Pod when configured. RBAC determines which API operations that identity can perform. An application that only serves HTTP requests usually does not need permission to list or modify Kubernetes resources. Disable or avoid unnecessary API credentials when possible and request the narrowest permissions required by a real use case.

### Q63. What belongs in a container security context?
Useful controls, subject to the platform's policy, include running as a non-root user, disabling privilege escalation, dropping unneeded Linux capabilities, using the runtime's default seccomp profile, and making the root filesystem read-only where the application supports it. These settings should be tested with the actual image and mounted paths: a JVM or library may need a writable temporary directory, so a read-only root filesystem may require an explicit writable \`emptyDir\` mount for \`/tmp\`.

Do not switch the whole application to root simply to work around a file-permission issue. Identify which path needs access and grant only that access. Security context controls complement, rather than replace, patched base images, dependency scanning, RBAC, and safe secret handling.

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
**Investigate:** Check \`kubectl get pods -n <ns>\`, \`kubectl describe pod <pod> -n <ns>\`, readiness status, probe events, Service selectors, and the Service's EndpointSlices. Confirm that the application is listening on the expected interface and port, that the Service \`targetPort\` matches, and that the management/health port has not been confused with the application port. Then check ingress routing, DNS, NetworkPolicy, TLS, authentication, and application logs.

**Likely causes:** Running only confirms that the container process exists. The app may not be Ready, may be bound to loopback, may have no matching Service endpoints, or may be reachable only on a different port.  
**Fix and prevention:** Correct the selector/port/probe configuration, then verify a real API request and dependency call—not just the Pod phase. Keep an after-deploy smoke test for a stable, non-destructive endpoint.

### Q72. Readiness probes fail after deployment.
Check the exact path, port, scheme, timeout, response code, Actuator exposure/security, and startup duration. Review \`kubectl describe pod\` events alongside startup logs. A new release may initialize more slowly, use a different management port, or expose a health group at a different path. A startup probe can protect genuinely slow initialization by delaying liveness/readiness enforcement until startup succeeds; it should not hide an app that never becomes ready.

Fix the underlying path, configuration, dependency, or startup problem before tuning thresholds. Base the startup window on measured cold-start time plus reasonable variability. Add the probe check to the rendered-manifest and post-deploy verification process so a misconfigured endpoint is found before production rollout.

### Q73. One service cannot call another using localhost.
The caller is connecting to itself, not the destination Pod. Replace localhost with the destination Service DNS name and the Service port. Verify the Service exists in the expected namespace and has ready EndpointSlices. A local Compose hostname, Kubernetes service name, and production DNS name are different environments; externalize the destination URL.

### Q74. A Service has no ready endpoints.
Compare the Service selector with the labels on the Pod template and inspect the EndpointSlices associated with that Service. For example:

\`\`\`bash
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<service>
\`\`\`

Look at endpoint readiness/serving/terminating conditions and the port recorded in the slice. Common causes include a label typo, a failing readiness probe, the wrong target port, or a Service and Pod in different namespaces. Correct the manifest or underlying readiness/configuration issue; do not manually edit controller-managed endpoint data. Add a basic Service-selector check to chart review.

### Q75. A configuration change is not reflected in the application.
Identify the source of truth (Helm values, ConfigMap, Secret, environment variable, or mounted file), then inspect the rendered Deployment and the running Pod's effective configuration without exposing secrets. Environment variables are read at process startup; changing their source does not mutate an existing process. ConfigMap volume updates may be eventual, but the application must reload the file. Trigger a controlled rollout or implement safe reload behavior, and consider a checksum annotation for Helm-managed configuration.

### Q76. Database connections are exhausted after scaling replicas.
Calculate the maximum connection demand: pool maximum per replica × maximum replicas, plus jobs, admin sessions, and other services. Inspect pool active/idle/waiting metrics and database connection usage. Reduce per-replica pool limits, apply sensible timeouts, fix leaked or long-held transactions, and coordinate database capacity. Scaling application Pods can increase total DB concurrency rather than improve throughput.

### Q77. A rolling deployment causes temporary errors.
Correlate each error with the rollout timeline and identify whether it came from a new Pod, an old Pod being terminated, or a downstream service. Check readiness accuracy, startup duration, \`maxSurge\`/\`maxUnavailable\`, capacity during surge, termination grace period, preStop hooks, and application shutdown logs. Also investigate keep-alive connections, load-balancer convergence, and whether clients retry non-idempotent operations.

New Pods can receive traffic too early if readiness is optimistic; old Pods can still receive requests after termination begins because routing changes take time. Improve readiness and graceful draining, maintain capacity for the rollout strategy, keep old and new versions compatible with the same schema/API contracts, and test the rollout under realistic load. Capture error rate and latency around deploys so the team can distinguish a rollout regression from a coincident dependency incident.

### Q78. The app receives SIGTERM but does not shut down gracefully.
Check Pod termination events, whether a \`preStop\` hook runs, the effective container command, and application shutdown logs. A shell wrapper may interfere with signal forwarding; prefer exec-form Docker entrypoints or a deliberate init process. Check the Spring Boot version: recent lines enable graceful shutdown for embedded servers by default, while older versions/configurations may require \`server.shutdown=graceful\`. \`spring.lifecycle.timeout-per-shutdown-phase\` controls the application lifecycle timeout where applicable.

The Pod grace period must cover hook time and application shutdown, with enough margin for the process to exit before SIGKILL. If a worker never terminates, investigate blocked I/O, missing timeouts, or non-cooperative background tasks instead of only increasing the grace period. Test while requests are in flight and check that clients handle an interrupted connection safely.

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
Start with \`kubectl rollout status deployment/orders-api -n <ns>\`, then inspect the Deployment, ReplicaSets, new Pods, events, image-pull status, readiness failures, and available capacity. Determine whether the new ReplicaSet cannot schedule, cannot pull the image, is crashing, or is running but never becomes Ready. An old ReplicaSet can intentionally remain available while Kubernetes tries to establish the new version.

Fix the cause or follow the release procedure to roll back. Do not delete old Pods merely to make the status look cleaner: they may be the only healthy capacity available. After recovery, add a check for the failure mode—such as pre-deploy image availability, probe verification, or resource validation—to reduce repeat incidents.

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

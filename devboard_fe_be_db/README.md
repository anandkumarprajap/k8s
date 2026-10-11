# DevBoard Kubernetes 

This guide explains how your DevBoard frontend, backend, and PostgreSQL
database connect inside a three-node Kind cluster, including the YAML
corrections, deployment commands, troubleshooting, ports, and
architecture diagram.

Your application flow is:

Browser → Frontend Service → Frontend Pods → Backend Service → Backend
Pods → PostgreSQL Service → PostgreSQL Pod

Kubernetes connects these components through Namespaces, label
selectors, Services, ConfigMaps, Secrets, environment variables, and DNS
names.

## Step 1 --- Understand and correct every YAML file
### 1.1 Recommended Project Structure

Keep all Kubernetes configuration files inside a single `k8s/` directory. This makes the deployment workflow easier to understand, maintain, and execute in the correct order.

#### Directory Structure

```text
GitHub-Actions-Revision/
└── k8s/
    ├── kind-config.yml
    ├── namespace.yml
    ├── configMap.yml
    ├── secrets.yml
    ├── postgres-init.yml
    ├── postgres-deployment.yml
    ├── postgres-service.yml
    ├── backend-deployment.yml
    ├── backend-service.yml
    ├── frontend-deployment.yml
    ├── frontend-service.yml
    └── README.md
```

#### File Descriptions

| File | Purpose |
|---|---|
| `kind-config.yml` | Creates the Kind cluster configuration and maps the host port to the cluster. |
| `namespace.yml` | Creates the `devboard` namespace to isolate application resources. |
| `configMap.yml` | Stores non-sensitive database configuration. |
| `secrets.yml` | Stores database credentials and other sensitive configuration. |
| `postgres-init.yml` | Provides SQL initialization scripts for the PostgreSQL database. |
| `postgres-deployment.yml` | Deploys PostgreSQL and manages its Pod. |
| `postgres-service.yml` | Provides stable internal DNS and network access to PostgreSQL. |
| `backend-deployment.yml` | Deploys the Go backend and injects database settings. |
| `backend-service.yml` | Provides stable internal DNS and network access to the backend. |
| `frontend-deployment.yml` | Deploys the frontend application with five replicas. |
| `frontend-service.yml` | Exposes the frontend through a NodePort Service. |
| `README.md` | Documents the architecture, configuration files, deployment commands, verification, and troubleshooting steps. |

**Important:** The `kind-config.yml` file configures the Kind cluster; it is not a Kubernetes resource that you apply using `kubectl`. The other YAML files define Kubernetes resources that you can apply to the cluster.


### 1.2 Correct the Namespace YAML

File: `namespace.yml`

```yml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: devboard
```
Connection: All application resources specify `namespace: devboard`.
This allows their names to be resolved within the same namespace.

### 1.3 Do you need `pod.yml`?

No. You do not need a separate frontend `pod.yml` for this deployment.

Your `frontend-deployment.yml` already defines a Pod template:

```yml
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: devboard-frontend
      template:
        metadata:
          labels:
            app: devboard-frontend
        spec:
          containers:
            - name: devboard-frontend
              image: anandkumarprajap/devboard-frontend:latest
```
The Deployment creates and maintains five Pods through a ReplicaSet. A
standalone Pod is useful for temporary debugging or simple experiments,
but it is not necessary for this application.

### 1.4 Correct the frontend Deployment

File: `frontend-deployment.yml`

```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: devboard-frontend-deployment
      namespace: devboard
      labels:
        app: devboard-frontend
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: devboard-frontend
      template:
        metadata:
          labels:
            app: devboard-frontend
        spec:
          containers:
            - name: devboard-frontend
              image: anandkumarprajap/devboard-frontend:latest
              ports:
                - name: http
                  containerPort: 4173
```
Important connection: `spec.selector.matchLabels.app` must match
`spec.template.metadata.labels.app`. The Service also selects this same
label.

### 1.5 Correct the frontend Service

File: `frontend-service.yml`

```yml
    apiVersion: v1
    kind: Service
    metadata:
      name: devboard-frontend-service
      namespace: devboard
    spec:
      type: NodePort
      selector:
        app: devboard-frontend
      ports:
        - name: http
          protocol: TCP
          port: 8080
          targetPort: 4173
          nodePort: 30080
```
Your original YAML contained `type: NodePort` twice. Keep it only once.

  Setting              Purpose
  -------------------- ------------------------------------------------
  `port: 8080`         Port exposed by the Service inside the cluster
  `targetPort: 4173`   Port on which the frontend container listens
  `nodePort: 30080`    Port exposed on each Kind node
  `selector`           Finds the frontend Pods by their labels

The Service's port `8080` does not mean the frontend container must
listen on `8080`. It forwards traffic to container port `4173`.

### 1.6 Correct the Secret

File: `secrets.yml`
```yml
    apiVersion: v1
    kind: Secret
    metadata:
      name: devboard-secrets
      namespace: devboard
    type: Opaque
    stringData:
      POSTGRES_PASSWORD: "devboard"
      POSTGRES_DB: "devboard"
```
Using `stringData` allows you to provide readable strings; Kubernetes
converts them into the Secret's stored data format.

Your original `type: opaque` is incorrect because the standard Secret
type is case-sensitive: `Opaque`.

Security note: `stringData` is convenient for learning, but the file
contains plaintext credentials. Do not commit real production
credentials to Git. Kubernetes Secrets also require appropriate RBAC and
encryption-at-rest configuration for stronger protection.

### 1.7 Correct the ConfigMap

File: `configMap.yml`
```yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: devboard-configmap
      namespace: devboard
    data:
      POSTGRES_USER: "devboard"
```
Connection: The backend and PostgreSQL Deployment read `POSTGRES_USER`
from this ConfigMap.

### 1.8 Correct the PostgreSQL initialization ConfigMap

File: `postgres-init.yml`

Keep your existing SQL schema and seed statements, but use this YAML
structure:
```yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: postgres-init
      namespace: devboard
    data:
      01_schema.sql: |
        CREATE TABLE IF NOT EXISTS projects (
          id SERIAL PRIMARY KEY,
          name VARCHAR(200) NOT NULL,
          description TEXT,
          owner_id INTEGER,
          created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
        );

        CREATE TABLE IF NOT EXISTS tasks (
          id SERIAL PRIMARY KEY,
          title VARCHAR(300) NOT NULL,
          description TEXT,
          project_id INTEGER NOT NULL
            REFERENCES projects(id) ON DELETE CASCADE,
          assignee_id INTEGER,
          status VARCHAR(20) NOT NULL DEFAULT 'todo'
            CHECK (status IN ('todo','in_progress','blocked','done')),
          priority VARCHAR(10) NOT NULL DEFAULT 'medium'
            CHECK (priority IN ('low','medium','high')),
          due_date DATE,
          created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
          updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
        );

        CREATE INDEX IF NOT EXISTS idx_tasks_project_id
          ON tasks(project_id);

        CREATE INDEX IF NOT EXISTS idx_tasks_status
          ON tasks(status);

        CREATE OR REPLACE FUNCTION touch_updated_at()
        RETURNS TRIGGER AS $$
        BEGIN
          NEW.updated_at = NOW();
          RETURN NEW;
        END;
        $$ LANGUAGE plpgsql;

        DROP TRIGGER IF EXISTS tasks_touch_updated_at ON tasks;

        CREATE TRIGGER tasks_touch_updated_at
        BEFORE UPDATE ON tasks
        FOR EACH ROW
        EXECUTE FUNCTION touch_updated_at();

      02_seed.sql: |
        INSERT INTO projects (id, name, description, owner_id) VALUES
          (1, 'DevBoard MVP', 'Ship the v1 task tracker', 1),
          (2, 'Marketing Site', 'Landing page + launch blog', 1)
        ON CONFLICT (id) DO NOTHING;

        INSERT INTO tasks
          (title, description, project_id, assignee_id,
           status, priority, due_date)
        VALUES
          ('Design the task schema', 'projects, tasks, statuses',
           1, 1, 'done', 'high', '2026-05-05'),
          ('Build the kanban board', 'drag and drop columns',
           1, 2, 'in_progress', 'high', '2026-06-18'),
          ('Wire up the dashboard hero', 'velocity + chip stats',
           1, 1, 'in_progress', 'medium', '2026-06-20'),
          ('Add the command bar', 'global search',
           1, 3, 'todo', 'medium', '2026-06-25'),
          ('Dark mode polish', 'grain + gradients',
           1, 2, 'todo', 'low', NULL),
          ('Fix flaky avatar colors', 'hash collision on initials',
           1, 1, 'blocked', 'high', '2026-06-15'),
          ('Write component tests', 'Vitest + testing-library',
           1, 3, 'todo', 'medium', NULL),
          ('Ship the v1 release', 'tag and announce',
           1, 1, 'todo', 'high', '2026-06-30'),
          ('Draft the launch blog post', '',
           2, 2, 'in_progress', 'medium', '2026-06-22'),
          ('Hero illustration', '',
           2, 3, 'todo', 'low', NULL);

        SELECT setval(
          'projects_id_seq',
          (SELECT MAX(id) FROM projects)
        );
```
This retains your two tables, indexes, update trigger, sample projects,
and task records. Keep the complete SQL in the actual file.

How initialization works: The PostgreSQL image runs `.sql` files mounted
under `/docker-entrypoint-initdb.d` when it initializes a new, empty
database data directory. These scripts do not automatically rerun every
time the Pod restarts.

### 1.9 Correct the PostgreSQL Deployment

File: `postgres-deployment.yml`
```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: postgres-deployment
      namespace: devboard
      labels:
        app: devboard-postgres
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: devboard-postgres
      template:
        metadata:
          labels:
            app: devboard-postgres
        spec:
          containers:
            - name: postgres
              image: postgres:16-alpine
              ports:
                - name: postgres
                  containerPort: 5432
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    configMapKeyRef:
                      name: devboard-configmap
                      key: POSTGRES_USER
                - name: POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: devboard-secrets
                      key: POSTGRES_PASSWORD
                - name: POSTGRES_DB
                  valueFrom:
                    secretKeyRef:
                      name: devboard-secrets
                      key: POSTGRES_DB
              volumeMounts:
                - name: init
                  mountPath: /docker-entrypoint-initdb.d
                  readOnly: true
          volumes:
            - name: init
              configMap:
                name: postgres-init
```
This fixes the volume indentation and ensures that the `init` volume and
its mount use the same name.

Important: This learning configuration has no persistent volume. If the
Pod is recreated with a fresh filesystem, database data may be lost. For
a more durable deployment, add a PersistentVolumeClaim and mount it at
`/var/lib/postgresql/data`.

### 1.10 Correct the PostgreSQL Service

File: `postgres-service.yml`
```yml
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres
      namespace: devboard
    spec:
      type: ClusterIP
      selector:
        app: devboard-postgres
      ports:
        - name: postgres
          protocol: TCP
          port: 5432
          targetPort: 5432
```
The Service name `postgres` is important: it is the hostname used in
your backend's PostgreSQL connection string.

### 1.11 Correct the backend Deployment

File: `backend-deployment.yml`
```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: devboard-backend-deployment
      namespace: devboard
      labels:
        app: devboard-backend
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: devboard-backend
      template:
        metadata:
          labels:
            app: devboard-backend
        spec:
          containers:
            - name: devboard-backend
              image: anandkumarprajap/devboard-backend:latest
              ports:
                - name: http
                  containerPort: 8080
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    configMapKeyRef:
                      name: devboard-configmap
                      key: POSTGRES_USER
                - name: POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: devboard-secrets
                      key: POSTGRES_PASSWORD
                - name: POSTGRES_DB
                  valueFrom:
                    secretKeyRef:
                      name: devboard-secrets
                      key: POSTGRES_DB
                - name: POSTGRES_URL
                  value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/$(POSTGRES_DB)?sslmode=disable"
                - name: PORT
                  value: "8080"
```
Kubernetes expands the earlier environment variables in `POSTGRES_URL`.
The backend application must actually read `POSTGRES_URL` and listen on
port `8080` for this configuration to work.

### 1.12 Correct the backend Service

File: `backend-service.yml`
```yml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend
      namespace: devboard
    spec:
      type: ClusterIP
      selector:
        app: devboard-backend
      ports:
        - name: http
          protocol: TCP
          port: 8080
          targetPort: 8080
```
The backend's stable internal address is:

`http://backend:8080`

This DNS name works for clients running inside the `devboard` namespace,
not directly in a user's browser outside the cluster.

### 1.13 Correct the Kind cluster configuration

File: `kind-config.yml`
```yml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    name: tws-cluster

    nodes:
      - role: control-plane
        image: kindest/node:v1.35.0
        extraPortMappings:
          - containerPort: 30080
            hostPort: 8080
            protocol: TCP

      - role: worker
        image: kindest/node:v1.35.0

      - role: worker
        image: kindest/node:v1.35.0
```
Your cluster has one control-plane node and two worker nodes. The port
mapping connects your computer's port `8080` to port `30080` on the Kind
control-plane node.

This mapping is designed for the frontend Service's NodePort. If a
cluster already exists, editing this file does not automatically change
that running cluster's node configuration.

## Step 2 --- Commands to create the Kind cluster and deploy the application

### 2.1 Check the files and tools

Run these commands from your repository root:

    cd ~/GitHub-Actions-Revision

    ls -l k8s/

    docker --version
    kind version
    kubectl version --client

Check the YAML syntax and structure before applying:

    kubectl apply --dry-run=client -f k8s/namespace.yml
    kubectl apply --dry-run=client -f k8s/configMap.yml
    kubectl apply --dry-run=client -f k8s/secrets.yml
    kubectl apply --dry-run=client -f k8s/postgres-init.yml

### 2.2 Create the Kind cluster

    kind create cluster --config k8s/kind-config.yml

Verify the cluster:

    kind get clusters
    kubectl cluster-info
    kubectl get nodes -o wide

Expected cluster name: `tws-cluster`.

Expected nodes: one control-plane and two workers. If `tws-cluster`
already exists, do not create it again; check its current configuration
and proceed with the existing cluster if it is suitable.

### 2.3 Deploy in dependency order

Run each command from the repository root.

1\. Namespace, configuration and secrets

    kubectl apply -f k8s/namespace.yml
    kubectl apply -f k8s/configMap.yml
    kubectl apply -f k8s/secrets.yml
    kubectl apply -f k8s/postgres-init.yml

2\. PostgreSQL database

    kubectl apply -f k8s/postgres-deployment.yml
    kubectl apply -f k8s/postgres-service.yml
    kubectl rollout status deployment/postgres-deployment -n devboard

3\. Backend API

    kubectl apply -f k8s/backend-deployment.yml
    kubectl apply -f k8s/backend-service.yml
    kubectl rollout status deployment/devboard-backend-deployment -n devboard

4\. Frontend application

    kubectl apply -f k8s/frontend-deployment.yml
    kubectl apply -f k8s/frontend-service.yml
    kubectl rollout status deployment/devboard-frontend-deployment -n devboard

Wait for the database before deploying the backend because the backend
may attempt to connect as soon as its container starts. If your backend
implements retry logic, it can tolerate the database becoming ready
later.

Alternative: After verifying the dependencies, you can apply the entire
directory with `kubectl apply -f k8s/`, but the directory's filename
ordering is not a substitute for application readiness checks.

### 2.4 Open DevBoard in your browser

Once the frontend Pods are ready, open:

<http://localhost:8080>

Your request travels through:

`localhost:8080 → Kind control-plane:30080 → frontend Service:8080 → frontend Pod:4173`

The NodePort Service selects whichever frontend Pods are available. You
do not need to create a separate Service for each frontend replica.

Frontend-to-backend warning: The browser cannot resolve the Kubernetes
hostname `backend`. If your React/Vite application calls the API,
configure a browser-reachable API URL or a reverse proxy. One convenient
design is for the frontend web server to proxy `/api` requests to
`http://backend:8080` from inside the cluster, with the frontend using
relative `/api/...` URLs. The proxy must actually be configured in the
frontend image; a Kubernetes Service alone does not create that proxy.

## Step 3 --- Verification and troubleshooting commands

### 3.1 Check the full application

    kubectl get all -n devboard
    kubectl get pods -n devboard -o wide
    kubectl get deployments -n devboard
    kubectl get services -n devboard
    kubectl get endpointslices -n devboard

You should see approximately:

  Component               Desired Pods
  ----------------------- --------------
  Frontend Deployment     5
  Backend Deployment      2
  PostgreSQL Deployment   1

A Deployment's desired replica count does not guarantee all Pods are
ready. Check the `READY` column and rollout status.

### 3.2 Verify frontend connectivity

    kubectl get svc devboard-frontend-service -n devboard
    kubectl get pods -n devboard -l app=devboard-frontend -o wide
    kubectl logs deployment/devboard-frontend-deployment -n devboard
    curl -I http://localhost:8080

If `curl` fails, inspect the Service, node port mapping, Pod readiness,
and container logs.

### 3.3 Verify backend connectivity

    kubectl get svc backend -n devboard

    kubectl logs deployment/devboard-backend-deployment -n devboard

    kubectl describe deployment devboard-backend-deployment -n devboard

Check the backend's health endpoint if your image implements one:

    kubectl port-forward svc/backend 8081:8080 -n devboard

In a second terminal, try:

    curl http://localhost:8081/health

This port-forward exposes the backend to your local machine temporarily.
It does not configure the frontend's browser API URL.

### 3.4 Verify PostgreSQL

    kubectl get pods -n devboard -l app=devboard-postgres

    kubectl logs deployment/postgres-deployment -n devboard

    kubectl exec -it deployment/postgres-deployment -n devboard -- \
      sh -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'

Check the schema and seed data:

    kubectl exec -it deployment/postgres-deployment -n devboard -- \
      sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "\dt"'

    kubectl exec -it deployment/postgres-deployment -n devboard -- \
      sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT id, name FROM projects;"'

Expected project names include `DevBoard MVP` and `Marketing Site` when
the initialization scripts ran successfully.

### 3.5 Common errors and fixes

  ---------------------------------------------------------------------------------
  Problem                        What to check
  ------------------------------ --------------------------------------------------
  `ImagePullBackOff`             Image name, Docker Hub visibility, tag, registry
                                 credentials

  `CrashLoopBackOff`             `kubectl logs` and `kubectl logs --previous`

  `CreateContainerConfigError`   ConfigMap/Secret names and key spelling

  Backend cannot connect to DB   `POSTGRES_URL`, PostgreSQL readiness, Service
                                 `postgres:5432`

  Frontend shows but data is     Browser API URL, CORS, reverse proxy, backend
  missing                        health

  `Connection refused`           Container listening port and Service `targetPort`

  `NodePort` inaccessible        Kind host mapping, NodePort, Service endpoints

  SQL tables missing             Initialization logs and whether the database data
                                 directory was already initialized

  Pods keep restarting           `kubectl describe pod`, events, resource limits,
                                 application logs
  ---------------------------------------------------------------------------------

Useful diagnostic commands:

    kubectl get events -n devboard --sort-by=.metadata.creationTimestamp

    kubectl describe pod <pod-name> -n devboard

    kubectl logs <pod-name> -n devboard

    kubectl logs <pod-name> -n devboard --previous

Replace `<pod-name>` with the actual Pod name returned by
`kubectl get pods`.







## 🛠️ Step 4: Component Connectivity Matrix

The core architecture of Kubernetes relies completely on **decoupled references, labels, selectors, and internal DNS names**—not physical file linking. Primitives declare dependencies dynamically at runtime.

### 4.1 Topology & Resource Ownership Map

All workloads reside securely inside an isolated operational boundary:

```mermaid
graph TD
    %% Core Styling
    classDef ns fill:#f8f9fa,stroke:#343a40,stroke-width:2px,stroke-dasharray: 5 5;
    classDef config fill:#fff3cd,stroke:#ffc107,stroke-width:1px;
    classDef deploy fill:#cce5ff,stroke:#004085,stroke-width:1px;
    classDef pod fill:#e2e3e5,stroke:#383d41,stroke-width:1px;
    classDef svc fill:#d4edda,stroke:#155724,stroke-width:1px;

    %% Namespace Boundary
    subgraph Devboard_NS [☸️ Namespace: devboard]
        
        %% Config & Secret Layer
        CM[📄 ConfigMap: devboard-configmap<br/>• POSTGRES_USER] --- CM_Inj(Env Injection)
        SEC[🔏 Secret: devboard-secrets<br/>• POSTGRES_PASSWORD<br/>• POSTGRES_DB] --- SEC_Inj(Env Injection)

        %% Services Network Layer
        Svc_FE[🔌 Service: frontend-svc] -.->|Routes to| Pod_FE
        Svc_BE[🔌 Service: backend-svc] -.->|Routes to| Pod_BE
        Svc_DB[🔌 Service: postgres-svc] -.->|Routes to| Pod_DB

        %% Workload Topology
        CM_Inj & SEC_Inj --> Deploy_BE[🚀 Backend Deployment]
        CM_Inj & SEC_Inj --> Deploy_DB[🚀 PostgreSQL Deployment]
        
        Deploy_FE[🚀 Frontend Deployment] -->|Spawns| Pod_FE(📦 5x Frontend Pods)
        Deploy_BE -->|Spawns| Pod_BE(📦 2x Backend Pods)
        Deploy_DB -->|Spawns| Pod_DB(📦 1x Database Pod)

    end

    %% Class Application
    class Devboard_NS ns;
    class CM,SEC config;
    class Deploy_FE,Deploy_BE,Deploy_DB deploy;
    class Pod_FE,Pod_BE,Pod_DB pod;
    class Svc_FE,Svc_BE,Svc_DB svc;
```

---

### 📋 Resource Reference breakdown

| Resource Component | Identifier Name | Injected Variables / Responsibilities |
| :--- | :--- | :--- |
| **☸️ Namespace** | `devboard` | Enforces the global structural isolation boundary for all application elements. |
| **📄 ConfigMap** | `devboard-configmap` | Injects non-sensitive configuration keys like `POSTGRES_USER` directly into runtimes. |
| **🔏 Secret** | `devboard-secrets` | Mounts base64 encrypted data entries: `POSTGRES_PASSWORD` and `POSTGRES_DB`. |
| **🚀 Frontend Deployment** | `frontend-deployment` | Maintains exactly **5 replicas** running user-interface presentation instances. |
| **🚀 Backend Deployment** | `backend-deployment` | Controls **2 replicas** executing calculations; consumes variables from configs and secrets. |
| **🚀 PostgreSQL Deployment** | `postgres-deployment` | Anchors exactly **1 replica** running core state storage; relies on configuration maps. |

> 📌 **Architectural Reality:** Deployments manage the operational lifecycle of your Pods. Services expose consistent internal IP structures by mapping labels onto running pods. Actual application layer traffic flows over the **frontend → backend → database** network route, detailed next in Step 5.








### 4.2 Which YAML field connects to which file?

  ------------------------------------------------------------------------------------
  Resource            Reference or matching field      What it connects
  ------------------- -------------------------------- -------------------------------
  All resources       `metadata.namespace: devboard`   Namespace

  Frontend Deployment `template.metadata.labels.app`   Frontend Pods

  Frontend Service    `spec.selector.app`              Frontend Pods

  Backend Deployment  `template.metadata.labels.app`   Backend Pods

  Backend Service     `spec.selector.app`              Backend Pods

  PostgreSQL          `template.metadata.labels.app`   Database Pod
  Deployment                                           

  PostgreSQL Service  `spec.selector.app`              Database Pod

  Backend and         `configMapKeyRef.name`           `devboard-configmap`
  PostgreSQL                                           

  Backend and         `secretKeyRef.name`              `devboard-secrets`
  PostgreSQL                                           

  PostgreSQL          `volumes.configMap.name`         `postgres-init`
  Deployment                                           

  PostgreSQL          `volumeMounts.mountPath`         `/docker-entrypoint-initdb.d`
  container                                            

  Backend             `POSTGRES_URL` hostname          `postgres` Service
  ------------------------------------------------------------------------------------

### 4.3 How the frontend Deployment connects to its Service

The frontend Deployment creates Pods with this label:

    template:
      metadata:
        labels:
          app: devboard-frontend

The frontend Service selects the same label:

    selector:
      app: devboard-frontend

Kubernetes finds the matching Pods and maintains the Service's endpoints
as Pods become ready or unavailable.

If the label differs --- for example, `app: frontend` in the Service
while the Pods use `app: devboard-frontend` --- the Service will not
select those Pods.

### 4.4 How the backend connects to PostgreSQL

Your backend reads configuration from the ConfigMap and Secret, then
uses this connection string:

    postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable

This example contains the learning credentials configured above; do not
use them in production.

Breakdown:

-   `devboard:devboard` --- database username and password.
-   `postgres` --- Kubernetes Service name.
-   `5432` --- PostgreSQL Service port.
-   `/devboard` --- database name.
-   `sslmode=disable` --- disables TLS for this local lab connection.

The backend connects to the Service DNS name, not a Pod IP. This remains
stable when Kubernetes replaces the database Pod.

### 4.5 How `postgres-init.yml` connects to the database

The PostgreSQL Deployment contains two references to the same volume:

    volumeMounts:
      - name: init
        mountPath: /docker-entrypoint-initdb.d
        readOnly: true

    volumes:
      - name: init
        configMap:
          name: postgres-init

The first reference mounts the volume inside the container. The second
defines its source as the `postgres-init` ConfigMap. The PostgreSQL
image then processes the SQL files during first-time database
initialization.

## Step 5 --- Architecture diagram: ports, Services, worker nodes and Pod connections

This diagram is ready to include in your `README.md`.

### 5.1 DevBoard application architecture

``` mermaid
flowchart TB
    Browser["User Browser<br/>localhost:8080"]
    CP["Kind Control Plane<br/>hostPort 8080 → nodePort 30080"]

    subgraph Cluster["Kind Cluster: tws-cluster"]
      subgraph NS["Namespace: devboard"]
        FS["Frontend Service<br/>NodePort 30080<br/>Service port 8080"]
        BS["Backend Service: backend<br/>ClusterIP port 8080"]
        DS["PostgreSQL Service: postgres<br/>ClusterIP port 5432"]

        subgraph W1["Worker Node 1"]
          F1["Frontend Pod 1<br/>containerPort 4173"]
          F2["Frontend Pod 2<br/>containerPort 4173"]
          B1["Backend Pod 1<br/>containerPort 8080"]
          DB["PostgreSQL Pod<br/>containerPort 5432"]
        end

        subgraph W2["Worker Node 2"]
          F3["Frontend Pod 3<br/>containerPort 4173"]
          F4["Frontend Pod 4<br/>containerPort 4173"]
          F5["Frontend Pod 5<br/>containerPort 4173"]
          B2["Backend Pod 2<br/>containerPort 8080"]
        end

        CM["ConfigMap<br/>devboard-configmap"]
        SEC["Secret<br/>devboard-secrets"]
        INIT["ConfigMap<br/>postgres-init<br/>SQL initialization"]
      end
    end

    Browser --> CP
    CP --> FS
    FS --> F1
    FS --> F2
    FS --> F3
    FS --> F4
    FS --> F5

    F1 -. "API calls via configured proxy" .-> BS
    F3 -. "API calls via configured proxy" .-> BS
    BS --> B1
    BS --> B2
    B1 -->|"postgres:5432"| DS
    B2 -->|"postgres:5432"| DS
    DS --> DB

    CM -. "POSTGRES_USER" .-> B1
    CM -. "POSTGRES_USER" .-> B2
    CM -. "POSTGRES_USER" .-> DB
    SEC -. "DB credentials" .-> B1
    SEC -. "DB credentials" .-> B2
    SEC -. "DB credentials" .-> DB
    INIT -. "Mounted SQL files" .-> DB
```

Note: The Pod distribution is illustrative, not guaranteed. Kubernetes
schedules Pods on available nodes, and Services select Pods by labels.
The frontend-to-backend connection requires an actual configured proxy
or another browser-accessible API endpoint.

### 5.2 Port mapping notes

  --------------------------------------------------------------------------
  Layer               Port      Meaning
  ------------------- --------- --------------------------------------------
  Your browser        `8080`    Host port on your computer

  Kind control plane  `30080`   NodePort reached through the Kind port
                                mapping

  Frontend Service    `8080`    Service port

  Frontend container  `4173`    Vite preview/web server

  Backend Service     `8080`    Internal API Service port

  Backend container   `8080`    API listening port

  PostgreSQL Service  `5432`    Internal database port

  PostgreSQL          `5432`    Database listening port
  container                     
  --------------------------------------------------------------------------

Remember: A Service port, a NodePort, and a container port are different
things. Kubernetes routes between them; they do not all need to have the
same number.

### 5.3 Check which worker node runs each Pod

    kubectl get pods -n devboard -o wide

The `NODE` column shows the actual placement. For example, a frontend
Pod may run on one worker while another runs on the second worker.
Kubernetes does not guarantee an even distribution unless you configure
scheduling rules.

To inspect a specific Pod:

    kubectl describe pod <pod-name> -n devboard

To view the nodes:

    kubectl get nodes -o wide

### 5.4 Final deployment checklist

## Deployment checklist

-   [ ] Kind cluster is running
-   [ ] Namespace and configuration exist
-   [ ] PostgreSQL is ready and initialized
-   [ ] Backend Deployment has two ready replicas
-   [ ] Frontend Deployment has five ready replicas
-   [ ] Services have matching endpoints
-   [ ] Frontend responds on `localhost:8080`

Copy verification commands:

``` bash
kubectl get nodes
kubectl get namespace devboard
kubectl get configmap,secret -n devboard
kubectl get pods -n devboard -o wide
kubectl get svc,endpointslices -n devboard
curl -I http://localhost:8080
```

## Final summary

-   Namespace groups your resources.
-   Deployments maintain the frontend, backend, and database Pods.
-   Services provide stable endpoints by selecting Pods through labels.
-   ConfigMaps store non-sensitive settings.
-   Secrets supply database credentials.
-   `postgres-init.yml` provides SQL scripts to initialize a new
    database.
-   Kind's port mapping and the frontend NodePort expose the frontend to
    your local browser.
-   The frontend must be explicitly configured to reach the backend API;
    internal Kubernetes DNS names are not directly accessible to the
    browser.

These notes provide a foundation for your DevBoard Kubernetes lab.
Before treating the deployment as production-ready, add database
persistence, stronger Secret management, health probes, resource
requests/limits, and appropriate network and access controls.


# ☸️ K8s Cluster Application Orchestrator

> A production-grade Kubernetes orchestration layer deploying highly resilient, self-healing microservice meshes with zero-downtime rolling updates.

---

## 🗺️ System Topology & Deployment Flow

```mermaid
graph TD
    %% Define Node Colors and Styles
    classDef cluster fill:#f9f9f9,stroke:#333,stroke-width:2px;
    classDef pod fill:#e1f5fe,stroke:#0288d1,stroke-width:1px;
    classDef net fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px;
    classDef data fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;

    %% CI/CD Flow
    subgraph GitOps_Pipeline [🔄 GitHub Actions & ArgoCD]
        Code[💻 Git Commit] -->|Triggers| CI[⚙️ CI Build & Test]
        CI -->|Pushes Image| CR[🐳 Container Registry]
        CR -->|Sync Manifest| Argo[🐙 ArgoCD Controller]
    end

    %% K8s Cluster Core Infrastructure
    subgraph K8s_Cluster [☸️ Kubernetes Cluster Namespace]
        Argo -->|Deploys| Ingress
        
        Ingress[🌐 Nginx Ingress Controller] -->|TLS Routing| Svc[🔌 K8s ClusterIP Service]
        
        subgraph Replica_Set [📦 Deployment Pod Mesh]
            Svc -->|Load Balance| Pod1[📦 App Pod v2 - Active]
            Svc -->|Rolling Update| Pod2[📦 App Pod v2 - Active]
            Svc -.->|Terminating| PodOld[📦 App Pod v1 - Draining]
        end

        Pod1 -->|Mount| PVC[💾 Persistent Volume Claim]
        Pod2 -->|Mount| PVC
    end

    %% State and Monitoring Layers
    PVC -->|Read/Write| DB[(🛢️ Distributed State Store)]
    Pod1 -.->|Telemetry Scrape| Prom[📊 Prometheus Monitor]
    Pod2 -.->|Telemetry Scrape| Prom

    %% Assign classes for styling
    class K8s_Cluster cluster;
    class Pod1,Pod2,PodOld pod;
    class Ingress,Svc net;
    class PVC,DB data;
```


# Explain 

> **Project setup covered here:** Kind cluster `tws-cluster`; namespace `devboard`; PostgreSQL `postgres:16-alpine`; Go backend with 2 replicas on port `8080`; frontend with 5 replicas on port `4173`; frontend NodePort `30080`, mapped to host port `8080`.

## 1. Project structure

```text
GitHub-Actions-Revision/
└── k8s/
    ├── kind-config.yml
    ├── namespace.yml
    ├── configMap.yml
    ├── secrets.yml
    ├── postgres-init.yml
    ├── postgres-deployment.yml
    ├── postgres-service.yml
    ├── backend-deployment.yml
    ├── backend-service.yml
    ├── frontend-deployment.yml
    ├── frontend-service.yml
    └── README.md
```

| File | Responsibility |
|---|---|
| `kind-config.yml` | Configures the local Kind nodes and host-to-node port mapping. |
| `namespace.yml` | Creates the `devboard` namespace. |
| `configMap.yml` | Holds non-sensitive database configuration. |
| `secrets.yml` | Holds database credentials. |
| `postgres-init.yml` | Provides SQL scripts for initial database setup. |
| `postgres-deployment.yml` | Runs the PostgreSQL container. |
| `postgres-service.yml` | Gives PostgreSQL stable in-cluster DNS: `postgres`. |
| `backend-deployment.yml` | Runs the Go API and supplies its database connection settings. |
| `backend-service.yml` | Gives the API stable in-cluster DNS: `backend`. |
| `frontend-deployment.yml` | Runs five frontend replicas. |
| `frontend-service.yml` | Exposes the frontend using NodePort `30080`. |

## 2. Important Kubernetes YAML fields

Most Kubernetes resource manifests share these fields:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
  namespace: devboard
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
        - name: example
          image: example-image:latest
```

- `apiVersion`: Kubernetes API group/version used for the resource.
- `kind`: resource type, such as `Deployment`, `Service`, `ConfigMap`, or `Secret`.
- `metadata.name`: resource name.
- `metadata.namespace`: namespace in which the resource lives. If omitted, `kubectl` normally uses the `default` namespace.
- `spec.replicas`: desired number of Pod replicas for a Deployment.
- `selector.matchLabels`: tells the Deployment which Pods it manages.
- `template.metadata.labels`: labels applied to Pods. These must match the Deployment selector.
- `containers.image`: container image to run.
- `ports.containerPort`: documents the port the application listens on inside the container; it does not by itself expose the app outside the cluster.

## 3. `kind-config.yml` — create the local cluster configuration

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tws-cluster
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
  - role: worker
  - role: worker
```

### What the code means

- `kind: Cluster`: this file describes a Kind cluster.
- `apiVersion`: the Kind configuration schema version.
- `name: tws-cluster`: gives the cluster its name.
- `role: control-plane`: the node that runs the Kubernetes control-plane components.
- `role: worker`: adds two worker nodes where application Pods can be scheduled.
- `extraPortMappings`: forwards a port from the computer running Kind to a port on the Kind node.
- `containerPort: 30080`: the NodePort on the Kind node.
- `hostPort: 8080`: the port you use from your local browser.
- `protocol: TCP`: uses TCP networking.

**Create the cluster:**

```bash
cd GitHub-Actions-Revision/k8s
kind create cluster --config kind-config.yml
kubectl cluster-info --context kind-tws-cluster
kubectl get nodes -o wide
```

`kind-config.yml` is not applied with `kubectl apply`. Kind reads it when creating the cluster. If the cluster already exists, editing this file does not automatically rebuild or reconfigure it.

## 4. `namespace.yml` — isolate project resources

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devboard
```

- `apiVersion: v1`: core Kubernetes API.
- `kind: Namespace`: creates a logical boundary for resources.
- `metadata.name: devboard`: names the namespace.

Create it and set a convenient default namespace for the current context:

```bash
kubectl apply -f namespace.yml
kubectl config set-context --current --namespace=devboard
kubectl get namespace devboard
```

The namespace helps keep DevBoard resources grouped together. It is not a security boundary by itself.

## 5. `configMap.yml` — non-sensitive configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devboard-configmap
  namespace: devboard
data:
  POSTGRES_USER: "devboard"
```

- `kind: ConfigMap`: stores ordinary configuration values.
- `metadata.name`: must match the name referenced by the Deployments.
- `data`: contains string key/value pairs.
- `POSTGRES_USER`: database username supplied to PostgreSQL.

Apply and inspect:

```bash
kubectl apply -f configMap.yml
kubectl get configmap devboard-configmap -n devboard
kubectl describe configmap devboard-configmap -n devboard
```

Do not store passwords, API keys, or other sensitive values in a ConfigMap.

## 6. `secrets.yml` — database credentials

A manifest using `stringData` can be written like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devboard-secrets
  namespace: devboard
type: Opaque
stringData:
  POSTGRES_PASSWORD: "devboard"
  POSTGRES_DB: "devboard"
```

- `kind: Secret`: stores sensitive configuration for use by Pods.
- `type: Opaque`: a general-purpose Secret type. The capitalization matters.
- `stringData`: lets you enter plain strings; Kubernetes converts them for storage.
- `POSTGRES_PASSWORD`: password passed to PostgreSQL.
- `POSTGRES_DB`: database name created by the PostgreSQL image on first initialization.

Apply and verify the Secret without printing its values:

```bash
kubectl apply -f secrets.yml
kubectl get secret devboard-secrets -n devboard
```

**Security note:** Kubernetes Secret values are not automatically encrypted just because they are stored in a Secret. This example password is suitable only for a disposable learning lab. Use a strong credential and appropriate cluster access controls for real deployments. Avoid committing real credentials to Git.

## 7. `postgres-init.yml` — initial SQL schema

The SQL initialization file is supplied to the PostgreSQL container using a ConfigMap. Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: devboard
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS projects (
      id SERIAL PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT DEFAULT '',
      created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS tasks (
      id SERIAL PRIMARY KEY,
      project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
      title TEXT NOT NULL,
      status TEXT NOT NULL DEFAULT 'todo',
      created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );
```

- `kind: ConfigMap`: packages the SQL file as configuration data.
- `metadata.name: postgres-init`: this exact name must match the ConfigMap volume reference in `postgres-deployment.yml`.
- `data.init.sql`: creates a file named `init.sql` inside the mounted directory.
- `CREATE TABLE IF NOT EXISTS`: creates a table only if it does not already exist.
- `projects`: stores project records.
- `tasks`: stores tasks and can link each task to a project using `project_id`.
- `ON DELETE CASCADE`: removes related tasks when their referenced project is deleted.

Apply and inspect:

```bash
kubectl apply -f postgres-init.yml
kubectl get configmap postgres-init -n devboard
```

**Important:** the official PostgreSQL container runs scripts in `/docker-entrypoint-initdb.d` only when initializing an empty database data directory. Updating this ConfigMap later does not automatically rerun the SQL against an already-initialized database. If tables are missing, check the PostgreSQL logs and database state before deciding how to reinitialize it; deleting database storage can permanently delete data.

## 8. `postgres-deployment.yml` — run PostgreSQL

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: devboard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devboard-postgres
  template:
    metadata:
      labels:
        app: devboard-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_DB
          volumeMounts:
            - name: postgres-init-volume
              mountPath: /docker-entrypoint-initdb.d
              readOnly: true
      volumes:
        - name: postgres-init-volume
          configMap:
            name: postgres-init
```

### Key sections

- `kind: Deployment`: Kubernetes keeps the requested number of PostgreSQL Pods running.
- `replicas: 1`: requests one database Pod. A single replica is suitable for a lab, not a highly available production database.
- `selector.matchLabels` and `template.metadata.labels`: must match so the Deployment manages the intended Pod.
- `image: postgres:16-alpine`: uses PostgreSQL 16 on Alpine Linux.
- `containerPort: 5432`: PostgreSQL's default listening port.
- `valueFrom.configMapKeyRef`: loads the username from `devboard-configmap`.
- `valueFrom.secretKeyRef`: loads the password and database name from `devboard-secrets`.
- `volumeMounts`: makes the SQL initialization files available inside the container.
- `mountPath: /docker-entrypoint-initdb.d`: special directory scanned by the official PostgreSQL image during first-time initialization.
- `readOnly: true`: the container can read, but not modify, the mounted initialization files.
- `volumes.configMap.name: postgres-init`: must match the ConfigMap name in `postgres-init.yml`.

Apply and check:

```bash
kubectl apply -f postgres-deployment.yml
kubectl get pods -n devboard -o wide
kubectl logs deployment/postgres-deployment -n devboard
```

**Data persistence:** this example does not define a PersistentVolumeClaim. Without persistent storage, data can be lost when the Pod is replaced. For anything beyond a temporary lab, configure persistent storage and a database backup plan.

## 9. `postgres-service.yml` — stable database DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: devboard
spec:
  selector:
    app: devboard-postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
  type: ClusterIP
```

- `kind: Service`: creates a stable network endpoint for the matching Pods.
- `metadata.name: postgres`: provides the in-cluster DNS name `postgres`.
- `selector`: must match the database Pod label `app: devboard-postgres`.
- `port: 5432`: Service port clients use.
- `targetPort: 5432`: port on the PostgreSQL container.
- `type: ClusterIP`: exposes the Service inside the cluster, not directly to your laptop.

Other Pods in the same namespace can connect to `postgres:5432`. The backend's database connection uses the Service name, not a temporary Pod IP.

## 10. `backend-deployment.yml` — run the Go API

The backend Deployment uses two replicas and obtains its database connection settings from Kubernetes configuration. Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: devboard
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devboard-backend
  template:
    metadata:
      labels:
        app: devboard-backend
    spec:
      containers:
        - name: backend
          image: anandkumarprajap/devboard-backend:latest
          ports:
            - containerPort: 8080
          env:
            - name: POSTGRES_URL
              value: "postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable"
```

### Key sections

- `replicas: 2`: requests two backend Pods for basic replica capacity.
- `image`: the container image for the Go API.
- `containerPort: 8080`: the API's container port.
- `POSTGRES_URL`: connection string containing the database username, password, hostname, port, database name, and SSL mode.
- `postgres`: the hostname resolves through the PostgreSQL Service inside the cluster.

The example connection string matches the sample lab credentials above. If you change the username, password, or database name, update the backend connection configuration too. For a safer setup, construct the connection string from Secret-backed environment variables rather than embedding a password in the Deployment YAML.

Apply and check:

```bash
kubectl apply -f backend-deployment.yml
kubectl get pods -n devboard -l app=devboard-backend -o wide
kubectl logs deployment/backend-deployment -n devboard
```

## 11. `backend-service.yml` — stable backend DNS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: devboard
spec:
  selector:
    app: devboard-backend
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

- `metadata.name: backend`: creates the stable in-cluster hostname `backend`.
- `selector`: matches the labels on backend Pods.
- `port: 8080`: port other Pods use to reach the backend Service.
- `targetPort: 8080`: port on the backend containers.
- `ClusterIP`: makes the API available inside Kubernetes.

The frontend container can reach `http://backend:8080` **from inside the cluster** if its requests are made by a server-side proxy. A browser running on your laptop cannot normally resolve the Kubernetes-only hostname `backend`. If frontend JavaScript calls `http://backend:8080` directly from the browser, configure an accessible API URL or route requests through a frontend web-server reverse proxy.

## 12. `frontend-deployment.yml` — run five frontend replicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: devboard
spec:
  replicas: 5
  selector:
    matchLabels:
      app: devboard-frontend
  template:
    metadata:
      labels:
        app: devboard-frontend
    spec:
      containers:
        - name: frontend
          image: anandkumarprajap/devboard-frontend:latest
          ports:
            - containerPort: 4173
```

- `replicas: 5`: requests five frontend Pods.
- `image`: the published DevBoard frontend image.
- `containerPort: 4173`: the port the frontend server listens on in the container.
- The Service selects the Pods using `app: devboard-frontend`.

A Deployment manages the desired number of replicas and can replace Pods that fail. Five replicas do not automatically guarantee high availability; node capacity, readiness checks, and application behavior also matter.

Apply and check:

```bash
kubectl apply -f frontend-deployment.yml
kubectl get pods -n devboard -l app=devboard-frontend -o wide
kubectl rollout status deployment/frontend-deployment -n devboard
```

## 13. `frontend-service.yml` — expose the frontend with NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: devboard
spec:
  type: NodePort
  selector:
    app: devboard-frontend
  ports:
    - name: http
      port: 8080
      targetPort: 4173
      nodePort: 30080
```

- `type: NodePort`: exposes the Service on a port on each Kubernetes node.
- `selector`: sends traffic to frontend Pods with the matching label.
- `port: 8080`: Service port inside the cluster.
- `targetPort: 4173`: frontend container port.
- `nodePort: 30080`: port exposed on the Kind node. The standard Kubernetes NodePort range is usually `30000–32767`.
- The Kind configuration maps local host port `8080` to node port `30080`.

Open the frontend at:

```text
http://localhost:8080
```

This relies on the cluster being created with the `extraPortMappings` shown in `kind-config.yml` and the frontend Service being healthy.

## 14. How the files connect

```mermaid
flowchart TD
    A["kind-config.yml"] --> B["Kind cluster: tws-cluster"]
    B --> C["namespace.yml: devboard"]
    C --> D["ConfigMap: devboard-configmap"]
    C --> E["Secret: devboard-secrets"]
    C --> F["ConfigMap: postgres-init"]
    D --> G["PostgreSQL Deployment"]
    E --> G
    F --> G
    G --> H["PostgreSQL Pod :5432"]
    H --> I["Service: postgres :5432"]
    I --> J["Backend Deployment: 2 replicas"]
    J --> K["Backend Pods :8080"]
    K --> L["Service: backend :8080"]
    L --> M["Frontend Pods :4173"]
    M --> N["Frontend Service: 8080 / NodePort 30080"]
    N --> O["Host browser: http://localhost:8080"]
```

The application request path is:

1. Your browser requests `http://localhost:8080`.
2. Kind forwards host port `8080` to node port `30080`.
3. The frontend NodePort Service sends traffic to a frontend Pod on port `4173`.
4. The frontend must send API requests to the backend using a valid in-cluster route (for example, a server-side proxy to `http://backend:8080`) or a browser-accessible API address.
5. The backend connects to the database at `postgres:5432`.
6. The PostgreSQL Service forwards traffic to the PostgreSQL Pod.

### Port reference

| Component | Port | Meaning |
|---|---:|---|
| Local computer | `8080` | Browser-facing host port |
| Kind node | `30080` | NodePort reached through Kind's port mapping |
| Frontend Service | `8080` | Kubernetes Service port |
| Frontend container | `4173` | Frontend server port |
| Backend Service/container | `8080` | Go API port |
| PostgreSQL Service/container | `5432` | Database port |

## 15. Apply manifests in order

Run commands from the `k8s/` directory. Create the Kind cluster first; do not apply `kind-config.yml` with `kubectl`.

```bash
# 1. Create the cluster (skip if it already exists)
kind create cluster --config kind-config.yml

# 2. Create the namespace
kubectl apply -f namespace.yml

# 3. Create configuration and database initialization resources
kubectl apply -f configMap.yml
kubectl apply -f secrets.yml
kubectl apply -f postgres-init.yml

# 4. Create PostgreSQL and its Service
kubectl apply -f postgres-deployment.yml
kubectl apply -f postgres-service.yml

# 5. Create backend and its Service
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml

# 6. Create frontend and its Service
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml

# 7. Check all resources
kubectl get all -n devboard
```

You can also apply all resource manifests in the directory after the cluster exists and the files are correct:

```bash
kubectl apply -f .
```

Applying a directory does not guarantee readiness order. If you use it, still verify that PostgreSQL is ready before debugging backend database errors.

## 16. Verification commands

```bash
# Cluster and nodes
kind get clusters
kubectl get nodes -o wide

# All DevBoard resources
kubectl get all -n devboard

# Pods, labels, IP addresses and node placement
kubectl get pods -n devboard -o wide --show-labels

# Services and NodePort
kubectl get services -n devboard

# Deployments and rollout status
kubectl get deployments -n devboard
kubectl rollout status deployment/postgres-deployment -n devboard
kubectl rollout status deployment/backend-deployment -n devboard
kubectl rollout status deployment/frontend-deployment -n devboard

# Logs
kubectl logs deployment/postgres-deployment -n devboard
kubectl logs deployment/backend-deployment -n devboard
kubectl logs deployment/frontend-deployment -n devboard

# Recent events
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp
```

Expected replica counts are one PostgreSQL Pod, two backend Pods, and five frontend Pods, provided the images can be pulled and all containers become healthy.

## 17. Troubleshooting

### Pods are `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff`

```bash
kubectl describe pod <pod-name> -n devboard
kubectl logs <pod-name> -n devboard
kubectl logs <pod-name> -n devboard --previous
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp
```

Look for image-pull errors, missing configuration keys, invalid image names, failed probes, resource pressure, or application startup errors.

### Service has no endpoints

```bash
kubectl get svc -n devboard
kubectl get endpoints -n devboard
kubectl get pods -n devboard --show-labels
```

A Service's `spec.selector` must match the Pod template labels. For example, the PostgreSQL Service selector `app: devboard-postgres` must match the labels on the PostgreSQL Pod.

### Backend cannot connect to PostgreSQL

```bash
kubectl logs deployment/backend-deployment -n devboard
kubectl logs deployment/postgres-deployment -n devboard
kubectl get svc postgres -n devboard
kubectl describe pod -l app=devboard-postgres -n devboard
```

Check that the database is ready, the hostname is `postgres`, port is `5432`, and username/password/database match the ConfigMap and Secret. Also verify that `postgres-init.yml` defines ConfigMap `postgres-init` and the Deployment mounts that same name. A name mismatch prevents the initialization ConfigMap from mounting.

### Frontend loads but API requests fail

Inspect the browser developer tools' Network tab and the frontend server configuration. Kubernetes DNS names such as `backend` resolve inside the cluster; they do not automatically resolve in a user's browser. Route API requests through a server-side reverse proxy or configure a browser-accessible API endpoint.

### Frontend does not open on `localhost:8080`

```bash
kubectl get svc frontend-service -n devboard
kubectl get pods -n devboard -l app=devboard-frontend -o wide
docker ps
```

Confirm the Service uses NodePort `30080`, the container port is `4173`, the Kind cluster was created with host `8080` mapped to node `30080`, and the frontend Pods are ready.

### SQL tables are missing

Review the PostgreSQL logs and confirm the `postgres-init` ConfigMap is mounted at `/docker-entrypoint-initdb.d`. The official image runs initialization scripts only on the first initialization of an empty data directory. Changing the ConfigMap after the database has initialized does not rerun scripts automatically.

## 18. Configuration corrections to check

Before deploying, confirm these names and values are consistent across the files:

- `namespace.yml` creates `devboard`; all application resources should use that namespace.
- The ConfigMap name is `devboard-configmap`; its key is `POSTGRES_USER`.
- The Secret name is `devboard-secrets`; its keys are `POSTGRES_PASSWORD` and `POSTGRES_DB`.
- The initialization ConfigMap name is `postgres-init`; the PostgreSQL volume reference must use exactly the same name.
- The PostgreSQL Service is named `postgres`, and the backend connection hostname must be `postgres`.
- The backend Service is named `backend`; any server-side frontend proxy must use that Service name.
- Backend labels and Service selector must match: `app: devboard-backend`.
- Frontend labels and Service selector must match: `app: devboard-frontend`.
- Frontend container port is `4173`; frontend NodePort is `30080`; Kind host mapping is `8080` to `30080`.
- Avoid duplicate YAML keys such as two `type:` entries in one Service manifest. YAML should define each key only once.

## 19. Clean up

> **Warning:** deleting the cluster removes the Kubernetes resources in that Kind cluster. Database data may be lost if it is stored only in ephemeral Pod storage.

```bash
kind delete cluster --name tws-cluster
```

## Summary

- **Kind** creates the local Kubernetes cluster and port mapping.
- **Namespace** groups DevBoard resources.
- **ConfigMap and Secret** supply configuration and credentials.
- **PostgreSQL Deployment + Service** run the database and provide stable DNS.
- **Backend Deployment + Service** run the Go API and provide stable DNS.
- **Frontend Deployment + NodePort Service** run five frontend replicas and expose the web UI at `http://localhost:8080`.
- **Labels and selectors** connect Services to the correct Pods.
- **SQL initialization** happens on first initialization of an empty PostgreSQL data directory, not every time the Pod restarts.
'''

out = Path("/mnt/data/README.md")
pypandoc.convert_text(readme, "md", format="md", outputfile=str(out), extra_args=["--standalone"])
print(f"Created {out} ({out.stat().st_size} bytes)")


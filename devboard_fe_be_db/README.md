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

    apiVersion: v1
    kind: Namespace
    metadata:
      name: devboard

Connection: All application resources specify `namespace: devboard`.
This allows their names to be resolved within the same namespace.

### 1.3 Do you need `pod.yml`?

No. You do not need a separate frontend `pod.yml` for this deployment.

Your `frontend-deployment.yml` already defines a Pod template:

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

The Deployment creates and maintains five Pods through a ReplicaSet. A
standalone Pod is useful for temporary debugging or simple experiments,
but it is not necessary for this application.

### 1.4 Correct the frontend Deployment

File: `frontend-deployment.yml`

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

Important connection: `spec.selector.matchLabels.app` must match
`spec.template.metadata.labels.app`. The Service also selects this same
label.

### 1.5 Correct the frontend Service

File: `frontend-service.yml`

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

    apiVersion: v1
    kind: Secret
    metadata:
      name: devboard-secrets
      namespace: devboard
    type: Opaque
    stringData:
      POSTGRES_PASSWORD: "devboard"
      POSTGRES_DB: "devboard"

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

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: devboard-configmap
      namespace: devboard
    data:
      POSTGRES_USER: "devboard"

Connection: The backend and PostgreSQL Deployment read `POSTGRES_USER`
from this ConfigMap.

### 1.8 Correct the PostgreSQL initialization ConfigMap

File: `postgres-init.yml`

Keep your existing SQL schema and seed statements, but use this YAML
structure:

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

This retains your two tables, indexes, update trigger, sample projects,
and task records. Keep the complete SQL in the actual file.

How initialization works: The PostgreSQL image runs `.sql` files mounted
under `/docker-entrypoint-initdb.d` when it initializes a new, empty
database data directory. These scripts do not automatically rerun every
time the Pod restarts.

### 1.9 Correct the PostgreSQL Deployment

File: `postgres-deployment.yml`

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

This fixes the volume indentation and ensures that the `init` volume and
its mount use the same name.

Important: This learning configuration has no persistent volume. If the
Pod is recreated with a fresh filesystem, database data may be lost. For
a more durable deployment, add a PersistentVolumeClaim and mount it at
`/var/lib/postgresql/data`.

### 1.10 Correct the PostgreSQL Service

File: `postgres-service.yml`

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

The Service name `postgres` is important: it is the hostname used in
your backend's PostgreSQL connection string.

### 1.11 Correct the backend Deployment

File: `backend-deployment.yml`

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

Kubernetes expands the earlier environment variables in `POSTGRES_URL`.
The backend application must actually read `POSTGRES_URL` and listen on
port `8080` for this configuration to work.

### 1.12 Correct the backend Service

File: `backend-service.yml`

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

The backend's stable internal address is:

`http://backend:8080`

This DNS name works for clients running inside the `devboard` namespace,
not directly in a user's browser outside the cluster.

### 1.13 Correct the Kind cluster configuration

File: `kind-config.yml`

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

## Step 4 --- How Namespace, Deployment, Secret, ConfigMap, init data, Pods and Services connect

The most important concept is that Kubernetes resources connect through
references, selectors and DNS names --- not because their YAML files are
physically linked together.

### 4.1 Connection map

Namespace: `devboard`

Common namespace for all application resources

ConfigMap

`devboard-configmap`

POSTGRES_USER

Secret

`devboard-secrets`

POSTGRES_PASSWORD, POSTGRES_DB

Frontend

Deployment

5 Pods

Backend

Deployment

2 Pods

PostgreSQL

Deployment

1 Pod

Deployments create Pods. Services route traffic to matching Pods. The
backend and database consume configuration and credentials.

The diagram above summarizes resource ownership and configuration.
Actual application traffic follows the frontend → backend → database
path, as shown in Step 5.

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

# CESAR School · Pós DevOps · Kubernetes

Repositório com os labs práticos do módulo de Orquestração de Containers com Kubernetes.

---

## Conceitos: como o Kubernetes funciona

O Kubernetes é **declarativo**: descrevemos o *estado desejado* em manifests YAML e o cluster trabalha continuamente para alcançá-lo. Quem faz esse trabalho são os **controllers**, que operam num *reconciliation loop*, comparando o que existe com o que foi pedido e agindo para convergir os dois.

O cluster se divide em **Control Plane** (decide o que deve acontecer) e **Worker Nodes** (onde os containers de fato rodam). Vejamos o que acontece quando rodamos um `kubectl apply`:

```mermaid
flowchart TD
    User(["💻 kubectl apply"])

    subgraph CP["🧠 Control Plane (o cérebro do cluster)"]
        direction TB
        API["🔵 API Server"]
        etcd[("📦 etcd")]
        Ctrl["⚙️ Controllers"]
        Sched["📅 Scheduler"]
    end

    subgraph WN["🖥️ Worker Node (onde os Pods rodam)"]
        direction TB
        Kubelet["🔧 Kubelet"]
        CR["🐳 Container Runtime"]
        CNI["🌐 CNI Plugin"]
        Pod["✅ Pod Running"]
    end

    User --> API
    API <--> etcd
    Ctrl <--> API
    Sched <--> API
    API --> Kubelet
    Kubelet --> CR --> CNI --> Pod
```

| Componente | Papel |
| --- | --- |
| 🔵 **API Server** | Porta de entrada: valida e processa todas as requisições |
| 📦 **etcd** | Banco que guarda o estado desejado do cluster |
| ⚙️ **Controllers** | Criam e reconciliam os recursos (Deployment → ReplicaSet → Pods) |
| 📅 **Scheduler** | Escolhe o melhor nó para cada Pod |
| 🔧 **Kubelet** | Executa as instruções no nó escolhido |
| 🐳 **Container Runtime** | Puxa a imagem e sobe o container (containerd / CRI-O) |
| 🌐 **CNI Plugin** | Atribui IP ao Pod e configura a rede |
| ✅ **Pod Running** | Pod pronto para receber tráfego |

> **Reconciliation loop:** esse ciclo nunca para. Se um Pod cair, o controller percebe a diferença entre o estado atual e o desejado e cria outro para restaurá-lo, sem intervenção manual.

---

## Lab 1: Workloads + Acesso + Persistência

Deploy completo do **TodoList** no cluster Kubernetes, cobrindo: Namespace, ConfigMap, Secret, PVC, Deployment, Service, Ingress e CronJob.

### Objetos que criaremos neste lab

Cada objeto do Kubernetes tem um papel específico. Criaremos oito, agrupados aqui por função:

```mermaid
flowchart TD
    K8s(["☸️ Objetos Lab 1"])

    K8s --> Iso["📁 Isolamento<br/>Namespace"]
    K8s --> Work["🚀 Workload<br/>Deployment → Pods"]
    K8s --> Conf["⚙️ Configuração<br/>ConfigMap · Secret"]
    K8s --> Stor["💾 Armazenamento<br/>PVC → PersistentVolume"]
    K8s --> Net["🌐 Rede / Acesso<br/>Service · Ingress"]
    K8s --> Task["⏰ Tarefas agendadas<br/>CronJob"]
```

Nos passos a seguir, criaremos cada um desses objetos individualmente. Ao final, há uma [visão geral](#visão-geral-como-todos-os-recursos-se-conectam) de como todos se conectam em runtime.

### Pré-requisitos

- Cluster [kind](https://kind.sigs.k8s.io/) rodando localmente
- NGINX Ingress Controller instalado no cluster
- `kubectl` configurado apontando para o cluster

### Estrutura dos manifests

```text
lab1/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── pvc.yaml
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── cronjob.yaml
```

### Como subir o ambiente

> Vamos aplicar os manifests na ordem abaixo. Cada passo depende do anterior.

#### 1. Namespace

Isola todos os recursos do lab. Todo manifest abaixo deve declarar `namespace: todolist-grupo-05`.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f namespace.yaml"]
    API["API Server"]
    etcd[("etcd")]
    NS["Namespace:<br/>todolist-grupo-05"]

    Apply --> API --> etcd --> NS
```

```bash
kubectl apply -f lab1/namespace.yaml
```

---

#### 2. ConfigMap

Armazena variáveis não-sensíveis (`APP_NAME`, `APP_PORT`, `APP_COLOR`) injetadas nos Pods via `envFrom`.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f configmap.yaml"]
    API["API Server"]
    etcd[("etcd")]
    CM["ConfigMap:<br/>todolist-config"]
    Pods["Pods"]

    Apply --> API --> etcd --> CM -->|envFrom: configMapRef| Pods
```

```bash
kubectl apply -f lab1/configmap.yaml
```

---

#### 3. Secret

Guarda `SESSION_KEY`, `ADMIN_USER`, `ADMIN_PASSWORD` e `CLEANUP_TOKEN`. Igual ao ConfigMap, mas os valores são codificados em base64 com restrições de acesso adicionais.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f secret.yaml"]
    API["API Server"]
    etcd[("etcd")]
    Sec["Secret:<br/>todolist-secret"]
    Pods["Pods"]
    Cron["CronJob:<br/>todolist-cleanup"]

    Apply --> API --> etcd --> Sec
    Sec -->|envFrom: secretRef| Pods
    Sec -->|secretKeyRef| Cron
```

```bash
kubectl apply -f lab1/secret.yaml
```

---

#### 4. PersistentVolumeClaim

Solicita um volume de `500Mi` (`ReadWriteOnce`) ao cluster. O Kubernetes provisiona o PersistentVolume e o vincula ao PVC; o Deployment o monta em `/data`, onde fica o banco `todos.db`.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f pvc.yaml"]
    API["API Server"]
    etcd[("etcd")]
    PVC["PVC:<br/>todolist-pvc"]
    PV["PersistentVolume"]
    Pods["Pods"]

    Apply --> API --> etcd --> PVC -->|bound| PV
    PVC -->|volumeMount /data| Pods
```

```bash
kubectl apply -f lab1/pvc.yaml
```

---

#### 5. Deployment

Garante que **2 réplicas** da imagem `andreffcastro/k8s-todolist:1.0.0` estejam sempre rodando, cada uma consumindo o ConfigMap e o Secret via `envFrom` e montando o PVC em `/data`.

O Deployment Controller cria um ReplicaSet, que cria e mantém os Pods até ficarem prontos:

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f deployment.yaml"]
    API["API Server"]
    etcd[("etcd")]
    DC["Deployment Controller"]
    RS["ReplicaSet"]
    Pods["Pods"]
    CM["ConfigMap:<br/>todolist-config"] & Sec["Secret:<br/>todolist-secret"] & PVC["PVC:<br/>todolist-pvc"]

    Apply --> API --> etcd --> DC --> RS --> Pods
    CM & Sec -->|envFrom| Pods
    PVC -->|volumeMount /data| Pods
```

```bash
kubectl apply -f lab1/deployment.yaml
```

---

#### 6. Service

Expõe os Pods via `ClusterIP` estável na porta `80 → 5000`, usando `selector: app: todolist` para balancear entre as réplicas.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f service.yaml"]
    API["API Server"]
    etcd[("etcd")]
    SVC["Service:<br/>todolist"]
    Pod1["Pod 1"]
    Pod2["Pod 2"]

    Apply --> API --> etcd --> SVC
    SVC -->|round robin| Pod1 & Pod2
```

```bash
kubectl apply -f lab1/service.yaml
```

---

#### 7. Ingress

Recebe requisições externas em `todolist-grupo-05.local` e roteia para o Service. Requer o NGINX Ingress Controller instalado.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f ingress.yaml"]
    API["API Server"]
    etcd[("etcd")]
    Ing["Ingress:<br/>todolist-ingress"]
    Browser(["Navegador"])
    IC["NGINX Ingress Controller"]
    SVC["Service:<br/>todolist"]

    Apply --> API --> etcd --> Ing
    Browser -->|HTTP| IC -->|lê as regras do| Ing
    Ing -->|roteia para| SVC
```

```bash
kubectl apply -f lab1/ingress.yaml
```

---

#### 8. CronJob

A cada 5 minutos (`*/5 * * * *`), um Job com a imagem `curlimages/curl:8` faz `POST /cleanup`, limpando itens concluídos do banco. O token vem diretamente do Secret.

```mermaid
flowchart LR
    Apply["kubectl apply<br/>-f cronjob.yaml"]
    API["API Server"]
    etcd[("etcd")]
    Cron["CronJob:<br/>todolist-cleanup"]
    Job["Job:<br/>curlimages/curl:8"]
    Sec["Secret:<br/>todolist-secret"]
    SVC["Service:<br/>todolist"]

    Apply --> API --> etcd --> Cron
    Cron -->|cria a cada 5 min| Job
    Sec -->|secretKeyRef| Job
    Job -->|POST /cleanup| SVC
```

```bash
kubectl apply -f lab1/cronjob.yaml
```

---

### Verificação

```bash
kubectl get all -n todolist-grupo-05
kubectl get pvc,ingress,cronjob -n todolist-grupo-05
```

### Acesso via navegador

Vamos adicionar a entrada abaixo ao `/etc/hosts`:

```text
127.0.0.1 todolist-grupo-05.local
```

A aplicação fica acessível em: [http://todolist-grupo-05.local](http://todolist-grupo-05.local)

---

### Visão geral: como todos os recursos se conectam

```mermaid
flowchart TD
    Browser(["Navegador"])
    Hosts["/etc/hosts"]
    Ing["Ingress:<br/>todolist-ingress"]
    SVC["Service:<br/>todolist"]
    Deploy["Deployment:<br/>todolist"]
    Pods["Pods (2 réplicas)"]
    CM["ConfigMap:<br/>todolist-config"]
    Sec["Secret:<br/>todolist-secret"]
    PVC["PVC:<br/>todolist-pvc"]
    Cron["CronJob:<br/>todolist-cleanup"]

    Browser --> Hosts --> Ing --> SVC --> Deploy --> Pods
    CM -->|envFrom| Pods
    Sec -->|envFrom| Pods
    PVC -->|volumeMount /data| Pods
    Sec -->|secretKeyRef| Cron
    Cron -->|POST /cleanup| SVC
```

---

## Lab 2: *em breve*

---

## Créditos

Disciplina ministrada pelo professor [@andreffcastro](https://github.com/andreffcastro), autor também da imagem [`andreffcastro/k8s-todolist`](https://hub.docker.com/r/andreffcastro/k8s-todolist) usada neste lab.

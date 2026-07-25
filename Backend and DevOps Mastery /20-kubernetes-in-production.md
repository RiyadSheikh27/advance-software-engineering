# Module 20 — Kubernetes in Production

> **Phase F — Infrastructure ও DevOps** | পূর্বশর্ত: M19
> পরের module: M21 (Cloud ও Infrastructure as Code)

---

## ১. যে autoscaler-টা load বাড়লে সিস্টেমকে আরও ধীর করে দিচ্ছিল

M31-এর payment platform Kubernetes-এ migrate করার পর, একটা peak traffic event-এ (M31-এর "মাস-শেষের payment spike") একটা অদ্ভুত প্যাটার্ন দেখা গেল: HPA (Horizontal Pod Autoscaler) নতুন pod বানাচ্ছিল ঠিকই, কিন্তু latency **আরও খারাপ** হচ্ছিল, ভালো হওয়ার বদলে।

তদন্তে তিনটা layered সমস্যা পাওয়া গেল:

**প্রথমত,** নতুন pod তৈরি হওয়ার পর তারা সাথে সাথে traffic পেতে শুরু করেছিল — কোনো readiness probe কনফিগার করা ছিল না। Django app-এর startup-এ M04-এর `gc.freeze()` আর connection pool warm-up-এ প্রায় ৮ সেকেন্ড লাগত, কিন্তু Kubernetes ধরে নিয়েছিল pod তৈরি হওয়া মানেই "ready" — নতুন pod-গুলো এখনো cold state-এ থাকা অবস্থায় request নিতে শুরু করল, timeout দিতে শুরু করল, HPA আরও বেশি pod বানাল প্রতিক্রিয়ায় (একটা misleading signal-এর ভিত্তিতে) — যেটা M02 §৮-এর "restart আরও worse করল" প্যাটার্নের একটা cluster-level পুনরাবৃত্তি।

**দ্বিতীয়ত,** resource `requests` অনেক কম set করা ছিল (আসল ব্যবহারের তুলনায়) "বেশি pod একটা node-এ fit করানোর" আশায় — যেটা M19 §৮-এর সতর্কতার সরাসরি লঙ্ঘন ছিল। ফলে node-গুলো overcommitted হয়ে গেল, CPU throttling শুরু হলো (M19 §৮-এর `nr_throttled`), আর pod-গুলো নিজেদের মধ্যে CPU-র জন্য প্রতিযোগিতা করতে লাগল।

**তৃতীয়ত,** database connection limit (M07-এর PgBouncer) নতুন pod সংখ্যার সাথে scale করেনি — প্রতিটা নতুন pod নতুন connection চাইছিল, কিন্তু PgBouncer-এর `default_pool_size` fixed ছিল, ফলে নতুন pod-গুলো connection-এর জন্য অপেক্ষা করছিল, cascading (M16-এর ধারণা, এখন K8s scaling-এর প্রেক্ষাপটে)।

এই ঘটনাটা দেখায় কেন Kubernetes শুধু "pod বাড়িয়ে দাও" এর চেয়ে অনেক বেশি — প্রতিটা scaling decision-এর একটা downstream ripple effect আছে যা M02, M04, M07, M16, M19-এর প্রতিটা layer-কে ছুঁয়ে যায়।

---

## ২. Pod, Deployment, এবং তাদের সম্পর্ক

### ২.১ Pod — সবচেয়ে ছোট deployable unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-api
spec:
  containers:
  - name: web
    image: myapp:v1.2.3
    ports: [{containerPort: 8000}]
```

**M19-এর container-এর সাথে সম্পর্ক:** Pod একটা বা একাধিক container-এর একটা wrapper যারা network namespace, storage share করে (M19 §৫-এর networking, একই pod-এর container-রা `localhost` দিয়ে একে অপরের সাথে কথা বলতে পারে)। বাস্তবে বেশিরভাগ pod-এ একটা main container থাকে (M31-এর Django app) এবং কখনো কখনো একটা **sidecar** (M16-এর service mesh proxy, বা log shipper)।

**গুরুত্বপূর্ণ — Pod সাধারণত সরাসরি তৈরি করা হয় না:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 4
  selector:
    matchLabels: {app: payment-api}
  template:               # ⚠️ এটাই Pod template
    metadata:
      labels: {app: payment-api}
    spec:
      containers:
      - name: web
        image: myapp:v1.2.3
```

**Deployment** Pod-দের জন্য একটা declarative controller — "সবসময় ৪টা Pod এই template অনুযায়ী চলুক" এই invariant বজায় রাখে (একটা Pod crash করলে automatically নতুন একটা তৈরি হয়) — M16-এর `Restart=always` (M19 §২.৪-এর systemd সমতুল্য) ধারণার cluster-level সংস্করণ।

### ২.২ StatefulSet — যখন identity গুরুত্বপূর্ণ

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  volumeClaimTemplates:    # ⚠️ প্রতিটা replica-র নিজস্ব persistent volume
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: {requests: {storage: 100Gi}}
```

**Deployment-এর সাথে মূল পার্থক্য:**

| | Deployment | StatefulSet |
|---|---|---|
| Pod নাম | Random suffix (`payment-api-7f8d9`) | Predictable, ordered (`postgres-0`, `postgres-1`) |
| Pod পরিচয় | Interchangeable — যেকোনোটা মুছে নতুন একটা একই | Persistent — `postgres-0` সবসময় `postgres-0`, তার নিজস্ব volume |
| Scaling order | সমান্তরাল | Sequential (0, তারপর 1, তারপর 2) |
| ব্যবহার | Stateless app (M31-এর payment API — M08-এর multi-tenancy আলোচনায় প্রতিটা instance interchangeable) | Database, message queue leader (M08-এর Patroni cluster, M13-এর RabbitMQ) |

**M08-এর Patroni/PostgreSQL replication-এর সাথে সরাসরি সংযোগ:** যদি M08-এর PostgreSQL primary+replica cluster Kubernetes-এ চালানো হয় (M19 §৬-এ যেখানে self-hosted বনাম managed database আলোচনা হয়েছিল), StatefulSet প্রয়োজনীয় কারণ **প্রতিটা replica-র identity গুরুত্বপূর্ণ** — `postgres-0` primary হতে পারে, `postgres-1`/`postgres-2` replica, আর তাদের volume (data) pod restart-এও একই replica-র সাথে bound থাকতে হবে (M19 §৬-এর volume persistence নীতি)।

### ২.৩ DaemonSet, Job, CronJob — সংক্ষিপ্ত

```yaml
# DaemonSet — প্রতিটা node-এ exactly একটা Pod (M24-এ log shipper/metrics agent-এর জন্য common)
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: log-agent}

# Job — একবার চলে, সম্পন্ন হলে থামে (M08-এর migration script-এর মতো one-off কাজ)
apiVersion: batch/v1
kind: Job
metadata: {name: db-migration}
spec:
  template:
    spec:
      containers: [{name: migrate, image: myapp:v1.2.3, command: ["python", "manage.py", "migrate"]}]
      restartPolicy: Never

# CronJob — M11-এর Celery Beat-এর Kubernetes-native বিকল্প
apiVersion: batch/v1
kind: CronJob
metadata: {name: partition-maintenance}
spec:
  schedule: "0 0 25 * *"   # M08 §৪.২-এর ensure_future_partitions task
  jobTemplate:
    spec:
      template:
        spec:
          containers: [{name: job, image: myapp:v1.2.3, command: ["python", "manage.py", "ensure_partitions"]}]
          restartPolicy: OnFailure
```

**M11 §৮.২-এর Celery Beat single-instance সতর্কতার সরাসরি সংযোগ:** CronJob Kubernetes-native ভাবে schedule guarantee দেয় (একটা নির্দিষ্ট সময়ে exactly একটা Job trigger, M11-এর "duplicate Beat instance" সমস্যা architecturally এড়িয়ে) — কিন্তু এটা Celery Beat-এর প্রতিস্থাপন **সবসময় না**, কারণ CronJob শুধু "নির্দিষ্ট সময়ে চালাও" করে, Celery Beat-এর মতো rich scheduling (M11-এর `crontab` expression-এর জটিল variant) বা task result tracking দেয় না সহজে।

---

## ৩. Probe — M19-এর Healthcheck-এর সম্পূর্ণ কাঠামো

### ৩.১ তিন ধরনের Probe

```yaml
containers:
- name: web
  livenessProbe:
    httpGet: {path: /health/live/, port: 8000}
    initialDelaySeconds: 15
    periodSeconds: 10
    failureThreshold: 3
  readinessProbe:
    httpGet: {path: /health/ready/, port: 8000}
    periodSeconds: 5
    failureThreshold: 2
  startupProbe:
    httpGet: {path: /health/live/, port: 8000}
    failureThreshold: 30      # ৩০ × periodSeconds পর্যন্ত startup-এর জন্য সময়
    periodSeconds: 2
```

| Probe | প্রশ্ন যা এটা জিজ্ঞেস করে | ব্যর্থ হলে কী হয় | §১-এর ঘটনায় ভূমিকা |
|---|---|---|---|
| **Liveness** | "এই process কি বেঁচে আছে (deadlock না)?" | Container **restart** | — |
| **Readiness** | "এই Pod কি এখন traffic নেওয়ার জন্য প্রস্তুত?" | Pod **Service endpoint থেকে সরানো** (traffic বন্ধ, কিন্তু restart না) | **এটাই অনুপস্থিত ছিল** |
| **Startup** | "Application কি এখনো শুরু হচ্ছে?" | Liveness/readiness probe শুরু হওয়া বিলম্বিত করে | সাহায্য করত এখানে |

**§১-এর ঘটনার সমাধান:** M04-এর `gc.freeze()`+connection warm-up-এর ৮ সেকেন্ড startup time-এর জন্য যদি readiness probe থাকত (`initialDelaySeconds` বা `startupProbe` দিয়ে সঠিকভাবে কনফিগার করা), নতুন pod সেই ৮ সেকেন্ড **Service endpoint-এর বাইরে** থাকত — কোনো traffic পেত না যতক্ষণ না সত্যিই প্রস্তুত, HPA-র misleading signal (failed request থেকে "আরও pod দরকার" ভুল সিদ্ধান্ত) কখনো তৈরি হতো না।

### ৩.২ Liveness Probe-এর একটা বিপজ্জনক ভুল

```python
# ❌ বিপজ্জনক — liveness probe DB check করছে
def liveness_check(request):
    connection.cursor().execute("SELECT 1")   # M10 §১২-এর fail-open নীতি লঙ্ঘন!
    return JsonResponse({"status": "ok"})
```

**কেন এটা M16-এর cascading failure তৈরি করে:** যদি M07-এর PostgreSQL সাময়িকভাবে slow/overloaded হয় (M08-এর migration lock, M07-এর VACUUM-এর সময়), এই liveness probe fail করবে — Kubernetes ধরে নেবে pod "dead" (deadlock-এ আছে), এটাকে **restart** করবে। কিন্তু pod নিজে perfectly healthy ছিল, শুধু একটা downstream dependency ধীর ছিল — restart কিছুই সমাধান করে না (DB এখনো slow), বরং নতুন connection re-establish করার overhead যোগ করে, যা M31-এর "restart আরও worse করল" প্যাটার্নের নিখুঁত পুনরাবৃত্তি।

> **Senior Tip:** "Liveness আর readiness probe-এ কী পার্থক্য রাখা উচিত কনটেন্টে?" — "Liveness probe-এ **শুধু** এই process নিজে deadlock/hung state-এ আছে কি না চেক করা উচিত (M19 §২.১-এর `SIGTERM` handler ঠিকমতো কাজ করছে কি না-এর মতোই মৌলিক প্রশ্ন) — কোনো external dependency ছোঁয়া উচিত না। Readiness probe-এ external dependency check করা যায় (M31-এর payment API-তে PostgreSQL, কারণ ছাড়া কোনো কাজ হবে না), কারণ readiness-এর ব্যর্থতার consequence অনেক হালকা — শুধু traffic থেকে সরিয়ে দেওয়া, restart না। এই পার্থক্যটাই M31-এর ঘটনায় liveness probe-এ DB check রাখার bug-টা এড়াতে পারত।"

---

## ৪. Resource Requests/Limits ও QoS Class — M19-এর cgroup-এর Cluster-level প্রয়োগ

### ৪.১ Requests বনাম Limits

```yaml
resources:
  requests: {cpu: "250m", memory: "512Mi"}   # scheduling guarantee — এই node-এ এতটুকু জায়গা থাকতেই হবে
  limits: {cpu: "500m", memory: "1Gi"}       # M19 §৮-এর cgroup hard limit
```

**M19 §৮-এর সরাসরি সম্প্রসারণ:** `requests` scheduler-কে বলে "এই Pod-এর জন্য এতটুকু resource guaranteed রাখো" (bin-packing decision-এ ব্যবহৃত), `limits` cgroup-level hard ceiling (M19-এ যা আলোচিত হয়েছিল)। যদি `requests` না দেওয়া হয়, scheduler কোনো guarantee ছাড়াই pod বসিয়ে দিতে পারে একটা ইতিমধ্যে-ব্যস্ত node-এ — §১-এর ঘটনার দ্বিতীয় কারণ এটাই ছিল।

### ৪.২ QoS Class — তিনটা স্তর

```
Guaranteed: requests == limits (CPU এবং memory উভয়ে)
  → সর্বোচ্চ priority, সবার শেষে evict হবে node pressure-এ

Burstable: requests < limits
  → মাঝারি priority, Guaranteed-এর পরে evict হবে

BestEffort: কোনো requests/limits নেই
  → সর্বনিম্ন priority, প্রথমে evict হবে
```

**M31-এর payment API-এর জন্য সঠিক QoS নীতি:**

```yaml
# Payment API — critical path, Guaranteed QoS উপযুক্ত
resources:
  requests: {cpu: "500m", memory: "1Gi"}
  limits: {cpu: "500m", memory: "1Gi"}   # ⚠️ requests == limits

# Analytics batch job — non-critical, Burstable যথেষ্ট
resources:
  requests: {cpu: "100m", memory: "256Mi"}
  limits: {cpu: "1000m", memory: "1Gi"}   # burst করতে পারে, কিন্তু guaranteed কম
```

> **Senior Tip:** "কোন workload-এ Guaranteed QoS ব্যবহার করবেন?" — "M31-এর latency-critical path (payment API) — যেখানে node memory pressure-এর সময় eviction একটা business-critical outage তৈরি করবে। Non-critical batch job-এ Burstable যথেষ্ট এবং বেশি resource-efficient (M21-এর FinOps নীতি — over-provisioning avoid করা), কারণ সেগুলো eviction সহ্য করতে পারে (M16-এর graceful degradation নীতির resource-scheduling সংস্করণ)।"

---

## ৫. Autoscaling — HPA, VPA, KEDA, Cluster Autoscaler

### ৫.১ HPA — M31-এর Capacity Math-এর Automated সংস্করণ

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: payment-api}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: payment-api}
  minReplicas: 4
  maxReplicas: 50
  metrics:
  - type: Resource
    resource: {name: cpu, target: {type: Utilization, averageUtilization: 70}}
```

**M31 §১-এর capacity formula-র সরাসরি automated সংস্করণ:** HPA CPU utilization ৭০% ছাড়ালে নতুন Pod তৈরি করে — এটা M31-এর "concurrency = workers × threads, RPS = concurrency / response_time" নীতির একটা automated, reactive প্রয়োগ। কিন্তু **§১-এর ঘটনা দেখায় কেন এটা যথেষ্ট না** — CPU utilization একটা lagging indicator, আর নতুন Pod-এর warm-up time (readiness probe ছাড়া) misleading signal তৈরি করতে পারে।

### ৫.২ KEDA — Queue-based Scaling, M11-এর Celery-র জন্য প্রাসঙ্গিক

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: celery-worker-scaler}
spec:
  scaleTargetRef: {name: celery-worker}
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
  - type: redis
    metadata:
      address: redis:6379
      listName: celery              # M11-এর Celery queue
      listLength: "50"                # queue-তে ৫০+ pending task হলে scale up
```

**M11 §১০-এর "queue depth monitor করুন" নীতির সরাসরি বাস্তবায়ন:** M11-এ আমরা বলেছিলাম Celery-র নিজস্ব `--autoscale` production-এ যথেষ্ট sophisticated না, KEDA-র মতো external autoscaler পছন্দনীয়। এই YAML সেই সুপারিশের concrete implementation — queue depth (M11-এর সবচেয়ে গুরুত্বপূর্ণ metric) সরাসরি scaling trigger, CPU utilization-এর মতো indirect proxy না।

### ৫.৩ VPA — Vertical Pod Autoscaler

```
HPA: কতগুলো Pod (horizontal)
VPA: প্রতিটা Pod-এর resource requests/limits কতটা হওয়া উচিত (vertical)

VPA বিশেষভাবে useful M19 §৮-এর "সঠিক limit কীভাবে ঠিক করব" প্রশ্নের
জন্য — ম্যানুয়াল load testing-এর বদলে, VPA actual usage observe করে
স্বয়ংক্রিয়ভাবে requests/limits recommend (বা automatically apply) করে।
```

⚠️ **সতর্কতা:** VPA এবং HPA একই resource metric (CPU) দিয়ে একসাথে ব্যবহার করলে conflict হতে পারে — VPA pod resize করতে চাইছে, HPA pod সংখ্যা বদলাতে চাইছে, দুইটা একই সংকেত নিয়ে কাজ করছে। সাধারণ practice: HPA custom metric (queue depth, M11-এর মতো) দিয়ে, VPA শুধু recommendation mode-এ (auto-apply না) ব্যবহার করা।

### ৫.৪ Cluster Autoscaler — Node-level Scaling

```
HPA/VPA/KEDA: Pod-level (existing node-এর মধ্যে)
Cluster Autoscaler: Node-level — যদি কোনো Pod "Pending" থাকে কারণ কোনো
                     node-এ যথেষ্ট জায়গা নেই, নতুন node যোগ করে cloud
                     provider থেকে (M21-এ EC2/node pool প্রসঙ্গে বিস্তারিত)
```

**§১-এর ঘটনার তৃতীয় স্তরের সমস্যা এখানে সংযুক্ত:** যদি HPA নতুন Pod তৈরি করে কিন্তু কোনো node-এ জায়গা না থাকে (M19 §৮-এর resource requests-এর কারণে), Cluster Autoscaler নতুন node provision করে — কিন্তু এই প্রক্রিয়ায় **কয়েক মিনিট** লাগতে পারে (cloud provider VM boot time, M02-এর TLS/connection setup overhead নতুন node-এ)। এই lag-টাই traffic spike-এর সবচেয়ে সংকটময় মুহূর্তে ঘটে, ঠিক যখন capacity সবচেয়ে বেশি দরকার।

> **Senior Tip:** "কীভাবে autoscaling lag reduce করবেন peak traffic-এ?" — "M31-এর 'predictable spike' আলোচনা এখানে সরাসরি প্রযোজ্য — যদি মাস-শেষের spike predictable হয় (M31-এর payment example-এর মতো), আমি **scheduled scaling** ব্যবহার করব (একটা CronJob যা peak-এর আগে `minReplicas` বাড়িয়ে দেয়, M20-এর CronJob দিয়ে HPA-র min override করে), reactive autoscaling-এর উপর সম্পূর্ণ নির্ভর না করে। Unpredictable spike-এর জন্য, node pool-এ কিছু 'buffer capacity' (over-provisioned, কিন্তু low-priority pod দিয়ে ভরা যা preempt করা যায়) রাখা একটা common pattern — যাতে Cluster Autoscaler-এর node boot time-এর অপেক্ষা করতে না হয়।"

---

## ৬. PodDisruptionBudget — M08-এর Zero-downtime-র Cluster-level Enforcement

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: payment-api-pdb}
spec:
  minAvailable: 3          # যেকোনো সময় অন্তত ৩টা Pod running থাকতেই হবে
  selector: {matchLabels: {app: payment-api}}
```

**M08 §৬-এর zero-downtime migration নীতির সরাসরি cluster-level সম্প্রসারণ:** PDB নিশ্চিত করে node maintenance, cluster upgrade, বা voluntary eviction-এর সময় (M21-এ spot instance reclamation প্রসঙ্গে ফিরে আসবে) একসাথে অনেকগুলো Pod সরিয়ে দেওয়া যাবে না — Kubernetes নিজে নিশ্চিত করে `minAvailable` সবসময় respected হচ্ছে, deployment-এর rolling update ছাড়াও।

---

## ৭. Affinity, Anti-Affinity, Taint/Toleration

### ৭.১ Pod Anti-Affinity — M16-এর Bulkhead-এর Node-level সংস্করণ

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector: {matchLabels: {app: payment-api}}
      topologyKey: "kubernetes.io/hostname"   # ⚠️ একই node-এ দুইটা payment-api Pod না
```

**M16-এর bulkhead নীতির সরাসরি প্রয়োগ, কিন্তু node-failure-এর প্রেক্ষাপটে:** এই কনফিগারেশন নিশ্চিত করে সব `payment-api` Pod ভিন্ন ভিন্ন node-এ ছড়িয়ে থাকবে — একটা node crash হলে **সব** payment-api Pod একসাথে হারাবে না, শুধু একটা। এটা M08-এর multi-AZ deployment নীতির (Patroni replica ভিন্ন AZ-তে) pod-level সংস্করণ।

### ৭.২ Taint/Toleration — Node Restriction

```bash
kubectl taint nodes gpu-node-1 workload=ml:NoSchedule
```

```yaml
tolerations:
- key: "workload"
  operator: "Equal"
  value: "ml"
  effect: "NoSchedule"
```

Taint একটা node-কে "সাধারণ Pod-এর জন্য না" চিহ্নিত করে; শুধু matching toleration-সহ Pod সেখানে schedule হতে পারে। **M09-এর polyglot infrastructure-এর node-level প্রয়োগ:** যদি একটা GPU node pool শুধু ML inference workload-এর জন্য (M30-এ বিস্তারিত), taint নিশ্চিত করে সাধারণ Django Pod ভুলবশত সেই দামী GPU node-এ schedule হবে না (M21-এর cost efficiency নীতি)।

---

## ৮. Deployment Strategy — M08-এর Migration Playbook-এর Runtime সংস্করণ

### ৮.১ Rolling Update (ডিফল্ট)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # একসাথে সর্বোচ্চ ২টা অতিরিক্ত Pod তৈরি হতে পারে
      maxUnavailable: 1    # একসাথে সর্বোচ্চ ১টা Pod unavailable হতে পারে
```

**M08 §৬.১-এর expand-contract-এর সাথে সরাসরি সম্পর্ক:** Rolling update মানে পুরনো এবং নতুন কোড **কিছু সময়ের জন্য একসাথে** চলবে (M08-এর zero-downtime migration নীতির ঠিক সেই প্রেক্ষাপট যা migration-কে backward-compatible হতে বাধ্য করেছিল) — এই কারণেই M08-এর "migration আর deployment আলাদা সময়ে ঘটে" সতর্কতা এখানে সরাসরি প্রাসঙ্গিক।

### ৮.২ Blue-Green Deployment

```
Blue (বর্তমান, v1) সম্পূর্ণভাবে চলছে
Green (নতুন, v2) সম্পূর্ণভাবে deploy হয়, কিন্তু কোনো traffic পাচ্ছে না
পরীক্ষা সম্পন্ন হলে → Service selector সুইচ করা হয় Blue থেকে Green-এ, তাৎক্ষণিক
সমস্যা হলে → সুইচ ফিরিয়ে নেওয়া Blue-তে, তাৎক্ষণিক rollback
```

**Trade-off বনাম Rolling Update:** Blue-Green-এ **কখনোই** পুরনো এবং নতুন কোড একসাথে live traffic serve করে না (M08-এর backward-compatibility চাপ কম) — কিন্তু double resource প্রয়োজন (সাময়িকভাবে দুইটা সম্পূর্ণ environment), আর database migration-এর ক্ষেত্রে এখনো M08-এর expand-contract প্রয়োজনীয় (কারণ উভয় version-ই একই database ব্যবহার করে switch-এর মুহূর্ত পর্যন্ত)।

### ৮.৩ Canary Deployment

```yaml
# একটা সরল canary — traffic-এর একটা ছোট অংশ নতুন version-এ
apiVersion: apps/v1
kind: Deployment
metadata: {name: payment-api-canary}
spec:
  replicas: 1   # মোট ২১টা Pod-এর মধ্যে ১টা canary = ~৫% traffic
```

**M16-এর chaos engineering নীতির সাথে সংযোগ — production-এ controlled risk:** Canary deployment নতুন কোড-কে সম্পূর্ণ traffic-এর একটা ছোট অংশে expose করে আগে, M25-এর incident-এর blast radius সীমিত রাখতে (M16-এর bulkhead নীতির deployment-level প্রয়োগ)। M06-এর error rate monitoring এখানে critical — canary-তে error rate spike দেখলে rollout **স্বয়ংক্রিয়ভাবে** থামানো (Flagger, Argo Rollouts-এর মতো টুল দিয়ে) M22-এ progressive delivery-তে সম্পূর্ণ বিস্তারিত হবে।

---

## ৯. Debugging — সাধারণ Failure State

### ৯.১ `CrashLoopBackOff`

```bash
kubectl logs <pod-name> --previous   # ⚠️ crash হওয়া container-এর log, current না
kubectl describe pod <pod-name>       # Events section-এ exit code
```

```
সাধারণ কারণ:
  - Application startup-এ crash (missing env var, DB connection ব্যর্থ)
  - M19 §২.১-এর signal handling সমস্যা (যদি এটা graceful shutdown-এর
    বদলে crash-এর মতো দেখায়)
  - M04-এর uncaught exception যা Gunicorn worker-কে বারবার crash করাচ্ছে
```

### ৯.২ `OOMKilled`

```bash
kubectl describe pod <pod-name>   # "Last State: Terminated, Reason: OOMKilled"
```

**M19 §১-এর সম্পূর্ণ debugging প্রক্রিয়া এখানে প্রযোজ্য** — cgroup memory accounting, শুধু M04-এর application profiling না। এখানে অতিরিক্ত: `limits.memory` বনাম actual usage তুলনা করা `kubectl top pod` দিয়ে, এবং M20 §৪-এ QoS class পুনর্বিবেচনা করা।

### ৯.৩ `ImagePullBackOff`

```
সাধারণ কারণ:
  - Image tag ভুল, বা image registry-তে নেই
  - Registry authentication ব্যর্থ (imagePullSecrets misconfigured)
  - M19 §৯-এর image scanning gate যদি deployment pipeline-এ block করে থাকে
```

### ৯.৪ এলোমেলো 502 — M02-এর সম্পূর্ণ প্রেক্ষাপট Kubernetes-এ

```
M02 §৭-এর "keep-alive timeout race" এবং connection draining আলোচনা
এখানে সরাসরি প্রযোজ্য — Kubernetes Service/Ingress-এর কনটেক্সটে:

  - preStop hook নেই (M02 §৭)
  - terminationGracePeriodSeconds খুব কম
  - Readiness probe না থাকায় (M20 §৩) নতুন Pod cold অবস্থায় traffic পাচ্ছে
  - PDB না থাকায় (M20 §৬) node maintenance-এ অনেক Pod একসাথে সরে যাচ্ছে
```

### ৯.৫ DNS Latency — M02-এর `ndots` সমস্যার সম্পূর্ণ প্রেক্ষাপট

M02 §৬.১-এ আমরা Kubernetes-এর `ndots:5` সমস্যা বিস্তারিত দেখেছিলাম। এখানে সেটা এখন **debugging context**-এ: pod-এর মধ্যে মাঝে মাঝে ৫ সেকেন্ডের latency spike দেখলে, প্রথম সন্দেহ M02-এর DNS conntrack race condition, সমাধান NodeLocal DNSCache deploy করা।

> **Senior Tip:** "Kubernetes-এ একটা production incident-এ প্রথম কী দেখবেন?" — "একটা systematic ক্রম অনুসরণ করি: `kubectl get pods` (কোনগুলো Running/CrashLoopBackOff/Pending), `kubectl describe pod` (Events section-এ scheduling/probe failure reason), `kubectl logs --previous` (যদি crash হয়ে থাকে), তারপর M19-এর container-level debugging (cgroup stats) যদি resource-related মনে হয়, আর M02-এর network debugging toolkit (`curl -w`, DNS check) যদি connectivity-related মনে হয়। Kubernetes নিজে একটা নতুন debugging layer যোগ করে, কিন্তু M02-M19-এর সব নীতি এর ভেতরেই থেকে যায় — Kubernetes সমস্যা তৈরি করে না, এটা existing সমস্যাগুলোকে নতুন উপায়ে প্রকাশ করে (scheduling delay, probe misconfiguration) অথবা নতুন coordination সমস্যা যোগ করে (multi-pod scaling, §১-এর ঘটনার মতো)।"

---

## ১০. Helm ও ArgoCD/GitOps — সংক্ষিপ্ত

```yaml
# Helm — templated Kubernetes manifest, M08-এর migration versioning-এর
# ধারণাগত সমতুল্য কিন্তু infrastructure config-এর জন্য
# values.yaml
replicaCount: 4
image: {repository: myapp, tag: v1.2.3}
resources:
  requests: {cpu: "250m", memory: "512Mi"}
```

```yaml
# ArgoCD — GitOps: Git repository-ই source of truth, cluster state
# automatically Git-এর সাথে sync হয় (M22-এর CI/CD-র deployment অংশ)
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source: {repoURL: "https://github.com/org/k8s-manifests", path: "payment-api"}
  syncPolicy: {automated: {prune: true, selfHeal: true}}
```

**M22-এর প্রস্তুতি:** GitOps-এর মূল ধারণা হলো — deployment একটা `kubectl apply` command না, বরং একটা **Git commit**। এটা M08-এর migration-এর version control নীতির (প্রতিটা schema change একটা tracked migration file) infrastructure-level সম্প্রসারণ — প্রতিটা infrastructure change auditable, revertible (git revert), এবং declarative।

---

## ১১. Interview Section

### প্রশ্ন ১ (Senior) — "Liveness আর readiness probe-এর পার্থক্য কী, এবং liveness probe-এ ভুল কনফিগারেশনের ঝুঁকি কী?"

**🌟 Senior/Staff Answer**
> "Liveness probe জিজ্ঞেস করে 'এই process কি বেঁচে আছে (deadlock না)?' — ব্যর্থ হলে container **restart** হয়। Readiness probe জিজ্ঞেস করে 'এই Pod কি এখন traffic নেওয়ার জন্য প্রস্তুত?' — ব্যর্থ হলে Pod শুধু Service endpoint থেকে সাময়িকভাবে সরানো হয়, restart না।
>
> সবচেয়ে বিপজ্জনক ভুল হলো liveness probe-এ external dependency (database, cache) চেক করা। যদি একটা downstream dependency সাময়িকভাবে ধীর হয় (M07-এর VACUUM, বা কোনো network issue), liveness probe fail করবে, Kubernetes সেই perfectly-healthy pod-কে **restart** করবে — যেটা কিছুই সমাধান করে না (dependency এখনো ধীর) কিন্তু connection re-establish-এর নতুন overhead যোগ করে, এবং যদি অনেক pod একসাথে একই dependency-র সমস্যায় পড়ে, mass restart একটা cascading failure তৈরি করতে পারে ঠিক যখন সিস্টেমের স্থিতিশীলতা সবচেয়ে বেশি দরকার।
>
> সঠিক ডিজাইন: liveness probe-এ শুধু process নিজের health (কোনো external call ছাড়া), readiness probe-এ external dependency check (যেখানে ব্যর্থতার consequence অনেক হালকা — traffic routing বন্ধ, ধ্বংসাত্মক restart না)।"

---

### প্রশ্ন ২ (Staff / Production Incident) — "HPA পুরোপুরি কাজ করছে (নতুন pod তৈরি হচ্ছে), কিন্তু traffic spike-এর সময় latency আরও খারাপ হচ্ছে। ডিবাগ করুন।"

**🌟 Senior/Staff Answer**
> "এটা একটা multi-layer সমস্যা হতে পারে, M20-এর 'সব কিছু autoscale করলেই যথেষ্ট না' নীতির একটা প্রকাশ — নতুন Pod তৈরি হওয়া মানেই সেটা কার্যকরভাবে capacity যোগ করছে তা না। আমি ক্রমান্বয়ে চেক করব:
>
> **১. Readiness probe আছে এবং সঠিকভাবে কনফিগার করা কি না।** যদি না থাকে, নতুন Pod cold state-এই traffic পাচ্ছে (M04-এর warm-up, connection pool initialization শেষ হওয়ার আগেই), যেটা আসলে latency **আরও খারাপ** করবে কারণ এই cold pod-গুলো ধীর response দিচ্ছে কিন্তু তবুও load balancer তাদের traffic পাঠাচ্ছে।
>
> **২. Resource requests বাস্তবসম্মত কি না।** যদি requests কম set করা থাকে (node-এ বেশি pod ঠাসার আশায়), নতুন Pod তৈরি হলেও CPU throttling (M19 §৮-এর `nr_throttled`) শুরু হতে পারে — Pod সংখ্যা বাড়ছে কিন্তু actual usable capacity সেই অনুপাতে বাড়ছে না।
>
> **৩. Downstream bottleneck — এই ক্ষেত্রে সবচেয়ে সম্ভাব্য।** M07/M02-এর মূল নীতি: 'pods × workers = DB connection'। যদি PgBouncer-এর `default_pool_size` fixed থাকে, নতুন Pod-দের প্রতিটা নতুন connection চাইবে, কিন্তু pool-এ জায়গা না থাকলে তারা **connection-এর জন্য অপেক্ষা** করবে — Pod নিজে সুস্থ, কিন্তু downstream bottleneck-এ latency যোগ হচ্ছে যা আরও বেশি Pod দিয়ে সমাধান হয় না, বরং একই bottleneck-এ আরও বেশি contention তৈরি করে।
>
> এই তিনটাই একসাথে ঘটতে পারে (যেমন M31-এর মূল ঘটনায় হয়েছিল), তাই আমি একটা একটা করে ঠিক করে প্রতিবার পরিমাপ করব — শুধু 'আরও autoscaling' যোগ করা এই সমস্যার সমাধান না, বরং প্রতিটা layer-এর bottleneck আলাদাভাবে সমাধান করা দরকার।"

---

### প্রশ্ন ৩ (Coding / Debugging) — "একটা Deployment-এ `maxUnavailable: 0, maxSurge: 1` সেট করা আছে ৪টা replica-র জন্য। Deploy ধীর কেন?"

**🌟 Senior/Staff Answer**
> "`maxUnavailable: 0` মানে rolling update-এর সময় **কোনো** existing Pod unavailable হতে পারবে না — নতুন Pod সম্পূর্ণভাবে ready হওয়ার পরেই একটা পুরনো Pod সরানো হবে। `maxSurge: 1` মানে একবারে শুধু **একটা** অতিরিক্ত Pod তৈরি হতে পারে।
>
> এর মানে ৪টা Pod আপডেট করতে Kubernetes-কে **সিরিয়ালি** কাজ করতে হবে: ১টা নতুন Pod তৈরি → readiness probe pass করা পর্যন্ত অপেক্ষা → ১টা পুরনো Pod সরানো → পরেরটার জন্য পুনরাবৃত্তি। ৪ বার এই cycle, প্রতিটাতে readiness probe-এর warm-up time (M20 §১-এর ৮ সেকেন্ডের উদাহরণ) — মোট deploy সময় প্রায় `4 × (pod_startup_time)`।
>
> এই কনফিগারেশন **ইচ্ছাকৃতভাবে conservative** — এটা zero-downtime নিশ্চিত করে সর্বোচ্চ সতর্কতার সাথে (কখনো capacity কমে না রোলআউটের সময়), কিন্তু deploy speed-এর বিনিময়ে। যদি deploy speed গুরুত্বপূর্ণ হয় এবং টিম সামান্য reduced capacity সহ্য করতে পারে rolling window-এ, `maxUnavailable: 1, maxSurge: 2` এর মতো একটা কম conservative কনফিগারেশন অনেক দ্রুত হবে (সমান্তরালে একাধিক Pod আপডেট)।
>
> এই সিদ্ধান্তটা M31-এর availability trade-off আলোচনার একটা deployment-level প্রয়োগ — কতটা conservative হতে হবে নির্ভর করে সিস্টেমের criticality আর deploy frequency-র উপর। M31-এর payment API-তে আমি `maxUnavailable: 0` রাখব (safety প্রথম), কিন্তু কম-critical internal tool-এ দ্রুত deploy-কে prioritize করব।"

---

### প্রশ্ন ৪ (Architecture Decision) — "আমাদের database-কে (M07/M08-এর PostgreSQL) Kubernetes-এ StatefulSet হিসেবে চালানো উচিত, নাকি managed service (RDS)?"

**🌟 Senior/Staff Answer**
> "এই সিদ্ধান্ত M09-এর polyglot persistence checklist এবং M19 §৬-এর 'self-hosted বনাম managed' আলোচনার একটা প্রত্যক্ষ প্রয়োগ।
>
> **StatefulSet-এ self-host করার পক্ষে যুক্তি:** cost savings (managed service-এর premium এড়ানো), infrastructure একই জায়গায় (Kubernetes-নেটিভ tooling, monitoring একই জায়গায়), আর কিছু সংগঠনে regulatory/data-residency কারণে cloud-managed service ব্যবহার করা যায় না।
>
> **Managed service-এর পক্ষে যুক্তি, এবং এটাই আমার ডিফল্ট সুপারিশ:** M08-এর সব operational জটিলতা — automated failover (Patroni-র সমতুল্য কিন্তু cloud provider-এর টিম maintain করে), automated backup+PITR, security patching, monitoring — এসব managed service নিজে করে। StatefulSet-এ self-hosted PostgreSQL চালাতে গেলে, টিমকে নিজে Patroni configure করতে হবে (M08 §৩), backup automation বানাতে হবে (M08 §১০), storage/volume management বুঝতে হবে (M19 §৬-এর PersistentVolume, network-attached storage-এর latency characteristic), আর একটা database-crash 3am-এ নিজেদেরই debug করতে হবে, cloud provider-এর SLA-backed support ছাড়া।
>
> আমার সুপারিশ: managed service দিয়ে শুরু করুন, যদি না **নির্দিষ্ট, ন্যায্যতাপ্রাপ্ত** কারণ থাকে (regulatory, বা টিমের গভীর database operations expertise + cost একটা critical constraint)। M09-এর checklist-এর একই মূলনীতি এখানে প্রযোজ্য: নতুন operational জটিলতা (এই ক্ষেত্রে, database operations নিজে করা) শুধু তখনই নেওয়া উচিত যখন measured প্রয়োজন এবং team capability উভয়ই স্পষ্ট।"

---

## ১২. হাতে-কলমে অনুশীলন

**১ — Readiness probe-এর প্রভাব দেখুন (৩০ মিনিট, Minikube/Kind দিয়ে)**
একটা Django app deploy করুন যেখানে startup-এ ইচ্ছাকৃত delay (`time.sleep(10)`) আছে। প্রথমে readiness probe ছাড়া deploy করুন, load test চালান deploy চলাকালীন — error rate দেখুন। তারপর readiness probe যোগ করে আবার পরীক্ষা করুন।

**২ — QoS class পরীক্ষা করুন (২০ মিনিট)**
তিনটা Pod বানান — Guaranteed, Burstable, BestEffort QoS দিয়ে। `kubectl describe pod` দিয়ে `QoS Class` field দেখুন। একটা memory-pressure scenario simulate করে (node-এর memory শেষ করে) দেখুন কোন Pod আগে evict হয়।

**৩ — CrashLoopBackOff ডিবাগ করুন (২৫ মিনিট)**
ইচ্ছাকৃতভাবে ভাঙা একটা Deployment বানান (ভুল env var, missing dependency)। `kubectl logs --previous`, `kubectl describe pod` দিয়ে root cause খুঁজে বের করুন।

**৪ — Rolling update সিমুলেট করুন (২৫ মিনিট)**
একটা Deployment-এ image version বদলান বিভিন্ন `maxSurge`/`maxUnavailable` কনফিগারেশন দিয়ে, রোলআউট সময় এবং একই সময়ে চলা Pod সংখ্যা পর্যবেক্ষণ করুন `kubectl get pods -w` দিয়ে।

---

## ১৩. মূল কথা

1. **Deployment stateless workload-এর জন্য, StatefulSet identity/persistent storage প্রয়োজন হলে** — M08-এর Patroni cluster-এর মতো database workload StatefulSet-এর natural fit।
2. **Liveness probe-এ কখনো external dependency check করবেন না** — ভুলভাবে করলে একটা downstream slowdown-কে mass pod restart-এ পরিণত করে, cascading failure তৈরি করে।
3. **Readiness probe না থাকলে নতুন Pod cold অবস্থায় traffic পায়** — HPA-কে misleading signal দিয়ে বিপরীত ফল তৈরি করতে পারে (§১-এর ঘটনা)।
4. **Resource requests M19-এর cgroup সীমার cluster-level bin-packing decision** — কম requests দিলে node overcommit হয়, throttling তৈরি করে।
5. **HPA শুধু symptom (CPU/queue depth) দেখে, downstream bottleneck (DB connection pool) দেখে না** — আরও Pod সবসময় সাহায্য করে না যদি bottleneck অন্য স্তরে থাকে।
6. **KEDA queue-depth-based scaling M11-এর 'CPU না, queue metric monitor করুন' নীতির সরাসরি বাস্তবায়ন** — Celery/RabbitMQ-র জন্য CPU-based HPA-র চেয়ে উপযুক্ত।
7. **PodDisruptionBudget M08-এর zero-downtime নীতিকে cluster maintenance-এর সময়েও enforce করে**, শুধু deployment-এর সময় না।
8. **Pod anti-affinity M16-এর bulkhead নীতির node-failure সংস্করণ** — সব critical Pod একই node-এ থাকলে একটা node crash-এ পুরো capacity হারানোর ঝুঁকি।
9. **Cluster Autoscaler-এর node boot latency predictable spike-এ scheduled scaling দিয়ে এড়ানো যায়** — শুধু reactive autoscaling-এর উপর নির্ভর না করে।
10. **Kubernetes debugging M02-M19-এর সব নীতির উপর দাঁড়িয়ে** — নতুন layer যোগ করে (scheduling, probe), কিন্তু আগের সমস্যাগুলো অদৃশ্য হয় না, নতুন উপায়ে প্রকাশ পায়।

---

## পরের Module

**M21 — Cloud ও Infrastructure as Code।** আজ আমরা একটা Kubernetes cluster-এর ভেতরের orchestration দেখলাম। পরের module-এ আমরা এক ধাপ উপরে উঠব — সেই cluster নিজে কোথায় চলে (AWS EC2/EKS, M20-এর node pool-এর ভিত্তি), কীভাবে provision হয় (Terraform, "code হিসেবে infrastructure" — M08-এর migration versioning নীতির সম্পূর্ণ infrastructure সম্প্রসারণ), storage/networking cloud-level এ (S3, VPC — M02-এর networking নীতির cloud-provider প্রয়োগ), আর cost engineering (M20-এর resource requests/QoS সিদ্ধান্তগুলোর প্রকৃত dollar-cost)।

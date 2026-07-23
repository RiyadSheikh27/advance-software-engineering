# Module 13 — RabbitMQ ও Broker Decision Matrix

> **Phase D — Async, Messaging & Streaming** | পূর্বশর্ত: M10, M11, M12
> পরের module: M14 (Event-Driven Architecture Patterns)

---

## ১. যে notification system-এ একজন ইউজার তার নিজের bank-এর email পেয়েছিল

একটা multi-bank aggregator platform-এ notification system RabbitMQ দিয়ে বানানো হয়েছিল। প্রতিটা bank-এর জন্য আলাদা routing দরকার ছিল — HSBC-র ইউজাররা HSBC-নির্দিষ্ট notification পাবে, DBS-এর ইউজাররা DBS-নির্দিষ্ট। ডেভেলপার একটা **direct exchange** ব্যবহার করলেন, routing key হিসেবে bank code দিয়ে।

কিন্তু একটা bug ছিল — routing key বানানো হচ্ছিল string concatenation দিয়ে `f"{bank_code}.{event_type}"`, আর একটা bank code-এ ভুলবশত একটা wildcard character (`*`) ছিল ডেটা migration-এর সময় (একটা করাপ্টেড ডেটা এন্ট্রি)। Direct exchange-এ এটা normally কোনো সমস্যা করত না (direct exchange exact match চায়) — কিন্তু কারো একটা পুরনো queue binding **topic exchange**-এর wildcard pattern দিয়ে করা ছিল একটা migration-এর সময় যা সম্পূর্ণ পরিষ্কার হয়নি।

ফলাফল: `hsbc.*` pattern-এ bind করা একটা queue, যেটা মূলত সব HSBC event ধরার জন্য বানানো হয়েছিল, ওই করাপ্টেড routing key-র কারণে **অন্য bank-এর event-ও ধরতে শুরু করল**। একজন DBS ইউজার HSBC-নির্দিষ্ট promotional email পেলেন — একটা privacy incident যা compliance review পর্যন্ত গড়িয়েছিল।

এই ঘটনাটা RabbitMQ-র সবচেয়ে গুরুত্বপূর্ণ ধারণাগত পার্থক্য তুলে ধরে Kafka থেকে (M12): **RabbitMQ-তে routing logic broker-এর ভেতরে থাকে** (exchange type, binding, routing key pattern) — Kafka-তে routing সহজ (topic-এর মধ্যে partition key দিয়ে), কিন্তু RabbitMQ-তে routing নিজেই একটা জটিল, শক্তিশালী, এবং **ভুল করার সুযোগ থাকা** সিস্টেম। এই module-এ আমরা এই routing model সম্পূর্ণভাবে বুঝব, আর কখন এই শক্তি প্রয়োজনীয়, কখন এটা অপ্রয়োজনীয় জটিলতা।

---

## ২. মৌলিক পার্থক্য — Message Broker বনাম Log

```mermaid
flowchart LR
    subgraph Kafka["Kafka — Log Model (M12)"]
        K1["Producer"] --> K2["Topic Partition<br/>(persistent log)"]
        K2 --> K3["Consumer পড়ে,<br/>offset নিজে ট্র্যাক করে<br/>message মুছে যায় না"]
    end
    subgraph RabbitMQ["RabbitMQ — Queue Model"]
        R1["Producer"] --> R2["Exchange<br/>(routing logic)"]
        R2 --> R3["Queue<br/>(message এখানে জমা)"]
        R3 --> R4["Consumer পড়ে ও ack করে,<br/>message queue থেকে সরে যায়"]
    end
```

| | Kafka | RabbitMQ |
|---|---|---|
| Message model | Persistent log, consumer offset নিজে ট্র্যাক করে | Queue, consume হলে (ack-এর পর) মুছে যায় |
| Replay | সম্ভব (offset rewind, retention period পর্যন্ত) | সাধারণত না (message consume হলে চলে যায়) |
| Routing জটিলতা | সরল (topic + partition key) | সমৃদ্ধ (exchange type, binding, routing key pattern) |
| Multiple independent consumer | Consumer group দিয়ে স্বাভাবিক (M12 §৫) | প্রতিটা consumer group-এর জন্য আলাদা queue বানাতে হয় |
| Message priority | নেই | Native support |
| Per-message TTL/delay | Workaround লাগে | Native support (plugin সহ) |
| Throughput ceiling | অত্যন্ত উচ্চ (partition-based scaling) | উচ্চ, কিন্তু সাধারণত Kafka-র চেয়ে কম |

**M09-এর polyglot persistence নীতি এখানেও প্রযোজ্য:** এই দুইটা ভিন্ন সমস্যা সমাধান করে ভিন্নভাবে ডিজাইন করা — একটা "কোনটা ভালো" প্রশ্ন না, বরং "কোন access pattern-এর জন্য কোনটা স্বাভাবিক ফিট" প্রশ্ন।

---

## ৩. Exchange Type — routing logic-এর হৃদয়

### ৩.১ Direct Exchange — exact match

```python
import pika

channel.exchange_declare(exchange="notifications", exchange_type="direct")
channel.queue_declare(queue="hsbc_notifications")
channel.queue_bind(exchange="notifications", queue="hsbc_notifications",
                    routing_key="hsbc")   # exact match

channel.basic_publish(exchange="notifications", routing_key="hsbc", body=message)
# ✅ শুধু routing_key="hsbc" বাইন্ড করা queue-তে যাবে
```

**ব্যবহার:** exact-match routing দরকার হলে — M31-এর example-এ, "শুধু নির্দিষ্ট merchant-এর webhook worker-এ কাজ পাঠানো।"

### ৩.২ Topic Exchange — pattern match (§১-এর ঝুঁকির উৎস)

```python
channel.exchange_declare(exchange="events", exchange_type="topic")

# * = ঠিক একটা word, # = শূন্য বা একাধিক word
channel.queue_bind(exchange="events", queue="hsbc_all",
                    routing_key="hsbc.*")           # hsbc.payment, hsbc.login — সবই
channel.queue_bind(exchange="events", queue="all_payments",
                    routing_key="*.payment")         # hsbc.payment, dbs.payment
channel.queue_bind(exchange="events", queue="everything",
                    routing_key="#")                 # সব কিছু
```

**§১-এর ঘটনার মূল শিক্ষা:** topic exchange-এর শক্তিই এর ঝুঁকি — wildcard pattern দিয়ে flexible routing পাওয়া যায়, কিন্তু routing key-তে কোনো data validation bug হলে সেটা **অপ্রত্যাশিত queue-তে message route করতে পারে**, কোনো explicit error ছাড়াই (broker-এর দৃষ্টিতে এটা সম্পূর্ণ বৈধ pattern match, কোনো ভুল না)।

> **Senior Tip:** "Topic exchange ব্যবহার করার সময় সবচেয়ে বড় ঝুঁকি কী?" — "routing key generation-এ কোনো bug থাকলে সেটা broker-level-এ চুপচাপ ভুল queue-তে message পাঠাবে, কোনো error/exception ছাড়াই — কারণ pattern matching নিজেই ঠিকঠাক কাজ করছে, শুধু input ভুল। তাই routing key তৈরি করা কোডে কঠোর validation/sanitization বাধ্যতামূলক (allowed character whitelist, wildcard character-এর মতো `*`/`#` কখনো data থেকে সরাসরি না আসা উচিত), আর প্রতিটা binding pattern-এর একটা automated test থাকা উচিত যা নিশ্চিত করে শুধু intended message-ই সেই queue-তে পৌঁছায়।"

### ৩.৩ Fanout Exchange — broadcast, routing key উপেক্ষা করে

```python
channel.exchange_declare(exchange="payment_broadcast", exchange_type="fanout")
channel.queue_bind(exchange="payment_broadcast", queue="analytics_queue")
channel.queue_bind(exchange="payment_broadcast", queue="fraud_queue")
channel.queue_bind(exchange="payment_broadcast", queue="notification_queue")

channel.basic_publish(exchange="payment_broadcast", routing_key="",   # উপেক্ষিত
                       body=payment_event)
# ✅ তিনটা queue-ই এই message পাবে
```

**এটা M12-এর interview প্রশ্ন ৪-এর ("payment event তিনটা downstream system-এ") RabbitMQ সংস্করণ।** Fanout exchange RabbitMQ-তে সেই একই কাজ করে যা Kafka-তে একটা topic একাধিক consumer group দিয়ে করে — **প্রতিটা bound queue তার নিজস্ব সম্পূর্ণ কপি পায়।**

### ৩.৪ Headers Exchange — কম ব্যবহৃত, কিন্তু জানা দরকার

```python
channel.exchange_declare(exchange="events_h", exchange_type="headers")
channel.queue_bind(exchange="events_h", queue="high_priority_bd",
                    arguments={"country": "BD", "priority": "high", "x-match": "all"})
```

Routing key-র বদলে message header দিয়ে match করে, একাধিক শর্ত (`x-match: all` = সব মিলতে হবে, `any` = যেকোনো একটা)। কার্যত কম ব্যবহৃত কারণ topic exchange-এর wildcard pattern বেশিরভাগ ব্যবহারিক প্রয়োজন কভার করে, কিন্তু multi-dimensional filtering (country + priority + type একসাথে) দরকার হলে headers exchange পরিষ্কার।

---

## ৪. Dead Letter Exchange (DLX) — M12-এর DLQ pattern-এর RabbitMQ-নেটিভ সংস্করণ

```python
# মূল queue — DLX কনফিগার করা
channel.queue_declare(
    queue="webhook_delivery",
    arguments={
        "x-dead-letter-exchange": "webhook_dlx",
        "x-dead-letter-routing-key": "failed",
        "x-message-ttl": 30000,          # ৩০ সেকেন্ডে TTL — expire হলে DLX-এ
    },
)

channel.exchange_declare(exchange="webhook_dlx", exchange_type="direct")
channel.queue_declare(queue="webhook_dlq")
channel.queue_bind(exchange="webhook_dlx", queue="webhook_dlq", routing_key="failed")
```

**একটা message DLX-এ যায় তিনটা পরিস্থিতিতে (M12-এর manual DLQ logic-এর তুলনায় RabbitMQ-তে এটা native):**

```
১. Consumer message-টা nack/reject করে (requeue=False দিয়ে)
২. Message-এর TTL expire হয়ে যায় (উপরের x-message-ttl)
৩. Queue length limit ছাড়িয়ে যায় (x-max-length)
```

```python
def callback(ch, method, properties, body):
    try:
        process_webhook(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except NonRetryableError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # → DLX-এ যাবে
    except RetryableError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)   # আবার মূল queue-তে
```

⚠️ **ফাঁদ — `requeue=True` দিয়ে unlimited retry loop:** যদি একটা message বারবার একই কারণে ব্যর্থ হয় (M11 §৬.৩-এর non-retryable error শ্রেণীবিভাগ ঠিকমতো না করলে), আর প্রতিবার `requeue=True` করা হয়, এটা M12-এর poison message সমস্যার RabbitMQ সংস্করণ তৈরি করে — একটা infinite retry loop যা consumer-কে ব্যস্ত রাখে কিন্তু কখনো progress করে না। **নিয়ম:** একটা retry counter header ট্র্যাক করুন, নির্দিষ্ট সংখ্যার পর `requeue=False` করে DLX-এ পাঠান।

---

## ৫. Delayed Message — Retry Queue Pattern

RabbitMQ-তে native delay নেই (একটা plugin ছাড়া), কিন্তু TTL + DLX দিয়ে একটা elegant pattern তৈরি করা যায় — M11 §৬.২-এর exponential backoff-এর RabbitMQ বাস্তবায়ন:

```python
# "delay queue" — কোনো consumer নেই, শুধু TTL expire হয়ে DLX-এ পাঠানোর জন্য
channel.queue_declare(
    queue="webhook_retry_5s",
    arguments={
        "x-message-ttl": 5000,                            # ৫ সেকেন্ড অপেক্ষা
        "x-dead-letter-exchange": "webhook_main_exchange",  # তারপর মূল queue-তে ফেরত
        "x-dead-letter-routing-key": "webhook_delivery",
    },
)
```

```mermaid
flowchart LR
    A["মূল Queue"] -->|ব্যর্থ| B["Retry Queue<br/>TTL=5s, কোনো consumer নেই"]
    B -->|"TTL expire<br/>(DLX দিয়ে)"| A
    A -->|"বারবার ব্যর্থ"| C["retry_10s queue"]
    C --> A
    A -->|"max_retries পার"| DLQ["ডেড লেটার queue<br/>(স্থায়ী)"]
```

প্রতিটা retry attempt-এ একটা ভিন্ন TTL queue ব্যবহার করে (5s, 10s, 20s...) — M11-এর `retry_backoff=True` ঠিক এই একই exponential pattern তৈরি করে, কিন্তু Celery-তে built-in মেকানিজম দিয়ে, RabbitMQ-তে এই manual queue chaining দিয়ে।

> **Senior Tip:** "RabbitMQ-তে delayed retry কীভাবে implement করবেন?" প্রশ্নে এই TTL+DLX pattern ব্যাখ্যা করার পর যোগ করুন: "এই pattern কাজ করে কিন্তু বেশ verbose — প্রতিটা delay interval-এর জন্য আলাদা queue। বিকল্প হলো `rabbitmq-delayed-message-exchange` plugin, যেটা সরাসরি per-message delay support করে, কিন্তু এটা একটা non-core plugin, তাই operational dependency বাড়ে। ছোট থেকে মাঝারি scale-এ TTL+DLX pattern নির্ভরযোগ্য কারণ এটা core RabbitMQ feature দিয়েই কাজ করে, কোনো অতিরিক্ত dependency ছাড়া।"

---

## ৬. Publisher Confirm ও Consumer Acknowledgment

### ৬.১ Publisher Confirm — M12-এর `acks` ধারণার RabbitMQ সংস্করণ

```python
channel.confirm_delivery()   # confirm mode চালু

try:
    channel.basic_publish(
        exchange="payment_events", routing_key="payment.created",
        body=message,
        properties=pika.BasicProperties(delivery_mode=2),  # persistent — ডিস্কে লেখা
    )
    # basic_publish confirm mode-এ ব্যর্থ হলে exception raise করে
except pika.exceptions.UnroutableError:
    logger.error("message_unroutable")   # কোনো queue এই routing key ধরেনি!
```

`delivery_mode=2` (persistent) M08 §২.১-এর `synchronous_standby_names`/M12-এর `acks=all`-এর একই মূলনীতি — message ডিস্কে লেখা নিশ্চিত হওয়ার আগে broker crash হলে data হারাতে পারে যদি persistent mode না থাকে।

**`UnroutableError` একটা RabbitMQ-নির্দিষ্ট সুবিধা যা Kafka-তে নেই:** যদি কোনো queue একটা routing key-র সাথে bound না থাকে, RabbitMQ (mandatory flag সহ) সেটা **জানিয়ে দেয়** — message silently হারিয়ে যায় না, exception আসে। এটা §১-এর ঘটনার বিপরীত সমস্যা সমাধান করে — routing bug যদি "কোনো queue-ই match করেনি" হতো (ভুল queue-তে যাওয়ার বদলে), সেটা অন্তত visible হতো।

### ৬.২ QoS/Prefetch — M31-এর capacity math-এর RabbitMQ প্রয়োগ

```python
channel.basic_qos(prefetch_count=10)
# একটা consumer একসাথে সর্বোচ্চ ১০টা unacknowledged message পাবে
```

M31 §৫.১-এর concurrency formula-র সরাসরি প্রয়োগ: `prefetch_count` নির্ধারণ করে একটা consumer কতগুলো message সমান্তরালে "হাতে" রাখতে পারে ack করার আগে। খুব বেশি prefetch দিলে একটা slow consumer অনেক message ধরে রাখে (অন্য consumer-দের জন্য available না), খুব কম দিলে network round-trip overhead বেশি (প্রতি message-এ নতুন fetch)।

```
prefetch_count = concurrent processing capacity (M31-এর সূত্র প্রয়োগ)
    = worker thread সংখ্যা × (নিরাপত্তা মার্জিন, সাধারণত ১.৫-২×)
```

---

## ৭. Quorum Queue বনাম Classic Mirrored Queue

### ৭.১ Classic Mirrored Queue — পুরনো HA পদ্ধতি (deprecated হচ্ছে)

```python
# পুরনো পদ্ধতি — policy দিয়ে mirroring
channel.queue_declare(queue="critical_queue")
# rabbitmqctl set_policy ha-all "^critical_queue$" '{"ha-mode":"all"}'
```

**সমস্যা:** Mirrored queue-তে একটা "master" node সব operation প্রসেস করে, mirror-রা asynchronously sync হয় (M08-এর async replication-এর মতোই risk) — network partition-এ split-brain-এর মতো ঝুঁকি, আর failover-এ data loss সম্ভাবনা থাকে।

### ৭.২ Quorum Queue — আধুনিক পছন্দ

```python
channel.queue_declare(
    queue="critical_queue",
    arguments={"x-queue-type": "quorum"},
)
```

**Raft consensus algorithm-ভিত্তিক** (M15-এ পূর্ণ বিস্তারিত) — M08-এর Patroni/etcd-এর মতো একই consensus নীতি, কিন্তু RabbitMQ-র নিজস্ব queue replication-এ প্রয়োগ করা। একটা majority quorum লিখিত নিশ্চিত করার পরই write সফল ধরা হয় — data loss ঝুঁকি অনেক কম mirrored queue-র চেয়ে, split-brain প্রতিরোধ built-in।

> **Senior Tip:** "কোন queue type ব্যবহার করবেন?" — "২০২৬-এ নতুন কোনো critical queue-তে **quorum queue** ডিফল্ট পছন্দ হওয়া উচিত, classic mirrored queue না — RabbitMQ টিম নিজেই মাইগ্রেশনের সুপারিশ করছে, কারণ quorum queue-এর Raft-ভিত্তিক consensus মূলত ওই একই ধরনের data-loss ঝুঁকি সমাধান করে যা M08-এ আমরা Patroni দিয়ে PostgreSQL-এ সমাধান করেছিলাম। শুধু non-critical, high-throughput, ephemeral queue-তে (যেখানে message loss গ্রহণযোগ্য, শুধু throughput matter করে) classic queue এখনো কিছুটা performance সুবিধা দিতে পারে।"

---

## ৮. সম্পূর্ণ Broker Decision Matrix

M09-M13 জুড়ে যা শেখা হয়েছে, তার সংশ্লেষণ:

| Broker | Model | সেরা fit | দুর্বলতা |
|---|---|---|---|
| **Kafka** | Persistent log, partition-based | High throughput, replay দরকার, একাধিক independent consumer, strict per-key ordering | Operational জটিলতা বেশি, simple task queue-র জন্য overkill |
| **RabbitMQ** | Queue + exchange routing | জটিল routing logic, priority queue, delayed message, moderate throughput task queue | Kafka-র মতো native replay নেই, extreme scale-এ Kafka-র চেয়ে সীমিত |
| **Redis Streams** (M10 §১১) | Log-like, কিন্তু memory-bound | ছোট-মাঝারি scale, ইতিমধ্যে Redis infrastructure আছে | Persistence guarantee দুর্বল, retention memory-limited |
| **SQS** (AWS managed) | Queue, at-least-once | Serverless/AWS-native stack, operational burden zero চাই | Vendor lock-in, ordering সীমিত (FIFO queue আলাদা, throughput কম) |
| **NATS** | Lightweight pub/sub, JetStream দিয়ে persistence | অতি-নিম্ন latency, microservice mesh internal messaging | Ecosystem/tooling Kafka-র চেয়ে কম পরিণত |
| **Pulsar** | Kafka-সদৃশ, কিন্তু compute/storage আলাদা (BookKeeper) | Multi-tenancy native, geo-replication built-in | Operational জটিলতা Kafka-র চেয়েও বেশি, ছোট টিমে অপ্রয়োজনীয় |

### ৮.১ সিদ্ধান্ত-গাছ (M09-এর polyglot persistence কাঠামোর প্রয়োগ)

```
প্রশ্ন ১: শুধু simple fire-and-forget task queue দরকার (M11-এর Celery ব্যবহার)?
  → হ্যাঁ: Redis (M10) যথেষ্ট ব্রোকার হিসেবে, নতুন সিস্টেম দরকার নেই

প্রশ্ন ২: জটিল routing logic দরকার (multi-criteria, priority, delayed message)?
  → হ্যাঁ: RabbitMQ

প্রশ্ন ৩: Event replay, একাধিক independent consumer, বা extreme throughput দরকার?
  → হ্যাঁ: Kafka

প্রশ্ন ৪: ইতিমধ্যে AWS-native, operational simplicity সবচেয়ে গুরুত্বপূর্ণ?
  → হ্যাঁ: SQS (+ SNS fanout-এর জন্য)

প্রশ্ন ৫: Multi-tenant SaaS platform, geo-replication native প্রয়োজন?
  → বিবেচনা করুন Pulsar (কিন্তু team capability যাচাই করুন, M09-এর checklist)
```

> **Senior Tip:** "Kafka নাকি RabbitMQ?" — এই প্রশ্নের সবচেয়ে ভুল উত্তর একটা definitively "Kafka ভালো, বেশি modern" বলা। সঠিক উত্তর ফ্রেম করুন M09-এর polyglot persistence প্রশ্নগুলো দিয়ে: "যদি আমাদের প্রধান প্রয়োজন জটিল routing (এই merchant-এর এই ধরনের event শুধু এই নির্দিষ্ট worker pool-এ) আর moderate throughput, RabbitMQ-র exchange model স্বাভাবিক fit এবং কম operational overhead। যদি আমাদের event replay/audit trail/একাধিক ভবিষ্যতের consumer দরকার, Kafka-র log model architecturally সঠিক। দুইটাই 'production-grade message broker' — পার্থক্য access pattern-এ, modernness-এ না।"

---

## ৯. Django/Celery Integration — RabbitMQ Broker হিসেবে

```python
# celery.py — M11-এর broker হিসেবে RabbitMQ
app.conf.broker_url = "amqp://user:pass@rabbitmq:5672/vhost"
app.conf.broker_transport_options = {
    "confirm_publish": True,       # §৬.১-এর publisher confirm
}

# Task priority — RabbitMQ-র native feature, Kafka-তে সহজে সম্ভব না
app.conf.task_queue_max_priority = 10
app.conf.task_default_priority = 5

@shared_task(priority=9)   # উচ্চ priority — OTP-র মতো critical task (M31-এর latency budget)
def send_otp_sms(phone, code):
    ...

@shared_task(priority=1)   # নিম্ন priority — marketing email
def send_marketing_email(user_id, campaign_id):
    ...
```

**M11 §৯.৩-এ আলাদা queue দিয়ে priority handle করা হয়েছিল** (`critical` বনাম `bulk` queue) — RabbitMQ broker হলে message-level priority-ও ব্যবহার করা যায়, একই queue-র মধ্যে, যেটা Redis broker-এ (M10) সহজে সম্ভব না। এটা RabbitMQ বেছে নেওয়ার একটা concrete কারণ হতে পারে যদি priority-based task ordering গুরুত্বপূর্ণ হয় শুধু আলাদা queue যথেষ্ট না হয়ে।

---

## ১০. Interview Section

### প্রশ্ন ১ (Senior) — "RabbitMQ আর Kafka-র মধ্যে মৌলিক architectural পার্থক্য কী?"

**🌟 Senior/Staff Answer**
> "সবচেয়ে fundamental পার্থক্য হলো message-এর 'জীবনচক্র' মডেল। RabbitMQ একটা traditional **queue** — message consume ও acknowledge হলে queue থেকে সরে যায়, প্রায় একটা to-do list-এর মতো, কাজ শেষ হলে item মুছে যায়। Kafka একটা **persistent log** — message consume হওয়ার পরও log-এ থেকে যায় retention period পর্যন্ত, consumer শুধু তাদের 'read position' (offset) ট্র্যাক করে, message নিজে অপরিবর্তিত থাকে।
>
> এই একটা পার্থক্য থেকে বাকি সব পার্থক্য বেরিয়ে আসে। Kafka-তে replay স্বাভাবিক (offset rewind), RabbitMQ-তে না (message চলে গেছে)। Kafka-তে একাধিক independent consumer group একই data স্বাধীনভাবে, নিজস্ব গতিতে পড়তে পারে, RabbitMQ-তে সেটার জন্য explicit fanout exchange দিয়ে একাধিক queue বানাতে হয়, প্রতিটা queue নিজস্ব কপি ধরে রাখে।
>
> বিপরীতে, RabbitMQ-র routing model (exchange type, binding pattern) Kafka-র চেয়ে অনেক বেশি expressive — complex, multi-criteria routing logic broker-level-এই করা যায়, Kafka-তে সেটা consumer-side application logic-এ করতে হয়। আর RabbitMQ-তে native message priority ও delayed delivery আছে, যেগুলো Kafka-তে workaround লাগে।
>
> সংক্ষেপে: simple task distribution এবং জটিল routing-এ RabbitMQ স্বাভাবিক, event streaming/audit trail/replay-এ Kafka স্বাভাবিক। এদের মধ্যে 'কোনটা ভালো' প্রশ্ন না — কোনটা আপনার access pattern-এর সাথে architecturally সাযুজ্যপূর্ণ, সেই প্রশ্ন।"

---

### প্রশ্ন ২ (Staff / Production Incident) — "একটা RabbitMQ queue-তে message জমতে থাকছে, consumer সচল আছে বলে মনে হচ্ছে। ডিবাগ করুন।"

**🌟 Senior/Staff Answer**
> "প্রথম চেক — consumer সত্যিই message consume করছে, নাকি শুধু connected কিন্তু stuck? `rabbitmqctl list_queues name messages consumers` দিয়ে queue depth আর consumer count দেখব, তারপর `rabbitmqctl list_consumers` দিয়ে প্রতিটা consumer-এর অবস্থা।
>
> **সম্ভাব্য কারণ, সবচেয়ে সাধারণ থেকে কম সাধারণ:**
>
> ১. **Prefetch/QoS misconfiguration (§৬.২)।** যদি একটা consumer অনেক message 'হাতে' নিয়ে রেখেছে (উচ্চ `prefetch_count`) কিন্তু ধীরে process করছে, বাকি message queue-তেই জমে থাকবে যদিও 'consumer সচল'। M31-এর capacity formula প্রয়োগ করে দেখব — actual processing rate কি প্রয়োজনীয় rate-এর চেয়ে কম।
>
> ২. **একটা poison message infinite requeue loop-এ (§৪-এর সতর্কতা)।** যদি একটা message বারবার `nack(requeue=True)` হচ্ছে (কোনো non-retryable error-কে ভুলবশত retryable ধরে), সেটা কার্যকরভাবে queue-র 'head' block করছে — নতুন message যোগ হতে থাকবে কিন্তু পুরনোটা কখনো শেষ হবে না processing-wise (M12-এর poison message সমস্যার হুবহু সমান্তরাল)।
>
> ৩. **Consumer ack করছে না — একটা bug যেখানে exception-এর পর `basic_ack`/`basic_nack` কোনোটাই কল হচ্ছে না।** RabbitMQ তখন সেই message unacknowledged রাখবে, consumer disconnect না হওয়া পর্যন্ত — queue depth বাড়তেই থাকবে যদিও 'consumer আছে' কারণ কোনো নতুন message deliver হচ্ছে না (unacked limit-এর কারণে, যদি prefetch limit ছোঁয়া হয়ে গেছে)।
>
> ৪. **Network partition-এর মতো কিছু consumer আর broker-এর মধ্যে**, যেখানে TCP connection 'দেখতে' জীবিত কিন্তু heartbeat miss হচ্ছে (M02-এর keep-alive/timeout আলোচনার সরাসরি প্রয়োগ) — RabbitMQ কিছুক্ষণ পর সেই connection বন্ধ করবে, কিন্তু ততক্ষণে message জমে গেছে।
>
> আমার প্রথম action হবে `rabbitmqctl list_consumers` দিয়ে actual delivery rate দেখা, তারপর application log-এ exception/error দেখা যেখানে ack/nack কল হচ্ছে কি না নিশ্চিত করা। যদি সব ঠিক দেখায় কিন্তু rate কম, তাহলে এটা capacity সমস্যা (M31-এর scaling ladder প্রয়োগ), bug না।"

---

### প্রশ্ন ৩ (Coding / Architecture) — "একটা multi-tenant SaaS-এ প্রতিটা tenant-এর event শুধু তাদের নিজস্ব worker-এ যাওয়া উচিত, tenant সংখ্যা কয়েকশো। কীভাবে design করবেন?"

**🌟 Senior/Staff Answer**
> "এখানে দুইটা approach আছে, দুইটাই বৈধ কিন্তু ভিন্ন trade-off:
>
> **RabbitMQ দিয়ে — topic exchange + tenant-based routing key:**
> ```python
> channel.exchange_declare(exchange='tenant_events', exchange_type='topic')
> channel.queue_bind(exchange='tenant_events', queue=f'tenant_{tenant_id}_queue',
>                    routing_key=f'{tenant_id}.#')
> ```
> এটা সহজবোধ্য, কিন্তু কয়েকশো tenant মানে কয়েকশো queue — RabbitMQ প্রতিটা queue-র জন্য কিছু overhead রাখে (memory, management), তাই এই স্কেলে এটা operationally ভারী হয়ে উঠতে পারে। আর §১-এর ঘটনার মতো — routing key generation-এ কোনো bug থাকলে ভুল tenant-এর queue-তে data যাওয়ার ঝুঁকি থাকে, তাই আমি প্রতিটা binding pattern-এর automated isolation test রাখব (M06-এর multi-tenant IDOR test-এর সমান্তরাল ধারণা, ভিন্ন layer-এ)।
>
> **Kafka দিয়ে — tenant_id partition key:**
> ```python
> producer.send('tenant.events', key=str(tenant_id).encode(), value=event)
> ```
> এখানে একটাই topic, tenant_id partition key হিসেবে (M12 §৭-এর নীতি) — একই tenant-এর সব event একই partition-এ, ordering guaranteed। Consumer-side-এ একটা single consumer group যথেষ্ট partition-অনুযায়ী assignment পাবে, কোনো per-tenant queue তৈরির দরকার নেই broker-level-এ। কিন্তু এখানে **isolation দুর্বল** — যদি একটা tenant-এর data অন্য tenant-এর worker-এ কখনো যাওয়া **একেবারেই না** ঘটতে পারে এমন কড়া guarantee দরকার হয় (M08-এর multi-tenancy RLS আলোচনার মতোই একটা compliance concern), partition-based approach-এ সেটা consumer-side filtering-এর উপর নির্ভর করে, broker enforce করে না।
>
> **আমার সুপারিশ:** যদি tenant isolation একটা soft requirement (performance/organization-এর জন্য, security-critical না), Kafka partition-based approach বেশি scalable এবং কম operational overhead। যদি tenant isolation একটা hard, compliance-driven requirement (কিছু enterprise customer সম্পূর্ণ physical isolation দাবি করে), তাদের জন্য RabbitMQ-র dedicated per-tenant queue (বা এমনকি M08-এর dedicated database-এর মতো, dedicated broker/vhost) বিবেচনা করব — শুধু সেই নির্দিষ্ট customer-দের জন্য, সবার জন্য না (M08 §৭.১-এর একই সিদ্ধান্ত-কাঠামো এখানে broker-level-এ প্রযোজ্য)।"

---

### প্রশ্ন ৪ (Scenario) — "RabbitMQ broker রিস্টার্ট হলো, কিছু in-flight message হারিয়ে গেছে। কেন, এবং কীভাবে প্রতিরোধ করবেন?"

**🌟 Senior/Staff Answer**
> "তিনটা জায়গায় এটা ঘটতে পারে, প্রতিটার আলাদা সমাধান:
>
> **১. Queue নিজে durable ছিল না।**
> ```python
> channel.queue_declare(queue='webhook_delivery', durable=True)  # ⚠️ ডিফল্ট False!
> ```
> Non-durable queue broker restart-এ **সম্পূর্ণ মুছে যায়**, তার সব message-সহ। এটা M07-এর `UNLOGGED` table-এর ধারণাগত সমতুল্য — কিছু ব্যবহারে ইচ্ছাকৃতভাবে গ্রহণযোগ্য (ephemeral, non-critical queue), কিন্তু business-critical queue-তে কখনো না।
>
> **২. Message persistent হিসেবে পাঠানো হয়নি।**
> ```python
> channel.basic_publish(..., properties=pika.BasicProperties(delivery_mode=2))  # ⚠️ ডিফল্ট 1 (non-persistent)
> ```
> এমনকি durable queue-তেও, যদি individual message `delivery_mode=2` না হয়, সেটা ডিস্কে লেখা হয় না — শুধু memory-তে থাকে, broker crash/restart-এ হারিয়ে যায়। এটা §৬.১-এ উল্লেখ করা হয়েছিল, কিন্তু এখানে গুরুত্ব পুনরাবৃত্তি করা দরকার: **durable queue + persistent message দুইটাই লাগে**, একটা যথেষ্ট না।
>
> **৩. Classic mirrored queue-তে asynchronous mirror sync।** যদি master node crash করে ঠিক তখন যখন কিছু message এখনো mirror-এ sync হয়নি (M08-এর async replication lag-এর সমান্তরাল ঝুঁকি), সেই message হারিয়ে যায় failover-এ।
>
> **প্রতিরোধ, সম্পূর্ণ:**
> - সব business-critical queue: `durable=True`
> - সব business-critical message: `delivery_mode=2`
> - Publisher confirm mode ব্যবহার (§৬.১) — নিশ্চিত হওয়া message আসলে broker-এ পৌঁছেছে ও persist হয়েছে, শুধু network-এ পাঠানো হয়েছে তা না
> - Critical queue-তে classic mirrored-এর বদলে **quorum queue** (§৭.২) — Raft-ভিত্তিক, data loss ঝুঁকি অনেক কম
>
> এই চারটা একসাথে না করলে, প্রতিটা individually 'ঠিক আছে' মনে হতে পারে কিন্তু সম্মিলিতভাবে durability-র একটা false sense of security তৈরি করে — যেটা এই ঘটনার মূল কারণ ছিল সম্ভবত।"

---

## ১১. হাতে-কলমে অনুশীলন

**১ — Exchange type তুলনা (৩০ মিনিট, Docker দিয়ে)**
Local RabbitMQ চালান (Docker)। একটা direct, একটা topic, একটা fanout exchange বানান, প্রতিটাতে ২-৩টা queue bind করুন ভিন্ন pattern দিয়ে। একই message ভিন্ন routing key দিয়ে পাঠিয়ে দেখুন কোন queue-তে পৌঁছায়।

**২ — DLX + retry pattern implement করুন (৩৫ মিনিট)**
§৫-এর TTL+DLX retry pattern বাস্তবায়ন করুন। একটা consumer বানান যা ইচ্ছাকৃতভাবে প্রথম ২ বার fail করে, তৃতীয়বার সফল হয় — দেখুন message সঠিকভাবে retry queue-তে ঘুরে ফিরে আসছে বাড়তে থাকা delay সহ।

**৩ — Durability টেস্ট (২৫ মিনিট)**
একটা non-durable queue-তে message পাঠান, RabbitMQ container restart করুন, message হারিয়ে গেছে দেখুন। তারপর durable queue + persistent message দিয়ে আবার পরীক্ষা করুন, message বেঁচে থাকা নিশ্চিত করুন।

**৪ — Prefetch-এর প্রভাব পরিমাপ করুন (২০ মিনিট)**
একই কাজ দুইটা ভিন্ন `prefetch_count` (১ বনাম ১০০) দিয়ে চালিয়ে multiple consumer-এর মধ্যে load distribution-এর পার্থক্য দেখুন — কম prefetch-এ ভালো load balancing, বেশি prefetch-এ কম network overhead কিন্তু worse balancing।

---

## ১২. মূল কথা

1. **RabbitMQ queue-based (consume হলে message সরে যায়), Kafka log-based (persist থাকে, offset ট্র্যাক হয়)** — এই একটা পার্থক্য থেকে বাকি সব পার্থক্য (replay, multi-consumer, routing model) বেরিয়ে আসে।
2. **Topic exchange-এর wildcard routing শক্তিশালী কিন্তু ঝুঁকিপূর্ণ** — routing key generation-এ কঠোর validation, প্রতিটা binding-এর automated isolation test বাধ্যতামূলক।
3. **Fanout exchange = broadcast to multiple queue** — একটা event একাধিক independent consumer-এ পাঠানোর RabbitMQ-নেটিভ সমাধান।
4. **DLX + TTL দিয়ে delayed retry pattern** — M11-এর exponential backoff-এর broker-level বাস্তবায়ন, কোনো external scheduler ছাড়াই।
5. **`requeue=True` সাবধানে ব্যবহার করুন** — non-retryable error-এ ভুলবশত requeue করলে infinite loop, M12-এর poison message সমস্যার সমান্তরাল।
6. **Durability-তে চারটা জিনিস একসাথে দরকার** — durable queue, persistent message (`delivery_mode=2`), publisher confirm, এবং quorum queue (classic mirrored-এর বদলে)।
7. **Quorum queue এখন ডিফল্ট পছন্দ critical queue-তে** — Raft consensus-ভিত্তিক, M08-এর Patroni-র সমান্তরাল নীতি প্রয়োগ।
8. **Broker নির্বাচন access pattern-নির্ভর, "modern বনাম পুরনো" প্রশ্ন না** — RabbitMQ জটিল routing/priority-তে স্বাভাবিক, Kafka replay/high-throughput/multi-consumer-এ স্বাভাবিক।
9. **RabbitMQ-তে native priority queue** — Redis broker-এ (M10/M11) এই feature নেই, শুধু আলাদা queue দিয়ে approximate করা যায়।

---

## পরের Module

**M14 — Event-Driven Architecture Patterns।** M11-M13 জুড়ে আমরা task queue আর message broker-এর mechanism শিখেছি। এখন এই সব একসাথে একটা architectural style-এ সংশ্লেষিত হবে: event বনাম command বনাম message-এর পার্থক্য, Transactional Outbox pattern (যেটা আমরা M31 থেকে বারবার উল্লেখ করেছি, কিন্তু কখনো সম্পূর্ণ বিস্তারিত করিনি) ও Inbox pattern, idempotency key design-এর সম্পূর্ণ কাঠামো, Saga pattern (M11-এর chain rollback সমস্যার সমাধান), CQRS, আর Event Sourcing — এবং সবচেয়ে গুরুত্বপূর্ণ, কেন বেশিরভাগ টিমের Event Sourcing দরকার নেই।

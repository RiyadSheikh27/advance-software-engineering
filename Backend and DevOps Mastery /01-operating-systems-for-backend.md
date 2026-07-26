# Module 01 — Operating Systems for Backend Engineers

> **Phase 0 — CS Foundation** | এই module M02-M33-এর একটা retroactive foundation —
> M04-এর GIL, M19-এর cgroups, M20-এর OOM behavior — এগুলোর সবকটার আসল ভিত্তি এখানে।
> পরের module: M03 (DSA ও Complexity)

---

## ১. যে "worker crash" আসলে একটা zombie process ছিল

M19 §১-এ আমরা একটা container দেখেছিলাম যেটা OOMKilled হচ্ছিল যদিও Python-এর memory profiling clean ছিল। এবার একটা ভিন্ন, আরও fundamental ঘটনা — একটা Gunicorn deployment-এ ধীরে ধীরে সার্ভারের `/proc`-এ process entry জমতে থাকল, `ps aux` চালালে শত শত entry দেখাচ্ছিল যাদের status `Z` (zombie), কিন্তু কোনো memory বা CPU ব্যবহার করছিল না।

টিম প্রথমে ভেবেছিল এটা M04-এর Python-level memory leak-এর একটা নতুন রূপ। কিন্তু `ps aux | grep Z` চালিয়ে দেখা গেল এগুলো আসলে **exited child process যাদের parent কখনো তাদের exit status collect (`wait()`) করেনি**। Gunicorn-এর একটা custom `worker_exit` hook একটা subprocess spawn করছিল (একটা image-processing script) কিন্তু কখনো সেই subprocess-এর জন্য `wait()` কল করছিল না — প্রতিটা সম্পন্ন subprocess একটা "zombie" হয়ে `/proc`-এ থেকে যাচ্ছিল, শুধু একটা process table entry (কোনো memory না) হিসেবে, যতক্ষণ না parent (Gunicorn worker) নিজেই exit করত।

এই ঘটনাটা এমন একটা layer-এর সমস্যা যেটা M04-এর Python-level profiling কখনো দেখাতে পারবে না (`py-spy`/`memray` Python heap দেখে, kernel process table না) এবং M19-এর cgroup memory accounting-ও দেখাবে না (zombie process কোনো memory ব্যবহার করে না) — এটা সম্পূর্ণ একটা ভিন্ন layer, **kernel-level process lifecycle**। এই module সেই layer-টা কভার করে যা M04, M19, M20-এর "নিচে" থেকে সবকিছুকে সম্ভব করে তোলে।

---

## ২. Process বনাম Thread — M04-এর GIL আলোচনার প্রকৃত ভিত্তি

### ২.১ Process — একটা Isolated Address Space

```
প্রতিটা process পায়:
  - একটা নিজস্ব virtual address space (§৩-এ বিস্তারিত)
  - একটা নিজস্ব file descriptor table
  - একটা নিজস্ব PID (Process ID)
  - Memory অন্য process-এর সাথে ডিফল্টে শেয়ার হয় না (M19-এর container
    isolation-এর মূল ভিত্তি এখানেই)
```

**M11 §৩.১-এর "prefork worker pool"-এর প্রকৃত মেকানিজম:** Gunicorn-এর `--workers 4` মানে ৪টা সম্পূর্ণ আলাদা OS process — প্রতিটার নিজস্ব memory space, নিজস্ব Python interpreter, নিজস্ব GIL (M04 §৪)। এই কারণেই M04-এর GIL একটা process-এর মধ্যে thread-দের সীমাবদ্ধ করে, কিন্তু ৪টা process-এর মধ্যে না — প্রতিটা process independent, OS scheduler প্রতিটাকে আলাদা CPU core-এ সমান্তরালে চালাতে পারে।

### ২.২ Thread — একটা Shared Address Space-এর মধ্যে Independent Execution

```
একটা process-এর ভেতরের সব thread শেয়ার করে:
  - Memory space (heap, global variable) — M04-এর GIL এই shared
    state-কে race condition থেকে রক্ষা করে
  - File descriptor table
  - কিন্তু প্রতিটা thread-এর নিজস্ব: stack, program counter, register
```

**M04 §৩-এর "প্রতিটা row-এ ১৬+ বাইট overhead" নীতির থ্রেড সংস্করণ:** thread তৈরি করা process তৈরির চেয়ে **সস্তা** কারণ memory space নতুন করে allocate করতে হয় না, শুধু একটা নতুন stack + kernel-level bookkeeping — এটাই M11 §৩.১-এর gevent/thread pool-এর efficiency-র মূল কারণ, prefork-এর তুলনায়।

### ২.৩ Context Switch — একটা লুকানো কিন্তু বাস্তব খরচ

```
Context switch: CPU একটা process/thread থেকে আরেকটায় সুইচ করার সময়
  যা করতে হয়:
    - বর্তমান process-এর register state সেভ করা
    - নতুন process-এর register state লোড করা
    - (process switch হলে) memory mapping (TLB) flush/reload করা —
      এটাই process switch-কে thread switch-এর চেয়ে ব্যয়বহুল করে

আনুমানিক খরচ: থ্রেড switch ~১-৪ মাইক্রোসেকেন্ড, process switch ~৫-১০×
বেশি (TLB flush-এর কারণে)
```

**M20-এর CPU throttling আলোচনার একটা লুকানো contributing factor:** যদি একটা container-এ CPU limit খুব কম হয় কিন্তু thread/process সংখ্যা বেশি (M04-এর `WEB_CONCURRENCY` সতর্কতার ঠিক সেই পরিস্থিতি), OS scheduler-কে ক্রমাগত context switch করতে হয় সেই সীমিত CPU সময়ের জন্য অনেকগুলো thread-এর মধ্যে প্রতিযোগিতা সামলাতে — এই context-switch overhead নিজেই কার্যকর CPU capacity-র একটা অংশ খেয়ে ফেলে, M20 §১-এর ঘটনায় "worker সংখ্যা বাড়ালেও latency খারাপ" রহস্যের একটা অতিরিক্ত স্তর।

> **Senior Tip:** "কখন thread ব্যবহার করবেন, কখন process (multiprocessing)?" — "M04 §৩-এর GIL নীতির সরাসরি প্রয়োগ: I/O-bound কাজে thread (GIL ছাড়া হয়, M04 §৪.২), CPU-bound কাজে process (GIL সম্পূর্ণ এড়িয়ে সমান্তরাল CPU ব্যবহার, কিন্তু context switch এবং memory overhead বেশি)। এই OS-level trade-off-টাই M11-এর Celery pool selection (prefork বনাম gevent) সিদ্ধান্তের প্রকৃত ভিত্তি।"

---

## ৩. Virtual Memory ও Page Cache — M19-এর cgroup Memory Accounting-এর নিচে

### ৩.১ Virtual বনাম Physical Address

```
প্রতিটা process তার নিজস্ব "virtual" address space দেখে (যেমন 0 থেকে
2^64), কিন্তু এটা সরাসরি physical RAM-এর সাথে ম্যাপ করা না — kernel
(page table দিয়ে) virtual address-কে actual physical address-এ
translate করে, প্রতিটা memory access-এ
```

**M19 §১-এর ঘটনার আরও গভীর ব্যাখ্যা:** M19-এ আমরা দেখেছিলাম cgroup memory accounting Python heap-এর চেয়ে বেশি জিনিস ধরে (tmpfs, page cache)। এখন কারণটা স্পষ্ট — **page cache** হলো kernel-এর একটা optimization: disk থেকে পড়া data RAM-এ cache করে রাখা (M10-এর cache-aside pattern-এর kernel-level সংস্করণ, application code লেখার আগেই OS নিজে এটা করে) যাতে পরের বার একই data দ্রুত পাওয়া যায়। এই cache **cgroup memory limit-এর মধ্যে গণনা** হয়, যদিও এটা "প্রয়োজনে ছেড়ে দেওয়া যায়" (M07-এর VACUUM-এর মতো, kernel প্রয়োজনে page cache reclaim করে)।

### ৩.২ Copy-on-Write — M04-এর `gc.freeze()` আলোচনার সম্পূর্ণ প্রকৃত মেকানিজম

```
fork() করার সময়, kernel সন্তান (child) process-কে বাবা-মা (parent)-র
পুরো memory copy করে দেয় না তাৎক্ষণিকভাবে — বরং parent আর child একই
physical memory page শেয়ার করে, যতক্ষণ না কেউ সেই page-এ **লেখে**।
লেখার মুহূর্তে (এবং তখনই), kernel সেই একটা page copy করে ("copy-on-write")
```

**M04 §৩.৩-এর `gc.freeze()`-এর সম্পূর্ণ, নিচের-স্তরের ব্যাখ্যা:** M04-এ আমরা বলেছিলাম Gunicorn `preload_app=True` দিয়ে master process-এ Django load করে, তারপর `fork()` করে worker বানায় — এবং GC যখন object scan করে (প্রতিটা object-এর header-এ লেখে), এটা copy-on-write ট্রিগার করে, shared memory আলাদা হয়ে যায়। এখন আমরা জানি **ঠিক কেন**: `fork()`-এর ঠিক পরে, সব worker parent-এর সাথে physical memory page শেয়ার করছিল (কোনো actual copy হয়নি এখনো) — কিন্তু GC যেই সেই page-এ **লিখল** (header বদলাতে), kernel বাধ্য হলো সেই page copy করতে, worker-প্রতি shared memory হারিয়ে গেল। `gc.freeze()` GC-কে সেই object-গুলো আর scan (এবং তাই write) না করতে বলে, copy-on-write ট্রিগার হওয়া থেকে বাঁচায়।

### ৩.৩ Page Fault — Latency-র একটা লুকানো উৎস

```
Minor page fault: page RAM-এ আছে, কিন্তু এই process-এর page table-এ
                   এখনো mapped না (দ্রুত, ~মাইক্রোসেকেন্ড)
Major page fault: page RAM-এ নেই, disk থেকে আনতে হবে (ধীর, ~মিলিসেকেন্ড
                   — M07-এর disk I/O-র সমতুল্য latency)
```

**M02-এর latency table-এর একটা missing layer:** M02 §১-এ আমরা L1 cache, RAM, SSD, disk-এর latency দেখেছিলাম — কিন্তু "major page fault" হলো সেই RAM-বনাম-disk latency gap-টা **memory access**-এর মধ্যে প্রকাশ পাওয়ার মেকানিজম। যদি একটা process-এর জন্য বরাদ্দ physical memory (M20-এর resource limit) খুব কম হয়, OS কিছু page swap out করতে বাধ্য হয় disk-এ (M07-এর disk I/O-র মতোই ধীর) — একটা "memory-only" operation হঠাৎ disk-latency-level ধীর হয়ে যেতে পারে, M20-এর "CPU/memory normal কিন্তু ধীর" রহস্যের আরেকটা সম্ভাব্য উৎস।

---

## ৪. Blocking বনাম Non-Blocking I/O — M04-এর asyncio-র প্রকৃত Kernel ভিত্তি

### ৪.১ Blocking I/O — ডিফল্ট আচরণ

```python
data = socket.recv(1024)   # ⚠️ blocking — thread এখানে "ঘুমিয়ে" থাকে
                             # kernel data না দেওয়া পর্যন্ত, M04-এর
                             # "GIL ছাড়া হয়" ঠিক এই মুহূর্তে ঘটে
```

**M04 §৪.২-এর GIL release-এর প্রকৃত kernel-level কারণ:** যখন একটা thread `socket.recv()` কল করে, এটা kernel-কে জিজ্ঞেস করছে "data আছে?" — যদি না থাকে, kernel সেই thread-কে "blocked" state-এ রাখে (scheduler সেটাকে আর CPU দেয় না, যতক্ষণ না data আসে) এবং **CPU অন্য কোনো runnable thread/process-কে দেয়**। Python-এ, এই blocking syscall-এর সময়ই GIL release হয় (M04 §৪.২-এর টেবিল) — কারণ interpreter জানে এই thread এখন kernel-এ "আটকে" আছে, তাই অন্য Python thread-কে চালানো নিরাপদ।

### ৪.২ epoll — M04-এর Event Loop-এর প্রকৃত Kernel Primitive

```c
// M04 §৬.১-এর "event loop epoll ব্যবহার করে ready fd জানতে" — এটাই সেই
// syscall, সরলীকৃত
epoll_wait(epoll_fd, events, max_events, timeout);
// একটা single syscall দিয়ে হাজার হাজার socket-এর মধ্যে কোনগুলো
// "ready" (data আছে, বা write করা যাবে) তা জানা যায় — প্রতিটা
// socket আলাদাভাবে চেক করার (O(n)) দরকার নেই
```

**M04 §৬.১-এর event loop diagram-এর প্রকৃত kernel implementation:** M04-এ আমরা event loop-কে "selector-এ (epoll) register করে" বলেছিলাম, বিস্তারিত ছাড়া। `epoll` (Linux-নির্দিষ্ট; macOS/BSD-তে `kqueue`, ধারণাগতভাবে অভিন্ন) হলো সেই kernel mechanism যা asyncio-কে **হাজার হাজার connection একটা single thread দিয়ে** মনিটর করতে দেয় — প্রতিটা connection-এ আলাদা thread (M02-এর C10K সমস্যা) ছাড়াই, কারণ kernel নিজেই "কোন socket-এ কিছু ঘটেছে" ট্র্যাক করে এবং একটা single, efficient কল দিয়ে জানায়।

> **Senior Tip:** "কেন Node.js/asyncio হাজার হাজার concurrent connection handle করতে পারে, কিন্তু thread-per-connection মডেল পারে না?" — "M01 §২.২-এর thread memory overhead (প্রতিটা thread-এর নিজস্ব stack, সাধারণত কয়েক MB ডিফল্টে) আর M01 §২.৩-এর context switch খরচ একসাথে — ১০,০০০ thread মানে ১০,০০০ stack (কয়েক GB) আর OS scheduler-এর ক্রমাগত context switch overhead। `epoll`-ভিত্তিক event loop-এ একটামাত্র thread, একটামাত্র stack, আর kernel নিজে efficiently জানায় কোন socket-এ attention দরকার — M02-এর C10K সমস্যার প্রকৃত সমাধান এই kernel primitive-এই লুকিয়ে।"

---

## ৫. Signal ও Zombie Process — §১-এর ঘটনার সম্পূর্ণ সমাধান

### ৫.১ Signal — M19 §২.১-এর সম্পূর্ণ প্রেক্ষাপট

```
M19 §২.১-এ আমরা SIGTERM/SIGKILL দেখেছিলাম shell-form CMD সমস্যার
প্রেক্ষাপটে। সম্পূর্ণ তালিকা:

SIGTERM (15): "দয়া করে বন্ধ হও" — catchable, graceful shutdown-এর সুযোগ
SIGKILL (9): "এখনই মরো" — uncatchable, M20-এর OOM killer ঠিক এভাবেই মারে
SIGCHLD: "তোমার একটা child process-এর state বদলেছে" — §৫.২-এর zombie
          সমস্যার মূল চাবিকাঠি
SIGHUP: ঐতিহাসিকভাবে "terminal বন্ধ হয়েছে," আধুনিক ব্যবহারে প্রায়ই
        "config reload করো" (M20-এর graceful config update-এর একটা
        প্রক্রিয়া)
```

### ৫.২ Zombie Process — §১-এর ঘটনার সম্পূর্ণ ব্যাখ্যা এবং সমাধান

```
একটা child process exit করলে, তার exit status kernel-এ ধরে রাখা হয়
যতক্ষণ না parent সেটা "collect" করে (wait()/waitpid() syscall দিয়ে)।
এই "exited কিন্তু collect হয়নি" অবস্থাটাই zombie (Z status) — এটা
কোনো memory/CPU ব্যবহার করে না (§১-এ যেমন observed হয়েছিল), শুধু
একটা process table entry
```

```python
import subprocess

# ❌ §১-এর ঘটনার মূল কারণ — subprocess তৈরি, কিন্তু কখনো wait() না
def worker_exit(server, worker):
    subprocess.Popen(["python", "cleanup_script.py"])
    # ⚠️ এই process exit করলে zombie হয়ে থাকবে, Gunicorn worker
    #    কখনো collect না করলে

# ✅ সঠিক — wait() কল করা, বা subprocess.run() (যা নিজেই wait করে)
def worker_exit(server, worker):
    subprocess.run(["python", "cleanup_script.py"], timeout=30)
```

**§১-এর ঘটনার root cause:** `subprocess.Popen()` একটা child process spawn করে কিন্তু immediately return করে (non-blocking) — parent কখনো `wait()` কল না করলে, child exit করার পরেও তার exit status kernel-এ আটকে থাকে, একটা zombie হিসেবে। `subprocess.run()` (বা explicit `process.wait()`) parent-কে child-এর সমাপ্তির জন্য অপেক্ষা করায় এবং exit status collect করে, zombie তৈরি হতে বাধা দেয়।

> **Senior Tip:** "কেন zombie process কখনো OOM killer দিয়ে মারা যায় না?" — "কারণ zombie কোনো memory ব্যবহার করে না (§১-এ M19-এর profiling clean দেখানোর কারণ) — এটা শুধু kernel-এর process table-এ একটা entry, যেটার একটা নির্দিষ্ট, সীমিত সংখ্যা থাকে (`pid_max`)। যদি zombie জমতে থাকে এবং কখনো clean না হয়, একসময় সিস্টেম নতুন process তৈরি করতে পারবে না ('cannot fork' error) — একটা সম্পূর্ণ ভিন্ন ধরনের resource exhaustion, M20-এর memory/CPU limit থেকে আলাদা, কিন্তু একই রকম বিপর্যয়কর।"

---

## ৬. OOM Killer ও Load Average — M20-এর ঘটনার Kernel-Level ভিত্তি

### ৬.১ OOM Killer Scoring — M19 §২.৩-এর সম্পূর্ণ বিস্তারিত

```bash
cat /proc/<pid>/oom_score        # M19-এ উল্লেখিত, এখানে সম্পূর্ণ প্রেক্ষাপট
```

Kernel প্রতিটা process-এর একটা "badness" score গণনা করে (memory ব্যবহার, runtime, `oom_score_adj`-এর ভিত্তিতে) — memory শেষ হয়ে গেলে (system-wide বা cgroup-wide, M19-এর container প্রসঙ্গে), সবচেয়ে বেশি score-এর process মারা হয়। এটাই M20-এর `OOMKilled` status-এর প্রকৃত kernel decision-making, যেটা M19-এ আমরা phenomenon হিসেবে দেখেছিলাম।

### ৬.২ Load Average — একটা প্রায়ই ভুল বোঝা Metric

```bash
uptime
# load average: 2.5, 1.8, 1.2   (১, ৫, ১৫ মিনিটের গড়)
```

```
❌ ভুল ধারণা: "load average = CPU utilization %"
✅ প্রকৃত অর্থ: load average = গড় কতগুলো process "runnable" ছিল
   (CPU-তে চলছে, অথবা CPU-র জন্য অপেক্ষা করছে, অথবা কিছু OS-এ
   uninterruptible I/O wait-এও অপেক্ষা করছে)

একটা 4-core মেশিনে load average 2.0 মানে গড়ে ২টা process
"runnable" ছিল — CPU-র অর্ধেক ব্যবহৃত (approximately)। কিন্তু
load average 8.0 একই মেশিনে মানে CPU সম্পূর্ণ saturated, প্রায়
দ্বিগুণ কাজ অপেক্ষা করছে যতটা মেশিন এখনই করতে পারে
```

**M24 §৩.৩-এর USE Method-এর Saturation metric-এর প্রকৃত kernel-level উৎস:** M24-এ আমরা "Saturation" (queued/waiting কাজ) কে গুরুত্বপূর্ণ metric হিসেবে চিহ্নিত করেছিলাম, শুধু Utilization না। Load average **ঠিক এই Saturation** measure করে — CPU utilization ৭০% দেখালেও (M20 §১-এর ঘটনার মতো), load average যদি core সংখ্যার চেয়ে অনেক বেশি হয়, এটা নির্দেশ করে অনেক process CPU-র জন্য অপেক্ষা করছে (M01 §২.৩-এর context switch overhead-ও বেড়ে যাচ্ছে সমান্তরালে)।

⚠️ **Linux-এর একটা সূক্ষ্মতা:** load average-এ **uninterruptible I/O wait** (disk-এর জন্য অপেক্ষারত process, `D` state)-ও গণনা হয় — তাই একটা উচ্চ load average মানে সবসময় CPU-bottleneck না, এটা disk I/O bottleneck-ও (M07-এর slow query-তে disk-wait) নির্দেশ করতে পারে। `vmstat`/`iostat` দিয়ে পার্থক্য করা যায় (নিচে §৭)।

---

## ৭. Debugging Toolkit — M25-এর Symptom-to-Layer Table-এ একটা নতুন Row

```bash
# Process ও memory
top / htop                    # real-time CPU/memory usage
ps aux | grep Z                # zombie process খোঁজা (§৫.২)
cat /proc/<pid>/status         # নির্দিষ্ট process-এর বিস্তারিত state

# Memory ও I/O
vmstat 1                       # প্রতি সেকেন্ডে system-wide memory/CPU/IO
iostat -x 1                    # disk I/O utilization, %util column
                                # গুরুত্বপূর্ণ (M07-এর disk bottleneck সনাক্তে)

# System call ও file access
strace -p <pid>                # একটা process কী syscall করছে, real-time
                                # (M02-এর network call, M07-এর file I/O
                                # সরাসরি দেখা যায়)
lsof -p <pid>                  # একটা process-এর সব খোলা file
                                # descriptor (M02 §২.২-এর ulimit সমস্যা
                                # ডিবাগ করতে)

# CPU profiling
perf top                       # kernel-level, cross-language CPU profiling
                                # (M04-এর py-spy-এর একটা language-agnostic,
                                # নিচু-স্তরের সংস্করণ)
```

**M25 §৫.৩-এর toolkit-এ একটা missing layer যোগ করা:** M25-এ আমরা `py-spy`/`EXPLAIN`/`kubectl`/`curl` দেখেছিলাম — এই সবগুলোই application, database, container, network layer। `strace`/`lsof`/`vmstat` হলো **এর নিচের**, সরাসরি kernel-interaction layer, যেটা প্রয়োজন হয় যখন উপরের সব layer "স্বাভাবিক" দেখায় কিন্তু সমস্যা এখনো আছে (§১-এর zombie ঘটনার মতো)।

> **Senior Tip:** "কখন `strace` ব্যবহার করবেন?" — "যখন M04-এর `py-spy dump` দেখায় worker একটা নির্দিষ্ট syscall-এ 'আটকে' আছে, কিন্তু কেন সেটা স্পষ্ট না। `strace -p <pid>` সরাসরি দেখায় ঠিক কোন syscall চলছে (connect, read, futex) এবং সেটা কতক্ষণ ধরে ব্লক আছে — এটা M02-এর network-level debugging আর M04-এর application-level debugging-এর মাঝের একটা layer, যেটা প্রায়ই উপেক্ষিত কিন্তু production incident-এ (M25) অত্যন্ত মূল্যবান।"

---

## ৮. cgroups ও Namespaces — M19-এর Container Isolation-এর সম্পূর্ণ Kernel ভিত্তি

### ৮.১ cgroups — Resource Limiting

```
M19 §৮-এ আমরা cgroups-কে "container resource limit-এর মেকানিজম"
হিসেবে ব্যবহার করেছিলাম। প্রকৃত সংজ্ঞা: cgroups (control groups)
হলো একটা kernel feature যা process-দের group করে এবং সেই group-এর
জন্য resource (CPU, memory, I/O) সীমাবদ্ধ/measure/prioritize করে —
Docker/Kubernetes কোনো নতুন isolation প্রযুক্তি আবিষ্কার করেনি,
তারা এই বিদ্যমান kernel feature ব্যবহার করে
```

### ৮.২ Namespaces — Visibility Isolation

```
cgroups সীমাবদ্ধ করে "কতটুকু" resource ব্যবহার করা যায়
Namespaces সীমাবদ্ধ করে "কী দেখা যায়" — M19-এর container-এর ভেতরে
`ps aux` চালালে শুধু সেই container-এর process দেখা যায়, host-এর
বাকি process না — এটা PID namespace-এর কারণে

Network namespace: M19 §৫-এর "প্রতিটা container-এর নিজস্ব network
                     stack" এর প্রকৃত মেকানিজম
Mount namespace: প্রতিটা container তার নিজস্ব filesystem view দেখে
                  (M19-এর "container filesystem ephemeral" নীতির ভিত্তি)
```

**M19-এর "container একটা process না, একটা isolation boundary" নীতির সম্পূর্ণ, প্রকৃত সংজ্ঞা:** একটা "container" আসলে **একটা সাধারণ Linux process**, শুধু একটা নির্দিষ্ট cgroup (resource limit) এবং namespace set (isolated view)-এর মধ্যে চলছে — কোনো জাদু ভার্চুয়ালাইজেশন প্রযুক্তি না, শুধু kernel-এর দুইটা feature-এর সুচিন্তিত সমন্বয়। এই কারণেই container-রা VM-এর চেয়ে অনেক হালকা (M19-এর "শুরু হতে সেকেন্ড লাগে, VM-এর মিনিটের বিপরীতে") — কোনো আলাদা kernel boot করতে হয় না, host kernel-ই শেয়ার হয়, শুধু isolation view আলাদা।

> **Senior Tip:** "Container আর VM-এর মধ্যে fundamental পার্থক্য কী?" — "VM একটা সম্পূর্ণ, আলাদা kernel চালায় (hypervisor দিয়ে virtualized hardware-এ) — সম্পূর্ণ isolation, কিন্তু ভারী (প্রতিটা VM-এর নিজস্ব kernel boot, কয়েক GB RAM base overhead)। Container host kernel-ই শেয়ার করে, শুধু M01 §৮-এর cgroups (resource limit) + namespaces (visibility) দিয়ে isolation তৈরি করে — অনেক হালকা, কিন্তু isolation VM-এর চেয়ে দুর্বল (একটা kernel-level vulnerability সব container-কে প্রভাবিত করতে পারে, M26-এর 'non-root user' সুপারিশের একটা মূল কারণ এটাই)।"

---

## ৯. Interview Section

### প্রশ্ন ১ (Senior) — "Process আর thread-এর মধ্যে পার্থক্য কী, এবং এটা M04-এর GIL আলোচনার সাথে কীভাবে সম্পর্কিত?"

**🌟 Senior/Staff Answer**
> "মৌলিক পার্থক্য হলো memory isolation — প্রতিটা process তার নিজস্ব virtual address space পায় (M01 §৩.১), thread-রা একই process-এর মধ্যে memory space শেয়ার করে। এই পার্থক্যটাই সরাসরি M04-এর GIL discussion-এর ভিত্তি — GIL একটা **single process**-এর মধ্যে Python bytecode execution serialize করে, কারণ সেই process-এর সব thread একই memory (এবং তাই refcount) শেয়ার করছে, যা atomic না হলে race condition তৈরি করবে।
>
> এই কারণেই M11-এর Celery `prefork` pool (একাধিক process) GIL সম্পূর্ণ এড়িয়ে যায় — প্রতিটা process-এর নিজস্ব, স্বাধীন GIL আছে, তাই সত্যিকারের সমান্তরাল CPU ব্যবহার সম্ভব। কিন্তু এই isolation-এর একটা খরচ আছে — process তৈরি করা thread তৈরির চেয়ে ব্যয়বহুল (সম্পূর্ণ নতুন memory space allocate করতে হয়, যদিও M04-এর `fork()`-এ copy-on-write এই খরচ কমায় শুরুতে), আর process-দের মধ্যে communication (IPC) memory-শেয়ারিং থ্রেডের চেয়ে জটিল।"

---

### প্রশ্ন ২ (Staff / Production Incident) — "একটা সার্ভারে ধীরে ধীরে হাজার হাজার zombie process জমে যাচ্ছে, কিন্তু memory/CPU normal। এটা কি একটা সমস্যা, এবং কেন?"

**🌟 Senior/Staff Answer**
> "হ্যাঁ, এটা একটা প্রকৃত সমস্যা, যদিও memory/CPU impact নেই — zombie process কোনো memory ব্যবহার করে না (§১-এর ঘটনায় যেমন observed), শুধু একটা kernel process-table entry। কিন্তু এই process table-এর একটা **সীমিত সংখ্যক entry** থাকে (`pid_max`), আর প্রতিটা zombie সেই সীমিত সংখ্যার একটা স্লট দখল করে রাখে যতক্ষণ না parent process সেটা `wait()` দিয়ে collect করে।
>
> যদি zombie জমতে থাকে এবং কখনো clean না হয়, একসময় সিস্টেম নতুন process **তৈরিই করতে পারবে না** — 'cannot fork: resource temporarily unavailable' error, যেটা M20-এর memory/CPU limit থেকে সম্পূর্ণ ভিন্ন, কিন্তু সমানভাবে বিপর্যয়কর একটা resource exhaustion।
>
> Root cause প্রায় সবসময় একটা parent process যে child spawn করে কিন্তু কখনো তাদের exit status collect করে না (M01 §৫.২-এর `subprocess.Popen()` বনাম `subprocess.run()` উদাহরণ)। সমাধান হলো কোড অডিট করা — যেকোনো জায়গায় `subprocess.Popen()`, `os.fork()`, বা অনুরূপ child-spawning call আছে, নিশ্চিত করা সেখানে একটা matching `wait()`/`waitpid()` কল আছে, অথবা `SIGCHLD` handler properly zombie reap করছে।"

---

### প্রশ্ন ৩ (Coding / Debugging) — "একটা high-concurrency API server-এ, load average core সংখ্যার (৮) চেয়ে অনেক বেশি (৩৫) দেখাচ্ছে, কিন্তু CPU utilization মাত্র ৪০%। এটা কীভাবে সম্ভব, এবং root cause কী হতে পারে?"

**🌟 Senior/Staff Answer**
> "এটা M01 §৬.২-এর load average-এর সবচেয়ে গুরুত্বপূর্ণ সূক্ষ্মতা প্রকাশ করে — Linux-এ load average শুধু CPU-runnable process গণনা করে না, **uninterruptible I/O wait** (disk-এর জন্য অপেক্ষারত process, `D` state)-ও গণনা করে। CPU utilization কম কিন্তু load average বেশি মানে অনেক process CPU-র জন্য অপেক্ষা করছে না, বরং **disk I/O-র জন্য** অপেক্ষা করছে।
>
> **আমার debugging ক্রম:** প্রথমে `vmstat 1` চালিয়ে `b` column (blocked processes) দেখব — যদি এটা বড় হয়, নিশ্চিত হয় I/O bottleneck। তারপর `iostat -x 1` দিয়ে disk `%util` দেখব — যদি এটা ১০০%-এর কাছাকাছি, disk saturated।
>
> সম্ভাব্য root cause: M07-এর একটা query যা index miss করছে (seq scan, বিশাল disk read), বা M07-এর VACUUM চলছে একটা বড় টেবিলে (heavy I/O), অথবা M08-এর একটা log/audit table যেখানে write volume disk-এর throughput ছাড়িয়ে গেছে। M24-এর USE method-এর ভাষায় — এটা 'Saturation' একটা resource-এ (disk) যেটা 'Utilization' metric (CPU) দিয়ে সম্পূর্ণ invisible ছিল, ঠিক M20 §১-এর ঘটনার মতো একটা layer-mismatch — ভুল metric দেখা মানে ভুল সিদ্ধান্তে পৌঁছানো (এখানে হয়তো কেউ CPU core বাড়ানোর প্রস্তাব দিত, যা এই সমস্যা সমাধান করত না, কারণ bottleneck disk-এ, CPU-তে না)।"

---

### প্রশ্ন ৪ (Architecture Decision) — "আমাদের একটা নতুন workload-এ, একটা সার্ভারে Docker container চালাব নাকি একটা full VM? Trade-off কী?"

**🌟 Senior/Staff Answer**
> "এই সিদ্ধান্ত M01 §৮-এর isolation mechanism পার্থক্যের একটা সরাসরি প্রয়োগ, M19/M21-এর infrastructure সিদ্ধান্তের একটা foundational স্তর।
>
> Container (cgroups + namespaces) হালকা এবং দ্রুত (M19-এর 'সেকেন্ডে শুরু হয়') কারণ host kernel শেয়ার হয়, কিন্তু isolation দুর্বলতর — একটা kernel-level vulnerability theoretically একটা container থেকে host বা অন্য container-এ প্রভাব ফেলতে পারে (M26-এর non-root user, M19-এর multi-stage build দিয়ে attack surface কমানো — এই সবগুলোই এই দুর্বলতর isolation-কে compensate করার প্রচেষ্টা)।
>
> VM (hypervisor-based, সম্পূর্ণ আলাদা kernel) শক্তিশালী isolation দেয় — একটা VM-এর মধ্যে একটা vulnerability সাধারণত অন্য VM বা host-কে সরাসরি প্রভাবিত করে না, কারণ তারা সত্যিকারের আলাদা kernel চালাচ্ছে। কিন্তু এই isolation-এর খরচ ভারী — প্রতিটা VM-এ নিজস্ব kernel boot overhead, বেশি base memory usage, ধীর startup।
>
> **আমার সুপারিশ:** M31-এর payment platform-এর মতো একটা multi-tenant system-এ, যেখানে বিভিন্ন merchant-এর workload একসাথে চলছে (M08-এর shared schema multi-tenancy-র মতোই একটা isolation প্রশ্ন), যদি একটা নির্দিষ্ট enterprise customer সম্পূর্ণ physical isolation দাবি করে (M08 §৭.১-এর dedicated database আলোচনার সমান্তরাল), তাদের workload একটা dedicated VM-এ রাখা যুক্তিসঙ্গত, শুধু premium tier হিসেবে। সাধারণ, ট্রাস্টেড internal workload-এ (M19-এর নিজস্ব containerized microservice), container-ই যথেষ্ট এবং dramatically বেশি cost-efficient (M21-এর FinOps নীতি) — সব workload-এ VM-level isolation করা একটা unnecessary খরচ যেখানে measured প্রয়োজন নেই (M09-এর checklist-এর infrastructure-isolation সংস্করণ)।"

---

## ১০. হাতে-কলমে অনুশীলন

**১ — Zombie process তৈরি ও পর্যবেক্ষণ করুন (২৫ মিনিট)**
একটা Python script লিখুন যা `subprocess.Popen()` দিয়ে child spawn করে কিন্তু কখনো `wait()` করে না। `ps aux | grep Z` দিয়ে zombie তৈরি হওয়া দেখুন। তারপর `subprocess.run()` দিয়ে ঠিক করে zombie তৈরি না হওয়া নিশ্চিত করুন।

**২ — Context switch overhead পরিমাপ করুন (৩০ মিনিট)**
একটা CPU-bound কাজ (M04-এর `sum(i*i ...)`) ১, ১০, ১০০টা thread দিয়ে চালিয়ে সময় তুলনা করুন একটা limited-core মেশিনে (বা `taskset` দিয়ে CPU affinity সীমিত করে)। থ্রেড সংখ্যা বাড়ার সাথে context-switch overhead-এর প্রভাব দেখুন।

**৩ — Copy-on-write নিজের চোখে দেখুন (৩০ মিনিট)**
একটা বড় list তৈরি করে `fork()` করুন (Python `os.fork()` দিয়ে, Linux/macOS-এ)। Parent আর child-এর `/proc/<pid>/status`-এ `VmRSS` (actual physical memory) তুলনা করুন fork-এর ঠিক পরে এবং child সেই list-এ কিছু লেখার পরে — copy-on-write-এর প্রভাব measure করুন।

**৪ — Load average বনাম CPU utilization পার্থক্য করুন (২৫ মিনিট)**
একটা script লিখুন যা অনেক file I/O করে (disk-heavy, CPU-light)। চালানোর সময় `uptime` (load average) আর `top` (CPU utilization) পাশাপাশি দেখুন — পার্থক্যটা lক্ষ্য করুন এবং `iostat`-এ disk `%util` verify করুন।

---

## ১১. মূল কথা

1. **Process memory isolation দেয়, thread শেয়ার করে** — M04-এর GIL শুধু একটা process-এর ভেতরে প্রযোজ্য, একাধিক process জুড়ে না (M11-এর prefork-এর ভিত্তি)।
2. **Context switch একটা লুকানো কিন্তু বাস্তব খরচ** — process switch thread switch-এর চেয়ে ব্যয়বহুল (TLB flush-এর কারণে)।
3. **Copy-on-write M04-এর `gc.freeze()` optimization-এর প্রকৃত kernel মেকানিজম** — GC-এর object header লেখা copy-on-write ট্রিগার করে, shared memory হারায়।
4. **Blocking syscall-এর সময়ই GIL release হয়** — M04-এর "I/O-তে GIL ছাড়া হয়" নীতির প্রকৃত kernel-level কারণ।
5. **epoll একটা single thread-কে হাজার হাজার connection মনিটর করতে দেয়** — M04-এর event loop-এর প্রকৃত kernel primitive, C10K সমস্যার সমাধান।
6. **Zombie process memory ব্যবহার করে না, কিন্তু process table entry নেয়** — জমতে থাকলে "cannot fork" outage তৈরি করতে পারে, M19/M20-এর memory-based OOM থেকে সম্পূর্ণ ভিন্ন failure mode।
7. **Load average CPU utilization না** — এটা runnable + uninterruptible-I/O-wait process গণনা করে, তাই কম CPU utilization সহ উচ্চ load average disk bottleneck নির্দেশ করে।
8. **cgroups resource সীমাবদ্ধ করে, namespaces visibility সীমাবদ্ধ করে** — এই দুইটার সমন্বয়ই "container", কোনো নতুন virtualization প্রযুক্তি না।
9. **Container VM-এর চেয়ে হালকা কারণ kernel শেয়ার করে**, কিন্তু সেই কারণেই isolation দুর্বলতর — M26-এর non-root user/attack surface সতর্কতার মূল কারণ।
10. **`strace`/`vmstat`/`iostat` M25-এর debugging toolkit-এর একটা missing, kernel-level layer** — যখন application/database/container layer সব "normal" দেখায় কিন্তু সমস্যা এখনো বিদ্যমান।

---

## পরের Module

**M03 — DSA ও Complexity for Backend।** আজ আমরা OS-level foundation দেখলাম যা M04, M19, M20-এর নিচে ছিল। পরের module-এ আমরা একটা ভিন্ন ধরনের foundation দেখব — data structure এবং algorithm যা M10 (Bloom filter, LRU, rate limiter), M12 (consistent hashing-এর ভিত্তি), M29 (Merkle tree), এবং M32 (geohash)-এ ব্যবহৃত হয়েছে কিন্তু কখনো from-scratch derive করা হয়নি।

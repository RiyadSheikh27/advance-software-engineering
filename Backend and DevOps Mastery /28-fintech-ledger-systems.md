# Module 28 — FinTech ও Ledger Systems

> **Phase H — Domain Specializations** | পূর্বশর্ত: M07, M08, M14, M18, M26
> পরের module: M29 (Blockchain Backend)

---

## ১. যে ৳০.০১-এর গরমিল একটা $২০ মিলিয়নের সিস্টেমকে থামিয়ে দিয়েছিল

M31-এর payment platform প্রায় দুই বছর সফলভাবে চলার পর, একটা quarterly financial audit-এ একটা reconciliation report ৳০.০১ (এক পয়সা) গরমিল দেখাল — merchant-দের কাছে PSP যা settle করেছে, আর internal ledger-এ যা দেখানো হয়েছে, তার মধ্যে। মোটের অঙ্কে এটা তুচ্ছ, কিন্তু finance team **পুরো payout প্রক্রিয়া বন্ধ** করে দিল যতক্ষণ না কারণ খুঁজে বের করা যায় — কারণ FinTech-এ একটা নিয়ম আছে: **একটা এক পয়সার গরমিলও মানে ledger আর সত্য বলছে না, এবং যদি এক পয়সা ভুল হতে পারে, তাহলে যেকোনো পরিমাণ ভুল হতে পারে।**

তদন্তে যা পাওয়া গেল — এটা M08 §৮.১-এর money-handling নীতির একটা সূক্ষ্ম লঙ্ঘন ছিল। একটা currency conversion function (একটা multi-currency merchant-এর জন্য) `Decimal` ব্যবহার করছিল সঠিকভাবে, কিন্তু একটা intermediate step-এ round করছিল **ভুল precision-এ** — `ROUND_HALF_UP` এর বদলে Python-এর ডিফল্ট `ROUND_HALF_EVEN` (banker's rounding) ব্যবহার হচ্ছিল, আর PSP নিজে `ROUND_HALF_UP` ব্যবহার করত তাদের conversion-এ। ৯৯.৯৯% ক্ষেত্রে এই দুইটা rounding mode একই ফলাফল দেয় — কিন্তু ঠিক `.5` boundary-তে (একটা নির্দিষ্ট amount-এ) তারা ভিন্ন হয়ে যায়, এক পয়সার পার্থক্য তৈরি করে।

এই ঘটনাটা এই module-এর কেন্দ্রীয় নীতি প্রতিষ্ঠা করে: **FinTech-এ "প্রায় সঠিক" মানে "ভুল।"** M04-M27-এর সব engineering discipline (M07-এর ACID, M14-এর outbox, M08-এর money handling, M18-এর invariant enforcement) এই একটা domain-এ তাদের সর্বোচ্চ, সবচেয়ে unforgiving প্রয়োগ খুঁজে পায় — এই module সেই সংশ্লেষণ।

---

## ২. Double-Entry Ledger — M18-এর Aggregate নীতির সম্পূর্ণ আর্থিক প্রয়োগ

### ২.১ মূল ধারণা — কেন Single-Column Balance যথেষ্ট না

```python
# ❌ Naive — শুধু একটা balance column
class Merchant(models.Model):
    balance_minor = models.BigIntegerField()

# সমস্যা: এই একটা সংখ্যা "সত্য" কোথা থেকে এসেছে তার কোনো ইতিহাস নেই।
# M08 §৯.১-এর audit log ছাড়া, "কেন balance এখন ৫০,০০০?" প্রশ্নের কোনো
# উত্তর নেই — শুধু "এখন এটাই" জানা যায়, "কীভাবে এখানে পৌঁছালো" না।
```

```python
# ✅ Double-entry — M18-এর Aggregate/Value Object নীতির সম্পূর্ণ প্রয়োগ
class LedgerEntry(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    account = models.ForeignKey("Account", on_delete=models.PROTECT)
    amount_minor = models.BigIntegerField()   # ধনাত্মক = debit, ঋণাত্মক = credit (convention)
    entry_type = models.CharField(max_length=16)   # "debit" / "credit"
    transaction = models.ForeignKey("LedgerTransaction", on_delete=models.PROTECT)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        # ⚠️ M08 §৯.১-এর audit-immutability নীতি — কখনো UPDATE/DELETE না,
        #    শুধু INSERT। ভুল হলে একটা "reversing entry" যোগ করা হয়,
        #    পুরনো entry কখনো মুছা/বদলানো হয় না
        constraints = [
            models.CheckConstraint(check=models.Q(amount_minor__gt=0), name="positive_amount")
        ]

class LedgerTransaction(models.Model):
    """একটা atomic financial event — M18-এর Aggregate Root"""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    description = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
```

**মূল invariant — M18 §৪.৩-এর Aggregate boundary নীতির সবচেয়ে কঠোর প্রয়োগ:** প্রতিটা `LedgerTransaction`-এ **সব entry-র sum শূন্য হতে হবে** (debit = credit) — এটা double-entry accounting-এর মৌলিক নিয়ম, এবং এটা M18-এর "Aggregate Root সব invariant enforce করে" নীতির সবচেয়ে কঠোর, সবচেয়ে non-negotiable উদাহরণ:

```python
class LedgerTransactionFactory:   # M18 §৫.২-এর Factory pattern-এর সরাসরি প্রয়োগ
    @staticmethod
    def create_transfer(from_account, to_account, amount: Money, description: str):
        with transaction.atomic():   # ⚠️ M07-এর ACID — সব entry একটা transaction-এ
            txn = LedgerTransaction.objects.create(description=description)
            LedgerEntry.objects.create(
                account=from_account, amount_minor=amount.amount_minor,
                entry_type="debit", transaction=txn,
            )
            LedgerEntry.objects.create(
                account=to_account, amount_minor=amount.amount_minor,
                entry_type="credit", transaction=txn,
            )
            # ⚠️ যদি এই দুইটা entry-র sum শূন্য না হয় (debit amount ≠ credit
            #    amount), এটা একটা bug — একটা DB-level CHECK constraint
            #    বা trigger দিয়ে এটা enforce করা উচিত, শুধু application
            #    logic-এর উপর নির্ভর না করে (M07-এর নীতি)
        return txn
```

**M07-এর ACID guarantee-র সরাসরি critical প্রয়োগ:** যদি এই দুইটা `LedgerEntry.objects.create()` কল-এর মাঝে একটা crash হয় (M11-এর at-least-once delivery-র মতো একটা ঝুঁকি, কিন্তু এখানে database transaction-এ), `transaction.atomic()` নিশ্চিত করে **কোনোটাই** persist হবে না — একটা "half-transaction" (শুধু debit, কোনো credit ছাড়া) কখনো ঘটতে পারে না। এটাই double-entry-র পুরো নিরাপত্তা মডেলের ভিত্তি।

### ২.২ Balance Calculation — Running বনাম Snapshot

```python
# ❌ Naive — প্রতিবার সব entry sum করা
def get_balance(account):
    return LedgerEntry.objects.filter(account=account).aggregate(
        total=Sum(Case(
            When(entry_type="credit", then="amount_minor"),
            When(entry_type="debit", then=-F("amount_minor")),
        ))
    )["total"]
# সমস্যা: M07-এর index scan হলেও, লক্ষ লক্ষ entry হলে এই aggregation
# ধীর হয়ে যায় — M07 §৬-এর query optimization নীতি প্রয়োজন
```

```python
# ✅ Snapshot + incremental — M08-এর materialized view নীতির প্রয়োগ
class Account(models.Model):
    cached_balance_minor = models.BigIntegerField(default=0)
    balance_as_of_transaction = models.ForeignKey(
        LedgerTransaction, null=True, on_delete=models.SET_NULL
    )   # ⚠️ কোন transaction পর্যন্ত balance calculate হয়েছে

def get_balance(account):
    # M07-এর covering index নীতির মতো — cached value ব্যবহার + শুধু
    # সাম্প্রতিক entry (cache-এর পরের) যোগ করা, পুরো history স্ক্যান না
    recent_entries = LedgerEntry.objects.filter(
        account=account, transaction_id__gt=account.balance_as_of_transaction_id
    )
    delta = recent_entries.aggregate(...)["total"] or 0
    return account.cached_balance_minor + delta
```

**M07-এর covering index/materialized aggregation নীতির সরাসরি প্রয়োগ:** এটা M31-এর payment dashboard-এর "সর্বশেষ payment" Subquery pattern (M05 §৫.৪)-এর একটা balance-calculation সংস্করণ — পুরো history বারবার scan করার বদলে, একটা checkpoint (cached balance) থেকে শুধু delta যোগ করা। **কিন্তু cached_balance কখনো "সত্য" না** — এটা একটা optimization, প্রকৃত source of truth সবসময় ledger entry-র সম্পূর্ণ history, ঠিক M08-এর read model বনাম write model (CQRS, M14 §৭) পার্থক্যের মতো।

> **Senior Tip:** "Cached balance আর actual ledger sum-এর মধ্যে গরমিল হলে কী করবেন?" — "M28 §১-এর ঘটনার direct প্রয়োগ — এটাকে কখনো 'সামান্য bug' হিসেবে treat করা উচিত না, এটা একটা **তদন্ত-দাবি করা ঘটনা**। প্রথম পদক্ষেপ payout/settlement (§৬) সাময়িকভাবে থামানো (M25-এর 'bleeding থামাও' নীতি), তারপর M25-এর 5 Whys দিয়ে root cause খোঁজা। কখনো cached value-কে 'ঠিক করে' চুপচাপ এগিয়ে যাওয়া উচিত না — যদি cache ভুল হতে পারে একবার, এটা আবার হতে পারে, আর root cause না জানলে সেটা প্রতিরোধ করা যায় না।"

---

## ৩. Payment Idempotency — M31/M11/M14-এর সংশ্লেষণ

### ৩.১ Idempotency-র তিনটা স্তর একসাথে — Payment-এর সম্পূর্ণ Lifecycle

```python
# M31-এর idempotency key (client-facing) + M11-এর task idempotency
# (internal processing) + M14-এর Outbox/Inbox (event-level) — সবকিছু
# একটা payment-এর জীবনচক্রে একসাথে কাজ করছে

def create_payment(merchant, payload, idempotency_key):   # M31-এর স্তর
    existing = Payment.objects.filter(
        merchant=merchant, idempotency_key=idempotency_key
    ).first()
    if existing:
        return existing
    with transaction.atomic():
        payment = Payment.objects.create(status="processing", idempotency_key=idempotency_key, ...)
        OutboxEvent.objects.create(event_type="payment.created", ...)   # M14-এর outbox
    return payment

@shared_task(bind=True, acks_late=True)   # M11-এর স্তর
def charge_via_psp(self, payment_id):
    payment = Payment.objects.get(pk=payment_id)
    if payment.status in ("succeeded", "failed"):   # M11 §৫-এর idempotency check
        return
    result = call_psp(payment.id, payment.amount_minor,
                       idempotency_key=f"psp-charge-{payment.id}")   # PSP-র নিজস্ব idempotency-ও!
    ...
```

**M31 §৯-এর "layered idempotency" নীতির সম্পূর্ণ প্রকাশ:** লক্ষ্য করুন **তিনটা আলাদা** idempotency key ব্যবহৃত হচ্ছে এই একটা payment flow-এ — client-facing (`idempotency_key`, merchant-এর retry থেকে সুরক্ষা), internal task (`payment.status` check, M11-এর at-least-once delivery থেকে সুরক্ষা), এবং **PSP-facing** (`psp-charge-{payment.id}`, যদি আমাদের নিজের retry PSP-কে দুইবার charge করতে বলে)। প্রতিটা স্তর একটা ভিন্ন duplicate-risk address করে — একটা মিস হলেই double-charge সম্ভব।

### ৩.২ PSP-র নিজস্ব Idempotency — Conformist Relationship-এর প্রয়োগ

```python
def call_psp(payment_id, amount_minor, idempotency_key):
    # M17 §৩.২-এর Conformist pattern — PSP তাদের নিজস্ব idempotency
    # header format দাবি করে, আমাদের সেটা মেনে চলতে হয়
    return psp_session.post(PSP_URL, json={"amount": amount_minor},
        headers={"Idempotency-Key": idempotency_key}, timeout=(3.05, 10))
```

**M17-এর Anti-Corruption Layer-এর reverse প্রয়োগ:** M18 §৭-এ আমরা দেখেছিলাম কীভাবে **incoming** PSP data-কে আমাদের vocabulary-তে translate করতে হয়। এখানে বিপরীত দিক — **outgoing** request-এ আমাদের নিজস্ব idempotency key-কে PSP-র expected format-এ translate করা। এই দুই-দিকের translation-ই একটা সম্পূর্ণ Anti-Corruption Layer গঠন করে।

---

## ৪. Webhook Design — M06/M26-এর সম্পূর্ণ Synthesis

### ৪.১ Signing — M26-এর Cryptography আলোচনার Payment-Specific প্রয়োগ

```python
import hmac, hashlib, time

def sign_webhook(payload_bytes: bytes, secret: str) -> str:
    timestamp = str(int(time.time()))
    signed_payload = f"{timestamp}.{payload_bytes.decode()}"
    signature = hmac.new(secret.encode(), signed_payload.encode(), hashlib.sha256).hexdigest()
    return f"t={timestamp},v1={signature}"

def verify_webhook(payload_bytes: bytes, header: str, secret: str, tolerance_sec=300) -> bool:
    parts = dict(p.split("=") for p in header.split(","))
    timestamp, signature = int(parts["t"]), parts["v1"]

    # ⚠️ M26 §১-এর replay attack প্রতিরোধ — timestamp tolerance
    if abs(time.time() - timestamp) > tolerance_sec:
        raise SecurityException("Webhook timestamp টোলারেন্সের বাইরে — সম্ভাব্য replay attack")

    expected = hmac.new(secret.encode(), f"{timestamp}.{payload_bytes.decode()}".encode(),
                         hashlib.sha256).hexdigest()
    return hmac.compare_digest(signature, expected)   # ⚠️ timing attack প্রতিরোধ
```

**M26 §২-এর নিরাপত্তা নীতির সংশ্লেষণ:** `hmac.compare_digest` (constant-time comparison) ব্যবহার করা M26-এর "timing attack" সতর্কতার একটা concrete প্রয়োগ — সাধারণ `==` comparison ব্যবহার করলে string-এর প্রথম mismatch-এ early-exit করে, যা attacker-কে byte-by-byte signature guess করতে সাহায্য করতে পারে টাইমিং measurement দিয়ে। Timestamp tolerance M26-এর replay attack prevention-এর webhook-নির্দিষ্ট প্রয়োগ — একটা পুরনো, চুরি হওয়া webhook payload আবার পাঠানো যাবে না (এমনকি valid signature থাকলেও) যদি timestamp অনেক পুরনো হয়।

### ৪.২ Out-of-Order Delivery — M12 §৭-এর Ordering নীতির Webhook সংস্করণ

```python
def handle_psp_webhook(payload):
    payment = Payment.objects.select_for_update().get(pk=payload["payment_id"])

    # M12 §১-এর sequence number নীতির প্রয়োগ — webhook out-of-order
    # আসতে পারে (M02-এর network delivery guarantee-র অভাব)
    if payload["event_sequence"] <= payment.last_webhook_sequence:
        return   # ⚠️ পুরনো/duplicate event, ignore করো — M12-এর
                  #    "physical delivery order guarantee নেই" নীতি

    payment.status = translate_psp_status(payload["status"])   # M18 §৭-এর ACL
    payment.last_webhook_sequence = payload["event_sequence"]
    payment.save()
```

**M12 §১-এর ঘটনার webhook সংস্করণ:** ঠিক যেমন Kafka-তে একটা payment-এর event ভিন্ন partition-এ গিয়ে out-of-order প্রসেস হতে পারত, PSP webhook-ও (আলাদা HTTP request, M02-এর কোনো ordering guarantee ছাড়া) out-of-order পৌঁছাতে পারে — একটা `succeeded` webhook একটা পুরনো `processing` webhook-এর **পরে** delivered হতে পারে network delay-র কারণে, কিন্তু event সৃষ্টির ক্রম উল্টো। `event_sequence` check (M12-এর offset ধারণার webhook সমতুল্য) এই সমস্যা প্রতিরোধ করে।

### ৪.৩ Retry with Backoff — M11/M16-এর Webhook Delivery-তে পূর্ণ প্রয়োগ

M06 §১০-এ আমরা এই টাস্ক দেখেছিলাম কিন্তু একটা "TODO" হিসেবে। M11 §৬.২-এর jitter এবং M16 §৩-এর retry budget একসাথে:

```python
@shared_task(bind=True, autoretry_for=(requests.RequestException,),
             retry_backoff=True, retry_backoff_max=3600, retry_jitter=True, max_retries=15)
def deliver_webhook(self, event_id):
    event = OutboxEvent.objects.get(id=event_id)
    if event.status == "delivered":   # M11 §৫-এর idempotency
        return
    resp = webhook_session.post(merchant.webhook_url, data=sign_and_serialize(event),
                                 headers={"X-Signature": ...}, timeout=(3.05, 10))
    if not (200 <= resp.status_code < 300):
        raise requests.RequestException(f"status={resp.status_code}")
    event.mark_delivered()
```

---

## ৫. Authorization বনাম Capture — একটা Domain-Specific Two-Phase Pattern

```
Authorization: card-এ টাকা "hold" করা (M07-এর select_for_update-এর
                মতো একটা lock, কিন্তু externally, PSP-র সিস্টেমে)
Capture: hold করা টাকা actually charge করা

এই দুই-ধাপের pattern M08-এর expand-contract-এর ধারণাগত সমান্তরাল —
একটা অপারেশনকে দুইটা পৃথক, প্রতিটা independently reversible ধাপে ভাগ করা
```

```python
def authorize_payment(payment):
    result = psp_client.authorize(payment.amount_minor, payment.card_token)
    payment.status = "authorized"
    payment.psp_auth_id = result["auth_id"]
    payment.save()
    # ⚠️ এই মুহূর্তে টাকা এখনো charge হয়নি — merchant পরে capture না
    # করলে (M31-এর inventory reservation না মেলার মতো, M17-এর Saga-র
    # compensating action প্রয়োজন) auth স্বয়ংক্রিয়ভাবে expire হয় (সাধারণত ৭ দিন)

def capture_payment(payment, amount_minor=None):
    # ⚠️ Partial capture সম্ভব — auth-এর চেয়ে কম amount capture করা যায়
    #    (M18-এর Money Value Object-এ এই constraint enforce করা উচিত)
    capture_amount = amount_minor or payment.amount_minor
    if capture_amount > payment.amount_minor:
        raise ValueError("Capture amount authorization-এর বেশি হতে পারে না")
    result = psp_client.capture(payment.psp_auth_id, capture_amount)
    payment.status = "captured"
    payment.save()
```

**M17-এর Saga pattern-এর একটা natural domain উদাহরণ:** authorize-then-capture নিজেই একটা two-step saga — যদি capture কখনো না ঘটে (merchant order fulfill করতে ব্যর্থ), auth স্বয়ংক্রিয়ভাবে expire হয়ে যায় (M10-এর TTL ধারণার payment-domain সংস্করণ), কোনো explicit compensating action ছাড়াই — PSP নিজেই এই "auto-reversal" behavior দেয় কারণ এটা এত সাধারণ একটা pattern এই domain-এ।

---

## ৬. Settlement ও Reconciliation — M25-এর Incident Response-এর Financial-Audit সংস্করণ

### ৬.১ Reconciliation — M08-এর Audit Log-এর চূড়ান্ত প্রয়োগ

```python
def reconcile_daily_settlement(date, psp_settlement_report):
    """M25-এর 5 Whys নীতির automated, প্রতিদিনের সংস্করণ —
    'প্রতিটা লেনদেনে আমাদের ledger আর PSP-র রেকর্ড কি একমত?'"""

    our_transactions = Payment.objects.filter(
        settled_at__date=date, status="captured"
    ).values("id", "amount_minor")
    psp_transactions = {t["reference"]: t["amount"] for t in psp_settlement_report}

    breaks = []   # ⚠️ M28 §১-এর "এক পয়সাও গুরুত্বপূর্ণ" নীতি
    for txn in our_transactions:
        psp_amount = psp_transactions.get(str(txn["id"]))
        if psp_amount is None:
            breaks.append({"type": "missing_in_psp", "payment_id": txn["id"]})
        elif psp_amount != txn["amount_minor"]:   # ⚠️ exact match, কোনো tolerance না
            breaks.append({
                "type": "amount_mismatch", "payment_id": txn["id"],
                "our_amount": txn["amount_minor"], "psp_amount": psp_amount,
            })

    if breaks:
        # M25 §২.২-এর severity নীতি — financial break সবসময় high-priority
        create_incident(severity="SEV2", breaks=breaks)
    return breaks
```

**M25-এর incident response নীতির একটা automated, domain-specific প্রয়োগ:** এই function-টা কার্যকরভাবে একটা **daily automated audit** যা M25-এর "কোনো gap tolerate না করা" দর্শন প্রয়োগ করে — এমনকি একটা এক পয়সার mismatch একটা incident তৈরি করে (M28 §১-এর ঘটনার নীতি অনুযায়ী), M25-এর severity classification ব্যবহার করে যথাযথ response নিশ্চিত করতে।

### ৬.২ Break Handling — যখন Reconciliation মেলে না

```
সাধারণ কারণ, M-জুড়ে যা আমরা দেখেছি তার প্রয়োগ:

১. Timing difference: PSP-র settlement আমাদের capture-এর ভিন্ন দিনে
   পড়েছে (timezone/cutoff issue, M08 §৮.২-এর DST/timezone সতর্কতার
   financial সংস্করণ)

২. Rounding/currency conversion (M28 §১-এর ঘটনা)

৩. Partial capture/refund যা reconciliation logic-এ সঠিকভাবে handle
   হয়নি (M28 §৫-এর two-phase pattern-এর edge case)

৪. Webhook missed/duplicate (M28 §৪.২-এর out-of-order delivery ঠিকমতো
   handle না হলে)
```

---

## ৭. Refund ও Chargeback

### ৭.১ Refund — M05 §৮.১-এর Over-Refund Race Condition-এর সম্পূর্ণ প্রেক্ষাপট

```python
def process_refund(payment_id, amount: Money, reason: str):
    with transaction.atomic():
        payment = Payment.objects.select_for_update().get(pk=payment_id)   # M05 §৮.১

        total_refunded = payment.refunds.aggregate(Sum("amount_minor"))["amount_minor__sum"] or 0
        if total_refunded + amount.amount_minor > payment.amount_minor:
            raise ValueError("Over-refund")   # M18 §৪.৩-এর Aggregate invariant

        refund = Refund.objects.create(payment=payment, amount_minor=amount.amount_minor, reason=reason)

        # M28 §২-এর double-entry ledger — refund নিজেও একটা ledger transaction
        LedgerTransactionFactory.create_transfer(
            from_account=platform_account, to_account=merchant.refund_liability_account,
            amount=amount, description=f"Refund for payment {payment_id}",
        )
    psp_client.refund(payment.psp_auth_id, amount.amount_minor)   # M31-এর "external
                                                                     # call transaction-এর
                                                                     # বাইরে" নীতি
    return refund
```

**M05, M18, M07 তিনটা module-এর একসাথে প্রয়োগ:** `select_for_update()` (M05-এর race prevention) + Aggregate invariant check (M18-এর over-refund prevention) + M31-এর "external call transaction-এর ভেতরে না" নীতি (PSP call `transaction.atomic()` block-এর **বাইরে**, M31-এর incident-এর মতো lock ধরে রাখা এড়াতে) — সবকিছু একটা single function-এ।

### ৭.২ Chargeback — একটা External, Adversarial Event

```python
def handle_chargeback(payment_id, chargeback_amount, dispute_reason):
    """Chargeback অন্য সব event থেকে ভিন্ন — এটা merchant-এর নিয়ন্ত্রণের
    বাইরে থেকে (card network) আসে, প্রায়ই বিলম্বিত (মাস পরে), এবং
    money তৎক্ষণাৎ deduct হয়ে যায় (PSP প্রথমে টাকা কেটে নেয়,
    dispute resolution পরে)"""
    with transaction.atomic():
        payment = Payment.objects.select_for_update().get(pk=payment_id)
        LedgerTransactionFactory.create_transfer(
            from_account=merchant.balance_account, to_account=platform_chargeback_reserve,
            amount=Money(chargeback_amount, payment.currency),
            description=f"Chargeback: {dispute_reason}",
        )
        payment.status = "disputed"
        payment.save()
    # M25-এর incident-এর মতো একটা workflow শুরু — merchant-কে evidence
    # জমা দেওয়ার সুযোগ, M17-এর Saga orchestration-এর একটা দীর্ঘমেয়াদী,
    # মানুষ-জড়িত সংস্করণ
    initiate_dispute_workflow(payment, dispute_reason)
```

**M17-এর Saga orchestration-এর একটা long-running, human-in-the-loop সংস্করণ:** Chargeback dispute resolution days/weeks ধরে চলতে পারে, একাধিক ধাপ (evidence submission, card network review, resolution) — এটা M17-এর orchestration-based Saga-র নীতি অনুসরণ করে (central state tracking, M25-এর incident timeline-এর মতো), কিন্তু automated system-এর বদলে মানুষ (merchant, finance team) জড়িত।

---

## ৮. Risk Engine ও AML/KYC — M18-এর Bounded Context-এর সম্পূর্ণ উদাহরণ

```python
# M18 §৩-এর ঠিক সেই bounded context উদাহরণ — "Merchant" শব্দটা risk
# context-এ ভিন্ন অর্থ বহন করে payment context-এর তুলনায়
class RiskAssessment(models.Model):   # risk bounded context-এর নিজস্ব model
    merchant_application = models.ForeignKey("MerchantApplication", on_delete=models.PROTECT)
    risk_score = models.IntegerField()
    kyc_status = models.CharField(max_length=16)
    sanctions_check_result = models.CharField(max_length=16)

def evaluate_payment_risk(payment) -> RiskDecision:   # M18 §৫.৩-এর Domain Service
    """একাধিক bounded context-এর তথ্য একসাথে প্রয়োজন — payment
    (payment context) + merchant risk profile (risk context) + এই
    নির্দিষ্ট transaction-এর pattern (fraud detection context)"""
    merchant_risk = get_merchant_risk_profile(payment.merchant_id)   # M18 §৩-এর
                                                                        # cross-context reference
    velocity_check = check_transaction_velocity(payment.merchant_id)   # M10-এর
                                                                          # rate-limiting নীতির
                                                                          # fraud-detection প্রয়োগ
    if merchant_risk.kyc_status != "verified":
        return RiskDecision.BLOCK
    if velocity_check.exceeds_threshold:
        return RiskDecision.MANUAL_REVIEW
    return RiskDecision.APPROVE
```

**M10-এর rate limiting নীতির fraud-detection প্রয়োগ:** "Transaction velocity" check (কতগুলো transaction একটা সময় window-এ) M10 §৮-এর rate limiter algorithm-এরই একটা প্রয়োগ, কিন্তু abuse-prevention-এর বদলে fraud-detection উদ্দেশ্যে — একই algorithm (sliding window, M10 §৮.৩), ভিন্ন business meaning।

---

## ৯. Interview Section

### প্রশ্ন ১ (Senior) — "Double-entry accounting কেন একটা payment system-এ প্রয়োজন, single balance column যথেষ্ট না কেন?"

**🌟 Senior/Staff Answer**
> "Single balance column শুধু 'বর্তমান অবস্থা' জানায়, 'কীভাবে সেখানে পৌঁছালো' না। M08 §৯.১-এর audit log নীতি এখানে চূড়ান্ত রূপে প্রয়োজন — যদি একটা balance ভুল হয় (M28 §১-এর ঘটনার মতো), single-column approach-এ debug করার কোনো উপায় নেই, কারণ পুরনো মান কোথাও নেই।
>
> Double-entry-র মূল guarantee হলো প্রতিটা transaction-এ debit = credit — এটা M18-এর Aggregate invariant-এর সবচেয়ে non-negotiable উদাহরণ। এই invariant-এর মানে হলো money কখনো 'তৈরি' বা 'অদৃশ্য' হতে পারে না সিস্টেমে — প্রতিটা টাকা কোথাও থেকে এসেছে, কোথাও গেছে। যদি কখনো সব account-এর balance sum করা হয়, সেটা সবসময় শূন্য হওয়া উচিত (platform-এর নিজস্ব account সহ) — এটা একটা built-in sanity check যা প্রতিদিন চালানো যায় (M28 §৬.১-এর reconciliation-এর মতো)।
>
> এবং সবচেয়ে গুরুত্বপূর্ণ — regulatory/audit প্রয়োজনে, 'দেখান কীভাবে এই balance-এ পৌঁছালেন' একটা প্রায়ই-জিজ্ঞাসিত প্রশ্ন, আর double-entry ledger সরাসরি সেই উত্তর দেয় (M07-এর ACID-guaranteed, immutable history), single balance column-এ যেটা সম্ভবই না।"

---

### প্রশ্ন ২ (Staff / Production Incident) — "Reconciliation report একটা ৫০,০০০ টাকার mismatch দেখাচ্ছে একটা নির্দিষ্ট merchant-এ। কীভাবে debug করবেন?"

**🌟 Senior/Staff Answer**
> "এটা M25-এর systematic debugging playbook-এর একটা financial-domain প্রয়োগ, কিন্তু higher stakes সহ — M28 §১-এর নীতি অনুযায়ী, এটাকে একটা SEV2 (বা উচ্চতর) incident হিসেবে treat করব সাথে সাথে, শুধু একটা 'accounting task' না।
>
> **প্রথম পদক্ষেপ — scope নির্ধারণ (M25-এর triage)।** এটা কি একটা single transaction-এর সমস্যা, নাকি একটা systematic pattern (M28 §৬.২-এর সব চারটা সাধারণ কারণ চেক করা)? আমি সেই নির্দিষ্ট merchant-এর সব transaction-এ M28 §৬.১-এর reconciliation function চালাব, দেখব একটা আলাদা break, নাকি একাধিক ছোট break যোগ করে ৫০,০০০ হয়েছে।
>
> **যদি একটা single, বড় transaction:** সরাসরি M28 §২-এর LedgerTransaction history দেখব — সেই transaction-এর সব entry, তাদের timestamp, সম্পর্কিত webhook (M28 §৪-এর delivery log)। যদি একটা duplicate charge বা duplicate refund সন্দেহ হয়, M28 §৩-এর idempotency key ব্যবহার করে সেই নির্দিষ্ট payment-এর সব সম্পর্কিত record খুঁজব।
>
> **যদি অনেক ছোট break যোগ হয়ে বড় হয়েছে:** এটা M28 §১-এর ঘটনার মতো একটা systematic rounding/conversion bug নির্দেশ করে — বিশেষত যদি এই merchant multi-currency ব্যবহার করে। আমি একটা sample transaction manually recalculate করব, প্রতিটা step-এ (M07-এর query optimization debugging-এর মতো, কিন্তু এখানে financial calculation-এ) exact rounding mode/precision চেক করে PSP-র expected behavior-এর সাথে তুলনা করে।
>
> **প্রতিরোধ, দীর্ঘমেয়াদী:** M28 §৬.১-এর daily automated reconciliation যদি ইতিমধ্যে না থাকে, এই ঘটনা সেটা তৈরির একটা শক্তিশালী কারণ — একটা ৫০,০০০ টাকার gap মাসের পর মাস ধরা না পড়া মানে reconciliation frequency/coverage-এ একটা গুরুতর gap আছে।"

---

### প্রশ্ন ৩ (Coding / Debugging) — "এই refund flow-এ কী সমস্যা আছে?"

```python
def process_refund(payment_id, amount):
    payment = Payment.objects.get(pk=payment_id)
    psp_client.refund(payment.psp_auth_id, amount)   # PSP-কে প্রথমে call
    with transaction.atomic():
        payment.refunded_amount += amount
        payment.save()
        Refund.objects.create(payment=payment, amount_minor=amount)
```

**🌟 Senior Answer**
> "তিনটা গুরুতর সমস্যা, এই handbook-এর একাধিক module-এর নীতি লঙ্ঘন করে:
>
> **১. `select_for_update()` অনুপস্থিত (M05 §৮.১)।** `payment.refunded_amount += amount` একটা read-modify-write race condition-এর শিকার — দুইটা concurrent refund request over-refund তৈরি করতে পারে, ঠিক M05-এর original bug-এর মতো।
>
> **২. PSP call transaction-এর আগে, এবং কোনো idempotency check ছাড়া — সবচেয়ে বিপজ্জনক।** যদি `psp_client.refund()` সফল হয় (PSP-তে টাকা ফেরত গেছে) কিন্তু তারপরের `transaction.atomic()` block ব্যর্থ হয় (database error, worker crash), আমাদের নিজের ledger-এ **কোনো রেকর্ড নেই** এই refund-এর, যদিও PSP-তে টাকা চলে গেছে — এটা M28 §১-এর ঘটনার চেয়েও খারাপ একটা reconciliation break তৈরি করবে, কারণ এখানে সম্পূর্ণ missing transaction, শুধু rounding না।
>
> **৩. Over-refund validation সম্পূর্ণ অনুপস্থিত (M18 §৪.৩)।** কোনো চেক নেই যে `total_refunded + amount > payment.amount_minor` — একটা merchant বা bug একই payment বারবার refund করাতে পারে।
>
> **সংশোধিত সংস্করণ — M28 §৭.১-এর সঠিক pattern:**
> ```python
> def process_refund(payment_id, amount: Money, reason: str):
>     with transaction.atomic():
>         payment = Payment.objects.select_for_update().get(pk=payment_id)  # ফিক্স ১
>         total_refunded = payment.refunds.aggregate(Sum('amount_minor'))['amount_minor__sum'] or 0
>         if total_refunded + amount.amount_minor > payment.amount_minor:  # ফিক্স ৩
>             raise ValueError('Over-refund')
>         refund = Refund.objects.create(payment=payment, amount_minor=amount.amount_minor,
>                                        status='pending', reason=reason)
>         # M28 §২-এর ledger entry এখানে, একই transaction-এ
>     # PSP call transaction-এর বাইরে (ফিক্স ২), refund status 'pending' থেকে
>     # 'completed'-এ update হবে PSP response-এর ভিত্তিতে, async ভাবে —
>     # M31-এর 'external call কখনো transaction-এর ভেতরে না' নীতির সরাসরি প্রয়োগ
>     charge_refund_via_psp.delay(refund.id)   # M11-এর idempotent Celery task
>     return refund
> ```
> এখানে database-level correctness (lock, invariant) আগে নিশ্চিত করা হচ্ছে, তারপর external, potentially-slow/unreliable PSP call async-এ পাঠানো হচ্ছে — M31-এর মূল architecture নীতির (payment creation-এ যেমন করা হয়েছিল) refund flow-এ সম্পূর্ণ, সামঞ্জস্যপূর্ণ প্রয়োগ।"

---

### প্রশ্ন ৪ (Architecture Decision) — "আমাদের একটা নতুন multi-currency feature-এ, currency conversion কোথায় করা উচিত — payment creation-এর সময়, নাকি settlement-এর সময়?"

**🌟 Senior/Staff Answer**
> "এই সিদ্ধান্তের গভীর ব্যবসায়িক এবং technical পরিণতি আছে, M28 §১-এর ঘটনার সরাসরি সতর্কতা মাথায় রেখে।
>
> **যদি payment creation-এর সময় convert করা হয়:** merchant সাথে সাথে জানে তাদের কত টাকা পাবে (local currency-তে), কিন্তু exchange rate সেই মুহূর্তে lock হয়ে যায় — যদি settlement কয়েকদিন পরে হয় (M28 §৬-এর সাধারণ settlement cycle), platform exchange rate risk বহন করে (rate বদলে গেলে platform লাভ/ক্ষতি করে)।
>
> **যদি settlement-এর সময় convert করা হয়:** merchant শুধু approximate amount জানে creation-এর সময়, actual amount settlement-এর দিনের rate-এ নির্ধারিত হয় — merchant exchange rate risk বহন করে, কিন্তু M28 §১-এর ঘটনার মতো rounding bug-এর সুযোগ প্রতিটা settlement cycle-এ বাড়ে (M08-এর batch processing-এর মতো, একবারে অনেক conversion, একটা systematic error বড় impact তৈরি করতে পারে)।
>
> **আমার সুপারিশ:** payment creation-এর সময় rate lock করা এবং সেই rate-টা explicitly, ledger entry-র সাথে **store** করা (M08-এর audit trail নীতি) — শুধু final amount না, ব্যবহৃত exchange rate-ও, যাতে M28 §৬.১-এর reconciliation-এ ঠিক কোন rate ব্যবহার হয়েছিল তা verify করা যায়। এবং critically — conversion logic-এ M28 §১-এর ঘটনার শিক্ষা প্রয়োগ করে, PSP-র rounding mode ঠিক match করে (একটা explicit, tested, documented decision — 'আমরা `ROUND_HALF_UP` ব্যবহার করছি কারণ PSP তাই করে,' অনুমান না) এবং একটা automated test (M23-এর boundary-case parametrize test) যা ঠিক `.5` boundary-তে verify করে।
>
> এই সিদ্ধান্তটা শুধু engineering-এর না — exchange rate risk কে বহন করবে (platform নাকি merchant) সেটা একটা business/legal সিদ্ধান্ত (M16-এর fail-open/fail-closed-এর মতো), আমার দায়িত্ব হলো সেই সিদ্ধান্ত যেই হোক, সেটা সঠিকভাবে, auditably, এবং penny-precision-এ implement করা।"

---

## ১০. হাতে-কলমে অনুশীলন

**১ — Double-entry ledger বানান ও invariant টেস্ট করুন (৪০ মিনিট)**
M28 §২-এর মডেল implement করুন। একটা টেস্ট লিখুন যা যাচাই করে প্রতিটা `LedgerTransaction`-এ debit-credit sum শূন্য। একটা ইচ্ছাকৃত ভুল transaction তৈরির চেষ্টা করে দেখুন constraint block করছে কি না।

**২ — Rounding boundary bug পুনরুৎপাদন করুন (৩০ মিনিট)**
`Decimal` দিয়ে একটা currency conversion function লিখুন `ROUND_HALF_UP` এবং `ROUND_HALF_EVEN` দুইভাবে। এমন একটা amount খুঁজুন যেখানে দুইটা ভিন্ন ফলাফল দেয় (M28 §১-এর ঘটনা নিজের চোখে দেখুন)।

**৩ — Webhook signature verification implement করুন (৩৫ মিনিট)**
M28 §৪.১-এর sign/verify function লিখুন। একটা টেস্ট লিখুন যা একটা পুরনো timestamp দিয়ে replay attack simulate করে, verification ব্যর্থ হওয়া নিশ্চিত করুন।

**৪ — Reconciliation function টেস্ট করুন (৩০ মিনিট)**
M28 §৬.১-এর reconciliation logic নিয়ে, ইচ্ছাকৃতভাবে একটা mismatch তৈরি করুন (একটা payment-এর amount সামান্য বদলে) এবং দেখুন `breaks` সঠিকভাবে সেটা ধরছে।

---

## ১১. মূল কথা

1. **FinTech-এ "প্রায় সঠিক" মানে "ভুল"** — এক পয়সার গরমিলও একটা তদন্ত-দাবি করা ঘটনা, "সামান্য" bug না।
2. **Double-entry ledger M18-এর Aggregate invariant নীতির সবচেয়ে কঠোর প্রয়োগ** — প্রতিটা transaction-এ debit = credit, database constraint দিয়ে enforce করা, শুধু application logic-এ না।
3. **Cached balance একটা optimization, কখনো source of truth না** — actual truth সবসময় সম্পূর্ণ ledger entry history।
4. **একটা payment-এ একাধিক স্তরের idempotency প্রয়োজন** — client-facing, internal task, এবং PSP-facing, প্রতিটা ভিন্ন duplicate-risk address করে।
5. **Webhook signing শুধু authenticity না, replay attack-ও প্রতিরোধ করে** — timestamp tolerance + constant-time comparison।
6. **Webhook out-of-order আসতে পারে** — sequence number দিয়ে ordering enforce করুন, delivery order-এর উপর নির্ভর না করে।
7. **External call (PSP) কখনো `transaction.atomic()` block-এর ভেতরে না** — M31-এর মূল নীতির financial-domain-এ সবচেয়ে critical প্রয়োগ, কারণ ব্যর্থতার পরিণতি এখানে টাকা।
8. **Authorization/capture একটা natural two-phase saga** — expire-based auto-reversal, explicit compensation ছাড়াই।
9. **Reconciliation প্রতিদিনের, automated, zero-tolerance audit হওয়া উচিত** — M25-এর incident severity নীতি financial break-এ সরাসরি প্রযোজ্য।
10. **Risk/fraud detection M18-এর bounded context এবং M10-এর rate limiting algorithm-এর একটা concrete সংশ্লেষণ** — একই টুল, ভিন্ন business meaning।

---

## পরের Module

**M29 — Blockchain Backend।** আজ আমরা traditional payment rail (PSP, card network) দেখলাম। পরের module-এ আমরা একটা সমান্তরাল কিন্তু architecturally ভিন্ন জগতে যাব — blockchain-based payment। Node/RPC infrastructure (M17-এর third-party integration নীতির blockchain সংস্করণ), transaction lifecycle ও nonce management (M28-এর idempotency নীতির একটা সম্পূর্ণ ভিন্ন, কিন্তু ধারণাগতভাবে সম্পর্কিত সমাধান), wallet management (M26-এর key management/HSM নীতির সরাসরি সম্প্রসারণ), আর blockchain indexing — যেখানে M28-এর "ledger" ধারণা একটা সম্পূর্ণ ভিন্ন, distributed, trustless দর্শনে পুনর্গঠিত হয়।

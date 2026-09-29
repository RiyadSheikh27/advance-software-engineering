# JavaScript vs Python vs Java: কোড কীভাবে চলে

## এক লাইনে মূল কথা

তিন ভাষাই আপনার কোডকে আগে `Bytecode` বানায়, তারপর চালায়। পার্থক্য হলো: কে, কখন, কীভাবে দ্রুত `Machine code` বানায়।

---

## ১. ৬টা শব্দ আগে জেনে নিন

| শব্দ | মানে |
|---|---|
| `Source code` | আপনার লেখা কোড |
| `Machine code` | 0 আর 1 দিয়ে লেখা নির্দেশ, যা CPU সরাসরি বোঝে |
| `Bytecode` | source code আর machine code-এর মাঝামাঝি সহজ ভাষা |
| `Interpreter` | bytecode একটা একটা করে পড়ে সাথে সাথে চালায় |
| `JIT` | চলার সময়ই বারবার চলা অংশকে দ্রুত machine code বানায় |
| `Virtual Machine` | সফটওয়্যারের ভেতরের কাল্পনিক কম্পিউটার, যা bytecode চালায় |

---

## ২. ছবি: কোড চলার পথ

```
JavaScript:  Source -> Parser -> Bytecode -> Interpreter --(বারবার চললে)--> JIT -> Machine code
Python:      Source -> Parser -> Bytecode -> Interpreter                     (সাধারণত JIT নেই)
Java:        Source -> javac  -> Bytecode -> Interpreter --(বারবার চললে)--> JIT -> Machine code
```

- `Parser`: কোড পড়ে বোঝার মতো কাঠামোতে ভাঙে।
- `javac`: Java-র কম্পাইলার। এটা প্রোগ্রাম চালানোর আগেই আলাদাভাবে চালাতে হয়।
- ইঞ্জিনের নাম: JavaScript-এ `V8`, Python-এ `CPython`, Java-তে `JVM`।

---

## ৩. উদাহরণ: `sum = 5 + 3` ভেতরে কীভাবে চলে

### JavaScript (`Register-based`)

`Register` হলো CPU-র ভেতরের দ্রুত ছোট জায়গা। `Accumulator` হলো মাঝের ফল রাখার বিশেষ জায়গা।

```
Ldar r0    ->  accumulator = 5
Add  r1    ->  accumulator = 5 + 3 = 8
Star r2    ->  sum = 8
```

### Python (`Stack-based`)

`Stack` হলো থালার স্তূপ: যা শেষে রাখা হয়, সেটাই আগে তোলা হয়।

```
LOAD a       stack: [5]
LOAD b       stack: [5, 3]
ADD          stack: [8]        <- ওপরের দুটো তুলে যোগ করা হলো
STORE sum    sum = 8
```

Python-এ সংখ্যা আসলে `Object` (তথ্যসহ প্যাকেট)। তাই `8` বানাতে নতুন object তৈরি হয়। এতে বাড়তি কাজ লাগে।

### Java (`Stack-based`)

```
iload_0      stack: [5]
iload_1      stack: [5, 3]
iadd         stack: [8]        <- i মানে int
istore_2     sum = 8
```

Java-র `int` সরাসরি একটা সংখ্যা, object নয়। তাই এটা সবচেয়ে হালকা।

---

## ৪. ছবি: `JIT` কেন দ্রুত করে

```
JIT ছাড়া (Python):    bytecode -> পড়ো -> চালাও -> পড়ো -> চালাও -> ...   (প্রতিবার একই কাজ)

JIT সহ (JS, Java):     bytecode -> চালাও -> চালাও -> [বারবার চলছে!]
                                                     |
                                                     v
                                          সরাসরি machine code বানিয়ে ফেলো
                                                     |
                                                     v
                                          পরের বার থেকে সরাসরি দ্রুত চলে
```

- JavaScript-এর JIT ধরে নেয় `a`, `b` সংখ্যাই থাকবে। হঠাৎ লেখা (string) এলে অনুমান ভুল হয়, তখন `Deoptimization` হয়, মানে ধীর কিন্তু নিরাপদ bytecode-এ ফিরে যায়।
- Java-র JIT দুই স্তরের: `C1` (দ্রুত, হালকা) তারপর `C2` (ধীরে, গভীর optimization)।

---

## ৫. ছবি: একসাথে অনেক কাজ

`Thread` মানে একটা কাজের ধারা।

```
JavaScript:   [কাজ1] [কাজ2] [কাজ3]  ->  Event Loop  ->  একটি thread
Python:       Thread1  Thread2  Thread3  ->  [GIL তালা]  ->  একবারে একটাই চলে
Java:         Thread1 -> core 1
              Thread2 -> core 2
              Thread3 -> core 3        (সত্যিকারের একসাথে চলে)
```

- `Event Loop`: চক্র, যা অপেক্ষার কাজ (ফাইল পড়া, নেটওয়ার্ক) পাশে রেখে পরের কাজ ধরে। একটি thread-ই অনেক কাজ সামলায়।
- `GIL` (Global Interpreter Lock): Python-এর তালা। এক সময়ে শুধু একটা thread Python কোড চালাতে পারে।

---

## ৬. তুলনা এক নজরে

| বিষয় | JavaScript | Python | Java |
|---|---|---|---|
| ইঞ্জিন | V8 | CPython | JVM |
| JIT | আছে | সাধারণত নেই | আছে |
| Bytecode ধরন | Register | Stack | Stack |
| ধরন (Typing) | Dynamic | Dynamic | Static |
| সংখ্যা | ইঞ্জিন অপ্টিমাইজ করে | সবসময় object | সরাসরি `int` |
| একসাথে কাজ | Event Loop | GIL আছে | সত্যিকারের multi-thread |
| মেমরি পরিষ্কার | Garbage Collector | গণনা (refcount) + GC | Garbage Collector |
| শুরু হতে সময় | কম | খুব কম | বেশি |
| দীর্ঘ চলায় গতি | ভালো | ধীর | খুব ভালো |

- `Dynamic typing`: ভ্যারিয়েবলের ধরন চলার সময় ঠিক হয়।
- `Static typing`: ধরন আগে লিখতে হয়, ভুল compile-এর সময়ই ধরা পড়ে।
- `Garbage Collector`: আর ব্যবহার না হওয়া ডাটা স্বয়ংক্রিয়ভাবে মুছে ফেলা ঝাড়ুদার।
- `refcount` (reference counting): একটা object কতজন ধরে আছে গোনা হয়, 0 হলে মুছে যায়।

---

## ৭. কোন কাজে কোনটা

| কাজ | সাধারণত মানায় |
|---|---|
| ব্রাউজারের ওয়েবসাইট | JavaScript |
| ডাটা সায়েন্স, AI, স্ক্রিপ্ট | Python |
| বড় ব্যাংকিং বা এন্টারপ্রাইজ সিস্টেম, অ্যান্ড্রয়েড | Java |

"সেরা ভাষা" বলে কিছু নেই। কাজ আর টিম দেখে বেছে নিতে হয়।

---

## ৮. মনে রাখার ৪ কথা

1. তিন ভাষাই আগে bytecode বানায়।
2. JavaScript ও Java বারবার চলা কোডকে JIT দিয়ে machine code বানায়, তাই দ্রুত। সাধারণ Python এটা করে না।
3. Java আগে compile করে ধরনের ভুল ধরে, বাকি দুটো চলার সময় দেখে।
4. একসাথে কাজে: JavaScript Event Loop, Python GIL-এর সীমায়, Java সত্যিকারের multi-thread।
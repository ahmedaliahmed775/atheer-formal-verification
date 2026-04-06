# مستودع التحقق الرسمي لبروتوكول أثير — الإصدار 2.0

> **براهين أمنية مُتحقَّق منها آلياً لبروتوكول أثير v2.0 للدفع المتنقل بدون إنترنت**
> جامعة العلوم والتكنولوجيا، صنعاء، اليمن

[![التحقق الرسمي](https://github.com/ahmedaliahmed775/atheer-formal-verification/actions/workflows/verify.yml/badge.svg)](https://github.com/ahmedaliahmed775/atheer-formal-verification/actions/workflows/verify.yml)
[![Tamarin](https://img.shields.io/badge/Tamarin%20Prover-%E2%89%A51.8.0-blue)](https://tamarin-prover.com)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![البراهين](https://img.shields.io/badge/البراهين-11%20%7C%209%20أمنية%20%2B%201%20سلامة%20%2B%201%20سريع-brightgreen)]()

---

## نظرة عامة

يحتوي هذا المستودع على **التحقق الأمني الرسمي الكامل** لبروتوكول أثير v2.0، المكوّن الثالث من منظومة أثير المتكاملة.

| المكوّن | المستودع | الوصف |
|---------|---------|-------|
| **Atheer SDK** | `atheer-sdk` | مكتبة Android للعميل (HCE، TEE، NFC) |
| **Atheer Switch** | `atheer-switch-backend` | بوابة الدفع والتسوية |
| **التحقق الرسمي** | *هذا المستودع* | نموذج Tamarin v2.0 والبراهين الرياضية |

تم إثبات أمان البروتوكول تحت **نموذج المهاجم Dolev-Yao** باستخدام [Tamarin Prover](https://tamarin-prover.com). يعمل التحقق **تلقائياً عبر GitHub Actions** عند كل تغيير في ملف البروتوكول.

---

## التحسينات الجديدة في v2.0

تم دمج خمسة تحسينات أمنية جوهرية في هذا الإصدار، مع التحقق الرسمي من كل منها:

### 1. سلسلة الثقة — الاشتقاق الديناميكي للتوكنات
**البرهان:** `Seed_Secrecy`

بدلاً من تحميل حزم توكنات (LUK Batches)، يُخزَّن **بذرة رئيسية (Master Seed) واحدة** مشفّرة داخل TEE. عند الحاجة لكل دفعة، يُشتق توكن فريد:
```
LUK_i = PRF(master_seed, hw_counter_i)
```
يمنح ذلك عدداً غير محدود تقريباً من المعاملات بدون إنترنت.

### 2. توقيع العتبة — MPC بين TEE والـ SIM
**البرهان:** `MPC_Split_Security`

مفتاح التوقيع مُقسَّم إلى جزأين **مستقلَّين تماماً**:
```
sk_tee_share  →  يُحفظ في TEE الهاتف
sk_sim_share  →  يُحفظ في شريحة SIM
```
توقيع صالح يتطلب **كلا الجزأين معاً**. اختراق Android بالكامل (Root) غير كافٍ.

### 3. تحديد المسافة — Distance Bounding عبر NFC
**البرهان:** `Distance_Bounding`

التاجر يُصدر تحدياً زمنياً (`db_challenge`). العميل يستجيب خلال نفس جولة NFC. أي هجوم ترحيل عبر الإنترنت يُدخل تأخيراً يُبطل الاستجابة.

### 4. نزاهة الجهاز — Play Integrity API
**البرهان:** `Device_Integrity`

قبل أي توفير للبذرة، تتحقق البوابة من توكن نزاهة Play موقَّع بمفتاح Google. المحاكيات والأجهزة المخترَقة مُستبعَدة **قبل الحصول على أي مواد تشفيرية**.

### 5. التوقيت الذري — عداد الأجهزة الأحادي الاتجاه
**البرهان:** `Counter_Monotonicity`

الجلسة المسلّحة (60 ثانية) مرتبطة بعداد `hw_monotonic_counter` بدلاً من ساعة نظام Android. لا يمكن للمهاجم التلاعب بالساعة لإبقاء نافذة التوقيع مفتوحة.

---

## البراهين الأمنية المُثبَتة (11 برهاناً)

| # | البرهان | الخاصية | التحسين |
|---|---------|----------|---------|
| L1 | `Authentication` | التسوية تستلزم مشاركة العميل الحقيقية | أساسي |
| L2 | `Anti_Replay` | (LUK، Nonce) يُقبَلان مرة واحدة فقط | أساسي |
| L3 | `Session_Integrity` | PABC مقيّد بعداد الأجهزة | #5 التوقيت الذري |
| L4 | `Seed_Secrecy` | البذرة لا يمكن للمهاجم تعلّمها | #1 سلسلة الثقة |
| L5 | `MPC_Split_Security` | جزء TEE وحده لا يكفي للتوقيع | #2 توقيع العتبة |
| L6 | `Distance_Bounding` | التسوية تستلزم برهان قرب فيزيائي | #3 تحديد المسافة |
| L7 | `Device_Integrity` | البذور لا تُوفَّر لأجهزة غير مُصادَق عليها | #4 نزاهة الجهاز |
| L8 | `Counter_Monotonicity` | عداد الأجهزة لا يُعاد استخدامه | #5 التوقيت الذري |
| L9 | `Merchant_ZeroKnowledge` | SoftPOS لا يحصل على مفاتيح العميل | أساسي |
| L10 | `Switch_NonFrameability` | السويتش لا يسوّي بدون توقيع MPC + قرب | أساسي |
| L11 | `Executability` | *(سلامة)* النموذج له آثار تنفيذ صحيحة | — |

---

## التحقق التلقائي عبر GitHub Actions

يُشغَّل التحقق تلقائياً عند **كل push أو Pull Request** يغيّر ملف `.spthy`.

### هيكل الـ Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                   GitHub Actions Pipeline               │
│                                                         │
│  [push / PR]                                            │
│       │                                                 │
│       ▼                                                 │
│  ① فحص بناء الجملة (Syntax Check) — 10 دقائق          │
│       │                                                 │
│       ▼                                                 │
│  ② إثبات Executability (سريع) — 15 دقيقة              │
│       │                                                 │
│       ▼                                                 │
│  ③ إثبات 9 براهين أمنية (متوازٍ) — حتى 60 دقيقة      │
│    ┌──────────┬──────────┬──────────┬─────────────┐    │
│    │ Auth.    │ Anti_Rep.│ Session  │ Seed_Secr.  │    │
│    │ MPC_Splt.│ Dist_Bnd │ Dev_Int. │ Ctr_Mono.   │    │
│    │          │          │          │ NonFrame.   │    │
│    └──────────┴──────────┴──────────┴─────────────┘    │
│       │                                                 │
│       ▼                                                 │
│  ④ ملخص نتائج موحد (Job Summary)                      │
└─────────────────────────────────────────────────────────┘
```

### التشغيل اليدوي (Workflow Dispatch)

من واجهة GitHub → **Actions** → **Atheer Protocol — Formal Verification** → **Run workflow**:

| الحقل | الوصف | مثال |
|-------|-------|-----|
| `lemma` | اسم برهان محدد (فارغ = الكل) | `MPC_Split_Security` |
| `timeout` | مهلة الإثبات بالدقائق | `30` |

---

## التثبيت والتشغيل المحلي

### المتطلبات

```bash
# Ubuntu / Debian
sudo apt-get install -y tamarin-prover graphviz

# macOS
brew install tamarin-prover

# التحقق
tamarin-prover --version  # يجب أن يكون >= 1.8.0
```

### تشغيل التحقق

```bash
# استنساخ المستودع
git clone https://github.com/ahmedaliahmed775/atheer-formal-verification.git
cd atheer-formal-verification

# إثبات سلامة النموذج أولاً (أسرع)
tamarin-prover atheer_protocol.spthy --prove=Executability

# إثبات برهان محدد
tamarin-prover atheer_protocol.spthy --prove=MPC_Split_Security

# إثبات جميع البراهين (قد يستغرق 30-90 دقيقة)
tamarin-prover atheer_protocol.spthy --prove \
  +RTS -N$(nproc) -RTS

# واجهة تفاعلية (مُوصى بها للاستكشاف)
tamarin-prover interactive atheer_protocol.spthy
# ثم: http://localhost:3001
```

### الناتج المتوقع

```
==============================================================================
summary of summaries:

analyzed: atheer_protocol.spthy

  Executability           (exists-trace): verified (1 step)
  Authentication          (all-traces):   verified (14 steps)
  Anti_Replay             (all-traces):   verified (3 steps)
  Session_Integrity       (all-traces):   verified (9 steps)
  Seed_Secrecy            (all-traces):   verified (7 steps)
  MPC_Split_Security      (all-traces):   verified (5 steps)
  Distance_Bounding       (all-traces):   verified (8 steps)
  Device_Integrity        (all-traces):   verified (4 steps)
  Counter_Monotonicity    (all-traces):   verified (3 steps)
  Merchant_ZeroKnowledge  (all-traces):   verified (6 steps)
  Switch_NonFrameability  (all-traces):   verified (9 steps)

==============================================================================
```

---

## هيكل المستودع

```
atheer-formal-verification/
│
├── .github/
│   └── workflows/
│       └── verify.yml              # GitHub Actions — CI/CD للتحقق
│
├── proofs/                         # نتائج الإثبات المحلية
│   └── .gitkeep
│
├── atheer_protocol.spthy           # النموذج الرسمي v2.0 (11 برهاناً)
├── README.md                       # هذا الملف
└── LICENSE                         # MIT
```

---

## الربط بكود أثير v2.0

| عنصر النموذج | مكوّن SDK | مكوّن Switch |
|-------------|----------|-------------|
| قاعدة `Client_MPC_KeyGen` | `AtheerKeystoreManager.kt` + SIM SE API | — |
| قاعدة `Client_Attest_Device` | Play Integrity API | — |
| قاعدة `Switch_Verify_Integrity` | — | `integrityController.js` |
| قاعدة `Switch_Provision_Seed` | — | `seedController.js` |
| قاعدة `Client_Send_APDU` | `AtheerApduService.kt` | — |
| قاعدة `Merchant_DB_Challenge` | — | `AtheerNfcReader.kt` |
| قاعدة `Switch_Verify_And_Settle` | — | `paymentController.js` |
| معادلة `mpc_combine` | `AtheerMPCSigner.kt` | — |
| قيد `Counter_Strictly_Increasing` | `HardwareMonotonicCounter.kt` | — |

---

## الاستشهاد العلمي

```bibtex
@inproceedings{almutawakel2026atheer,
  title     = {Eliminating Cloud Dependency in Mobile Payments via
               Isolated Cellular Routing and SoftPOS},
  author    = {Al-Mutawakel, Ahmed and Al-Mekhlafi, Nabil and Al-Fahidi, Bilal},
  booktitle = {University of Science and Technology, Sana'a},
  year      = {2026},
  note      = {أداة التحقق الرسمي v2.0 متاحة على:
               \url{https://github.com/ahmedaliahmed775/atheer-formal-verification}}
}
```

---

## الترخيص

رخصة MIT — انظر [LICENSE](LICENSE).

---

*مستودع التحقق الرسمي لأثير v2.0 — جامعة العلوم والتكنولوجيا، صنعاء، اليمن*

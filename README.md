# مستودع التحقق الرسمي لبروتوكول أثير

> **براهين أمنية مُتحقَّق منها آلياً لبروتوكول أثير للدفع المتنقل بدون إنترنت**
> جامعة العلوم والتكنولوجيا، صنعاء، اليمن

[![Tamarin](https://img.shields.io/badge/Tamarin%20Prover-%E2%89%A51.6.0-blue)](https://tamarin-prover.com)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![الحالة](https://img.shields.io/badge/البراهين-7%20%7C%206%20أمنية%20%2B%201%20سلامة-brightgreen)]()

---

## نظرة عامة

يحتوي هذا المستودع على **التحقق الأمني الرسمي الكامل** لبروتوكول أثير، وهو المكوّن الثالث من منظومة أثير المتكاملة:

| المكوّن | المستودع | الوصف |
|---------|---------|-------|
| **Atheer SDK** | `atheer-sdk` | مكتبة Android للعميل (HCE، TEE، NFC) |
| **Atheer Switch** | `atheer-switch-backend` | بوابة الدفع والتسوية |
| **التحقق الرسمي** | *هذا المستودع* | نموذج Tamarin والبراهين الرياضية |

تم إثبات أمان البروتوكول تحت **نموذج المهاجم Dolev-Yao** — أقوى نموذج مهاجم رمزي في علم التحقق من بروتوكولات التشفير — باستخدام أداة [Tamarin Prover](https://tamarin-prover.com)، وهي نفس الأداة التي تم بها التحقق من بروتوكولَي TLS 1.3 وSignal.

---

## معمارية البروتوكول

```
┌──────────────────────────────────────────────────────────────────┐
│                    مسار بروتوكول أثير                           │
│                                                                  │
│  المرحلة الأولى: توفير التوكنات (متصل، TLS 1.3)               │
│  ┌──────────┐   دفعة LUK مشفّرة          ┌──────────────────┐  │
│  │  العميل  │◄──────────────────────────│  أثير سويتش      │  │
│  │  (TEE)   │   تسجيل المفتاح العام     │  (البوابة)       │  │
│  └──────────┘──────────────────────────►└──────────────────┘  │
│                                                                  │
│  المرحلة الثانية: تبادل NFC بدون إنترنت                       │
│  ┌──────────┐   APDU (PABC + Token)      ┌──────────────────┐  │
│  │  العميل  │──────────────────────────►│  التاجر          │  │
│  │  (SDK)   │   [جلسة بصمة 60 ثانية]   │  (SoftPOS)       │  │
│  └──────────┘                            └──────────────────┘  │
│                                                                  │
│  المرحلة الثالثة: التسوية عبر APN الخاص (IPSec)               │
│  ┌──────────┐   ChargeRequest موقّع      ┌──────────────────┐  │
│  │  التاجر  │──────────────────────────►│  أثير سويتش      │  │
│  │ (SoftPOS)│◄──────────────────────────│  ← البنك الأساسي │  │
│  └──────────┘   إشعار تسوية ناجحة        └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**الابتكارات الأمنية الرئيسية المُنمذَجة:**

- **التشفير البيومتري المُعتمد مسبقاً (PABC):** المفتاح الخاص ECDSA المقيم في TEE لا يمكن الوصول إليه إلا خلال جلسة مسلّحة مدتها 60 ثانية، تُفتح حصراً بعد المصادقة البيومترية الناجحة على مستوى TEE.
- **مفاتيح الاستخدام المحدود (LUKs):** كل توكن مرتبط تشفيرياً بمعاملة واحدة فقط. بعد الاستهلاك، لا يمكن إعادة استخدامه أبداً.
- **هوية الثقة الصفرية (Zero-Trust):** السويتش يستخرج هوية العميل حصراً من سجله الداخلي — ولا يثق أبداً بالادعاءات الواردة في الحمولة.
- **منع الإعادة بالـ Nonce:** كل طلب تسوية يتضمن nonce فريداً يُتحقق منه مقابل ذاكرة تخزين مؤقت.

---

## الخصائص الأمنية المُثبَتة

| # | Lemma | الخاصية | التهديد المُبطَل |
|---|-------|----------|----------------|
| 1 | `Authentication` | اتفاقية غير حقنية على حقائق الدفع | انتحال الهوية، المعاملات المزوّرة |
| 2 | `Anti_Replay` | كل (LUK، Nonce) يُقبَل مرة واحدة فقط | هجمات الإعادة المتزامنة واللامتزامنة |
| 3 | `Session_Integrity` | الـ PABC يُولَّد فقط داخل الجلسة المسلّحة | التلاعب الزمني، التوقيعات المسروقة |
| 4 | `LUK_Secrecy` | رمز التوكن لا يمكن للمهاجم تعلّمه | التنصت، سرقة التوكن |
| 5 | `Merchant_ZeroKnowledge` | SoftPOS لا يحصل على المفتاح الخاص للعميل | التاجر الخبيث، تطبيق SoftPOS الملتوي |
| 6 | `Switch_NonFrameability` | السويتش لا يسوّي معاملات مزوّرة | الرجل في المنتصف، خداع البروتوكول |
| 7 | `Executability` | *(سلامة)* توجد آثار تنفيذ صحيحة على الأقل | التحقق من صحة النموذج |

---

## التثبيت

### المتطلبات الأساسية

أداة Tamarin Prover تعمل على **macOS** أو **Linux** (يُوصى بـ Ubuntu/Debian).

#### الخيار الأول — النسخة الجاهزة (مُوصى به)

```bash
# Ubuntu / Debian
sudo apt-get update
sudo apt-get install -y tamarin-prover

# macOS (Homebrew)
brew install tamarin-prover
```

#### الخيار الثاني — البناء من المصدر

```bash
# تثبيت التبعيات
sudo apt-get install -y haskell-stack graphviz

# استنساخ وبناء Tamarin
git clone https://github.com/tamarin-prover/tamarin-prover.git
cd tamarin-prover
stack install
```

#### التحقق من التثبيت

```bash
tamarin-prover --version
# المتوقع: tamarin-prover 1.x.x
```

---

## تشغيل التحقق

### إثبات جميع البراهين (سطر الأوامر)

```bash
# استنساخ هذا المستودع
git clone https://github.com/ahmedaliahmed775/atheer-formal-verification.git
cd atheer-formal-verification

# تشغيل الإثبات الكامل (قد يستغرق 5–30 دقيقة حسب الجهاز)
tamarin-prover atheer_protocol.spthy --prove

# إثبات Lemma محدد فقط
tamarin-prover atheer_protocol.spthy --prove=Authentication
tamarin-prover atheer_protocol.spthy --prove=Anti_Replay
tamarin-prover atheer_protocol.spthy --prove=Executability
```

### الواجهة التفاعلية (مُوصى بها للاستكشاف)

```bash
# تشغيل واجهة الويب التفاعلية لـ Tamarin
tamarin-prover interactive atheer_protocol.spthy

# ثم افتح المتصفح على:
# http://localhost:3001
```

عبر الواجهة التفاعلية يمكنك:
- تصفح آثار الهجمات بشكل مرئي
- فحص أشجار الإثبات لكل Lemma
- تصور مخططات تسلسل الرسائل
- توجيه البحث عن الإثبات يدوياً عند الحاجة

### الناتج المتوقع

```
==============================================================================
ملخص النتائج:

analyzed: atheer_protocol.spthy

  Executability          (exists-trace):  verified (1 step)
  Authentication         (all-traces):    verified (12 steps)
  Anti_Replay            (all-traces):    verified (3 steps)
  Session_Integrity      (all-traces):    verified (8 steps)
  LUK_Secrecy            (all-traces):    verified (6 steps)
  Merchant_ZeroKnowledge (all-traces):    verified (4 steps)
  Switch_NonFrameability (all-traces):    verified (7 steps)

==============================================================================
```

---

## هيكل المستودع

```
atheer-formal-verification/
│
├── atheer_protocol.spthy          # ملف نموذج Tamarin الرئيسي
│                                  # يحتوي على القواعد والقيود والبراهين
│
├── proofs/                        # مخرجات الإثبات المُحسَبة مسبقاً
│   └── .gitkeep
│
├── README.md                      # هذا الملف (بالعربية)
├── FORMAL_ANALYSIS_REPORT.md      # تقرير التحليل الأمني التفصيلي (بالعربية)
└── LICENSE                        # ترخيص MIT
```

---

## فهم النموذج

### هرمية الأطراف

```
أطراف بروتوكول أثير:
  - العميل (C):    يمتلك مفتاح التوقيع sk_tee المقيم في TEE
  - التاجر (M):    SoftPOS، مُحيل بدون معرفة (Zero-Knowledge)، يمتلك sk_merch
  - السويتش (S):   بوابة أثير، يمتلك Switch_Registry
  - المهاجم:       Dolev-Yao، يتحكم في جميع قنوات الشبكة
```

### نموذج القنوات

| القناة | نمذجة Tamarin | قدرة المهاجم |
|--------|--------------|-------------|
| توفير TLS (المرحلة 1) | `Out/In` (مُصادَق عليه) | يراقب، لا يزوّر |
| APDU عبر NFC (المرحلة 2) | `Out/In` (محلي) | يعترض، لا يزوّر بدون مفتاح TEE |
| APN الخاص (المرحلة 3) | حقيقة `SentOnAPN` | يراقب، لا يحقن بدون مفتاح التاجر |

### الحقائق الرئيسية المستخدمة

| الحقيقة | دائمة؟ | المعنى |
|---------|--------|--------|
| `!TEE_PrivKey(dev, sk)` | نعم | المفتاح الخاص TEE مرتبط بالجهاز |
| `!LUK_Registry(id, tok, state)` | نعم (قابلة للتحديث) | حالة دورة حياة LUK |
| `ArmedSession(dev, sid, sk, ts)` | لا (خطية) | نافذة الـ 60 ثانية النشطة |
| `ConsumedLUK(token)` | حقيقة إجراء | تم استهلاك التوكن |

---

## الربط بكود أثير

| عنصر النموذج | مكوّن SDK | مكوّن Switch |
|-------------|----------|-------------|
| قاعدة `TEE_KeyGen` | `AtheerKeystoreManager.kt` | — |
| قاعدة `Client_Biometric_Auth` | `AtheerPaymentSession.kt` | — |
| قاعدة `Client_Send_APDU` | `AtheerApduService.kt` | — |
| قاعدة `Merchant_Receive_APDU` | `AtheerNfcReader.kt` | — |
| قاعدة `Switch_Verify_And_Settle` | — | `paymentController.js` |
| قيد `LUK_Single_Use` | سجل التوكن المحلي | جدول حالة LUK |
| قيد `Nonce_Uniqueness` | — | ذاكرة تخزين مؤقت للـ Nonce |

---

## الاستشهاد العلمي

إذا استخدمت هذا النموذج الرسمي في بحثك، يُرجى الاستشهاد بـ:

```bibtex
@inproceedings{almutawakel2026atheer,
  title     = {Eliminating Cloud Dependency in Mobile Payments via
               Isolated Cellular Routing and SoftPOS},
  author    = {Al-Mutawakel, Ahmed and Al-Mekhlafi, Nabil and Al-Fahidi, Bilal},
  booktitle = {University of Science and Technology, Sana'a},
  year      = {2026},
  note      = {أداة التحقق الرسمي متاحة على:
               \url{https://github.com/ahmedaliahmed775/atheer-formal-verification}}
}
```

---

## الترخيص

هذا المشروع مرخّص تحت رخصة MIT — انظر ملف [LICENSE](LICENSE) للتفاصيل.

---

*مستودع التحقق الرسمي لأثير — جامعة العلوم والتكنولوجيا، صنعاء، اليمن*

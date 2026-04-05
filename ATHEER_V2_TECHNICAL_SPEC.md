# Atheer Protocol V2 - Technical Specification

## 1. Overview
يقدم هذا المستند المعمارية الفنية المحدثة لبروتوكول "أثير" (Atheer Protocol V2) لعمليات الدفع عبر الهواتف الذكية دون اتصال بالإنترنت (Offline-First). تم تصميم هذه المعمارية لتكون عملية (Practical)، سريعة التكامل (Fast Integration)، فائقة الأمان، ومتوافقة مباشرة مع البنى التحتية الحالية للمحافظ الإلكترونية والبنوك.

## 2. Core Architectural Changes (V1 vs V2)

| Feature | V1 (Legacy) | V2 (Modern Fast-Track) |
| :--- | :--- | :--- |
| **Token Provisioning** | تنزيل حزم من توكنات الـ LUK مسبقاً لكل مستخدم وتخزينها كقوائم. | تنزيل "بذرة مركزية" (`Master Seed`) مشفرة مرة واحدة فقط واشتقاق التوكنات محلياً. |
| **Offline Limit Control** | غير متوفر - يعتمد على رفض السويتش لاحقاً إذا تجاوز الرصيد المتاح. | **Offline Limits:** يتم تمرير سقف الأوفلاين للـ TEE لرفض المبالغ الكبيرة قبل لمس الـ SoftPOS. |
| **Signatures** | توقيع أحادي من הـ TEE (PABC) | **Dual MPC Signature:** توقيع مشترك عبر (TEE + SIM Applet) لحماية فائقة ضد صلاحيات الروت الأندرويد. |
| **Timing & State** | يعتمد التوقيت على ساعة الهاتف `Timestamp`. | يعتمد على **Monotonic Hardware Counters** `uptimeMillis` لتجنب التلاعب الزمني. |
| **Anti-Relay** | غير مقيد بزمن توصيل. | **Distance Bounding (RTT):** قيد زمني لرحلة الذهاب والإياب بين الأجهزة. |
| **Revocation** | يتطلب حظر كل LUK على حدة. | **Seed Blacklisting:** إبطال سريع لـ Master Seed يوقف شجرة الارتباط بالكامل. |

## 3. Implementation Blueprint for Teams

### 3.1 Atheer Mobile SDK Integration
الـ SDK مسؤول عن إدارة البذرة (Seed) وضمان حماية الجلسة دون المساس بتجربة المستخدم.

- **Storage (AtheerTokenManager):**
  - يستلم הـ SDK البذرة המשفرة باستخدام خوارزميات أندرويد القياسية وتُخزن في `EncryptedSharedPreferences`.
  - يستخدم خوارزمية (HKDF) لاشتقاق التوكن (LUK) عند كل عملية دفع بناءً على عداد محلي متزامن وتحديد السقف.
  
- **Hardware Integration (AtheerPaymentSession):**
  - يستخدم `SystemClock.uptimeMillis()` لعد 60 ثانية بدقة مطلقة، لا تتأثر بتغييرات النظام.
  - يستدعي Play Integrity API للتحقق من عدم ترويت الجهاز (يتم التخزين المؤقت للشهادة لتسريع الأداء الأوفلاين).

- **The MPC Channel (Soft-SIM Interface):**
  - عبر واجهة `OMAPI` الـ SDK يتصل بالـ SIM يطلب نصف توقيع للكريبتوجرام.

### 3.1.5 Merchant Strict Online Requirement
> [!IMPORTANT]
> **قيد الاتصال الإلزامي للتاجر (Strict Online Merchant):**
> النظام لا يدعم سيناريو (Store and Forward) نهائياً. يُشترط معمارياً أن يكون تطبيق التاجر (SoftPOS) متصلاً بشبكة الـ APN الخاصة للتحقق اللحظي من السيرفر لحظة ملامسة הـ NFC. إذا كان التاجر أوفلاين، تُرْفَض العملية فوراً من قبل جهازه لحماية النظام من أي تزامن متأخر.

### 3.2 Atheer Switch Backend
الـ Switch أصبح نظاماً مرجعياً لحظياً بدلاً من خزان مستهلك للمساحة.

- **Seed Vault (tokenService.js):**
  - عند تسجيل الجهاز، يقوم הסيرفر بتوليد `Master Seed`، يخزن (Hash) الخاص به، ويرسل النسخة الأصلية مشفرة.
  
- **Instant Derivation Verifier:**
  - عند محاولة التاجر تنفيذ الـ ChargeRequest، يقوم السيرفر بحساب الـ LUK الخاص بالدورة المطلوبة رياضيًا ويطابقه.
  
- **Emergency Revocation:**
  - يتم تفعيل زر الإبطال (Revoke) الذي يمنع أي LUK قد يصدر من تلك البذرة بشكل حاسم.

## 4. Formal Verification (Tamarin)
الكود المحدث داخل `atheer_protocol.spthy` يمثل إثباتاً جبرياً قاطعاً على حتمية أمان هذه المواصفة ضد الخصوم من مستوى Dolev-Yao، متضمناً التحقق من خصائص (Authentication, Anti-Replay, RTT Distance Bounding, Securing Limits, and Zero-Knowledge Proofs).

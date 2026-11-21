<div dir="rtl">

# 🔥 مهارة Flutter + Firebase + BLoC الاحترافية للإنتاج

**حوّل أي أداة ذكاء اصطناعي برمجية (Claude Code، Cursor، Windsurf، Copilot، Codex، Gemini CLI) إلى مهندس Flutter & Firebase محترف.**

🌐 **[English README](README.md)** · **[دليل التثبيت الكامل](skills/flutter-firebase-bloc/INSTALL.md)**

---

## ⚡ التثبيت بأمر واحد فقط

```bash
npx skills add anwarsafy/flutter-firebase-bloc-skill
```

### أو عبر Claude Code Marketplace:
```text
/plugin marketplace add anwarsafy/flutter-firebase-bloc-skill
/plugin install flutter-firebase-bloc@flutter-firebase-bloc-marketplace
```

---

## 🛡️ ماذا تضمن لك هذه المهارة تلقائياً؟

1. **معمارية نظيفة صارمة (Clean Architecture):** طبقة `domain/` بلغة Dart خالصة بدون أي استيراد لـ Flutter أو Firebase.
2. **Offline-First حقيقي:** كتابات فورية بدون `await` على `Firestore` مع تتبع حالة المزامنة (`hasPendingWrites`) وبانر انقطاع الاتصال.
3. **أمان وحماية ضد الاختراق:**
   - منع ترقية الصلاحيات من العميل (`isPro`, `role`, `boardCount`) في `firestore.rules`.
   - منع كشط قواعد بيانات المستخدمين (`allow list: if false` على `publicProfiles`).
   - التحقق من الحقول والأحجام (`hasOnly` + `size()`) على الإنشاء والتعديل.
   - تفعيل `App Check` (`Play Integrity` / `App Attest` / `reCAPTCHA Enterprise`) + تخزين آمن `flutter_secure_storage` + ترويسات أمان الويب في `firebase.json`.
4. **حالة واحدة لكل BLoC + أخطاء مترجمة بالكامل (عربي RTL / إنجليزي):** تغطية شاملة لكل استثناءات Firebase مع الحفاظ على البيانات أثناء فشل التحديث.
5. **إشعارات خالية من التسريب + Cloud Functions v2 + GoRouter:** حارس `AuthStatus.unknown`، إشعارات الموبايل والويب (VAPID)، وروابط عميقة.
6. **Flavors + Remote Config + CI/CD احترافي:** بيئات `dev`/`staging`/`prod`، رفع رموز Crashlytics تلقائياً، واختبارات Unit وBLoC وWidget وEmulator.

---

## ⭐ ادعم المشروع
إذا أفادتك المهارة أو اختصرت عليك وقتاً في مشاريعك، لا تنسَ دعم المستودع بنجمة **⭐ Star** ومشاركته مع مجتمع Flutter!

</div>

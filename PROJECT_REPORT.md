# 📋 تقرير شامل — Enterprise HR Mobilization System
> **تاريخ التقرير:** 14 يوليو 2026  
> **نوع المشروع:** Google Apps Script (GAS) Web Application  
> **Script ID:** `1pqVB8IVoPNwVUnRZh1ZZVFM99TCZakD0pbOM8jFcPsPcHePdVrwKZjmF`

---

## 1. نظرة عامة على المشروع

نظام متكامل لإدارة عمليات استقدام وتجهيز الكوادر البشرية (Mobilization)، مبني بالكامل على **Google Apps Script** ومتكامل مع Sheets وDrive وCalendar وEmail. يُقدَّم كـ Web App يُشغَّله المستخدم من داخل Google Workspace.

### الغرض الأساسي
- تتبع المرشحين من لحظة التسجيل حتى السفر (Mobilization)
- إدارة وثائق كل مرشح ورفعها على Google Drive
- مراجعة الوثائق والموافقة/الرفض عليها
- جدولة المتابعات عبر Google Calendar
- لوحة تحكم (Dashboard) تعرض KPIs لحظية
- نظام صلاحيات متعدد الأدوار (RBAC)

---

## 2. هيكل الملفات

```
المشروع/
├── Code.js              — Backend Controller (Entry Point & Dashboard API)
├── Database.js          — Data Layer (CRUD على Google Sheets)
├── DriveManager.js      — إدارة Google Drive (مجلدات + ملفات)
├── CalendarManager.js   — إدارة Google Calendar (أحداث المتابعة)
├── BackupService.js     — خدمة النسخ الاحتياطي اليومي
├── TestUtils.js         — أدوات الاختبار (Mocks & Assertions)
├── Tests.js             — مجموعة الاختبارات الوحدوية (Unit Tests)
├── ProductionTest.js    — اختبارات التكامل على الخدمات الحقيقية
├── Index.html           — Shell الصفحة الرئيسية (HTML هيكل)
├── Script.html          — كامل منطق الـ Frontend (JS ~3200 سطر)
├── Styles.html          — نظام التصميم (CSS ~1436 سطر)
└── appsscript.json      — إعدادات المشروع والصلاحيات (OAuth Scopes)
```

---

## 3. الـ Backend — تفصيل كل ملف

### 3.1 `Code.js` — Backend Controller
**الحجم:** 252 سطر

| الدالة | الدور | الصلاحيات |
|--------|-------|-----------|
| `doGet(e)` | Entry point للـ Web App — يُقدِّم Index.html | عام |
| `include(filename)` | يحقن Styles.html و Script.html داخل Index | داخلي |
| `api_getDashboardData()` | يُجمِّع كل KPIs من Sheets + Calendar | كل الأدوار |

**ميزات `api_getDashboardData`:**
- **Cache بـ CacheService:** مدة 60 ثانية (كـ fallback)
- **Invalidation يدوي:** كل دالة كتابة تستدعي `cache.remove('dashboard_data')` قبل الـ return
- **KPIs المُحسَبة:**
  - `activeCount` — المرشحون غير المُغلَقين
  - `missingDocs` — من لم يكملوا الوثائق
  - `visaPending / visaCompleted / mobilized`
  - `pendingMedical / bookedMedical / docsUnderPreparing`
  - `pendingValidation` — مرشحون لديهم وثائق بانتظار المراجعة
  - `hasPassport / hasPhoto / hasAcademicCert / hasMedicalExam / hasMedicalAnalysis / hasVisa / hasCV`
  - `missing[كل نوع وثيقة]`
  - `eventsToday / eventsTomorrow / eventsThisWeek / eventsOverdue / eventsHighPriority`

---

### 3.2 `Database.js` — Data Layer
**الحجم:** 452 سطر

#### جداول Google Sheets المستخدمة

| اسم الجدول | الأعمدة |
|-----------|--------|
| `tbl_Candidates` | CandidateID, FullName, Position, Department, Email, Phone, Nationality, OfferSalary, AssignedCoordinatorEmail, CurrentStatus, CreatedAt, UpdatedAt, DriveFolderID, Notes, LocalServerPath, HR_Code, Recruitment_Type, Batch_Number **(18 عمود)** |
| `tbl_Documents` | DocumentID, CandidateID, DocType, FileName, FileURL, UploadDate, ApprovalStatus, ApprovedBy, VersionNumber, Remarks |
| `tbl_Users` | UserID, Email, Role, Name |
| `tbl_SystemLogs` | LogID, Timestamp, CandidateID, Actor, Event |
| `tbl_Events` | EventID, CandidateID, CandidateName, Title, Description, EventDate, EventTime, Priority, ReminderMinutesBefore, Status, GoogleCalendarEventId, GoogleCalendarLink, CreatedBy, CreatedAt, UpdatedAt |

#### دوال الـ CRUD

| الدالة | الوصف | الأدوار المسموحة |
|--------|-------|----------------|
| `getCurrentUserRole_()` | يستخرج دور المستخدم الحالي من tbl_Users | داخلي |
| `requireRole_(allowedRoles)` | RBAC Guard — يُعيد `{authorized, role, error}` | داخلي |
| `getSpreadsheet_()` | يفتح الـ Spreadsheet مرة واحدة (مُخزَّن في cache) | داخلي |
| `getSheet_(sheetName)` | يُعيد sheet بالاسم أو يُلقي خطأ | داخلي |
| `generateUUID_()` | يُولِّد UUID فريد لكل record | داخلي |
| `api_getAllCandidates()` | كل المرشحين كـ array of objects | الكل |
| `api_createCandidate(data)` | إضافة مرشح جديد (18 حقل) | Admin, HR |
| `api_updateCandidateDetails(id, updates)` | تحديث بيانات المرشح (Phone, Notes, LocalServerPath, HR_Code, Recruitment_Type) | Admin, HR, Coordinator |
| `api_updateCandidateStatus(id, newStatus)` | تغيير الحالة (من قائمة بيضاء 13 حالة) | Admin, HR, Coordinator |
| `api_getDocumentsByCandidate(id)` | وثائق مرشح معين | الكل |
| `api_reviewDocument(candidateId, documentId, approvalStatus, remarks)` | مراجعة وثيقة (Approved/Rejected) | Admin, HR |
| `api_getAllDocuments()` | كل الوثائق (للـ Dashboard KPIs) | الكل |
| `api_writeLog_(id, actor, event)` | كتابة سجل تدقيق (لا يُلقي استثناء) | داخلي |
| `api_getAuditLog(id)` | سجل تدقيق مرشح | Admin, HR |

#### حالات المرشح المسموح بها (13 حالة)
```
New Candidate → Documents Requested → Documents Under Preparing →
Pending Passport / Pending Photo / Pending Academic Certificate /
Pending Medical / Booked a medical examination →
Documents Complete → Visa Pending → Visa Completed → Mobilized → Closed
```

#### أنواع التوظيف المسموح بها
```
Internal | External
```

---

### 3.3 `DriveManager.js` — إدارة الملفات
**الحجم:** 200 سطر

| الدالة | الوصف | الأدوار |
|--------|-------|--------|
| `getRootFolder_()` | يجيب/يُنشئ مجلد "Oman Mobilization" الجذر (مع cache بـ PropertiesService) | داخلي |
| `api_createCandidateFolder(id, name)` | ينشئ مجلد للمرشح داخل الجذر (يمنع التكرار) | Admin, HR |
| `api_uploadFileToDrive(candidateId, folderId, docType, fileName, base64, mime)` | يرفع ملف base64 على Drive مع validation | Admin, HR, Coordinator |
| `api_writeDocumentRecord_(...)` | يكتب metadata الوثيقة في tbl_Documents (داخلي) | داخلي |
| `api_storeFolderIdForCandidate_(id, folderId)` | يحفظ DriveFolderID في صف المرشح | داخلي |

**قواعد رفع الملفات:**
- الأنواع المسموحة: `application/pdf` | `image/jpeg` | `image/png`
- الحجم الأقصى: **10 MB** (تُحسَب تقريبيًا من طول base64)
- إعادة الرفع: يُعيد تسمية الملف القديم بـ `[ARCHIVE]` بدلاً من الحذف
- الحالة الأولية للوثيقة: **`Pending Review`** دائماً

---

### 3.4 `CalendarManager.js` — أحداث المتابعة
**الحجم:** 456 سطر

| الدالة | الوصف | الأدوار |
|--------|-------|--------|
| `getAppCalendar_()` | يجيب/يُنشئ تقويم "HR Mobilization — Follow-ups" (مع cache) | داخلي |
| `getEventsSheet_()` | يجيب/يُنشئ tbl_Events تلقائياً إن لم تكن موجودة | داخلي |
| `getCandidatePhoneMap_()` | خريطة CandidateID → Phone للـ Events | داخلي |
| `normalizeEventDateString_(value)` | يُوحِّد تنسيق التاريخ إلى YYYY-MM-DD | داخلي |
| `authorizeCalendarScope()` | Helper لمنح صلاحية Calendar (يُشغَّل مرة واحدة) | يدوي |
| `api_createCalendarEvent(id, eventData)` | ينشئ حدث في Calendar ويحفظه في tbl_Events | Admin, HR, Coordinator |
| `api_getEventsByCandidate(id)` | أحداث مرشح مرتبة بالتاريخ + Phone | الكل |
| `api_getAllUpcomingEvents()` | كل الأحداث Active فقط + Phone | الكل |
| `api_updateCalendarEvent(id, updates)` | يحدِّث الحدث في Calendar وفي الـ Sheet | Admin, HR, Coordinator |
| `api_deleteCalendarEvent(id)` | حذف نهائي من Calendar ومن الـ Sheet | Admin, HR |
| `api_markEventCompleted(id)` | يُغيِّر Status إلى Completed | Admin, HR, Coordinator |

**ملاحظات التقويم:**
- مدة الحدث الزمني: 30 دقيقة تلقائياً
- الأحداث بدون وقت: تُنشأ كـ All-Day Event
- التذكيرات: Popup + Email بنفس المدة
- في حالة فشل Google Calendar: يُسجِّل التحذير ويكمل حفظ الـ Sheet (لا يُوقف العملية)

---

### 3.5 `BackupService.js` — النسخ الاحتياطي
**الحجم:** 33 سطر

| الدالة | الوصف |
|--------|-------|
| `createDailyBackup()` | ينسخ الـ Spreadsheet إلى مجلد Backup بـ timestamp |

**Script Properties المطلوبة:**
- `SPREADSHEET_ID` — ID الـ Spreadsheet
- `BACKUP_FOLDER_ID` — ID مجلد النسخ الاحتياطي

**عند الفشل:** يُرسل بريد تنبيه إلى admin تلقائياً.

---

## 4. الـ Frontend — تفصيل الواجهة

### 4.1 `Index.html` — هيكل الصفحة (38 سطراً فقط)
```html
<header id="topbar">          ← شريط أعلى (User info + Notifications)
<nav id="sidebar">            ← قائمة جانبية (4 أقسام)
<main id="app-view">          ← منطقة المحتوى الرئيسية
<div id="toast-container">    ← إشعارات Toast
<div id="modal-root">         ← Modals
```

### 4.2 `Script.html` — منطق الـ Frontend (~3200 سطر)

#### Objects الرئيسية

| Object | الدور |
|--------|-------|
| `App` | إدارة الـ State العام (مرشحون، فلاتر، KPIs، حالة Sidebar) |
| `GAS` | Bridge للاتصال بـ GAS backend عبر `google.script.run` + Mock للتطوير |
| `Toast` | إشعارات (success / error / warning / default) |
| `Modal` | نوافذ منبثقة مع overlay |
| `Router` | التنقل بين الـ Views |
| `Views` | رسم كل View (Dashboard, Candidates, Calendar, Audit, Detail) |
| `fmt` | تنسيق التواريخ والأحرف الأولى |

#### الـ Views المتاحة (4 + تفاصيل)

| View | المحتوى |
|------|---------|
| `dashboard` | KPI Cards + جدول مرشحين مفلتَر + Export (CSV/JSON/KPI) |
| `candidates` | قائمة المرشحين مع بحث وفلاتر وترتيب |
| `candidate-detail` | تفاصيل مرشح + وثائقه + أحداثه + سجل تدقيق |
| `new-candidate` | فورم إضافة مرشح جديد |
| `calendar` | عرض تقويم الأحداث |
| `audit` | سجل التدقيق العام |

#### ميزات الـ Dashboard بالتفصيل
- **KPI Cards** قابلة للنقر (تُفلتر الجدول)
- **Quick Filter Buttons** (Visa Pending, Visa Completed, Mobilized, Docs Complete)
- **Advanced Filters Panel:**
  - فلتر الحالة (تضمين)
  - فلتر الحالة (استثناء)
  - فلتر الوثائق المفقودة (Passport / Photo / Academic Certificate / Medical Examination / Medical Analysis / Visa / CV)
  - فلتر اكتمال الوثائق (0-100% بـ range slider)
  - فلتر القسم (Department)
  - فلتر الـ Batch Number
  - بحث نصي حر
- **Export:** CSV الجدول الحالي / JSON / تقرير KPI كـ CSV
- **Dark Mode** (محفوظ في localStorage)
- **Sidebar Collapse** (محفوظ في localStorage)
- **فلاتر محفوظة** في localStorage (تبقى بعد إعادة التحميل)

#### الـ GAS Bridge — الـ Mock
عند تشغيل الصفحة خارج GAS (للتطوير)، يُعيد `GAS._mockData()` بيانات مُسبَقة لـ 8 مرشحين وعشرات الوثائق. يدعم mock كل الـ API calls التالية:
```
api_getDashboardData, api_getAllCandidates, api_getAllDocuments,
api_createCandidate, api_updateCandidateStatus, api_updateCandidateDetails,
api_getDocumentsByCandidate, api_reviewDocument, api_getAuditLog,
api_createCandidateFolder, api_uploadFileToDrive,
api_sendRejectionEmail, api_sendWelcomeEmail,
api_getAllUpcomingEvents, api_getEventsByCandidate,
api_createCalendarEvent, api_updateCalendarEvent,
api_deleteCalendarEvent, api_markEventCompleted
```

### 4.3 `Styles.html` — نظام التصميم (~1436 سطر)
- **نمط Fluent UI / Azure-inspired**
- **CSS Variables** لكل الألوان والمسافات
- **Dark Mode كامل** (عبر class `dark-mode` على `body`)
- **Responsive** مع دعم Sidebar Collapsed
- **Badge System** لحالات المرشحين (12 لون مختلف)
- **Toast / Modal / Form Styles**

---

## 5. نظام الصلاحيات (RBAC)

| الدور | الوصف | ما يستطيع فعله |
|-------|-------|---------------|
| **Admin** | المدير العام | كل شيء |
| **HR** | موظف الموارد البشرية | إنشاء/تحديث المرشحين، رفع ومراجعة الوثائق، إدارة التقويم، الحذف |
| **Coordinator** | منسق | تحديث بيانات وحالات المرشحين، رفع وثائق، إنشاء/تحديث أحداث التقويم |
| **Viewer** | مشاهد | قراءة فقط (لا يمكنه تغيير أي شيء) |

---

## 6. الاختبارات

### 6.1 `Tests.js` — الاختبارات الوحدوية (27 اختبار)

| Suite | عدد الاختبارات | المغطَّى |
|-------|--------------|---------|
| `suite_Code` | 4 | api_getDashboardData (2)، api_uploadFileToDrive، include() |
| `suite_Database_Candidates` | 10 | getAllCandidates (3)، createCandidate (2)، updateStatus (3)، RBAC (2) |
| `suite_Database_Documents` | 3 | getDocuments (2)، وجود api_reviewDocument |
| `suite_Database_AuditLog` | 4 | getAuditLog (2)، تراكم السجلات، api_writeLog_ عند غياب الجدول |
| `suite_DriveManager` | 6 | createFolder (3)، uploadFile (2)، versioning |
| **المجموع** | **27** | **✅ كلها يجب أن تنجح** |

**الـ Test Runners:**
```javascript
runAllTests()    // يُشغِّل كل الـ 27 اختباراً
```

### 6.2 `TestUtils.js` — أدوات الاختبار

**MockFactory يُحاكي:**
- `SpreadsheetApp` (مع `getSheetByName` + `insertSheet` + `appendRow` + `getRange`)
- `DriveApp` (مع `getFoldersByName` + `createFolder` + `getFolderById` مع tracking)
- `Session` (مستخدم ثابت: `test.user@yourcompany.com`)
- `Utilities` (UUID عشوائي، base64، blob)
- `ScriptApp` (URL ثابت)

**Assert Helpers:** `equals`, `isTrue`, `isFalse`, `notNull`, `isNull`, `throws`

### 6.3 `ProductionTest.js` — اختبارات التكامل

**يُشغَّل على الخدمات الحقيقية (يحتاج SPREADSHEET_ID):**

| الخطوة | ما تفعله |
|--------|---------|
| STEP 1 | ينشئ مرشح تجريبي في Sheets |
| STEP 2 | ينشئ مجلد Drive حقيقي |
| STEP 3 | يرفع ملف PDF تجريبي |
| STEP 4 | يقرأ المرشح من Sheets للتحقق |
| STEP 5 | يقرأ سجل الوثائق |
| STEP 6 | يقرأ Audit Log |
| cleanup | `cleanupProductionTest(id)` — يمسح كل البيانات التجريبية |

---

## 7. الإعدادات والـ Configuration

### 7.1 `appsscript.json`
```json
{
  "timeZone": "Africa/Cairo",
  "runtimeVersion": "V8",
  "webapp": {
    "executeAs": "USER_DEPLOYING",
    "access": "ANYONE"
  }
}
```

**OAuth Scopes (5 صلاحيات):**

| Scope | الغرض |
|-------|-------|
| `auth/spreadsheets` | قراءة/كتابة Google Sheets |
| `auth/drive` | إدارة Google Drive |
| `auth/calendar` | إدارة Google Calendar |
| `auth/userinfo.email` | معرفة البريد الإلكتروني للمستخدم |
| `auth/script.send_mail` | إرسال البريد الإلكتروني (MailApp) |

### 7.2 `Script Properties` المطلوبة (تُضبط من Project Settings)

| المفتاح | الغرض |
|--------|-------|
| `SPREADSHEET_ID` | ID الـ Google Sheet الرئيسي |
| `BACKUP_FOLDER_ID` | ID مجلد النسخ الاحتياطي |
| `ROOT_FOLDER_ID` | *(auto-cached)* ID مجلد "Oman Mobilization" |
| `APP_CALENDAR_ID` | *(auto-cached)* ID تقويم المتابعات |

### 7.3 `.clasp.json`
```json
{
  "scriptId": "1pqVB8IVoPNwVUnRZh1ZZVFM99TCZakD0pbOM8jFcPsPcHePdVrwKZjmF",
  "rootDir": "",
  "scriptExtensions": [".js", ".gs"],
  "htmlExtensions": [".html"]
}
```
يُستخدم `clasp` (Command Line Apps Script) لمزامنة الكود بين Replit وGoogle Apps Script.

---

## 8. إحصائيات الكود

| الملف | عدد الأسطر | عدد الدوال |
|-------|-----------|-----------|
| Script.html | 3,209 | ~80 function/method |
| Styles.html | 1,436 | — (CSS فقط) |
| CalendarManager.js | 456 | 10 |
| Database.js | 452 | 14 |
| Tests.js | 513 | 8 suites + helpers |
| TestUtils.js | 307 | MockFactory + Assert + TestRunner |
| Code.js | 252 | 3 |
| ProductionTest.js | 258 | 2 |
| DriveManager.js | 200 | 6 |
| BackupService.js | 33 | 1 |
| Index.html | 38 | — (HTML فقط) |
| **المجموع** | **~7,154** | **~130+** |

---

## 9. الملاحظات والمشاكل المكتشفة

### 9.1 ✅ ما يعمل بشكل صحيح
- نظام الصلاحيات RBAC مُطبَّق على كل الـ API functions
- كل عمليات الكتابة تُبطل الـ Cache تلقائياً
- الـ Audit Log يُسجَّل لكل عملية تعديل
- الوثائق لا تُحذف — تُعاد تسميتها بـ `[ARCHIVE]`
- `api_writeLog_` لا يُوقف التطبيق في حالة فشله
- `getEventsSheet_()` يُنشئ الجدول تلقائياً إن لم يكن موجوداً
- `getRootFolder_()` و `getAppCalendar_()` يستخدمان PropertiesService cache
- الـ Frontend Mock يغطي كل الـ API functions

### 9.2 ⚠️ ملاحظات تحتاج انتباهاً

| # | الملاحظة | الملف | الخطورة |
|---|---------|-------|--------|
| 1 | seed rows في Tests.js تحتوي **17 قيمة** بدلاً من 18 (مفيش Batch_Number) | Tests.js سطر 121-125 و 323-328 | منخفضة — لا تُسبب فشل الاختبارات |
| 2 | `EmailService.js` مذكور في Tests.js header (`api_sendRejectionEmail`, `api_sendPackageSubmissionAlert`) لكن **الملف غير موجود** في المشروع | Tests.js سطر 11-13 | متوسطة — الوظيفة غير مكتملة |
| 3 | `api_updateCalendarEvent` يقرأ الـ Spreadsheet مرتين (مرة للـ Event ومرة للـ Candidate Status) دون استخدام `getSpreadsheet_()` cache | CalendarManager.js سطر 315-324 | منخفضة — أداء |
| 4 | `App.state.currentUser` في الـ Frontend يُظهر `role: 'HR_COORDINATOR'` كـ hardcoded — لا يُجلَب الدور الحقيقي من الـ Backend تلقائياً | Script.html سطر 20 | متوسطة |
| 5 | الـ `api_sendWelcomeEmail` و `api_sendRejectionEmail` موجودان في الـ Mock لكن غير موجودين في الـ Backend | Script.html سطر 201-202 | متوسطة |
| 6 | `notification-btn` موجود في الـ Topbar لكن لا يفعل شيئاً حالياً | Script.html سطر 359 | منخفضة |

### 9.3 ❌ ما هو مفقود كلياً
- **`EmailService.js`** — خدمة البريد الإلكتروني (مذكورة في Tests.js لكن غير موجودة)
- **وظيفة Notifications** — الزر موجود لكن بدون منطق
- **User Profile fetch** — الدور الحقيقي لا يُجلَب من الـ Backend
- **Delete/Archive candidate** — لا توجد وظيفة حذف مرشح في الـ Backend (إلا في cleanupProductionTest)

---

## 10. كيفية تشغيل وتطوير المشروع

### للتطوير المحلي (في Replit)
```bash
# مراجعة وتعديل الكود مباشرة
# لا يوجد server يعمل — الكود يُشغَّل داخل Google Apps Script فقط
```

### لرفع التعديلات لـ Google Apps Script
```bash
# يجب تثبيت clasp أولاً
npm install -g @google/clasp
clasp login
clasp push    # يرفع كل الملفات لـ GAS
```

### لتشغيل الاختبارات
1. افتح المشروع في [script.google.com](https://script.google.com)
2. اختر دالة `runAllTests` من القائمة المنسدلة
3. اضغط ▶ Run
4. افتح View → Logs لرؤية النتائج

### للاختبار التكاملي (Production Test)
1. تأكد أن `SPREADSHEET_ID` مضبوط في Script Properties
2. شغِّل `runProductionTest()`
3. بعد الانتهاء شغِّل `cleanupProductionTest("candidateId")`

---

## 11. ملخص حالة الاختبارات (بعد آخر تعديل)

| Suite | الاختبارات | الحالة المتوقعة |
|-------|-----------|----------------|
| suite_Code | 4 | ✅ كلها تنجح |
| suite_Database_Candidates | 10 | ✅ كلها تنجح |
| suite_Database_Documents | 3 | ✅ كلها تنجح |
| suite_Database_AuditLog | 4 | ✅ كلها تنجح |
| suite_DriveManager | 6 | ✅ كلها تنجح |
| **المجموع** | **27 / 27** | **✅ 100% Pass** |

---

*تم إعداد هذا التقرير بتاريخ 14 يوليو 2026 بناءً على قراءة وتحليل كل ملفات المشروع.*

# Node.js `util` Module: دليل شامل للمطورين المتقدمين

**نظرة عامة حول الوحدة**: وحدة `node:util` توفر مجموعة من الدوال والفئات المساعدة التي تم تصميمها في الأساس لدعم احتياجات Node.js الداخلية. وعلى الرغم من ذلك، فإن العديد من هذه الأدوات مفيدة جداً لمطوري التطبيقات والوحدات أيضاً. الوحدة مستقرة وموثقة بشكل جيد، وتغطي حالات استخدام متنوعة تتراوح بين تحويل الدوال غير المتزامنة إلى وعود، وفحص الأنواع، وإنشاء رسائل مُنسقة للتصحيح.[^1_1]

## 1. نظرة عامة على الوحدة (Module Overview)

### متى تستخدم `node:util`؟

وحدة `util` هي الخيار الأمثل في الحالات التالية:[^1_2]

1. **تحويل الدوال القائمة على callbacks إلى Promises**: عندما تعمل مع مكتبات قديمة تستخدم نمط callback (مثل نسخ قديمة من `fs`)، يمكنك استخدام `util.promisify()` لتحويلها إلى دوال حديثة تدعم `async/await`.[^1_3]
2. **تصحيح الأخطاء (Debugging)**: يوفر `util.inspect()` تمثيلاً قابلاً للقراءة للكائنات المعقدة، خاصة تلك التي تحتوي على مراجع دائرية (circular references).[^1_4]
3. **معالجة أخطاء النظام**: للحصول على أسماء وأوصاف معيارية للأخطاء في نظام التشغيل.
4. **فحص الأنواع**: وحدة `util.types` توفر فحوصات نوع دقيقة لا توفرها JavaScript العادية.
5. **معالجة MIME Types**: الفئة `MIMEType` توفر واجهة برمجية معيارية للتعامل مع أنواع MIME.

**بدائل خارجية**: بالمقارنة مع مكتبات مثل `lodash` أو `chalk`:

- `util.inspect()` أقل قوة من `util` في المكتبات الخارجية، لكنه مدمج بدون تبعيات
- `util.parseArgs()` يوفر معالجة أساسية لسطر الأوامر، لكن `yargs` أو `commander` أقوى للتطبيقات المعقدة
- `util.debuglog()` خفيف الوزن لكن `winston` أو `pino` أفضل للتطبيقات الإنتاجية الكبيرة

---

## 2. تفاصيل الدوال والخصائص (API Deep Dive)

![Node.js util Module: Complete API Hierarchy and Function Categories](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/f68cc66dea52146f503895081ae6ca0d/f8eb1f5c-b37e-41f4-b429-1adefc531970/95ace40a.png)

Node.js util Module: Complete API Hierarchy and Function Categories

### أ) دوال تحويل الـ Callbacks والـ Promises

#### `util.promisify(original)`

**الدالة**:

```javascript
util.promisify(original: Function): Function
```

**الوصف**: تحول دالة قائمة على callback (بنمط error-first) إلى دالة ترجع Promise. هذه من أكثر الدوال استخداماً في Node.js الحديث.[^1_3]

| المعامل    | النوع    | مطلوب/اختياري | الوصف                                                                     |
| :--------- | :------- | :------------ | :------------------------------------------------------------------------ |
| `original` | Function | مطلوب         | دالة callback-based تتوقع callback كآخر معامل بصيغة `(err, result) => {}` |

**القيمة المرجعة**: دالة جديدة ترجع Promise بدلاً من استقبال callback.

**مثال واقعي - معالجة الملفات**:

```javascript
const fs = require("fs");
const util = require("util");

// تحويل fs.readFile من callback إلى Promise
const readFile = util.promisify(fs.readFile);

// الاستخدام مع async/await
async function processImageFile(imagePath) {
  try {
    // قراءة ملف الصورة (مثال: معالجة صورة JPEG)
    const imageBuffer = await readFile(imagePath);

    console.log(`Image size: ${imageBuffer.length} bytes`);

    // يمكن إرسالها إلى خدمة معالجة الصور
    // const processed = await imageProcessingService.resize(imageBuffer);

    return imageBuffer;
  } catch (err) {
    console.error(`Failed to read image: ${err.message}`);
    throw err;
  }
}

// الاستدعاء
processImageFile("./photo.jpg");
```

**الأخطاء الشائعة**:

1. **نسيان ربط `this`**: عند استخدام `promisify` مع methods من كائنات، يجب استخدام `.bind()`:

```javascript
const db = require("sqlite3");
const getUser = util.promisify(db.get.bind(db)); // خطأ شائع: نسيان .bind
```

2. **افتراض أن الدالة الأصلية تتبع نمط error-first**: `promisify` يتوقع أن يكون آخر معامل callback بصيغة `(err, result) => {}`.
3. **عدم التعامل مع الأخطاء**: عندما ترفع Promise، يجب استخدام `try/catch` أو `.catch()`.

---

#### `util.callbackify(original)`

**الدالة**:

```javascript
util.callbackify(original: AsyncFunction): Function
```

**الوصف**: تحول دالة async (أو دالة ترجع Promise) إلى دالة callback-based. هذا معاكس `promisify()` تماماً.[^1_5]

| المعامل    | النوع    | مطلوب/اختياري | الوصف                      |
| :--------- | :------- | :------------ | :------------------------- |
| `original` | Function | مطلوب         | دالة async أو ترجع Promise |

**القيمة المرجعة**: دالة جديدة تستقبل callback من نوع `(err, result) => {}`.

**مثال واقعي - واجهة برمجية حديثة لنظام قديم**:

```javascript
const util = require("util");

// دالة حديثة (async/await)
async function validateUser(userId) {
  if (!userId || userId < 1) {
    throw new Error("Invalid user ID");
  }

  // محاكاة استدعاء قاعدة بيانات
  return { id: userId, name: "Ahmed", email: `user${userId}@example.com` };
}

// تحويل لاستخدام مع الكود القديم
const validateUserCallback = util.callbackify(validateUser);

// استخدام مع نمط callback القديم (مثل في Express middleware قديم)
function legacyRoute(req, res) {
  const userId = req.params.id;

  validateUserCallback(userId, (err, user) => {
    if (err) {
      return res.status(400).json({ error: err.message });
    }
    res.json({ success: true, user });
  });
}

// الاستدعاء
validateUserCallback(5, (err, user) => {
  if (err) console.error(err);
  else console.log(user); // { id: 5, name: 'Ahmed', ... }
});
```

**الأخطاء الشائعة**:

1. **رفع Promise بقيمة falsy** (مثل `null`): Node.js سيلفها في `Error` بخاصية `reason`:

```javascript
async function fn() {
  throw null; // سيصبح Error مع reason: null
}
const cb = util.callbackify(fn);
cb((err, result) => {
  console.log(err.reason); // null
});
```

---

### ب) دوال التنسيق والفحص (Formatting \& Inspection)

#### `util.format(format[, ...args])`

**الدالة**:

```javascript
util.format(format: string, ...args): string
```

**الوصف**: تنسق سلسلة نصية بصيغة `printf`-like مع معاملات متعددة. تدعم مواصفات خاصة مثل `%s`, `%d`, إلخ.[^1_6]

| معرّف | الغرض                    | المثال                                        |
| :---- | :----------------------- | :-------------------------------------------- |
| `%s`  | سلسلة نصية               | `'Name: %s'` مع `'Ahmed'` → `'Name: Ahmed'`   |
| `%d`  | رقم عددي                 | `'Count: %d'` مع `42` → `'Count: 42'`         |
| `%i`  | عدد صحيح (parseInt)      | `'Port: %i'` مع `'8080'` → `'Port: 8080'`     |
| `%f`  | عدد عشري (parseFloat)    | `'Price: %f'` مع `'19.99'` → `'Price: 19.99'` |
| `%j`  | JSON                     | `'Data: %j'` مع `{a:1}` → `'Data: {"a":1}'`   |
| `%o`  | Object مع non-enumerable | مفيد للتصحيح                                  |
| `%%`  | علامة النسبة             | `'50%%'` → `'50%'`                            |

**مثال واقعي - تسجيل بيانات معقدة في السجلات**:

```javascript
const util = require("util");

// محاكاة معالج سجلات
class LogService {
  logUserAction(userId, action, metadata) {
    const timestamp = new Date().toISOString();
    const formatted = util.format(
      '[%s] User %d performed action "%s" with data: %j',
      timestamp,
      userId,
      action,
      metadata
    );
    console.log(formatted);
  }
}

const logger = new LogService();
logger.logUserAction(101, "login", { ip: "192.168.1.1", device: "mobile" });
// Output: [2025-12-31T10:11:00.000Z] User 101 performed action "login" with data: {"ip":"192.168.1.1","device":"mobile"}
```

**الأخطاء الشابعة**:

1. **عدم تطابق عدد المعاملات**: إذا لم يكن هناك معامل لكل مواصفة، تُترك المواصفة كما هي:

```javascript
util.format("%s:%s", "foo"); // 'foo:%s'
```

2. **استخدام في حلقات tight**: `util.format` بطيء ويمكن أن يقفل event loop. تجنبه في مسارات الكود الحساسة للأداء.

---

#### `util.inspect(object[, options])`

**الدالة**:

```javascript
util.inspect(object: any, options?: InspectOptions): string
```

**الوصف**: ترجع تمثيلاً نصياً لكائن بصيغة قابلة للقراءة، مخصصة للتصحيح. هذا هو نفس الذي يستخدمه `console.log` داخلياً.[^1_4]

| الخيار             | النوع               | القيمة الافتراضية | الوصف                                             |
| :----------------- | :------------------ | :---------------- | :------------------------------------------------ |
| `showHidden`       | boolean             | `false`           | إظهار الخصائص non-enumerable والـ WeakMap entries |
| `depth`            | number \| null      | `2`               | عمق التكرار في الكائنات المتداخلة                 |
| `colors`           | boolean             | `false`           | استخدام أكواد ANSI للألوان                        |
| `customInspect`    | boolean             | `true`            | استدعاء دوال `[util.inspect.custom]()` المخصصة    |
| `showProxy`        | boolean             | `false`           | إظهار `target` و `handler` للـ Proxies            |
| `maxArrayLength`   | integer             | `100`             | أقصى عدد عناصر في المصفوفات                       |
| `maxStringLength`  | integer             | `10000`           | أقصى عدد أحرف في السلاسل النصية                   |
| `breakLength`      | integer             | `80`              | طول السطر قبل كسره إلى أسطر متعددة                |
| `compact`          | boolean \| integer  | `3`               | تكثيف التنسيق                                     |
| `sorted`           | boolean \| Function | `false`           | ترتيب خصائص الكائن                                |
| `numericSeparator` | boolean             | `false`           | إضافة `_` كل 3 أرقام (مثل `1_000_000`)            |

**مثال واقعي - تصحيح بيانات معقدة**:

```javascript
const util = require("util");

class UserDatabase {
  inspectUserRecord(user) {
    // الكائن له مراجع دائرية
    const record = {
      id: 1,
      name: "خالد",
      email: "khaled@example.com",
      permissions: ["read", "write", "admin"],
      metadata: {
        created: new Date("2025-01-01"),
        updated: new Date("2025-12-31"),
        config: { retries: 3, timeout: 5000 },
      },
    };

    // أضف مرجع دائري
    record.self = record;

    // تصحيح مع رؤية المراجع الدائرية بوضوح
    console.log(
      util.inspect(record, {
        depth: null, // بلا حد للعمق
        colors: true, // ألوان للقراءة
        breakLength: 60, // أسطر أقصر
      })
    );
  }
}

const db = new UserDatabase();
db.inspectUserRecord();

/* الإخراج:
{
  id: 1,
  name: 'خالد',
  email: 'khaled@example.com',
  permissions: [ 'read', 'write', 'admin' ],
  metadata: {
    created: 2025-01-01T00:00:00.000Z,
    updated: 2025-12-31T00:00:00.000Z,
    config: { retries: 3, timeout: 5000 }
  },
  self: <ref *1> [Circular]
}
*/
```

**دالة فحص مخصصة (Custom Inspect Function)**:

```javascript
const util = require("util");

class PasswordHasher {
  constructor(plaintext) {
    this.plaintext = plaintext;
    this.hash = "sha256:" + Buffer.from(plaintext).toString("base64");
  }

  // دالة فحص مخصصة - لا تكشف كلمة السر
  [util.inspect.custom]() {
    return `PasswordHasher [hash: ${this.hash.substring(0, 20)}...]`;
  }
}

const pwd = new PasswordHasher("SuperSecret123!");
console.log(util.inspect(pwd));
// Output: PasswordHasher [hash: sha256:U3VwZXJTZWNyZXQx...]
```

**الأخطاء الشائعة**:

1. **عدم ضبط الأعماق للكائنات الكبيرة**: الإعدادات الافتراضية قد تقطع المعلومات المهمة:

```javascript
const deep = { a: { b: { c: { d: "value" } } } };
console.log(util.inspect(deep)); // يقتطع عند العمق 2
console.log(util.inspect(deep, { depth: null })); // عرض كامل
```

2. **الأداء في الحلقات**: `inspect` بطيء. استخدمه للتصحيح فقط، ليس للبيانات الإنتاجية.

---

#### `util.isDeepStrictEqual(val1, val2[, options])`

**الدالة**:

```javascript
util.isDeepStrictEqual(val1: any, val2: any, options?: { skipPrototype?: boolean }): boolean
```

**الوصف**: مقارنة عميقة وصارمة بين قيمتين، تفحص الأنواع والهياكل والقيم.[^1_7]

| المعامل   | النوع  | مطلوب/اختياري | الوصف                                                     |
| :-------- | :----- | :------------ | :-------------------------------------------------------- |
| `val1`    | any    | مطلوب         | أول قيمة للمقارنة                                         |
| `val2`    | any    | مطلوب         | ثاني قيمة للمقارنة                                        |
| `options` | Object | اختياري       | `{ skipPrototype: boolean }` - تجاهل مقارنة الـ prototype |

**القيمة المرجعة**: `true` إذا كانت القيم متساوية بشكل عميق وصارم، وإلا `false`.

**مثال واقعي - التحقق من صحة البيانات في الاختبارات**:

```javascript
const util = require("util");
const assert = require("assert");

class PaymentValidator {
  validateTransactionData(actualTransaction, expectedTransaction) {
    // استخدام عميق لمقارنة الكائنات المعقدة
    if (!util.isDeepStrictEqual(actualTransaction, expectedTransaction)) {
      throw new Error("Transaction data mismatch");
    }

    console.log("✓ Transaction data is valid");
  }
}

// مثال الاستخدام
const validator = new PaymentValidator();

const actual = {
  id: "12345",
  amount: 99.99,
  currency: "USD",
  status: "completed",
  metadata: { retries: 2, timestamp: "2025-12-31T10:11:00Z" },
};

const expected = {
  id: "12345",
  amount: 99.99,
  currency: "USD",
  status: "completed",
  metadata: { retries: 2, timestamp: "2025-12-31T10:11:00Z" },
};

validator.validateTransactionData(actual, expected);

// الأخطاء الشائعة:
// 1. مقارنة 1 مع '1' - سيرجع false (صارمة)
util.isDeepStrictEqual(1, "1"); // false - أنواع مختلفة

// 2. NaN يُعتبر مساوياً لـ NaN (استثناء Node.js)
util.isDeepStrictEqual(NaN, NaN); // true

// 3. -0 و 0 ليسا متساويين
util.isDeepStrictEqual(0, -0); // false
```

**الأخطاء الشائعة**:

1. **خلط مع `==` أو `===`**: هذه دالة عميقة تفحص الهيكل بالكامل، ليست مقارنة سطحية.
2. **نسيان الخيار `skipPrototype`**: إذا أردت مقارنة كائنات بـ prototypes مختلفة:

```javascript
class A {}
class B {}
const a = new A(),
  b = new B();
util.isDeepStrictEqual(a, b); // false (prototypes مختلفة)
util.isDeepStrictEqual(a, b, { skipPrototype: true }); // true (الخصائص متساوية)
```

---

### ج) دوال التصحيح والإهمال

#### `util.debuglog(section[, callback])`

**الدالة**:

```javascript
util.debuglog(section: string, callback?: Function): Function
```

**الوصف**: تنشئ دالة تسجيل مشروطة بناءً على متغير البيئة `NODE_DEBUG`. مفيدة للتصحيح بدون حذف التعليقات من الكود.[^1_8]

| المعامل    | النوع    | مطلوب/اختياري | الوصف                                   |
| :--------- | :------- | :------------ | :-------------------------------------- |
| `section`  | string   | مطلوب         | اسم القسم (مثل `'app'` أو `'database'`) |
| `callback` | Function | اختياري       | دالة تُستدعى في المرة الأولى للتحسين    |

**القيمة المرجعة**: دالة logging يمكن استدعاؤها مثل `console.log`.

**مثال واقعي - تصحيح تطبيق ويب معقد**:

```javascript
const util = require("util");

// في ملف api.js
const debugAPI = util.debuglog("api");

// في ملف database.js
const debugDB = util.debuglog("db");

function fetchUserFromDatabase(userId) {
  debugDB("Querying database for user ID: %d", userId);

  // ... كود الاستعلام ...

  debugDB("Database query completed in %d ms", 45);
  return { id: userId, name: "محمد" };
}

function handleUserRequest(req, res) {
  const userId = req.params.id;
  debugAPI("Received request for user: %s", userId);

  const user = fetchUserFromDatabase(userId);
  debugAPI("Sending response: %j", user);
  res.json(user);
}

// الاستخدام:
// بدون تفعيل: لا شيء يُطبع
handleUserRequest({ params: { id: 5 } }, {});

// مع التفعيل:
// $ NODE_DEBUG=api,db node app.js
// API 1234: Received request for user: 5
// DB 1234: Querying database for user ID: 5
// DB 1234: Database query completed in 45 ms
// API 1234: Sending response: {"id":5,"name":"محمد"}
```

**الخاصية `debuglog().enabled`**:

```javascript
const log = util.debuglog("myapp");

if (log.enabled) {
  // العملية مكلفة - قم بها فقط إذا كان التصحيح مفعلاً
  const expensiveDebugData = JSON.stringify(largeObject);
  log("Debug data: %s", expensiveDebugData);
}
```

**الأخطاء الشائعة**:

1. **عدم استخدام wildcards**: `NODE_DEBUG=*` لتفعيل جميع الأقسام.
2. **عدم التحقق من `enabled` للعمليات المكلفة**: قد تؤثر على الأداء حتى بدون طباعة.

---

#### `util.diff(actual, expected)`

**الدالة**:

```javascript
util.diff(actual: string | Array, expected: string | Array): Array
```

**الوصف**: تحسب الفروقات بين سلسلتين أو مصفوفتين باستخدام خوارزمية Myers (نفسها المستخدمة في `assert.deepStrictEqual()`).[^1_9]

| المعامل    | النوع           | مطلوب/اختياري | الوصف           |
| :--------- | :-------------- | :------------ | :-------------- |
| `actual`   | string \| Array | مطلوب         | القيمة الفعلية  |
| `expected` | string \| Array | مطلوب         | القيمة المتوقعة |

**القيمة المرجعة**: مصفوفة من الفروقات، كل عنصر `[flag, value]`:

- `0` = قيمة مشتركة
- `1` = موجودة في `expected` فقط
- `-1` = موجودة في `actual` فقط

**مثال واقعي - مقارنة نتائج معالجة النصوص**:

```javascript
const util = require("util");

class TextDiffAnalyzer {
  analyzeChanges(originalText, modifiedText) {
    const diffs = util.diff(originalText, modifiedText);

    let output = "";
    diffs.forEach(([flag, value]) => {
      if (flag === 0) {
        output += value; // قيمة مشتركة - لا تغيير
      } else if (flag === 1) {
        output += `[+${value}]`; // مضافة
      } else if (flag === -1) {
        output += `[-${value}]`; // محذوفة
      }
    });

    return output;
  }
}

const analyzer = new TextDiffAnalyzer();

const original = "مرحبا بك في Node.js";
const modified = "مرحبا بك في Node.js v20";

console.log(analyzer.analyzeChanges(original, modified));
// مرحبا بك في Node.js[ v20]
```

**الأخطاء الشائعة**:

1. **عدم فهم الـ flags**: يجب أن تفهم معنى 0, 1, -1 قبل الاستخدام.
2. **استخدام مع بيانات ضخمة**: الخوارزمية معقدة O(N\*D)، قد تكون بطيئة.

---

### د) فئات (Classes) للتعامل مع أنواع البيانات المتقدمة

#### `util.MIMEType`

**الفئة**:

```javascript
new MIMEType(input: string)
```

**الوصف**: فئة معيارية لتحليل وتعديل أنواع MIME (مثل `text/html`, `application/json`). توفر واجهة برمجية آمنة وموثوقة.[^1_9]

| الخاصية   | النوع      | قراءة/كتابة | الوصف                                         |
| :-------- | :--------- | :---------- | :-------------------------------------------- |
| `type`    | string     | قراءة/كتابة | النوع الأساسي (مثل `'text'`, `'application'`) |
| `subtype` | string     | قراءة/كتابة | النوع الفرعي (مثل `'html'`, `'json'`)         |
| `essence` | string     | قراءة فقط   | الشكل الكامل `type/subtype`                   |
| `params`  | MIMEParams | قراءة فقط   | المعاملات الإضافية (مثل `charset`)            |

**مثال واقعي - خادم ملفات ذكي**:

```javascript
const util = require("util");
const fs = require("fs");

class SmartFileServer {
  sendFile(filePath, mimeTypeString, res) {
    try {
      // تحليل MIME type
      const mime = new util.MIMEType(mimeTypeString);

      // التحقق من النوع والتعديل إذا لزم الأمر
      if (mime.type === "text") {
        // أضف charset للملفات النصية
        mime.params.set("charset", "utf-8");
      }

      // قراءة وإرسال الملف
      const fileContent = fs.readFileSync(filePath);
      res.setHeader("Content-Type", mime.toString());
      res.send(fileContent);

      console.log(`Sent ${mime.essence} file`);
    } catch (err) {
      console.error("Invalid MIME type:", err.message);
      res.status(400).send("Invalid MIME type");
    }
  }
}

// الاستخدام
const server = new SmartFileServer();

// مثال: معالجة ملف JSON
const mime = new util.MIMEType("application/json");
console.log(mime.type); // 'application'
console.log(mime.subtype); // 'json'
console.log(mime.essence); // 'application/json'
console.log(String(mime)); // 'application/json'

// تعديل الـ MIME type
mime.type = "text";
console.log(String(mime)); // 'text/json'
```

**الأخطاء الشائعة**:

1. **MIME types غير صحيحة**: ستطرح خطأ TypeError:

```javascript
new util.MIMEType("invalid"); // TypeError: Invalid MIME type
```

2. **نسيان أن الخصائص getters/setters**: لا تعديل مباشر:

```javascript
mime.type = "video"; // صحيح (setter)
mime.type; // قراءة (getter)
```

---

### هـ) فئات الترميز (Encoding Classes)

#### `util.TextEncoder` و `util.TextDecoder`

**الدوال**:

```javascript
new TextEncoder()
new TextDecoder([encoding[, options]])
```

**الوصف**: فئات معيارية لتحويل النصوص بين سلاسل JavaScript و `Uint8Array`.[^1_9]

**TextEncoder**:

```javascript
const util = require("util");

// تحويل نص إلى bytes
const encoder = new util.TextEncoder();
const text = "مرحباً بالعالم";
const bytes = encoder.encode(text);

console.log(bytes); // Uint8Array [ 216, 145, 216, 177, ... ]

// أو بطريقة سريعة
const { read, written } = encoder.encodeInto("Hello", new Uint8Array(10));
```

**TextDecoder**:

```javascript
const util = require("util");

// تحويل bytes إلى نص
const decoder = new util.TextDecoder("utf-8");
const bytes = new Uint8Array([72, 101, 108, 108, 111]);
const text = decoder.decode(bytes);

console.log(text); // 'Hello'

// مع streaming (معالجة ديناميكية)
const decoder2 = new util.TextDecoder("utf-8");
console.log(decoder2.decode(bytes1, { stream: true }));
console.log(decoder2.decode(bytes2, { stream: false })); // finalize
```

---

### و) فحص الأنواع (util.types)

**الوصف**: وحدة فرعية متقدمة لفحص أنواع البيانات. توفر فحوصات دقيقة لا توفرها `typeof` أو `instanceof`.[^1_2]

**أمثلة شائعة**:

```javascript
const util = require("util");

// فحص Promises
util.types.isPromise(Promise.resolve()); // true

// فحص Dates
util.types.isDate(new Date()); // true

// فحص Regular Expressions
util.types.isRegExp(/pattern/); // true

// فحص TypedArrays
util.types.isUint8Array(new Uint8Array()); // true
util.types.isInt32Array(new Int32Array()); // true

// فحص Maps و Sets
util.types.isMap(new Map()); // true
util.types.isSet(new Set()); // true

// فحص Generators
function* gen() {
  yield 1;
}
util.types.isGeneratorObject(gen()); // true

// فحص Proxies
const proxy = new Proxy({}, {});
util.types.isProxy(proxy); // true

// فحص WeakMaps و WeakSets
util.types.isWeakMap(new WeakMap()); // true
util.types.isWeakSet(new WeakSet()); // true
```

**الفائدة الحقيقية - فحص متقدم في middleware**:

```javascript
const util = require("util");

class RequestValidator {
  validateRequestBody(body) {
    // تحقق من أن body ليس Proxy (قد يكون تحتك هاك)
    if (util.types.isProxy(body)) {
      throw new Error("Invalid request body");
    }

    // تأكد من أن البيانات ليست Promise (خطأ async)
    if (util.types.isPromise(body)) {
      throw new Error("Body cannot be a Promise");
    }

    return true;
  }
}
```

---

## 3. مقارنات بين الدوال المتشابهة (Comparisons)

![Node.js util Module: Comparison of Similar Functions and Use Cases](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/f68cc66dea52146f503895081ae6ca0d/0e1c589e-4608-479a-ae3f-4867ad65d1df/ca97ade5.png)

Node.js util Module: Comparison of Similar Functions and Use Cases

### مقارنة `util.promisify` vs `util.callbackify`

| الجانب               | `util.promisify`                    | `util.callbackify`                |
| :------------------- | :---------------------------------- | :-------------------------------- |
| **الغرض**            | تحويل callback → Promise            | تحويل Promise/async → callback    |
| **الاستخدام الشائع** | تحديث كود قديم                      | واجهات قديمة مع كود حديث          |
| **المدخل**           | دالة callback-based                 | دالة async أو Promise-returning   |
| **المخرج**           | دالة ترجع Promise                   | دالة callback-based               |
| **الأداء**           | أسرع قليلاً                         | أبطأ قليلاً (wrapper overhead)    |
| **الحالات العملية**  | `fs.readFile`, `crypto.randomBytes` | مكتبات قديمة بـ callback-only API |

---

### مقارنة `util.inspect` vs `console.log`

| الجانب        | `util.inspect`      | `console.log` |
| :------------ | :------------------ | :------------ |
| **السيطرة**   | كاملة عبر options   | محدودة        |
| **المخرج**    | سلسلة نصية          | مطبوعة مباشرة |
| **الأداء**    | أبطأ (صيغة معقدة)   | أسرع          |
| **الاستخدام** | تصحيح متقدم         | سجلات سريعة   |
| **المرونة**   | دوال custom inspect | محدودة        |

---

### مقارنة `util.format` vs Template Literals

```javascript
// util.format
util.format("Hello %s, you have %d messages", name, count);

// Template Literals
`Hello ${name}, you have ${count} messages`;
```

| الجانب        | `util.format` | Template Literals |
| :------------ | :------------ | :---------------- |
| **الأداء**    | أبطأ          | أسرع              |
| **القراءة**   | مثل printf    | أكثر pythonic     |
| **الأنواع**   | تحويل تلقائي  | دقة أعلى          |
| **الاستخدام** | سجلات legacy  | كود حديث          |

---

## 4. المزايا والعيوب والحالات الاستخدام

### **المزايا** ✓

1. **بدون تبعيات خارجية**: كل شيء مدمج في Node.js الأساسي.[^1_2]
2. **معايير قياسية**: تتبع معايير Node.js وWeb Standards (مثل MIME types).[^1_9]
3. **أداء عالي**: دوال مُحسّنة للاستخدام الداخلي في Node.js.
4. **توافق عالي**: تعمل مع جميع الإصدارات الحديثة.
5. **سهولة التعلم**: واجهات برمجية بسيطة وموثقة بشكل جيد.

### **العيوب** ✗

1. **محدودة في المرونة**: مقارنة بـ lodash أو chalk.
2. **`inspect` بطيء في البيانات الضخمة**: قد يقفل event loop.[^1_8]
3. **معالجة سطر الأوامر بسيطة**: `parseArgs` أقل قوة من yargs/commander.[^1_10]
4. **عدم وجود معالج logs متقدم**: `debuglog` أساسي، قارن مع winston/pino.[^1_2]

### **أفضل 3 حالات استخدام**

#### 1️⃣ **تحويل المكتبات القديمة إلى async/await**

```javascript
// استخدام promisify لـ fs أو database callbacks
const readFile = util.promisify(fs.readFile);
const data = await readFile("file.txt");
```

**لماذا?** توافق كامل مع Node.js الحديثة بدون مكتبات خارجية.

---

#### 2️⃣ **التصحيح المتقدم للكائنات المعقدة**

```javascript
// استخدام inspect للدوال والكائنات ذات المراجع الدائرية
console.log(util.inspect(complexObject, { depth: null, colors: true }));
```

**لماذا?** أفضل من `JSON.stringify` لأنه يعالج الدوال والـ undefined والمراجع الدائرية.

---

#### 3️⃣ **مقارنة البيانات في الاختبارات**

````javascript
// استخدام isDeepStrictEqual بدلاً من JSON.stringify
if (util.isDeepStrictEqual(actual, expected)) { /* pass */ }
```**لماذا?** أكثر أماناً وسرعة من المقارنات اليدوية.

***

## 5. الخلاصة والتوصيات

وحدة `util` في Node.js هي أداة قوية وخفيفة الوزن لحالات استخدام محددة. الاستثمار في فهمها يحسن جودة الكود ويقلل التبعيات الخارجية. استخدمها للتحويل بين callback و Promise، التصحيح المتقدم، ومعالجة البيانات المعقدة، لكن اعتمد على مكتبات متخصصة للحالات الأكثر تعقيداً مثل معالجة سطر الأوامر المتقدمة أو التسجيل الإنتاجي الكامل.

**الخط الزمني للفهم**:
- **الأساسيات** (ساعة): `promisify`, `format`, `inspect`
- **المتقدمة** (3 ساعات): `callbackify`, `isDeepStrictEqual`, `parseArgs`
- **الخبير** (يوم): كل الدوال + الحالات الحدية + التحسينات
<span style="display:none">[^1_100][^1_101][^1_102][^1_103][^1_104][^1_105][^1_106][^1_107][^1_108][^1_109][^1_11][^1_110][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88][^1_89][^1_90][^1_91][^1_92][^1_93][^1_94][^1_95][^1_96][^1_97][^1_98][^1_99]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/util.md
[^1_2]: https://www.w3schools.com/nodejs/nodejs_util.asp
[^1_3]: https://www.geeksforgeeks.org/node-js/node-js-util-promisify-method/
[^1_4]: https://www.geeksforgeeks.org/node-js/node-js-util-inspect-method/
[^1_5]: https://mycuriosity.blog/nodejs-utilpromisify-convert-callbacks-to-promises
[^1_6]: https://nodejs.org/download/release/v9.2.0/docs/api/util.html
[^1_7]: https://www.codingtag.com/util-isdeepstrictequal-method-in-nodejs
[^1_8]: https://nodejs.org/download/release/v6.17.1/docs/api/util.html
[^1_9]: https://nodejs.org/api/util.html
[^1_10]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/spe.3313
[^1_11]: https://wepub.org/index.php/IJCSIT/article/view/4929
[^1_12]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_13]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_14]: https://ieeexplore.ieee.org/document/9424850/
[^1_15]: https://zenodo.org/record/836878
[^1_16]: https://ieeexplore.ieee.org/document/11209752/
[^1_17]: https://www.prci.org/315933.aspx
[^1_18]: https://joss.theoj.org/papers/10.21105/joss.04680
[^1_19]: https://www.semanticscholar.org/paper/6ba62f43564bff6da082a3e955cc9c02fc411b40
[^1_20]: https://jsret.knpub.com/index.php/jrest/article/view/920
[^1_21]: https://biblio.ugent.be/publication/8629595/file/8629596.pdf
[^1_22]: https://arxiv.org/pdf/2501.16945.pdf
[^1_23]: https://arxiv.org/pdf/2109.01002.pdf
[^1_24]: http://arxiv.org/pdf/2303.13041.pdf
[^1_25]: https://arxiv.org/pdf/1205.6363.pdf
[^1_26]: https://arxiv.org/pdf/2211.01473.pdf
[^1_27]: https://arxiv.org/pdf/2111.08684.pdf
[^1_28]: http://arxiv.org/pdf/2401.11361.pdf
[^1_29]: https://www.geeksforgeeks.org/node-js/node-js-util-isdeepstrictequal-method/
[^1_30]: https://stackoverflow.com/questions/10465423/how-can-i-list-all-the-functions-in-my-node-js-script
[^1_31]: https://node.readthedocs.io/en/stable/api/util/
[^1_32]: https://nodejs.org/download/release/v8.11.3/docs/api/util.html
[^1_33]: https://stackoverflow.com/questions/50409172/pro-and-cons-of-using-util-inspect-for-checking-deep-object-equality
[^1_34]: https://docs.deno.com/api/node/util/
[^1_35]: https://www.w3schools.com/nodejs/ref_modules.asp
[^1_36]: https://github.com/browserify/node-util/issues
[^1_37]: https://nodejs.org/docs/latest/api/util.html
[^1_38]: https://learn.microsoft.com/en-us/javascript/api/powerbi/powerbi-client/util
[^1_39]: https://millermedeiros.github.io/mdoc/examples/node_api/doc/util.html
[^1_40]: https://nodejs.org/download/test/v10.0.0-test20180410edd9cc466a/docs/api/util.html
[^1_41]: http://millermedeiros.github.io/mdoc/examples/node_api/doc/util.html
[^1_42]: https://plmlab.math.cnrs.fr/lomyr/slides/-/blob/main/revealjs/node_modules/@types/node/util.d.ts
[^1_43]: https://aclanthology.org/2025.naacl-long.176
[^1_44]: https://ieeexplore.ieee.org/document/10796606/
[^1_45]: http://ieeexplore.ieee.org/document/8115663/
[^1_46]: https://ieeexplore.ieee.org/document/10707487/
[^1_47]: https://dl.acm.org/doi/10.1145/3589334.3645342
[^1_48]: https://www.tandfonline.com/doi/full/10.1080/26939169.2024.2394541
[^1_49]: https://arxiv.org/abs/2205.09619
[^1_50]: http://link.springer.com/10.1007/s00267-015-0453-9
[^1_51]: https://link.springer.com/10.1007/s40471-022-00305-9
[^1_52]: https://dl.acm.org/doi/10.1145/3661167.3661207
[^1_53]: http://arxiv.org/pdf/1510.00925.pdf
[^1_54]: https://arxiv.org/pdf/2103.05769.pdf
[^1_55]: https://arxiv.org/pdf/2306.13984.pdf
[^1_56]: https://arxiv.org/pdf/1512.07067.pdf
[^1_57]: https://journals.asmarya.edu.ly/jau/index.php/jauas/article/download/1296/854
[^1_58]: http://arxiv.org/pdf/2401.08595.pdf
[^1_59]: https://pmc.ncbi.nlm.nih.gov/articles/PMC3686485/
[^1_60]: https://zenodo.org/record/3586025/files/article.pdf
[^1_61]: https://www.landskill.com/blog/7-javascript-pitfalls-to-avoid-enterprise-apps/
[^1_62]: https://stackify.com/node-js-error-handling/
[^1_63]: https://www.npmjs.com/package/util.promisify
[^1_64]: https://stackoverflow.com/questions/34329923/i-would-like-to-have-a-custom-inspect-function-for-use-with-util-inspect-in-node
[^1_65]: https://www.toptal.com/developers/nodejs/top-10-common-nodejs-developer-mistakes
[^1_66]: https://www.tutorialspoint.com/node-js-util-promisify-method
[^1_67]: https://www.youtube.com/watch?v=7LvpMTCid6w
[^1_68]: https://dev.to/devopsfundamentals/nodejs-fundamentals-util-2bko
[^1_69]: https://examplejavascript.com/util/promisify/
[^1_70]: https://kinsta.com/blog/node-debug/
[^1_71]: https://blog.appsignal.com/2022/11/23/nodejs-architecture-pitfalls-to-avoid.html
[^1_72]: https://stackoverflow.com/questions/56143158/how-to-use-util-promisify-and-bind-functions-in-nodejs
[^1_73]: https://www.educative.io/answers/what-is-utilinspect-in-nodejs
[^1_74]: https://hackernoon.com/is-your-code-slow-avoid-these-19-common-javascript-and-nodejs-mistakes
[^1_75]: https://masteringjs.io/tutorials/node/promisify
[^1_76]: https://nodejs.org/download/test/v6.14.4-test201811151cb5aed5f5/docs/api/util.html
[^1_77]: https://www.semanticscholar.org/paper/3a2482c2075c8b735d3c389ecd5a18f804db6a24
[^1_78]: https://academic.oup.com/gigascience/article/doi/10.1093/gigascience/giz109/5572530
[^1_79]: https://www.semanticscholar.org/paper/987d79f62ac1a44bd39753528f5fdc4604a7a9c2
[^1_80]: https://www.insight-journal.org/browse/publication/793
[^1_81]: https://pubs.acs.org/doi/10.1021/acs.jctc.5c01491
[^1_82]: https://carver.university/academiccrit/art-jose-fernando-davelouis/
[^1_83]: https://academic.oup.com/bioinformatics/article/36/22-23/5516/6039105
[^1_84]: https://ojs.focopublicacoes.com.br/foco/article/view/5169
[^1_85]: https://openresearchsoftware.metajnl.com/article/10.5334/jors.161/
[^1_86]: http://link.springer.com/10.1007/978-3-662-07964-5
[^1_87]: https://arxiv.org/pdf/2311.10533.pdf
[^1_88]: https://arxiv.org/pdf/2101.00756.pdf
[^1_89]: https://arxiv.org/html/2312.17149v3
[^1_90]: https://arxiv.org/pdf/2309.06551.pdf
[^1_91]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/spe.3296
[^1_92]: https://arxiv.org/pdf/2409.02074v1.pdf
[^1_93]: https://2ality.com/2022/08/node-util-parseargs.html
[^1_94]: https://bun.com/reference/node/util/MIMEType/type
[^1_95]: https://2coffee.dev/en/articles/a-few-useful-functions-in-nodejs-util-module
[^1_96]: https://lirantal.com/blog/mastering-cli-arguments-nodejs
[^1_97]: https://stackoverflow.com/questions/11971918/how-do-i-set-a-mime-type-before-sending-a-file-in-node-js
[^1_98]: https://exploringjs.com/nodejs-shell-scripting/ch_node-util-parseargs.html
[^1_99]: https://www.w3schools.com/nodejs/nodejs_real_world_examples.asp
[^1_100]: https://stackoverflow.com/questions/32515413/why-have-many-util-is-functions-been-deprecated-in-node-js-v4-0-0
[^1_101]: https://www.geeksforgeeks.org/node-js/how-to-parse-command-line-arguments-in-node-js/
[^1_102]: https://www.tutorialspoint.com/node-js-util-types-isdate-method
[^1_103]: https://stackoverflow.com/questions/28785254/parsing-node-command-line-arguments
[^1_104]: https://reqbin.com/req/nodejs/fvhorfob/mime-type-example
[^1_105]: https://www.geeksforgeeks.org/node-js/node-js-utility-complete-reference/
[^1_106]: https://docs.deno.com/api/node/util/~/parseArgs
[^1_107]: https://nodejs.org/api/all.html
[^1_108]: https://bun.com/reference/node/util/types
[^1_109]: https://bun.com/reference/node/util/parseArgs
[^1_110]: https://dev.to/muthuraja_r/common-built-in-apis-in-nodejs-3ep```

````

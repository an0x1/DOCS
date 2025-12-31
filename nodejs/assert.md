# دليل شامل لـ Node.js Assert Module: مرجع عميق للمطورين المحترفين

## نظرة عامة على الـ Module

**الـ `assert` module** هو **وحدة أساسية** في Node.js توفر مجموعة من دوال التحقق (Assertion Functions) المستخدمة للتحقق من صحة البيانات والعمليات الحسابية والمنطقية خلال تطوير واختبار التطبيقات. وهو يختلف عن أطر العمل الكاملة مثل Jest أو Mocha في كونه **خفيف الوزن وبدون تبعيات خارجية**، مما يجعله مثاليًا للاختبارات البسيطة والتحقق من الثوابت في الكود.[^1_1][^1_2][^1_3][^1_4]

![Node.js Assert Module: Complete Structure and Decision Flow](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/05776144f1818e490fb4be13ae2647dc/efc01c88-3d45-4b8b-8d0d-97e49f6259c4/9de5f822.png)

Node.js Assert Module: Complete Structure and Decision Flow

### متى تستخدم الـ Assert Module؟

يُستخدم الـ `assert` module في الحالات التالية:

| الحالة الاستخدام              | التفاصيل                                              |
| :---------------------------- | :---------------------------------------------------- |
| **الاختبارات البسيطة**        | للمشاريع الصغيرة التي لا تحتاج إلى مزايا متقدمة[^1_1] |
| **التحقق من الثوابت**         | التحقق من الافتراضات في الكود حتى في production[^1_5] |
| **عدم الاعتماد على التبعيات** | بما أنها وحدة أساسية، لا تحتاج npm install[^1_4]      |
| **التطوير السريع**            | الاختبار السريع للدوال والوحدات دون إعداد معقد[^1_6]  |

### الـ Assert vs أطر العمل الأخرى

| المميز                 | Node.js Assert                    | Jest                            | Mocha                     |
| :--------------------- | :-------------------------------- | :------------------------------ | :------------------------ |
| **التثبيت**            | مدمج (لا يحتاج تثبيت)             | npm install مطلوب               | npm install مطلوب         |
| **مزايا مدمجة**        | فقط الـ assertions                | assertions + mocking + coverage | يتطلب مكتبات إضافية       |
| **سهولة الإعداد**      | فوري                              | لا توجد مشاكل إعداد             | يحتاج إعدادات إضافية      |
| **الاستخدام النموذجي** | اختبارات بسيطة، التحقق من الثوابت | front-end و React               | backend و Node.js المعقد  |
| **الأداء**             | سريع جدًا                         | أبطأ نسبيًا (features كثيرة)    | يمكن أن يكون أسرع من Jest |

---

## 1. أنماط الـ Assertion: Strict vs Legacy

### Strict Assertion Mode (الموصى به)

في **الوضع الصارم** (Strict Mode)، جميع الدوال تستخدم المقارنة الصارمة `===` و `Object.is()`، مما يوفر **سلوكًا متوقعًا وآمنًا**:[^1_2][^1_3][^1_7]

```javascript
const assert = require("assert").strict;
// أو
import { strict as assert } from "node:assert";
// أو
import assert from "node:assert/strict";
```

**مثال تطبيقي - معالجة بيانات المستخدم:**

```javascript
const assert = require("assert").strict;

// داخل دالة تتحقق من بيانات المستخدم
function validateUserProfile(user) {
  // التحقق من الهوية (Identity) وليس القيمة فقط
  assert.strictEqual(user.age, 25, "عمر المستخدم يجب أن يكون 25 بالضبط");

  // لن تمر هذه الاختبارات
  assert.strictEqual(25, "25"); // AssertionError: لا تمر (number !== string)
}
```

### Legacy Assertion Mode (مستهجن - لا تستخدمه!)

في **الوضع القديم**، تستخدم المقارنة المرنة `==`، مما قد يسبب **نتائج غير متوقعة**:[^1_7][^1_8][^1_2]

```javascript
const assert = require("assert"); // الوضع القديم

// تحذير: هذا لن يرمي خطأ!
assert.deepEqual("+00000000", false); // لا يرمي خطأ (غير متوقع!)
assert.deepEqual(/a/gi, new Date()); // لا يرمي خطأ (خطير!)
```

**التوصية الحاسمة:** استخدم **Strict Mode فقط**. الوضع القديم قد يُزال في نسخة مستقبلية.[^1_3][^1_9][^1_2][^1_7]

---

## 2. الدوال الأساسية للمقارنة الإجمالية (Equality)

### assert.ok() / assert()

**التوقيع:**

```javascript
assert.ok(value[, message])
assert(value[, message])  // الاسم المختصر
```

**الوصف:** التحقق من أن القيمة **تُقيّم إلى true** (truthy). تُرمي `AssertionError` إذا كانت القيمة falsy.[^1_10][^1_2][^1_3]

| الباراميتر | النوع           | مطلوب/اختياري | الوصف                     |
| :--------- | :-------------- | :------------ | :------------------------ |
| `value`    | Any             | مطلوب         | القيمة المراد التحقق منها |
| `message`  | string \| Error | اختياري       | رسالة الخطأ المخصصة       |

**القيم الـ Falsy:** `false`, `0`, `''`, `null`, `undefined`, `NaN`.[^1_2]

**مثال تطبيقي - التحقق من الاتصال بقاعدة البيانات:**

```javascript
const assert = require("assert").strict;

async function ensureDatabaseConnection(db) {
  // التحقق من أن الاتصال نشط
  assert(db.isConnected, "قاعدة البيانات يجب أن تكون متصلة");

  // التحقق من أن الـ pool موجود
  assert(db.connectionPool, "يجب أن يكون هناك connection pool");
}

// الاستخدام
ensureDatabaseConnection(activeDb); // تمر بسهولة
ensureDatabaseConnection(closedDb); // AssertionError: قاعدة البيانات يجب أن تكون متصلة
```

**الأخطاء الشائعة:**

1. **عدم توفير رسالة مفيدة** - الرسالة الغامضة تصعّب التصحيح:

```javascript
assert(result); // خطأ غامض
assert(result, "نتيجة معالجة الصورة يجب أن تكون موجودة"); // واضح ومفيد
```

2. **نسيان التحويل إلى Boolean في الشروط المعقدة:**

```javascript
// خطأ: يتحقق من 5 وليس من الشرط
assert(array.length > 3); // يرمي خطأ لأن 5 = true لكن رسالة غير واضحة

// الأفضل: واضح ومباشر
assert(
  array.length > 3,
  `طول الـ array (${array.length}) يجب أن يكون أكثر من 3`
);
```

---

### assert.strictEqual() / assert.equal()

**التوقيع:**

```javascript
assert.strictEqual(actual, expected[, message])  // Strict Mode
assert.equal(actual, expected[, message])         // Legacy Mode
```

**الوصف:** المقارنة **السطحية** بين قيمتين. في Strict Mode تستخدم `===`.[^1_3][^1_2]

| الباراميتر | النوع           | مطلوب/اختياري | الوصف           |
| :--------- | :-------------- | :------------ | :-------------- |
| `actual`   | Any             | مطلوب         | القيمة الفعلية  |
| `expected` | Any             | مطلوب         | القيمة المتوقعة |
| `message`  | string \| Error | اختياري       | رسالة الخطأ     |

**مثال تطبيقي - التحقق من حالة API Response:**

```javascript
const assert = require("assert").strict;

function validateHttpStatusCode(response) {
  // التحقق من أن الـ status code بالضبط 200 (number وليس string!)
  assert.strictEqual(response.status, 200, "رمز الحالة يجب أن يكون 200");

  // التحقق من نوع البيانات المرجعة
  assert.strictEqual(
    typeof response.data,
    "object",
    "البيانات يجب أن تكون object"
  );
}

// سيفشل لأن '200' !== 200
validateHttpStatusCode({ status: "200", data: {} });
// AssertionError: رمز الحالة يجب أن يكون 200
```

**الفرق بين `strictEqual` و `equal`:**

```javascript
const assert = require("assert").strict;

assert.strictEqual(1, "1"); // ❌ AssertionError (1 !== '1')
assert.strictEqual(NaN, NaN); // ✅ OK (معالجة خاصة ل NaN)
assert.strictEqual(1, 1); // ✅ OK
```

---

### assert.notStrictEqual() / assert.notEqual()

**التوقيع:**

```javascript
assert.notStrictEqual(actual, expected[, message])
assert.notEqual(actual, expected[, message])
```

**الوصف:** العكس من `strictEqual` - التحقق من **عدم المساواة**.[^1_2][^1_3]

**مثال تطبيقي - التحقق من أن البيانات تم تعديلها:**

```javascript
const assert = require("assert").strict;

function assertDataChanged(originalHash, newHash) {
  // التأكد من أن الـ hash تغير (البيانات تم تعديلها)
  assert.notStrictEqual(
    originalHash,
    newHash,
    "البيانات يجب أن تكون قد تغيرت (hashes مختلفة)"
  );
}

const originalData = { name: "أحمد", age: 25 };
const modifiedData = { name: "أحمد", age: 26 };

const hash1 = JSON.stringify(originalData);
const hash2 = JSON.stringify(modifiedData);

assertDataChanged(hash1, hash2); // ✅ تمر
assertDataChanged(hash1, hash1); // ❌ AssertionError
```

---

## 3. المقارنة العميقة (Deep Equality)

### assert.deepStrictEqual()

**التوقيع:**

```javascript
assert.deepStrictEqual(actual, expected[, message])
```

**الوصف:** المقارنة **العميقة** المتكررة لجميع خصائص الكائنات والمصفوفات، مع استخدام `===` للمقارنات البدائية.[^1_3][^1_2]

| الباراميتر | النوع           | مطلوب/اختياري | الوصف          |
| :--------- | :-------------- | :------------ | :------------- |
| `actual`   | Object \| Array | مطلوب         | الكائن الفعلي  |
| `expected` | Object \| Array | مطلوب         | الكائن المتوقع |
| `message`  | string \| Error | اختياري       | رسالة الخطأ    |

**قواعد المقارنة العميقة في Strict Mode:**

- **المقارنة البدائية:** تستخدم `Object.is()` (معالجة خاصة ل `NaN`)[^1_2]
- **الخصائص الخاصة:** `Error` messages، `RegExp` flags، `Date` timestamps دائمًا تُقارن[^1_2]
- **الـ Prototypes:** **لا** تُقارن (فقط الخصائص الخاصة)[^1_2]
- **الـ Symbol:** الرموز الخاصة **تُقارن** بهويتها[^1_2]
- **Collections:** `Map` و `Set` تُقارنان مع تجاهل الترتيب[^1_2]

**مثال تطبيقي - التحقق من بنية استجابة JSON:**

```javascript
const assert = require("assert").strict;

function validateApiResponse(response) {
  const expectedResponse = {
    status: 200,
    data: {
      users: [
        { id: 1, name: "أحمد", email: "ahmed@example.com" },
        { id: 2, name: "فاطمة", email: "fatima@example.com" },
      ],
      total: 2,
    },
    timestamp: new Date("2025-01-01"),
  };

  // مقارنة عميقة: جميع الخصائص المتداخلة
  assert.deepStrictEqual(
    response,
    expectedResponse,
    "البيانات المرجعة لا تطابق البنية المتوقعة"
  );
}

// سيفشل: الـ timestamp مختلف
const actualResponse = {
  status: 200,
  data: {
    users: [
      { id: 1, name: "أحمد", email: "ahmed@example.com" },
      { id: 2, name: "فاطمة", email: "fatima@example.com" },
    ],
    total: 2,
  },
  timestamp: new Date("2025-01-02"), // مختلف!
};

validateApiResponse(actualResponse); // AssertionError
```

**مثال متقدم - مقارنة Objects مع Symbols:**

```javascript
const assert = require("assert").strict;

const sym1 = Symbol("unique");
const sym2 = Symbol("unique");

const obj1 = { [sym1]: "value", name: "test" };
const obj2 = { [sym1]: "value", name: "test" };
const obj3 = { [sym2]: "value", name: "test" };

assert.deepStrictEqual(obj1, obj2); // ✅ OK (نفس الـ symbol)
assert.deepStrictEqual(obj1, obj3); // ❌ AssertionError (symbols مختلفة)
```

---

### assert.deepEqual() (مستهجن)

**التوقيع:**

```javascript
assert.deepEqual(actual, expected[, message])
```

**التحذير:** في **Strict Mode** تتصرف مثل `deepStrictEqual`. في **Legacy Mode** تستخدم `==`. **لا تستخدمها!**[^1_7][^1_2]

---

### assert.partialDeepStrictEqual()

**التوقيع:**

```javascript
assert.partialDeepStrictEqual(actual, expected[, message])
```

**الوصف:** مقارنة عميقة حيث **يجب أن تحتوي actual على جميع خصائص expected**، لكن actual قد يحتوي على خصائص إضافية:[^1_11]

**مثال تطبيقي - التحقق من استجابة API بخصائص إضافية:**

```javascript
const assert = require("assert").strict;

function validateMinimalUserData(response) {
  const expectedMinimum = {
    id: 1,
    name: "أحمد",
    email: "ahmed@example.com",
  };

  // لا نهتم بالخصائص الإضافية مثل metadata, timestamp, etc
  assert.partialDeepStrictEqual(
    response,
    expectedMinimum,
    "يجب أن تحتوي الاستجابة على البيانات الأساسية للمستخدم"
  );
}

// سيمر رغم وجود خصائص إضافية
const fullResponse = {
  id: 1,
  name: "أحمد",
  email: "ahmed@example.com",
  createdAt: "2025-01-01",
  updatedAt: "2025-01-15",
  metadata: { source: "api", version: 2 },
  roles: ["user", "admin"],
};

validateMinimalUserData(fullResponse); // ✅ تمر
```

**الفرق بين `deepStrictEqual` و `partialDeepStrictEqual`:**

```javascript
const assert = require("assert").strict;

const expected = { a: 1, b: 2 };
const actual = { a: 1, b: 2, c: 3 };

assert.deepStrictEqual(actual, expected); // ❌ AssertionError (actual أكبر)
assert.partialDeepStrictEqual(actual, expected); // ✅ OK
```

---

## 4. معالجة الأخطاء والاستثناءات

### assert.throws()

**التوقيع:**

```javascript
assert.throws(fn[, error][, message])
```

**الوصف:** التحقق من أن الدالة **تطرح (throw) خطأ**. الـ `error` يمكن أن يكون Class أو RegExp أو validation function.[^1_3][^1_2]

| الباراميتر | النوع                       | مطلوب/اختياري | الوصف                  |
| :--------- | :-------------------------- | :------------ | :--------------------- |
| `fn`       | Function                    | مطلوب         | الدالة المراد اختبارها |
| `error`    | Class \| RegExp \| Function | اختياري       | نوع/نمط الخطأ المتوقع  |
| `message`  | string                      | اختياري       | رسالة الفشل            |

**مثال تطبيقي - التحقق من معالجة الأخطاء في معالج الملفات:**

```javascript
const assert = require("assert").strict;
const fs = require("fs");

function validateFileHandling() {
  // اختبار 1: رمي خطأ من نوع TypeError
  assert.throws(
    () => JSON.parse(undefined),
    SyntaxError,
    "JSON.parse يجب أن يرمي SyntaxError للقيم غير الصحيحة"
  );

  // اختبار 2: التحقق من رسالة الخطأ باستخدام RegExp
  assert.throws(
    () => {
      if (!fs.existsSync("/invalid/path")) {
        throw new Error("الملف غير موجود: /invalid/path");
      }
    },
    /الملف غير موجود/,
    'رسالة الخطأ يجب أن تحتوي على "الملف غير موجود"'
  );

  // اختبار 3: validation function مخصص
  assert.throws(
    () => {
      throw new Error("عملية فشلت لسبب معين");
    },
    (err) => {
      return err instanceof Error && err.message.includes("عملية فشلت");
    },
    "الخطأ يجب أن يكون Error مع رسالة محددة"
  );
}

validateFileHandling(); // ✅ تمر جميع الاختبارات
```

**الأخطاء الشائعة:**

```javascript
const assert = require("assert").strict;

// ❌ خطأ: لا تُمرر الدالة كـ string!
assert.throws('JSON.parse("invalid")');

// ✅ صحيح: مرر function
assert.throws(() => JSON.parse("invalid"));

// ❌ خطأ: لا تستدعِ الدالة مباشرة
assert.throws(JSON.parse("invalid")); // سيرمي خطأ قبل assert

// ✅ صحيح: مررها كـ arrow function
assert.throws(() => JSON.parse("invalid"));
```

---

### assert.rejects()

**التوقيع:**

```javascript
assert.rejects(asyncFn[, error][, message])
```

**الوصف:** نسخة async من `throws` - تتحقق من أن Promise **يُرفض (reject)**.[^1_12][^1_3][^1_2]

**مثال تطبيقي - اختبار عمليات async مع معالجة الأخطاء:**

```javascript
const assert = require("assert").strict;

async function validateAsyncOperations() {
  // اختبار 1: التحقق من رفع Promise لخطأ معين
  await assert.rejects(
    async () => {
      const response = await fetch("https://invalid-api.example.com/data");
      if (!response.ok) throw new Error("HTTP Error: " + response.status);
      return response.json();
    },
    Error,
    "العملية يجب أن تفشل مع Error"
  );

  // اختبار 2: التحقق من رسالة الخطأ باستخدام validation function
  await assert.rejects(
    async () => {
      // محاكاة عملية قاعدة بيانات تفشل
      throw new TypeError("البيانات المُدخلة غير صحيحة");
    },
    (err) => {
      return err instanceof TypeError && err.message.includes("البيانات");
    },
    "الخطأ يجب أن يكون TypeError مع رسالة محددة"
  );

  // اختبار 3: التحقق من Promise مباشرة
  const rejectedPromise = Promise.reject(new Error("اتصال قاعدة البيانات فشل"));

  await assert.rejects(rejectedPromise, Error);
}

validateAsyncOperations(); // تمر جميع الاختبارات
```

**الفرق بين `throws` و `rejects`:**

```javascript
const assert = require("assert").strict;

// throws - للدوال المتزامنة
assert.throws(() => {
  throw new Error("خطأ مزامن");
});

// rejects - للدوال غير المتزامنة والـ Promises
await assert.rejects(async () => {
  throw new Error("خطأ غير متزامن");
});

// أو مع Promise مباشرة
await assert.rejects(Promise.reject(new Error("فشل")));
```

---

### assert.doesNotThrow()

**التوقيع:**

```javascript
assert.doesNotThrow(fn[, error][, message])
```

**الوصف:** التحقق من أن الدالة **لا تطرز خطأ**.[^1_3][^1_2]

**مثال تطبيقي - التحقق من أن العمليات آمنة:**

```javascript
const assert = require("assert").strict;

function validateSafeOperations() {
  // التحقق من أن الحساب آمن
  assert.doesNotThrow(() => {
    const result = 10 / 2;
    if (result === 0) throw new Error("قسمة على صفر");
    return result;
  }, "العملية يجب أن تكتمل بدون أخطاء");

  // التحقق من أن معالجة الـ JSON آمنة
  assert.doesNotThrow(() => {
    const validJson = '{"name":"test","value":123}';
    JSON.parse(validJson);
  });
}

validateSafeOperations(); // ✅ تمر
```

---

### assert.doesNotReject()

**التوقيع:**

```javascript
assert.doesNotReject(asyncFn[, error][, message])
```

**الوصف:** نسخة async من `doesNotThrow`.[^1_3][^1_2]

**مثال تطبيقي:**

```javascript
const assert = require("assert").strict;

async function validateAsyncSafety() {
  // التحقق من أن العملية الغير المتزامنة آمنة
  await assert.doesNotReject(async () => {
    const data = await fetchValidData();
    return processData(data);
  }, "العملية يجب أن تكتمل بدون رفع Promise");
}
```

---

### assert.ifError()

**التوقيع:**

```javascript
assert.ifError(value);
```

**الوصف:** إذا كانت القيمة ليست `null` أو `undefined`، **يتم رفعها كخطأ**. مفيدة في callbacks التقليدية.[^1_3][^1_2]

**مثال تطبيقي - معالجة Callbacks التقليدية:**

```javascript
const assert = require("assert").strict;
const fs = require("fs");

function readFileWithAssertion(filePath) {
  return new Promise((resolve, reject) => {
    fs.readFile(filePath, "utf8", (err, data) => {
      // إذا كان هناك خطأ، ارفعه فوراً
      assert.ifError(err);

      // في هذه النقطة، نحن متأكدون أن `err` هو null
      resolve(data);
    });
  });
}

// الاستخدام
readFileWithAssertion("./data.json")
  .then((data) => console.log("تمت قراءة البيانات:", data))
  .catch((err) => console.error("خطأ:", err.message));
```

---

## 5. مطابقة الأنماط (Regular Expressions)

### assert.match()

**التوقيع:**

```javascript
assert.match(string, regexp[, message])
```

**الوصف:** التحقق من أن النص **يطابق** النمط (regex).[^1_3][^1_2]

| الباراميتر | النوع  | مطلوب/اختياري | الوصف               |
| :--------- | :----- | :------------ | :------------------ |
| `string`   | string | مطلوب         | النص المراد اختباره |
| `regexp`   | RegExp | مطلوب         | النمط المتوقع       |
| `message`  | string | اختياري       | رسالة الخطأ         |

**مثال تطبيقي - التحقق من صيغة البريد الإلكتروني والهاتف:**

```javascript
const assert = require("assert").strict;

function validateContactInformation(email, phone) {
  // التحقق من صيغة البريد الإلكتروني
  assert.match(
    email,
    /^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/,
    "البريد الإلكتروني يجب أن يكون بصيغة صحيحة"
  );

  // التحقق من صيغة رقم الهاتف (مثال: +966-50-1234567)
  assert.match(
    phone,
    /^\+\d{1,3}-\d{2,3}-\d{6,9}$/,
    "رقم الهاتف يجب أن يكون بصيغة: +XXX-XX-XXXXXX"
  );
}

validateContactInformation("user@example.com", "+966-50-1234567"); // ✅ تمر
validateContactInformation("invalid-email", "+966501234567"); // ❌ AssertionError
```

---

### assert.doesNotMatch()

**التوقيع:**

```javascript
assert.doesNotMatch(string, regexp[, message])
```

**الوصف:** العكس من `match` - التحقق من أن النص **لا يطابق** النمط.[^1_2][^1_3]

**مثال تطبيقي - التحقق من السلامة:**

```javascript
const assert = require("assert").strict;

function validateSafeInput(userInput) {
  // التحقق من عدم احتواء الـ input على أكواد ضارة
  assert.doesNotMatch(
    userInput,
    /<script|javascript:|onclick|on\w+=/gi,
    "الـ input يحتوي على أكواد ضارة محتملة"
  );

  // التحقق من عدم احتواء كلمات محظورة
  assert.doesNotMatch(
    userInput,
    /badword|profanity|forbidden/i,
    "الـ input يحتوي على كلمات محظورة"
  );
}

validateSafeInput("السلام عليكم ورحمة الله"); // ✅ تمر
validateSafeInput('<script>alert("xss")</script>'); // ❌ AssertionError
```

---

## 6. الفشل الصريح والتحكم

### assert.fail()

**التوقيع:**

```javascript
assert.fail([message])
assert.fail(actual, expected[, message[, operator[, stackStartFn]]])
```

**الوصف:** رفع `AssertionError` بقوة بدون شروط.[^1_3][^1_2]

**مثال تطبيقي - التحكم في تدفق الاختبار:**

```javascript
const assert = require("assert").strict;

function processUserInput(input) {
  if (!input) {
    assert.fail("يجب تقديم البيانات المدخلة");
  }

  if (input.type === "admin") {
    // معالجة خاصة
    console.log("معالجة مسؤول نظام");
  } else if (input.type === "user") {
    console.log("معالجة مستخدم عادي");
  } else {
    assert.fail(
      input.type,
      ["admin", "user"],
      "نوع المستخدم غير معروف",
      "∈" // Operator
    );
  }
}

processUserInput({ type: "user" }); // ✅
processUserInput({ type: "unknown" }); // ❌ AssertionError
```

---

## 7. Assertion Error والمعالجة

### class assert.AssertionError

**الوصف:** جميع الأخطاء التي تطرحها وحدة assert هي instances من `AssertionError`.[^1_2][^1_3]

**الخصائص:**

| الخاصية            | النوع   | الوصف                                   |
| :----------------- | :------ | :-------------------------------------- |
| `message`          | string  | رسالة الخطأ                             |
| `name`             | string  | دائمًا "AssertionError"                 |
| `code`             | string  | دائمًا "ERR_ASSERTION"                  |
| `actual`           | any     | القيمة الفعلية                          |
| `expected`         | any     | القيمة المتوقعة                         |
| `operator`         | string  | نوع المقارنة (e.g., '===', 'deepEqual') |
| `generatedMessage` | boolean | هل تم توليد الرسالة تلقائيًا            |

**مثال تطبيقي - التعامل مع الأخطاء:**

```javascript
const assert = require("assert").strict;

function analyzeAssertionError() {
  try {
    assert.strictEqual(5, 10, "الأرقام يجب أن تتطابق");
  } catch (err) {
    // فحص الخطأ
    console.log("نوع الخطأ:", err.name); // AssertionError
    console.log("الكود:", err.code); // ERR_ASSERTION
    console.log("الفعلي:", err.actual); // 5
    console.log("المتوقع:", err.expected); // 10
    console.log("العامل:", err.operator); // ===
    console.log("الرسالة:", err.message); // الأرقام يجب أن تتطابق
    console.log("رسالة مولدة؟:", err.generatedMessage); // false

    // معالجة الخطأ
    if (err instanceof assert.AssertionError) {
      console.log("اكتشفنا assertion error - هذا متوقع");
    }
  }
}

analyzeAssertionError();
```

**إنشاء `AssertionError` مخصص:**

```javascript
const assert = require("assert").strict;

const customError = new assert.AssertionError({
  actual: 100,
  expected: 50,
  operator: ">",
  message: "القيمة يجب أن تكون أقل من 50",
  stackStartFn: processData,
});

function processData() {
  throw customError;
}

try {
  processData();
} catch (err) {
  console.log("رسالة مخصصة:", err.message);
}
```

---

## 8. Call Tracker (مستهجن - لا تستخدمه!)

### class assert.CallTracker (Deprecated)

**الحالة:** **مستهجنة وسيتم حذفها**. استخدم [`node:test` mock helper](https://nodejs.org/api/test.html#mocking) بدلاً منها.[^1_13][^1_14][^1_2]

```javascript
// ❌ لا تستخدم CallTracker
const tracker = new assert.CallTracker();

// ✅ استخدم mock helper بدلاً منها
import { mock } from "node:test";
const myFunc = mock.fn();
```

---

## 9. أفضل الممارسات والحالات الاستخدام

### ✅ أفضل الممارسات

1. **استخدم Strict Mode فقط:**

```javascript
const assert = require("assert").strict; // ✅ الطريقة الصحيحة
```

2. **وفّر رسائل خطأ واضحة ومفيدة:**

```javascript
// ❌ غامضة
assert.strictEqual(result, expected);

// ✅ واضحة
assert.strictEqual(
  result,
  expected,
  `نتيجة معالجة الصورة (${result}px) يجب أن تساوي (${expected}px)`
);
```

3. **استخدم `deepStrictEqual` للكائنات:**

```javascript
const expected = { name: "test", value: 123 };
assert.deepStrictEqual(actual, expected); // ✅ يقارن جميع الخصائص
```

4. **استخدم `partialDeepStrictEqual` عند تجاهل الخصائص الإضافية:**

```javascript
// عند اختبار API response مع metadata إضافي
assert.partialDeepStrictEqual(response, minimalExpected);
```

5. **استخدم `throws` و `rejects` للتحقق من معالجة الأخطاء:**

```javascript
assert.throws(() => invalidFunction(), TypeError);
await assert.rejects(invalidAsync(), TypeError);
```

### ❌ ما يجب تجنبه

1. **استخدام Legacy Mode:**

```javascript
const assert = require("assert"); // ❌ تجنب!
```

2. **استخدام CallTracker:**

```javascript
new assert.CallTracker(); // ❌ مستهجنة - استخدم mock
```

3. **عدم توفير رسائل خطأ:**

```javascript
assert(condition); // ❌ رسالة افتراضية غير واضحة
```

4. **مقارنة Objects بـ `strictEqual`:**

```javascript
assert.strictEqual({ a: 1 }, { a: 1 }); // ❌ سيفشل (مراجع مختلفة)
assert.deepStrictEqual({ a: 1 }, { a: 1 }); // ✅ صحيح
```

---

## 10. حالات الاستخدام العملية الأعلى

### حالة 1: التحقق من الثوابت في Middleware

```javascript
const assert = require("assert").strict;

function authenticationMiddleware(req, res, next) {
  // التحقق من وجود headers الإلزامية
  assert(req.headers.authorization, "Authorization header مطلوب");

  // التحقق من صيغة Token
  assert.match(
    req.headers.authorization,
    /^Bearer\s+.+$/,
    'Authorization يجب أن يكون "Bearer <token>"'
  );

  next();
}
```

### حالة 2: اختبار معالج البيانات

```javascript
const assert = require("assert").strict;

async function processUserRegistration(userData) {
  // التحقق من البيانات المدخلة
  assert.strictEqual(typeof userData, "object", "userData يجب أن يكون object");
  assert.strictEqual(
    typeof userData.email,
    "string",
    "email يجب أن يكون string"
  );
  assert.match(
    userData.email,
    /^[^@]+@[^@]+\.[^@]+$/,
    "email يجب أن يكون بصيغة صحيحة"
  );

  // معالجة البيانات
  const user = await saveUser(userData);

  // التحقق من النتائج
  assert(user.id, "المستخدم الجديد يجب أن يحصل على ID");
  assert.strictEqual(
    user.email,
    userData.email,
    "البريد الإلكتروني يجب أن يبقى كما هو"
  );

  return user;
}
```

### حالة 3: اختبار API Client

```javascript
const assert = require("assert").strict;

async function testApiClient() {
  const client = new ApiClient("https://api.example.com");

  // اختبار الاتصال الناجح
  const response = await client.get("/users/1");
  assert.strictEqual(response.status, 200);
  assert(response.data.id);
  assert.match(response.data.email, /.+@.+\..+/);

  // اختبار معالجة الأخطاء
  await assert.rejects(
    () => client.get("/invalid/endpoint"),
    (err) => err.status === 404
  );

  // اختبار البيانات الكاملة
  assert.deepStrictEqual(response.data, {
    id: 1,
    name: "أحمد",
    email: "ahmed@example.com",
    roles: ["user"],
  });
}
```

---

## الخلاصة والتوصيات

**الـ `assert` module** هو أداة قوية وخفيفة الوزن لـ:

- ✅ الاختبارات البسيطة والسريعة
- ✅ التحقق من الثوابت في Production
- ✅ المشاريع بدون تبعيات إضافية
- ✅ Debugging والتطوير السريع

**التوصيات الحاسمة:**

1. استخدم **Strict Mode دائمًا** `require('assert').strict`
2. وفّر **رسائل خطأ واضحة** تساعد في التصحيح
3. استخدم `deepStrictEqual` للكائنات والمصفوفات
4. استخدم `throws`/`rejects` لاختبار الأخطاء
5. **تجنب** Legacy Mode و CallTracker (مستهجنة)

---

## المراجع

وثائق Node.js الرسمية وأفضل الممارسات[^1_15][^1_6][^1_4][^1_5][^1_10][^1_7][^1_3][^1_2]
<span style="display:none">[^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88][^1_89][^1_90][^1_91][^1_92][^1_93][^1_94][^1_95]</span>

<div align="center">⁂</div>

[^1_1]: https://www.w3schools.com/nodejs/nodejs_assert.asp
[^1_2]: https://nodejs.org/docs/latest/api/assert.html
[^1_3]: https://nodejs.org/api/assert.html
[^1_4]: https://www.geeksforgeeks.org/node-js/node-js-assert-module/
[^1_5]: https://stackoverflow.com/questions/33083168/node-js-should-i-keep-asserts-in-production-code
[^1_6]: https://blog.appsignal.com/2024/08/14/an-introduction-to-unit-testing-in-nodejs.html
[^1_7]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/assert/
[^1_8]: https://gitlab.tms.id/root/frontend-ebp/-/tree/ad847fde5654ebf6ab559fbe434147e2d2ebadc0/node_modules/assert
[^1_9]: https://github.com/mysticatea/eslint-plugin-node/issues/218
[^1_10]: https://www.w3schools.com/nodejs/met_assert.asp
[^1_11]: https://bun.com/reference/node/assert/default/partialDeepStrictEqual
[^1_12]: https://www.geeksforgeeks.org/node-js/node-js-assert-rejects-function/
[^1_13]: https://docs.deno.com/api/node/all_symbols
[^1_14]: https://docs.deno.com/api/node/assert/~/assert.CallTracker
[^1_15]: https://www.linkedin.com/pulse/nodejs-guide-1-mastering-assertion-testing-lahiru-sandaruwan-nzdnc
[^1_16]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_17]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_18]: https://www.semanticscholar.org/paper/cd57de2e8838433c5b705d62d1c1194b83905ae1
[^1_19]: https://dl.acm.org/doi/10.1145/3649476.3660378
[^1_20]: https://journals.ysu.am/index.php/arm-issues/article/view/13924
[^1_21]: https://vestnik.dgtu.ru/jour/article/view/1620
[^1_22]: http://tst.stu.cn.ua/article/view/337129
[^1_23]: https://jurnal.kdi.or.id/index.php/bt/article/view/3274
[^1_24]: http://transactions.ismir.net/articles/10.5334/tismir.111/
[^1_25]: https://www.semanticscholar.org/paper/959d24d21efe31eae401be73c1f7d8e636805655
[^1_26]: http://arxiv.org/pdf/2411.16927.pdf
[^1_27]: https://zenodo.org/record/5055072/files/TR-Precrime-2020-02.pdf
[^1_28]: http://arxiv.org/pdf/2502.02708.pdf
[^1_29]: http://arxiv.org/pdf/2402.00386.pdf
[^1_30]: https://arxiv.org/pdf/2503.04057.pdf
[^1_31]: https://arxiv.org/pdf/2309.10264.pdf
[^1_32]: https://arxiv.org/pdf/2011.09784.pdf
[^1_33]: https://arxiv.org/pdf/2305.14808.pdf
[^1_34]: https://qunitjs.com/api/assert/deepEqual/
[^1_35]: https://github.com/nodejs/node/blob/main/doc/api/assert.md
[^1_36]: https://blog.logrocket.com/using-assert-modules-to-verify-invariants-in-nodejs/
[^1_37]: https://www.chaijs.com/api/assert/
[^1_38]: https://stackoverflow.com/questions/13225274/the-difference-between-assert-equal-and-assert-deepequal-in-javascript-testing-w
[^1_39]: https://www.geeksforgeeks.org/node-js/node-js-assert-function/
[^1_40]: https://techblog.topdesk.com/coding/frontend-testing-with-jest-assertions-deep-dive/
[^1_41]: https://www.w3schools.com/nodejs/met_assert_deepequal.asp
[^1_42]: https://nodejs.org/download/release/v4.5.0/docs/api/assert.html
[^1_43]: https://www.thisdot.co/blog/deep-dive-into-node-js-with-james-snell
[^1_44]: https://iojs.org/api/assert.html
[^1_45]: https://www.npmjs.com/package/assert?activeTab=versions
[^1_46]: https://www.ibm.com/docs/en/datapower-gateway/10.5.x?topic=apis-assert-module
[^1_47]: https://infosecwriteups.com/how-assertions-can-get-you-hacked-da22c84fb8f6
[^1_48]: http://library.iated.org/view/LEAHY2017FOU
[^1_49]: https://utafitionline.com/index.php/jsic/article/view/1494
[^1_50]: https://www.semanticscholar.org/paper/33a730ee455fc34c556ed64639c608c8faa25257
[^1_51]: https://userarea.eupvsec.org/proceedings/25th-EU-PVSEC-WCPEC-5/4AV.3.46/
[^1_52]: https://www.semanticscholar.org/paper/8a23be61506b8c64361cd90e6a024ff628671e79
[^1_53]: https://iopscience.iop.org/article/10.1088/1748-0221/16/08/C08002
[^1_54]: https://vestnik-ngo.kz/2707-4226/article/view/108702
[^1_55]: https://link.springer.com/10.1007/s11948-022-00374-5
[^1_56]: http://peer.asee.org/34484
[^1_57]: https://dl.acm.org/doi/10.1145/3742430
[^1_58]: https://arxiv.org/pdf/2408.01751.pdf
[^1_59]: http://www.cs.toronto.edu/~chechik/courses06/csc410/rosenblum_assert95.pdf
[^1_60]: http://arxiv.org/pdf/2407.08138.pdf
[^1_61]: https://arxiv.org/pdf/2003.01668.pdf
[^1_62]: https://arxiv.org/pdf/2101.00756.pdf
[^1_63]: https://www.browserstack.com/guide/jest-vs-mocha
[^1_64]: https://www.geeksforgeeks.org/software-testing/jest-vs-mocha-which-one-should-you-choose/
[^1_65]: https://bun.com/reference/node/assert
[^1_66]: https://saucelabs.com/resources/blog/jest-vs-mocha
[^1_67]: https://www.linkedin.com/posts/touseef-iqbal-836400209_introduction-to-assertion-testing-assertion-activity-7331547782014484481-3TIP
[^1_68]: https://www.designveloper.com/blog/mocha-vs-jest-vs-jasmine/
[^1_69]: https://testgrid.io/blog/jest-vs-mocha/
[^1_70]: https://playwright.dev/docs/test-assertions
[^1_71]: https://www.reddit.com/r/node/comments/fmywgt/when_and_correct_approach_to_use_the_assert_module/
[^1_72]: https://www.reddit.com/r/node/comments/q55mh2/jest_or_mocha_for_testing_nodejs_application/
[^1_73]: https://gitlab.cs.nuim.ie/u240298/tigerprivacy/-/blob/d0d763c32ae5571aca7fe2d792bf1f68661f4d5d/backend/node_modules/@types/node/assert.d.ts
[^1_74]: https://github.com/goldbergyoni/nodejs-testing-best-practices
[^1_75]: https://testrigor.com/blog/modern-javascript-testing-frameworks/
[^1_76]: http://arxiv.org/pdf/1510.00925.pdf
[^1_77]: http://arxiv.org/pdf/2501.18145.pdf
[^1_78]: https://zenodo.org/record/5500461/files/NodeXP__NOde_js_server_side_JavaScript_injection_vulnerability_DEtection_and_eXPloitation%20(1).pdf
[^1_79]: http://downloads.hindawi.com/journals/tswj/2014/316014.pdf
[^1_80]: http://arxiv.org/pdf/2203.14659.pdf
[^1_81]: http://arxiv.org/pdf/2208.09873.pdf
[^1_82]: https://arxiv.org/pdf/2302.06527.pdf
[^1_83]: https://www.dataraccoon.com/knowledge/testing_mock
[^1_84]: https://vitest.dev/api/assert
[^1_85]: https://stackoverflow.com/questions/31766603/how-to-assert-partially-deep-equal-compare-objects-in-javascript
[^1_86]: https://github.com/nodejs/node/issues/47492
[^1_87]: https://www.chaijs.com/api/bdd/
[^1_88]: https://stackoverflow.com/questions/35782435/node-js-assert-throws-with-async-functions-promises/35782749
[^1_89]: https://stackoverflow.com/questions/79477533/alternative-to-assert-in-node-js-with-strict-mode
[^1_90]: https://www.w3schools.com/nodejs/met_assert_deepstrictequal.asp
[^1_91]: https://stackoverflow.com/questions/54247064/mocha-node-how-to-use-assert-rejects
[^1_92]: https://gitlab.emse.fr/fatima.ghaoui/laundary-application-frontend/-/blob/main/node_modules/@types/node/assert.d.ts
[^1_93]: https://github.com/nodejs/node/actions/runs/12106232358
[^1_94]: https://nodejs.org/download/release/v13.1.0/docs/api/assert.html
[^1_95]: https://csgit.ucalgary.ca/keshav.bhardwaj1/cpsc-471-project/-/blob/cd26483568e917877169a7c13846dbafe7f75037/node_modules/@types/node/assert.d.ts

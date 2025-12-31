# موثّق شامل لوحدة Node.js URL - دليل متعمّق

يقدم هذا الدليل شرحاً مفصلاً لجميع الدوال والخصائص والفئات غير المُستَهلكة في وحدة `url` في Node.js، موجهاً للمطورين المتقدمين الذين يسعون لفهم عميق لكيفية عمل معالجة عناوين URL والحالات الاستخدامية العملية. تشمل الوثائق مقارنات شاملة بين الـ API المختلفة، نصائح الأداء، والأخطاء الشائعة التي يجب تجنبها.

![Node.js URL Module Architecture and APIs](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/0732a0fdabdd4773cf6a948623c9c171/aec205dc-d578-412e-91b7-23e97e88d695/8c166156.png)

Node.js URL Module Architecture and APIs

## 1. نظرة عامة على الوحدة (Module Overview)

وحدة `url` في Node.js توفر مجموعة شاملة من الأدوات المساعدة لمعالجة وتحليل عناوين URL (Uniform Resource Locators). تم تصميمها لتتوافق مع معيار WHATWG الذي تستخدمه متصفحات الويب الحديثة، مما يسمح للمطورين بكتابة كود موحد يعمل بشكل متسق في بيئات المتصفح وخوادم Node.js.[^3_1][^3_2]

تقدم الوحدة **واجهتين برمجيتين رئيسيتين**: الأولى هي معيار WHATWG الحديث (Modern API) وهي الخيار المفضل لجميع الأكواد الجديدة، والثانية هي الـ Legacy API التي تم الحفاظ على دعمها للتوافقية العكسية مع الأكواد القديمة. يجب على المطورين تجنب استخدام Legacy API تماماً عند بدء مشاريع جديدة لأنها تعاني من مشاكل أمان وقابلية صيانة.[^3_3][^3_1]

### متى تستخدم وحدة URL؟

تُستخدم هذه الوحدة في السيناريوهات التالية:

- **تحليل عناوين URL المعقدة**: عند الحاجة لاستخراج مكونات محددة من عنوان URL مثل البروتوكول والمضيف ومسار الملف
- **بناء عناوين URL برمجياً**: عند الحاجة لإنشاء عناوين URL من خلال تجميع الأجزاء المختلفة
- **معالجة معاملات البحث (Query Parameters)**: عند العمل مع المعاملات الديناميكية في طلبات HTTP
- **تحويل مسارات الملفات**: عند تحويل مسارات نظام الملفات إلى عناوين URL والعكس
- **معالجة النطاقات الدولية (Internationalized Domains)**: عند العمل مع أسماء نطاقات تحتوي على أحرف Unicode

## 2. تفاصيل API الحديثة (WHATWG API Deep Dive)

### الفئة الأساسية: URL

تمثل فئة `URL` العمود الفقري لمعالجة عناوين الويب في Node.js. كل خاصية في هذه الفئة مُنفذة كـ getter و setter على نموذج الفئة الأولي (prototype)، وليست كخاصية بيانات مباشرة على الكائن نفسه.[^3_1]

#### **Constructor: new URL(input[, base])**

```javascript
// مثال 1: تحليل عنوان URL مطلق
const url1 = new URL(
  "https://user:password@example.com:8080/path?query=value#hash"
);

// مثال 2: تحليل عنوان نسبي باستخدام base URL
const url2 = new URL("/products/laptop", "https://shop.example.com");
// النتيجة: https://shop.example.com/products/laptop

// مثال 3: معالجة النطاقات الدولية
const url3 = new URL("https://مثال.السعودية");
// يتم تحويلها تلقائياً إلى Punycode: https://xn--mgbh0fb.xn--mgberp4a5d4ar/
```

**جدول المعاملات:**

| المعامل | النوع  | مطلوب/اختياري | الوصف                                                                                                    |
| :------ | :----- | :------------ | :------------------------------------------------------------------------------------------------------- |
| `input` | string | مطلوب         | عنوان URL مطلق أو نسبي. إذا كان نسبياً، يجب توفير `base`. إذا لم يكن string، يتم تحويله إلى string أولاً |
| `base`  | string | اختياري       | عنوان URL الأساسي المستخدم لحل العناوين النسبية. يتم تجاهله إذا كان `input` مطلقاً                       |

**القيمة المرجعة:** كائن `URL` يحتوي على جميع مكونات عنوان الويب.

**الأخطاء الشائعة:**

- عدم توفير `base` عند محاولة تحليل عنوان نسبي، مما يؤدي إلى رفع `TypeError`
- افتراض أن `delete` سيحذف الخصائص (لا يعمل مع كائنات URL)[^3_1]
- عدم التعامل مع النطاقات الدولية بشكل صحيح قبل إرسالها عبر الشبكة

#### **الخصائص الأساسية (Core Properties)**

##### **url.href** (مقروء/قابل للكتابة)

يحصل على أو يعيّن السلسلة المسلسلة الكاملة للـ URL.

```javascript
const url = new URL("https://example.com/path");
console.log(url.href); // 'https://example.com/path'

// تعديل القيمة
url.href = "https://newsite.com/newpath";
// هذا مكافئ لـ: url = new URL('https://newsite.com/newpath')
```

##### **url.protocol** (مقروء/قابل للكتابة)

يحصل على أو يعيّن بروتوكول عنوان الويب (مثل `http:`, `https:`, `ftp:`).

```javascript
const url = new URL("https://example.com");
console.log(url.protocol); // 'https:'

// تغيير البروتوكول - لكن فقط بين الـ "special" schemes
url.protocol = "http";
console.log(url.href); // 'http://example.com'

// محاولة تغيير إلى بروتوكول غير خاص (special) يتم تجاهلها
url.protocol = "custom";
console.log(url.protocol); // 'http:' (لم يتغير)
```

**البروتوكولات الخاصة (Special Schemes)** التي تدعم التبديل: `ftp`, `file`, `http`, `https`, `ws`, `wss`

##### **url.hostname** و **url.host** (مقروء/قابل للكتابة)

- `hostname`: اسم المضيف فقط بدون المنفذ
- `host`: اسم المضيف مع المنفذ (إن وُجد)

```javascript
const url = new URL("https://api.example.com:3000/data");

console.log(url.hostname); // 'api.example.com'
console.log(url.host); // 'api.example.com:3000'

// تعديل الـ hostname (لا يؤثر على المنفذ)
url.hostname = "api.newsite.com";
console.log(url.href); // 'https://api.newsite.com:3000/data'

// تعديل الـ host (يتضمن تغيير المنفذ)
url.host = "api.example.com:8080";
console.log(url.href); // 'https://api.example.com:8080/data'
```

##### **url.port** (مقروء/قابل للكتابة)

يحصل على أو يعيّن رقم المنفذ. الكود التالي يوضح السلوك المعقد للمنافذ الافتراضية:

```javascript
const url = new URL("https://example.com:8443");
console.log(url.port); // '8443'

// المنافذ الافتراضية تُحول إلى سلسلة نصية فارغة
url.port = "443"; // المنفذ الافتراضي لـ HTTPS
console.log(url.port); // '' (سلسلة فارغة)
console.log(url.href); // 'https://example.com/' (بدون حرف :)

// المنافذ خارج النطاق (1-65535) يتم تجاهلها
url.port = 99999;
console.log(url.port); // '443' (لم يتغير)

// الأرقام المنطقية مع بادئة صالحة يتم قبولها
url.port = "8080abc";
console.log(url.port); // '8080'
```

##### **url.pathname** (مقروء/قابل للكتابة)

يحصل على أو يعيّن جزء المسار من عنوان الويب.

```javascript
const url = new URL("https://api.example.com/users/123/profile");
console.log(url.pathname); // '/users/123/profile'

// تغيير المسار
url.pathname = "/products/456";
console.log(url.href); // 'https://api.example.com/products/456'

// الأحرف غير الصالحة يتم تشفيرها تلقائياً
url.pathname = "/file with spaces.pdf";
console.log(url.href); // 'https://api.example.com/file%20with%20spaces.pdf'
```

**حالة استخدام واقعية:**

```javascript
// بناء مسارات API ديناميكية
const baseUrl = new URL("https://api.github.com");
baseUrl.pathname = "/users/octocat/repos";
// النتيجة: https://api.github.com/users/octocat/repos
```

##### **url.search** و **url.searchParams** (مقروء/قابل للكتابة)

- `search`: السلسلة الكاملة للمعاملات (بما فيها علامة `?`)
- `searchParams`: كائن `URLSearchParams` يسمح بمعالجة المعاملات بشكل برمجي

```javascript
const url = new URL("https://example.com/search?q=nodejs&limit=10&lang=ar");

// الوصول إلى السلسلة الخام
console.log(url.search); // '?q=nodejs&limit=10&lang=ar'

// الوصول إلى معامل محدد
console.log(url.searchParams.get("q")); // 'nodejs'
console.log(url.searchParams.get("lang")); // 'ar'

// تعديل المعاملات
url.searchParams.set("limit", "20");
url.searchParams.append("sort", "stars");
console.log(url.href);
// 'https://example.com/search?q=nodejs&limit=20&lang=ar&sort=stars'

// ملاحظة مهمة: تعديل searchParams يؤثر على search تلقائياً
// لكن تعديل search مباشرة لا يؤثر على searchParams
```

**تحذير الأمان:** عند استخدام `.searchParams` لتعديل الـ URL، يجب الانتباه إلى أن بعض الأحرف قد تُشفّر بطريقة مختلفة عما يتوقعه المطور.[^3_1]

##### **url.hash** (مقروء/قابل للكتابة)

يحصل على أو يعيّن جزء الـ fragment من عنوان الويب (الجزء بعد `#`).

```javascript
const url = new URL("https://example.com/docs#introduction");
console.log(url.hash); // '#introduction'

url.hash = "conclusion";
console.log(url.href); // 'https://example.com/docs#conclusion'

// الأحرف الخاصة يتم تشفيرها
url.hash = "section-with-special-#-chars";
console.log(url.href); // 'https://example.com/docs#section-with-special-%23-chars'
```

##### **url.username** و **url.password** (مقروء/قابل للكتابة)

الوصول إلى بيانات المصادقة المضمنة في الـ URL. **تنبيه أمني:** لا تستخدم كلمات المرور في عناوين URL![^3_1]

```javascript
const url = new URL("https://admin:secretpass@api.example.com");
console.log(url.username); // 'admin'
console.log(url.password); // 'secretpass'

url.username = "newadmin";
url.password = "newpass";
console.log(url.href); // 'https://newadmin:newpass@api.example.com'
```

##### **url.origin** (مقروء فقط)

يرجع الأصل (Origin) وهو البروتوكول + المضيف + المنفذ فقط.

```javascript
const url = new URL("https://user:pass@example.com:8080/path?query#hash");
console.log(url.origin); // 'https://example.com:8080'
// لاحظ أن username و password لا يُضمنان في origin
```

#### **الدوال الثابتة (Static Methods)**

##### **URL.canParse(input[, base])** - دالة للتحقق من صلاحية الـ URL

```javascript
const isValid1 = URL.canParse("https://example.com"); // true
const isValid2 = URL.canParse("/path/only"); // false (نسبي بدون base)
const isValid3 = URL.canParse("/path", "https://example.com"); // true

// حالة استخدام: التحقق من صحة عنوان قبل معالجته
function processURL(urlString) {
  if (!URL.canParse(urlString)) {
    throw new Error(`Invalid URL: ${urlString}`);
  }
  return new URL(urlString);
}
```

##### **URL.parse(input[, base])** - دالة بديلة للمُنشئ

```javascript
const url = URL.parse("https://example.com/path");
console.log(url); // كائن URL أو null

// الفرق مع المُنشئ: يرجع null بدلاً من رفع استثناء
const invalidUrl = URL.parse("not a valid url");
console.log(invalidUrl); // null

// حالة استخدام آمنة:
const parsed = URL.parse(userInput);
if (parsed) {
  // معالجة العنوان
} else {
  // التعامل مع الخطأ
}
```

#### **الدوال الخاصة بـ Blob (لـ File Handling)**

##### **URL.createObjectURL(blob)** و **URL.revokeObjectURL(id)**

```javascript
const { Blob, resolveObjectURL } = require("node:buffer");

// إنشاء Blob وتحويله إلى URL
const blob = new Blob(["Hello World"]);
const blobUrl = URL.createObjectURL(blob);
console.log(blobUrl); // 'blob:nodedata:...'

// استرجاع الـ Blob لاحقاً
const retrievedBlob = resolveObjectURL(blobUrl);

// تحرير الذاكرة (مهم جداً لتجنب تسرب الذاكرة)
URL.revokeObjectURL(blobUrl);

// حالة استخدام عملية:
function createFileDownloadLink(content, filename) {
  const blob = new Blob([content], { type: "text/plain" });
  return URL.createObjectURL(blob);
}
```

#### **دوال التسلسل النصي**

##### **url.toString()** و **url.toJSON()**

```javascript
const url = new URL("https://example.com/path");

// الطريقتان تُرجعان نفس النتيجة
console.log(url.toString()); // 'https://example.com/path'
console.log(url.toJSON()); // 'https://example.com/path'
console.log(url.href); // 'https://example.com/path'

// الفائدة: toJSON يُستدعى تلقائياً عند استخدام JSON.stringify
const urls = [new URL("https://example.com"), new URL("https://github.com")];
console.log(JSON.stringify(urls));
// ["https://example.com/","https://github.com/"]
```

### الفئة الثانية: URLSearchParams

تُستخدم هذه الفئة للتعامل مع معاملات البحث (Query Parameters) بطريقة معيارية. تحتوي على جميع المعاملات كأزواج مفتاح-قيمة، وتدعم المفاتيح المكررة.[^3_1]

#### **البنّاءات (Constructors)**

```javascript
// 1. URLSearchParams فارغة
const params1 = new URLSearchParams();

// 2. من سلسلة نصية
const params2 = new URLSearchParams("foo=bar&abc=xyz&abc=123");

// 3. من كائن عادي (ملاحظة: لا يدعم المفاتيح المكررة)
const params3 = new URLSearchParams({ user: "john", limit: "10" });

// 4. من مصفوفة أو iterable
const params4 = new URLSearchParams([
  ["foo", "bar"],
  ["abc", "xyz"],
  ["abc", "123"], // المفاتيح المكررة مدعومة هنا
]);

// 5. استنساخ URLSearchParams موجودة
const params5 = new URLSearchParams(params2);
```

#### **دوال المعالجة الرئيسية**

```javascript
const params = new URLSearchParams();

// append: إضافة معامل جديد (يدعم المفاتيح المكررة)
params.append("color", "red");
params.append("color", "blue");
params.append("size", "large");

// set: تعيين معامل (يستبدل جميع القيم السابقة)
params.set("color", "green"); // يستبدل 'red' و 'blue' بـ 'green'

// get: الحصول على أول قيمة
console.log(params.get("color")); // 'green'

// getAll: الحصول على جميع القيم
console.log(params.getAll("color")); // ['green']

// has: التحقق من وجود معامل
console.log(params.has("size")); // true

// delete: حذف معامل
params.delete("color");
console.log(params.has("color")); // false
```

#### **التكرار والفرز**

```javascript
const params = new URLSearchParams("z=1&a=2&b=3");

// forEach: التكرار على جميع المعاملات
params.forEach((value, name) => {
  console.log(`${name}=${value}`);
});

// keys/values/entries: الحصول على iterators
for (const name of params.keys()) {
  console.log(name); // z, a, b
}

for (const value of params.values()) {
  console.log(value); // 1, 2, 3
}

for (const [name, value] of params.entries()) {
  console.log(`${name}=${value}`);
}

// أو استخدام for...of مباشرة (يستدعي entries)
for (const [name, value] of params) {
  console.log(`${name}=${value}`);
}

// sort: ترتيب المعاملات (مفيد لتحسين الـ cache)
params.sort();
console.log(params.toString()); // 'a=2&b=3&z=1'
```

#### **الخاصية size والتسلسل النصي**

```javascript
const params = new URLSearchParams("foo=bar&abc=xyz&abc=123");

console.log(params.size); // 3 (عدد جميع الأزواج، بما فيها المكررة)

console.log(params.toString()); // 'foo=bar&abc=xyz&abc=123'

// إعادة تعيين السلسلة
params.toString(); // يُرجع string جاهز للاستخدام في url.search
```

**الفروقات المهمة بين URLSearchParams والـ querystring (الوحدة القديمة):**

| الميزة                 | URLSearchParams | querystring          |
| :--------------------- | :-------------- | :------------------- |
| المفاتيح المكررة       | كل قيمة منفصلة  | تُحول إلى مصفوفة     |
| تشفير الفراغات         | `+` علامة زائد  | `%20` في البحث       |
| علامة الاستفهام الأولى | تُحذف           | تُصبح جزء المفتاح    |
| قابلية التخصيص         | محدودة          | قابلة للتخصيص الكامل |

```javascript
// مثال عملي: هجرة من querystring إلى URLSearchParams
const querystring = require("querystring");
const { URLSearchParams } = require("url");

// الطريقة القديمة
const oldParams = querystring.parse("foo=bar&abc=xyz&abc=123");
// { foo: 'bar', abc: ['xyz', '123'] }

// الطريقة الجديدة
const newParams = new URLSearchParams([
  ["foo", "bar"],
  ["abc", "xyz"],
  ["abc", "123"],
]);
```

### الفئة الثالثة: URLPattern (متقدم جداً)

توفر هذه الفئة القدرة على مطابقة عناوين URL ضد نمط معين، مما يسمح بـ routing ديناميكي وتطبيقات ويب متقدمة.

#### **البنّاء والاستخدام الأساسي**

```javascript
// نمط بسيط
const pattern = new URLPattern("https://example.com/users/:id/profile");

// اختبار إذا كان عنوان يطابق النمط
console.log(pattern.test("https://example.com/users/123/profile")); // true
console.log(pattern.test("https://example.com/users/abc/profile")); // true
console.log(pattern.test("https://example.com/users/123")); // false

// استخراج القيم من عنوان
const match = pattern.exec("https://example.com/users/123/profile");
console.log(match.pathname.groups); // { id: '123' }
```

#### **الأنماط المتقدمة**

```javascript
// استخدام wildcards
const apiPattern = new URLPattern("https://api.example.com/v*/users/*.json");

// استخدام regex groups
const routePattern = new URLPattern({
  protocol: "https",
  hostname: "example.com",
  pathname: "/api/:version/users/:id",
});

// حساسية الحروف الكبيرة والصغيرة
const caseInsensitivePattern = new URLPattern("https://example.com/*", {
  ignoreCase: true,
});
```

**حالة استخدام واقعية: نظام routing بسيط**

```javascript
class SimpleRouter {
  constructor() {
    this.routes = [];
  }

  addRoute(pattern, handler) {
    this.routes.push({
      pattern: new URLPattern(pattern),
      handler,
    });
  }

  route(url) {
    for (const route of this.routes) {
      const match = route.pattern.exec(url);
      if (match) {
        return route.handler(match);
      }
    }
    return null; // No matching route
  }
}

// الاستخدام
const router = new SimpleRouter();
router.addRoute("https://api.example.com/users/:id", (match) => {
  console.log(`Get user: ${match.pathname.groups.id}`);
});

router.route("https://api.example.com/users/456");
// Output: Get user: 456
```

## 3. الدوال العامة للوحدة (Module-Level Functions)

### دوال معالجة النطاقات الدولية

#### **url.domainToASCII(domain)** و **url.domainToUnicode(domain)**

تُستخدم هاتان الدالتان لتحويل أسماء النطاقات بين صيغة Unicode والصيغة ASCII (Punycode).[^3_1]

```javascript
const url = require("url");

// تحويل من Unicode إلى ASCII (Punycode)
console.log(url.domainToASCII("مثال.السعودية"));
// 'xn--mgbh0fb.xn--mgberp4a5d4ar'

console.log(url.domainToASCII("café.fr"));
// 'xn--caf-dma.fr'

// تحويل من ASCII إلى Unicode
console.log(url.domainToUnicode("xn--mgbh0fb.xn--mgberp4a5d4ar"));
// 'مثال.السعودية'

console.log(url.domainToUnicode("xn--caf-dma.fr"));
// 'café.fr'

// النطاقات غير الصالحة ترجع سلسلة فارغة
console.log(url.domainToASCII("invalid..domain"));
// ''
```

**حالة استخدام عملية: إرسال عناوين دولية عبر الشبكة**

```javascript
// في خادم استقبال البيانات من مستخدمين دوليين
function normalizeInternationalDomain(userInput) {
  try {
    const ascii = url.domainToASCII(userInput);
    if (ascii === "") {
      throw new Error("Invalid domain");
    }
    return ascii;
  } catch (err) {
    console.error("Domain processing error:", err);
    return null;
  }
}

const userDomain = "مثال.السعودية";
const normalizedDomain = normalizeInternationalDomain(userDomain);
// يمكن الآن استخدام Punycode بأماناً في الشبكة
```

### دوال تحويل مسارات الملفات

#### **url.pathToFileURL(path[, options])** و **url.fileURLToPath(url[, options])**

هاتان الدالتان توفر تحويلاً آمناً بين مسارات الملفات وعناوين file URLs، مع التعامل الصحيح مع الأحرف الخاصة.[^3_1][^3_4][^3_5]

```javascript
const { pathToFileURL, fileURLToPath } = require("url");

// تحويل من مسار إلى file URL
const filePath = "/home/user/documents/report.pdf";
const fileUrl = pathToFileURL(filePath);
console.log(fileUrl.href); // 'file:///home/user/documents/report.pdf'

// تحويل من file URL إلى مسار
const urlString = "file:///home/user/documents/report.pdf";
const path = fileURLToPath(urlString);
console.log(path); // '/home/user/documents/report.pdf'

// التعامل الصحيح مع الأحرف الخاصة
const pathWithSpaces = "/home/user/My Documents/file name.txt";
const urlWithSpaces = pathToFileURL(pathWithSpaces);
console.log(urlWithSpaces.href);
// 'file:///home/user/My%20Documents/file%20name.txt'

// التعامل مع النطاقات الدولية
const internationalPath = "/home/用户/文档/报告.pdf";
const internationalUrl = pathToFileURL(internationalPath);
console.log(internationalUrl.href);
// 'file:///home/%E7%94%A8%E6%88%B7/%E6%96%87%E6%A1%A3/%E6%8A%A5%E5%91%8A.pdf'
```

**مثال عملي: إنشاء رابط تحميل آمن للملفات**

```javascript
const fs = require("fs");
const path = require("path");
const { pathToFileURL } = require("url");

function createSecureFileLink(filePath) {
  // التحقق من أن الملف موجود
  if (!fs.existsSync(filePath)) {
    throw new Error(`File not found: ${filePath}`);
  }

  // الحصول على المسار الكامل
  const absolutePath = path.resolve(filePath);

  // تحويل إلى file URL بأمان
  return pathToFileURL(absolutePath).href;
}

// الاستخدام
try {
  const downloadLink = createSecureFileLink("./data/export.csv");
  console.log(`File URL: ${downloadLink}`);
  // File URL: file:///absolute/path/to/data/export.csv
} catch (err) {
  console.error(err);
}
```

#### **url.fileURLToPathBuffer(url[, options])**

نسخة متقدمة من `fileURLToPath` ترجع Buffer بدلاً من string، مفيدة للمسارات التي تحتوي على ترميز غير صحيح.[^3_1]

```javascript
const { fileURLToPathBuffer } = require("url");

// للمسارات التي قد تحتوي على بايتات غير صالحة كـ UTF-8
const urlWithInvalidUTF8 = "file:///path/with/%FF%FE/bytes";
const pathBuffer = fileURLToPathBuffer(urlWithInvalidUTF8);

console.log(pathBuffer);
// <Buffer 2f 70 61 74 68 2f 77 69 74 68 2f ff fe 2f 62 79 74 65 73>
```

### دوال التنسيق والتحويل

#### **url.format(URL[, options])**

تقدم هذه الدالة تخصيصاً مرناً لتسلسل URL النصي.[^3_1]

```javascript
const { URL } = require("url");
const url = require("url");

const myURL = new URL(
  "https://user:password@example.com:8080/path?query=value#hash"
);

// التنسيق الافتراضي (يتضمن كل شيء)
console.log(url.format(myURL));
// 'https://user:password@example.com:8080/path?query=value#hash'

// إزالة المصادقة
console.log(url.format(myURL, { auth: false }));
// 'https://example.com:8080/path?query=value#hash'

// إزالة الـ fragment والمعاملات
console.log(url.format(myURL, { fragment: false, search: false }));
// 'https://user:password@example.com:8080/path'

// استخدام Unicode بدلاً من Punycode
const intlURL = new URL("https://مثال.السعودية");
console.log(url.format(intlURL, { unicode: true }));
// 'https://مثال.السعودية/'

console.log(url.format(intlURL, { unicode: false })); // الافتراضي
// 'https://xn--mgbh0fb.xn--mgberp4a5d4ar/'
```

**جدول خيارات format:**

| الخيار     | النوع   | الافتراضي | الوصف                             |
| :--------- | :------ | :-------- | :-------------------------------- |
| `auth`     | boolean | true      | تضمين username و password         |
| `fragment` | boolean | true      | تضمين الـ hash                    |
| `search`   | boolean | true      | تضمين معاملات البحث               |
| `unicode`  | boolean | false     | استخدام Unicode بدلاً من Punycode |

#### **url.urlToHttpOptions(url)**

تحويل كائن URL إلى كائن خيارات جاهز للاستخدام مع `http.request()` و `https.request()`.[^3_1]

```javascript
const { URL, urlToHttpOptions } = require("url");

const myURL = new URL(
  "https://username:password@api.example.com:8443/v1/data?token=abc123"
);

const options = urlToHttpOptions(myURL);
console.log(options);

// {
//   protocol: 'https:',
//   hostname: 'api.example.com',
//   port: 8443,
//   pathname: '/v1/data',
//   search: '?token=abc123',
//   path: '/v1/data?token=abc123',
//   hash: '',
//   href: 'https://username:password@api.example.com:8443/v1/data?token=abc123',
//   auth: 'username:password'
// }

// استخدام مباشر في HTTP request
const https = require("https");
https
  .request(options, (res) => {
    console.log(`Status: ${res.statusCode}`);
    res.on("data", (chunk) => {
      console.log(chunk.toString());
    });
  })
  .end();
```

## 4. مقارنات ومخططات اتخاذ القرار (Comparison \& Decision Making)

### WHATWG URL API vs Legacy API

| المعيار            | WHATWG URL (حديث)              | Legacy url.parse (قديم)             |
| :----------------- | :----------------------------- | :---------------------------------- |
| **الحالة**         | معيار موحد مع المتصفحات        | مُستَهلك، غير موصى به               |
| **الأداء**         | أسرع بـ 4-5 مرات[^3_2]         | أبطأ، بدون تحسينات                  |
| **الأمان**         | معيار آمن موثوق                | عرضة للأخطاء الأمنية                |
| **معالجة الأخطاء** | يرفع استثناء لـ URLs غير صالحة | يعالجها بطريقة متساهلة              |
| **الدعم**          | مدعوم بالكامل في Node.js       | سيتم إزالته في الإصدارات المستقبلية |
| **الاستخدام**      | موصى به لجميع الأكواد الجديدة  | استخدم فقط للتوافقية العكسية        |

```javascript
// مثال: الفروقات العملية
const testUrl = "http://example.com";

// WHATWG API - الطريقة الحديثة
const modern = new URL(testUrl);
console.log(modern.hostname); // 'example.com'

// Legacy API - الطريقة القديمة (مُستَهلكة)
const legacy = require("url").parse(testUrl);
console.log(legacy.hostname); // 'example.com'
// ⚠️ تحذير: Legacy API موجود فقط للتوافقية
```

### URLSearchParams vs querystring

| الجانب               | URLSearchParams | querystring         |
| :------------------- | :-------------- | :------------------ |
| **المعيار**          | WHATWG standard | Node.js legacy      |
| **المفاتيح المكررة** | معالجة صحيحة    | تحويل إلى مصفوفة    |
| **الفراغات**         | `+` (معيار)     | `%20` (غير معياري)  |
| **التخصيص**          | محدود (معياري)  | قابل للتخصيص الكامل |
| **التوصية**          | استخدم دائماً   | قديم، تجنبه         |

```javascript
// مثال مقارنة عملي
const queryStr = "name=john doe&role=admin&role=user";

// URLSearchParams
const params1 = new URLSearchParams(queryStr);
console.log(params1.getAll("role")); // ['admin', 'user']

// querystring (طريقة قديمة)
const qs = require("querystring");
const params2 = qs.parse(queryStr);
console.log(params2.role); // ['admin', 'user'] (مصفوفة)
```

### متى تستخدم كل طريقة؟

```
┌─────────────────────────────────────────────────────┐
│          اختيار الطريقة الصحيحة                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│ هل تحتاج لمعالجة عنوان URL كاملاً؟                 │
│ ├─ نعم ──→ استخدم new URL()                       │
│ └─ لا                                              │
│    │                                               │
│    هل تحتاج لمعالجة معاملات البحث فقط؟            │
│    ├─ نعم ──→ استخدم URLSearchParams              │
│    └─ لا                                          │
│       │                                            │
│       هل تحتاج لتحويل بين مسارات و URLs؟         │
│       ├─ نعم ──→ استخدم pathToFileURL/           │
│       │         fileURLToPath                     │
│       └─ لا                                       │
│          │                                         │
│          هل تحتاج لعمليات routing متقدمة؟        │
│          ├─ نعم ──→ استخدم URLPattern            │
│          └─ لا ──→ معالجة بسيطة من أساسيات URL  │
│                                                   │
└─────────────────────────────────────────────────────┘
```

## 5. المزايا والعيوب وحالات الاستخدام (Pros, Cons \& Use Cases)

### المزايا الرئيسية

1. **معيارية عالمية**: تطبيق معيار WHATWG الموحد الذي تستخدمه جميع متصفحات الويب الحديثة، مما يسمح بمشاركة الكود بسهولة بين الخادم والعميل.[^3_1]
2. **أداء عالي جداً**: مع تحسينات Node.js 18 باستخدام مكتبة Ada لتحليل URLs، حقق تحسناً بمعدل 4-5 مرات مقارنة بالإصدارات السابقة.[^3_2][^3_6]
3. **معالجة آمنة للنطاقات الدولية**: دعم تلقائي لأسماء النطاقات بأحرف Unicode مع تحويل Punycode الصحيح.[^3_1]
4. **API موحدة وسهلة الاستخدام**: خصائص getters/setters واضحة وسهلة الفهم بدلاً من كائنات معقدة.
5. **معالجة صحيحة للأحرف الخاصة**: تشفير تلقائي وآمن للأحرف غير الآمنة في URLs.[^3_1]

### العيوب والقيود

1. **عدم المرونة العميقة**: بعض التطبيقات قد تحتاج معالجة مخصصة للعلاقات أو أنماط غير معيارية، وقد لا توفر الوحدة هذه المرونة.
2. **رفع الاستثناءات للـ URLs غير الصالحة**: قد يكون أحياناً مزعجاً، رغم أنه أكثر أماناً.[^3_3]
3. **المزاهاة مع Legacy APIs في بعض الحالات**: عند العمل مع أكواد قديمة، قد تكون بعض الاختلافات معقدة.[^3_7]
4. **تشفير مختلف للفراغات في searchParams**: URLSearchParams تستخدم `+` بينما `pathname` يستخدم `%20`، مما قد يسبب ارتباكاً.[^3_1]

### أفضل 3 حالات استخدام

#### 1. **بناء وتحليل URLs في تطبيقات ويب الخادمية (Web Applications)**

```javascript
// في خادم Node.js - معالجة طلبات المستخدمين
const http = require("http");
const { URL } = require("url");

const server = http.createServer((req, res) => {
  const urlObject = new URL(req.url, `http://${req.headers.host}`);

  // استخراج معاملات البحث
  const searchQuery = urlObject.searchParams.get("q");
  const page = urlObject.searchParams.get("page") || 1;

  // معالجة الطلب
  if (searchQuery) {
    console.log(`Search for: ${searchQuery} (Page: ${page})`);
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ query: searchQuery, page }));
  } else {
    res.writeHead(400);
    res.end("Missing query parameter");
  }
});

server.listen(3000);
```

#### 2. **تحويل بين مسارات الملفات و URLs (File System Operations)**

```javascript
// في تطبيق إلكترون أو نظام إدارة ملفات
const { pathToFileURL, fileURLToPath } = require("url");
const fs = require("fs");
const path = require("path");

class FileManager {
  // تحويل مسار نظام ملفات إلى URL للعرض في متصفح
  getFileUrl(filePath) {
    const absolutePath = path.resolve(filePath);
    return pathToFileURL(absolutePath).href;
  }

  // تحويل عنوان URL إلى مسار آمن للقراءة من النظام
  getFilePath(fileUrl) {
    try {
      return fileURLToPath(fileUrl);
    } catch (err) {
      throw new Error("Invalid file URL");
    }
  }

  // تحميل ملف من URL
  loadFileFromUrl(fileUrl) {
    const filePath = this.getFilePath(fileUrl);
    return fs.readFileSync(filePath, "utf-8");
  }
}

// الاستخدام
const manager = new FileManager();
const fileUrl = manager.getFileUrl("./data/config.json");
console.log(`File URL: ${fileUrl}`);

const content = manager.loadFileFromUrl(fileUrl);
console.log(content);
```

#### 3. **نظام Routing ديناميكي للواجهات البرمجية (REST APIs)**

```javascript
const { URL, URLPattern } = require("url");

class APIRouter {
  constructor() {
    this.routes = [];
  }

  // تسجيل مسار API
  register(pattern, method, handler) {
    this.routes.push({
      pattern: new URLPattern(pattern),
      method,
      handler,
    });
  }

  // توجيه الطلب إلى المعالج المناسب
  route(urlString, method, body) {
    for (const route of this.routes) {
      if (route.method !== method) continue;

      const match = route.pattern.exec(urlString);
      if (match) {
        return route.handler(match, body);
      }
    }
    return { status: 404, error: "Not Found" };
  }
}

// الاستخدام
const router = new APIRouter();

// تسجيل المسارات
router.register(
  "https://api.example.com/v1/users/:userId/posts/:postId",
  "GET",
  (match, body) => {
    const userId = match.pathname.groups.userId;
    const postId = match.pathname.groups.postId;
    return {
      status: 200,
      data: `Post ${postId} by user ${userId}`,
    };
  }
);

// اختبار الـ routing
const result = router.route(
  "https://api.example.com/v1/users/123/posts/456",
  "GET",
  null
);
console.log(result);
// { status: 200, data: 'Post 456 by user 123' }
```

## 6. نصائح الأداء والأفضليات (Performance Tips \& Best Practices)

### تحسين الأداء

1. **استخدم `URL.canParse()` قبل `new URL()`**

```javascript
// بطيء - يرفع استثناء عند الخطأ
try {
  const url = new URL(userInput);
} catch (e) {
  // معالجة الخطأ
}

// أسرع - تحقق أولاً
if (URL.canParse(userInput)) {
  const url = new URL(userInput);
}
```

2. **أعد استخدام كائنات URL بدلاً من إعادة إنشاؤها**

```javascript
// سيء - إنشاء كائن جديد لكل معامل
function updateParams(urlString, params) {
  let currentUrl = new URL(urlString);
  for (const [key, value] of Object.entries(params)) {
    currentUrl.searchParams.set(key, value);
  }
  return currentUrl.href;
}

// أفضل - إعادة الاستخدام
const baseUrl = new URL("https://api.example.com/search");
function updateParams(params) {
  const url = new URL(baseUrl);
  for (const [key, value] of Object.entries(params)) {
    url.searchParams.set(key, value);
  }
  return url.href;
}
```

3. **استخدم `urlToHttpOptions()` لـ HTTP requests**

```javascript
// بدلاً من بناء الخيارات يدوياً
const url = new URL("https://api.example.com:8443/v1/data?token=abc");
const options = urlToHttpOptions(url); // آمن وكامل

https
  .request(options, (res) => {
    // ...
  })
  .end();
```

4. **استفد من `sort()` في URLSearchParams**

```javascript
// فرز المعاملات يحسن الـ caching في الخوادم
const params = new URLSearchParams("z=1&a=2&b=3");
params.sort(); // الآن: a=2&b=3&z=1
// نفس الترتيب يسمح بـ caching أفضل
```

### أفضليات الكود (Best Practices)

1. **تجنب Legacy APIs تماماً**

```javascript
// ❌ لا تستخدم
const parsed = require("url").parse("https://example.com");

// ✅ استخدم هذا
const parsed = new URL("https://example.com");
```

2. **تحقق من صحة URLs المدخلات**

```javascript
// ✅ جيد
function processUrl(userInput) {
  if (!URL.canParse(userInput)) {
    throw new Error("Invalid URL");
  }
  return new URL(userInput);
}
```

3. **استخدم URLSearchParams بدلاً من string manipulation**

```javascript
// ❌ سيء
const url = "https://api.example.com/search?q=nodejs&page=1";
const modified = url.replace("?q=nodejs", "?q=javascript");

// ✅ جيد
const urlObj = new URL("https://api.example.com/search");
urlObj.searchParams.set("q", "javascript");
urlObj.searchParams.set("page", "1");
const modified = urlObj.href;
```

4. **اعتن بـ Punycode عند التعامل مع أسماء نطاقات دولية**

```javascript
// عند قبول أسماء نطاقات من المستخدمين
function validateInternationalDomain(domain) {
  const ascii = url.domainToASCII(domain);
  if (ascii === "") {
    throw new Error("Invalid domain");
  }
  return ascii; // احفظ النسخة ASCII للشبكة
}
```

## الخلاصة (Conclusion)

وحدة Node.js `url` توفر حلاً شاملاً وموثوقاً وسريعاً لجميع عمليات معالجة عناوين الويب. من خلال استخدام معيار WHATWG الموحد، تضمن التوافقية مع متصفحات الويب وتوفر أداءً محسّناً بشكل ملحوظ.

**التوصيات النهائية:**

1. استخدم `new URL()` لتحليل أي عنوان ويب[^3_1]
2. استخدم `URLSearchParams` لمعالجة معاملات البحث[^3_1]
3. تجنب `url.parse()` والـ Legacy APIs بشكل كامل[^3_2][^3_3]
4. استفد من `pathToFileURL()` و `fileURLToPath()` للعمل الآمن مع الملفات[^3_1]
5. استخدم `URLPattern` للتطبيقات المتقدمة التي تحتاج routing معقد[^3_1]

مع فهم عميق لهذه الأدوات، ستكون قادراً على بناء تطبيقات ويب متينة وآمنة وفعالة في Node.js.
<span style="display:none">[^3_10][^3_11][^3_12][^3_13][^3_14][^3_15][^3_16][^3_17][^3_18][^3_19][^3_20][^3_21][^3_22][^3_23][^3_24][^3_25][^3_26][^3_27][^3_28][^3_29][^3_30][^3_31][^3_32][^3_33][^3_34][^3_35][^3_36][^3_37][^3_38][^3_39][^3_40][^3_41][^3_42][^3_43][^3_44][^3_45][^3_46][^3_47][^3_48][^3_49][^3_50][^3_51][^3_52][^3_53][^3_54][^3_55][^3_56][^3_57][^3_58][^3_59][^3_60][^3_61][^3_62][^3_8][^3_9]</span>

<div align="center">⁂</div>

[^3_1]: https://nodejs.org/api/url.html
[^3_2]: https://arxiv.org/pdf/2311.10533.pdf
[^3_3]: https://stackoverflow.com/questions/59375013/node-legacy-url-parse-deprecated-what-to-use-instead
[^3_4]: https://www.geeksforgeeks.org/node-js/node-js-url-pathtofileurl-api/
[^3_5]: https://www.geeksforgeeks.org/node-js/node-js-url-fileurltopath-api/
[^3_6]: https://blog.rafaelgss.dev/state-of-nodejs-performance-2023
[^3_7]: https://www.linkedin.com/pulse/how-migrate-from-querystring-urlsearchparams-nodejs-vladimír-gorej
[^3_8]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^3_9]: https://academic.oup.com/nar/article/53/D1/D495/7848843
[^3_10]: https://www.semanticscholar.org/paper/23cac78121a77ac997ccc504a126589d65b4d6a7
[^3_11]: https://ieeexplore.ieee.org/document/9652617/
[^3_12]: https://academic.oup.com/database/article/doi/10.1093/database/bax052/4049442
[^3_13]: https://www.semanticscholar.org/paper/565fe632320cce6468650fb01687d43bbccc93b3
[^3_14]: https://www.semanticscholar.org/paper/3cfb194cfbe04f6e7683c94c71ee9679d35d1959
[^3_15]: https://arxiv.org/pdf/2109.01002.pdf
[^3_16]: http://arxiv.org/pdf/1802.08391.pdf
[^3_17]: https://arxiv.org/pdf/1205.6363.pdf
[^3_18]: http://arxiv.org/pdf/1704.07887.pdf
[^3_19]: https://arxiv.org/html/2403.10205v1
[^3_20]: https://arxiv.org/pdf/2108.08027.pdf
[^3_21]: https://www.mdpi.com/2078-2489/8/4/148/pdf?version=1510913285
[^3_22]: http://arxiv.org/pdf/2411.08833.pdf
[^3_23]: https://www.nodeapp.cn/url.html
[^3_24]: https://yinm.info/20201110/
[^3_25]: https://blog.csdn.net/angkangcong2367/article/details/102003403
[^3_26]: https://www.w3schools.com/nodejs/nodejs_url.asp
[^3_27]: https://www.nodebeginner.org/blog/post/nodejs-tutorial-whatwg-url-parser/
[^3_28]: https://www.geeksforgeeks.org/node-js/node-js-url-parseurlstring-parsequerystring-slashesdenotehost-api/
[^3_29]: https://www.npmjs.com/package/whatwg-url
[^3_30]: https://stackoverflow.com/questions/7517332/node-js-url-parse-result-back-to-string
[^3_31]: https://www.geeksforgeeks.org/node-js/node-js-url-complete-reference/
[^3_32]: https://stackoverflow.com/questions/48196706/new-url-whatwg-url-api
[^3_33]: https://www.linkedin.com/pulse/what-url-module-nodejs-manoj-shrestha
[^3_34]: https://www.geeksforgeeks.org/node-js/node-js-url-method/
[^3_35]: https://nodejs.org/download/rc/v14.17.4-rc.0/docs/api/url.html
[^3_36]: https://stackoverflow.com/questions/17184791/node-js-url-parse-and-pathname-property
[^3_37]: https://www.youtube.com/watch?v=6X9LxaUSngw
[^3_38]: https://www.w3tutorials.net/blog/nodejs-whatwg-url-api/
[^3_39]: https://www.npmjs.com/package/url-parse
[^3_40]: https://nodejs.org/download/release/v6.7.0/docs/api/url.html
[^3_41]: https://url.spec.whatwg.org
[^3_42]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/spe.3296
[^3_43]: http://www.journalijdr.com/sites/default/files/issue-pdf/18695.pdf
[^3_44]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/spe.3313
[^3_45]: https://www.aclweb.org/anthology/P15-1038.pdf
[^3_46]: http://arxiv.org/pdf/2409.07360.pdf
[^3_47]: https://arxiv.org/html/2312.17149v3
[^3_48]: https://arxiv.org/pdf/2503.01662.pdf
[^3_49]: https://onlinelibrary.wiley.com/doi/full/10.1002/spe.3296
[^3_50]: https://robertmarshall.dev/blog/migrating-from-query-string-to-urlsearchparams/
[^3_51]: http://nodesource.com/blog/State-of-Nodejs-Performance-2024/
[^3_52]: https://www.brandonpugh.com/til/javascript/urlsearchparams-query-string/
[^3_53]: https://stackoverflow.com/questions/72046841/url-pathtofileurl-is-not-a-function-how-to-convert-a-file-path-to-an-url-in-no
[^3_54]: https://stackoverflow.com/questions/39266970/what-is-the-difference-between-url-parameters-and-query-strings
[^3_55]: https://www.tutorialspoint.com/nodejs/nodejs_fileurltopath_method.htm
[^3_56]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[^3_57]: https://github.com/nodejs/node/issues/41521
[^3_58]: https://github.com/nodejs/performance/issues/33
[^3_59]: https://dev.to/nerdyman/replacing-query-string-with-native-urlsearchparams-4kdg
[^3_60]: https://exploringjs.com/nodejs-shell-scripting/ch_nodejs-path.html
[^3_61]: https://github.com/nodejs/node/issues/643
[^3_62]: https://www.reddit.com/r/golang/comments/10huint/why_and_when_should_i_use_url_parameters_over_the/

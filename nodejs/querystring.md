# دليل عميق شامل لوحدة Node.js: querystring

وحدة `querystring` في Node.js توفر مجموعة من الأدوات القوية والسريعة لمعالجة سلاسل الاستعلام (Query Strings) في عناوين URL. هذا الدليل يقدم شرحاً تفصيلياً وعملياً لكل دالة وخاصية في الوحدة، مع أمثلة واقعية وحالات استخدام فعلية. الوحدة توفر أداء عالية جداً (أسرع بـ 3 مرات من `URLSearchParams`)، لكنها تم تصنيفها كـ "Legacy API" منذ Node.js 14، والبديل الموصى به هو `URLSearchParams` من معيار WHATWG للكود الجديد.[^1_1][^1_2][^1_3]

![The querystring module processing flow showing transformation of URL query strings](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/35b5eb7769441e9fbcb3acc251e9dd8a/1c805088-e3b9-4195-a211-a47e1c78b587/c2682f3a.png)

The querystring module processing flow showing transformation of URL query strings

## نظرة عامة على الوحدة

وحدة `querystring` توفر واجهة برمجية بسيطة وفعالة لتحويل سلاسل الاستعلام إلى كائنات JavaScript والعكس. على سبيل المثال، إذا كان لديك URL مثل `https://example.com/search?name=أحمد&age=30`، تستطيع هذه الوحدة استخراج المعاملات وتحويلها إلى كائن قابل للاستخدام في البرنامج.[^1_4][^1_1]

**الاستقرار والحالة:** الوحدة تتمتع باستقرار تام (Stability Level: 2) وآمنة للاستخدام في بيئات الإنتاج. ومع ذلك، بدءاً من Node.js 14، تم تصنيفها كـ "Legacy API" (واجهة برمجية موروثة)، ويُفضل استخدام `URLSearchParams` للكود الجديد.[^1_1][^1_2]

### متى تستخدم querystring بدلاً من البدائل

استخدم `querystring` في الحالات التالية:

- **تطبيقات الويب:** معالجة معاملات البحث والفلاتر والترقيم في عناوين URL[^1_5][^1_4]
- **واجهات برمجية (APIs):** تحليل معاملات الاستعلام من الطلبات الواردة بسرعة عالية[^1_6]
- **معالجة النماذج:** استخراج بيانات النماذج المرسلة عبر GET requests[^1_4]
- **أداء عالي:** عندما تكون السرعة حرجة (أسرع من URLSearchParams بـ ~3x)[^1_2][^1_7]
- **معاملات مخصصة:** عندما تحتاج إلى فاصلات غير القياسية (`&` و `=`)[^1_8][^1_1]

## API العميقة: الدوال والخصائص

### 1. دالة `querystring.parse()` - تحويل سلسلة استعلام إلى كائن

هذه الدالة تقوم بتحويل سلسلة استعلام (مثل `foo=bar&baz=qux`) إلى كائن JavaScript قابل للاستخدام.[^1_1]

**التوقيع:**

```javascript
querystring.parse(str[, sep[, eq[, options]]])
```

**المعاملات:**

| المعامل                      | النوع    | مطلوب/اختياري | الوصف                                       |
| :--------------------------- | :------- | :------------ | :------------------------------------------ |
| `str`                        | String   | مطلوب         | سلسلة الاستعلام المراد تحليلها              |
| `sep`                        | String   | اختياري       | الفاصل بين الأزواج (الافتراضي: `&`)         |
| `eq`                         | String   | اختياري       | الفاصل بين المفتاح والقيمة (الافتراضي: `=`) |
| `options.decodeURIComponent` | Function | اختياري       | دالة فك ترميز مخصصة                         |
| `options.maxKeys`            | Number   | اختياري       | الحد الأقصى للمفاتيح (الافتراضي: 1000)      |

**القيمة المرجعة:** كائن JavaScript بدون نموذج أولي من `Object`.[^1_1]

**مثال عملي واقعي - معالجة معاملات البحث:**

```javascript
const querystring = require('node:querystring');

// استخراج معاملات البحث من URL حقيقي
const searchQuery = 'q=nodejs%20tutorials&category=backend&page=2&sort=recent';
const parsed = querystring.parse(searchQuery);

console.log(parsed);
// {
//   q: 'nodejs tutorials',
//   category: 'backend',
//   page: '2',
//   sort: 'recent'
// }

// استخدام في تطبيق Express حقيقي
app.get('/search', (req, res) => {
  const query = req.url.split('?')[^1_1];
  const params = querystring.parse(query);

  const searchTerm = params.q;
  const pageNum = parseInt(params.page) || 1;

  // البحث في قاعدة بيانات حقيقية
  database.search(searchTerm, pageNum).then(results => {
    res.json(results);
  });
});
```

**معالجة الفاصلات المخصصة:** يمكنك استخدام فاصلات غير قياسية إذا كان لديك متطلبات خاصة:[^1_9][^1_1]

```javascript
const querystring = require("node:querystring");

// بدلاً من foo=bar&baz=qux، استخدم foo:bar;baz:qux
const customQuery = "username:john;password:secret;role:admin";
const parsed = querystring.parse(customQuery, ";", ":");

console.log(parsed);
// { username: 'john', password: 'secret', role: 'admin' }
```

**معالجة المفاتيح المكررة:** عندما تكون هناك مفاتيح متطابقة، يتم تحويلها إلى مصفوفة:[^1_1]

```javascript
const querystring = require("node:querystring");

// مثال حقيقي: فلتات متعددة في متجر إلكتروني
const filterQuery =
  "category=electronics&tag=popular&tag=bestseller&tag=onSale";
const filters = querystring.parse(filterQuery);

console.log(filters);
// {
//   category: 'electronics',
//   tag: ['popular', 'bestseller', 'onSale']
// }

// استخدام النتائج في البحث
const products = findProducts({
  category: filters.category,
  tags: { $in: filters.tag },
});
```

**الأخطاء الشائعة والحلول:**

1. **عدم معالجة الترميز غير UTF-8:** إذا كنت تستخدم ترميز مخصص (مثل GBK)، تحتاج إلى توفير دالة فك ترميز مخصصة.[^1_1]
2. **عدم التحقق من نوع القيمة المكررة:** قد تكون القيمة مصفوفة بدلاً من سلسلة، لذا تحقق دائماً من النوع.[^1_9]
3. **استخدام maxKeys:** بشكل افتراضي، يتم معالجة 1000 مفتاح فقط، وقد تفقد المفاتيح الإضافية.[^1_1]

---

### 2. دالة `querystring.stringify()` - تحويل كائن إلى سلسلة استعلام

هذه الدالة تقوم بالعملية العكسية - تحويل كائن JavaScript إلى سلسلة استعلام.[^1_1]

**التوقيع:**

```javascript
querystring.stringify(obj[, sep[, eq[, options]]])
```

**المعاملات:**

| المعامل                      | النوع    | مطلوب/اختياري | الوصف                                       |
| :--------------------------- | :------- | :------------ | :------------------------------------------ |
| `obj`                        | Object   | مطلوب         | الكائن المراد تحويله                        |
| `sep`                        | String   | اختياري       | الفاصل بين الأزواج (الافتراضي: `&`)         |
| `eq`                         | String   | اختياري       | الفاصل بين المفتاح والقيمة (الافتراضي: `=`) |
| `options.encodeURIComponent` | Function | اختياري       | دالة ترميز مخصصة                            |

**الأنواع المدعومة:** strings, numbers, bigints, booleans، والمصفوفات من هذه الأنواع.[^1_1]

**مثال عملي - بناء عناوين URL ديناميكية:**

```javascript
const querystring = require("node:querystring");

// حالة واقعية: بناء رابط بحث في متجر إلكتروني
function buildProductSearchUrl(baseUrl, filters) {
  const query = querystring.stringify(filters);
  return `${baseUrl}?${query}`;
}

const searchParams = {
  search: "سماعات رأس",
  category: "electronics",
  minPrice: 100,
  maxPrice: 500,
  inStock: true,
  page: 1,
};

const url = buildProductSearchUrl(
  "https://shop.example.com/products",
  searchParams
);

console.log(url);
// https://shop.example.com/products?search=%D8%B3%D9%85%D8%A7%D8%B9%D8%A7%D8%AA%20%D8%B1%D8%A3%D8%B3&category=electronics&minPrice=100&maxPrice=500&inStock=true&page=1
```

**معالجة المصفوفات:** عندما تكون القيمة مصفوفة، يتم إنشاء مفاتيح مكررة:[^1_6]

```javascript
const querystring = require("node:querystring");

// حالة واقعية: فلترة متقدمة بوسوم متعددة
const params = {
  search: "نتائج البحث",
  tags: ["مهم", "عاجل", "جديد"],
  authors: ["علي", "محمد", "فاطمة"],
};

const query = querystring.stringify(params);
console.log(query);
// search=%D9%86%D8%AA%D8%A7%D8%A6%D8%AC%20%D8%A7%D9%84%D8%A8%D8%AD%D8%AB&tags=%D9%85%D9%87%D9%85&tags=%D8%B9%D8%A7%D8%AC%D9%84&tags=%D8%AC%D8%AF%D9%8A%D8%AF&authors=%D8%B9%D9%84%D9%8A&authors=%D9%85%D8%AD%D9%85%D8%AF&authors=%D9%81%D8%A7%D8%B7%D9%85%D8%A9
```

**الأخطاء الشائعة:**

1. **قيم غير محدودة:** الأرقام `Infinity` و `NaN` يتم تحويلها إلى سلاسل فارغة.[^1_1]
2. **الكائنات المتداخلة:** لا يتم دعمها بشكل مباشر.[^1_5]
3. **القيم undefined:** يتم حذفها من النتيجة.[^1_1]

---

### 3. دالة `querystring.escape()` - ترميز السلاسل

تقوم بترميز السلسلة باستخدام percent-encoding المحسّن خصيصاً لمعاملات الاستعلام.[^1_1]

**التوقيع:**

```javascript
querystring.escape(str);
```

**مثال واقعي:**

```javascript
const querystring = require("node:querystring");

// ترميز قيم واحدة لاستخدام يدوي
const userQuery = "Node.js & Express - Tutorial 2024";
const encoded = querystring.escape(userQuery);

console.log(encoded);
// Node.js%20%26%20Express%20-%20Tutorial%202024

// استخدام عملي: إعادة تعيين دالة الترميز
const customEscape = (str) => {
  return str
    .replace(/\s+/g, "+") // استخدم + للمسافات بدلاً من %20
    .replace(/[!]/g, "%21") // ترميز إضافي للإشارات
    .replace(/[*]/g, "%2A");
};

querystring.escape = customEscape;

const params = { message: "Hello World! This is *important*" };
const query = querystring.stringify(params);
console.log(query);
// message=Hello+World%21+This+is+%2Aimportant%2A
```

---

### 4. دالة `querystring.unescape()` - فك الترميز

تقوم بفك ترميز السلاسل المشفرة بصيغة percent-encoding.[^1_1]

**التوقيع:**

```javascript
querystring.unescape(str);
```

**مثال عملي:**

```javascript
const querystring = require("node:querystring");

// فك ترميز قيمة مشفرة
const encodedValue = "Hello%20World%21%20This%20is%20Node.js";
const decoded = querystring.unescape(encodedValue);

console.log(decoded);
// Hello World! This is Node.js

// معالجة النصوص العربية والصينية
const encodedArabic = "%D8%A7%D9%84%D8%B9%D8%B1%D8%A8%D9%8A%D8%A9";
const decodedArabic = querystring.unescape(encodedArabic);

console.log(decodedArabic);
// العربية
```

---

### 5. أسماء بديلة (Aliases)

الوحدة توفر أسماء بديلة مفيدة:[^1_1]

```javascript
const querystring = require("node:querystring");

// querystring.decode هو بديل لـ querystring.parse
const query1 = querystring.parse("a=1&b=2");
const query2 = querystring.decode("a=1&b=2");
// query1 و query2 متطابقان

// querystring.encode هو بديل لـ querystring.stringify
const str1 = querystring.stringify({ a: 1, b: 2 });
const str2 = querystring.encode({ a: 1, b: 2 });
// str1 و str2 متطابقان
```

---

## مقارنات واختيارات

### querystring مقابل URLSearchParams

| الميزة              | querystring           | URLSearchParams        |
| :------------------ | :-------------------- | :--------------------- |
| **المعيار**         | Node.js legacy        | WHATWG standard        |
| **الأداء**          | أسرع بـ ~3x           | أبطأ[^1_2][^1_7]       |
| **الفاصل المخصص**   | مدعوم بالكامل         | غير مدعوم              |
| **توافق المتصفح**   | لا                    | نعم[^1_2]              |
| **الترميز المخصص**  | مدعوم بالكامل         | محدود                  |
| **الحالة الحالية**  | Legacy (مستقرة)       | الموصى به للكود الجديد |
| **معالجة المسافات** | %20 (percent-encoded) | + (plus-encoded)[^1_2] |

**متى تختار querystring:** عندما تحتاج أداء عالية في تطبيقات Node.js الخادمية، أو عندما تحتاج فاصلات مخصصة، أو عندما يكون الكود القديم يستخدمها بالفعل.[^1_7][^1_2]

**متى تختار URLSearchParams:** للكود الجديد الذي يحتاج توافق مع المتصفح، أو عندما لا تكون الأداء حرجة.[^1_3][^1_2]

---

## حالات الاستخدام والتطبيقات الواقعية

### تطبيق عملي 1: محرك بحث كامل

```javascript
const http = require("http");
const querystring = require("node:querystring");

http
  .createServer((req, res) => {
    if (req.url.startsWith("/search")) {
      const [path, query] = req.url.split("?");
      const params = querystring.parse(query);

      const searchTerm = params.q || "";
      const page = parseInt(params.page) || 1;
      const filters = params.category;

      // بحث في قاعدة البيانات
      database.search(searchTerm, page, filters).then((results) => {
        res.writeHead(200, { "Content-Type": "application/json" });
        res.end(JSON.stringify(results));
      });
    }
  })
  .listen(3000);
```

### تطبيق عملي 2: معالجة نماذج POST

```javascript
const http = require("http");
const querystring = require("node:querystring");

http
  .createServer((req, res) => {
    if (req.method === "POST" && req.url === "/login") {
      let body = "";

      req.on("data", (chunk) => {
        body += chunk.toString();
      });

      req.on("end", () => {
        const credentials = querystring.parse(body);
        const { username, password } = credentials;

        if (isValidCredentials(username, password)) {
          res.writeHead(200);
          res.end("Login successful!");
        } else {
          res.writeHead(401);
          res.end("Invalid credentials");
        }
      });
    }
  })
  .listen(3000);
```

---

## المزايا والعيوب التفصيلية

**المزايا الرئيسية:**

- ✅ **أداء عالية جداً:** أسرع بـ 3x من `URLSearchParams` الموحدة[^1_2]
- ✅ **دعم كامل للفاصلات المخصصة:** يمكنك استخدام أي فاصل تريده[^1_1]
- ✅ **ترميز مخصص:** يمكنك استبدال دوال الترميز والفك الكامل[^1_1]
- ✅ **استقرار عالي:** API مستقرة وموثوقة منذ سنوات[^1_1]
- ✅ **خفيفة الوزن:** حجم صغير وبدون تبعيات[^1_4]

**العيوب والقيود:**

- ❌ **API موروثة:** موصوفة بـ "Legacy" في التوثيق الرسمي[^1_2][^1_1]
- ❌ **عدم التوافق مع المتصفح:** لا تعمل في JavaScript للمتصفحات[^1_2]
- ❌ **قيود النوع:** لا تدعم الكائنات المتداخلة بشكل مباشر[^1_1]
- ❌ **محدودية maxKeys:** الحد الافتراضي 1000 مفتاح[^1_1]
- ❌ **عدم الالتزام بمعايير WHATWG:** مختلفة عن المعايير الويب[^1_2]

---

## أفضل الممارسات والتوصيات

1. **استخدم querystring للأداء العالية:** إذا كنت تعالج ملايين الطلبات في تطبيق Node.js.[^1_2]
2. **استخدم URLSearchParams للتوافق:** إذا كنت تريد كود موحد بين الخادم والمتصفح.[^1_2]
3. **تحقق دائماً من نوع المفاتيح المكررة:** استخدم `Array.isArray()` قبل الوصول للقيم.[^1_9]
4. **استخدم `maxKeys: 0` للبيانات الضخمة:** عندما تتوقع أكثر من 1000 معامل.[^1_1]
5. **لا تعتمد على الخصائص الموروثة:** الكائن المرجع لا يرث من `Object`.[^1_1]
6. **صفّ المعاملات الفارغة:** قبل الاستدعاء إلى `stringify()`.[^1_5]

---

## الخلاصة والتوصيات النهائية

وحدة `querystring` في Node.js تبقى أداة قوية وسريعة جداً لمعالجة سلاسل الاستعلام في تطبيقات الخادم. على الرغم من تصنيفها كـ "Legacy API"، فإنها لا تزال الخيار الأمثل عندما تكون السرعة حرجة، أو عندما تحتاج إلى فاصلات مخصصة، أو تحتاج إلى ترميز متقدم. للكود الجديد والتطبيقات التي تحتاج توافق مع المتصفح، استخدم `URLSearchParams` من معيار WHATWG.[^1_7][^1_3][^1_2][^1_1]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88][^1_89][^1_90]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/querystring.md
[^1_2]: https://www.linkedin.com/pulse/how-migrate-from-querystring-urlsearchparams-nodejs-vladimír-gorej
[^1_3]: https://www.codemonday.com/blogs/node-js-alternative-after-url-parse-and-querystring-deprecated
[^1_4]: https://www.w3schools.com/nodejs/ref_querystring.asp
[^1_5]: https://devcrud.com/node-js-querystring-module-parsing-query-strings-made-simple-complete-guide/
[^1_6]: https://www.geeksforgeeks.org/node-js/node-js-querystring-stringify-method/
[^1_7]: https://vladimirgorej.com/blog/how-to-migrate-from-querystring-to-url-search-params-in-nodejs/
[^1_8]: https://nodejs.org/dist/latest-v6.x/docs/api/querystring.html
[^1_9]: https://www.geeksforgeeks.org/node-js/node-js-query-string-complete-reference/
[^1_10]: https://www.semanticscholar.org/paper/280d16a11c4c98fb11c284380ec2200568a4545f
[^1_11]: http://link.springer.com/10.1007/978-1-4302-6059-2
[^1_12]: http://transactions.ismir.net/articles/10.5334/tismir.111/
[^1_13]: https://www.semanticscholar.org/paper/959d24d21efe31eae401be73c1f7d8e636805655
[^1_14]: http://arxiv.org/pdf/1904.03801.pdf
[^1_15]: https://arxiv.org/pdf/2111.07238.pdf
[^1_16]: https://arxiv.org/pdf/1809.08319.pdf
[^1_17]: https://arxiv.org/pdf/2403.14940.pdf
[^1_18]: https://arxiv.org/pdf/2109.01002.pdf
[^1_19]: https://arxiv.org/pdf/2305.04032.pdf
[^1_20]: http://arxiv.org/pdf/2401.08595.pdf
[^1_21]: https://arxiv.org/pdf/2401.15843.pdf
[^1_22]: https://docs.deno.com/api/node/querystring/
[^1_23]: https://www.pabbly.com/tutorials/querystring/
[^1_24]: https://www.npmjs.com/package/query-string
[^1_25]: https://www.jsdocs.io/package/query-string
[^1_26]: https://nodejs.org/download/release/v4.3.0/docs/api/querystring.html
[^1_27]: https://billjs.github.io/query-string/
[^1_28]: https://stackoverflow.com/questions/17983532/overriding-node-js-querystring-escape-within-a-single-module
[^1_29]: https://wiki.hsoub.com/Node.js/querystring
[^1_30]: https://www.geeksforgeeks.org/node-js/node-js-query-string/
[^1_31]: https://www.fastly.com/documentation/reference/vcl/functions/query-string/
[^1_32]: https://bun.com/reference/node/querystring
[^1_33]: https://stackoverflow.com/questions/29136374/what-the-difference-between-qs-and-querystring
[^1_34]: https://stackoverflow.com/questions/16903476/node-js-http-get-request-with-query-string-parameters
[^1_35]: https://github.com/sindresorhus/query-string?tab=readme-ov-file
[^1_36]: https://coreui.io/answers/how-to-use-querystring-in-nodejs/
[^1_37]: https://www.tandfonline.com/doi/full/10.1080/17446651.2024.2363540
[^1_38]: https://egarp.lt/index.php/LUMIN/article/view/61
[^1_39]: https://ieeexplore.ieee.org/document/10633444/
[^1_40]: https://ai.jmir.org/2023/1/e49023
[^1_41]: https://www.granthaalayahpublication.org/journals/granthaalayah/article/view/5713
[^1_42]: https://www.cambridge.org/core/product/identifier/S0899823X25000765/type/journal_article
[^1_43]: https://ijsra.net/node/7864
[^1_44]: https://journals.lww.com/10.4103/ijves.ijves_37_23
[^1_45]: https://ieeexplore.ieee.org/document/10911386/
[^1_46]: https://dl.acm.org/doi/10.1145/3661167.3661207
[^1_47]: https://peerj.com/articles/cs-1421
[^1_48]: https://arxiv.org/pdf/1710.00454.pdf
[^1_49]: https://arxiv.org/html/2409.04667
[^1_50]: https://www.techrxiv.org/articles/preprint/Handling_Complex_Queries_Using_Query_Trees/14845212/files/28582164.pdf
[^1_51]: https://arxiv.org/pdf/1505.05187.pdf
[^1_52]: https://arxiv.org/pdf/2304.00472.pdf
[^1_53]: https://arxiv.org/pdf/1607.04452.pdf
[^1_54]: https://www.aclweb.org/anthology/D19-1263.pdf
[^1_55]: https://stackoverflow.com/questions/316781/how-to-build-query-string-with-javascript
[^1_56]: https://robertmarshall.dev/blog/migrating-from-query-string-to-urlsearchparams/
[^1_57]: https://developer.mozilla.org/ar/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
[^1_58]: https://www.linkedin.com/pulse/url-encoderdecoder-master-encoding-web-development-vikas-kapse-xn1nf
[^1_59]: https://www.claravine.com/a-query-on-using-query-strings-parameters/
[^1_60]: https://stackoverflow.com/questions/59889140/different-output-from-encodeuricomponent-vs-urlsearchparams
[^1_61]: https://gsnedders.html5.org/yui/yui/api/querystring-stringify-simple.js.html
[^1_62]: https://nodejs.org/api/querystring.html
[^1_63]: https://github.com/auth0/node-samlp/issues/137
[^1_64]: https://www.youtube.com/watch?v=BQy1EdUCq-o
[^1_65]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[^1_66]: https://nodejs.org/download/rc/v8.12.0-rc.2/docs/api/deprecations.html
[^1_67]: https://dev.to/captainsafia/node-module-deep-dive-querystring-ea9
[^1_68]: https://stackoverflow.com/questions/39266970/what-is-the-difference-between-url-parameters-and-query-strings
[^1_69]: https://arxiv.org/pdf/2308.08667.pdf
[^1_70]: https://dl.acm.org/doi/pdf/10.1145/3656460
[^1_71]: http://arxiv.org/pdf/2311.17620.pdf
[^1_72]: https://arxiv.org/pdf/2107.10164.pdf
[^1_73]: http://arxiv.org/pdf/1704.07887.pdf
[^1_74]: https://arxiv.org/pdf/2101.00756.pdf
[^1_75]: http://arxiv.org/pdf/2401.07053.pdf
[^1_76]: https://www.garysieling.com/blog/handling-duplicate-query-string-arguments-express-js/
[^1_77]: https://stackoverflow.com/questions/1714786/query-string-encoding-of-a-javascript-object
[^1_78]: https://www.reddit.com/r/PHPhelp/comments/tpu7dn/checking_for_multiple_url_parameters_with_same_key/
[^1_79]: https://www.geeksforgeeks.org/javascript/how-to-encode-and-decode-a-url-in-javascript/
[^1_80]: https://community.postman.com/t/using-duplicate-query-param-name-to-select-different-values/54523
[^1_81]: https://www.tinybird.co/blog/extract-query-parameter-clickhouse
[^1_82]: https://stackoverflow.com/questions/73236103/what-to-use-instead-of-querystring-nodequerystring-or-urlsearchparams-for
[^1_83]: https://stackoverflow.com/questions/17415169/how-to-build-webclient-querystring-with-duplicate-keys
[^1_84]: https://www.thecodingforums.com/threads/query-string-encoding-decoding.74587/
[^1_85]: https://nodejs.org/api/deprecations.html
[^1_86]: https://github.com/github/fetch/issues/697
[^1_87]: https://github.com/khalidsalomao/simple-query-string
[^1_88]: https://github.com/nodejs/node/issues/44911
[^1_89]: https://dev.to/hardik_b2d8f0bca/url-encoderdecoder-master-url-encoding-for-web-development-47j0
[^1_90]: https://github.com/Moya/Moya/issues/528

# وثائق Node.js زليب (zlib) - دليل عميق شامل

## نظرة عامة على الموديول

**زليب** (zlib) هو موديول Node.js متخصص في **ضغط البيانات وفك الضغط** (Compression and Decompression) باستخدام ثلاث خوارزميات رئيسية حديثة: **Gzip** (الضغط مع معلومات رأس), **Deflate/Inflate** (الضغط الأساسي بدون رأس), **Brotli** (خوارزمية Google المتقدمة), و **Zstandard/Zstd** (خوارزمية فيسبوك الحديثة). يُبنى هذا الموديول على **Node.js Streams API**، مما يسمح بمعالجة البيانات بكفاءة عالية حتى للملفات الضخمة دون تحميل البيانات بالكامل في الذاكرة.[^1_1][^1_2]

**متى تستخدم هذا الموديول بدلاً من المكتبات الخارجية؟**

- يأتي زليب **مع Node.js افتراضياً** - لا تحتاج تثبيت مكتبات إضافية.[^1_1]
- يدعم **أربع خوارزميات** (Gzip, Deflate, Brotli, Zstd) بواجهة موحدة.[^1_1]
- يستخدم **libuv thread pool** للعمليات الثقيلة، مما يحافظ على عدم توقف الحلقة الرئيسية (Event Loop).[^1_3]
- متوافق تماماً مع **HTTP Content-Encoding** (للاستجابات المضغوطة).[^1_1]
- يوفر **واجهتان**: Streaming (للبيانات الكبيرة) و Convenience methods (للبيانات الصغيرة).[^1_1]

المكتبات الخارجية مفيدة فقط إذا احتجت خوارزميات متخصصة أو أداء مخصص لحالات نادرة جداً.

## الخوارزميات الأساسية والمقارنات

قبل الدخول للتفاصيل، من المهم فهم الفروقات بين الخوارزميات الأربع:

| الخوارزمية    | سرعة الضغط          | نسبة الضغط               | سرعة فك الضغط       | حالة الاستخدام الأمثل                         |
| :------------ | :------------------ | :----------------------- | :------------------ | :-------------------------------------------- |
| **Gzip**      | ⭐⭐⭐ سريعة        | ⭐⭐⭐ جيدة (2.56:1)     | ⭐⭐⭐⭐ سريعة جداً | الويب (التوافق الأقصى)، الأرشيفات العامة      |
| **Deflate**   | ⭐⭐⭐ سريعة        | ⭐⭐⭐ جيدة              | ⭐⭐⭐⭐ سريعة جداً | الضغط الأساسي بدون رأس (header-less)          |
| **Brotli**    | ⭐ بطيئة جداً       | ⭐⭐⭐⭐ ممتازة (3.08:1) | ⭐⭐⭐⭐ سريعة جداً | المحتوى الثابت المخزن مسبقاً (CSS, JS, Fonts) |
| **Zstandard** | ⭐⭐⭐⭐ سريعة جداً | ⭐⭐⭐⭐ ممتازة (2.86:1) | ⭐⭐⭐⭐ سريعة جداً | المحتوى الديناميكي، الوقت الفعلي (JSON, HTML) |

**مثال مقارنة حقيقي**: لملف JSON بحجم 523 MB:[^1_4]

- **Zstandard**: 0.56 ثانية ضغط، نسبة ضغط 81.7%
- **Brotli**: 759 ثانية ضغط، نسبة ضغط 87.8% (800x أبطأ!)
- **Gzip**: 5.67 ثانية ضغط، نسبة ضغط 81.7%

**الخلاصة**: استخدم **Zstandard** للسرعة والتوازن، **Brotli** للمحتوى الثابت المخزن، **Gzip** للتوافق القديم.[^1_5]

![Node.js zlib Compression Architecture and Data Flow](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/bb6e792a678c9e841275c2acbbc9f815/6b20b210-207f-4e26-ac69-932d5a8e3123/233b3c1a.png)

Node.js zlib Compression Architecture and Data Flow

## الواجهات الرئيسية (API)

### الواجهة الأولى: Streaming (التدفق)

استخدم فئات التدفق (`createGzip`, `createDeflate`, إلخ) **عندما تتعامل مع بيانات كبيرة أو مستمرة** (ملفات، اتصالات شبكية):

```javascript
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";
import { pipeline } from "node:stream";

// ضغط ملف JSON كبير باستخدام Gzip
const source = createReadStream("large-data.json");
const destination = createWriteStream("large-data.json.gz");
const gzip = createGzip({ level: 6 }); // مستوى ضغط متوسط

pipeline(source, gzip, destination, (err) => {
  if (err) {
    console.error("خطأ في الضغط:", err);
  } else {
    console.log("تم الضغط بنجاح!");
  }
});
```

**المميزات**:

- ✅ معالجة البيانات **على دفعات** (chunks) - توفير ذاكرة ضخم
- ✅ يعمل مع `pipeline()` أو الربط المباشر (piping)
- ✅ دعم `flush()` و `reset()` و `params()` أثناء العملية

### الواجهة الثانية: Convenience Methods (طريقة سريعة)

استخدم الدوال `gzip()`, `deflate()`, `brotliCompress()` إلخ **للبيانات الصغيرة فقط** (أقل من بضعة MB):

```javascript
import { gzip } from "node:zlib";
import { promisify } from "node:util";

const gzipPromise = promisify(gzip);

const payload = JSON.stringify({
  user: "Ahmed",
  email: "ahmed@example.com",
  timestamp: Date.now(),
});

// طريقة 1: Callback
gzip(payload, (err, compressed) => {
  if (err) throw err;
  console.log(`الحجم الأصلي: ${payload.length}, المضغوط: ${compressed.length}`);
});

// طريقة 2: Promise (أنظف)
const compressed = await gzipPromise(payload);
console.log(`تم الضغط من ${payload.length} إلى ${compressed.length}`);
```

**المميزات**:

- ✅ واجهة **بسيطة وسريعة**
- ✅ كود أقل
- ❌ تحميل **البيانات كاملة في الذاكرة** (خطير للملفات الكبيرة)

---

## التفاصيل الكاملة لكل دالة/فئة

### 1. فئات التدفق (Stream Classes)

#### `zlib.createGzip([options])`

**الوصف**: ينشئ تدفق (Transform stream) يضغط البيانات باستخدام **Gzip** (معيار HTTP الأكثر شيوعاً).

**جدول المعاملات**:

| المعامل              | النوع   | مطلوب/اختياري | الوصف                                                       |
| :------------------- | :------ | :------------ | :---------------------------------------------------------- |
| `options`            | Object  | اختياري       | خيارات الضغط (انظر فئة `Options` أدناه)                     |
| `options.level`      | 0-9     | اختياري       | مستوى الضغط: 0=بلا ضغط، 1=أسرع، 6=افتراضي، 9=أفضل ضغط       |
| `options.memLevel`   | 1-9     | اختياري       | استخدام الذاكرة: 1=قليل (256K)، 8=افتراضي (256K)، 9=الأقصى  |
| `options.windowBits` | 9-15    | اختياري       | حجم نافذة البحث: أكبر = ضغط أفضل + ذاكرة أكثر (افتراضي: 15) |
| `options.chunkSize`  | integer | اختياري       | حجم القطع المعالجة (افتراضي: 16KB)                          |

**القيمة المرجعة**: `Gzip` stream object يرث من `Transform`.

**مثال واقعي: ضغط استجابة HTTP**:

```javascript
import zlib from "node:zlib";
import http from "node:http";
import fs from "node:fs";
import { pipeline } from "node:stream";

http
  .createServer((request, response) => {
    const acceptEncoding = request.headers["accept-encoding"] || "";

    // قراءة ملف HTML كبير
    const htmlFile = fs.createReadStream("index.html");

    if (acceptEncoding.includes("gzip")) {
      response.writeHead(200, {
        "Content-Encoding": "gzip",
        Vary: "Accept-Encoding",
      });
      // ربط الملف -> ضغط Gzip -> الاستجابة
      pipeline(htmlFile, zlib.createGzip({ level: 6 }), response, (err) => {
        if (err) {
          console.error("خطأ في الضغط:", err);
          response.end();
        }
      });
    } else {
      // بدون ضغط للمتصفحات القديمة
      response.writeHead(200);
      pipeline(htmlFile, response, (err) => {
        if (err) console.error(err);
      });
    }
  })
  .listen(3000);
```

**الأخطاء الشائعة**:

- ❌ استخدام `createGzip()` لملفات ضخمة جداً (>500MB) **بدون تدفق** - ستنفد الذاكرة
- ❌ عدم معالجة أخطاء التدفق - قد يفقد البيانات بصمت
- ❌ مستوى ضغط 9 للمحتوى الديناميكي - سيبطئ السيرفر بشكل حاد

---

#### `zlib.createDeflate([options])`

**الوصف**: نفس `createGzip` لكن **بدون رأس Gzip** (raw DEFLATE). تُستخدم في البروتوكولات المخصصة.

**المعاملات**: متطابقة مع `createGzip`.

**مثال**: في بروتوكول WebSocket مع ضغط:

```javascript
import zlib from "node:zlib";

const deflateStream = zlib.createDeflate({
  level: zlib.constants.Z_DEFAULT_COMPRESSION,
});

// في WebSocket extension negotiation
websocket.on("message", (data) => {
  deflateStream.write(data);
  deflateStream.flush(zlib.constants.Z_SYNC_FLUSH, () => {
    // أرسل البيانات المضغوطة
  });
});
```

---

#### `zlib.createInflate([options])`

**الوصف**: فك ضغط بيانات **DEFLATE raw**.

**مثال**: استقبال بيانات مضغوطة من WebSocket:

```javascript
import zlib from "node:zlib";

const inflateStream = zlib.createInflate();

websocket.on("message", (compressedData) => {
  inflateStream.write(compressedData);
});

inflateStream.on("data", (decompressed) => {
  console.log("البيانات الأصلية:", decompressed.toString());
});
```

---

#### `zlib.createGunzip([options])`

**الوصف**: فك ضغط بيانات **Gzip** (الأكثر استخداماً).

**مثال عملي: تحميل ملف مضغوط من الإنترنت**:

```javascript
import https from "node:https";
import zlib from "node:zlib";
import fs from "node:fs";
import { pipeline } from "node:stream";

https.get("https://api.example.com/data.json.gz", (response) => {
  const output = fs.createWriteStream("data.json");
  const gunzip = zlib.createGunzip();

  pipeline(response, gunzip, output, (err) => {
    if (err) {
      console.error("فشل التحميل:", err);
    } else {
      console.log("تم فك الضغط بنجاح");
    }
  });
});
```

---

#### `zlib.createUnzip([options])`

**الوصف**: **كاشف ذكي** - يتعرف تلقائياً على نوع الضغط (Gzip أو Deflate) من الرأس ويختار الفاك المناسب.

**مثال**: استجابة HTTP قد تكون بأي نوع ضغط:

```javascript
import http from "node:http";
import zlib from "node:zlib";
import { pipeline } from "node:stream";
import fs from "node:fs";

http.get("https://example.com/data", (response) => {
  const file = fs.createWriteStream("data.txt");

  // Unzip سيتعرف على النوع تلقائياً
  pipeline(
    response,
    zlib.createUnzip(), // بدون الحاجة للتحقق من Content-Encoding
    file,
    (err) => console.error(err)
  );
});
```

---

#### `zlib.createBrotliCompress([options])`

**الوصف**: ضغط باستخدام **Brotli** (خوارزمية Google الحديثة، أفضل نسبة ضغط).

**معاملات خاصة بـ Brotli**:

| المعامل                        | النوع   | الافتراضي | الوصف                               |
| :----------------------------- | :------ | :-------- | :---------------------------------- |
| `params`                       | Object  | `{}`      | خيارات Brotli المتقدمة              |
| `params[BROTLI_PARAM_MODE]`    | 0, 1, 2 | 0         | 0=عام، 1=نصي، 2=الخطوط              |
| `params[BROTLI_PARAM_QUALITY]` | 0-11    | 11        | جودة الضغط (أعلى = أبطأ)            |
| `params[BROTLI_PARAM_LGWIN]`   | 10-24   | 22        | حجم النافذة (ذاكرة أكثر = ضغط أفضل) |

**مثال واقعي: ضغط ملفات CSS و JavaScript للويب**:

```javascript
import zlib from "node:zlib";
import fs from "node:fs";
import { pipeline } from "node:stream";

const cssFile = fs.createReadStream("styles.css");
const output = fs.createWriteStream("styles.css.br");

// إعدادات Brotli محسّنة للنصوص
const brotli = zlib.createBrotliCompress({
  chunkSize: 32 * 1024, // معالجة قطع أكبر = أسرع
  params: {
    [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
    [zlib.constants.BROTLI_PARAM_QUALITY]: 11, // أفضل جودة
    [zlib.constants.BROTLI_PARAM_SIZE_HINT]: fs.statSync("styles.css").size,
  },
});

pipeline(cssFile, brotli, output, (err) => {
  if (err) throw err;
  console.log("تم ضغط CSS بـ Brotli");
});
```

**الأخطاء الشابعة**:

- ❌ استخدام Brotli **quality 11** للمحتوى الديناميكي - سيأخذ **قرون** (أحرفياً!)
- ❌ نسيان `BROTLI_PARAM_SIZE_HINT` - فقدان فرصة التحسين
- ✅ استخدم `BROTLI_PARAM_QUALITY` من 4-6 للمحتوى الديناميكي، 9-11 للمخزن مسبقاً فقط

---

#### `zlib.createBrotliDecompress([options])`

**الوصف**: فك ضغط Brotli.

```javascript
import zlib from "node:zlib";
import fs from "node:fs";
import { pipeline } from "node:stream";

const compressed = fs.createReadStream("data.br");
const output = fs.createWriteStream("data.txt");

pipeline(compressed, zlib.createBrotliDecompress(), output, (err) => {
  if (err) throw err;
  console.log("تم فك ضغط Brotli");
});
```

---

#### `zlib.createZstdCompress([options])` و `zlib.createZstdDecompress([options])`

**الوصف**: ضغط وفك ضغط **Zstandard** (خوارزمية Facebook). أسرع من Brotli مع نسبة ضغط قريبة جداً.

**معاملات خاصة**:

```javascript
const zstd = zlib.createZstdCompress({
  params: {
    [zlib.constants.ZSTD_c_compressionLevel]: 10, // المستوى 1-22
    [zlib.constants.ZSTD_c_checksumFlag]: 1, // تفعيل التحقق من الصحة
  },
});
```

**مثال: ضغط JSON الديناميكي على الويب**:

```javascript
import zlib from 'node:zlib';
import http from 'node:http';
import { pipeline } from 'node:stream';

http.createServer((req, res) => {
  const data = JSON.stringify({ users: [...] });

  if (req.headers['accept-encoding']?.includes('zstd')) {
    res.writeHead(200, {
      'Content-Encoding': 'zstd',
      'Content-Type': 'application/json'
    });

    const zstd = zlib.createZstdCompress({
      params: {
        [zlib.constants.ZSTD_c_compressionLevel]: 3 // سريع
      }
    });

    res.write(data);
    res.pipe(zstd);
  } else {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.write(data);
  }
}).listen(3000);
```

---

### 2. دوال الضغط السريعة (Convenience Methods)

#### `zlib.gzip(buffer[, options], callback)` / `zlib.gzipSync(buffer[, options])`

**الوصف**: ضغط البيانات **كاملة** باستخدام Gzip مرة واحدة.

**معاملات**:

| المعامل    | النوع            | مطلوب           | الوصف                          |
| :--------- | :--------------- | :-------------- | :----------------------------- |
| `buffer`   | Buffer \| String | نعم             | البيانات المراد ضغطها          |
| `options`  | Object           | لا              | خيارات مثل `level`, `memLevel` |
| `callback` | Function         | نعم (async فقط) | `(err, result) => {}`          |

**مثال: حفظ بيانات JSON مضغوطة**:

```javascript
import zlib from "node:zlib";
import fs from "node:fs";

const userData = JSON.stringify({
  id: 12345,
  name: "محمد علي",
  email: "mohammed@example.com",
  createdAt: new Date(),
});

// طريقة غير المتزامنة (Async)
zlib.gzip(userData, (err, compressed) => {
  if (err) throw err;
  fs.writeFile("user-data.json.gz", compressed, (err) => {
    if (err) throw err;
    console.log(`تم حفظ البيانات المضغوطة (${compressed.length} بايت)`);
  });
});

// أو المتزامنة (Sync) - بحذر!
try {
  const compressed = zlib.gzipSync(userData);
  fs.writeFileSync("user-data.json.gz", compressed);
  console.log("تم الضغط والحفظ");
} catch (err) {
  console.error(err);
}
```

**الأخطاء الشائعة**:

- ❌ استخدام `gzipSync` على البيانات الكبيرة - **سيوقف البرنامج** (blocking)
- ❌ عدم التعامل مع الخطأ - الفشل الصامت
- ✅ استخدم Async دائماً للبيانات الكبيرة

---

#### دوال Deflate و InflateRaw

```javascript
// ضغط Deflate raw (بدون رأس)
zlib.deflate(data, (err, compressed) => { ... });
zlib.deflateSync(data);

// فك Deflate raw
zlib.inflate(compressedData, (err, original) => { ... });
zlib.inflateSync(compressedData);

// فك Deflate raw (يتوقع بيانات بدون رأس تماماً)
zlib.inflateRaw(rawData, (err, original) => { ... });
```

---

#### `zlib.brotliCompress()` و `zlib.brotliDecompress()`

```javascript
zlib.brotliCompress(
  cssContent,
  {
    params: {
      [zlib.constants.BROTLI_PARAM_QUALITY]: 11,
    },
  },
  (err, compressed) => {
    // حفظ للاستخدام لاحقاً
  }
);
```

---

#### `zlib.zstdCompress()` و `zlib.zstdDecompress()`

```javascript
// ضغط سريع مع تحقق من الصحة
zlib.zstdCompress(
  jsonData,
  {
    params: {
      [zlib.constants.ZSTD_c_checksumFlag]: 1,
    },
  },
  (err, result) => {
    console.log("تم بنجاح");
  }
);
```

---

#### `zlib.unzip(buffer[, options], callback)`

**الوصف**: فك ضغط **ذكي** - يكتشف نوع الضغط (Gzip/Deflate) تلقائياً.

```javascript
import zlib from "node:zlib";
import fs from "node:fs";

const compressedBuffer = fs.readFileSync("mystery.dat");

zlib.unzip(compressedBuffer, (err, original) => {
  if (err) {
    console.error("نوع ضغط غير معروف أو بيانات تالفة");
    return;
  }
  console.log("البيانات الأصلية:", original.toString());
});
```

---

### 3. دوال معالجة متقدمة (Stream Methods)

#### `stream.flush([kind,] callback)`

**الوصف**: **إفراغ القائمة المؤجلة** - إخراج كل البيانات المخزنة مؤقتاً من الضاغط دون انتظار نهاية البيانات.

**الحالات الحقيقية**: مفيدة جداً للاستجابات الحية (Live Responses) مثل:

- سجلات الخادم المباشرة
- بث البيانات الفعلي للعملاء
- WebSocket بضغط

```javascript
import zlib from "node:zlib";
import http from "node:http";

http
  .createServer((req, res) => {
    res.writeHead(200, { "Content-Encoding": "gzip" });
    const gzip = zlib.createGzip();
    gzip.pipe(res);

    // محاكاة رسائل قادمة تدريجياً
    setInterval(() => {
      const message = `[${new Date().toISOString()}] حدث جديد\n`;
      gzip.write(message);

      // إفراغ الفوري بدلاً من انتظار 16KB من البيانات
      gzip.flush(zlib.constants.Z_SYNC_FLUSH);
    }, 1000);
  })
  .listen(3000);
```

**قيم الإفراغ**:

- `Z_NO_FLUSH`: لا تفرغ (افتراضي - انتظر المزيد من البيانات)
- `Z_PARTIAL_FLUSH`: أفرغ الجزء الممكن
- `Z_SYNC_FLUSH`: أفرغ وحاذِ للحد الأدنى من البايت
- `Z_FULL_FLUSH`: أفرغ الكل وأعد تعيين الحالة
- `Z_FINISH`: أفرغ الكل وأنهِ الضغط (افتراضي للنهاية)

---

#### `stream.params(level, strategy, callback)`

**الوصف**: **تغيير ديناميكي** لمستوى الضغط والإستراتيجية أثناء الضغط (فقط Deflate/Gzip).

```javascript
import zlib from "node:zlib";
import fs from "node:fs";

const input = fs.createReadStream("huge-file.bin");
const output = fs.createWriteStream("output.gz");
const gzip = zlib.createGzip({ level: 1 }); // ابدأ بسرعة

input.pipe(gzip).pipe(output);

// بعد 5 ثوان، زد جودة الضغط
setTimeout(() => {
  gzip.params(9, zlib.constants.Z_DEFAULT_STRATEGY, (err) => {
    if (err) throw err;
    console.log("تم تحسين الضغط");
  });
}, 5000);
```

---

#### `stream.reset()`

**الوصف**: **إعادة تعيين** الحالة الداخلية (فقط Deflate/Inflate). يُستخدم نادراً جداً.

```javascript
import zlib from "node:zlib";

const deflate = zlib.createDeflate();

// بعد فك ضغط كامل، أعد التعيين لضغط جديد
deflate.on("finish", () => {
  deflate.reset();
  // يمكن الآن ضغط بيانات جديدة
});
```

---

#### `stream.close([callback])`

**الوصف**: **إغلاق** معالج الضاغط الداخلي (في لغة C).

```javascript
const gzip = zlib.createGzip();

gzip.on("finish", () => {
  gzip.close(() => {
    console.log("تم الإغلاق والتنظيف");
  });
});
```

---

#### `stream.bytesWritten`

**الوصف**: عدد البايتات **المكتوبة** للضاغط (قبل الضغط).

```javascript
const gzip = zlib.createGzip();
let totalWritten = 0;

gzip.on("data", () => {
  console.log(`تم كتابة ${gzip.bytesWritten} بايت للضاغط`);
  console.log(
    `نسبة الضغط المؤقتة: ${((gzip.bytesWritten / totalWritten) * 100).toFixed(
      2
    )}%`
  );
});
```

---

### 4. دالة CRC32 (Checksum)

#### `zlib.crc32(data[, value])`

**الوصف**: حساب **CRC32** للتحقق من سلامة البيانات (ليس تشفير).

```javascript
import zlib from "node:zlib";

const text = "مرحبا بالعالم";

let crc = zlib.crc32(text);
console.log(`CRC32 الأول: ${crc}`);

// يمكن حساب CRC لأجزاء متعددة
const part2 = " من جديد";
crc = zlib.crc32(part2, crc);
console.log(`CRC32 النهائي: ${crc}`);

// استخدام: التحقق من تطابق الملفات
const original = zlib.crc32("file-content");
const transmitted = zlib.crc32("file-content"); // نفس النص
console.log(`متطابقة: ${original === transmitted}`);
```

---

## ثوابت وخيارات الضغط (Constants \& Tuning)

### مستويات الضغط

```javascript
// Zlib/Deflate مستويات (0-9)
zlib.constants.Z_NO_COMPRESSION; // 0 - بدون ضغط
zlib.constants.Z_BEST_SPEED; // 1 - الأسرع
zlib.constants.Z_DEFAULT_COMPRESSION; // 6 - التوازن (الافتراضي)
zlib.constants.Z_BEST_COMPRESSION; // 9 - الأفضل ضغطاً (بطيء جداً)

// Brotli مستويات (0-11)
zlib.constants.BROTLI_DEFAULT_QUALITY; // 11 - جودة عالية جداً
zlib.constants.BROTLI_MIN_QUALITY; // 0 - أسرع
```

### إستراتيجيات الضغط (Compression Strategies)

```javascript
zlib.constants.Z_DEFAULT_STRATEGY; // 0 - عام (الافتراضي)
zlib.constants.Z_FILTERED; // 1 - للبيانات المصفاة (صور)
zlib.constants.Z_HUFFMAN_ONLY; // 2 - Huffman فقط (سريع جداً)
zlib.constants.Z_RLE; // 3 - Run-Length Encoding (بيانات متكررة)
zlib.constants.Z_FIXED; // 4 - جداول Huffman ثابتة
```

### ضبط الذاكرة والسرعة

```javascript
// الحساب: (1 << (windowBits + 2)) + (1 << (memLevel + 9))
// مثال: windowBits=15 + memLevel=8 = 128K + 128K = 256K

const options = {
  level: 6, // التوازن
  windowBits: 15, // 32K نافذة (افتراضي)
  memLevel: 8, // 128K للديناميكي (افتراضي)
  chunkSize: 16 * 1024, // 16KB معالجة لكل جزء
};

// لمحيط محدود (IoT):
const iotOptions = {
  level: 1, // أسرع
  windowBits: 9, // 512B نافذة فقط
  memLevel: 1, // 256B فقط!
  chunkSize: 4 * 1024, // 4KB معالجة
  // النتيجة: ذاكرة أقل لكن ضغط أسوأ
};

// للخوادم الضخمة (CDN):
const cdnOptions = {
  level: 9, // أفضل ضغط
  windowBits: 15, // أكبر نافذة
  memLevel: 9, // أقصى ذاكرة
  chunkSize: 64 * 1024, // معالجة كبيرة = أسرع
};
```

---

## الأخطاء الشائعة والحلول

### 1. تجزئة الذاكرة (Memory Fragmentation)

**المشكلة**: إنشاء عدد كبير من الضاغطات في حلقة يسبب **تجزئة الذاكرة** حتى لو لم تحتفظ بها.[^1_1][^1_6]

```javascript
// ❌ خطر جداً - 30,000 ضاغط متزامن
for (let i = 0; i < 30000; ++i) {
  zlib.deflate(payload, (err, buffer) => {});
}
```

**الحل**: **استخدم Object Pool أو محدود العدد المتزامن**:

```javascript
// ✅ محدود إلى 10 فقط متزامن
const pLimit = require("p-limit");
const limit = pLimit(10);

const promises = items.map((item) =>
  limit(
    () =>
      new Promise((resolve, reject) => {
        zlib.gzip(item, (err, result) => {
          if (err) reject(err);
          else resolve(result);
        });
      })
  )
);

await Promise.all(promises);
```

---

### 2. مشاكل Thread Pool

**المشكلة**: العمليات ستنتظر في الطابور إذا كان thread pool ممتلء.[^1_3]

```javascript
// افتراضياً: 4 threads فقط
// لو بدأت 100 gzip معاً، 96 منها ستنتظر!

// الحل: زيادة Thread Pool Size
process.env.UV_THREADPOOL_SIZE = 8;
import zlib from "node:zlib";
// الآن 8 threads متاح
```

---

### 3. بيانات غير كاملة (Truncated Data)

**المشكلة**: فك ضغط بيانات **ناقصة** يرفع خطأ افتراضياً.

```javascript
// ❌ خطأ: بيانات ناقصة
const truncated = Buffer.from("eJzT0yMA", "base64"); // ناقصة
zlib.gunzip(truncated, (err, result) => {
  console.log(err); // Z_DATA_ERROR
});

// ✅ الحل: استخدم finishFlush = Z_SYNC_FLUSH
zlib.gunzip(
  truncated,
  {
    finishFlush: zlib.constants.Z_SYNC_FLUSH,
  },
  (err, result) => {
    if (result) console.log("فك جزئي:", result.toString());
  }
);
```

---

## حالات الاستخدام الفعلية والتوصيات

### 📡 **سيرفر Web API**

```javascript
import zlib from "node:zlib";
import http from "node:http";
import { pipeline } from "node:stream";

http
  .createServer((req, res) => {
    const encoding = req.headers["accept-encoding"] || "";
    const data = JSON.stringify({
      /* بيانات API */
    });

    res.setHeader("Vary", "Accept-Encoding");

    if (encoding.includes("zstd")) {
      // المحتوى الديناميكي: Zstandard هو الأفضل
      res.writeHead(200, { "Content-Encoding": "zstd" });
      const zstd = zlib.createZstdCompress({
        params: {
          [zlib.constants.ZSTD_c_compressionLevel]: 3,
        },
      });
      return pipeline([res, zstd], res, (err) => err && console.error(err));
    } else if (encoding.includes("gzip")) {
      res.writeHead(200, { "Content-Encoding": "gzip" });
      return pipeline(
        [res, zlib.createGzip({ level: 6 })],
        res,
        (err) => err && console.error(err)
      );
    }

    res.writeHead(200);
    res.end(data);
  })
  .listen(3000);
```

---

### 🖼️ **معالجة الملفات الضخمة**

```javascript
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";
import { pipeline } from "node:stream";

async function compressLargeFile(input, output) {
  return new Promise((resolve, reject) => {
    const source = createReadStream(input);
    const dest = createWriteStream(output);
    const gzip = createGzip({
      level: 6,
      chunkSize: 32 * 1024, // معالجة قطع كبيرة
    });

    pipeline(source, gzip, dest, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}

compressLargeFile("large-dataset.json", "backup.json.gz");
```

---

### 💾 **نسخ احتياطية ضغوطة**

```javascript
import zlib from "node:zlib";
import crypto from "node:crypto";
import fs from "node:fs";

async function createBackup(data, password) {
  // 1. ضغط
  const compressed = await zlib.promises.gzip(data);

  // 2. تشفير (اختياري)
  const cipher = crypto.createCipher("aes-256-cbc", password);
  let encrypted = cipher.update(compressed);
  encrypted = Buffer.concat([encrypted, cipher.final()]);

  // 3. حفظ
  fs.writeFileSync(`backup-${Date.now()}.gz.enc`, encrypted);
}

// الملخص: Compress → Encrypt → Store
```

---

### 📊 **معالجة البيانات الفعلية (Real-time)**

```javascript
import { EventEmitter } from "node:events";
import zlib from "node:zlib";

class CompressedDataStream extends EventEmitter {
  constructor(algorithm = "gzip") {
    super();
    this.compressor =
      algorithm === "gzip"
        ? zlib.createGzip({ level: 1 }) // سريع
        : zlib.createZstdCompress();

    this.compressor.on("data", (chunk) => {
      this.emit("compressed", chunk);
    });
  }

  write(data) {
    this.compressor.write(data);
    this.compressor.flush(zlib.constants.Z_SYNC_FLUSH);
  }

  end() {
    this.compressor.end();
  }
}

const stream = new CompressedDataStream("zstd");
stream.on("compressed", (chunk) => {
  // أرسل البيانات المضغوطة فوراً
  websocket.send(chunk);
});
```

---

## المزايا والعيوب والمقارنات النهائية

### ✅ **مزايا zlib**

1. **مدمج في Node.js** - لا تثبيت إضافي[^1_1]
2. **أداء عالي جداً** - محسّن على مستوى C[^1_1]
3. **مرونة** - أربع خوارزميات بواجهة موحدة[^1_1]
4. **معايير ويب** - متوافق مع HTTP Content-Encoding[^1_1]
5. **Streaming أولاً** - معالجة آمنة للبيانات الضخمة[^1_1]

### ❌ **عيوب zlib**

1. **تجزئة الذاكرة** مع العمليات المتزامنة الكثيفة[^1_6][^1_1]
2. **Brotli بطيء جداً** للمحتوى الديناميكي[^1_4]
3. **Thread pool محدود** (4 threads افتراضياً)[^1_3]
4. **واجهة Callback قديمة** - يفضل Promise/Streams[^1_1]
5. **Zstd تجريبي فقط** (Stability: 1)[^1_1]

---

### جدول المقارنة النهائي

| الخاصية        | Gzip         | Deflate      | Brotli       | Zstandard      |
| :------------- | :----------- | :----------- | :----------- | :------------- |
| **سرعة الضغط** | ⭐⭐⭐ سريعة | ⭐⭐⭐ سريعة | ⭐ بطيء جداً | ⭐⭐⭐⭐ سريع  |
| **نسبة الضغط** | 2.56:1 متوسط | 2.5:1        | 3.08:1 ممتاز | 2.86:1 ممتاز   |
| **سرعة الفك**  | ⭐⭐⭐⭐     | ⭐⭐⭐⭐     | ⭐⭐⭐⭐     | ⭐⭐⭐⭐       |
| **التوافق**    | 99% متصفحات  | 95%          | 95.9%        | 80%            |
| **الحالة**     | Stable       | Stable       | Stable       | Experimental   |
| **أفضل لـ**    | ويب عام      | بروتوكولات   | محتوى ثابت   | محتوى ديناميكي |

---

## الخلاصة والتوصيات

1. **للويب الحديث**: استخدم **Zstandard** للديناميكي و **Brotli** للمخزن[^1_5]
2. **للتوافق الأقصى**: ابدأ بـ **Gzip** ثم انتقل لـ Zstandard[^1_5]
3. **للأداء العالية**: اجمع بين **Streaming** و **Thread Pool المخصص**[^1_3][^1_1]
4. **تجنب الأخطاء**: لا تنشئ آلاف الضاغطات معاً[^1_1]
5. **الضبط الدقيق**: اختبر المستويات والنوافذ على بيانات حقيقية[^1_7]

---

**مراجع**:
[^1_1] - Node.js Official Zlib Documentation | [^1_2] - UNB zlib Reference | [^1_4] - Speed Vitals Compression Comparison | [^1_5] - Cloudflare Zstandard Blog | [^1_3] - Thread Pool Documentation
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_8][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/zlib.md
[^1_2]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/zlib/
[^1_3]: https://shiftasia.com/community/understanding-the-thread-pool-and-libuv-in-node-js/
[^1_4]: https://speedvitals.com/blog/zstd-vs-brotli-vs-gzip/
[^1_5]: https://blog.cloudflare.com/new-standards/
[^1_6]: https://www.brand.dev/blog/preventing-memory-issues-in-node-js-sharp-a-journey
[^1_7]: https://nodejs.org/download/release/v9.0.0/docs/api/zlib.html
[^1_8]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_9]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_10]: http://transactions.ismir.net/articles/10.5334/tismir.111/
[^1_11]: https://www.semanticscholar.org/paper/959d24d21efe31eae401be73c1f7d8e636805655
[^1_12]: https://www.qeios.com/read/J1P9L9/pdf
[^1_13]: http://arxiv.org/pdf/2401.07053.pdf
[^1_14]: http://arxiv.org/pdf/2408.04230.pdf
[^1_15]: https://arxiv.org/pdf/1801.06144.pdf
[^1_16]: https://arxiv.org/pdf/2007.03305.pdf
[^1_17]: https://arxiv.org/pdf/2403.14940.pdf
[^1_18]: https://arxiv.org/pdf/1205.6363.pdf
[^1_19]: https://arxiv.org/pdf/2101.00756.pdf
[^1_20]: https://stackabuse.com/python-zlib-library-tutorial/
[^1_21]: https://developer.mozilla.org/en-US/docs/Web/API/DecompressionStream/DecompressionStream
[^1_22]: https://docs.micropython.org/en/latest/library/zlib.html
[^1_23]: https://www.reddit.com/r/csharp/comments/1nkbsm4/deflate_vs_zlib/
[^1_24]: https://www.w3schools.com/nodejs/nodejs_zlib.asp
[^1_25]: https://docs.python.org/3/library/zlib.html
[^1_26]: https://github.com/isaacs/minizlib
[^1_27]: https://millermedeiros.github.io/mdoc/examples/node_api/doc/zlib.html
[^1_28]: https://stackoverflow.com/questions/1316357/zlib-decompression-in-python
[^1_29]: https://ruby-doc.org/stdlib-2.6.8/libdoc/zlib/rdoc/Zlib.html
[^1_30]: https://docs.deno.com/api/node/zlib/
[^1_31]: https://www.geeksforgeeks.org/python/zlib-decompresss-in-python/
[^1_32]: https://stackoverflow.com/questions/44343371/data-compression-libraries-brotli-vs-zlib
[^1_33]: https://nodejs.org/api/zlib.html
[^1_34]: https://en.wikipedia.org/wiki/Zlib
[^1_35]: https://www.geeksforgeeks.org/node-js/node-js-zlib-complete-reference/
[^1_36]: https://nodejs.org/download/release/v0.10.46/docs/api/zlib.html
[^1_37]: https://github.com/kevin-cantwell/zlib
[^1_38]: http://journal-jceees.com/Publication/DisplayArticle/29665
[^1_39]: http://dergipark.org.tr/en/doi/10.46810/tdfd.1425959
[^1_40]: http://biorxiv.org/lookup/doi/10.1101/642553
[^1_41]: https://dx.plos.org/10.1371/journal.pone.0314691
[^1_42]: https://www.semanticscholar.org/paper/bfe4e5b192a2380cdfd487005a58ad683ef58630
[^1_43]: https://aocs.onlinelibrary.wiley.com/doi/10.1007/BF02672207
[^1_44]: https://arxiv.org/pdf/1606.00519.pdf
[^1_45]: https://academic.oup.com/bioinformatics/article-pdf/35/19/3826/30061651/btz144.pdf
[^1_46]: https://pmc.ncbi.nlm.nih.gov/articles/PMC11658801/
[^1_47]: https://arxiv.org/pdf/2308.12275.pdf
[^1_48]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/cpe.6465
[^1_49]: https://pmc.ncbi.nlm.nih.gov/articles/PMC11956953/
[^1_50]: https://arxiv.org/pdf/1905.07224.pdf
[^1_51]: https://arxiv.org/html/2409.12161v1
[^1_52]: https://paulcalvano.com/2024-03-19-choosing-between-gzip-brotli-and-zstandard-compression/
[^1_53]: https://www.zlib.net/manual.html
[^1_54]: https://www.reddit.com/r/cpp/comments/13fbixk/how_does_memory_pool_combat_memory_fragmentation/
[^1_55]: https://www.debugbear.com/blog/http-compression-gzip-brotli
[^1_56]: https://websockets.readthedocs.io/en/15.0/topics/compression.html
[^1_57]: https://koder.ai/blog/zstd-vs-brotli-vs-gzip-api-compression
[^1_58]: https://practical-scheme.net/gauche/man/gauche-refe/Zlib-compression-library.html
[^1_59]: https://github.com/websockets/ws/issues/1369
[^1_60]: https://manishrjain.com/compression-algo-moving-data
[^1_61]: https://docs.verygoodsecurity.com/vault/developer-tools/larky/library-api/zlib
[^1_62]: https://github.com/nodejs/node/issues/47709
[^1_63]: https://lemire.me/blog/2021/06/30/compressing-json-gzip-vs-zstd/
[^1_64]: https://hackage.haskell.org/package/zlib-0.5.4.1/docs/Codec-Compression-Zlib-Internal.html


# Node.js Stream Module: شرح متعمق شامل

## نظرة عامة على الوحدة

**Stream** (التدفق) في Node.js هو مفهوم أساسي لمعالجة البيانات بكفاءة عالية، خاصة عند التعامل مع ملفات كبيرة أو بيانات الشبكة أو العمليات المستمرة. وحدة `node:stream` توفر واجهة برمجية تجريدية لعمل مع البيانات المتدفقة (streaming data)، بدلاً من تحميل كامل البيانات في الذاكرة مرة واحدة. كل Stream هو instance من `EventEmitter`، مما يعني أنه يمكن الاستماع إلى أحداث مختلفة والتفاعل معها.[^2_1][^2_2][^2_3]

عندما تستخدم الـ streams بدلاً من تحميل البيانات كاملة في الذاكرة، تحصل على عدة مزايا: توفير استهلاك الذاكرة، تحسين الأداء، والقدرة على معالجة بيانات أكبر من حجم الذاكرة المتاحة. في الواقع، معظم API في Node.js (مثل `fs.createReadStream()` و`http.ServerResponse`) تستخدم الـ streams بالفعل. الـ streams مفيدة جداً عند مقارنتها بالمكتبات الخارجية لأنها جزء من المعيار الأساسي للغة ولا تتطلب تبعيات إضافية.[^2_2][^2_4]

![Node.js Stream Module Class Hierarchy and Architecture](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/dfcd66572d44bf7c4589c855e57add7c/e5506f9f-0929-43ee-be95-146ca8224dfc/e6744467.png)

Node.js Stream Module Class Hierarchy and Architecture

## أنواع البيانات والأوضاع (Modes and Data Types)

### أوضاع التدفق (Flowing Modes)

**الأوضاع الثلاثة لـ Readable Streams**: أولاً، عندما يكون `readableFlowing === null`، لا يوجد آلية لاستهلاك البيانات، وبالتالي لن يتم إنتاج أي بيانات. ثانياً، عندما يكون `readableFlowing === false`، يكون Stream في الوضع المتوقف (paused mode) حيث يجب استدعاء `read()` يدويًا. ثالثاً، عندما يكون `readableFlowing === true`، يكون Stream في الوضع المتدفق (flowing mode) حيث تُرسل البيانات تلقائياً عند توفرها.[^2_1][^2_5]

يمكن الانتقال إلى الوضع المتدفق بثلاث طرق: إضافة listener لحدث `'data'`، استدعاء دالة `resume()`، أو استدعاء `pipe()` لإرسال البيانات إلى stream كاتب. وللعودة إلى الوضع المتوقف، يمكن استدعاء `pause()` أو إزالة جميع الـ destinations المتصلة.[^2_1]

### Object Mode (وضع الكائنات)

**Object Mode**: بشكل افتراضي، تعمل جميع streams على Buffers و Strings فقط. ومع ذلك، يمكنك تفعيل "object mode" بتمرير `objectMode: true` عند إنشاء Stream. في وضع الكائنات، يمكن للـ stream معالجة أي كائن JavaScript (باستثناء `null` الذي له معنى خاص في الـ streams). هذا مفيد جداً عند العمل مع JSON أو البيانات المنظمة.[^2_3][^2_6][^2_7][^2_1]

***

## التحليل العميق للدوال والفئات (API Deep Dive)

### Readable Streams - الـ Streams القابلة للقراءة

**Function Signature**: `new Readable([options])`

**الوصف**: تمثل `Readable` واجهة مجردة لمصدر البيانات. تُستخدم لقراءة البيانات من مصدر ما (ملف، socket، أو أي مصدر آخر).[^2_5][^2_1]

#### جدول الخصائص والمعاملات الأساسية:

| المعامل/الخاصية | النوع | المطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `options.highWaterMark` | `<number>` | اختياري | حد المخزن المؤقت بالبايتات (Default: 16KB) |
| `options.encoding` | `<string>` | اختياري | الترميز الافتراضي (utf8, ascii, base64, etc) |
| `options.objectMode` | `<boolean>` | اختياري | تفعيل وضع الكائنات (Default: false) |
| `options.emitClose` | `<boolean>` | اختياري | إصدار حدث 'close' عند إغلاق Stream (Default: true) |
| `options.signal` | `<AbortSignal>` | اختياري | إشارة للإلغاء المبكر |

**الأحداث الرئيسية**:

- **`'data'`**: يُصدر عند توفر البيانات للاستهلاك. يُستخدم في الوضع المتدفق.[^2_5][^2_1]
- **`'readable'`**: يُصدر عند توفر بيانات جديدة في المخزن المؤقت. يُستخدم في الوضع المتوقف.[^2_1][^2_5]
- **`'end'`**: يُصدر عند انتهاء البيانات (لا توجد بيانات إضافية).[^2_5][^2_1]
- **`'error'`**: يُصدر عند حدوث خطأ أثناء القراءة.[^2_1][^2_5]
- **`'close'`**: يُصدر عند إغلاق Stream والموارد المرتبطة به.[^2_1]
- **`'pause'`**: يُصدر عند استدعاء `pause()` والـ stream في وضع متدفق.[^2_1]
- **`'resume'`**: يُصدر عند استدعاء `resume()` والـ stream في وضع متوقف.[^2_1]


#### الدوال الأساسية:

**`readable.read([size])`**

- **الوصف**: قراءة وإزالة البيانات من المخزن المؤقت الداخلي.[^2_5][^2_1]
- **المعاملات**:
    - `size` (اختياري): عدد البايتات المراد قراءتها. إذا لم يُحدد، تُرجع جميع البيانات المتاحة.
- **القيمة المرجعة**: `<Buffer>` | `<string>` | `<null>` | `<any>`
- **حالة الاستخدام الحقيقية**:

```javascript
const fs = require('fs');
const readStream = fs.createReadStream('largefile.txt', { highWaterMark: 64 * 1024 });

readStream.on('readable', () => {
  let chunk;
  while (null !== (chunk = readStream.read())) {
    console.log(`Read ${chunk.length} bytes`);
    // معالجة كل chunk بشكل منفصل
    processChunk(chunk);
  }
});
```

**الأخطاء الشائعة**: عدم التحقق من `null` عند استدعاء `read()` يؤدي إلى محاولة معالجة data عند انتهاء Stream. أيضاً، الخلط بين وضع `'readable'` و `'data'` يسبب عدم استدعاء `read()` تلقائياً.[^2_5][^2_1]

***

**`readable.pause()`**

- **الوصف**: إيقاف Stream المتدفق والتحول إلى الوضع المتوقف.[^2_1]
- **القيمة المرجعة**: `this` (للتسلسل)
- **متى تستخدمها**: عند الحاجة للتحكم بسرعة معالجة البيانات يدويًا.

```javascript
readable.on('data', (chunk) => {
  console.log('Processing:', chunk);
  readable.pause(); // إيقاف مؤقت لمعالجة البيانات
  
  setTimeout(() => {
    console.log('Resuming after 1 second');
    readable.resume(); // استئناف القراءة
  }, 1000);
});
```


***

**`readable.resume()`**

- **الوصف**: استئناف Stream المتوقف وتحويله إلى وضع متدفق.[^2_1]
- **القيمة المرجعة**: `this`
- **الفرق عن `read()`**: `resume()` تبدأ التدفق التلقائي، بينما `read()` تقرأ يدويًا عند الحاجة.

***

**`readable.pipe(destination[, options])`**

- **الوصف**: توصيل Stream قابل للقراءة بـ Stream قابل للكتابة وتوجيه جميع البيانات تلقائياً.[^2_5][^2_1]
- **المعاملات**:
    - `destination`: Writable stream أو Duplex/Transform stream.
    - `options.end`: إذا كان `true` (default)، ينهي الـ destination عند انتهاء المصدر.
- **القيمة المرجعة**: `destination` (للتسلسل)
- **الفائدة الرئيسية**: معالجة الـ backpressure تلقائياً.[^2_5][^2_1]

```javascript
const fs = require('fs');
const zlib = require('zlib');

// سيط بسيط: نسخ ملف
fs.createReadStream('input.txt').pipe(fs.createWriteStream('output.txt'));

// سيط معقد مع ضغط: input -> gzip -> output
fs.createReadStream('archive.tar')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('archive.tar.gz'));
```

**الأخطاء الشامة**: عدم التحقق من الأخطاء في كل Readable في السلسلة قد يؤدي لـ unhandled errors. استخدم `stream.pipeline()` للتعامل الصحيح مع الأخطاء.[^2_8][^2_1]

***

**`readable.unpipe([destination])`**

- **الوصف**: فصل Writable stream عن الـ Readable.[^2_1]
- **المعاملات**: `destination` (اختياري). إذا لم يُحدد، تُفصل جميع الـ destinations.
- **القيمة المرجعة**: `this`

```javascript
const readable = fs.createReadStream('file.txt');
const writable = fs.createWriteStream('copy.txt');

readable.pipe(writable);

// بعد 5 ثوان، فصل Stream
setTimeout(() => {
  readable.unpipe(writable);
  console.log('Unpipeد!');
}, 5000);
```


***

**`readable.setEncoding(encoding)`**

- **الوصف**: تعيين الترميز الافتراضي للبيانات المقروءة.[^2_1]
- **المعاملات**: `encoding` (utf8, ascii, base64, hex, latin1, etc)
- **الفائدة**: بدلاً من الحصول على Buffers، تحصل على strings مشفرة.

```javascript
const readStream = fs.createReadStream('file.txt');
readStream.setEncoding('utf8'); // بدلاً من Buffer

readStream.on('data', (chunk) => {
  console.log(typeof chunk); // "string" بدلاً من "object"
});
```


***

### الخصائص الهامة للـ Readable Streams:

| الخاصية | النوع | الوصف |
| :-- | :-- | :-- |
| `readable.readableFlowing` | `<boolean> \| <null>` | حالة التدفق الحالية (null/false/true) |
| `readable.readableLength` | `<number>` | عدد البايتات المتاحة في المخزن المؤقت |
| `readable.readableHighWaterMark` | `<number>` | حد المخزن المؤقت المُعين |
| `readable.readableEnded` | `<boolean>` | true بعد إصدار 'end' |
| `readable.readableAborted` | `<boolean>` | true إذا تم إنهاء قبل 'end' |
| `readable.destroyed` | `<boolean>` | true بعد استدعاء `destroy()` |
| `readable.readableObjectMode` | `<boolean>` | true إذا كان في object mode |


***

### Writable Streams - الـ Streams القابلة للكتابة

**Function Signature**: `new Writable([options])`

**الوصف**: تمثل `Writable` واجهة مجردة للوجهة التي تُكتب إليها البيانات (ملف، socket، stdout، etc).[^2_1]

#### جدول الخصائص:

| المعامل/الخاصية | النوع | المطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `options.highWaterMark` | `<number>` | اختياري | حد المخزن المؤقت (Default: 16KB) |
| `options.objectMode` | `<boolean>` | اختياري | وضع الكائنات (Default: false) |
| `options.emitClose` | `<boolean>` | اختياري | إصدار حدث close (Default: true) |
| `options.defaultEncoding` | `<string>` | اختياري | الترميز الافتراضي (Default: utf8) |
| `options.signal` | `<AbortSignal>` | اختياري | إشارة الإلغاء |

#### الأحداث الأساسية:

- **`'drain'`**: يُصدر عند تفريغ المخزن المؤقت للكتابة، تنبيه بأنه يمكن الاستمرار.[^2_1]
- **`'finish'`**: يُصدر بعد استدعاء `end()` وتفريغ جميع البيانات.[^2_1]
- **`'error'`**: يُصدر عند حدوث خطأ أثناء الكتابة.[^2_1]
- **`'close'`**: يُصدر عند إغلاق Stream.[^2_1]
- **`'pipe'`**: يُصدر عند توصيل Readable بهذا Stream.[^2_1]
- **`'unpipe'`**: يُصدر عند فصل Readable عن هذا Stream.[^2_1]


#### الدوال الأساسية:

**`writable.write(chunk[, encoding][, callback])`**

- **الوصف**: كتابة البيانات إلى Stream.[^2_1]
- **المعاملات**:
    - `chunk`: البيانات (String, Buffer, TypedArray, DataView, أو object في object mode).
    - `encoding`: الترميز إذا كان chunk string (utf8, ascii, hex, etc).
    - `callback`: تُستدعى عند اكتمال كتابة هذا chunk.
- **القيمة المرجعة**: `<boolean>` - `true` إذا كان يمكن الاستمرار في الكتابة، `false` إذا كان يجب الانتظار للـ 'drain'.
- **التعامل مع الـ Backpressure**:

```javascript
const fs = require('fs');
const source = getDataSource(); // مصدر بيانات
const writeStream = fs.createWriteStream('output.txt');

function writeData() {
  let chunk;
  while ((chunk = source.getNextChunk()) !== null) {
    const canContinue = writeStream.write(chunk);
    
    if (!canContinue) {
      console.log('Buffer full, waiting for drain...');
      writeStream.once('drain', writeData); // استئناف بعد التفريغ
      return;
    }
  }
}

writeStream.on('error', (error) => {
  console.error('Write error:', error);
});

writeData();
```

**الأخطاء الشامة**: تجاهل قيمة `false` المرجعة من `write()` يؤدي إلى استهلاك الذاكرة بشكل كبير. يجب دائماً معالجة الـ backpressure عند الكتابة المتكررة.[^2_9][^2_1]

***

**`writable.end([chunk[, encoding]][, callback])`**

- **الوصف**: إنهاء الكتابة إلى Stream.[^2_1]
- **المعاملات**: نفس معاملات `write()` مع فارق أن هذه آخر كتابة.
- **القيمة المرجعة**: `this`
- **ملاحظة مهمة**: لا ينهي Stream فوراً، بل عند `process.nextTick()` التالي.[^2_10]

```javascript
const fs = require('fs');
const writeStream = fs.createWriteStream('output.txt');

writeStream.write('Hello, ');
writeStream.write('World!');
writeStream.end('\nGoodbye!'); // كتابة نهائية وإغلاق

writeStream.on('finish', () => {
  console.log('All data written and stream ended');
});
```


***

**`writable.cork()`**

- **الوصف**: تجميع جميع عمليات الكتابة المتتالية في مخزن مؤقت واحد.[^2_1]
- **الفائدة**: تحسين الأداء عند كتابة عدة chunks صغيرة متوالية.

**`writable.uncork()`**

- **الوصف**: تفريغ المخزن المؤقت وإرسال جميع البيانات المجمعة.[^2_1]

```javascript
const writeStream = getWritableStream();

writeStream.cork();
writeStream.write('Data 1');
writeStream.write('Data 2');
writeStream.write('Data 3');

process.nextTick(() => {
  writeStream.uncork(); // إرسال الثلاثة دفعة واحدة
});
```


***

### Duplex Streams - التدفقات ثنائية الاتجاه

**Function Signature**: `new Duplex([options])`

**الوصف**: جمع واجهات Readable و Writable في كائن واحد. يسمح بقراءة وكتابة البيانات في نفس الوقت (bidirectional communication).[^2_1]

**الحالات الاستخدام الواقعية**:

1. **TCP Sockets**: التواصل ثنائي الاتجاه بين عميل وخادم.[^2_11][^2_1]
2. **WebSocket Connections**: تبادل البيانات في كلا الاتجاهين.[^2_11]
3. **Crypto Streams**: تشفير البيانات الداخلة وفك تشفيرها.[^2_11]

#### مثال عملي - خادم Echo:

```javascript
const net = require('net');
const server = net.createServer((socket) => {
  // socket هو Duplex stream
  socket.write('Welcome to Echo Server!\n');
  
  socket.on('data', (data) => {
    console.log('Received:', data.toString());
    socket.write(`Echo: ${data}`); // إرسال البيانات مرة أخرى
  });
  
  socket.on('end', () => {
    console.log('Client disconnected');
  });
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');
});
```


***

### Transform Streams - تحويل البيانات

**Function Signature**: `new Transform([options])`

**الوصف**: نوع خاص من Duplex streams يحول البيانات الداخلة قبل إخراجها. يُستخدم عند الحاجة لتعديل البيانات أثناء مرورها.[^2_1]

#### الدوال الحيوية للتنفيذ:

**`transform._transform(chunk, encoding, callback)`**

- **الوصف**: تحويل كل chunk من البيانات.[^2_1]
- **المعاملات**:
    - `chunk`: البيانات المراد تحويلها.
    - `encoding`: الترميز (إذا كانت string).
    - `callback`: استدعاء مع النتيجة أو الخطأ.
- **الاستخدام**: يجب استدعاء `this.push()` لإرسال البيانات المحولة أو `callback(error)` إذا حدث خطأ.

**`transform._flush(callback)`** (اختيارية)

- **الوصف**: تُستدعى عند انتهاء جميع البيانات، لتفريغ أي بيانات متبقية.[^2_1]


#### مثال عملي - Transform لتحويل الأحرف الصغيرة إلى كبيرة:

```javascript
const { Transform } = require('stream');
const fs = require('fs');

class UppercaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    const upperChunk = chunk.toString().toUpperCase();
    this.push(upperChunk);
    callback(); // إشارة بانتهاء معالجة هذا chunk
  }
}

// الاستخدام
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');
const uppercaseTransform = new UppercaseTransform();

readStream
  .pipe(uppercaseTransform)
  .pipe(writeStream)
  .on('finish', () => {
    console.log('Transformation complete!');
  });
```


#### مثال متقدم - معالجة الأخطاء في Transform:

```javascript
class SafeTransform extends Transform {
  _transform(chunk, encoding, callback) {
    try {
      const result = JSON.parse(chunk.toString());
      this.push(JSON.stringify(result, null, 2));
      callback();
    } catch (error) {
      callback(error); // إرسال الخطأ إلى Pipeline
    }
  }
}

const pipeline = require('stream').pipeline;

pipeline(
  fs.createReadStream('data.json'),
  new SafeTransform(),
  fs.createWriteStream('formatted.json'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline completed');
    }
  }
);
```


***

### PassThrough Streams

**Function Signature**: `new PassThrough([options])`

**الوصف**: Transform stream بسيط جداً يمرر البيانات بدون تعديل. يُستخدم للتتبع أو القياس أو التقسيم.[^2_12][^2_1]

```javascript
const { PassThrough } = require('stream');
const fs = require('fs');

const passThrough = new PassThrough();

// قياس البيانات المارة
let bytesRead = 0;
passThrough.on('data', (chunk) => {
  bytesRead += chunk.length;
  console.log(`Bytes read so far: ${bytesRead}`);
});

// الاستخدام في pipeline
fs.createReadStream('largefile.bin')
  .pipe(passThrough)
  .pipe(fs.createWriteStream('copy.bin'))
  .on('finish', () => {
    console.log(`Total bytes: ${bytesRead}`);
  });
```


***

## دوال المساعدة والأدوات (Utility Functions)

### `stream.pipeline(source, ...transforms, destination[, options])`

**الوصف**: طريقة آمنة لربط عدة streams مع معالجة تلقائية للأخطاء والـ backpressure.[^2_2][^2_1]

**المعاملات**:

- `source`: Readable stream أو iterable أو async generator.
- `transforms`: Zero أو أكثر من Transform/Duplex streams.
- `destination`: Writable stream.
- `options.signal`: AbortSignal لإلغاء Pipeline.
- `options.end`: هل تنهي الـ destination عند انتهاء المصدر (Default: true).

**الفائدة الرئيسية**: معالجة أخطاء جميع streams دفعة واحدة.[^2_8][^2_1]

```javascript
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('input.txt.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);
```


***

### `stream.finished(stream[, options])`

**الوصف**: انتظر حتى ينتهي Stream من القراءة أو الكتابة.[^2_2][^2_1]

**المعاملات**:

- `stream`: Readable أو Writable stream.
- `options.error`: عد error events (Default: true).
- `options.readable`: عد readable events.
- `options.writable`: عد writable events.
- `options.cleanup`: إزالة listeners بعد الانتهاء (Default: false).

```javascript
const { finished } = require('stream');
const fs = require('fs');

const readStream = fs.createReadStream('file.txt');

finished(readStream, (err) => {
  if (err) {
    console.error('Stream encountered error:', err);
  } else {
    console.log('Stream successfully completed');
  }
});

readStream.resume(); // يجب أن نبدأ القراءة
```


***

### `stream.Readable.from(iterable[, options])`

**الوصف**: إنشاء Readable stream من iterable أو async iterable.[^2_2][^2_1]

**المعاملات**:

- `iterable`: Array, generator, أو async generator.
- `options`: نفس خيارات Readable.

```javascript
const { Readable } = require('stream');

// من array
const readable = Readable.from([1, 2, 3, 4, 5]);

readable.on('data', (chunk) => {
  console.log('Data:', chunk);
});

// من async generator
const readableAsync = Readable.from(async function* () {
  for (let i = 0; i < 5; i++) {
    yield `Item ${i}`;
    await new Promise(resolve => setTimeout(resolve, 100));
  }
});

readableAsync.pipe(process.stdout);
```


***

### `stream.addAbortSignal(signal, stream)`

**الوصف**: إضافة إمكانية الإلغاء باستخدام AbortSignal.[^2_1]

```javascript
const { Readable, addAbortSignal } = require('stream');
const controller = new AbortController();

const readStream = fs.createReadStream('largefile.bin');
addAbortSignal(controller.signal, readStream);

// إلغاء بعد 5 ثوانٍ
setTimeout(() => {
  controller.abort();
}, 5000);
```


***

## مقارنات واختيارات (Comparisons \& Decision Making)

### جدول المقارنة بين أنواع Streams المختلفة:

| الميزة | Readable | Writable | Duplex | Transform | PassThrough |
| :-- | :-- | :-- | :-- | :-- | :-- |
| **قراءة البيانات** | ✅ | ❌ | ✅ | ✅ | ✅ |
| **كتابة البيانات** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **تحويل البيانات** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **ثنائي الاتجاه** | ❌ | ❌ | ✅ | ✅ | ❌ |
| **الاستخدام الشائع** | قراءة ملفات | كتابة ملفات | TCP sockets | ضغط، تشفير | قياس، logging |
| **التعقيد** | منخفض | منخفض | متوسط | متوسط | منخفض جداً |


***

## المزايا والعيوب والحالات الاستخدام (Pros, Cons \& Use Cases)

### مزايا استخدام Streams:[^2_4][^2_13][^2_14]

1. **توفير الذاكرة**: بدلاً من تحميل ملف 1GB كاملاً، تقرأ chunks بـ 64KB فقط.
2. **أداء عالية**: معالجة البيانات بشكل تدريجي بدلاً من انتظار التحميل الكامل.
3. **معالجة Backpressure**: Node.js تدير تلقائياً سرعات البيانات المختلفة بين مصدر ووجهة.
4. **قابلية الدمج**: يمكن دمج عدة streams بسهولة باستخدام `pipe()` أو `pipeline()`.
5. **معالجة البيانات المستمرة**: مثالية للعمل مع API responses أو WebSocket data.

### عيوب وتحديات:[^2_15][^2_16]

1. **منحنى التعلم**: مفهوم الـ backpressure والأوضاع المختلفة يمكن أن يكون معقداً.
2. **معالجة الأخطاء**: يجب الانتباه لأحداث error في كل stream في السلسلة.
3. **debugging صعب**: تتبع البيانات المتدفقة أصعب من البيانات المجمعة.
4. **متطلبات الذاكرة**: حتى مع streams، المخزن المؤقت (`highWaterMark`) يستهلك ذاكرة.

### الحالات الاستخدام الثلاث الأهم:[^2_14]

#### 1. معالجة الملفات الكبيرة جداً

```javascript
const fs = require('fs');

// بدلاً من:
// const data = fs.readFileSync('4GB_file.bin'); // ❌ سيستنزف الذاكرة

// استخدم:
fs.createReadStream('4GB_file.bin', { highWaterMark: 64 * 1024 })
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('4GB_file.bin.gz'));
```


#### 2. تحويل البيانات والمعالجة الفورية (ETL)

```javascript
const csv = require('csv-parser');
const { Transform } = require('stream');

fs.createReadStream('data.csv')
  .pipe(csv()) // تحويل CSV إلى كائنات
  .pipe(new Transform({
    objectMode: true,
    transform(record, encoding, callback) {
      // تصفية وتحويل السجلات
      if (record.age > 18) {
        record.isAdult = true;
        this.push(record);
      }
      callback();
    }
  }))
  .pipe(JSONStream.stringify())
  .pipe(process.stdout);
```


#### 3. خوادم الشبكة والتطبيقات الفورية (Real-time)

```javascript
const http = require('http');
const server = http.createServer((req, res) => {
  // req و res كلاهما streams
  
  if (req.method === 'POST' && req.url === '/upload') {
    // Stream البيانات المرفوعة مباشرة إلى ملف
    req.pipe(fs.createWriteStream('uploaded_file.bin'))
      .on('finish', () => {
        res.statusCode = 200;
        res.end('File uploaded successfully');
      });
  }
});

server.listen(3000);
```


***

## أنماط متقدمة وأفضل الممارسات (Advanced Patterns \& Best Practices)

### معالجة الأخطاء الصحيحة:

**الطريقة الخاطئة** ❌:

```javascript
readStream.pipe(transformStream).pipe(writeStream);
// إذا حدث خطأ في readStream أو transformStream، لن نعرفه!
```

**الطريقة الصحيحة** ✅:

```javascript
const { pipeline } = require('stream');

pipeline(
  readStream,
  transformStream,
  writeStream,
  (err) => {
    if (err) {
      console.error('Pipeline error:', err);
      // تنظيف الموارد
    }
  }
);
```


***

### التعامل مع Object Mode بشكل صحيح:

```javascript
const { Transform, Readable } = require('stream');

// إنشاء Readable في object mode
const objectReadable = Readable.from([
  { name: 'Alice', age: 30 },
  { name: 'Bob', age: 25 },
  { name: 'Charlie', age: 35 }
], { objectMode: true });

// Transform في object mode
class FilterAdults extends Transform {
  constructor() {
    super({ objectMode: true });
  }

  _transform(obj, encoding, callback) {
    if (obj.age >= 18) {
      this.push(obj);
    }
    callback();
  }
}

objectReadable
  .pipe(new FilterAdults())
  .on('data', (obj) => {
    console.log(`${obj.name} is an adult`);
  });
```---

## الخلاصة والتوصيات

وحدة `stream` في Node.js هي أداة أساسية وقوية لمعالجة البيانات بكفاءة عالية. تُستخدم في معظم تطبيقات Node.js الحقيقية سواء بشكل مباشر أو غير مباشر (عبر http, fs, وغيرها).[^2_14][^2_2][^2_1]

**الخطوات الأساسية عند استخدام Streams**:
1. اختر النوع الصحيح (Readable, Writable, Transform, Duplex).[^2_2]
2. استخدم `pipeline()` أو `finished()` للتعامل مع الأخطاء.[^2_8][^2_1]
3. احترم الـ backpressure عند الكتابة المتكررة.[^2_9][^2_1]
4. اختبر مع بيانات حقيقية وكبيرة للتأكد من الأداء.[^2_14]
5. وثّق سلوك التحويل إذا كنت تنشئ Transform streams مخصصة.[^2_1]

**الموارد الإضافية المهمة**:
- الوثائق الرسمية: https://nodejs.org/api/stream.html
- مقالات عملية حول الـ backpressure.[^2_17]
- أمثلة واقعية من مشاريع مفتوحة المصدر.[^2_18]

استخدام Streams بشكل صحيح يمكّن تطبيقاتك من معالجة ملايين الطلبات بكفاءة وبأقل استهلاك للذاكرة.[^2_13][^2_4]
<span style="display:none">[^2_19][^2_20][^2_21][^2_22][^2_23][^2_24][^2_25][^2_26][^2_27][^2_28][^2_29][^2_30][^2_31][^2_32][^2_33][^2_34][^2_35][^2_36][^2_37][^2_38][^2_39][^2_40][^2_41][^2_42][^2_43][^2_44][^2_45][^2_46][^2_47][^2_48][^2_49][^2_50][^2_51][^2_52][^2_53][^2_54][^2_55][^2_56][^2_57][^2_58][^2_59][^2_60][^2_61][^2_62][^2_63][^2_64][^2_65][^2_66][^2_67][^2_68][^2_69][^2_70][^2_71][^2_72][^2_73][^2_74][^2_75][^2_76][^2_77][^2_78][^2_79][^2_80][^2_81][^2_82][^2_83]</span>

<div align="center">⁂</div>

[^2_1]: https://arxiv.org/pdf/1801.06144.pdf
[^2_2]: https://github.com/nodejs/node/blob/main/doc/api/stream.md?plain=1
[^2_3]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/stream/
[^2_4]: https://www.sandersdenardi.com/readable-writable-transform-streams-node/
[^2_5]: http://nodejs.p2hp.com/api/v16/stream/
[^2_6]: https://nodejs.org/en/learn/modules/how-to-use-streams
[^2_7]: https://www.freecodecamp.org/news/node-js-streams-everything-you-need-to-know-c9141306be93/
[^2_8]: https://stackoverflow.com/questions/21771220/error-handling-with-node-js-streams/53261298
[^2_9]: https://www.linkedin.com/pulse/handling-backpressure-nodejs-efficiently-managing-large-rahman-mflfc
[^2_10]: https://stackoverflow.com/questions/35437744/nodejs-streaming-readable-writable-misunderstood
[^2_11]: https://blog.dennisokeeffe.com/blog/2024-07-11-duplex-streams-in-nodejs
[^2_12]: https://blog.dennisokeeffe.com/blog/2024-07-14-nodejs-streams-in-action-with-the-aws-cdk
[^2_13]: https://blog.appsignal.com/2022/02/02/use-streams-to-build-high-performing-nodejs-applications.html
[^2_14]: https://rootstack.com/en/blog/nodejs-streams-deep-dive-and-practical-applications
[^2_15]: https://stackoverflow.com/questions/28355079/how-do-node-js-streams-work
[^2_16]: https://stackoverflow.com/questions/20769132/whats-the-proper-way-to-handle-back-pressure-in-a-node-js-transform-stream
[^2_17]: https://nodejs.org/en/learn/modules/backpressuring-in-streams
[^2_18]: https://nicolashery.com/parse-data-files-using-nodejs-streams/
[^2_19]: https://arxiv.org/pdf/2308.13436.pdf
[^2_20]: https://dl.acm.org/doi/pdf/10.1145/3625468.3652194
[^2_21]: https://arxiv.org/pdf/2211.13461.pdf
[^2_22]: http://arxiv.org/pdf/2108.02797.pdf
[^2_23]: https://arxiv.org/pdf/2412.08646.pdf
[^2_24]: https://arxiv.org/pdf/2101.04622.pdf
[^2_25]: https://arxiv.org/pdf/2403.14940.pdf
[^2_26]: https://docs.deno.com/api/node/stream/
[^2_27]: https://www.yld.io/blog/streams-readable-writable-transform-flow-control
[^2_28]: https://www.geeksforgeeks.org/node-js/node-js-streams/
[^2_29]: https://dev.to/rajeshrenato/understanding-nodejs-streams-readable-writable-transform-with-custom-examples-544j
[^2_30]: https://blog.risingstack.com/the-definitive-guide-to-object-streams-in-node-js/
[^2_31]: https://nodejs.org/download/release/v4.8.6/docs/api/stream.html
[^2_32]: https://www.dennisokeeffe.com/blog/2024-07-08-readable-streams-in-nodejs
[^2_33]: https://developers.cloudflare.com/workers/runtime-apis/nodejs/streams/
[^2_34]: https://www.thenodebook.com/streams/transform-streams
[^2_35]: https://www.geeksforgeeks.org/node-js/node-js-stream-complete-reference/
[^2_36]: https://nodejs.org/api/stream.html
[^2_37]: https://blog.logrocket.com/guide-node-js-readable-streams/
[^2_38]: https://al-kindipublisher.com/index.php/jcsts/article/view/9727
[^2_39]: https://onepetro.org/SPEADIP/proceedings/25ADIP/792780
[^2_40]: https://www.semanticscholar.org/paper/070734e43fdf78d6b874ce418facd42d220a7ed3
[^2_41]: https://eajournals.org/ejcsit/vol13-issue41-2025/data-engineering-the-catalyst-for-aviation-industry-transformation/
[^2_42]: https://www.frontiersin.org/articles/10.3389/fmech.2023.1242612/full
[^2_43]: https://www.semanticscholar.org/paper/8a23be61506b8c64361cd90e6a024ff628671e79
[^2_44]: https://jurnal.itscience.org/index.php/ijmdsa/article/view/3741
[^2_45]: https://www.semanticscholar.org/paper/3852eaccbe017a5d12414f2ffd91f7a69870c19e
[^2_46]: https://gisrrj.com/GISRRJ247519
[^2_47]: https://biss.pensoft.net/article/183270/
[^2_48]: https://www.tib-op.org/ojs/index.php/bis/article/download/41/117
[^2_49]: https://zenodo.org/record/5163394/files/Preprint-TPDS-2021.pdf
[^2_50]: https://www.isprs-ann-photogramm-remote-sens-spatial-inf-sci.net/IV-4/243/2018/isprs-annals-IV-4-243-2018.pdf
[^2_51]: https://arxiv.org/pdf/1801.08648.pdf
[^2_52]: https://www.epj-conferences.org/articles/epjconf/pdf/2019/19/epjconf_chep2018_04030.pdf
[^2_53]: https://arxiv.org/pdf/1512.07067.pdf
[^2_54]: https://arxiv.org/pdf/2107.07296.pdf
[^2_55]: https://arxiv.org/pdf/2504.02364.pdf
[^2_56]: https://www.w3schools.com/nodejs/nodejs_streams.asp
[^2_57]: https://stackoverflow.com/questions/77476725/node-js-streaming-backpressure-handling-between-services
[^2_58]: https://www.youtube.com/watch?v=23Zylsizers
[^2_59]: https://techinsights.manisuec.com/nodejs/backpressure-stream-optimization/
[^2_60]: https://dev.to/devopsfundamentals/nodejs-fundamentals-duplex-1n47
[^2_61]: https://betterstack.com/community/guides/scaling-nodejs/nodejs-streams/
[^2_62]: https://www.linkedin.com/posts/aajinkya_100daysoflearning-fullstackengineering-activity-7381747054538178560-s3J0
[^2_63]: https://dev.to/codexstoney/handling-backpressure-in-nodejs-streams-2dck
[^2_64]: https://nodejs.org/en/blog/feature/streams2
[^2_65]: https://dev.to/imsushant12/efficient-data-handling-with-nodejs-streams-4483
[^2_66]: https://nodejs.org/en/guides/backpressuring-in-streams
[^2_67]: https://arxiv.org/pdf/2107.13708.pdf
[^2_68]: https://arxiv.org/pdf/1810.13430.pdf
[^2_69]: http://arxiv.org/pdf/2307.07087.pdf
[^2_70]: http://arxiv.org/pdf/2209.07427.pdf
[^2_71]: http://arxiv.org/pdf/2405.06606.pdf
[^2_72]: https://arxiv.org/pdf/1208.4176.pdf
[^2_73]: http://arxiv.org/pdf/2502.20533.pdf
[^2_74]: https://stackoverflow.com/questions/21771220/error-handling-with-node-js-streams
[^2_75]: https://bun.com/reference/node/stream/default/PassThrough
[^2_76]: https://stackoverflow.com/questions/14269233/node-js-how-to-read-a-stream-into-a-buffer
[^2_77]: https://stackoverflow.com/questions/49317685/read-is-not-implemented-on-readable-stream
[^2_78]: https://coreui.io/answers/how-to-handle-stream-errors-in-nodejs/
[^2_79]: https://github.com/reactphp/stream
[^2_80]: https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams
[^2_81]: https://www.thenodebook.com/streams/readable-streams
[^2_82]: https://nodejs.org/api/errors.html
[^2_83]: https://nodejs.org/download/release/v5.4.0/docs/api/stream.html```


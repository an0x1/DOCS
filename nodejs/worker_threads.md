
# دليل عميق: وحدة `worker_threads` في Node.js

**ملخص تنفيذي:** وحدة `worker_threads` توفر آلية قوية لتنفيذ أكواد JavaScript بشكل متوازي في نفس العملية. بخلاف `child_process` أو `cluster`، يمكن للـ worker threads أن تشارك الذاكرة، مما يجعلها الخيار الأمثل للمهام المكثفة على المعالج (CPU-intensive) مثل معالجة الصور وتحليل البيانات. هذا الدليل يغطي كل دالة وخاصية في الوحدة مع أمثلة عملية واقعية وأفضل الممارسات.[^1_1][^1_2]

***

## 1. نظرة عامة على الوحدة

### لماذا تحتاج إلى `worker_threads`؟

Node.js مشهور بمعمارية الحلقة الواحدة (single-threaded event loop)، وهي مثالية للعمليات I/O غير المتزامنة. لكن عندما تحتاج تطبيقك لتنفيذ حسابات معقدة (مثل معالجة صور ضخمة، تحليل JSON كبير، أو حسابات رياضية)، فإن هذه المهام تحجب الحلقة الرئيسية وتبطء الاستجابة.[^1_3][^1_4]

هنا تأتي دور `worker_threads`: تسمح بتشغيل أكواد JavaScript في threads معزولة، مما يحرر الحلقة الرئيسية للتعامل مع طلبات I/O الأخرى. بخلاف `child_process`، الـ worker threads خفيفة الوزن وتشارك الذاكرة، مما يجعل التواصل بينها أسرع بكثير.[^1_2][^1_3]

### متى تستخدم worker_threads؟

استخدم `worker_threads` عندما:

- تحتاج لتنفيذ مهام **مكثفة على المعالج** (حسابات رياضية، معالجة صور، تحليل بيانات ضخمة)
- المهام تستغرق أكثر من 50 ملي ثانية
- تريد استخدام **عدة أنوية CPU** للتوازي الفعلي
- تحتاج لمشاركة ذاكرة بكفاءة بين threads[^1_4]

**لا تستخدم** `worker_threads` إذا كانت مهامك تعتمد على I/O (استدعاءات قواعد البيانات، طلبات HTTP) - فالـ async/await أفضل في هذه الحالات.[^1_3]

***

## 2. واجهة برمجية عميقة (API Deep Dive)

### 2.1 إنشاء Worker - `new Worker(filename[, options])`

**التوقيع:**```javascript
const worker = new Worker(filename, [options])

```

**المعاملات:**

| المعامل | النوع | اختياري | الوصف |
|--------|-------|---------|--------|
| `filename` | `string \| URL` | مطلوب | مسار ملف Worker (مطلق أو نسبي يبدأ بـ `./` أو `../`) أو `data:` URL. إذا كان `eval: true` فهذا يكون كود JavaScript مباشرة |
| `argv` | `any[]` | اختياري | قائمة معاملات تُلحق بـ `process.argv` في Worker |
| `env` | `Object` | اختياري | متغيرات البيئة الأولية (افتراضي: نسخة من `process.env`). استخدم `worker.SHARE_ENV` لمشاركة البيئة |
| `eval` | `boolean` | اختياري | إذا كانت `true` يتم تفسير `filename` كأكواد JavaScript بدلاً من مسار |
| `execArgv` | `string[]` | اختياري | معاملات Node.js CLI (مثل `--max-old-space-size=2048`) |
| `stdin` | `boolean` | اختياري | تفعيل أنبوب `process.stdin` (افتراضي: `false`) |
| `stdout` | `boolean` | اختياري | منع نقل `process.stdout` تلقائياً للوالد (افتراضي: `false`) |
| `stderr` | `boolean` | اختياري | منع نقل `process.stderr` تلقائياً للوالد (افتراضي: `false`) |
| `workerData` | `any` | اختياري | بيانات تُنسخ وتُمرر للـ Worker (متاح عبر `workerData` في الـ Worker) |
| `trackUnmanagedFds` | `boolean` | اختياري | تتبع وإغلاق ملفات FDs عند خروج Worker (افتراضي: `true`) |
| `transferList` | `Object[]` | اختياري | قائمة الكائنات المراد نقل ملكيتها (ضروري إذا احتوت `workerData` على `MessagePort`) |
| `resourceLimits` | `Object` | اختياري | حدود محرك JavaScript (الذاكرة والمكدس) |
| `name` | `string` | اختياري | اسم Worker للتصحيح والمراقبة |

**القيمة المرجعة:** كائن `Worker` (يرث من `EventEmitter`)

**أحداث Worker:**

| الحدث | المعامل | الوصف |
|------|---------|--------|
| `'online'` | بدون | Worker بدأ تنفيذ الكود |
| `'message'` | `value: any` | استقبال رسالة من `parentPort.postMessage()` |
| `'messageerror'` | `error: Error` | فشل فك ترميز الرسالة |
| `'error'` | `error: Error` | استثناء لم يتم التعامل معه في Worker |
| `'exit'` | `code: number` | Worker توقف (الحدث النهائي) |

**مثال عملي - معالجة صور:**

```javascript
// main.js
const { Worker } = require('node:worker_threads');
const path = require('path');

function processImageInWorker(imagePath, width, height) {
  return new Promise((resolve, reject) => {
    // إنشاء worker بمسار ملف الـ worker
    const worker = new Worker('./image-worker.js', {
      workerData: { imagePath, width, height }
    });

    // استقبال الصورة المعالجة من Worker
    worker.on('message', (processedBuffer) => {
      resolve(processedBuffer);
      // لا نحتاج terminate هنا لأن Worker سينتهي تلقائياً
    });

    // التعامل مع الأخطاء
    worker.on('error', (err) => {
      reject(new Error(`صورة فشل معالجتها: ${err.message}`));
    });

    // التحقق من الخروج غير الطبيعي
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker توقف برمز: ${code}`));
      }
    });
  });
}

// الاستخدام
(async () => {
  try {
    const result = await processImageInWorker(
      './input.jpg', 
      800, 
      600
    );
    console.log('تم معالجة الصورة بنجاح');
  } catch (err) {
    console.error('خطأ:', err.message);
  }
})();
```

```javascript
// image-worker.js
const { parentPort, workerData } = require('node:worker_threads');
const sharp = require('sharp'); // مكتبة معالجة الصور

async function resizeImage() {
  try {
    const { imagePath, width, height } = workerData;
    
    // معالجة الصورة (عملية مكثفة على المعالج)
    const resizedImage = await sharp(imagePath)
      .resize(width, height, { fit: 'cover' })
      .toBuffer();
    
    // إرسال النتيجة للخيط الرئيسي
    parentPort.postMessage(resizedImage);
  } catch (err) {
    throw new Error(`فشل تغيير حجم الصورة: ${err.message}`);
  }
}

resizeImage();
```

**الأخطاء الشائعة:**

- نسيان استدعاء `worker.terminate()` عند الانتهاء (يؤدي لتسرب الذاكرة)
- محاولة نقل كائنات غير قابلة للنقل في `transferList`
- تجاوز عدد worker threads أكثر من عدد أنوية CPU
- عدم التعامل مع حدث `'error'`

***

### 2.2 الخصائص العامة

#### 2.2.1 `isMainThread` - التحقق من كون الكود في الخيط الرئيسي

**النوع:** `boolean` (قراءة فقط)

**الوصف:** تُرجع `true` إذا كان الكود يعمل في الخيط الرئيسي، و `false` إذا كان داخل Worker thread.

**مثال:**

```javascript
const { Worker, isMainThread } = require('node:worker_threads');

if (isMainThread) {
  console.log('هذا في الخيط الرئيسي');
  
  // إنشاء worker
  const worker = new Worker(__filename);
  worker.on('message', (msg) => console.log(msg));
} else {
  console.log('هذا في worker thread');
  // parentPort متاح هنا فقط
}
```


#### 2.2.2 `parentPort` - قنوات التواصل مع الخيط الأب

**النوع:** `null | MessagePort` (قراءة فقط)

**الوصف:** إذا كان الكود يعمل داخل Worker، يوفر `parentPort` ميناء رسائل للتواصل مع الخيط الأب. في الخيط الرئيسي، تكون قيمته `null`.

**الدوال:**


| الدالة | المعاملات | الوصف |
| :-- | :-- | :-- |
| `parentPort.postMessage(value, transferList)` | `value: any`, `transferList: Object[]` | إرسال رسالة للخيط الأب |
| `parentPort.on('message', handler)` | `handler: (value) => void` | استقبال رسائل من الخيط الأب |
| `parentPort.once(event, handler)` | حسب الحدث | استقبال حدث واحد فقط |
| `parentPort.close()` | - | إغلاق القنة (لا توجد رسائل بعدها) |
| `parentPort.start()` | - | بدء استقبال الرسائل يدويًا |
| `parentPort.ref() / unref()` | - | التحكم في حفظ العملية نشطة |

**مثال - حساب فيبوناتشي متوازي:**

```javascript
// main.js
const { Worker } = require('node:worker_threads');

function calculateFibonacci(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./fib-worker.js', { workerData: n });
    
    worker.on('message', (result) => {
      resolve(result);
    });
    
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`خروج غير متوقع: ${code}`));
    });
  });
}

(async () => {
  // حساب عدة قيم بتوازي
  const results = await Promise.all([
    calculateFibonacci(40),
    calculateFibonacci(41),
    calculateFibonacci(42)
  ]);
  
  console.log('النتائج:', results);
})();
```

```javascript
// fib-worker.js
const { parentPort, workerData } = require('node:worker_threads');

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// حساب قيمة فيبوناتشي
const result = fibonacci(workerData);

// إرسال النتيجة للخيط الأب
parentPort.postMessage(result);
```


#### 2.2.3 `workerData` - البيانات المُمررة من الخيط الأب

**النوع:** `any` (قراءة فقط)

**الوصف:** نسخة مستنسخة من البيانات المُمررة عبر `workerData` في اختيارات `new Worker()`. يتم النسخ باستخدام خوارزمية الاستنساخ المنظم (HTML structured clone algorithm).

**مثال:**

```javascript
// main.js
const { Worker } = require('node:worker_threads');

const worker = new Worker('./worker.js', {
  workerData: {
    csvFile: '/data/huge-file.csv',
    delimiter: ',',
    encoding: 'utf-8'
  }
});

worker.on('message', (data) => console.log('تم معالجة', data.rows, 'صف'));
```

```javascript
// worker.js
const { parentPort, workerData } = require('node:worker_threads');
const fs = require('fs');
const readline = require('readline');

async function parseCSV() {
  const { csvFile, delimiter, encoding } = workerData;
  
  let rowCount = 0;
  const stream = fs.createReadStream(csvFile, { encoding });
  
  const rl = readline.createInterface({
    input: stream,
    crlfDelay: Infinity
  });
  
  for await (const line of rl) {
    // معالجة كل سطر
    const fields = line.split(delimiter);
    rowCount++;
  }
  
  parentPort.postMessage({ rows: rowCount });
}

parseCSV();
```


#### 2.2.4 `threadId` - معرّف الخيط الفريد

**النوع:** `number` (قراءة فقط)

**الوصف:** معرف فريد لكل خيط داخل العملية. مفيد للتديغ والمراقبة.

```javascript
const { Worker, threadId } = require('node:worker_threads');

console.log(`معرف الخيط الحالي: ${threadId}`);

if (threadId === 0) {
  console.log('هذا هو الخيط الرئيسي');
}
```


#### 2.2.5 `SHARE_ENV` - مشاركة متغيرات البيئة

**النوع:** `symbol` (قراءة فقط)

**الوصف:** رمز خاص يشير لمشاركة نفس متغيرات البيئة بين الخيط الأب والـ Worker. بدون هذا، كل Worker يحصل على نسخة مستقلة من `process.env`.

```javascript
const { Worker, SHARE_ENV } = require('node:worker_threads');

// مشاركة البيئة
const worker = new Worker('./worker.js', { env: SHARE_ENV });

// تغيير متغير بيئة في Worker
// سيؤثر على العملية الأم أيضاً
```


***

### 2.3 دوال إدارة البيانات البيئية

#### 2.3.1 `setEnvironmentData(key, value)` - تعيين بيانات البيئة العامة

**الاستدعاء:**

```javascript
setEnvironmentData(key, value)
```

**المعاملات:**


| المعامل | النوع | الوصف |
| :-- | :-- | :-- |
| `key` | `any` | مفتاح قابل للنسخ (عادة string) |
| `value` | `any` | القيمة المراد تعيينها (يمكن نسخها) |

**الوصف:** تعيين بيانات بيئية عامة تتوفر تلقائياً لجميع Wokers الجدد في السياق. مفيدة لتمرير تكوينات عامة.

**مثال - تكوين عام لمعالجات الصور:**

```javascript
// main.js
const { Worker, setEnvironmentData } = require('node:worker_threads');

// تعيين إعدادات عامة
setEnvironmentData('maxImageSize', { width: 2000, height: 2000 });
setEnvironmentData('imageQuality', 85);

// جميع Wokers الجدد ستصل إليها
for (let i = 0; i < 4; i++) {
  const worker = new Worker('./image-worker.js');
}
```

```javascript
// image-worker.js
const { getEnvironmentData, workerData } = require('node:worker_threads');

const maxSize = getEnvironmentData('maxImageSize');
const quality = getEnvironmentData('imageQuality');

console.log(`الحد الأقصى للحجم: ${JSON.stringify(maxSize)}`);
```


#### 2.3.2 `getEnvironmentData(key)` - استرجاع بيانات البيئة

**الاستدعاء:**

```javascript
const value = getEnvironmentData(key)
```

**القيمة المرجعة:** نسخة من البيانات المُعينة عبر `setEnvironmentData()`.

***

### 2.4 `MessageChannel` - قنوات اتصال متقدمة

#### الوصف والاستخدام

`MessageChannel` يوفر قنوات اتصال ثنائية الاتجاه بين Wokers. مفيدة عندما تريد اتصالاً مباشراً بين worker واثنين دون المرور عبر الخيط الأب.

**الاستدعاء:**

```javascript
const { port1, port2 } = new MessageChannel()
```

**الخصائص:**


| الخاصية | النوع | الوصف |
| :-- | :-- | :-- |
| `port1` | `MessagePort` | الميناء الأول |
| `port2` | `MessagePort` | الميناء الثاني (متصل بـ port1) |

**مثال - اتصال مباشر بين Wokers:**

```javascript
// main.js
const { Worker, MessageChannel } = require('node:worker_threads');

const worker1 = new Worker('./worker1.js');
const worker2 = new Worker('./worker2.js');

// إنشاء قناة اتصال مباشرة
const { port1, port2 } = new MessageChannel();

// نقل port1 لـ worker1 و port2 لـ worker2
worker1.postMessage({ port: port1 }, [port1]);
worker2.postMessage({ port: port2 }, [port2]);

console.log('تم ربط الـ workers مباشرة');
```

```javascript
// worker1.js - المُرسل
const { parentPort } = require('node:worker_threads');

parentPort.once('message', ({ port }) => {
  // إرسال بيانات عبر القناة المباشرة
  port.postMessage({ data: 'بيانات من Worker 1' });
  port.close();
});
```

```javascript
// worker2.js - المستقبل
const { parentPort } = require('node:worker_threads');

parentPort.once('message', ({ port }) => {
  port.once('message', (msg) => {
    console.log('استقبل:', msg.data);
    port.close();
  });
});
```


***

### 2.5 `MessagePort` - الميناء الفردي

#### الدوال الأساسية

```javascript
port.postMessage(value[, transferList])
```

**المعاملات:**


| المعامل | النوع | الوصف |
| :-- | :-- | :-- |
| `value` | `any` | البيانات المراد إرسالها |
| `transferList` | `Object[]` | قائمة الكائنات المراد نقل ملكيتها |

**الكائنات القابلة للنقل:**

- `ArrayBuffer` - نقل الذاكرة (لا يمكن استخدامها بعدها في الجهة المُرسلة)
- `MessagePort` - نقل ميناء آخر
- `FileHandle` - مقبضة ملف

**الكائنات القابلة للاستنساخ (النسخ):**

- Objects عادية، Arrays، Maps، Sets
- Typed Arrays و Buffers (يتم استنساخها، لا نقل الملكية)
- Regular Expressions، Dates، URLs
- WebAssembly.Module instances

**مثال - نقل ArrayBuffer لتحسين الأداء:**

```javascript
// main.js
const { Worker } = require('node:worker_threads');

const imageBuffer = Buffer.alloc(10_000_000); // 10MB
const worker = new Worker('./processor.js');

// نقل buffer بدلاً من نسخها (أسرع بكثير!)
worker.postMessage(
  { buffer: imageBuffer },
  [imageBuffer.buffer]  // نقل الملكية
);

// imageBuffer لن تكون قابلة للاستخدام هنا بعد الآن
console.log(imageBuffer.length); // سيكون 0 (تم نقل الملكية)
```

```javascript
// processor.js
const { parentPort } = require('node:worker_threads');

parentPort.on('message', ({ buffer }) => {
  console.log(`استقبل buffer بحجم: ${buffer.length} بايت`);
  // معالجة البيانات بكفاءة (لا تأثير نسخ)
});
```


#### الأحداث

```javascript
port.on('message', (value) => { })    // استقبال رسالة
port.on('messageerror', (error) => { }) // فشل في فك التشفير
port.on('close', () => { })            // تم إغلاق القناة

port.start()                           // بدء الاستقبال يدويًا
port.close()                           // إغلاق القناة
```


***

### 2.6 مشاركة الذاكرة - `SharedArrayBuffer` و `Atomics`

#### مشاركة الذاكرة بدون نسخ

عندما تريد عدة Wokers أن تشارك نفس البيانات الضخمة بدون نسخها، استخدم `SharedArrayBuffer`:

```javascript
// main.js
const { Worker } = require('node:worker_threads');

// إنشاء ذاكرة مشتركة (1024 عناصر int32)
const sharedMemory = new SharedArrayBuffer(1024 * 4);
const sharedArray = new Int32Array(sharedMemory);

// تعبئة البيانات
for (let i = 0; i < 1024; i++) {
  sharedArray[i] = i * 2;
}

// تمرير الذاكرة المشتركة
const worker = new Worker('./processor.js');
worker.postMessage({ sharedMemory });

// مراقبة التغييرات في الوقت الفعلي
setInterval(() => {
  console.log(`أول عنصر معالج: ${sharedArray[^1_0]}`);
}, 100);
```

```javascript
// processor.js
const { parentPort } = require('node:worker_threads');

parentPort.on('message', ({ sharedMemory }) => {
  const sharedArray = new Int32Array(sharedMemory);
  
  // معالجة البيانات (بدون نسخ)
  for (let i = 0; i < sharedArray.length; i++) {
    sharedArray[i] = sharedArray[i] * 3;
  }
  
  console.log('تمت المعالجة');
});
```


#### التزامن - `Atomics` API

عند استخدام `SharedArrayBuffer` مع عدة Wokers، تحتاج لتجنب race conditions:

```javascript
// main.js
const { Worker, Atomics } = require('node:worker_threads');

const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);

// تعيين قيمة أولية آمنة
Atomics.store(sharedArray, 0, 0);

for (let i = 0; i < 4; i++) {
  const worker = new Worker('./counter.js');
  worker.postMessage({ sharedBuffer });
}

// الانتظار حتى ينتهي كل Worker
setTimeout(() => {
  const finalCount = Atomics.load(sharedArray, 0);
  console.log(`العدد النهائي: ${finalCount}`);
}, 5000);
```

```javascript
// counter.js
const { parentPort, Atomics } = require('node:worker_threads');

parentPort.on('message', ({ sharedBuffer }) => {
  const sharedArray = new Int32Array(sharedBuffer);
  
  // إضافة آمنة عبر Atomics
  for (let i = 0; i < 10000; i++) {
    Atomics.add(sharedArray, 0, 1);  // إضافة آمنة
  }
});
```

**عمليات Atomics الرئيسية:**


| العملية | الوصف |
| :-- | :-- |
| `Atomics.load(arr, index)` | قراءة آمنة |
| `Atomics.store(arr, index, value)` | كتابة آمنة |
| `Atomics.add(arr, index, value)` | إضافة آمنة |
| `Atomics.compareExchange(arr, i, expected, new)` | تحديث شرطي |
| `Atomics.wait(int32Array, index, value[, timeout])` | انتظار حتى قيمة معينة |
| `Atomics.notify(int32Array, index[, count])` | إيقاظ Wokers في انتظار |


***

### 2.7 دوال متقدمة

#### 2.7.1 `postMessageToThread(threadId, value[, transferList][, timeout])` - إرسال رسالة لأي thread

**الاستدعاء:**

```javascript
await postMessageToThread(threadId, value, transferList, timeout)
```

**المعاملات:**


| المعامل | النوع | الوصف |
| :-- | :-- | :-- |
| `threadId` | `number` | معرف الخيط المستهدف |
| `value` | `any` | البيانات |
| `transferList` | `Object[]` | كائنات للنقل |
| `timeout` | `number` | انتظار القصوى (ملي ثانية) |

**الوصف:** إرسال رسالة مباشرة لأي خيط دون الحاجة للمرور عبر الخيط الأب. مفيدة في هياكل Wokers المعقدة.

**مثال:**

```javascript
// main.js
const { Worker, postMessageToThread, threadId } = require('node:worker_threads');

const worker1 = new Worker('./nested-worker.js');
const worker2 = new Worker('./nested-worker.js');

// الخيط الرئيسي يرسل أوامر مباشرة
setInterval(() => {
  postMessageToThread(worker1.threadId, { command: 'status' });
  postMessageToThread(worker2.threadId, { command: 'status' });
}, 1000);
```


***

## 3. مقارنات واختيارات

### Worker Threads vs Child Processes vs Cluster

| الميزة | Worker Threads | Child Processes | Cluster |
| :-- | :-- | :-- | :-- |
| **عزل الذاكرة** | معزول لكن قابل للمشاركة | كامل العزل | كامل العزل |
| **التكلفة العامة** | منخفضة جداً | عالية | عالية جداً |
| **سرعة التواصل** | سريعة جداً (ذاكرة مشتركة) | بطيئة (IPC) | بطيئة (IPC) |
| **حالات الاستخدام** | حسابات CPU مكثفة | تشغيل برامج خارجية | توازن أحمال الخوادم |
| **وقت الإنشاء** | أقل من 1ms | 50-100ms | 500ms+ |
| **قابلية التوسع** | حتى عدد أنوية CPU | محدودة | أنوية CPU |
| **أفضل مثال** | معالجة صور، حسابات رياضية | تشغيل ffmpeg | خادم HTTP متعدد المعالجات |

**الخلاصة:** اختر worker threads للعمليات الثقيلة على المعالج، child processes للعزل الكامل وتشغيل برامج خارجية، و cluster لموازنة أحمال خوادم الويب.[^1_5][^1_3]

***

## 4. الفوائد والقيود والحالات الاستخدام

### الفوائد (المميزات)

1. **استخدام متعدد الأنوية:** تسخير جميع أنوية CPU للمعالجة الفعلية المتوازية
2. **خفيفة الوزن:** رفع الأداء بدون تكاليف إنشاء عمليات جديدة
3. **مشاركة الذاكرة:** نقل بيانات ضخمة دون نسخها (عبر `ArrayBuffer` و `SharedArrayBuffer`)
4. **عزل آمن:** كل thread له V8 instance منفصلة، مما يقلل من مشاكل التزامن
5. **متوافقة مع معظم APIs:** النسخ الحالية من Node.js تدعم معظم وحدات Node

### القيود (العيوب)

1. **حد أقصى من Wokers:** لا تستطيع إنشاء أكثر من عدد أنوية CPU دون تدهور الأداء
2. **استهلاك الذاكرة:** كل Worker لها نسختها الخاصة من محرك V8 (حوالي 10-20MB لكل thread)
3. **تعقيد البرمجة:** التعامل مع message passing والتزامن أصعب من الكود العادي
4. **عدم القدرة على مشاركة بعض الكائنات:** الدوال وبعض الكائنات الأصلية لا تُنسخ بسهولة
5. **نقل البيانات الضخمة:** حتى مع `SharedArrayBuffer`، الكائنات المعقدة قد تكون بطيئة

***

### أفضل 3 حالات استخدام

#### 1. معالجة الصور والملفات الضخمة

**الحالة:** تطبيق ويب يحتاج لتغيير أحجام الصور المرفوعة تلقائياً

```javascript
// main.js - خادم Express
const express = require('express');
const { Worker } = require('node:worker_threads');
const os = require('os');

const app = express();

// مجموعة من Wokers (عدد الأنوية)
const numWorkers = os.cpus().length;
const workerPool = [];

class WorkerPool {
  constructor(numWorkers) {
    this.workers = [];
    this.taskQueue = [];
    
    for (let i = 0; i < numWorkers; i++) {
      const worker = new Worker('./image-worker.js');
      worker.busy = false;
      worker.on('message', this.handleWorkerMessage.bind(this));
      this.workers.push(worker);
    }
  }

  handleWorkerMessage(msg) {
    // معالجة النتيجة...
  }

  execute(task) {
    return new Promise((resolve, reject) => {
      const availableWorker = this.workers.find(w => !w.busy);
      
      if (availableWorker) {
        availableWorker.busy = true;
        // ...
      } else {
        this.taskQueue.push({ task, resolve, reject });
      }
    });
  }
}

const pool = new WorkerPool(numWorkers);

app.post('/upload', async (req, res) => {
  try {
    const resized = await pool.execute({
      imagePath: req.file.path,
      width: 800,
      height: 600
    });
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

**السبب:** معالجة الصور تأخذ وقتاً طويلاً وتحجب الحلقة الرئيسية. باستخدام Wokers يبقى الخادم يستجيب للطلبات الأخرى.

***

#### 2. حسابات البيانات الإحصائية

**الحالة:** تحليل ملايين سجلات البيانات

```javascript
// analytics-worker.js
const { parentPort, workerData } = require('node:worker_threads');

function analyzeData(data) {
  const stats = {
    sum: 0,
    mean: 0,
    variance: 0,
    stdDev: 0,
    min: Infinity,
    max: -Infinity
  };

  // حسابات معقدة
  for (const value of data) {
    stats.sum += value;
    stats.min = Math.min(stats.min, value);
    stats.max = Math.max(stats.max, value);
  }

  stats.mean = stats.sum / data.length;
  
  // حساب الانحراف المعياري
  let variance = 0;
  for (const value of data) {
    variance += Math.pow(value - stats.mean, 2);
  }
  stats.variance = variance / data.length;
  stats.stdDev = Math.sqrt(stats.variance);

  return stats;
}

parentPort.on('message', (chunkData) => {
  const result = analyzeData(chunkData);
  parentPort.postMessage(result);
});
```

**السبب:** الحسابات الإحصائية على البيانات الضخمة تحتاج معالجة متوازية.

***

#### 3. معالجة JSON والبيانات المنظمة

**الحالة:** تحويل صيغ بيانات ضخمة (JSON إلى CSV أو XML)

```javascript
// main.js
const fs = require('fs');
const { Worker } = require('node:worker_threads');

function processLargeJSON(filePath, chunkSize = 10000) {
  return new Promise((resolve, reject) => {
    const stream = fs.createReadStream(filePath, { encoding: 'utf8' });
    let buffer = '';
    let processed = 0;
    const results = [];
    let activeWorkers = 0;
    const maxWorkers = 4;

    const workers = [];
    for (let i = 0; i < maxWorkers; i++) {
      const worker = new Worker('./json-processor.js');
      worker.on('message', (result) => {
        results.push(result);
        activeWorkers--;
        processNextChunk();
      });
      workers.push(worker);
    }

    function processNextChunk() {
      // معالجة الأجزاء بتوازي
      if (buffer.length >= chunkSize && activeWorkers < maxWorkers) {
        const chunk = buffer.slice(0, chunkSize);
        buffer = buffer.slice(chunkSize);
        activeWorkers++;
        
        const worker = workers[processed % maxWorkers];
        worker.postMessage(chunk);
      }
    }

    stream.on('data', (chunk) => {
      buffer += chunk;
      processNextChunk();
    });

    stream.on('end', () => {
      // معالجة الجزء المتبقي
      if (buffer.length > 0) {
        activeWorkers++;
        const worker = workers[processed % maxWorkers];
        worker.postMessage(buffer);
      }
    });

    stream.on('error', reject);
  });
}
```

**السبب:** ملفات JSON الضخمة (مئات MB) تتطلب معالجة متوازية لتجنب توقف الخادم.

***

## 5. أفضل الممارسات والنصائح

### 1. استخدام مجموعات (Worker Pools)

بدلاً من إنشاء Worker جديد لكل مهمة (مكلف جداً):

```javascript
// ❌ خطأ: إنشاء Worker لكل مهمة
function heavyTask(data) {
  const worker = new Worker('./worker.js');
  worker.postMessage(data);
  // ...
}

// ✅ صحيح: استخدام مجموعة
class WorkerPool {
  constructor(scriptPath, size = 4) {
    this.workers = Array(size).fill().map(() => new Worker(scriptPath));
    this.taskQueue = [];
    this.activeWorkers = new Set();
  }

  async run(data) {
    // انتظر حتى يتوفر worker
    let worker = this.workers.find(w => !this.activeWorkers.has(w));
    
    if (!worker) {
      await new Promise(resolve => {
        this.taskQueue.push(() => resolve());
      });
      worker = this.workers[^1_0];
    }

    this.activeWorkers.add(worker);

    return new Promise((resolve, reject) => {
      worker.once('message', resolve);
      worker.once('error', reject);
      worker.postMessage(data);
    }).finally(() => {
      this.activeWorkers.delete(worker);
      const callback = this.taskQueue.shift();
      callback?.();
    });
  }
}
```


### 2. تجنب نقل الكائنات الكبيرة بلا داع

```javascript
// ❌ بطيء: نسخ كائن ضخم
worker.postMessage({ largeArray: bigArray });

// ✅ سريع: نقل الملكية
worker.postMessage({ buffer }, [bigArray.buffer]);

// ✅ سريع: مشاركة الذاكرة
const shared = new SharedArrayBuffer(bigArray.byteLength);
new Int32Array(shared).set(bigArray);
worker.postMessage({ shared });
```


### 3. التعامل مع الأخطاء بشكل صحيح

```javascript
const worker = new Worker('./worker.js');

worker.on('error', (err) => {
  console.error('Worker error:', err);
  // أعد إنشاء الـ Worker
});

worker.on('exit', (code) => {
  if (code !== 0) {
    console.error(`Worker exited with code ${code}`);
  }
});

// مهم: استدعاء terminate عند الانتهاء
setTimeout(() => {
  worker.terminate().then(() => console.log('Worker terminated'));
}, 5000);
```


### 4. مراقبة استهلاك الذاكرة

```javascript
const { Worker } = require('node:worker_threads');

const worker = new Worker('./worker.js', {
  resourceLimits: {
    maxOldGenerationSizeMb: 512,    // 512MB للذاكرة القديمة
    maxYoungGenerationSizeMb: 128   // 128MB للذاكرة الجديدة
  }
});

// مراقبة الاستخدام
setInterval(() => {
  const memory = process.memoryUsage();
  console.log(`استهلاك الذاكرة: ${Math.round(memory.heapUsed / 1024 / 1024)}MB`);
}, 1000);
```


### 5. عدم تجاوز عدد أنوية CPU

```javascript
const os = require('os');
const { Worker } = require('node:worker_threads');

const cpuCount = os.cpus().length;
const workerCount = Math.min(cpuCount, 4); // لا تزيد عن 4

console.log(`إنشاء ${workerCount} من ${cpuCount} أنوية`);

for (let i = 0; i < workerCount; i++) {
  new Worker('./worker.js');
}
```---

## 6. الخلاصة والتوصيات

**متى تستخدم worker_threads:**
- ✅ معالجة صور/فيديو، حسابات رياضية معقدة
- ✅ تحليل بيانات ضخمة، معالجة JSON/CSV
- ✅ عندما تستغرق المهمة أكثر من 50ms

**متى لا تستخدمها:**
- ❌ عمليات I/O عادية (قواعد البيانات، HTTP)
- ❌ مهام بسيطة وسريعة
- ❌ عندما لا تملك سيطرة على الكود (مكتبات خارجية)

**الممارسات الذهبية:**
1. استخدم مجموعات (worker pools) دائماً
2. لا تنقل بيانات ضخمة بدون نقل الملكية
3. اتعامل مع الأخطاء والخروج بشكل سليم
4. راقب استهلاك الذاكرة والأداء
5. عدد الـ workers يجب ألا يتجاوز عدد أنوية CPU

***

## المراجع

 Node.js Official Documentation - Worker Threads[^1_1]
 GitHub Node.js Repository - worker_threads.md[^1_2]
 LinkedIn - Comparing Child Processes, Clusters, and Worker Threads[^1_3]
 Last9.io - Understanding Worker Threads in Node.js[^1_4]
 Alvin Lal - Single Thread vs Child Process vs Worker Threads vs Cluster[^1_5]
 Stack Overflow - SharedArrayBuffer and Atomics Parallelism[^1_6]
 Dev.to - High Performance JavaScript with Web Workers[^1_7]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_8][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88][^1_89][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/worker_threads.md
[^1_2]: https://nodejs.org/docs/latest/api/worker_threads.html
[^1_3]: https://www.linkedin.com/pulse/comparing-child-processes-clusters-worker-threads-nodejs-srikanth-r-z2sgc
[^1_4]: https://last9.io/blog/understanding-worker-threads-in-node-js/
[^1_5]: https://alvinlal.netlify.app/blog/single-thread-vs-child-process-vs-worker-threads-vs-cluster-in-nodejs/
[^1_6]: https://stackoverflow.com/questions/45230334/how-to-achieve-parallelism-for-sharedarraybuffer-and-atomics
[^1_7]: https://dev.to/rigalpatel001/high-performance-javascript-simplified-web-workers-sharedarraybuffer-and-atomics-3ig1
[^1_8]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_9]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_10]: http://transactions.ismir.net/articles/10.5334/tismir.111/
[^1_11]: https://www.semanticscholar.org/paper/959d24d21efe31eae401be73c1f7d8e636805655
[^1_12]: https://jsret.knpub.com/index.php/jrest/article/download/199/158
[^1_13]: https://arxiv.org/pdf/1901.05350.pdf
[^1_14]: http://arxiv.org/pdf/2402.12274.pdf
[^1_15]: https://arxiv.org/html/2401.16551v1
[^1_16]: https://arxiv.org/pdf/2403.14940.pdf
[^1_17]: https://arxiv.org/pdf/1205.6363.pdf
[^1_18]: http://arxiv.org/pdf/2407.14331.pdf
[^1_19]: https://arxiv.org/pdf/2101.00756.pdf
[^1_20]: https://www.w3schools.com/nodejs/nodejs_worker_threads.asp
[^1_21]: https://www.freecodecamp.org/news/how-to-implement-multi-threading-in-nodejs-with-worker-threads-full-handbook/
[^1_22]: https://www.geeksforgeeks.org/node-js/node-js-worker-threads/
[^1_23]: http://nodesource.com/blog/worker-threads-nodejs-multithreading-in-javascript/
[^1_24]: https://www.geeksforgeeks.org/node-js/differentiate-between-worker-threads-and-clusters-in-node-js/
[^1_25]: https://betterstack.com/community/guides/scaling-nodejs/nodejs-workers-explained/
[^1_26]: https://stackoverflow.com/questions/tagged/node-worker-threads
[^1_27]: https://docs.deno.com/api/node/worker_threads/
[^1_28]: https://loaders.gl/docs/developer-guide/concepts/worker-threads
[^1_29]: https://nodejs.org/download/release/v13.1.0/docs/api/worker_threads.html
[^1_30]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/worker_threads/
[^1_31]: https://dev.to/deepal/deep-dive-into-worker-threads-in-node-js-4bc8
[^1_32]: https://www.reddit.com/r/javascript/comments/z8mbmw/nodejs_multithreading_with_worker_threads_series/
[^1_33]: https://blog.logrocket.com/multithreading-node-js-worker-threads/
[^1_34]: https://www.simplilearn.com/tutorials/nodejs-tutorial/nodejs-worker-threads
[^1_35]: https://nodejs.org/api/worker_threads.html
[^1_36]: https://www.typeerror.org/docs/node~10_lts/worker_threads
[^1_37]: https://bun.com/reference/node/worker_threads
[^1_38]: https://onlinelibrary.wiley.com/doi/10.1002/itl2.137
[^1_39]: https://link.springer.com/10.1007/s00421-023-05298-x
[^1_40]: https://trialsjournal.biomedcentral.com/articles/10.1186/s13063-022-06708-9
[^1_41]: https://bpspsychub.onlinelibrary.wiley.com/doi/10.1111/lcrp.70010
[^1_42]: https://oarjst.com/node/701
[^1_43]: https://ijetcsit.org/index.php/ijetcsit/article/view/99
[^1_44]: https://www.nature.com/articles/s41372-022-01423-4
[^1_45]: http://link.springer.com/10.1007/978-3-319-29919-8_5
[^1_46]: http://ar.iiarjournals.org/lookup/doi/10.21873/anticanres.16597
[^1_47]: https://ieeexplore.ieee.org/document/10223568/
[^1_48]: https://arxiv.org/pdf/1507.01101.pdf
[^1_49]: https://arxiv.org/pdf/1908.05790.pdf
[^1_50]: https://arxiv.org/pdf/2202.11464.pdf
[^1_51]: https://arxiv.org/pdf/2109.05276.pdf
[^1_52]: https://arxiv.org/pdf/2105.07497.pdf
[^1_53]: https://arxiv.org/pdf/2212.04771v1.pdf
[^1_54]: http://arxiv.org/pdf/2407.11582.pdf
[^1_55]: https://arxiv.org/pdf/2310.02959.pdf
[^1_56]: https://snyk.io/blog/node-js-multithreading-with-worker-threads/
[^1_57]: https://github.com/orgs/nodejs/discussions/44264
[^1_58]: https://developers.redhat.com/articles/2025/10/10/nodejs-20-memory-management-containers
[^1_59]: https://www.nicholasadamou.com/notes/worker-threads-in-nodejs
[^1_60]: https://java-design-patterns.com/patterns/thread-pool-executor/
[^1_61]: https://www.linkedin.com/pulse/optimizing-memory-management-nodejs-high-traffic-applications-8cyke
[^1_62]: https://blog.nashtechglobal.com/mastering-kafka-client-patterns-part-3-thread-pool-pattern/
[^1_63]: https://www.reddit.com/r/node/comments/en3tlr/what_is_the_difference_between_child/
[^1_64]: https://www.alibabacloud.com/blog/java-thread-pool-implementation-and-best-practices-in-business-applications_601528
[^1_65]: https://stackoverflow.com/questions/56312692/what-is-the-difference-between-child-process-and-worker-threads
[^1_66]: https://dev.to/devopsfundamentals/nodejs-fundamentals-workerthreads-mld
[^1_67]: https://stackoverflow.com/questions/77392833/looking-for-a-design-pattern-to-get-node-js-worker-thread-results-back-into-a-st
[^1_68]: https://www.reddit.com/r/node/comments/11rahni/what_are_the_differences_between_clusters_worker/
[^1_69]: http://arxiv.org/pdf/1404.7548.pdf
[^1_70]: https://zenodo.org/record/200339/files/FPGA17paper.pdf
[^1_71]: https://arxiv.org/pdf/1309.2772.pdf
[^1_72]: https://arxiv.org/pdf/1602.05365.pdf
[^1_73]: http://arxiv.org/pdf/2401.12815.pdf
[^1_74]: https://arxiv.org/pdf/2401.09359.pdf
[^1_75]: http://arxiv.org/pdf/1911.01508.pdf
[^1_76]: http://arxiv.org/pdf/2410.01901.pdf
[^1_77]: https://sbfl.net/blog/2017/10/31/sharedarraybuffer-atomics/
[^1_78]: https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/node/worker_threads.d.ts
[^1_79]: https://www.reddit.com/r/node/comments/1687s6t/sharedarraybufferatomics_is_anyone_doing_anything/
[^1_80]: https://javascript.plainenglish.io/a-brief-guide-on-worker-threads-in-node-js-80bd1e7846cb
[^1_81]: https://blogtitle.github.io/using-javascript-sharedarraybuffers-and-atomics/
[^1_82]: https://leapcell.io/blog/unlocking-node-js-scalability-with-worker-threads
[^1_83]: https://docs.deno.com/api/node/worker_threads/~/MessagePort
[^1_84]: https://www.digitalocean.com/community/tutorials/how-to-use-multithreading-in-node-js
[^1_85]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer
[^1_86]: https://bun.com/reference/node/worker_threads/MessagePort/postMessage
[^1_87]: https://stackoverflow.com/questions/28177794/how-do-i-get-errors-to-the-console-from-node-webworker-threads
[^1_88]: https://github.com/nodejs/help/issues/5065
[^1_89]: https://stackoverflow.com/questions/60223441/workers-in-javascript-not-so-fast```


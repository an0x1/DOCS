
## 1. Module Overview (نظرة عامة)

- Brief introduction.
- When to use this module vs. external libraries?


## 2. API Deep Dive (تفاصيل الدوال والخصائص)

For *each* function/class in the module, provide:

- **Function Signature:** `module.function(param1, [param2])`
- **Description:** Simple, clear explanation of what it does.
- **Parameters Table:**


| Parameter | Type | Required/Optional | Description |
| :-- | :-- | :-- | :-- |

- **Return Value:** What does it return?
- **Real-World Example:** A code block showing a practical use case (with comments).
- **Common Pitfalls:** What mistakes do developers usually make here?


## 3. Comparison \& Decision Making (مقارنات)

- If the module has similar functions (e.g., `exec` vs `spawn` vs `fork`), create a comparison table:
    - Difference in functionality.
    - Performance implications.
    - When to use which?


## 4. Pros, Cons \& Use Cases (المزايا والعيوب والحالات)

- **Pros:** Why is this module powerful?
- **Cons:** Limitations or performance costs.
- **Top 3 Use Cases:** specific scenarios where this module is the best choice.


## 5. Visual Explanation (Optional)

- Create a Mermaid diagram if it helps explain the flow (e.g., Process communication, Event loop interaction).

---
**Start the documentation now for: [child_process]**

يُستخدم الموديول **`child_process`** لإنشاء عمليات فرعية (Child Processes) من داخل Node.js للتعامل مع أوامر النظام، السكربتات، وتوزيع الأحمال على أكثر من نواة CPU، مع دعم قوي للتواصل بين العمليات عبر stdin/stdout/IPC. يناسب هذا الموديول التطبيقات المتوسطة والمتقدمة التي تحتاج تنفيذ أوامر خارجية، تشغيل سكربتات منفصلة، أو عزل المهام الثقيلة عن الـ Event Loop الرئيسي.[^3_1][^3_2][^3_3][^3_4]

***

## 1. نظرة عامة على `child_process` (Module Overview)

- يوفر الموديول دوال لإنشاء عمليات فرعية: **`spawn`**, **`exec`**, **`execFile`**, **`fork`**، بالإضافة لنسخ متزامنة Sync، ودوال منخفضة المستوى مثل `spawnSync`.[^3_5][^3_1]
- كل عملية فرعية يتم تمثيلها بكائن **`ChildProcess`** الذي يعرّض أحداثًا (`exit`, `error`, `close`, `message`) وتيارات (`stdin`, `stdout`, `stderr`).[^3_6][^3_1]


### متى تستخدم `child_process` بدلاً من مكتبات خارجية؟

- عند تشغيل أوامر النظام (مثل `ffmpeg`, `git`, `tar`) بدلاً من استخدام حزم Node مخصصة لكل منها.[^3_4]
- عند تقسيم مهام CPU-intensive (مثل image processing أو report generation) إلى عمليات Node.js منفصلة باستخدام `fork` للاستفادة من تعدد الأنوية.[^3_3][^3_4]
- عند الحاجة لدمج برنامج/خدمة خارجية (legacy binary, Python script) داخل منظومة Node بدون إعادة كتابة المنطق.[^3_4]

***

## 2. API Deep Dive (تفاصيل الدوال والخصائص)

### 2.1 الدالة `spawn` – `child_process.spawn()`

**التوقيع:**

```js
const { spawn } = require('node:child_process');

const child = spawn(command, args?, options?);
```

**الوصف (Description):**
تستخدم الدالة `spawn` لإنشاء عملية جديدة وتشغيل أمر نظام أو برنامج خارجي مع **تيارات Stream حية** لـ `stdout` و `stderr`، دون إنشاء shell افتراضيًا.[^3_1][^3_3]

#### Parameters

| Parameter | Type | Required/Optional | Description |
| :-- | :-- | :-- | :-- |
| `command` | `string` | Required | اسم الأمر أو المسار الكامل للتنفيذ (مثل `ffmpeg`, `/usr/bin/node`).[^3_1] |
| `args` | `string[]` | Optional | قائمة الوسيطات التي تُمرر إلى البرنامج (مثل `['-i', 'input.mp4']`).[^3_1] |
| `options` | `Object` | Optional | كائن إعدادات متقدمة لسلوك العملية.[^3_1] |
| `options.cwd` | `string` | Optional | مسار مجلد العمل للـ child بدلاً من `process.cwd()`. |
| `options.env` | `Object` | Optional | متغيرات البيئة للـ child (يتم نسخ `process.env` عادة وتعديله). |
| `options.stdio` | `string` \| array | Optional | إعداد قنوات `stdin`, `stdout`, `stderr` (مثل `'pipe'`, `'inherit'`, `'ignore'`).[^3_1] |
| `options.shell` | `boolean`\|string | Optional | إذا true أو مسار shell: يتم تشغيل الأمر داخل shell (مثل `/bin/sh`).[^3_1] |
| `options.detached` | `boolean` | Optional | تشغيل العملية مستقلة عن الأب (daemon-like).[^3_7][^3_8] |
| `options.uid`/`gid` | `number` | Optional | تشغيل العملية كـ user/group ID معين (أنظمة Unix). |
| `options.signal` | `AbortSignal` | Optional | إلغاء العملية عبر `AbortController`.[^3_2] |
| `options.timeout` | `number` | Optional | حد أقصى بالملّي ثانية قبل إنهاء العملية (مع `killSignal`). |

#### Return Value

- ترجع كائن **`ChildProcess`** يحتوي على:
    - `pid` رقم العملية.
    - `stdin`, `stdout`, `stderr` كـ Streams (إذا كانت `stdio` تسمح بذلك).
    - دوال مثل `kill()` وخصائص مثل `exitCode`, `signalCode`.[^3_6][^3_1]


#### Real-World Example – تشغيل `ffmpeg` لمعالجة فيديو

```js
const { spawn } = require('node:child_process');
const path = require('node:path');

function transcodeVideo(inputPath, outputPath) {
  return new Promise((resolve, reject) => {
    const ffmpegPath = 'ffmpeg'; // يفترض أن ffmpeg مثبت في PATH

    const args = [
      '-y',             // overwrite output
      '-i', inputPath,  // input file
      '-c:v', 'libx264',
      '-preset', 'fast',
      '-crf', '23',
      '-c:a', 'aac',
      outputPath,
    ];

    const child = spawn(ffmpegPath, args, {
      stdio: ['ignore', 'pipe', 'pipe'],
    });

    child.stdout.on('data', (chunk) => {
      // بعض إصدارات ffmpeg لا تستخدم stdout كثيراً
      console.log('ffmpeg stdout:', chunk.toString());
    });

    child.stderr.on('data', (chunk) => {
      // ffmpeg يكتب progress على stderr
      const message = chunk.toString();
      if (message.includes('time=')) {
        console.log('Progress:', message.trim());
      }
    });

    child.on('error', (err) => {
      reject(err);
    });

    child.on('close', (code, signal) => {
      if (code === 0) {
        resolve({ success: true, outputPath });
      } else {
        reject(
          new Error(`ffmpeg failed with code ${code} (signal: ${signal})`),
        );
      }
    });
  });
}

// الاستخدام
(async () => {
  try {
    const input = path.join(__dirname, 'uploads', 'input.mp4');
    const output = path.join(__dirname, 'processed', 'output_720p.mp4');
    const result = await transcodeVideo(input, output);
    console.log('✅ Video processed:', result.outputPath);
  } catch (err) {
    console.error('❌ Video processing failed:', err.message);
  }
})();
```


#### Common Pitfalls (أخطاء شائعة)

- نسيان قراءة `stdout`/`stderr` في العمليات التي تنتج إخراجًا ضخمًا مما قد يسبب امتلاء buffer وتوقف العملية.[^3_3][^3_4]
- استخدام `shell: true` مع مدخلات غير موثوقة (Risk of command injection).
- عدم التعامل مع حدث `error` (مثلاً لو لم يتم العثور على البرنامج).

***

### 2.2 الدالة `exec` – `child_process.exec()`

**التوقيع:**

```js
const { exec } = require('node:child_process');

const child = exec(command, options?, callback?);
```

**الوصف:**
تستخدم `exec` لتنفيذ أمر في **shell**، وتقوم بتجميع كامل الإخراج في buffer (نص) ثم تمريره إلى callback عند الانتهاء؛ مناسبة للأوامر القصيرة ذات مخرجات محدودة.[^3_9][^3_1]

#### Parameters

| Parameter | Type | Required/Optional | Description |
| :-- | :-- | :-- | :-- |
| `command` | `string` | Required | سلسلة الأمر كما تكتب في shell (مثال: `'ls -la /var/log'`).[^3_9] |
| `options` | `Object` | Optional | إعدادات التنفيذ. |
| `options.cwd` | `string` | Optional | مجلد العمل. |
| `options.env` | `Object` | Optional | متغيرات البيئة. |
| `options.timeout` | `number` | Optional | حد أقصى للوقت بالملّي ثانية قبل إنهاء العملية. |
| `options.maxBuffer` | `number` | Optional | أقصى حجم للإخراج بالكيلوبايت تقريبًا (افتراضي ~1MB). |
| `options.shell` | `string` | Optional | shell مخصص مثل `/bin/bash`. |
| `options.signal` | `AbortSignal` | Optional | إلغاء الأمر. |
| `callback` | `(error, stdout, stderr) => void` | Optional | callback يتم استدعاؤه بعد انتهاء العملية. |

#### Return Value

- كائن `ChildProcess` + في حالة استخدام callback، يتم تمرير:
    - `error`: يحتوي على `code` و `signal` عند الفشل.
    - `stdout`: نص كامل للإخراج القياسي.
    - `stderr`: نص كامل للأخطاء.[^3_2][^3_9]


#### Real-World Example – فحص حالة خدمة نظامية

```js
const { exec } = require('node:child_process');

function checkNginxStatus() {
  return new Promise((resolve, reject) => {
    exec('systemctl is-active nginx', { timeout: 3000 }, (error, stdout, stderr) => {
      if (error) {
        if (error.code === 3) {
          // systemctl is-active يعيد 3 إذا كانت الخدمة غير فعّالة
          resolve({ active: false, rawOutput: stdout.trim() });
        } else {
          return reject(error);
        }
      }
      resolve({ active: stdout.trim() === 'active', rawOutput: stdout.trim() });
    });
  });
}

// الاستخدام
(async () => {
  const status = await checkNginxStatus();
  console.log('Nginx active?', status.active);
})();
```


#### Common Pitfalls

- نسيان ضبط `maxBuffer` عند توقع إخراج كبير مما يسبب خطأ `ERR_CHILD_PROCESS_STDIO_MAXBUFFER`.[^3_4]
- استخدام `exec` لأوامر ذات إخراج ضخم بدلاً من `spawn` (مكلف في الذاكرة).
- إدخال بيانات مباشرة من المستخدم في `command` → ثغرات command injection (يفضل `spawn` أو `execFile`).

***

### 2.3 الدالة `execFile` – `child_process.execFile()`

**التوقيع:**

```js
const { execFile } = require('node:child_process');

const child = execFile(file, args?, options?, callback?);
```

**الوصف:**
تشبه `exec` لكنها **لا تنشئ shell**، بل تستدعي ملف تنفيذي مباشرة مع arguments؛ أكثر أمانًا عند تمرير مدخلات المستخدم.[^3_2][^3_4]

#### Parameters

| Parameter | Type | Required/Optional | Description |
| :-- | :-- | :-- | :-- |
| `file` | `string` | Required | مسار الملف التنفيذي (مثل `/usr/bin/convert`). |
| `args` | `string[]` | Optional | الوسيطات. |
| `options` | `Object` | Optional | مثل `exec` (cwd, env, timeout, maxBuffer, signal). |
| `callback` | `(error, stdout, stderr) => void` | Optional | نتيجة التنفيذ. |

#### Return Value

- `ChildProcess` + callback بنفس نمط `exec`.[^3_2]


#### Real-World Example – استخدام `ImageMagick` لمعالجة الصور

```js
const { execFile } = require('node:child_process');
const path = require('node:path');

function generateThumbnail(inputPath, outputPath) {
  return new Promise((resolve, reject) => {
    const convertPath = 'convert'; // ImageMagick

    const args = [
      inputPath,
      '-resize', '300x300',
      '-strip',
      outputPath,
    ];

    execFile(convertPath, args, { timeout: 15000 }, (error, stdout, stderr) => {
      if (error) {
        return reject(error);
      }
      resolve({ success: true, outputPath });
    });
  });
}
```


#### Common Pitfalls

- افتراض أن `PATH` يحتوي على التنفيذية؛ يجب أحيانًا تحديد المسار الكامل.
- نسيان ضبط `timeout` عند تنفيذ برامج خارجية قد تتجمّد.

***

### 2.4 الدالة `fork` – `child_process.fork()`

**التوقيع:**

```js
const { fork } = require('node:child_process');

const child = fork(modulePath, args?, options?);
```

**الوصف:**
تستخدم `fork` لإنشاء **عملية Node.js فرعية** وتشغيل ملف JavaScript محدد، مع قناة **IPC** مدمجة للتواصل عبر `process.send()` وحدث `message`؛ مثالية لتوزيع الأعمال الثقيلة على عدّة عمليات.[^3_5][^3_3][^3_2]

#### Parameters

| Parameter | Type | Required/Optional | Description |
| :-- | :-- | :-- | :-- |
| `modulePath` | `string` | Required | مسار ملف `.js` الذي سيتم تشغيله في العملية الفرعية. |
| `args` | `string[]` | Optional | Arguments يتم تمريرها إلى child عبر `process.argv`. |
| `options` | `Object` | Optional | إعدادات التفرع. |
| `options.cwd` | `string` | Optional | مجلد العمل للـ child. |
| `options.env` | `Object` | Optional | متغيرات البيئة. |
| `options.execPath` | `string` | Optional | مسار `node` المستخدم بدلاً من `process.execPath`.[^3_2][^3_8] |
| `options.execArgv` | `string[]` | Optional | وسيطات إضافية لمحرك Node (مثل `['--inspect=9229']`). |
| `options.silent` | `boolean` | Optional | إذا true، تكون `stdout` و`stderr` pipes بدلاً من الوراثة. |
| `options.signal` | `AbortSignal` | Optional | إلغاء child. |

#### Return Value

- كائن **`ChildProcess`** مع قناة IPC:
    - `child.send(message[, sendHandle][, options][, callback])`
    - `child.on('message', handler)`
    - `process.on('message', handler)` داخل child.[^3_8][^3_2]


#### Real-World Example – توزيع حسابات ثقيلة على عمليات فرعية

**ملف السيرفر `server.js`:**

```js
const http = require('node:http');
const { fork } = require('node:child_process');

const server = http.createServer((req, res) => {
  if (req.url === '/report') {
    // إنشاء عملية فرعية لتوليد تقرير كبير
    const reportWorker = fork('./workers/generateReport.js', [], {
      env: { NODE_ENV: 'production' },
    });

    reportWorker.send({ type: 'START', payload: { userId: 42 } });

    reportWorker.on('message', (msg) => {
      if (msg.type === 'PROGRESS') {
        console.log('Report progress:', msg.value);
      }
      if (msg.type === 'DONE') {
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ reportUrl: msg.reportUrl }));
        reportWorker.disconnect();
      }
    });

    reportWorker.on('error', (err) => {
      console.error('Worker error:', err);
      res.statusCode = 500;
      res.end('Report failed');
    });

    reportWorker.on('exit', (code) => {
      console.log('Worker exited with code', code);
    });
  } else {
    res.end('OK');
  }
});

server.listen(3000);
```

**ملف العامل `workers/generateReport.js`:**

```js
const path = require('node:path');
const fs = require('node:fs').promises;

process.on('message', async (msg) => {
  if (msg.type === 'START') {
    const { userId } = msg.payload;

    try {
      process.send({ type: 'PROGRESS', value: 10 });

      // محاكاة توليد تقرير ثقيل
      const reportData = await buildUserReport(userId);

      process.send({ type: 'PROGRESS', value: 80 });

      const reportPath = path.join(__dirname, '../reports', `user-${userId}.json`);
      await fs.mkdir(path.dirname(reportPath), { recursive: true });
      await fs.writeFile(reportPath, JSON.stringify(reportData, null, 2));

      process.send({ type: 'DONE', reportUrl: `/reports/user-${userId}.json` });
    } catch (err) {
      process.send({ type: 'ERROR', message: err.message });
      process.exit(1);
    }
  }
});

async function buildUserReport(userId) {
  // عمليات CPU-intensive (تجميع بيانات، تحليل، إلخ)
  return {
    userId,
    generatedAt: new Date().toISOString(),
    stats: { orders: 120, totalSpent: 5400 },
  };
}
```


#### Common Pitfalls

- نسيان استدعاء `child.disconnect()` عند انتهاء التواصل مما يطيل عمر العملية.
- إرسال كائنات كبيرة جداً عبر IPC مما قد يؤثر على الأداء (يفضل إرسال معرفات وإعادة جلب البيانات من مصدر مشترك).

***

### 2.5 الكلاس `ChildProcess`

يتم إرجاعه من جميع الدوال: `spawn`, `exec`, `execFile`, `fork`.[^3_1][^3_6]

#### أهم الخصائص

- **`pid`**: رقم العملية.
- **`stdin` / `stdout` / `stderr`**: Streams قابلة للقراءة/الكتابة حسب إعداد `stdio`.
- **`connected`**: هل قناة IPC ما زالت متصلة (في حالة `fork`).
- **`exitCode`**: كود الخروج بعد انتهاء العملية.
- **`signalCode`**: الإشارة التي أنهت العملية إن وجدت.


#### أهم الأحداث

| Event | Description |
| :-- | :-- |
| `spawn` | يتم إطلاقه مباشرة بعد إنشاء العملية بنجاح. |
| `error` | فشل في الإنشاء (مثل أمر غير موجود أو أذونات غير كافية). |
| `exit` | عند خروج العملية (قبل إغلاق streams). |
| `close` | بعد إغلاق جميع الـ stdio. |
| `message` | رسائل IPC في عمليات `fork`. |

#### أهم الدوال

- **`child.kill([signal])`**: إرسال إشارة (مثل `SIGTERM`, `SIGKILL`).
- **`child.send(message, [...])`**: إرسال رسالة IPC في حالة `fork`.
- **`child.disconnect()`**: غلق قناة IPC.

***

## 3. Comparison \& Decision Making (مقارنات)

### 3.1 جدول مقارنة `spawn` vs `exec` vs `execFile` vs `fork`

| Method | Shell? | Output Handling | IPC Support | Best Use Case | Memory Efficiency | Performance |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| `spawn()` | No (optional shell) | Streams | No | أوامر طويلة/إخراج كبير (مثل `tail -f`, `ffmpeg`).[^3_3][^3_4] | عالي جدًا (streaming) | ممتاز I/O |
| `exec()` | Yes | Single Buffer | No | أوامر shell قصيرة ذات إخراج صغير (مثل `ls`, `whoami`).[^3_4][^3_9] | محدود بـ `maxBuffer` | مريح لكن محدود |
| `execFile()` | No | Single Buffer | No | تشغيل تنفيذيات آمنة بدون shell (image processing).[^3_4] | محدود بـ `maxBuffer` | أسرع من exec |
| `fork()` | No (Node.js) | Streams + IPC | Yes | توزيع مهام Node.js CPU-heavy مع message passing.[^3_3][^3_10][^3_4] | جيد، عملية كاملة لكل child | ممتاز لـ CPU |

![مقارنة شاملة بين طرق child_process المختلفة](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/39de4e20fbfc2e49eb0f49008de9d120/4febfede-e9e4-43e3-9924-3a9f90a92a33/e57db8ce.png)

مقارنة شاملة بين طرق child_process المختلفة

### 3.2 متى تستخدم أي دالة؟

- استخدم **`spawn`** عندما:
    - تحتاج إلى التعامل مع تيار إخراج مستمر أو كبير.
    - لا تحتاج إلى shell features (pipes, redirection) إلا لو فعلت `shell: true`.
- استخدم **`exec`** عندما:
    - تحتاج أمر shell بسيط والنتيجة نص صغير ومرة واحدة.
    - تحتاج features shell مثل `&&` أو `|`.
- استخدم **`execFile`** عندما:
    - لديك تنفيذية معروفة وتريد تجنب مخاطر shell.
    - تمرر arguments آمنة (مثل مسارات ملفات من المستخدم).
- استخدم **`fork`** عندما:
    - تريد تشغيل سكربتات Node.js أخرى.
    - تحتاج قناة IPC لتبادل رسائل JSON.
    - تريد توزيع الحمل على عدة أنوية مع نموذج worker.

***

## 4. Pros, Cons \& Use Cases (المزايا والعيوب والحالات)

### Pros (المزايا)

- يسمح بعزل المهام الثقيلة في عمليات مستقلة مما يحمي الـ Event Loop من الانسداد.[^3_3][^3_4]
- تكامل ممتاز مع النظام: تشغيل أدوات موجودة (ffmpeg, ImageMagick, git, Python).
- `fork` يوفر قناة IPC جاهزة لتصميم معماريات multi-process (cluster-like).[^3_5][^3_10]


### Cons (العيوب)

- إدارة العمليات الفرعية تزيد من تعقيد الكود (lifecycle, error handling, retries).
- لكل عملية Node جديدة هناك overhead في الذاكرة والـ startup time.
- تزامن state بين عمليات متعددة يحتاج تصميم جيد (Redis, DB, message queues).


### Top 3 Use Cases (أهم 3 حالات استخدام)

1. **توزيع المهام CPU-intensive**
    - مثال: توليد تقارير PDF أو معالجة صور عالية الدقة باستخدام `fork` لكل مهمة.[^3_4]
2. **دمج أدوات CLI في الـ backend**
    - مثال: استدعاء `ffmpeg`, `sox`, `convert`, `git` عبر `spawn` أو `execFile` ضمن API.[^3_3][^3_4]
3. **بناء معماريات multi-process**
    - مثال: إنشاء pool من workers باستخدام `fork` لمعالجة طوابير رسائل (queue) من RabbitMQ أو Kafka.

***

## 5. Visual Explanation (Mermaid Diagram)

### 5.1 دورة حياة عملية فرعية مع قنوات الاتصال

```mermaid
flowchart LR
    A[Parent Process (Node.js)] -->|spawn / exec / execFile / fork| B[Child Process]

    subgraph IO [STDIO Streams]
      A <-->|stdin| B
      A <-->|stdout| B
      A <-->|stderr| B
    end

    subgraph IPC [IPC Channel (fork only)]
      A <-->|message / send()| B
    end

    A -->|listen| C[Events: 'spawn', 'error', 'exit', 'close']
    B -->|exit code / signal| A
```

![Node.js Child Process Communication Flow - دورة حياة العمليات الفرعية](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/39de4e20fbfc2e49eb0f49008de9d120/2b9b464b-ea51-40ce-b5c5-de95494b25df/61b107f7.png)

Node.js Child Process Communication Flow - دورة حياة العمليات الفرعية

هذا المخطط يوضح كيف ينشئ الـ Parent عملية فرعية، وكيف يتم تمرير البيانات عبر `stdin/stdout/stderr`، وكيف يتم استخدام IPC في `fork` لتبادل الرسائل.[^3_1][^3_8]

***

إذا رغبت في قسم إضافي يغطي **النسخ المتزامنة Sync APIs** (`execSync`, `spawnSync`) مع تحذيرات استخدام في الإنتاج، يمكن إضافته كملحق متقدم.
<span style="display:none">[^3_11][^3_12][^3_13][^3_14][^3_15][^3_16][^3_17][^3_18][^3_19][^3_20][^3_21][^3_22][^3_23][^3_24][^3_25][^3_26][^3_27][^3_28]</span>

<div align="center">⁂</div>

[^3_1]: https://nodejs.org/api/child_process.html

[^3_2]: https://nodejs.org/docs/latest/api/child_process.html

[^3_3]: https://zaiste.net/posts/nodejs-child-process-spawn-exec-fork-async-await/

[^3_4]: https://devcrud.com/node-js-child_process-module-executing-system-commands-with-spawn-exec-fork-and-execfile/

[^3_5]: https://github.com/nodejs/node/blob/main/doc/api/child_process.md

[^3_6]: https://node.readthedocs.io/en/stable/api/child_process/

[^3_7]: https://nodejs.cn/api-v18/child_process/options_detached.html

[^3_8]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/child_process/

[^3_9]: https://wiki.hsoub.com/Node.js/child_process

[^3_10]: https://www.geeksforgeeks.org/node-js/difference-between-spawn-and-fork-methods-in-node-js/

[^3_11]: https://arxiv.org/pdf/2101.00756.pdf

[^3_12]: https://arxiv.org/pdf/2110.14162.pdf

[^3_13]: http://arxiv.org/pdf/1704.07887.pdf

[^3_14]: https://arxiv.org/pdf/2403.14940.pdf

[^3_15]: https://arxiv.org/pdf/2107.10164.pdf

[^3_16]: https://arxiv.org/pdf/2203.13737.pdf

[^3_17]: https://arxiv.org/pdf/1801.06144.pdf

[^3_18]: https://arxiv.org/pdf/1512.07067.pdf

[^3_19]: https://www.freecodecamp.org/news/node-js-child-processes-everything-you-need-to-know-e69498fe970a/

[^3_20]: https://www.c-sharpcorner.com/article/what-is-the-difference-between-spawn-exec-and-fork-in-node-js/

[^3_21]: https://dancon.gitbooks.io/node-js-api/api/child_process.html

[^3_22]: https://stackoverflow.com/questions/59646654/detach-a-spawn-child-process-after-the-start

[^3_23]: https://www.w3schools.com/nodejs/nodejs_child_process.asp

[^3_24]: https://iojs.org/api/child_process.html

[^3_25]: https://devcrud.com/node-js-spawn-exec-fork-and-execfile-executing-external-commands/

[^3_26]: https://www.geeksforgeeks.org/node-js/node-js-child-process/

[^3_27]: https://job.js.org/questions/nodejs/spawn-vs-fork/

[^3_28]: https://bun.com/reference/node/child_process/spawn


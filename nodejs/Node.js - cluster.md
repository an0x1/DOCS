Now I have sufficient information to create the comprehensive documentation. Let me generate the final report.

# موثقة شاملة عميقة لوحدة Node.js - cluster

**مقدمة عامة:** وحدة `cluster` في Node.js توفر آلية قوية لإنشاء واستدارة عمليات متعددة على نفس الجهاز، مما يسمح بالاستفادة من معالجات متعددة النوى (multi-core processors) وتوزيع الأحمال بكفاءة. الوحدة مدمجة في النواة وتستخدم آلية `child_process.fork()` لإنشاء عمليات عاملة (workers) متصلة عبر قنوات IPC (Inter-Process Communication)، مما يمكّن التطبيقات من معالجة آلاف الطلبات المتزامنة بسهولة وفعالية.[^1_1][^1_2]

## نظرة عامة على وحدة Cluster

### مقدمة

وحدة `cluster` توفر تجميع (clustering) من عمليات Node.js يمكنها توزيع الأحمال الحاسوبية بين خيوطها (threads) من التطبيق. عندما لا يكون عزل العملية مطلوباً، يُنصح باستخدام وحدة `worker_threads` بدلاً منها، التي تسمح بتشغيل خيوط تطبيق متعددة داخل مثيل Node.js واحد.[^1_2][^1_1]

تسمح وحدة `cluster` بإنشاء سهل لعمليات فرعية تشترك جميعها في منافذ الخادم. هذا يعني أن العملية الأساسية (primary process) تستمع على منفذ معين، وتقبل الاتصالات الواردة، ثم توزعها على العمليات العاملة (worker processes) بطريقة معادلة. جميع العمليات العاملة تستمع على نفس المنفذ.[^1_3][^1_4]

### متى تستخدم وحدة Cluster؟

استخدم وحدة `cluster` في الحالات التالية:

- **تطبيقات الويب متعددة الطلبات:** عندما تحتاج تطبيقات الويب الخاصة بك إلى معالجة آلاف الطلبات المتزامنة، توزيع الحمل على عدة عمليات يحسن الأداء بشكل كبير.[^1_5][^1_6]
- **الاستفادة من معالجات متعددة النوى:** إذا كان لديك جهاز بعدة نوى معالج (CPU cores)، يمكن لوحدة `cluster` إنشاء عملية منفصلة لكل نواة.[^1_2]
- **تحسين الموثوقية:** إذا تعطلت عملية عاملة، يمكن للعملية الأساسية إعادة تشغيلها تلقائياً.[^1_6][^1_5]
- **جدولة مهام CPU-intensive:** بدلاً من استخدام `worker_threads` للمهام ثقيلة الحساب، يمكن استخدام `cluster` للعزل الكامل بين العمليات.[^1_2]

**البدائل:**

- **worker_threads:** للعمليات الخفيفة الوزن ضمن عملية واحدة (single process)
- **PM2 أو Forever:** لإدارة العمليات في الإنتاج (production)
- **Kubernetes أو Docker:** للتوسع الأفقي (horizontal scaling) عبر عدة أجهزة

---

## كيفية عمل وحدة Cluster

### معمارية النظام

تُنشأ عمليات العاملين (worker processes) باستخدام دالة `child_process.fork()`، مما يسمح لهم بالتواصل مع العملية الأساسية عبر IPC وتمرير مقابض الخادم (server handles) ذهاباً وإياباً.[^1_1][^1_2]

تدعم وحدة `cluster` طريقتين لتوزيع الاتصالات الواردة:

1. **Round-Robin (الإعادة الدورية):** الطريقة الافتراضية على جميع الأنظمة ما عدا Windows. تستمع العملية الأساسية على منفذ، تقبل الاتصالات الجديدة، وتوزعها على العمليات العاملة بطريقة دورية. هذا يضمن توزيع متوازن للأحمال.[^1_4][^1_5][^1_1][^1_2]
2. **SCHED_NONE (عدم الجدولة):** تنشئ العملية الأساسية socket الاستماع وترسله إلى العمليات العاملة المهتمة. تقبل العمليات العاملة الاتصالات الواردة مباشرة. هذه الطريقة قد تعطي أداء أفضل نظرياً، لكن في الممارسة العملية قد يؤدي إلى عدم التوازن في التوزيع.[^1_1][^1_2]

### نقاط تقنية مهمة

- **server.listen({fd: 7}):** يتم تمرير الرسالة إلى العملية الأساسية، لذلك سيتم الاستماع على واصف الملف (file descriptor) 7 في الوالد (parent)، وليس في العملية العاملة.[^1_1][^1_2]
- **server.listen(handle):** سيؤدي الاستماع على مقابض صريحة إلى استخدام العملية العاملة للمقبض المسموح، بدلاً من التحدث إلى العملية الأساسية.[^1_2][^1_1]
- **server.listen(0):** عادة يسبب الخادم الاستماع على منفذ عشوائي. لكن في cluster، كل عملية عاملة ستستقبل نفس المنفذ "العشوائي". لحل هذا، توليد رقم منفذ بناءً على معرّف العملية العاملة (worker ID).[^1_1][^1_2]

---

## تفاصيل الدوال والخصائص (API Deep Dive)

### Class: Worker

تمثل فئة `Worker` عملية عاملة بكل البيانات العامة والطرق المتعلقة بها. في العملية الأساسية، يمكن الحصول عليها عبر `cluster.workers`. في عملية عاملة، يمكن الحصول عليها عبر `cluster.worker`.[^1_2][^1_1]

#### Property: worker.id

**التوقيع:** `{integer}`

**الوصف:** كل عملية عاملة جديدة تُعطى معرّفاً (ID) فريداً خاصاً بها. هذا المعرّف يُخزّن في الخاصية `id`. طالما العملية العاملة حية، هذا هو المفتاح الذي يُفهرسها في `cluster.workers`.[^1_2]

| المعامل | النوع | مطلوب/اختياري | الوصف                             |
| :------ | :---- | :------------ | :-------------------------------- |
| -       | -     | -             | هذه خاصية قراءة فقط، بدون معاملات |

**القيمة المرجعة:** عدد صحيح يمثل معرّف العملية العاملة الفريد

**مثال من الواقع:**

```javascript
// تطبيق لإدارة محركات البحث بتوزيع مهام الفهرسة
const cluster = require("node:cluster");
const os = require("node:os");

if (cluster.isPrimary) {
  const numCPUs = os.availableParallelism();
  console.log(`Primary ${process.pid} forking ${numCPUs} workers`);

  // إنشاء عاملين لكل نواة معالج
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // تتبع العمليات العاملة
  for (const id in cluster.workers) {
    console.log(`Worker ${id} is running`);
  }
} else {
  // كل عملية عاملة تستخدم معرّفها لتحديد نطاق مهامها
  const workerId = cluster.worker.id;
  const documentsPerWorker = 1000;
  const startIdx = (workerId - 1) * documentsPerWorker;

  console.log(
    `Worker ${workerId} indexing documents ${startIdx} to ${
      startIdx + documentsPerWorker - 1
    }`
  );

  // محاكاة عملية فهرسة ثقيلة
  for (let i = startIdx; i < startIdx + documentsPerWorker; i++) {
    // فهرسة المستند
  }
}
```

**أخطاء شائعة:**

- محاولة الوصول إلى `cluster.worker.id` في العملية الأساسية (سيؤدي إلى `undefined`)
- الافتراض أن معرّفات العمليات العاملة متتالية تماماً (قد لا تكون كذلك في حالة إعادة التشغيل)

---

#### Method: worker.isConnected()

**التوقيع:** `worker.isConnected()`

**الوصف:** تُرجع هذه الدالة `true` إذا كانت العملية العاملة متصلة بعملية أساسية عبر قناة IPC الخاصة بها، وإلا تُرجع `false`. تتصل العملية العاملة بالعملية الأساسية بعد إنشاؤها. تنقطع بعد إصدار حدث `'disconnect'`.[^1_2]

| المعامل | النوع | مطلوب/اختياري | الوصف           |
| :------ | :---- | :------------ | :-------------- |
| -       | -     | -             | لا توجد معاملات |

**القيمة المرجعة:** boolean (true إذا كانت متصلة، false خلاف ذلك)

**مثال من الواقع:**

```javascript
// نظام مراقبة صحة العمليات العاملة
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  const workers = {};

  // إنشاء عمليات عاملة
  for (let i = 0; i < 2; i++) {
    const worker = cluster.fork();
    workers[worker.id] = worker;
  }

  // مراقبة الاتصالات كل 5 ثوانٍ
  setInterval(() => {
    console.log("Worker connection status:");
    for (const id in workers) {
      const connected = workers[id].isConnected();
      console.log(
        `  Worker ${id}: ${connected ? "connected" : "disconnected"}`
      );
    }
  }, 5000);
} else {
  // عملية عاملة بسيطة
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end("Worker response\n");
    })
    .listen(8000);
}
```

**أخطاء شائعة:**

- استخدام هذه الدالة قبل التأكد من أن العملية العاملة تم إنشاؤها بنجاح

---

#### Method: worker.isDead()

**التوقيع:** `worker.isDead()`

**الوصف:** تُرجع هذه الدالة `true` إذا كانت عملية العامل قد انتهت (إما بسبب خروج طبيعي أو بسبب إشارة). وإلا تُرجع `false`. إذا لم تنتهِ العملية العاملة بعد، فإن الدالة تُرجع `undefined`.[^1_2]

| المعامل | النوع | مطلوب/اختياري | الوصف           |
| :------ | :---- | :------------ | :-------------- |
| -       | -     | -             | لا توجد معاملات |

**القيمة المرجعة:** boolean أو undefined

**مثال من الواقع:**

```javascript
// تطبيق لاكتشاف وإعادة تشغيل العمليات العاملة الميتة
const cluster = require("node:cluster");
const http = require("node:http");
const os = require("node:os");

if (cluster.isPrimary) {
  const numWorkers = os.availableParallelism();

  // إنشاء عمليات عاملة
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  // فحص صحة العمليات العاملة كل 10 ثوانٍ
  setInterval(() => {
    for (const id in cluster.workers) {
      const worker = cluster.workers[id];
      if (worker.isDead()) {
        console.log(`Worker ${id} is dead. Restarting...`);
        cluster.fork(); // إعادة تشغيل عملية عاملة جديدة
      }
    }
  }, 10000);
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end(`Worker ${cluster.worker.id} response\n`);
    })
    .listen(8000);
}
```

**أخطاء شائعة:**

- الاعتماد على هذه الدالة وحدها للكشف عن الأعطال بدلاً من الاستماع إلى حدث `'exit'`

---

#### Method: worker.kill([signal])

**التوقيع:** `worker.kill([signal])`

| المعامل | النوع  | مطلوب/اختياري | الوصف                                                  |
| :------ | :----- | :------------ | :----------------------------------------------------- |
| signal  | string | اختياري       | اسم إشارة القتل المراد إرسالها. الافتراضي: `'SIGTERM'` |

**الوصف:** تقتل هذه الدالة العملية العاملة. في العملية الأساسية، تفعل ذلك بقطع اتصال `worker.process`، وبعد القطع، تقتل باستخدام الإشارة. في العملية العاملة، تقتل العملية بالإشارة. لا تنتظر دالة `kill()` قطع الاتصال الكامل؛ لها نفس السلوك كـ `worker.process.kill()`.[^1_2]

هذه الطريقة لها بديل: `worker.destroy()` للتوافق مع الإصدارات الأقدم.[^1_2]

**القيمة المرجعة:** Worker (مرجع إلى الكائن worker نفسه)

**مثال من الواقع:**

```javascript
// نظام لإيقاف العمليات العاملة بشكل آمن
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  const worker = cluster.fork();

  // إيقاف العملية العاملة بعد 30 ثانية
  setTimeout(() => {
    console.log("Sending SIGTERM to worker");
    worker.kill("SIGTERM"); // أو worker.destroy()
  }, 30000);

  worker.on("exit", (code, signal) => {
    if (signal) {
      console.log(`Worker killed by signal: ${signal}`);
    } else if (code !== 0) {
      console.log(`Worker exited with code: ${code}`);
    }
  });
} else {
  // عملية عاملة تتعامل مع الإشارات
  process.on("SIGTERM", () => {
    console.log("Worker received SIGTERM, closing server gracefully");
    server.close(() => {
      process.exit(0);
    });
  });

  const server = http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end("Worker response\n");
    })
    .listen(8000);
}
```

**أخطاء شائعة:**

- عدم معالجة إشارات الإيقاف في العملية العاملة، مما قد يؤدي إلى فقدان البيانات
- استخدام SIGKILL بدلاً من SIGTERM (SIGKILL لا يسمح بالإيقاف الآمن)

---

#### Method: worker.disconnect()

**التوقيع:** `worker.disconnect()`

**الوصف:** في عملية عاملة، تغلق هذه الدالة جميع الخوادم، تنتظر حدث `'close'` على تلك الخوادم، ثم تقطع قناة IPC. في العملية الأساسية، يتم إرسال رسالة داخلية إلى العملية العاملة تسبب استدعاء `.disconnect()` على نفسها. يسبب قطع الاتصال تعيين خاصية `.exitedAfterDisconnect`.[^1_2]

بعد إغلاق الخادم، لن يقبل اتصالات جديدة، لكن قد تُقبل الاتصالات من قِبل أي عملية عاملة أخرى تستمع. ستُسمح الاتصالات الموجودة بالإغلاق بشكل طبيعي.[^1_2]

**القيمة المرجعة:** Worker (مرجع إلى الكائن worker)

**مثال من الواقع:**

```javascript
// تطبيق يتعامل مع الإيقاف الآمن مع انتظار الاتصالات الحالية
const cluster = require("node:cluster");
const net = require("node:net");
const os = require("node:os");

if (cluster.isPrimary) {
  const numWorkers = os.availableParallelism();

  // إنشاء عمليات عاملة
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  // معالج لإيقاف الخادم بشكل آمن
  process.on("SIGTERM", () => {
    console.log(
      "Primary received SIGTERM. Disconnecting workers gracefully..."
    );

    for (const id in cluster.workers) {
      const worker = cluster.workers[id];
      worker.disconnect(); // بدء قطع الاتصال

      // إذا لم ينقطع الاتصال خلال 5 ثوانٍ، فاقتله
      const timeout = setTimeout(() => {
        worker.kill();
      }, 5000);

      worker.on("disconnect", () => {
        clearTimeout(timeout);
        console.log(`Worker ${id} disconnected`);
      });
    }
  });
} else {
  // عملية عاملة بخادم TCP
  const server = net.createServer((socket) => {
    // الاتصالات الطويلة قد تمنع قطع الاتصال
    socket.write("Connected to worker\n");
  });

  server.listen(8000);

  // معالج لرسائل قطع الاتصال
  process.on("message", (msg) => {
    if (msg === "shutdown") {
      console.log("Worker received shutdown message");
      // أغلق الاتصالات النشطة هنا
    }
  });
}
```

**أخطاء شائعة:**

- عدم تنفيذ آلية مهلة زمنية للعمليات التي لا تقطع الاتصال في الوقت المناسب
- عدم إغلاق الاتصالات العميلة بشكل صريح قبل قطع الاتصال

---

#### Property: worker.exitedAfterDisconnect

**النوع:** boolean

**الوصف:** هذه الخاصية تكون `true` إذا خرجت العملية العاملة بسبب `.disconnect()`. إذا خرجت العملية العاملة بأي طريقة أخرى، تكون `false`. إذا لم تخرج العملية العاملة بعد، تكون `undefined`. تسمح الخاصية `worker.exitedAfterDisconnect` بالتمييز بين الخروج الطوعي والعرضي، حيث قد تختار العملية الأساسية عدم إعادة تشغيل عملية عاملة بناءً على هذه القيمة.[^1_2]

**مثال من الواقع:**

```javascript
// إدارة ذكية لإعادة تشغيل العمليات العاملة
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  // إنشاء عملية عاملة
  const worker = cluster.fork();

  cluster.on("exit", (worker, code, signal) => {
    if (worker.exitedAfterDisconnect === true) {
      // هذا خروج متعمد - لا تعيد التشغيل
      console.log("Worker exited voluntarily - no restart needed");
    } else {
      // هذا خروج غير متوقع - أعد التشغيل
      console.log("Worker crashed - restarting...");
      cluster.fork();
    }
  });

  // إيقاف متعمد بعد 60 ثانية
  setTimeout(() => {
    console.log("Gracefully disconnecting worker");
    worker.disconnect();
  }, 60000);
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end("Worker response\n");
    })
    .listen(8000);
}
```

**أخطاء شائعة:**

- الاعتماد على هذه الخاصية قبل التأكد من وقوع حدث `'exit'`

---

#### Method: worker.send(message[, sendHandle[, options]][, callback])

**التوقيع:**

```javascript
worker.send(message[, sendHandle[, options]][, callback])
```

| المعامل    | النوع    | مطلوب/اختياري | الوصف                                                                                            |
| :--------- | :------- | :------------ | :----------------------------------------------------------------------------------------------- |
| message    | Object   | مطلوب         | الرسالة المراد إرسالها (يجب أن تكون قابلة للتسلسل إذا كان `serialization` مضبوطاً على `'json'`)  |
| sendHandle | Handle   | اختياري       | مقبض (مثل socket أو server)، يُمرّر إلى العملية العاملة                                          |
| options    | Object   | اختياري       | كائن خيارات؛ قد تحتوي على `keepOpen` (boolean) للإشارة بإبقاء المقبض مفتوحاً في العملية المرسِلة |
| callback   | Function | اختياري       | دالة تُستدعى عند إرسال الرسالة                                                                   |

**الوصف:** ترسل هذه الدالة رسالة إلى عملية عاملة أو عملية أساسية، اختيارياً مع مقبض. في العملية الأساسية، تُرسل رسالة إلى عملية عاملة محددة. في عملية عاملة، تُرسل رسالة إلى العملية الأساسية. هذه الطريقة مطابقة لـ `ChildProcess.send()` في العملية الأساسية و `process.send()` في عملية عاملة.[^1_2]

**القيمة المرجعة:** boolean

**مثال من الواقع:**

```javascript
// نظام مراقبة توزيع الأحمال مع رسائل IPC
const cluster = require("node:cluster");
const http = require("node:http");
const os = require("node:os");

if (cluster.isPrimary) {
  let totalRequests = 0;
  const requestCounts = {};

  // إنشاء عمليات عاملة
  const numWorkers = os.availableParallelism();
  for (let i = 0; i < numWorkers; i++) {
    const worker = cluster.fork();
    requestCounts[worker.id] = 0;

    // استقبال التقارير من العمليات العاملة
    worker.on("message", (msg) => {
      if (msg.cmd === "request_handled") {
        requestCounts[worker.id]++;
        totalRequests++;
      }
    });
  }

  // إرسال إحصائيات لكل عملية عاملة كل 10 ثوانٍ
  setInterval(() => {
    for (const id in cluster.workers) {
      cluster.workers[id].send({
        cmd: "stats_request",
        totalRequests: totalRequests,
      });
    }
  }, 10000);

  // إرسال أمر إيقاف الخادم إلى جميع العمليات العاملة
  process.on("SIGTERM", () => {
    for (const id in cluster.workers) {
      cluster.workers[id].send({ cmd: "shutdown" });
    }
  });
} else {
  let requestCount = 0;

  // معالج الرسائل من العملية الأساسية
  process.on("message", (msg) => {
    if (msg.cmd === "stats_request") {
      console.log(
        `Worker ${cluster.worker.id} stats: ${requestCount} requests, total: ${msg.totalRequests}`
      );
    } else if (msg.cmd === "shutdown") {
      console.log("Worker received shutdown command");
      server.close(() => {
        process.exit(0);
      });
    }
  });

  // خادم HTTP
  const server = http
    .createServer((req, res) => {
      requestCount++;
      res.writeHead(200);
      res.end("OK\n");

      // إخبار العملية الأساسية بمعالجة الطلب
      process.send({ cmd: "request_handled" });
    })
    .listen(8000);
}
```

**أخطاء شائعة:**

- محاولة إرسال كائنات غير قابلة للتسلسل عند استخدام `serialization: 'json'`
- عدم التعامل مع حالة فشل الإرسال (عندما تُرجع الدالة false)

---

#### Property: worker.process

**النوع:** ChildProcess

**الوصف:** جميع العمليات العاملة يتم إنشاؤها باستخدام `child_process.fork()`، والكائن المُرجع من هذه الدالة يُخزّن في `.process`. في عملية عاملة، يتم تخزين `process` العام.[^1_2]

يستدعي العمل `process.exit(0)` إذا حدث حدث `'disconnect'` على `process` و `.exitedAfterDisconnect` ليس `true`. هذا يحمي من قطع الاتصال العرضي.[^1_2]

**مثال من الواقع:**

```javascript
// الوصول إلى معلومات العملية الفرعية
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  const worker = cluster.fork();

  // الوصول إلى معلومات العملية الفرعية
  console.log(`Worker PID: ${worker.process.pid}`);
  console.log(
    `Worker memory: ${worker.process.memoryUsage().heapUsed / 1024 / 1024}MB`
  );

  // مراقبة استخدام الذاكرة
  setInterval(() => {
    const memMB = worker.process.memoryUsage().heapUsed / 1024 / 1024;
    console.log(`Worker ${worker.id} heap: ${memMB.toFixed(2)}MB`);

    // إعادة تشغيل إذا تجاوزت الذاكرة 500MB
    if (memMB > 500) {
      console.log("Worker memory limit exceeded - restarting");
      worker.kill();
    }
  }, 5000);
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end("Worker response\n");
    })
    .listen(8000);
}
```

---

#### Events على Worker

##### Event: 'disconnect'

**الوصف:** يُصدر حدث `'disconnect'` مشابه لـ `cluster.on('disconnect')`، لكن خاص بعملية عاملة محددة.[^1_2]

```javascript
cluster.fork().on("disconnect", () => {
  console.log("Worker has disconnected");
});
```

##### Event: 'error'

**الوصف:** نفس الحدث الذي توفره `child_process.fork()`. داخل عملية عاملة، قد يتم استخدام `process.on('error')`.[^1_2]

##### Event: 'exit'

**المعاملات:**

- `code` {number} - رمز الخروج، إذا خرج بشكل طبيعي
- `signal` {string} - اسم الإشارة (مثل `'SIGHUP'`) التي أسفرت عن قتل العملية

**الوصف:** مشابه لـ `cluster.on('exit')`، لكن خاص بعملية عاملة محددة.[^1_2]

```javascript
worker.on("exit", (code, signal) => {
  if (signal) {
    console.log(`worker was killed by signal: ${signal}`);
  } else if (code !== 0) {
    console.log(`worker exited with error code: ${code}`);
  } else {
    console.log("worker success!");
  }
});
```

##### Event: 'listening'

**الوصف:** مشابه لـ `cluster.on('listening')`، لكن خاص بعملية عاملة محددة. لا يُصدر في العملية العاملة نفسها.[^1_2]

```javascript
worker.on("listening", (address) => {
  console.log(`Worker is listening on ${address.address}:${address.port}`);
});
```

##### Event: 'message'

**المعاملات:**

- `message` {Object}
- `handle` {undefined | Object}

**الوصف:** مشابه لـ حدث `'message'` في كائن `cluster`، لكن خاص بعملية عاملة محددة.[^1_2]

---

### دوال وخصائص Cluster الأساسية

#### Function: cluster.fork([env])

**التوقيع:**

```javascript
cluster.fork([env]);
```

| المعامل | النوع  | مطلوب/اختياري | الوصف                                                   |
| :------ | :----- | :------------ | :------------------------------------------------------ |
| env     | Object | اختياري       | أزواج مفتاح/قيمة لإضافتها إلى متغيرات بيئة عملية العامل |

**الوصف:** تُنشئ عملية عاملة جديدة. لا يمكن استدعاء هذه الدالة إلا من العملية الأساسية.[^1_2]

**القيمة المرجعة:** cluster.Worker

**مثال من الواقع:**

```javascript
// إنشاء عمليات عاملة بمتغيرات بيئة مختلفة
const cluster = require("node:cluster");
const os = require("node:os");

if (cluster.isPrimary) {
  const numWorkers = os.availableParallelism();

  // إنشاء عمليات عاملة بمتغيرات بيئة مخصصة
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork({
      WORKER_TYPE: i % 2 === 0 ? "api" : "worker",
      WORKER_INDEX: i,
    });
  }
}
```

---

#### Function: cluster.disconnect([callback])

**التوقيع:**

```javascript
cluster.disconnect([callback]);
```

| المعامل  | النوع    | مطلوب/اختياري | الوصف                                                           |
| :------- | :------- | :------------ | :-------------------------------------------------------------- |
| callback | Function | اختياري       | تُستدعى عند قطع اتصال جميع العمليات العاملة وإغلاق جميع المقابض |

**الوصف:** تستدعي `.disconnect()` على كل عملية عاملة في `cluster.workers`. عند قطعها الاتصال، سيتم إغلاق جميع المقابض الداخلية، مما يسمح للعملية الأساسية بالخروج بشكل آمن إذا لم يكن هناك أي حدث آخر ينتظر.[^1_2]

**مثال من الواقع:**

```javascript
// الإيقاف الآمن للتطبيق الكامل
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  const numWorkers = 4;

  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  process.on("SIGTERM", () => {
    console.log("Disconnecting all workers...");
    cluster.disconnect(() => {
      console.log("All workers disconnected");
      process.exit(0);
    });
  });
}
```

---

#### Property: cluster.isPrimary

**النوع:** boolean

**الوصف:** تُرجع `true` إذا كانت العملية أساسية. يتم تحديد ذلك بواسطة `process.env.NODE_UNIQUE_ID`. إذا كان `process.env.NODE_UNIQUE_ID` غير معرّف، فإن `isPrimary` يكون `true`.[^1_2]

**ملاحظة:** `cluster.isMaster` هو الاسم الذي تم إهماله لهذه الخاصية.[^1_2]

**مثال من الواقع:**

```javascript
const cluster = require("node:cluster");

if (cluster.isPrimary) {
  console.log("I am the primary process");
  // كود إدارة العمليات العاملة هنا
} else {
  console.log(`I am a worker (ID: ${cluster.worker.id})`);
  // كود التطبيق الفعلي هنا
}
```

---

#### Property: cluster.isWorker

**النوع:** boolean

**الوصف:** تُرجع `true` إذا كانت العملية ليست أساسية (عكس `cluster.isPrimary`).[^1_2]

---

#### Property: cluster.worker

**النوع:** Object

**الوصف:** مرجع إلى كائن العملية العاملة الحالية. غير متوفر في العملية الأساسية.[^1_2]

**مثال من الواقع:**

```javascript
const cluster = require("node:cluster");
const http = require("node:http");

if (cluster.isPrimary) {
  console.log("I am primary");
  cluster.fork();
  cluster.fork();
} else if (cluster.isWorker) {
  console.log(`I am worker #${cluster.worker.id}`);

  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end(`Served by worker ${cluster.worker.id}\n`);
    })
    .listen(8000);
}
```

---

#### Property: cluster.workers

**النوع:** Object

**الوصف:** كائن hash يُخزّن كائنات العمليات العاملة النشطة، مفهرسة بحقل `id`. يسهل التكرار عبر جميع العمليات العاملة. متوفر فقط في العملية الأساسية.[^1_2]

تُحذف العملية العاملة من `cluster.workers` بعد قطع اتصالها وخروجها. ترتيب الأحداث قد لا يكون محدداً مسبقاً.[^1_2]

**مثال من الواقع:**

```javascript
// بث رسالة إلى جميع العمليات العاملة
const cluster = require("node:cluster");

for (const worker of Object.values(cluster.workers)) {
  worker.send("big announcement to all workers");
}

// أو حلقة قديمة الطراز:
for (const id in cluster.workers) {
  cluster.workers[id].send("announcement");
}
```

---

#### Property: cluster.schedulingPolicy

**النوع:** string

**القيم الممكنة:**

- `cluster.SCHED_RR` - Round-Robin (الإعادة الدورية)
- `cluster.SCHED_NONE` - ترك التوزيع للنظام التشغيلي

**الوصف:** سياسة الجدولة، إما `cluster.SCHED_RR` للإعادة الدورية أو `cluster.SCHED_NONE` لترك الجدولة للنظام التشغيلي. هذا إعداد عام يُجمّد بشكل فعلي بعد إنشاء أول عملية عاملة أو استدعاء `.setupPrimary()`.[^1_2]

`SCHED_RR` هو الافتراضي على جميع أنظمة التشغيل ما عدا Windows. سيتغيّر Windows إلى `SCHED_RR` بمجرد أن تتمكن libuv من توزيع مقابض IOCP بكفاءة.[^1_2]

يمكن أيضاً تعيين `cluster.schedulingPolicy` من خلال متغير البيئة `NODE_CLUSTER_SCHED_POLICY`. القيم الصحيحة: `'rr'` و `'none'`.[^1_2]

**مثال من الواقع:**

```javascript
// اختيار سياسة الجدولة بناءً على النظام
const cluster = require("node:cluster");

if (process.platform === "win32") {
  // الإعادة الدورية على Windows
  cluster.schedulingPolicy = cluster.SCHED_RR;
} else {
  // ترك الجدولة للنظام على Unix
  cluster.schedulingPolicy = cluster.SCHED_NONE;
}

// الآن أنشئ العمليات العاملة
if (cluster.isPrimary) {
  for (let i = 0; i < 4; i++) {
    cluster.fork();
  }
}
```

---

#### Property: cluster.settings

**النوع:** Object

**الخصائص:**

| الخاصية         | النوع              | القيمة الافتراضية                   | الوصف                                              |
| :-------------- | :----------------- | :---------------------------------- | :------------------------------------------------- |
| `exec`          | string             | `process.argv[^1_1]`                | مسار الملف للعملية العاملة                         |
| `args`          | string[]           | `process.argv.slice(2)`             | معاملات نصية يتم تمريرها إلى العملية العاملة       |
| `cwd`           | string             | undefined (ترث من العملية الأساسية) | مجلد العمل الحالي لعملية العاملة                   |
| `execArgv`      | string[]           | `process.execArgv`                  | معاملات مسموح بها للتمرير إلى Node.js              |
| `serialization` | string             | false                               | نوع التسلسل المستخدم: `'json'` أو `'advanced'`     |
| `silent`        | boolean            | false                               | إذا كانت true، لا تُرسل المخرجات إلى `stdio` الأب  |
| `stdio`         | Array              | undefined                           | يُعيّن `stdio` للعمليات المشعبة                    |
| `uid`           | number             | undefined                           | تعيين هوية المستخدم للعملية                        |
| `gid`           | number             | undefined                           | تعيين هوية المجموعة للعملية                        |
| `inspectPort`   | number \| Function | undefined                           | يُعيّن منفذ المفتش (debugger port) للعملية العاملة |
| `windowsHide`   | boolean            | false                               | إخفاء نافذة العملية المشعبة على Windows            |

**الوصف:** بعد استدعاء `.setupPrimary()` أو `.fork()`، سيحتوي هذا الكائن على الإعدادات، بما فيها القيم الافتراضية. لا يُقصد تعديل أو تعيين هذا الكائن يدويّاً.[^1_2]

**مثال من الواقع:**

```javascript
// عرض الإعدادات الحالية للـ cluster
const cluster = require("node:cluster");

cluster.setupPrimary({
  exec: "worker.js",
  args: ["--use", "https"],
  silent: true,
  serialization: "advanced",
});

console.log("Cluster settings:");
console.log(cluster.settings);
// الناتج:
// {
//   exec: 'worker.js',
//   args: [ '--use', 'https' ],
//   cwd: undefined,
//   silent: true,
//   ...
// }
```

---

#### Function: cluster.setupPrimary([settings])

**التوقيع:**

```javascript
cluster.setupPrimary([settings]);
```

| المعامل  | النوع  | مطلوب/اختياري | الوصف                                                                                                           |
| :------- | :----- | :------------ | :-------------------------------------------------------------------------------------------------------------- |
| settings | Object | اختياري       | كائن إعدادات يتضمن: exec, args, cwd, serialization, silent, stdio, uid, gid, inspectPort, windowsHide, execArgv |

**الوصف:** تُستخدم `setupPrimary()` لتغيير السلوك الافتراضي لـ 'fork'. بعد استدعاؤها، ستكون الإعدادات موجودة في `cluster.settings`. أي تغييرات إعدادات تؤثر فقط على استدعاءات `.fork()` المستقبلية ولا تؤثر على العمليات العاملة التي تعمل بالفعل.[^1_2]

الخاصية الوحيدة لعملية عاملة التي لا يمكن تعيينها عبر `.setupPrimary()` هي `env` التي تُمرّر إلى `.fork()`.[^1_2]

يسري الإعدادات الافتراضية على الاستدعاء الأول فقط؛ الإعدادات الافتراضية للاستدعاءات اللاحقة هي القيم الحالية في وقت استدعاء `.setupPrimary()`.[^1_2]

**القيمة المرجعة:** undefined

**مثال من الواقع:**

```javascript
// إنشاء عمليات عاملة بإعدادات مختلفة
const cluster = require("node:cluster");

if (cluster.isPrimary) {
  // إعداد عمليات عاملة HTTPS
  cluster.setupPrimary({
    exec: "worker.js",
    args: ["--use", "https"],
    silent: true,
  });
  const httpsWorker = cluster.fork(); // worker HTTPS

  // تغيير الإعدادات للعمليات التالية
  cluster.setupPrimary({
    exec: "worker.js",
    args: ["--use", "http"],
    silent: false,
  });
  const httpWorker = cluster.fork(); // worker HTTP
}
```

---

### أحداث Cluster (Cluster Events)

#### Event: 'fork'

**المعاملات:** `worker` {cluster.Worker}

**الوصف:** عند تشعيب (fork) عملية عاملة جديدة، سيُصدر وحدة cluster حدث `'fork'`. يمكن استخدام هذا لتسجيل نشاط العملية العاملة وإنشاء مهلة زمنية مخصصة.[^1_2]

```javascript
const timeouts = [];

cluster.on("fork", (worker) => {
  timeouts[worker.id] = setTimeout(() => {
    console.error("Something must be wrong with the connection ...");
  }, 2000);
});

cluster.on("listening", (worker, address) => {
  clearTimeout(timeouts[worker.id]);
});

cluster.on("exit", (worker, code, signal) => {
  clearTimeout(timeouts[worker.id]);
});
```

---

#### Event: 'online'

**المعاملات:** `worker` {cluster.Worker}

**الوصف:** بعد تشعيب عملية عاملة جديدة، يجب على العملية العاملة أن ترسل رسالة "online". عند استقبال رسالة "online" من قِبل العملية الأساسية، سيُصدر حدث `'online'`.[^1_2]

الفرق بين `'fork'` و `'online'` هو أن `'fork'` يُصدر عندما تشعب العملية الأساسية عملية عاملة، و `'online'` يُصدر عند تشغيل العملية العاملة.[^1_2]

```javascript
cluster.on("online", (worker) => {
  console.log("Yay, the worker responded after it was forked");
});
```

---

#### Event: 'listening'

**المعاملات:**

- `worker` {cluster.Worker}
- `address` {Object} - كائن العنوان يحتوي على: `address`, `port`, `addressType`

**الوصف:** بعد استدعاء `listen()` من عملية عاملة، عند إصدار حدث `'listening'` على الخادم، سيُصدر حدث `'listening'` أيضاً على `cluster` في العملية الأساسية.[^1_2]

```javascript
cluster.on("listening", (worker, address) => {
  console.log(
    `A worker is now connected to ${address.address}:${address.port}`
  );
});
```

---

#### Event: 'disconnect'

**المعاملات:** `worker` {cluster.Worker}

**الوصف:** يُصدر هذا الحدث بعد قطع قناة IPC للعملية العاملة. قد يحدث هذا عندما تخرج العملية العاملة بشكل آمن، أو يتم قتلها، أو يتم قطع اتصالها يدويّاً.[^1_2]

قد تكون هناك تأخير بين الأحداث `'disconnect'` و `'exit'`. يمكن استخدام هذه الأحداث للكشف عما إذا كانت العملية عالقة في التنظيف أو كان هناك اتصالات طويلة الأجل.[^1_2]

```javascript
cluster.on("disconnect", (worker) => {
  console.log(`The worker #${worker.id} has disconnected`);
});
```

---

#### Event: 'exit'

**المعاملات:**

- `worker` {cluster.Worker}
- `code` {number} - رمز الخروج إذا خرجت بشكل طبيعي
- `signal` {string} - اسم الإشارة (مثل `'SIGHUP'`) التي أسفرت عن قتل العملية

**الوصف:** عند موت أي من العمليات العاملة، سيُصدر وحدة cluster حدث `'exit'`. يمكن استخدام هذا لإعادة تشغيل العملية العاملة بالاستدعاء لـ `.fork()` مرة أخرى.[^1_2]

```javascript
cluster.on("exit", (worker, code, signal) => {
  console.log(
    "worker %d died (%s). restarting...",
    worker.process.pid,
    signal || code
  );
  cluster.fork();
});
```

---

#### Event: 'message'

**المعاملات:**

- `worker` {cluster.Worker}
- `message` {Object}
- `handle` {undefined | Object}

**الوصف:** يُصدر هذا الحدث عندما تستقبل العملية الأساسية رسالة من أي عملية عاملة.[^1_2]

---

#### Event: 'setup'

**المعاملات:** `settings` {Object}

**الوصف:** يُصدر هذا الحدث كل مرة يتم استدعاء `.setupPrimary()`. كائن `settings` هو كائن `cluster.settings` في الوقت الذي تم استدعاء `.setupPrimary()` فيه وهو استشاري فقط، حيث يمكن إجراء استدعاءات متعددة لـ `.setupPrimary()` في tick واحد.[^1_2]

---

## مقارنات واتخاذ القرارات

### مقارنة: Round-Robin vs SCHED_NONE

| الميزة                 | Round-Robin (SCHED_RR)                | SCHED_NONE                                          |
| :--------------------- | :------------------------------------ | :-------------------------------------------------- |
| **الأداء النظري**      | جيد مع توزيع متوازن                   | قد يكون أفضل في بعض الحالات                         |
| **التوازن**            | موازنة مضمونة تقريباً                 | قد يحدث عدم توازن بسبب جدولة النظام                 |
| **التنفيذ**            | العملية الأساسية توزع الطلبات يدويّاً | كل عملية عاملة تقبل الاتصالات مباشرة                |
| **الأنظمة الافتراضية** | جميع الأنظمة ما عدا Windows           | Windows                                             |
| **الملاحظات الواقعية** | افتراضي وموثوق                        | قد تؤدي 70% من الاتصالات إلى عمليتين فقط من 8[^1_1] |

---

### مقارنة: Cluster vs Worker Threads vs Child Processes

| الميزة               | Cluster                     | Worker Threads              | Child Processes            |
| :------------------- | :-------------------------- | :-------------------------- | :------------------------- |
| **العزل**            | عملية منفصلة بذاكرة منفصلة  | نفس العملية مع ذاكرة مشتركة | عملية منفصلة مستقلة تماماً |
| **النفقات**          | أعلى - عملية كاملة          | منخفضة - خيط فقط            | عالية جداً                 |
| **التواصل**          | IPC عبر رسائل               | shared memory               | stdio، pipes، IPC          |
| **الحالات المثالية** | خوادم الويب متعددة العمليات | حسابات CPU-intensive        | تشغيل برامج خارجية         |
| **Scalability**      | مثالي للـ clustering        | جيد للمهام المتزامنة        | لا يقياسي                  |

---

## الفوائد والعيوب والحالات الاستخدامية

### الفوائد

1. **تحسين الأداء بشكل كبير:** باستخدام جميع نوى المعالج، يمكن لتطبيق Node.js معالجة آلاف الطلبات بدلاً من مئات.[^1_7][^1_4][^1_5]
2. **الموثوقية والتوفر المستمر:** إذا تعطلت عملية عاملة، يمكن للعملية الأساسية استكشافها وإعادة تشغيلها تلقائياً، مما يضمن استمرارية الخدمة.[^1_5][^1_6]
3. **معالجة تلقائية للأحمال:** لا حاجة لتكوين معقد - توزيع الطلبات يتم تلقائياً.[^1_2]
4. **عزل العمليات:** كل عملية عاملة لها ذاكرة خاصة بها، مما يمنع تأثر عملية عاملة بتعطل الأخرى.[^1_2]

---

### العيوب والقيود

1. **عدم توازن محتمل مع SCHED_NONE:** في بعض الحالات، 70% من الاتصالات قد تنتهي في عمليتين فقط من 8.[^1_1]
2. **صعوبة مشاركة الحالة (Stateful Operations):** بما أن كل عملية عاملة لديها ذاكرة منفصلة، مشاركة الحالة بين العمليات تتطلب آليات إضافية (مثل Redis أو قواعد البيانات).[^1_2]
3. **الذاكرة الإضافية:** كل عملية عاملة تستهلك ذاكرة كبيرة (عادة 30-50MB لعملية Node.js فارغة).[^1_5]
4. **التعقيد في الإدارة:** إدارة دورة حياة العمليات العاملة تتطلب كود إضافي.[^1_2]
5. **محدود للعمليات CPU-Intensive:** وحدة `cluster` لا توفر تحسناً للعمليات الثقيلة CPU لأن العمليات تتشارك نفس الحلقة أحداث الأساسية.[^1_2]

---

### أفضل 3 حالات استخدام

**1. خوادم الويب HTTP/HTTPS (الحالة الأولى)**

```javascript
// استخدام cluster لخادم ويب
const cluster = require("node:cluster");
const http = require("node:http");
const os = require("node:os");

if (cluster.isPrimary) {
  const numCPUs = os.availableParallelism();

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} died - restarting`);
    cluster.fork();
  });
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200);
      res.end("Hello from worker " + cluster.worker.id + "\n");
    })
    .listen(8000);
}
```

**2. تطبيقات REST API متزامنة**
تطبيقات واجهات برمجية (APIs) تتعامل مع آلاف الطلبات المتزامنة بدون حسابات ثقيلة CPU.

**3. خدمات التطبيقات متعددة الخدمات (Microservices)**
عندما تحتاج كل عملية عاملة إلى تشغيل نسخة من الخدمة الصغيرة بشكل مستقل.[^1_6][^1_5]

---

## مثال متقدم: نظام متكامل مع Graceful Shutdown

```javascript
// مثال متكامل: خادم HTTP متقدم مع shutdown آمن
const cluster = require("node:cluster");
const http = require("node:http");
const os = require("node:os");

const PORT = 3000;
const GRACEFUL_SHUTDOWN_TIMEOUT = 5000;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  const numCPUs = os.availableParallelism();
  const workers = new Set();

  // إنشاء عمليات عاملة
  for (let i = 0; i < numCPUs; i++) {
    const worker = cluster.fork();
    workers.add(worker);
  }

  // معالج إعادة التشغيل
  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code})`);
    workers.delete(worker);

    // لا تعيد التشغيل إذا كان shutdown متعمداً
    if (!worker.exitedAfterDisconnect) {
      const newWorker = cluster.fork();
      workers.add(newWorker);
    }
  });

  // معالج الإيقاف الآمن
  process.on("SIGTERM", gracefulShutdown);
  process.on("SIGINT", gracefulShutdown);

  function gracefulShutdown() {
    console.log("Primary: initiating graceful shutdown");

    let shutdownTimeout = setTimeout(() => {
      console.log("Primary: forcing shutdown after timeout");
      process.exit(1);
    }, GRACEFUL_SHUTDOWN_TIMEOUT);

    for (const worker of workers) {
      worker.disconnect();

      worker.on("disconnect", () => {
        workers.delete(worker);
        if (workers.size === 0) {
          clearTimeout(shutdownTimeout);
          process.exit(0);
        }
      });
    }
  }
} else {
  // عملية عاملة
  const server = http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Response from worker ${cluster.worker.id}\n`);
  });

  server.listen(PORT);

  // معالج إيقاف آمن في العملية العاملة
  process.on("SIGTERM", () => {
    console.log(`Worker ${cluster.worker.id}: closing connections`);
    server.close(() => {
      process.exit(0);
    });
  });

  // معالج رسائل من العملية الأساسية
  process.on("message", (msg) => {
    if (msg.cmd === "shutdown") {
      console.log(`Worker ${cluster.worker.id}: received shutdown signal`);
      server.close(() => {
        process.exit(0);
      });
    }
  });

  console.log(`Worker ${cluster.worker.id} started on port ${PORT}`);
}
```

---

## الخلاصة

وحدة `cluster` في Node.js توفر أداة قوية وفعالة للاستفادة من معالجات متعددة النوى وتحسين أداء التطبيقات بشكل كبير. من خلال العملية الأساسية التي توزع الطلبات على عمليات عاملة متعددة، يمكن للتطبيقات معالجة أحمال عمل أكبر بكثير. مع الاهتمام بالإدارة الصحيحة لدورة حياة العمليات وتطبيق آليات الإيقاف الآمن، يمكن بناء تطبيقات موثوقة وقابلة للتوسع. استخدم `cluster` للخوادم متعددة الطلبات، وفكر في `worker_threads` للمهام الحسابية الثقيلة ضمن عملية واحدة، و`child_process` للبرامج الخارجية المستقلة.[^1_4][^1_6][^1_5][^1_1][^1_2]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_8][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88][^1_89][^1_9][^1_90][^1_91][^1_92][^1_93][^1_94][^1_95][^1_96][^1_97]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/cluster.md
[^1_2]: https://nodejs.org/api/cluster.html
[^1_3]: https://www.w3schools.com/nodejs/nodejs_cluster.asp
[^1_4]: https://innosufiyan.hashnode.dev/introduction-to-load-balancing-in-nodejs
[^1_5]: https://betterstack.com/community/guides/scaling-nodejs/node-clustering/
[^1_6]: https://www.gcc-marketing.com/scaling-your-node-js-applications-with-cluster-modules-for-efficient-load-balancing/
[^1_7]: https://www.digitalocean.com/community/tutorials/how-to-scale-node-js-applications-with-clustering
[^1_8]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_9]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_10]: http://transactions.ismir.net/articles/10.5334/tismir.111/
[^1_11]: https://www.ijisrt.com/envision-edtech-revolutionizing-intelligent-education-through-ai-and-innovation-for-a-smarter-tomorrow
[^1_12]: https://www.semanticscholar.org/paper/959d24d21efe31eae401be73c1f7d8e636805655
[^1_13]: https://www.semanticscholar.org/paper/52efec79cdfa96974bb046eb82c60e2c66421473
[^1_14]: https://academic.oup.com/bioinformatics/article/24/6/876/192449
[^1_15]: https://www.semanticscholar.org/paper/bdbd4968cf95800c6ebe9dfd549bd67fbefcc4e6
[^1_16]: https://www.semanticscholar.org/paper/1b0985412c5cf9904e67b88ee1ae4e1256775fa8
[^1_17]: https://arxiv.org/pdf/2403.10954.pdf
[^1_18]: https://arxiv.org/pdf/2101.00756.pdf
[^1_19]: http://arxiv.org/pdf/1507.02798.pdf
[^1_20]: https://journals.sagepub.com/doi/pdf/10.1177/23998083241293833
[^1_21]: http://arxiv.org/pdf/2312.12697.pdf
[^1_22]: https://arxiv.org/pdf/2110.14162.pdf
[^1_23]: https://arxiv.org/pdf/2306.13984.pdf
[^1_24]: http://arxiv.org/pdf/2401.10582.pdf
[^1_25]: https://freud.readthedocs.io/en/latest/modules/cluster.html
[^1_26]: https://stackoverflow.com/questions/8534462/nodejscluster-how-to-send-data-from-master-to-all-or-single-child-workers
[^1_27]: https://docs.ceph.com/en/latest/mgr/modules/
[^1_28]: https://www.geeksforgeeks.org/node-js/differentiate-between-worker-threads-and-clusters-in-node-js/
[^1_29]: https://aerospike.com/docs/develop/client/node/async/cluster/
[^1_30]: https://scikit-learn.org/stable/modules/clustering.html
[^1_31]: https://gist.github.com/jpoehls/2232358
[^1_32]: https://stackoverflow.com/questions/47682576/python-cluster-module-confusing-class-methods
[^1_33]: https://nodejs.org/download/release/v0.8.27/docs/api/cluster.html
[^1_34]: https://mirrors.huaweicloud.com/nodejs/v0.11.5/docs/api/cluster.html
[^1_35]: https://learn.microsoft.com/en-us/powershell/module/failoverclusters/?view=windowsserver2025-ps
[^1_36]: https://dev.to/leapcell/understanding-nodejs-cluster-the-core-concepts-1c9e
[^1_37]: https://nodejs.org/download/release/v0.8.4/docs/api/cluster.html
[^1_38]: https://www.geeksforgeeks.org/machine-learning/clustering-in-machine-learning/
[^1_39]: https://www.npmjs.com/package/worker-communication
[^1_40]: https://millermedeiros.github.io/mdoc/examples/node_api/doc/cluster.html
[^1_41]: https://doc.akka.io/libraries/akka-core/current/typed/cluster.html
[^1_42]: https://ieeexplore.ieee.org/document/11196547/
[^1_43]: https://ieeexplore.ieee.org/document/10823883/
[^1_44]: https://www.cambridge.org/core/product/identifier/S002202992400027X/type/journal_article
[^1_45]: https://ieeexplore.ieee.org/document/10695533/
[^1_46]: https://ieeexplore.ieee.org/document/8577911/
[^1_47]: http://ieeexplore.ieee.org/document/7414039/
[^1_48]: https://dl.acm.org/doi/10.1145/3314487.3314490
[^1_49]: https://www.semanticscholar.org/paper/f20d1175d75108a5566ce73ca101d722fee04834
[^1_50]: https://www.semanticscholar.org/paper/9e9649d97ba496e9e77dcb2a15e8a3d01745bf78
[^1_51]: http://archives.pdx.edu/ds/psu/16527
[^1_52]: https://www.mdpi.com/1424-8220/21/17/5906/pdf
[^1_53]: http://thesai.org/Downloads/Volume13No4/Paper_63-A_New_Combination_Approach_to_CPU_Scheduling.pdf
[^1_54]: https://arxiv.org/pdf/2205.07314.pdf
[^1_55]: https://arxiv.org/html/2503.16166
[^1_56]: https://arxiv.org/pdf/2409.15704v1.pdf
[^1_57]: https://pmc.ncbi.nlm.nih.gov/articles/PMC8433676/
[^1_58]: https://arxiv.org/pdf/1309.3096.pdf
[^1_59]: https://arxiv.org/ftp/arxiv/papers/0912/0912.0606.pdf
[^1_60]: https://www.w3schools.com/nodejs/ref_worker.asp
[^1_61]: https://www.geeksforgeeks.org/node-js/how-to-create-load-balancing-servers-using-node-js/
[^1_62]: https://innosufiyan.hashnode.dev/enhancing-cluster-management-with-pm2
[^1_63]: https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/
[^1_64]: https://bun.com/reference/node/cluster/Cluster/schedulingPolicy
[^1_65]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/graceful-shutdown-cluster
[^1_66]: https://blog.bitsrc.io/nodejs-performance-optimization-with-clustering-b52915054cc2
[^1_67]: https://www.youtube.com/watch?v=6lHvks6R6cI
[^1_68]: https://dev.to/acanimal/graceful-shutdown-nodejs-http-server-when-using-pm2-44-10o0
[^1_69]: https://javascript.plainenglish.io/resolving-node-js-cluster-bottlenecks-overcoming-round-robin-load-balancing-issues-025b68587754
[^1_70]: https://stackoverflow.com/questions/55203749/nodejs-cluster-useful-use-cases
[^1_71]: https://projectai.in/projects/96364664-c52c-40e1-8020-5b70fbd612d5/tasks/94816efc-7d8b-4172-aab8-5400da348ad4
[^1_72]: https://stackoverflow.com/questions/43971263/nodejs-cluster-not-using-round-robin-developing-on-windows
[^1_73]: https://www.insia.ai/blog-posts/scaling-node-js-to-infinity-and-beyond-cluster-and-load-balancing-for-galactic-performance
[^1_74]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1002/cpe.5823
[^1_75]: https://arxiv.org/pdf/2012.00365.pdf
[^1_76]: http://arxiv.org/pdf/2503.13072.pdf
[^1_77]: https://arxiv.org/pdf/2204.10768.pdf
[^1_78]: https://arxiv.org/pdf/1512.07067.pdf
[^1_79]: http://arxiv.org/pdf/2311.11095.pdf
[^1_80]: http://arxiv.org/pdf/2401.08595.pdf
[^1_81]: https://vocal.media/fyi/inter-process-communication-ipc
[^1_82]: https://leapcell.io/blog/understanding-nodejs-cluster
[^1_83]: https://taejunham.github.io/data/cereal_isca2020.pdf
[^1_84]: https://documentation.suse.com/sle-ha/12-SP5/html/SLE-HA-all/cha-ha-manual-config.html
[^1_85]: https://www.linkedin.com/pulse/comparing-child-processes-clusters-worker-threads-nodejs-srikanth-r-z2sgc
[^1_86]: https://stackoverflow.com/questions/15333131/serialization-as-an-ipc-mechanism
[^1_87]: https://cdn.jsdelivr.net/npm/@types/node@20.5.6/cluster.d.ts
[^1_88]: https://nodejs.org/api/child_process.html
[^1_89]: https://code.kx.com/q/basics/ipc/
[^1_90]: https://csgit.ucalgary.ca/keshav.bhardwaj1/cpsc-471-project/-/blob/cd26483568e917877169a7c13846dbafe7f75037/node_modules/@types/node/cluster.d.ts
[^1_91]: https://www.geeksforgeeks.org/distributed-systems/interprocess-communication-in-distributed-systems/
[^1_92]: https://www.veritas.com/support/en_US/doc/129673366-129673369-0/uxrt-731_v25084395-129673369
[^1_93]: https://bun.com/reference/node/cluster/Worker/process
[^1_94]: https://github.com/topics/inter-process-communication
[^1_95]: http://mongodb.asprain.cn/nodejs18.2.1/all.html
[^1_96]: https://stackoverflow.com/questions/56656498/how-do-cluster-and-worker-threads-work-in-node-js
[^1_97]: https://dl.acm.org/doi/10.1109/IROS40897.2019.8968462

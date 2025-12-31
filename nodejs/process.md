
# توثيق شامل: كائن `process` العام في Node.js

توثيق مرجعي متقدم يغطي جميع الوظائف والخصائص والأحداث غير المهجورة في كائن `process` العام، مع أمثلة عملية واقعية وشروحات تفصيلية لمطوري Node.js المتقدمين.

## 1. نظرة عامة على الوحدة (Module Overview)

### ما هو كائن `process`؟[^1_1]

كائن `process` هو كائن عام (global object) يوفر معلومات شاملة عن عملية Node.js الحالية والتحكم فيها. يعتبر `process` مثيلاً من `EventEmitter`، مما يعني أنه يدعم نظام الأحداث الكامل في Node.js. يتم الوصول إليه بدون الحاجة إلى استيراده بشكل صريح في أي مكان في التطبيق.[^1_2][^1_1]

### متى يتم استخدام `process` بدلاً من مكتبات خارجية؟[^1_1][^1_2]

| الحالة | استخدام `process` | بديل خارجي |
| :-- | :-- | :-- |
| قراءة متغيرات البيئة | ✅ استخدام `process.env` | `dotenv` للملفات |
| معالجة الإشارات (Signals) | ✅ `process.on('SIGTERM')` | مدير عمليات خارجي |
| مراقبة الذاكرة والأداء | ✅ `process.memoryUsage()` | أدوات APM متخصصة |
| الاتصال بين العمليات (IPC) | ✅ `process.send()` مع child processes | RabbitMQ, Redis |
| جدولة العمليات | ❌ استخدم `process.nextTick()` | لا توجد بدائل عملية |

## 2. تفاصيل الدوال والخصائص (API Deep Dive)

![Node.js Event Loop Phases and Callback Execution Order](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/aad37a1e381910c09abc999caed49b4b/4ec07e75-3f9c-4de6-a7f6-1f11421249fb/d8067fc7.png)

Node.js Event Loop Phases and Callback Execution Order

### الخصائص الأساسية (Basic Properties)

#### `process.env`[^1_1][^1_3][^1_4]

**الصيغة:** `process.env`

**الوصف:** كائن يحتوي على جميع متغيرات البيئة المعرفة في نظام التشغيل. يتم استخدامه لقراءة كل المتغيرات المعرفة في المتغيرات البيئية النظامية (environment variables) أو المُعرّفة عند بدء العملية.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `key` | string | مطلوب | اسم متغير البيئة المراد الوصول إليه |
| القيمة | string \| undefined | مطلوب | قيمة المتغير أو `undefined` إذا لم يكن معرّفاً |

**القيمة المرجعة:** قيمة string تمثل قيمة متغير البيئة، أو `undefined` إذا لم يكن المتغير موجوداً.[^1_4]

**مثال عملي واقعي: تحميل الإعدادات بناءً على البيئة**

```javascript
// app.js - ملف الإعدادات المركزي
const config = {
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT) || 5432,
    username: process.env.DB_USER || 'admin',
    password: process.env.DB_PASSWORD,
    ssl: process.env.DB_SSL === 'true'
  },
  api: {
    port: parseInt(process.env.API_PORT) || 3000,
    corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000'
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    prettyPrint: process.env.NODE_ENV !== 'production'
  },
  features: {
    enableCache: process.env.ENABLE_CACHE === 'true',
    cacheTimeout: parseInt(process.env.CACHE_TIMEOUT) || 3600000
  }
};

// تحديد البيئة الحالية
const currentEnv = process.env.NODE_ENV || 'development';
console.log(`تشغيل التطبيق في بيئة: ${currentEnv}`);
console.log(`قاعدة البيانات: ${config.database.host}:${config.database.port}`);

module.exports = config;
```

**أخطاء شائعة:**

- عدم التحقق من وجود المتغير قبل استخدامه (قد يسبب `undefined`)
- عدم تحويل القيم الرقمية من string إلى number
- تخزين كلمات السر بشكل مباشر في الكود بدلاً من استخدام المتغيرات

***

#### `process.argv`[^1_1]

**الصيغة:** `process.argv`

**الوصف:** مصفوفة تحتوي على معاملات سطر الأوامر (command-line arguments) التي تم تمريرها عند تشغيل عملية Node.js. العنصر الأول هو مسار Node.js التنفيذي، والعنصر الثاني هو مسار الملف المنفذ، والعناصر المتبقية هي الحجج المخصصة.

**القيمة المرجعة:** مصفوفة من strings تمثل معاملات الأوامر.

**مثال عملي واقعي: برنامج معالجة الصور من سطر الأوامر**

```javascript
// image-processor.js
const path = require('path');
const fs = require('fs');

function parseArguments() {
  const args = process.argv.slice(2); // تخطي node ومسار الملف
  
  if (args.length < 2) {
    console.error('الاستخدام: node image-processor.js <inputFile> <operation> [options]');
    console.error('العمليات المدعومة: resize, crop, convert, compress');
    process.exit(1);
  }

  return {
    inputFile: args[^1_0],
    operation: args[^1_1],
    width: parseInt(args[^1_2]) || 800,
    height: parseInt(args[^1_3]) || 600,
    format: args[^1_4] || 'jpeg',
    quality: parseInt(args[^1_5]) || 80
  };
}

const config = parseArguments();
console.log(`معالجة الملف: ${config.inputFile}`);
console.log(`العملية: ${config.operation}`);
console.log(`الحجم: ${config.width}x${config.height}`);
```

**تشغيل البرنامج:**

```bash
node image-processor.js input.png resize 1024 768 webp 85
```


***

#### `process.pid` و `process.ppid`[^1_1]

**الصيغة:** `process.pid` و `process.ppid`

**الوصف:** `process.pid` يرجع معرّف العملية الحالية (Process ID)، بينما `process.ppid` يرجع معرّف العملية الأب (Parent Process ID). هذان الرقمان فريدان على نظام التشغيل ويمكن استخدامهما للتشخيص والمراقبة.

**مثال عملي واقعي: نظام مراقبة عمليات**

```javascript
// process-monitor.js
const fs = require('fs');
const path = require('path');

class ProcessMonitor {
  constructor() {
    this.logDir = './process-logs';
    this.ensureLogDir();
    this.startMonitoring();
  }

  ensureLogDir() {
    if (!fs.existsSync(this.logDir)) {
      fs.mkdirSync(this.logDir, { recursive: true });
    }
  }

  startMonitoring() {
    // تسجيل معلومات العملية الأساسية
    const processInfo = {
      pid: process.pid,
      ppid: process.ppid,
      startTime: new Date().toISOString(),
      platform: process.platform,
      nodeVersion: process.version,
      execPath: process.execPath
    };

    const logPath = path.join(this.logDir, `process-${process.pid}.json`);
    fs.writeFileSync(logPath, JSON.stringify(processInfo, null, 2));

    // مراقبة الذاكرة بشكل دوري
    setInterval(() => {
      const memUsage = process.memoryUsage();
      const timestamp = new Date().toISOString();
      
      console.log(`[${timestamp}] PID: ${process.pid} | RSS: ${(memUsage.rss / 1024 / 1024).toFixed(2)}MB`);
    }, 5000);
  }
}

new ProcessMonitor();
```


***

### دوال التحكم والمعالجة

#### `process.exit([code])`[^1_1]

**الصيغة:** `process.exit([code])`

**المعاملات:**


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `code` | integer | اختياري | رمز الخروج (0 = نجاح، 1+ = فشل). الافتراضي: 0 |

**الوصف:** توقف فوري للعملية مع إنهاء حلقة الأحداث. جميع أحداث `'exit'` يتم تنفيذها بشكل متزامن قبل الإنهاء النهائي.

**مثال عملي واقعي: معالج الأخطاء المركزي مع الإنهاء الآمن**

```javascript
// error-handler.js
class ErrorHandler {
  constructor() {
    this.setupGracefulShutdown();
  }

  setupGracefulShutdown() {
    // معالجة الأخطاء غير المعالجة
    process.on('uncaughtException', (error, origin) => {
      console.error(`❌ خطأ غير معالج من ${origin}:`, error);
      this.cleanup('UNCAUGHT_EXCEPTION', 1);
    });

    // معالجة الوعود المرفوضة
    process.on('unhandledRejection', (reason, promise) => {
      console.error('❌ وعد مرفوض بدون معالج:', reason);
      this.cleanup('UNHANDLED_REJECTION', 1);
    });

    // الإشارات
    process.on('SIGTERM', () => {
      console.log('📋 تم استقبال SIGTERM - بدء الإنهاء الآمن');
      this.cleanup('SIGTERM', 0);
    });

    process.on('SIGINT', () => {
      console.log('📋 تم استقبال SIGINT - بدء الإنهاء الآمن');
      this.cleanup('SIGINT', 0);
    });
  }

  cleanup(reason, exitCode) {
    console.log(`🔄 تنظيف الموارد... (السبب: ${reason})`);
    
    // مثال: إغلاق الاتصالات
    // await database.close();
    // await cache.disconnect();
    // await logger.flush();

    setTimeout(() => {
      console.log(`✅ تنظيف مكتمل. الخروج برمز ${exitCode}`);
      process.exit(exitCode);
    }, 2000);
  }
}

// استخدام
const errorHandler = new ErrorHandler();

// اختبار
setTimeout(() => {
  throw new Error('اختبار خطأ غير معالج');
}, 3000);
```

**أخطاء شائعة:**

- استدعاء `process.exit()` في حدث `'exit'` (سيتم تجاهله)
- عدم انتظار إغلاق الاتصالات قبل الإنهاء
- عدم معالجة الأخطاء المتزامنة بشكل صحيح

***

#### `process.nextTick(callback[, ...args])`[^1_5][^1_6][^1_7]

**الصيغة:** `process.nextTick(callback[, ...args])`

**المعاملات:**


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `callback` | Function | مطلوب | الدالة المراد تنفيذها |
| `...args` | any | اختياري | معاملات يتم تمريرها للدالة |

**القيمة المرجعة:** `undefined`

**الوصف:** جدولة تنفيذ الدالة في طور `nextTick queue` - أي قبل أن يعود تحكم الحدث للـ event loop. هذا أسرع بكثير من `setImmediate()` لكنه قد يسبب جوع I/O إذا تم استدعاؤه بشكل متكرر.

**مثال عملي واقعي: نظام معالجة الطلبات المتزامنة**

```javascript
// request-processor.js
class RequestProcessor {
  constructor() {
    this.requestQueue = [];
    this.isProcessing = false;
  }

  async queueRequest(request) {
    this.requestQueue.push(request);
    
    // استخدام nextTick لمعالجة الطلبات بعد اكتمال الطلب الحالي
    process.nextTick(() => this.processQueue());
  }

  async processQueue() {
    if (this.isProcessing || this.requestQueue.length === 0) {
      return;
    }

    this.isProcessing = true;

    while (this.requestQueue.length > 0) {
      const request = this.requestQueue.shift();
      
      try {
        console.log(`⏳ معالجة الطلب: ${request.id}`);
        const result = await this.handleRequest(request);
        console.log(`✅ نجحت معالجة الطلب: ${request.id}`);
      } catch (error) {
        console.error(`❌ فشلت معالجة الطلب: ${request.id}`, error);
      }

      // السماح لـ I/O بالمعالجة
      if (this.requestQueue.length > 0) {
        await new Promise(resolve => setImmediate(resolve));
      }
    }

    this.isProcessing = false;
  }

  async handleRequest(request) {
    // محاكاة معالجة غير متزامنة
    return new Promise(resolve => {
      setTimeout(() => resolve(`نتيجة: ${request.data}`), 100);
    });
  }
}

// الاستخدام
const processor = new RequestProcessor();

processor.queueRequest({ id: 1, data: 'طلب أول' });
processor.queueRequest({ id: 2, data: 'طلب ثاني' });
processor.queueRequest({ id: 3, data: 'طلب ثالث' });
```


***

#### `process.memoryUsage()`[^1_8][^1_9][^1_1]

**الصيغة:** `process.memoryUsage([detailed])`

**المعاملات:**


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `detailed` | boolean | اختياري | إذا كان true، يرجع معلومات تفصيلية إضافية |

**القيمة المرجعة:** كائن يحتوي على:

- `rss` (Resident Set Size): إجمالي الذاكرة المخصصة للعملية بالبايت
- `heapTotal`: إجمالي حجم الـ heap المخصص
- `heapUsed`: الجزء من الـ heap المستخدم فعلاً
- `external`: ذاكرة الموارد الخارجية (C++ addons)
- `arrayBuffers`: ذاكرة buffer instances

**مثال عملي واقعي: نظام مراقبة تسرب الذاكرة**

```javascript
// memory-monitor.js
class MemoryMonitor {
  constructor(options = {}) {
    this.threshold = options.threshold || 500; // بالميجابايت
    this.interval = options.interval || 10000; // بالميلي ثانية
    this.history = [];
    this.maxHistorySize = 100;
    this.startMonitoring();
  }

  startMonitoring() {
    setInterval(() => {
      const memUsage = process.memoryUsage();
      const rssInMB = memUsage.rss / 1024 / 1024;
      const heapUsedInMB = memUsage.heapUsed / 1024 / 1024;
      
      const record = {
        timestamp: Date.now(),
        rss: rssInMB,
        heapUsed: heapUsedInMB,
        heapTotal: memUsage.heapTotal / 1024 / 1024
      };

      this.history.push(record);
      if (this.history.length > this.maxHistorySize) {
        this.history.shift();
      }

      this.checkForLeaks(record);
      this.printReport(record);
    }, this.interval);
  }

  checkForLeaks(current) {
    if (this.history.length < 5) return;

    // التحقق من الاتجاه التصاعدي المستمر
    const recentRecords = this.history.slice(-5);
    const isIncreasing = recentRecords.every((rec, i) => 
      i === 0 || rec.rss >= recentRecords[i - 1].rss
    );

    if (isIncreasing && current.rss > this.threshold) {
      console.warn(
        `⚠️ تحذير: قد يكون هناك تسرب ذاكرة! RSS: ${current.rss.toFixed(2)}MB`
      );
      
      // حفظ heap dump للتحليل
      this.captureHeapSnapshot();
    }
  }

  printReport(record) {
    console.log(`
📊 تقرير الذاكرة:
  RSS: ${record.rss.toFixed(2)}MB
  Heap Total: ${record.heapTotal.toFixed(2)}MB
  Heap Used: ${record.heapUsed.toFixed(2)}MB
  Heap Free: ${(record.heapTotal - record.heapUsed).toFixed(2)}MB
  Heap Usage: ${((record.heapUsed / record.heapTotal) * 100).toFixed(2)}%
    `);
  }

  captureHeapSnapshot() {
    // ملاحظة: يتطلب مكتبة heapdump أو استخدام v8 module
    console.log('💾 محاولة حفظ heap snapshot...');
  }

  getReport() {
    if (this.history.length === 0) return null;

    const latest = this.history[this.history.length - 1];
    const min = this.history.reduce((m, r) => r.rss < m.rss ? r : m, latest);
    const max = this.history.reduce((m, r) => r.rss > m.rss ? r : m, latest);

    return {
      current: latest,
      min: min.rss,
      max: max.rss,
      average: (this.history.reduce((sum, r) => sum + r.rss, 0) / this.history.length)
    };
  }
}

// الاستخدام
const monitor = new MemoryMonitor({ threshold: 400, interval: 5000 });

// اختبار: إنشاء بيانات كبيرة
setTimeout(() => {
  const largeArray = new Array(10000000).fill('x');
  console.log('✏️ تم إنشاء مصفوفة كبيرة');
}, 2000);
```


***

#### `process.cpuUsage([previousValue])`[^1_1]

**الصيغة:** `process.cpuUsage([previousValue])`

**المعاملات:**


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `previousValue` | Object | اختياري | نتيجة استدعاء سابق لقياس الفرق |

**القيمة المرجعة:** كائن بخصائص:

- `user`: وقت استخدام CPU بالمايكروثانية (user space)
- `system`: وقت استخدام CPU بالمايكروثانية (system space)

**مثال عملي واقعي: نظام قياس أداء المعالجة**

```javascript
// performance-monitor.js
class PerformanceMonitor {
  measureTaskDuration(taskName, task) {
    const startCpu = process.cpuUsage();
    const startTime = process.hrtime.bigint();

    // تنفيذ المهمة
    const result = task();

    const endTime = process.hrtime.bigint();
    const endCpu = process.cpuUsage(startCpu);

    const realTime = Number(endTime - startTime) / 1_000_000; // بالميلي ثانية
    const userCpu = endCpu.user / 1000; // تحويل إلى ميلي ثانية
    const systemCpu = endCpu.system / 1000;

    return {
      taskName,
      realTime: realTime.toFixed(2) + 'ms',
      userCpu: userCpu.toFixed(2) + 'ms',
      systemCpu: systemCpu.toFixed(2) + 'ms',
      totalCpu: (userCpu + systemCpu).toFixed(2) + 'ms',
      result
    };
  }
}

// مثال الاستخدام
const monitor = new PerformanceMonitor();

// معالجة الصور
const imageProcessResult = monitor.measureTaskDuration('معالجة الصور', () => {
  let sum = 0;
  for (let i = 0; i < 100_000_000; i++) {
    sum += Math.sqrt(i);
  }
  return sum;
});

console.log('نتائج القياس:', imageProcessResult);
```


***

### أحداث الدورة الحياة (Lifecycle Events)

#### حدث `'beforeExit'`[^1_1]

**الوصف:** يُطلق قبل أن تغلق العملية بشكل طبيعي (عندما لا توجد مهام معلقة في event loop). يسمح بجدولة عمل إضافي متزامن.

**مثال عملي:**

```javascript
process.on('beforeExit', (code) => {
  console.log(`العملية على وشك الإنهاء برمز: ${code}`);
  
  // يمكن جدولة عمل إضافي
  process.nextTick(() => {
    console.log('مهمة إضافية');
  });
});

process.on('exit', (code) => {
  console.log(`العملية تنهي الآن برمز: ${code}`);
  // فقط عمليات متزامنة هنا
});

console.log('برنامج البدء');
```


***

#### حدث `'exit'`[^1_1]

**الوصف:** آخر حدث قبل إنهاء العملية. جميع listeners يجب أن تكون متزامنة.

**أخطاء شائعة:**

- محاولة جدولة عمل متزامن (setInterval, setTimeout)
- العمل مع streams (غير آمن)

***

### أحداث معالجة الأخطاء (Error Handling Events)

#### حدث `'uncaughtException'`[^1_10][^1_11][^1_1]

**الصيغة:** `process.on('uncaughtException', (err, origin) => {})`

**المعاملات:**

- `err`: Error object
- `origin`: string ('uncaughtException' أو 'unhandledRejection')

**الوصف:** يُطلق عندما يرمي كود غير معالج exception. يُستخدم كـ last resort لمعالجة الأخطاء.

**مثال عملي واقعي: نظام معالجة الأخطاء المحترف**

```javascript
// error-management.js
const fs = require('fs');
const path = require('path');

class ErrorManagement {
  constructor() {
    this.errorLog = path.join('./logs', 'errors.log');
    this.ensureLogDirectory();
    this.setupErrorHandlers();
  }

  ensureLogDirectory() {
    const logDir = path.dirname(this.errorLog);
    if (!fs.existsSync(logDir)) {
      fs.mkdirSync(logDir, { recursive: true });
    }
  }

  setupErrorHandlers() {
    // معالجة الأخطاء غير المعالجة
    process.on('uncaughtException', (error, origin) => {
      this.logError({
        type: 'uncaughtException',
        origin,
        message: error.message,
        stack: error.stack,
        timestamp: new Date().toISOString()
      });

      // تنفيذ تنظيف متزامن فقط
      this.performSynchronousCleanup();

      // الخروج الآمن
      process.exit(1);
    });

    // معالجة الوعود المرفوضة
    process.on('unhandledRejection', (reason, promise) => {
      this.logError({
        type: 'unhandledRejection',
        reason: reason instanceof Error ? reason.message : String(reason),
        stack: reason instanceof Error ? reason.stack : null,
        timestamp: new Date().toISOString()
      });
    });

    // معالج مراقبة بدون توقف
    process.on('uncaughtExceptionMonitor', (error, origin) => {
      console.error(`📋 تحذير من ${origin}:`, error.message);
    });
  }

  performSynchronousCleanup() {
    console.log('🔄 تنفيذ تنظيف متزامن...');
    
    // مثال: إغلاق ملفات
    // fs.closeSync(fileDescriptor);
    
    // إرسال تنبيه (متزامن فقط)
    try {
      fs.writeFileSync(
        path.join('./logs', 'crash.log'),
        `تحطم في: ${new Date().toISOString()}\n`,
        { flag: 'a' }
      );
    } catch (err) {
      console.error('فشل حفظ سجل التحطم:', err);
    }
  }

  logError(errorInfo) {
    const message = `
[${errorInfo.timestamp}] ${errorInfo.type}
الأصل: ${errorInfo.origin || 'unknown'}
الرسالة: ${errorInfo.message || errorInfo.reason}
الـ Stack: ${errorInfo.stack || 'N/A'}
---
`;
    fs.appendFileSync(this.errorLog, message);
    console.error('❌', errorInfo.message || errorInfo.reason);
  }
}

// الاستخدام
const errorManager = new ErrorManagement();

// اختبار
setTimeout(() => {
  // خطأ غير معالج
  undefined.property.method();
}, 2000);
```


***

#### حدث `'unhandledRejection'`[^1_12][^1_10][^1_1]

**الصيغة:** `process.on('unhandledRejection', (reason, promise) => {})`

**الوصف:** يُطلق عندما يتم رفض Promise بدون معالج `.catch()`.

***

### أحداث الإشارات (Signal Events)

#### معالجة الإشارات (SIGTERM, SIGINT, إلخ)[^1_13][^1_14][^1_1]

**الصيغة:** `process.on('SIGTERM' | 'SIGINT' | 'SIGHUP', () => {})`

**الإشارات الشائعة:**


| الإشارة | التفعيل | الاستخدام |
| :-- | :-- | :-- |
| `SIGINT` | `Ctrl+C` | إيقاف تفاعلي |
| `SIGTERM` | مدير العمليات | إنهاء آمن |
| `SIGHUP` | إغلاق الطرفية | إعادة تحميل الإعدادات |
| `SIGUSR2` | مخصص | تشغيل مراقب |

**مثال عملي واقعي: خادم ويب مع إنهاء آمن**

```javascript
// web-server.js
const http = require('http');
const DatabaseConnection = require('./db-connection');

class WebServer {
  constructor() {
    this.server = null;
    this.db = new DatabaseConnection();
    this.activeConnections = new Set();
    this.shuttingDown = false;
  }

  start(port = 3000) {
    this.server = http.createServer((req, res) => {
      if (this.shuttingDown) {
        res.writeHead(503, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'الخادم يغلق' }));
        return;
      }

      this.handleRequest(req, res);
    });

    this.server.on('connection', (conn) => {
      this.activeConnections.add(conn);
      conn.on('close', () => this.activeConnections.delete(conn));
    });

    // معالجة الإشارات
    this.setupSignalHandlers();

    this.server.listen(port, () => {
      console.log(`✅ الخادم يستمع على المنفذ ${port}`);
    });
  }

  setupSignalHandlers() {
    const gracefulShutdown = async (signal) => {
      console.log(`\n📋 استقبال ${signal} - بدء الإنهاء الآمن`);
      
      this.shuttingDown = true;

      // إغلاق الاتصالات الجديدة
      this.server.close(() => {
        console.log('✅ توقف استقبال الاتصالات الجديدة');
      });

      // إغلاق الاتصالات النشطة
      const closeConnectionsPromise = new Promise((resolve) => {
        let closed = 0;
        const total = this.activeConnections.size;

        if (total === 0) {
          resolve();
          return;
        }

        this.activeConnections.forEach((conn) => {
          conn.destroy();
          closed++;
          if (closed === total) resolve();
        });
      });

      // إغلاق الموارد
      try {
        await closeConnectionsPromise;
        await this.db.close();
        console.log('✅ تم إغلاق جميع الموارد');
        process.exit(0);
      } catch (error) {
        console.error('❌ خطأ أثناء الإنهاء الآمن:', error);
        process.exit(1);
      }

      // إجبار الإنهاء بعد timeout
      setTimeout(() => {
        console.error('⚠️ timeout الإنهاء الآمن - إنهاء قسري');
        process.exit(1);
      }, 30000);
    };

    process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
    process.on('SIGINT', () => gracefulShutdown('SIGINT'));
  }

  handleRequest(req, res) {
    console.log(`${req.method} ${req.url}`);
    
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ 
      status: 'ok',
      timestamp: new Date().toISOString()
    }));
  }
}

// تشغيل الخادم
const server = new WebServer();
server.start(3000);
```


***

## 3. مقارنات واتخاذ القرارات (Comparison \& Decision Making)

### مقارنة: `process.nextTick()` vs `setImmediate()` vs `setTimeout()`[^1_6][^1_7][^1_15][^1_5]

| الميزة | `process.nextTick()` | `setImmediate()` | `setTimeout()` |
| :-- | :-- | :-- | :-- |
| **متى يتم التنفيذ** | قبل event loop | في check phase | بعد الـ timers phase |
| **الأداء** | الأسرع (لا overhead) | سريع نسبياً | بطيء نسبياً |
| **الإشارة** | عملية Node.js | معايير standard | معايير standard |
| **استخدام I/O** | قد يسبب جوع I/O | آمن | آمن |
| **الحالات المثالية** | التنبيهات الفورية | كسر الحلقات المكثفة | تأخيرات منتظمة |
| **المثال** | EventEmitter | معالجة البيانات | polling دوري |

**مثال عملي يوضح الفرق:**

```javascript
// event-loop-demo.js
console.log('1. كود متزامن - الأول');

process.nextTick(() => console.log('2. nextTick - الثاني'));

Promise.resolve().then(() => console.log('3. Promise - الثالث'));

setImmediate(() => console.log('4. setImmediate - الرابع'));

setTimeout(() => console.log('5. setTimeout - الخامس'), 0);

console.log('6. كود متزامن - السادس');

// الناتج:
// 1. كود متزامن - الأول
// 6. كود متزامن - السادس
// 2. nextTick - الثاني
// 3. Promise - الثالث
// 4. setImmediate - الرابع
// 5. setTimeout - الخامس
```


***

### مقارنة: `process.send()` vs أنظمة IPC الأخرى

| النظام | الاستخدام | الأداء | التعقيد |
| :-- | :-- | :-- | :-- |
| `process.send()` | child processes داخلي | عالي جداً | منخفض |
| `net.Socket` | unix sockets | عالي | متوسط |
| `worker_threads` | threads داخل العملية | عالي جداً | متوسط-عالي |
| `RabbitMQ` | أنظمة موزعة | متوسط | عالي جداً |
| `Redis Pub/Sub` | broadcast متزامن | متوسط | عالي |


***

## 4. المزايا والعيوب والحالات الاستخدامية (Pros, Cons \& Use Cases)

### مزايا كائن `process`[^1_2][^1_1]

✅ **تكامل محلي كامل:** لا حاجة لمكتبات خارجية
✅ **أداء ممتازة:** مباشر في Node.js C++
✅ **معالجة الإشارات:** التحكم الكامل بحياة العملية
✅ **مراقبة الموارد:** معلومات CPU والذاكرة الفورية
✅ **IPC محلي:** اتصال سريع بين العمليات

### عيوب وقيود

❌ **نطاق محلي فقط:** لا يدعم عمليات على أجهزة مختلفة
❌ **استهلاك الذاكرة:** كل عملية لها `process` منفصل
❌ **معالجة الأخطاء صعبة:** قد تؤدي الأخطاء لتحطم العملية
❌ **عدم التوافق مع Worker threads:** بعض الخصائص غير متاحة

### الحالات الاستخدامية الأعلى (Top 3 Use Cases)

**1. تطبيقات CLI (Command Line Interface)**

```javascript
// cli-app.js - برنامج معالجة الملفات
const { argv, env, exit } = require('process');

// قراءة معاملات الأوامر والبيئة
const inputFile = argv[^1_2];
const outputFormat = env.OUTPUT_FORMAT || 'json';

if (!inputFile) {
  console.error('استخدام: node cli-app.js <file>');
  exit(1);
}

// معالجة والخروج
console.log(`معالجة: ${inputFile} -> ${outputFormat}`);
```

**2. خوادم الويب (Web Servers)**

```javascript
// server.js - معالجة الإنهاء الآمن
const { on, exit, pid } = require('process');

const server = require('express')();

on('SIGTERM', () => {
  console.log(`[PID ${pid}] إيقاف الخادم...`);
  server.close(() => exit(0));
});
```

**3. تطبيقات معالجة البيانات (Data Processing)**

```javascript
// data-processor.js - مراقبة الموارد
const { memoryUsage, nextTick } = require('process');

async function processLargeDataset(data) {
  for (let i = 0; i < data.length; i++) {
    await new Promise(r => nextTick(r)); // سماح GC بالعمل
    processChunk(data[i]);
    
    // تحقق من الذاكرة
    const { heapUsed } = memoryUsage();
    if (heapUsed > MEMORY_LIMIT) {
      console.log('تنظيف الذاكرة...');
      global.gc?.();
    }
  }
}
```


***

## 5. رسم بياني مرئي: دورة حياة العملية (Visual Explanation)

**شرح المراحل:**

1. **كود متزامن:** تنفيذ فوري للكود الموجود
2. **microtask queue:** Promises و queueMicrotask
3. **nextTick queue:** process.nextTick
4. **Timers phase:** setTimeout, setInterval
5. **Pending callbacks:** عمليات أنظمة معلقة
6. **Poll phase:** I/O callbacks
7. **Check phase:** setImmediate
8. **Close callbacks:** إغلاق الاتصالات

***

## 6. أفضل الممارسات والنصائح المتقدمة

### نصيحة 1: معالجة الأخطاء الشاملة[^1_11][^1_10]

```javascript
// comprehensive-error-handling.js
const { on } = require('process');

// طبقة 1: الأخطاء المتزامنة
process.on('uncaughtException', (error) => {
  console.error('❌ Uncaught:', error);
  process.exit(1);
});

// طبقة 2: الأخطاء غير المتزامنة
process.on('unhandledRejection', (reason) => {
  console.error('❌ Unhandled Promise:', reason);
  // لا تخرج هنا - فقط سجل
});

// طبقة 3: التحذيرات
process.on('warning', (warning) => {
  console.warn(`⚠️ ${warning.name}: ${warning.message}`);
});
```


### نصيحة 2: تجنب تسرب الذاكرة[^1_9][^1_8]

```javascript
// memory-leak-prevention.js
const { memoryUsage, nextTick } = require('process');

class DataProcessor {
  process(largeArray) {
    // معالجة دفعات صغيرة
    for (let i = 0; i < largeArray.length; i += 1000) {
      const batch = largeArray.slice(i, i + 1000);
      
      // السماح بـ GC بين الدفعات
      nextTick(() => {
        this.processBatch(batch);
        
        // تحقق من الذاكرة
        const { heapUsed } = memoryUsage();
        console.log(`Heap: ${(heapUsed / 1024 / 1024).toFixed(2)}MB`);
      });
    }
  }
}
```


### نصيحة 3: استخدام signals لإعادة التحميل الفورية[^1_16]

```javascript
// hot-reload.js
const fs = require('fs');
const { on } = require('process');

let config = require('./config.json');

on('SIGUSR2', () => {
  console.log('📋 إعادة تحميل الإعدادات...');
  config = require('./config.json');
  console.log('✅ تم التحديث');
});
```---

## 7. الخاتمة

كائن `process` هو **القلب** من تطبيقات Node.js، حيث يوفر تحكماً كاملاً على حياة العملية، معالجة الأخطاء، الاتصال بين العمليات، ومراقبة الموارد. الإتقان الجيد لـ `process` يؤدي إلى:

🎯 **تطبيقات أكثر استقراراً:** معالجة صحيحة للأخطاء
🎯 **أداء أفضل:** فهم event loop والـ garbage collection
🎯 **عمليات متزامنة آمنة:** استخدام صحيح للـ IPC
🎯 **إنهاء آمن:** graceful shutdown بدون فقدان البيانات
<span style="display:none">[^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_68][^1_69][^1_70][^1_71][^1_72][^1_73][^1_74][^1_75][^1_76][^1_77][^1_78][^1_79][^1_80][^1_81][^1_82][^1_83][^1_84][^1_85][^1_86][^1_87][^1_88]</span>

<div align="center">⁂</div>

[^1_1]: https://nodejs.org/docs/latest/api/process.html
[^1_2]: https://www.bomberbot.com/node/node-js-process-object-a-comprehensive-guide/
[^1_3]: https://blog.q-bit.me/understanding-the-node-js-process-variable/
[^1_4]: https://www.javascripttutorial.net/nodejs-tutorial/nodejs-process-env/
[^1_5]: https://architchoudhary.hashnode.dev/setimmediate-vs-processnexttick-understanding-the-differences-in-nodejs
[^1_6]: https://www.geeksforgeeks.org/node-js/difference-between-process-nexttick-and-setimmediate-methods/
[^1_7]: https://stackoverflow.com/questions/17502948/nexttick-vs-setimmediate-visual-explanation
[^1_8]: https://www.cloudbees.com/blog/understanding-garbage-collection-in-node-js
[^1_9]: https://nodejs.org/en/learn/diagnostics/memory/understanding-and-tuning-memory
[^1_10]: https://github.com/Best-of-NodeJS/NodeJS-Best-Practices/blob/master/sections/errorhandling/catchunhandledpromiserejection.md
[^1_11]: https://stackoverflow.com/questions/7310521/node-js-best-practice-exception-handling
[^1_12]: https://www.geeksforgeeks.org/node-js/node-js-process-unhandledrejection-event/
[^1_13]: https://stackoverflow.com/questions/42450501/catching-sigterm-vs-catching-sigint
[^1_14]: https://blog.bitsrc.io/proper-way-to-add-graceful-shutdown-nodejs-6c7b35c047aa
[^1_15]: https://www.greatfrontend.com/questions/quiz/what-is-the-difference-between-settimeout-setimmediate-and-processnexttick
[^1_16]: https://www.geeksforgeeks.org/node-js/node-js-process-signal-events/
[^1_17]: https://ieeexplore.ieee.org/document/11256676/
[^1_18]: http://naukaru.ru/en/nauka/textbook/2477/view
[^1_19]: https://ieeexplore.ieee.org/document/10198296/
[^1_20]: https://www.sciopen.com/article/10.26599/AIR.2024.9150038
[^1_21]: https://dl.acm.org/doi/10.1145/3664647.3681530
[^1_22]: http://biorxiv.org/lookup/doi/10.1101/2025.03.20.644282
[^1_23]: https://www.semanticscholar.org/paper/280d16a11c4c98fb11c284380ec2200568a4545f
[^1_24]: https://arxiv.org/abs/2406.11643
[^1_25]: http://www.ecoeet.com/Object-Detection-of-Macroplastic-Waste-Using-Unmanned-Aerial-Vehicles-in-Urban-Canal,189888,0,2.html
[^1_26]: http://portal.acm.org/citation.cfm?doid=62297.62370
[^1_27]: https://arxiv.org/pdf/0912.2861.pdf
[^1_28]: https://zenodo.org/record/4550441/files/MAP-EuroPlop2020aPaper.pdf
[^1_29]: https://dl.acm.org/doi/pdf/10.1145/3615318.3615323
[^1_30]: https://arxiv.org/pdf/2502.13412.pdf
[^1_31]: https://arxiv.org/pdf/2502.09766.pdf
[^1_32]: http://arxiv.org/pdf/2401.07053.pdf
[^1_33]: https://arxiv.org/pdf/2111.07238.pdf
[^1_34]: https://arxiv.org/pdf/2405.13807.pdf
[^1_35]: https://nodejs.org/api/globals.html
[^1_36]: https://www.geeksforgeeks.org/node-js/node-js-global-objects/
[^1_37]: https://stackoverflow.com/questions/43627622/what-is-the-global-object-in-nodejs
[^1_38]: https://snyk.io/blog/10-modern-node-js-runtime-features/
[^1_39]: https://node.readthedocs.io/en/latest/api/globals/
[^1_40]: https://nodejs.org/api/process.html
[^1_41]: https://www.growin.com/blog/javascript-features-for-developers/
[^1_42]: https://nodejs.org/en/learn/asynchronous-work/understanding-setimmediate
[^1_43]: https://bun.com/reference/node/process
[^1_44]: https://www.scribd.com/document/942255214/Node-js-Complete-Material-AY-2024-2025-2
[^1_45]: https://stackoverflow.com/questions/15349733/setimmediate-vs-nexttick
[^1_46]: https://nodejs.org/download/rc/v4.5.0-rc.1/docs/api/globals.html
[^1_47]: https://github.com/goldbergyoni/nodebestpractices
[^1_48]: http://www.journalijdr.com/sites/default/files/issue-pdf/18695.pdf
[^1_49]: http://arxiv.org/pdf/1901.03575.pdf
[^1_50]: http://arxiv.org/pdf/2105.13699.pdf
[^1_51]: http://arxiv.org/pdf/2401.08595.pdf
[^1_52]: https://arxiv.org/pdf/2408.03696.pdf
[^1_53]: http://arxiv.org/pdf/2410.14381.pdf
[^1_54]: https://arxiv.org/html/2401.17403v1
[^1_55]: https://arxiv.org/pdf/1608.04295.pdf
[^1_56]: https://blog.stackademic.com/node-js-standard-streams-stdin-stdout-and-stderr-b71529d6630d
[^1_57]: https://betterstack.com/community/questions/how-to-read-env-vars-in-node-js/
[^1_58]: https://stackoverflow.com/questions/15339379/node-js-spawning-a-child-process-interactively-with-separate-stdout-and-stderr-s
[^1_59]: https://nodejs.org/en/learn/command-line/how-to-read-environment-variables-from-nodejs
[^1_60]: https://nodejs.org/api/child_process.html
[^1_61]: https://stackoverflow.com/questions/16659037/should-i-use-process-nexttick-or-setimmediate-for-asynchronous-iteration
[^1_62]: https://www.geeksforgeeks.org/node-js/node-js-process-env-property/
[^1_63]: https://blog.logrocket.com/using-stdout-stdin-stderr-node-js/
[^1_64]: https://www.linkedin.com/pulse/demystifying-nodejs-event-loop-processnexttick-vs-battle-passos-rf1gf
[^1_65]: https://www.w3schools.com/nodejs/nodejs_environment.asp
[^1_66]: https://nodejs.org/api/stream.html
[^1_67]: https://stackoverflow.com/questions/22312671/setting-environment-variables-for-node-to-retrieve
[^1_68]: https://www.educative.io/answers/setimmediate-vs-processnexttick-in-nodejs
[^1_69]: https://configu.com/blog/node-js-environment-variables-working-with-process-env-and-dotenv/
[^1_70]: http://arxiv.org/pdf/1604.05841.pdf
[^1_71]: https://arxiv.org/pdf/2204.10455.pdf
[^1_72]: http://eudl.eu/pdf/10.4108/eai.14-12-2015.2262678
[^1_73]: https://durham-repository.worktribe.com/preview/1168600/6226.pdf
[^1_74]: http://arxiv.org/pdf/1610.04790.pdf
[^1_75]: https://dl.acm.org/doi/pdf/10.1145/3656389
[^1_76]: http://arxiv.org/pdf/2111.10589.pdf
[^1_77]: http://arxiv.org/pdf/1704.03764.pdf
[^1_78]: https://blog.appsignal.com/2022/09/28/minimize-heap-allocations-in-nodejs.html
[^1_79]: https://shiftasia.com/community/understanding-garbage-collection-in-node-js/
[^1_80]: https://leapcell.io/blog/nodejs-process-exit-strategies
[^1_81]: https://github.com/orgs/nodejs/discussions/51198
[^1_82]: https://www.geeksforgeeks.org/node-js/node-js-process-memoryusage-method/
[^1_83]: https://www.reddit.com/r/node/comments/ro6wso/question_how_to_handle_events_to_safely_terminate/
[^1_84]: https://stackoverflow.com/questions/40867345/catch-all-uncaughtexception-for-node-js-app
[^1_85]: https://stackoverflow.com/questions/12023359/what-do-the-return-values-of-node-js-process-memoryusage-stand-for
[^1_86]: https://www.reddit.com/r/typescript/comments/1324m6b/what_are_the_best_practices_for_adding_fallback/
[^1_87]: https://digitalerena.com/node-js-topic31/
[^1_88]: https://dev.to/yusadolat/nodejs-graceful-shutdown-a-beginners-guide-40b6```


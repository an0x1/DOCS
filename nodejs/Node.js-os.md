
Now I have sufficient information to create the comprehensive documentation. Let me proceed with generating the final report:

# التوثيق العميق لمودولة Node.js: `os` (نظام التشغيل)

## ملخص المحتويات الرئيسية

توثيق شامل ومتعمق لمودولة **`os`** في Node.js، التي توفر مجموعة من الدوال والخصائص للتفاعل مع نظام التشغيل. تغطي هذه الوثيقة **جميع الدوال والخصائص غير المستهلكة (non-deprecated)** مع أمثلة عملية واقعية وتوضيح الحالات الشائعة للأخطاء. تستخدم المودولة للحصول على معلومات النظام (وحدة المعالجة المركزية والذاكرة والشبكة)، وكشف المنصة، وإدارة عمليات النظام بطريقة آمنة وفعالة عبر الأنظمة المختلفة.

***

## 1. نظرة عامة على المودولة (Module Overview)

### مقدمة عن مودولة `os`

مودولة **`os`** هي مودولة أساسية في Node.js توفر واجهة برمجية (API) للتفاعل مع نظام التشغيل الأساسي (Underlying Operating System). تسمح للمطورين بالوصول إلى معلومات حرجة حول النظام دون الاعتماد على مكتبات خارجية معقدة.[^1_1]

### متى تستخدم مودولة `os`؟

يتم استخدام مودولة `os` في السيناريوهات التالية:[^1_2][^1_3][^1_4]

1. **مراقبة النظام (System Monitoring):** بناء أدوات تتبع صحة النظام وقياس استهلاك الموارد في الوقت الفعلي.
2. **الكشف عن المنصة والتوافقية العابرة (Cross-Platform Compatibility):** تكييف السلوك بناءً على نظام التشغيل (مسارات الملفات المختلفة على Windows مقابل POSIX).
3. **تحسين الموارد (Resource Optimization):** توزيع المهام على عدد صحيح من خيوط العمل (`worker_threads`) بناءً على عدد نوى المعالج المتاحة.
4. **تطبيقات الخادم (Server Applications):** الحصول على معلومات الشبكة وإدارة أولويات العمليات.
5. **أدوات DevOps والتشخيص:** جمع بيانات النظام للتسجيل والمراقبة.

### `os` مقابل المكتبات الخارجية

| الميزة | `os` Module | المكتبات الخارجية |
| :-- | :-- | :-- |
| **الحجم** | مدمج (0 تبعيات خارجية) | متغير (قد تكون ثقيلة) |
| **الأداء** | سريع جداً (وصول مباشر للنظام) | قد يكون أبطأ |
| **التجميل** | معلومات أساسية وخام | معالجة متقدمة وتنسيق جميل |
| **الاستقرار** | مستقر جداً (API أساسي) | يعتمد على المكتبة |
| **الحالات** | معلومات عامة عن النظام | مراقبة متقدمة وتحليل معقد |


***

## 2. تفاصيل الدوال والخصائص (API Deep Dive)

### A. معلومات النظام الأساسية (System Information)

#### `os.platform()`

**الدالة الكاملة:**```javascript
os.platform()

```

**الوصف:** تُرجع سلسلة نصية تحدد نظام التشغيل الذي تم ترجمة Node.js عليه. هذه القيمة ثابتة في وقت الترجمة.

| المعامل | النوع | مطلوب/اختياري | الوصف |
|--------|-------|-------------|--------|
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية تمثل المنصة. القيم الممكنة: `'aix'`, `'darwin'` (macOS), `'freebsd'`, `'linux'`, `'openbsd'`, `'sunos'`, `'win32'` (Windows). القيمة `'android'` قد تُرجع على نظام Android (تجريبي).[^1_1]

**مثال عملي واقعي: نظام لاختيار مسار الملفات حسب المنصة**

```javascript
const os = require('os');
const path = require('path');

// دالة للحصول على مسار بيانات التطبيق حسب نظام التشغيل
function getAppDataPath(appName) {
  const platform = os.platform();
  let dataPath;
  
  switch(platform) {
    case 'darwin': // macOS
      dataPath = path.join(os.homedir(), 'Library', 'Application Support', appName);
      break;
    case 'win32': // Windows
      const appData = process.env.APPDATA || os.homedir();
      dataPath = path.join(appData, appName);
      break;
    case 'linux':
      dataPath = path.join(os.homedir(), `.${appName.toLowerCase()}`);
      break;
    default:
      dataPath = path.join(os.homedir(), `.${appName}`);
  }
  
  return dataPath;
}

console.log('App Data Path:', getAppDataPath('MyImageEditor'));
// Windows: C:\Users\username\AppData\Roaming\MyImageEditor
// macOS: /Users/username/Library/Application Support/MyImageEditor
// Linux: /home/username/.myimageeditor
```

**الأخطاء الشائعة:**

1. **الافتراض بأن القيمة ستتغير**: `os.platform()` ثابتة طوال حياة العملية. لا تحتاج إلى استدعاؤها مراراً.
2. **نسيان معالجة الحالات غير المتوقعة**: استخدم دائماً `default` في `switch` للمنصات غير المعروفة.
3. **استخدام قيم `process.platform` مباشرة**: بدلاً من ذلك استخدم `os.platform()` لوضوح أفضل.

***

#### `os.arch()`

**الدالة الكاملة:**

```javascript
os.arch()
```

**الوصف:** تُرجع معمارية المعالج (CPU Architecture) التي تم ترجمة Node.js عليها. هذا يخبرك ما إذا كان النظام 32-bit أم 64-bit أم ARM وغيره.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية. القيم الممكنة: `'arm'`, `'arm64'` (أجهزة ARM 64-bit)، `'ia32'` (32-bit x86)، `'loong64'` (معمارية LOONG)، `'mips'` و `'mipsel'`، `'ppc64'` و `'ppc64le'`، `'riscv64'`، `'s390x'`، `'x64'` (معمارية x86 64-bit الأكثر شيوعاً).[^1_1]

**مثال عملي واقعي: توسيع الميزات بناءً على معمارية المعالج**

```javascript
const os = require('os');
const crypto = require('crypto');

// دالة لاختيار خوارزمية تشفير متقدمة على المعالجات الحديثة
function selectEncryptionMode() {
  const arch = os.arch();
  
  // على المعالجات الحديثة (x64, arm64)
  if (arch === 'x64' || arch === 'arm64') {
    // استخدم AES-256-GCM (authenticated encryption)
    return 'aes-256-gcm';
  }
  
  // على المعالجات الأقدم (ia32)
  if (arch === 'ia32') {
    // استخدم AES-128 (أخف وزناً)
    return 'aes-128-cbc';
  }
  
  // fallback للمعمارات النادرة
  return 'aes-192-cbc';
}

console.log('CPU Architecture:', os.arch());
console.log('Selected Encryption:', selectEncryptionMode());
```

**الأخطاء الشابعة:**

1. **الخلط بين `os.arch()` و `process.arch`**: كلاهما متطابق عملياً، لكن `os.arch()` أكثر وضوحاً.
2. **افتراض أن المعالج المضيف هو نفس معمارية Node.js**: في الحاويات (Containers)، قد تختلف.
3. **عدم التعامل مع المعمارات النادرة**: MIPS و PowerPC موجودة في أجهزة متخصصة.

***

#### `os.type()`

**الدالة الكاملة:**

```javascript
os.type()
```

**الوصف:** تُرجع اسم نظام التشغيل كما يُرجعه `uname(3)` على أنظمة POSIX أو `ver` على Windows. هذا يختلف عن `os.platform()` بأنه يُرجع اسم النظام التفصيلي.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية. القيم الممكنة: `'Linux'` (Linux)، `'Darwin'` (macOS/iOS kernel)، `'Windows_NT'` (Windows NT وما بعده)، `'FreeBSD'`، `'OpenBSD'`، `'SunOS'` (Solaris).[^1_4][^1_1]

**مثال عملي واقعي: تحديد معالجات خاصة بنظام التشغيل**

```javascript
const os = require('os');
const { exec } = require('child_process');

// دالة لتنفيذ أوامر خاصة بكل نظام تشغيل
function getSystemStats() {
  const osType = os.type();
  let command;
  
  if (osType === 'Linux') {
    // استخدم lsb_release و /proc/meminfo على Linux
    command = 'cat /etc/os-release && free -h';
  } else if (osType === 'Darwin') {
    // استخدم system_profiler على macOS
    command = 'system_profiler SPHardwareDataType | grep Memory';
  } else if (osType === 'Windows_NT') {
    // استخدم systeminfo على Windows
    command = 'systeminfo | findstr /C:"Physical Memory"';
  } else {
    console.log('Unknown OS type:', osType);
    return;
  }
  
  exec(command, (error, stdout, stderr) => {
    if (error) {
      console.error('Error executing command:', error);
      return;
    }
    console.log('System Stats:\n', stdout);
  });
}

console.log('OS Type:', os.type());
getSystemStats();
```

**الأخطاء الشابعة:**

1. **الخلط بين `os.type()` و `os.platform()`**: `type()` أكثر تفصيلاً.
2. **افتراض قيم محددة على Windows**: قد تحصل على `'Windows_NT'` بصيغ مختلفة في الإصدارات القديمة.
3. **عدم توثيق الأوامر الخاصة بكل نظام**: سهل التطبيق لكن معقد للصيانة.

***

#### `os.release()`

**الدالة الكاملة:**

```javascript
os.release()
```

**الوصف:** تُرجع رقم إصدار نظام التشغيل. يُحسب من `uname(3)` على POSIX وفrom `GetVersionExW()` على Windows.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية تمثل رقم الإصدار. أمثلة:[^1_1]

- Linux: `'5.15.0-46-generic'`
- macOS: `'21.6.0'`
- Windows 10: `'10.0.19042'`
- Windows 11: `'10.0.22621'`

**مثال عملي واقعي: منع تشغيل التطبيق على إصدارات نظام قديمة جداً**

```javascript
const os = require('os');
const semver = require('semver'); // مكتبة اختيارية للمقارنة

// دالة للتحقق من توافق الإصدار
function checkSystemCompatibility() {
  const release = os.release();
  const osType = os.type();
  
  // حدد الحد الأدنى المقبول لكل نظام تشغيل
  const minimumVersions = {
    'Windows_NT': '10.0.17763', // Windows Server 2019
    'Darwin': '10.13',           // macOS High Sierra
    'Linux': '4.15'              // kernel قديم نسبياً
  };
  
  const minVersion = minimumVersions[osType];
  
  if (!minVersion) {
    console.warn('Unknown OS type, compatibility check skipped');
    return true;
  }
  
  // مقارنة بسيطة (للإنتاج استخدم semver)
  if (release < minVersion) {
    console.error(
      `Your ${osType} version (${release}) is older than the minimum required (${minVersion}). ` +
      `Please upgrade your operating system.`
    );
    return false;
  }
  
  console.log(`✓ System is compatible (${osType} ${release})`);
  return true;
}

checkSystemCompatibility();
```

**الأخطاء الشابعة:**

1. **عدم معالجة صيغ الإصدار المختلفة**: Windows و Linux يستخدمان صيغ مختلفة تماماً.
2. **استخدام مقارنة نصية بسيطة**: استخدم مكتبة `semver` للمقارنات الموثوقة.
3. **الافتراض بأن رقم الإصدار الأعلى = أفضل دائماً**: قد يكون هناك تقهقر في الأداء.

***

#### `os.version()`

**الدالة الكاملة:**

```javascript
os.version()
```

**الوصف:** تُرجع سلسلة نصية تحديد إصدار Kernel. يختلف عن `os.release()` لأنه يتضمن معلومات بناء إضافية (build information).


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية. أمثلة:[^1_4][^1_1]

- Windows: `'Windows 10 Enterprise 10.0.19044 (Build 19044)'`
- Linux: `'#49-Ubuntu SMP Tue Aug 2 08:49:28 UTC 2022'`
- macOS: `'Darwin Kernel Version 21.6.0: ...'`

**مثال عملي واقعي: تسجيل معلومات النظام المفصلة لأغراض التشخيص**

```javascript
const os = require('os');
const fs = require('fs');
const path = require('path');

// دالة لإنشاء تقرير نظام مفصل للدعم الفني
function generateSystemReport(outputFile = 'system-report.txt') {
  const report = {
    timestamp: new Date().toISOString(),
    platform: os.platform(),
    arch: os.arch(),
    type: os.type(),
    release: os.release(),
    version: os.version(), // معلومات kernel مفصلة
    uptime: os.uptime(),
    hostname: os.hostname(),
    cpuCount: os.cpus().length,
    totalMemory: `${(os.totalmem() / 1024 / 1024 / 1024).toFixed(2)} GB`,
    freeMemory: `${(os.freemem() / 1024 / 1024 / 1024).toFixed(2)} GB`,
    nodeVersion: process.version,
    v8Version: process.versions.v8
  };
  
  // اكتب التقرير إلى ملف
  const reportText = Object.entries(report)
    .map(([key, value]) => `${key}: ${value}`)
    .join('\n');
  
  fs.writeFileSync(outputFile, reportText);
  console.log(`System report saved to: ${outputFile}`);
  console.log(reportText);
}

generateSystemReport();
```

**الأخطاء الشابعة:**

1. **الخلط بين `os.version()` و `os.release()`**: كلاهما متعلق بالإصدار لكن يختلف التفاصيل.
2. **محاولة تحليل سلسلة `version()` النصية**: صيغتها غير ثابتة وقد تختلف.
3. **استخدام `version()` لمقارنة الإصدارات**: استخدم `release()` بدلاً منها.

***

#### `os.machine()`

**الدالة الكاملة:**

```javascript
os.machine()
```

**الوصف:** تُرجع نوع الجهاز (Machine Type) كسلسلة نصية. يُحسب من `uname(3)` على POSIX و `RtlGetVersion()` على Windows. هذا أكثر تفصيلاً من `os.arch()`.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية. أمثلة: `'arm'`, `'arm64'`, `'aarch64'`, `'mips'`, `'mips64'`, `'ppc64'`, `'ppc64le'`, `'s390x'`, `'i386'`, `'i686'`, `'x86_64'`.[^1_1]

**مثال عملي واقعي: تحسين الأداء بناءً على نوع الجهاز**

```javascript
const os = require('os');

// دالة لتحديد استراتيجية معالجة الصور حسب الجهاز
function getImageProcessingConfig() {
  const machine = os.machine();
  
  const config = {
    // أجهزة ARM محمولة: معالجة خفيفة
    arm: {
      maxThreads: 2,
      jpegQuality: 75,
      cacheSizeMB: 32,
      useGPU: false
    },
    // أجهزة ARM 64-bit: أقوى قليلاً
    arm64: {
      maxThreads: 4,
      jpegQuality: 85,
      cacheSizeMB: 128,
      useGPU: true
    },
    // x86_64 desktop: أداء عالي
    x86_64: {
      maxThreads: os.cpus().length,
      jpegQuality: 95,
      cacheSizeMB: 512,
      useGPU: true
    },
    // fallback
    default: {
      maxThreads: 2,
      jpegQuality: 80,
      cacheSizeMB: 64,
      useGPU: false
    }
  };
  
  return config[machine] || config.default;
}

console.log('Machine Type:', os.machine());
console.log('Image Processing Config:', getImageProcessingConfig());
```

**الأخطاء الشابعة:**

1. **استخدام `os.machine()` لاكتشاف نوع الهاتف**: استخدم النوى (cores) والذاكرة بدلاً منها.
2. **افتراض أن كل `arm` هي نفسها**: ARM لديها أنواع متعددة (v6, v7, v8...).
3. **عدم اختبار على أجهزة حقيقية**: المحاكاة قد لا تعكس السلوك الحقيقي.

***

### B. معلومات وحدة المعالجة المركزية والموارد (CPU \& Resource Information)

#### `os.cpus()`

**الدالة الكاملة:**

```javascript
os.cpus()
```

**الوصف:** تُرجع مصفوفة من الكائنات تحتوي على معلومات حول كل نواة منطقية في المعالج. كل كائن يحتوي على معلومات أداء الـ CPU.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** مصفوفة من الكائنات. كل كائن يحتوي على:[^1_1]

- `model` (String): اسم نموذج CPU
- `speed` (Number): سرعة الساعة بـ MHz
- `times` (Object):
    - `user`: ملي ثانية في وضع المستخدم
    - `nice`: ملي ثانية في وضع nice (POSIX فقط)
    - `sys`: ملي ثانية في وضع النظام
    - `idle`: ملي ثانية في وضع الخمول
    - `irq`: ملي ثانية في وضع معالجة المقاطعات

**مثال عملي واقعي: حساب استخدام CPU الفعلي**

```javascript
const os = require('os');

// دالة لحساب نسبة استخدام CPU
function calculateCPUUsage(interval = 1000) {
  // قراءة أولية
  const cpus1 = os.cpus();
  
  setTimeout(() => {
    // قراءة ثانية بعد فترة
    const cpus2 = os.cpus();
    
    // حساب الفرق لكل نواة
    cpus1.forEach((cpu1, index) => {
      const cpu2 = cpus2[index];
      
      // مجموع الوقت الكلي
      const totalTime1 = Object.values(cpu1.times).reduce((a, b) => a + b, 0);
      const totalTime2 = Object.values(cpu2.times).reduce((a, b) => a + b, 0);
      const totalTimeDiff = totalTime2 - totalTime1;
      
      // وقت الخمول
      const idleDiff = cpu2.times.idle - cpu1.times.idle;
      
      // نسبة الاستخدام
      const usagePercent = 100 - ~~(100 * idleDiff / totalTimeDiff);
      
      console.log(`Core ${index + 1}: ${usagePercent}% used`);
    });
  }, interval);
}

// مثال: قياس استخدام CPU
console.log('Measuring CPU usage for 1 second...');
calculateCPUUsage(1000);

// قائمة معلومات CPU
console.log('\nCPU Information:');
const cpus = os.cpus();
cpus.forEach((cpu, index) => {
  console.log(`Core ${index + 1}:`);
  console.log(`  Model: ${cpu.model}`);
  console.log(`  Speed: ${cpu.speed} MHz`);
});
```

**الأخطاء الشابعة:**

1. **استخدام `os.cpus().length` لتحديد عدد خيوط العمل**: استخدم `os.availableParallelism()` بدلاً منها.[^1_5][^1_6][^1_7]
2. **عدم التعامل مع تأخير القراءات**: يجب انتظار بعض الوقت بين قراءتين لحساب استخدام CPU دقيق.
3. **الافتراض بأن جميع الأنوية متشابهة**: قد تختلف سرعات الساعة.
4. **إهمال قيم `nice`**: معدومة على Windows والمهمة على Unix.

***

#### `os.availableParallelism()`

**الدالة الكاملة:**

```javascript
os.availableParallelism()
```

**الوصف:** تُرجع تقديراً للحد الأدنى من درجة التوازي (Parallelism) التي يجب على البرنامج استخدامها. هذه دالة **حديثة وموصى بها** بدلاً من `os.cpus().length`.[^1_6][^1_7][^1_5][^1_1]


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** عدد صحيح موجب دائماً (أكثر من الصفر). تأخذ في الاعتبار:[^1_1]

- عدد النوى الفعلي
- قيود الحاوية (Docker/Kubernetes cgroups)
- متغيرات البيئة مثل `OMP_NUM_THREADS`

**مثال عملي واقعي: إنشاء pool من worker threads بحجم محسوب تلقائياً**

```javascript
const os = require('os');
const { Worker } = require('worker_threads');
const path = require('path');

class WorkerPool {
  constructor(workerScript, options = {}) {
    this.workerScript = workerScript;
    this.workers = [];
    this.taskQueue = [];
    this.activeWorkers = 0;
    
    // استخدم availableParallelism() الحديثة
    this.poolSize = options.poolSize || os.availableParallelism();
    this.maxQueueSize = options.maxQueueSize || 1000;
    
    console.log(`Initialized worker pool with ${this.poolSize} workers`);
    this.initializeWorkers();
  }
  
  initializeWorkers() {
    for (let i = 0; i < this.poolSize; i++) {
      const worker = new Worker(this.workerScript);
      
      worker.on('message', (result) => {
        const task = this.currentTask;
        if (task) {
          task.resolve(result);
        }
        this.processNextTask();
      });
      
      worker.on('error', (error) => {
        const task = this.currentTask;
        if (task) {
          task.reject(error);
        }
        this.processNextTask();
      });
      
      this.workers.push(worker);
    }
  }
  
  async executeTask(data) {
    return new Promise((resolve, reject) => {
      if (this.taskQueue.length >= this.maxQueueSize) {
        reject(new Error('Task queue is full'));
        return;
      }
      
      this.taskQueue.push({ data, resolve, reject });
      this.processNextTask();
    });
  }
  
  processNextTask() {
    if (this.taskQueue.length === 0 || this.activeWorkers >= this.poolSize) {
      return;
    }
    
    const task = this.taskQueue.shift();
    this.activeWorkers++;
    
    const worker = this.workers[this.activeWorkers - 1];
    this.currentTask = task;
    worker.postMessage(task.data);
  }
  
  terminate() {
    this.workers.forEach(w => w.terminate());
  }
}

// الاستخدام
console.log(`System supports ${os.availableParallelism()} parallel tasks`);
// const pool = new WorkerPool('./worker.js');
```

**الأخطاء الشابعة:**

1. **استخدام `os.cpus().length` في الحاويات**: لن تحترم القيود المفروضة.[^1_5]
2. **عدم معالجة الفشل في أنظمة نادرة**: قد ترجع 1 كـ fallback.
3. **إنشاء أكثر من `availableParallelism()` من خيوط**: سيؤدي للتنافس على الموارد.

***

#### `os.totalmem()` و `os.freemem()`

**الدوال الكاملة:**

```javascript
os.totalmem()
os.freemem()
```

**الوصف:**

- `os.totalmem()`: تُرجع إجمالي الذاكرة المتاحة للنظام بالبايت.
- `os.freemem()`: تُرجع الذاكرة المتاحة حالياً بالبايت.

| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدوال لا تقبل معاملات |

**قيمة الإرجاع:** عدد صحيح يمثل البايتات.[^1_1]

**مثال عملي واقعي: مراقب ذاكرة في الوقت الفعلي مع تنبيهات**

```javascript
const os = require('os');

class MemoryMonitor {
  constructor(options = {}) {
    this.warningThreshold = options.warningThreshold || 0.8; // 80%
    this.criticalThreshold = options.criticalThreshold || 0.95; // 95%
    this.checkInterval = options.checkInterval || 5000; // 5 seconds
    this.startMonitoring();
  }
  
  getMemoryStats() {
    const total = os.totalmem();
    const free = os.freemem();
    const used = total - free;
    const percentUsed = used / total;
    
    return {
      total: this.formatBytes(total),
      free: this.formatBytes(free),
      used: this.formatBytes(used),
      percentUsed: (percentUsed * 100).toFixed(2) + '%',
      rawPercent: percentUsed
    };
  }
  
  formatBytes(bytes) {
    const units = ['B', 'KB', 'MB', 'GB'];
    let size = bytes;
    let unitIndex = 0;
    
    while (size >= 1024 && unitIndex < units.length - 1) {
      size /= 1024;
      unitIndex++;
    }
    
    return `${size.toFixed(2)} ${units[unitIndex]}`;
  }
  
  startMonitoring() {
    setInterval(() => {
      const stats = this.getMemoryStats();
      
      if (stats.rawPercent > this.criticalThreshold) {
        console.error(`🔴 CRITICAL: Memory usage is ${stats.percentUsed}`);
        this.onCritical?.(stats);
      } else if (stats.rawPercent > this.warningThreshold) {
        console.warn(`🟠 WARNING: Memory usage is ${stats.percentUsed}`);
        this.onWarning?.(stats);
      } else {
        console.log(`✓ Normal: Memory usage is ${stats.percentUsed}`);
      }
    }, this.checkInterval);
  }
}

// الاستخدام
const monitor = new MemoryMonitor({
  warningThreshold: 0.7,
  criticalThreshold: 0.9,
  checkInterval: 10000
});

monitor.onCritical = (stats) => {
  // قد نقرر تقليل حجم الـ cache أو إعادة تشغيل عملية
  console.log('Taking action: Clearing caches...');
};

monitor.onWarning = (stats) => {
  // قد نسجل هذا الحدث للتحليل
  console.log('Logging memory warning...');
};
```

**الأخطاء الشابعة:**

1. **الخلط بين الذاكرة المجانية والذاكرة المتاحة**: `freemem()` قد تشمل ذاكرة مخزنة مؤقتاً.
2. **قراءة واحدة فقط**: استخدم المراقبة المستمرة لفهم الاتجاهات.
3. **عدم حساب استخدام العملية الحالية**: استخدم `process.memoryUsage()` للعملية الحالية.

***

#### `os.uptime()`

**الدالة الكاملة:**

```javascript
os.uptime()
```

**الوصف:** تُرجع مدة تشغيل النظام بالثواني.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** عدد عشري يمثل الثواني.[^1_1]

**مثال عملي واقعي: تحديد ما إذا كان النظام قد أعيد تشغيله مؤخراً**

```javascript
const os = require('os');

// دالة للتحقق من استقرار النظام بناءً على فترة التشغيل
function getSystemStability() {
  const uptimeSeconds = os.uptime();
  const uptimeDays = uptimeSeconds / (24 * 60 * 60);
  
  let stability;
  if (uptimeDays < 1) {
    stability = '⚠️ UNSTABLE - Recently rebooted';
  } else if (uptimeDays < 7) {
    stability = '⚠️ UNSTABLE - Rebooted within the week';
  } else if (uptimeDays < 30) {
    stability = '✓ FAIR - Running for several weeks';
  } else if (uptimeDays < 365) {
    stability = '✓✓ GOOD - Running for several months';
  } else {
    stability = '✓✓✓ EXCELLENT - Running for over a year';
  }
  
  return {
    seconds: uptimeSeconds,
    days: uptimeDays.toFixed(2),
    stability
  };
}

console.log('System Uptime:', getSystemStability());

// مثال آخر: مراقبة إعادة التشغيل
function hasRebootedRecently(hoursThreshold = 24) {
  const uptimeSeconds = os.uptime();
  const hoursUp = uptimeSeconds / 3600;
  return hoursUp < hoursThreshold;
}

console.log('Rebooted recently?', hasRebootedRecently(12) ? 'Yes' : 'No');
```

**الأخطاء الشابعة:**

1. **عدم التعامل مع الوقت الكسري**: استخدم `Math.floor()` إذا أردت أيام كاملة فقط.
2. **افتراض أن uptime عالي = لا توجد مشاكل**: قد يشير إلى عدم الصيانة.
3. **عدم مقارنة الـ uptime بملفات السجل**: استخدم مع logs لفهم السلوك الكامل.

***

#### `os.loadavg()`

**الدالة الكاملة:**

```javascript
os.loadavg()
```

**الوصف:** تُرجع متوسط حمل النظام (System Load Average) للـ 1 و 5 و 15 دقيقة الأخيرة. هذه قيمة Unix فقط - ترجع `[0, 0, 0]` على Windows.[^1_1]


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** مصفوفة بـ 3 أرقام عشرية: `[load1min, load5min, load15min]`.[^1_1]

**مثال عملي واقعي: قرار ديناميكي للقبول/الرفض بناءً على حمل النظام**

```javascript
const os = require('os');

// دالة للتحكم الديناميكي في قبول الطلبات
function shouldAcceptRequest() {
  const loadAvg = os.loadavg();
  const cpuCount = os.availableParallelism();
  const load1min = loadAvg[^1_0];
  
  // حمل النظام = عدد العمليات المتوسط
  // إذا كان load > cpuCount، النظام مثقل الحمل
  const loadRatio = load1min / cpuCount;
  
  return {
    currentLoad: load1min.toFixed(2),
    cpuCount,
    loadRatio: loadRatio.toFixed(2),
    shouldAccept: loadRatio < 0.8, // اقبل الطلبات إذا كان الحمل أقل من 80%
    recommendation: loadRatio > 1.0 ? 'REJECT - System overloaded' : 
                    loadRatio > 0.8 ? 'THROTTLE - System busy' :
                    'ACCEPT - System healthy'
  };
}

console.log('Load Analysis:', shouldAcceptRequest());

// مثال متقدم: Middleware للتحكم في معدل الطلبات
const express = require('express');
const app = express();

function loadBalancingMiddleware(req, res, next) {
  const loadAvg = os.loadavg();
  const cpuCount = os.availableParallelism();
  const load1min = loadAvg[^1_0];
  const loadRatio = load1min / cpuCount;
  
  if (loadRatio > 2.0) {
    // النظام محمل بشكل خطير - رفض الطلب
    return res.status(503).json({ error: 'Service temporarily unavailable' });
  }
  
  if (loadRatio > 1.5) {
    // النظام مشغول - أضف تأخير عشوائي
    const delay = Math.random() * 1000;
    return setTimeout(() => next(), delay);
  }
  
  // النظام سليم
  next();
}

// app.use(loadBalancingMiddleware);
```

**الأخطاء الشابعة:**

1. **استخدام `loadavg()` على Windows**: ترجع دائماً `[0, 0, 0]`.
2. **الخلط بين load average والاستخدام الفعلي**: load 4 على نظام 4-core لا يعني 100% استخدام.
3. **عدم التمييز بين أنواع المهام**: I/O-bound و CPU-bound لهما تأثيرات مختلفة على load.

***

### C. معلومات المستخدم والبيئة (User \& Environment Information)

#### `os.homedir()`

**الدالة الكاملة:**

```javascript
os.homedir()
```

**الوصف:** تُرجع مسار المجلد الرئيسي للمستخدم الحالي.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية تمثل المسار:[^1_1]

- POSIX: تستخدم متغير البيئة `$HOME` أم UID لو لم يكن معرفاً
- Windows: تستخدم `USERPROFILE` أو مسار الملف الشخصي

**مثال عملي واقعي: إنشاء مجلدات إعدادات التطبيق الخاصة بالمستخدم**

```javascript
const os = require('os');
const path = require('path');
const fs = require('fs');

// دالة لإنشاء بيئة إعدادات التطبيق
function initializeAppConfig(appName) {
  const homeDir = os.homedir();
  const platform = os.platform();
  
  let configDir;
  
  // مسار مختلف حسب نظام التشغيل (متابعة أفضل الممارسات)
  if (platform === 'darwin') {
    // macOS: ~/Library/Application Support/AppName
    configDir = path.join(homeDir, 'Library', 'Application Support', appName);
  } else if (platform === 'win32') {
    // Windows: %APPDATA%/AppName
    configDir = path.join(homeDir, 'AppData', 'Roaming', appName);
  } else {
    // Linux: ~/.appname
    configDir = path.join(homeDir, `.${appName.toLowerCase()}`);
  }
  
  // تأكد من وجود المجلد
  if (!fs.existsSync(configDir)) {
    fs.mkdirSync(configDir, { recursive: true });
    console.log(`Created config directory: ${configDir}`);
  }
  
  // أنشئ ملف إعدادات افتراضي
  const configFile = path.join(configDir, 'config.json');
  if (!fs.existsSync(configFile)) {
    const defaultConfig = {
      theme: 'light',
      language: 'en',
      version: '1.0.0'
    };
    fs.writeFileSync(configFile, JSON.stringify(defaultConfig, null, 2));
  }
  
  return configDir;
}

const appConfigDir = initializeAppConfig('MyDataProcessor');
console.log('Config directory:', appConfigDir);

// مثال آخر: حفظ البيانات المؤقتة
function getCacheDirectory(appName) {
  const homeDir = os.homedir();
  const cacheDir = path.join(homeDir, `.${appName}-cache`);
  
  if (!fs.existsSync(cacheDir)) {
    fs.mkdirSync(cacheDir, { recursive: true });
  }
  
  return cacheDir;
}
```

**الأخطاء الشابعة:**

1. **استخدام hardcoded paths**: استخدم `os.homedir()` دائماً لتوافقية أفضل.
2. **عدم التعامل مع الحالات النادرة**: قد يكون المستخدم بدون home directory (على سبيل المثال، بعض خوادم CI/CD).
3. **تجاهل قيم متغيرات البيئة**: المستخدم قد يعيد تعريف `HOME` أو `USERPROFILE`.

***

#### `os.hostname()`

**الدالة الكاملة:**

```javascript
os.hostname()
```

**الوصف:** تُرجع اسم المضيف (Hostname) لنظام التشغيل.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** سلسلة نصية تمثل اسم المضيف. مثال: `'my-laptop'` أو `'web-server-01'`.[^1_1]

**مثال عملي واقعي: نظام تسجيل موزع يتتبع الخادم المصدر**

```javascript
const os = require('os');
const fs = require('fs');
const path = require('path');

// دالة للتسجيل المركزي مع معلومات الخادم
class DistributedLogger {
  constructor(logDir = './logs') {
    this.hostname = os.hostname();
    this.logDir = logDir;
    this.ensureLogDirectory();
  }
  
  ensureLogDirectory() {
    if (!fs.existsSync(this.logDir)) {
      fs.mkdirSync(this.logDir, { recursive: true });
    }
  }
  
  log(level, message, metadata = {}) {
    const timestamp = new Date().toISOString();
    const logEntry = {
      timestamp,
      level,
      hostname: this.hostname,
      message,
      ...metadata
    };
    
    // اكتب في ملف خاص بكل خادم
    const logFile = path.join(
      this.logDir,
      `${this.hostname}-${level.toLowerCase()}.log`
    );
    
    fs.appendFileSync(
      logFile,
      JSON.stringify(logEntry) + '\n'
    );
    
    // أيضاً، أرسل إلى خدمة تسجيل مركزية
    this.sendToRemoteLogger(logEntry);
  }
  
  sendToRemoteLogger(entry) {
    // محاكاة إرسال إلى ELK Stack أو Splunk
    console.log(`[${entry.hostname}] ${entry.level}: ${entry.message}`);
    // في الإنتاج: استخدم axios أو fetch لإرسال البيانات
  }
  
  info(message, metadata) { this.log('INFO', message, metadata); }
  error(message, metadata) { this.log('ERROR', message, metadata); }
  warn(message, metadata) { this.log('WARN', message, metadata); }
}

// الاستخدام
const logger = new DistributedLogger();
logger.info('Application started', { version: '1.0.0' });
logger.error('Database connection failed', { code: 'ECONNREFUSED' });
```

**الأخطاء الشابعة:**

1. **افتراض أن hostname فريد**: قد يكون متطابقاً في عدة خوادم.
2. **عدم التعامل مع أسماء hostname المتغيرة**: استخدم identifiers ثابتة أو environment variables.
3. **تخزين hostname في cache بدون تحديث**: أعد استدعاء الدالة إذا كنت تتوقع تغييرات.

***

#### `os.userInfo([options])`

**الدالة الكاملة:**

```javascript
os.userInfo([options])
```

**الوصف:** تُرجع كائناً يحتوي على معلومات المستخدم الحالي الفعال (Effective User).


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `options` | Object | اختياري |  |
| `options.encoding` | String | اختياري | ترميز الحرف: `'utf8'` (افتراضي) أو `'buffer'` |

**قيمة الإرجاع:** كائن يحتوي على:[^1_1]

- `uid` (Number/BigInt): معرّف المستخدم (POSIX فقط، -1 على Windows)
- `gid` (Number/BigInt): معرّف المجموعة (POSIX فقط، -1 على Windows)
- `username` (String/Buffer): اسم المستخدم
- `homedir` (String/Buffer): مسار المجلد الرئيسي
- `shell` (String/Buffer): shell الافتراضي (null على Windows)

**مثال عملي واقعي: تحديد صلاحيات النظام المتاحة للتطبيق**

```javascript
const os = require('os');

// دالة للتحقق من صلاحيات التشغيل
function checkUserPermissions() {
  const userInfo = os.userInfo();
  const isRoot = userInfo.uid === 0;
  const platform = os.platform();
  
  return {
    username: userInfo.username,
    homeDirectory: userInfo.homedir,
    isRoot,
    canAccessSystemFiles: isRoot || platform === 'win32',
    capabilities: {
      canReadPrivateFiles: isRoot,
      canModifySystemConfig: isRoot || (platform === 'win32' && isAdmin()),
      canAccessNetwork: true,
      canCreateTempFiles: true,
      canAccessHomeDir: true
    }
  };
}

function isAdmin() {
  // محاكاة - في الإنتاج ستحتاج إلى فحص أكثر دقة
  return false;
}

console.log('User Permissions:', checkUserPermissions());

// مثال متقدم: تقييد صلاحيات العملية
function enforceSecurityPolicy() {
  const userInfo = os.userInfo();
  
  // منع التشغيل كـ root في بيئة الإنتاج
  if (userInfo.uid === 0 && process.env.NODE_ENV === 'production') {
    console.error('⚠️  WARNING: Running as root in production is not recommended!');
    console.error('   Please create a dedicated non-root user for this application.');
  }
  
  // سجل معلومات المستخدم للتدقيق
  console.log(`Application running as: ${userInfo.username} (UID: ${userInfo.uid})`);
}

enforceSecurityPolicy();
```

**الأخطاء الشابعة:**

1. **عدم التعامل مع الأخطاء**: إذا لم يكن للمستخدم `username` أو `homedir`، تُرجع `SystemError`.
2. **افتراض `uid`/`gid` على Windows**: هذه القيم دائماً -1.
3. **الاعتماد على `shell` للبحث عن البرامج**: قد تكون null أو تُشير إلى برنامج غير موجود.

***

### D. معلومات الشبكة (Network Information)

#### `os.networkInterfaces()`

**الدالة الكاملة:**

```javascript
os.networkInterfaces()
```

**الوصف:** تُرجع كائناً يحتوي على معلومات واجهات الشبكة والعناوين المخصصة لها.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| (بدون معاملات) | - | - | هذه الدالة لا تقبل معاملات |

**قيمة الإرجاع:** كائن بنية: `{ [interfaceName]: [addressArray] }`. كل عنوان يحتوي على:[^1_1]

- `address` (String): عنوان IPv4 أو IPv6
- `netmask` (String): قناع الشبكة
- `family` (String): `'IPv4'` أو `'IPv6'`
- `mac` (String): عنوان MAC
- `internal` (Boolean): `true` إذا كانت الواجهة محلية (loopback)
- `scopeid` (Number): معرف النطاق IPv6 فقط
- `cidr` (String): عنوان مع بادئة التوجيه (CIDR)

**مثال عملي واقعي: اكتشاف عناوين IP المحلية لسيرفر**

```javascript
const os = require('os');

// دالة للحصول على عنوان IP المحلي الأساسي
function getLocalIpAddress() {
  const interfaces = os.networkInterfaces();
  
  for (const name of Object.keys(interfaces)) {
    const addresses = interfaces[name];
    
    for (const addr of addresses) {
      // تجاهل العناوين الداخلية والـ IPv6
      if (addr.family === 'IPv4' && !addr.internal) {
        return addr.address;
      }
    }
  }
  
  return '127.0.0.1'; // fallback
}

// دالة لإدراج جميع واجهات الشبكة مع تفاصيلها
function listNetworkInterfaces() {
  const interfaces = os.networkInterfaces();
  const report = {};
  
  for (const [name, addresses] of Object.entries(interfaces)) {
    report[name] = addresses.map(addr => ({
      address: addr.address,
      family: addr.family,
      mac: addr.mac,
      internal: addr.internal,
      cidr: addr.cidr
    }));
  }
  
  return report;
}

console.log('Local IP:', getLocalIpAddress());
console.log('Network Interfaces:', listNetworkInterfaces());

// مثال متقدم: اختيار الواجهة المناسبة لسيرفر ويب
function getServerBindAddress() {
  const interfaces = os.networkInterfaces();
  
  // أولويات الاختيار:
  // 1. Ethernet (eth0, en0)
  // 2. أي واجهة عامة
  // 3. localhost
  
  const prioritizedNames = ['eth0', 'en0', 'en1', 'wlan0'];
  
  for (const name of prioritizedNames) {
    if (interfaces[name]) {
      for (const addr of interfaces[name]) {
        if (addr.family === 'IPv4' && !addr.internal) {
          return addr.address;
        }
      }
    }
  }
  
  // fallback: أي واجهة عامة
  for (const [name, addresses] of Object.entries(interfaces)) {
    for (const addr of addresses) {
      if (addr.family === 'IPv4' && !addr.internal) {
        console.log(`Using interface ${name}: ${addr.address}`);
        return addr.address;
      }
    }
  }
  
  return 'localhost';
}

console.log('Server Bind Address:', getServerBindAddress());
```

**الأخطاء الشابعة:**

1. **تجاهل العناوين IPv6**: قد يكون لديك عناوين IPv6 مهمة.
2. **الاعتماد على ترتيب معين للواجهات**: الترتيب قد يتغير.
3. **عدم التعامل مع الواجهات المؤقتة**: قد تكون هناك VPN أو واجهات افتراضية أخرى.
4. **افتراض عنوان MAC موجود دائماً**: قد تكون null على بعض الأنظمة.

***

### E. إدارة العمليات (Process Management)

#### `os.getPriority([pid])` و `os.setPriority([pid,] priority)`

**الدوال الكاملة:**

```javascript
os.getPriority([pid])
os.setPriority([pid,] priority)
```

**الوصف:** تسمح بقراءة وتعديل أولوية جدولة العمليات (Scheduling Priority). الأولوية العالية = احتصول على وقت CPU أكثر.


| المعامل | النوع | مطلوب/اختياري | الوصف |
| :-- | :-- | :-- | :-- |
| `pid` | Number | اختياري | معرّف العملية. افتراضي: 0 (العملية الحالية) |
| `priority` | Number | مطلوب (setPriority فقط) | القيمة: -20 (أعلى أولوية) إلى 19 (أقل أولوية) |

**قيمة الإرجاع:**[^1_1]

- `getPriority()`: عدد صحيح يمثل الأولوية الحالية
- `setPriority()`: لا تُرجع قيمة (void)

**الأخطاء المحتملة:**

- `ESRCH`: العملية المطلوبة غير موجودة
- `EPERM`: لا توجد صلاحيات كافية (يتطلب غالباً root/admin)
- `EINVAL`: قيمة أولوية غير صحيحة

**مثال عملي واقعي: إدارة أولويات المهام المختلفة**

```javascript
const os = require('os');
const { spawn } = require('child_process');

// دالة لتشغيل عملية بأولوية محددة
function runProcessWithPriority(command, args, priority = 0) {
  const child = spawn(command, args);
  
  try {
    // عيّن أولوية العملية
    os.setPriority(child.pid, priority);
    console.log(`Process ${child.pid} started with priority ${priority}`);
  } catch (error) {
    if (error.code === 'EPERM') {
      console.error('Insufficient permissions to set priority (may need root)');
    } else {
      console.error('Error setting priority:', error.message);
    }
  }
  
  return child;
}

// مثال: عملية معالجة صور عالية الأولوية
// const imageProcessor = runProcessWithPriority('node', ['image-processor.js'], -5);

// مثال متقدم: ديناميكي تعديل الأولويات حسب الحمل
class AdaptivePriorityManager {
  constructor() {
    this.processes = new Map();
  }
  
  registerProcess(pid, taskType) {
    this.processes.set(pid, { taskType, createdAt: Date.now() });
  }
  
  adjustPriorities() {
    const loadAvg = os.loadavg();
    const cpuCount = os.availableParallelism();
    const loadRatio = loadAvg[^1_0] / cpuCount;
    
    for (const [pid, info] of this.processes) {
      let newPriority = 0;
      
      // تقليل الأولويات إذا كان النظام مثقل الحمل
      if (loadRatio > 2.0) {
        // أولويات منخفضة لجميع المهام
        newPriority = 10;
      } else if (loadRatio > 1.0) {
        // أولويات متوسطة
        if (info.taskType === 'background') {
          newPriority = 5;
        } else if (info.taskType === 'interactive') {
          newPriority = -5;
        }
      } else {
        // النظام سليم - أعطِ كل واحد أولويته الطبيعية
        newPriority = info.taskType === 'interactive' ? -5 : 0;
      }
      
      try {
        os.setPriority(pid, newPriority);
      } catch (error) {
        // تجاهل الأخطاء إذا لم تعد العملية موجودة
        if (error.code === 'ESRCH') {
          this.processes.delete(pid);
        }
      }
    }
  }
}

// الاستخدام
const manager = new AdaptivePriorityManager();

// التحقق من الأولوية الحالية
const currentPriority = os.getPriority();
console.log('Current process priority:', currentPriority);
```

**الأخطاء الشابعة:**

1. **محاولة تعيين أولويات عالية سالبة كمستخدم عادي**: يتطلب صلاحيات root/admin.
2. **عدم معالجة الأخطاء عند تعديل أولويات العمليات الأخرى**: قد لا تكون لديك الصلاحيات.
3. **افتراض أن الأولويات لها تأثير فوري**: قد يستغرق وقتاً لظهور التأثير.
4. **على Windows**: الأولويات تُعين إلى 6 فئات فقط بدلاً من النطاق الكامل -20 إلى 19.

***

### F. الثوابت والخصائص المتقدمة (Constants \& Advanced Properties)

#### `os.constants`

**الخاصية الكاملة:**

```javascript
os.constants
```

**الوصف:** كائن يحتوي على ثوابت نظام التشغيل (System Constants) الشائعة: إشارات العمليات (Signals)، رموز الأخطاء (Error Codes)، و ثوابت الأولويات.

**الأقسام الرئيسية:**[^1_1]

##### أ) `os.constants.signals` - إشارات العمليات

مثال:[^1_1]

```javascript
const os = require('os');
console.log(os.constants.signals);
// Output:
// {
//   SIGHUP: 1,
//   SIGINT: 2,
//   SIGQUIT: 3,
//   SIGILL: 4,
//   ...
//   SIGTERM: 15,
// }
```

**مثال عملي واقعي: معالج الإشارات المتقدمة**

```javascript
const os = require('os');

// دالة لتسجيل معالجات الإشارات تلقائياً
function setupSignalHandlers() {
  const signals = os.constants.signals;
  
  // معالجات الإشارات المهمة
  const handlerMap = {
    SIGINT: () => {
      console.log('\n✓ Received SIGINT (Ctrl+C). Shutting down gracefully...');
      cleanup();
      process.exit(0);
    },
    
    SIGTERM: () => {
      console.log('✓ Received SIGTERM. Shutting down gracefully...');
      cleanup();
      process.exit(0);
    },
    
    SIGUSR1: () => {
      console.log('✓ Received SIGUSR1. Reloading configuration...');
      reloadConfig();
    },
    
    SIGUSR2: () => {
      console.log('✓ Received SIGUSR2. Dumping diagnostics...');
      dumpDiagnostics();
    }
  };
  
  for (const [signal, handler] of Object.entries(handlerMap)) {
    if (signals[signal]) {
      process.on(signal, handler);
      console.log(`Registered handler for ${signal}`);
    }
  }
}

function cleanup() {
  console.log('Cleaning up resources...');
  // أغلق الاتصالات، احفظ الحالة، إلخ
}

function reloadConfig() {
  console.log('Reloading configuration...');
  // أعد تحميل ملفات الإعدادات
}

function dumpDiagnostics() {
  console.log('=== Diagnostics Report ===');
  console.log('Memory:', os.freemem() / 1024 / 1024 / 1024, 'GB free');
  console.log('Uptime:', os.uptime() / 3600, 'hours');
  console.log('Load Average:', os.loadavg());
}

setupSignalHandlers();
```


##### ب) `os.constants.errno` - رموز الأخطاء

مثال: `ENOENT` (ملف غير موجود)، `EACCES` (رفع الوصول مرفوض)، إلخ.

##### ج) `os.constants.priority` - ثوابت الأولويات

```javascript
const os = require('os');
console.log(os.constants.priority);
// Output:
// {
//   PRIORITY_LOW: 19,
//   PRIORITY_BELOW_NORMAL: 10,
//   PRIORITY_NORMAL: 0,
//   PRIORITY_ABOVE_NORMAL: -7,
//   PRIORITY_HIGH: -14,
//   PRIORITY_HIGHEST: -20
// }
```

**مثال عملي:[**

```javascript
const os = require('os');

// دالة لتعيين أولوية باستخدام ثوابت موثوقة
function setPriorityByLevel(pid, level) {
  const priorityMap = {
    'idle': os.constants.priority.PRIORITY_LOW,
    'normal': os.constants.priority.PRIORITY_NORMAL,
    'high': os.constants.priority.PRIORITY_HIGH,
    'realtime': os.constants.priority.PRIORITY_HIGHEST
  };
  
  const priority = priorityMap[level];
  if (priority === undefined) {
    throw new Error(`Unknown priority level: ${level}`);
  }
  
  try {
    os.setPriority(pid, priority);
    console.log(`Set process ${pid} to ${level} priority`);
  } catch (error) {
    console.error(`Failed to set priority: ${error.message}`);
  }
}

// الاستخدام
setPriorityByLevel(process.pid, 'high');
```


#### `os.EOL` و `os.devNull` و `os.tmpdir()`

**الخصائص الكاملة:**

```javascript
os.EOL           // نهاية السطر
os.devNull       // جهاز null
os.tmpdir()      // مجلد الملفات المؤقتة
```

**الوصف:**[^1_1]

- `os.EOL`: الفاصل الخاص بنهاية السطر (`'\n'` على POSIX, `'\r\n'` على Windows)
- `os.devNull`: مسار جهاز null (`'/dev/null'` على POSIX, `'\\.\\nul'` على Windows)
- `os.tmpdir()`: مجلد الملفات المؤقتة للنظام

**مثال عملي واقعي: معالج نصوص متعدد المنصات**

```javascript
const os = require('os');
const fs = require('fs');
const path = require('path');

// دالة لقراءة ملف نصي بدون الاعتماد على نهايات السطر المختلفة
function readCrossplatformFile(filepath) {
  const content = fs.readFileSync(filepath, 'utf8');
  // اقسم حسب أي نهاية سطر
  return content.split(/\r?\n/);
}

// دالة لكتابة ملف بـ EOL الصحيح
function writeCrossplatformFile(filepath, lines) {
  const content = lines.join(os.EOL);
  fs.writeFileSync(filepath, content, 'utf8');
}

// دالة لتشغيل أمر بدون أن نرى outputه (redirect إلى /dev/null)
function runSilently(command) {
  const { execSync } = require('child_process');
  execSync(command, {
    stdio: ['pipe', 'ignore', 'ignore'] // stdout و stderr إلى /dev/null
  });
}

// دالة لإنشاء ملف مؤقت
function createTemporaryFile(prefix = 'app-', suffix = '.tmp') {
  const tmpDir = os.tmpdir();
  const tmpFile = path.join(
    tmpDir,
    `${prefix}${Date.now()}${suffix}`
  );
  
  fs.writeFileSync(tmpFile, '');
  console.log(`Created temporary file: ${tmpFile}`);
  
  // عند الانتهاء: fs.unlinkSync(tmpFile);
  return tmpFile;
}

console.log('EOL:', JSON.stringify(os.EOL));
console.log('devNull:', os.devNull);
console.log('tmpdir:', os.tmpdir());
```


***

## 3. المقارنات واتخاذ القرارات (Comparisons \& Decision Making)

### المقارنة بين طرق الحصول على عدد النوى

| الطريقة | المميزات | العيوب | الاستخدام |
| :-- | :-- | :-- | :-- |
| `os.cpus().length` | بسيطة ومباشرة | تتجاهل قيود Docker/cgroups | legacy code فقط |
| `os.availableParallelism()` | حديثة، تحترم القيود، موصى بها من Node.js | تتطلب Node.js 19+ | **الخيار الأول** |
| `require('os').cpus().length` | نفس `os.cpus()` | نفس المشاكل | عدم الاستخدام |

**التوصية:** استخدم دائماً `os.availableParallelism()` في الكود الحديث.[^1_7][^1_6][^1_5]

### المقارنة بين طرق مراقبة الموارد

| المتغير | الغرض | التحديث | الاستخدام |
| :-- | :-- | :-- | :-- |
| `os.freemem()` | الذاكرة المجانية الفورية | واحدة فقط | لقطة سريعة |
| `process.memoryUsage()` | ذاكرة العملية الحالية فقط | في الوقت الفعلي | تتبع تسرب الذاكرة |
| `os.loadavg()` | حمل النظام العام | محسوبة ديناميكياً | اتجاهات الأداء |
| مكتبات خارجية (systeminformation) | مراقبة متقدمة | مستمرة | شاملة لكن ثقيلة |


***

## 4. المميزات والعيوب وحالات الاستخدام (Pros, Cons \& Use Cases)

### المميزات (Pros) لمودولة `os`:[^1_3][^1_2][^1_4]

1. **مدمجة في Node.js**: لا توجد تبعيات خارجية (Zero Dependencies).
2. **أداء عالي**: وصول مباشر إلى النظام الأساسي دون overhead.
3. **متعددة المنصات**: نفس الكود يعمل على Windows و Linux و macOS و BSD.
4. **توافقية طويلة**: API مستقر وثابت عبر إصدارات Node.js.
5. **بسيطة وموثقة بشكل جيد**: سهلة التعلم والاستخدام.
6. **معايير منخفضة للدراسة**: لا توجد منحنى تعلم شديد.

### العيوب (Cons) لمودولة `os`:[^1_8]

1. **معلومات أساسية فقط**: لا توفر مراقبة متقدمة (استخدام CPU الفعلي في الوقت الفعلي).
2. **عدم توفر بعض التفاصيل**: لا تعطي معلومات عن عمليات معينة (استخدم `ps` أو مكتبات مخصصة).
3. **قراءة واحدة فقط**: إذا أردت حساب نسب التغيير، يجب قراءة مرتين مع تأخير.
4. **قيود على الأنظمة المحدودة**: في الحاويات، قد لا توفر معلومات دقيقة بدون قراءة cgroups.
5. **معلومات stateless**: لا تحتفظ بسجل تاريخي.

### حالات الاستخدام الأساسية (Top 3 Use Cases):

#### 1. **مراقبة صحة النظام وتنبيهات الأداء**

- بناء لوحة معلومات تظهر استخدام CPU والذاكرة.
- إرسال تنبيهات عندما يتجاوز الاستخدام حد معين.
- تسجيل مقاييس الأداء للتحليل.

**مثال:**

```javascript
const os = require('os');

setInterval(() => {
  const freeMem = os.freemem();
  const totalMem = os.totalmem();
  const percentUsed = ((totalMem - freeMem) / totalMem * 100).toFixed(2);
  
  if (percentUsed > 80) {
    console.warn(`⚠️  High memory usage: ${percentUsed}%`);
  }
}, 30000);
```


#### 2. **توزيع المهام على نوى المعالج**

- استخدام `os.availableParallelism()` لتحديد عدد worker threads.
- معالجة متوازية للبيانات الكبيرة (صور، JSON، إلخ).
- تحسين الأداء على أنظمة متعددة النوى.

**مثال:**

```javascript
const os = require('os');
const { Worker } = require('worker_threads');

const numWorkers = os.availableParallelism();
const workers = [];

for (let i = 0; i < numWorkers; i++) {
  workers.push(new Worker('./worker.js'));
}
```


#### 3. **التطبيقات متعددة المنصات والتكيف**

- اختيار المسارات الصحيحة (Windows vs. Unix).
- استخدام أوامر مختلفة حسب نظام التشغيل.
- حفظ ملفات الإعدادات في المجلدات الموحدة.

**مثال:**

```javascript
const os = require('os');
const path = require('path');

function getConfigPath(appName) {
  const home = os.homedir();
  
  if (os.platform() === 'darwin') {
    return path.join(home, 'Library/Application Support', appName);
  } else if (os.platform() === 'win32') {
    return path.join(home, 'AppData/Roaming', appName);
  } else {
    return path.join(home, `.${appName}`);
  }
}
```---

## الخلاصة والتوصيات النهائية

مودولة **`os`** في Node.js هي أداة قوية وموثوقة للتفاعل مع نظام التشغيل. **التوصيات الرئيسية:**

1. **استخدم `os.availableParallelism()`** بدلاً من `os.cpus().length` لتحديد عدد العمليات.

2. **راقب الموارد بشكل مستمر** باستخدام `setInterval()` وليس بقراءة واحدة.

3. **اختبر على منصات متعددة** (Windows, macOS, Linux) قبل النشر.

4. **استخدم الثوابت من `os.constants`** بدلاً من hardcoding القيم.

5. **تعامل مع الأخطاء بعناية** خاصة عند التعامل مع الأولويات والصلاحيات.

6. **للمراقبة المتقدمة** استخدم مكتبات مثل `systeminformation` أو `pm2`.

بهذه الوثيقة الشاملة، لديك الآن فهم عميق لكل دالة وخاصية في مودولة `os` مع أمثلة واقعية وقابلة للاستخدام الفوري في مشاريعك.[^1_2][^1_3][^1_4][^1_1]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://nodejs.org/api/os.html
[^1_2]: https://www.geeksforgeeks.org/node-js/node-js-os/
[^1_3]: https://www.softpost.org/nodejs/os-module-in-node-js
[^1_4]: https://www.w3schools.com/nodejs/nodejs_os.asp
[^1_5]: https://stackoverflow.com/questions/26049013/node-js-get-number-of-processors-available
[^1_6]: https://shapkarin.me/articles/node.js-multi-core-cluster/
[^1_7]: https://blog.appsignal.com/2021/02/03/improving-node-application-performance-with-clustering.html
[^1_8]: https://www.npmjs.com/package/systeminformation
[^1_9]: https://www.semanticscholar.org/paper/70cb8385c32baf37d808faf6d111515b23ee7b19
[^1_10]: https://www.semanticscholar.org/paper/5d26db42561feea7ada6942d5871757fa77b21c4
[^1_11]: https://www.semanticscholar.org/paper/10ac33cbfa4da3d05b6716b1e02f6856af8e521e
[^1_12]: http://arxiv.org/pdf/1704.07887.pdf
[^1_13]: http://arxiv.org/pdf/2401.08595.pdf
[^1_14]: https://arxiv.org/pdf/1801.06144.pdf
[^1_15]: https://zenodo.org/record/5727094/files/main.pdf
[^1_16]: http://arxiv.org/pdf/2404.19614.pdf
[^1_17]: https://arxiv.org/pdf/2101.00756.pdf
[^1_18]: https://arxiv.org/pdf/2203.13737.pdf
[^1_19]: http://arxiv.org/pdf/2310.17318.pdf
[^1_20]: https://www.geeksforgeeks.org/node-js/node-js-os-cpus-method/
[^1_21]: https://www.youtube.com/watch?v=9mF7T77x6jM
[^1_22]: https://www.codingtag.com/os-in-nodejs
[^1_23]: https://nodejs.org/download/release/v5.0.0/docs/api/os.html
[^1_24]: https://www.youtube.com/watch?v=_dl3hxSQSis
[^1_25]: https://www.javascripttutorial.net/nodejs-tutorial/nodejs-os-module/
[^1_26]: https://www.w3schools.com/nodejs/nodejs_modules.asp
[^1_27]: https://nodejs.org/download/release/v5.4.1/docs/api/os.html
[^1_28]: https://www.geeksforgeeks.org/node-js/node-js-os-complete-reference/
[^1_29]: https://www.w3schools.com.cach3.com/nodejs/ref_os.asp.html
[^1_30]: https://www.docs4dev.com/docs/node/6_lts/os.html
[^1_31]: https://docs.deno.com/api/node/os/
[^1_32]: https://www.w3resource.com/node.js/nodejs-os.php
[^1_33]: https://nodejs-es.github.io/api/en/os.html
[^1_34]: https://bun.com/reference/node/os
[^1_35]: http://haxefoundation.github.io/hxnodejs/js/node/Os.html
[^1_36]: https://www.semanticscholar.org/paper/668bbb6d7e3b70df1bdf3156d75618d6a02dabaa
[^1_37]: https://academic.oup.com/comjnl/article-pdf/60/1/60/10329287/bxw065.pdf
[^1_38]: http://arxiv.org/pdf/2410.21036.pdf
[^1_39]: https://dl.acm.org/doi/pdf/10.1145/3650200.3656597
[^1_40]: https://arxiv.org/pdf/1012.3452.pdf
[^1_41]: https://www.epj-conferences.org/articles/epjconf/pdf/2019/19/epjconf_chep2018_08024.pdf
[^1_42]: http://arxiv.org/pdf/1705.10756.pdf
[^1_43]: http://arxiv.org/pdf/2408.06130.pdf
[^1_44]: http://arxiv.org/pdf/2311.11095.pdf
[^1_45]: https://devcrud.com/node-js-os-module-system-operations-cpu-memory-network-info/
[^1_46]: https://zetcode.com/python/os-getpriority/
[^1_47]: https://www.digitalocean.com/community/tutorials/how-to-use-multithreading-in-node-js
[^1_48]: https://zetcode.com/python/os-setpriority/
[^1_49]: https://www.linkedin.com/pulse/benchmarking-concurrency-go-vs-nodejs-worker-threads-sukanta-majhi-wpenc
[^1_50]: https://cuvette.tech/blog/priority-scheduling-in-os-a-comprehensive-guide
[^1_51]: https://stackoverflow.com/questions/10332565/how-to-get-system-statistics-with-node-js
[^1_52]: https://www.geeksforgeeks.org/node-js/node-js-os-getpriority-method/
[^1_53]: https://www.luisllamas.es/en/get-system-information-with-nodejs/
[^1_54]: https://www.geeksforgeeks.org/operating-systems/priority-scheduling-in-operating-system/
[^1_55]: https://github.com/mrmlnc/fast-glob/issues/420
[^1_56]: https://data-flair.training/blogs/priority-scheduling-algorithm-in-operating-system/
[^1_57]: https://ayon.li/node-js-cluster-but-with-worker-threads```


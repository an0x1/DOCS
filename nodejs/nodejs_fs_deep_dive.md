# Deep Dive: Node.js `fs` Module - شرح عميق لوحدة نظام الملفات

---

## 1. نظرة عامة على وحدة fs

### مقدمة
وحدة `fs` (File System) في Node.js توفر واجهة برمجية شاملة للتفاعل مع نظام الملفات. وهي تتيح قراءة وكتابة وحذف وتعديل الملفات والمجلدات بطرق متعددة. الوحدة مبنية على معايير POSIX (Portable Operating System Interface) وتوفر ثلاث طرق للتعامل مع العمليات: **Callbacks**، **Promises**، و**Synchronous**.

### متى تستخدم fs بدلاً من مكتبات خارجية؟

| الحالة | الخيار الأفضل |
|--------|------------|
| عمليات أساسية (قراءة، كتابة، حذف) | `fs` - بدون مكتبات إضافية |
| معالجة مسارات معقدة | استخدم `path` مع `fs` |
| نسخ ملفات بخيارات متقدمة | `fs.cp()` (Node.js 16.7+) بدلاً من مكتبات |
| مراقبة تغييرات الملفات | `fs.watch()` أو `fs.watchFile()` |
| عمليات مجلدات معقدة | `fs` كافي للعمليات العادية |
| معالجة ملفات ضخمة جداً (> 1GB) | استخدم **Streams** من `fs` |
| تجريف أنماط ملفات (glob) | `fs.glob()` (Node.js 16.10+) |

---

## 2. العمليات الأساسية: القراءة والكتابة

### 2.1 قراءة الملفات

#### `fs.readFile(path, [options], callback)` - Callback API

**التوقيع:**
```javascript
fs.readFile(path, [options], callback)
```

**المعاملات:**
| المعامل | النوع | مطلوب؟ | الوصف |
|--------|-------|--------|--------|
| `path` | string/Buffer/URL | ✓ | مسار الملف المراد قراءته |
| `options` | Object/string | ✗ | خيارات القراءة (encoding, flag, signal) |
| `encoding` | string | ✗ | الترميز: 'utf8', 'base64', 'hex', الخ. الافتراضي: null (يرجع Buffer) |
| `flag` | string | ✗ | العلامات: 'r' (قراءة - افتراضي), 'r+', الخ |
| `signal` | AbortSignal | ✗ | للإلغاء المرن للعملية |
| `callback` | Function | ✓ | (err, data) => {...} |

**مثال عملي واقعي - قراءة ملف إعدادات JSON:**

```javascript
const fs = require('fs');
const path = require('path');

// مثال حقيقي: قراءة ملف إعدادات التطبيق
const configPath = path.join(__dirname, 'config.json');

fs.readFile(configPath, 'utf8', (err, data) => {
  if (err) {
    // معالجة الأخطاء المختلفة
    if (err.code === 'ENOENT') {
      console.error('ملف الإعدادات غير موجود');
      return;
    }
    if (err.code === 'EACCES') {
      console.error('لا توجد أذونات كافية لقراءة الملف');
      return;
    }
    console.error('خطأ غير متوقع:', err);
    return;
  }

  try {
    const config = JSON.parse(data);
    console.log('الإعدادات تم تحميلها بنجاح:', config);
    // استخدم الإعدادات هنا
  } catch (parseErr) {
    console.error('خطأ في تحليل JSON:', parseErr.message);
  }
});
```

**الأخطاء الشائعة:**
❌ عدم التحقق من نوع الخطأ (ENOENT مقابل EACCES مقابل EISDIR)
❌ محاولة استخدام البيانات قبل انتهاء العملية
❌ فتح ملف ضخم جداً وتحميله كله في الذاكرة (استخدم streams بدلاً من ذلك)

---

#### `fsPromises.readFile(path, [options])` - Promise API (الطريقة الحديثة الموصى بها)

**التوقيع:**
```javascript
fsPromises.readFile(path, [options])  // يرجع Promise
```

**مثال عملي - مع async/await:**

```javascript
const fs = require('fs').promises;
const path = require('path');

async function loadApplicationConfig() {
  try {
    const configPath = path.join(process.cwd(), 'app-config.json');
    
    // قراءة الملف كـ UTF-8
    const configData = await fs.readFile(configPath, 'utf8');
    
    // تحليل JSON
    const config = JSON.parse(configData);
    
    console.log('✓ تم تحميل الإعدادات');
    return config;
    
  } catch (err) {
    // معالجة شاملة للأخطاء
    if (err.code === 'ENOENT') {
      console.error('الملف غير موجود. جاري إنشاء إعدادات افتراضية...');
      return getDefaultConfig();
    }
    if (err instanceof SyntaxError) {
      console.error('تنسيق JSON غير صحيح');
      throw err;
    }
    throw err;
  }
}

// الاستخدام
loadApplicationConfig()
  .then(config => console.log(config))
  .catch(err => console.error('فشل:', err));
```

**مزايا Promise API:**
✓ كود أنظف وأسهل في القراءة
✓ معالجة أخطاء موحدة مع try/catch
✓ دعم async/await الحديث
✓ تكامل أفضل مع Promises الأخرى

---

#### `fs.readFileSync(path, [options])` - Synchronous API

**التوقيع:**
```javascript
fs.readFileSync(path, [options])  // يحجب Event Loop
```

**مثال:**

```javascript
const fs = require('fs');

try {
  // استخدام آمن فقط في بداية البرنامج
  const configData = fs.readFileSync('./config.json', 'utf8');
  const config = JSON.parse(configData);
  console.log('الإعدادات:', config);
} catch (err) {
  console.error('خطأ:', err.message);
  process.exit(1);  // أوقف البرنامج في حالة الخطأ
}
```

**⚠️ تحذير مهم:**
- لا تستخدم Sync في معالجات الطلبات (request handlers)
- تستخدم فقط في: بداية التطبيق، CLI tools، أو scripts
- تحجب Event Loop وتوقف معالجة جميع الطلبات الأخرى

---

### 2.2 كتابة الملفات

#### `fs.writeFile(path, data, [options], callback)`

**التوقيع:**
```javascript
fs.writeFile(path, data, [options], callback)
```

**المعاملات:**
| المعامل | النوع | الوصف |
|--------|-------|--------|
| `path` | string/Buffer/URL | مسار الملف |
| `data` | string/Buffer | البيانات المراد كتابتها |
| `options.flag` | string | 'w' (الكتابة والاستبدال), 'a' (إضافة), 'wx' (خطأ إذا موجود), إلخ |
| `options.encoding` | string | 'utf8' (افتراضي), 'ascii', 'base64', إلخ |
| `options.mode` | integer | أذونات الملف (مثل 0o644) |
| `options.signal` | AbortSignal | للإلغاء المرن |

**مثال عملي - حفظ سجلات الأخطاء:**

```javascript
const fs = require('fs').promises;
const path = require('path');

class ErrorLogger {
  constructor(logDir = './logs') {
    this.logDir = logDir;
  }

  async logError(error, context = {}) {
    try {
      // إنشاء المجلد إذا لم يكن موجوداً
      await fs.mkdir(this.logDir, { recursive: true });

      const timestamp = new Date().toISOString();
      const logEntry = {
        timestamp,
        message: error.message,
        stack: error.stack,
        context,
      };

      const logPath = path.join(
        this.logDir, 
        `errors-${new Date().toISOString().split('T')[0]}.json`
      );

      // قراءة السجل الموجود (إن وجد)
      let logs = [];
      try {
        const existingData = await fs.readFile(logPath, 'utf8');
        logs = JSON.parse(existingData);
      } catch (e) {
        // الملف غير موجود أو فارغ - لا مشكلة
      }

      // إضافة السجل الجديد
      logs.push(logEntry);

      // حفظ السجل (مع استبدال كامل الملف)
      await fs.writeFile(
        logPath, 
        JSON.stringify(logs, null, 2),
        { encoding: 'utf8', mode: 0o644 }
      );

      console.log(`✓ تم تسجيل الخطأ: ${logPath}`);
    } catch (writeErr) {
      console.error('فشل تسجيل الخطأ:', writeErr);
    }
  }
}

// الاستخدام
const logger = new ErrorLogger();
try {
  throw new Error('خطأ في معالجة البيانات');
} catch (err) {
  await logger.logError(err, { userId: 123, action: 'fetchData' });
}
```

**الأخطاء الشائعة:**
❌ استخدام 'w' دائماً (يستبدل الملف) - استخدم 'a' للإضافة
❌ عدم تحديد التشفير للنصوص - استخدم 'utf8' صراحة
❌ محاولة الكتابة إلى مجلد غير موجود - أنشئ المجلد أولاً

---

### 2.3 إضافة بيانات إلى ملف

#### `fs.appendFile(path, data, [options], callback)`

**التوقيع:**
```javascript
fs.appendFile(path, data, [options], callback)
```

**مثال عملي - تسجيل أحداث التطبيق:**

```javascript
const fs = require('fs').promises;
const path = require('path');

class EventLogger {
  constructor(logPath = './app-events.log') {
    this.logPath = logPath;
  }

  async logEvent(event, level = 'INFO') {
    const timestamp = new Date().toISOString();
    const logLine = `[${timestamp}] ${level}: ${event}\n`;

    try {
      await fs.appendFile(this.logPath, logLine, 'utf8');
    } catch (err) {
      console.error('فشل تسجيل الحدث:', err);
    }
  }

  async getUserActivity(userId, action) {
    const event = `User ${userId} performed: ${action}`;
    await this.logEvent(event, 'USER_ACTION');
  }
}

// الاستخدام
const logger = new EventLogger('./events.log');
await logger.logEvent('تم بدء التطبيق', 'START');
await logger.getUserActivity(42, 'login');
await logger.logEvent('معالجة الطلب اكتملت', 'SUCCESS');
```

**الفرق بين writeFile و appendFile:**

| العملية | writeFile | appendFile |
|--------|-----------|-----------|
| السلوك | يستبدل الملف كاملاً | يضيف في النهاية |
| استخدام الذاكرة | يحمل البيانات كاملة | أقل استهلاكاً |
| السرعة | أبطأ للملفات الكبيرة | أسرع للإضافات |
| الحالات | حفظ الحالة، الإعدادات | السجلات، الأحداث |

---

## 3. العمليات المتقدمة

### 3.1 العمل مع FileHandle (Handle العالي المستوى)

#### `fsPromises.open(path, flags, [mode])` - فتح ملف

**التوقيع:**
```javascript
fsPromises.open(path, flags, [mode])
```

**الأعلام (Flags):**
| العلم | الوصف |
|------|--------|
| `'r'` | قراءة فقط |
| `'w'` | كتابة (إنشاء أو استبدال) |
| `'a'` | إضافة |
| `'r+'` | قراءة وكتابة |
| `'w+'` | إنشاء أو استبدال مع القراءة |
| `'wx'` | كتابة (خطأ إذا كان موجوداً) |
| `'ax'` | إضافة (خطأ إذا كان موجوداً) |

**مثال عملي - معالجة ملف كبير بقراءة أجزاء محددة:**

```javascript
const fs = require('fs').promises;

async function readFileChunks(filePath, chunkSize = 1024) {
  let fileHandle;
  try {
    // فتح الملف بـ read mode
    fileHandle = await fs.open(filePath, 'r');
    
    // الحصول على معلومات الملف
    const stats = await fileHandle.stat();
    console.log(`حجم الملف: ${stats.size} بايت`);

    // قراءة الملف على دفعات
    const buffer = Buffer.alloc(chunkSize);
    let position = 0;
    let chunkNumber = 1;

    while (position < stats.size) {
      const { bytesRead, buffer: chunk } = await fileHandle.read(
        buffer,
        0,           // offset في buffer
        chunkSize,   // عدد البايتات المراد قراءتها
        position     // الموضع في الملف
      );

      console.log(`الجزء ${chunkNumber}: تم قراءة ${bytesRead} بايت`);
      
      // معالجة الجزء
      const data = chunk.toString('utf8', 0, bytesRead);
      processChunk(data);

      position += bytesRead;
      chunkNumber++;

      if (bytesRead < chunkSize) break;  // انتهى الملف
    }

  } finally {
    // إغلاق الملف دائماً
    if (fileHandle) {
      await fileHandle.close();
    }
  }
}

function processChunk(data) {
  console.log(`معالجة الجزء: ${data.substring(0, 50)}...`);
}

// الاستخدام
await readFileChunks('./large-file.txt', 4096);
```

**طرق FileHandle المهمة:**
| الطريقة | الوصف |
|--------|--------|
| `read(buffer, offset, length, position)` | قراءة بايتات محددة |
| `write(buffer/string, offset, length, position)` | كتابة بيانات |
| `readFile([options])` | قراءة الملف كاملاً |
| `writeFile(data, [options])` | كتابة الملف كاملاً |
| `truncate(len)` | قص الملف إلى حجم معين |
| `sync()` | مزامنة مع القرص |
| `datasync()` | مزامنة البيانات فقط |
| `stat([options])` | معلومات الملف |
| `close()` | إغلاق الملف |

---

### 3.2 العمل مع المجلدات

#### `fsPromises.readdir(path, [options])`

**التوقيع:**
```javascript
fsPromises.readdir(path, [options])
```

**المعاملات:**
| المعامل | الوصف |
|--------|--------|
| `path` | مسار المجلد |
| `options.encoding` | ترميز الأسماء ('utf8' افتراضي, 'buffer') |
| `options.withFileTypes` | ترجع Dirent objects بدلاً من strings |
| `options.recursive` | قراءة المجلدات فرعية (Node.js 18.17+) |

**مثال عملي - مسح ملفات المشروع واستخراج الإحصائيات:**

```javascript
const fs = require('fs').promises;
const path = require('path');

async function analyzeProjectStructure(projectDir) {
  const stats = {
    totalFiles: 0,
    totalDirs: 0,
    filesByType: {},
    totalSize: 0,
  };

  async function traverse(dir) {
    try {
      const entries = await fs.readdir(dir, { withFileTypes: true });

      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);

        if (entry.isDirectory()) {
          // تخطي المجلدات المشهورة
          if (['node_modules', '.git', 'dist'].includes(entry.name)) {
            console.log(`⊘ تم تخطي: ${entry.name}`);
            continue;
          }
          stats.totalDirs++;
          await traverse(fullPath);  // الخوض العميق (deep recursion)
        } else {
          stats.totalFiles++;
          
          // استخراج نوع الملف
          const ext = path.extname(entry.name);
          stats.filesByType[ext] = (stats.filesByType[ext] || 0) + 1;

          // الحصول على حجم الملف
          const fileStat = await fs.stat(fullPath);
          stats.totalSize += fileStat.size;
        }
      }
    } catch (err) {
      console.error(`خطأ في قراءة ${dir}:`, err.message);
    }
  }

  await traverse(projectDir);
  return stats;
}

// الاستخدام
const analysis = await analyzeProjectStructure('.');
console.log('تحليل المشروع:', JSON.stringify(analysis, null, 2));
/*
{
  "totalFiles": 150,
  "totalDirs": 25,
  "filesByType": {
    ".js": 80,
    ".json": 20,
    ".md": 10
  },
  "totalSize": 5242880
}
*/
```

**الأخطاء الشابة:**
❌ عدم تخطي node_modules (سيبطئ العملية!)
❌ عدم معالجة الأخطاء في المجلدات المحمية
❌ فتح عدد لا نهائي من الملفات في نفس الوقت (استخدم streams للملفات الكبيرة)

---

#### `fsPromises.mkdir(path, [options])`

**التوقيع:**
```javascript
fsPromises.mkdir(path, [options])
```

**مثال عملي - إنشاء هيكل مشروع:**

```javascript
const fs = require('fs').promises;
const path = require('path');

async function initializeProjectStructure(projectName) {
  const projectPath = path.join('.', projectName);
  
  const directories = [
    '',
    'src',
    'src/config',
    'src/controllers',
    'src/models',
    'src/utils',
    'src/middleware',
    'tests',
    'tests/unit',
    'tests/integration',
    'public',
    'public/uploads',
    'logs',
    'docs',
  ];

  try {
    // إنشاء جميع المجلدات (recursive يعني إنشاء المسارات الأبوية أيضاً)
    for (const dir of directories) {
      const fullPath = path.join(projectPath, dir);
      await fs.mkdir(fullPath, { recursive: true, mode: 0o755 });
      console.log(`✓ تم إنشاء: ${fullPath}`);
    }

    console.log(`✓ تم إنشاء هيكل المشروع: ${projectPath}`);
  } catch (err) {
    console.error('فشل إنشاء هيكل المشروع:', err);
  }
}

// الاستخدام
await initializeProjectStructure('my-app');
```

---

### 3.3 حذف الملفات والمجلدات

#### `fsPromises.unlink(path)` - حذف ملف

```javascript
const fs = require('fs').promises;

async function deleteFile(filePath) {
  try {
    await fs.unlink(filePath);
    console.log(`✓ تم حذف الملف: ${filePath}`);
  } catch (err) {
    if (err.code === 'ENOENT') {
      console.warn('الملف غير موجود');
    } else {
      throw err;
    }
  }
}
```

#### `fsPromises.rm(path, [options])` - حذف ملف أو مجلد (أحدث من rmdir)

**التوقيع:**
```javascript
fsPromises.rm(path, [options])
```

**المعاملات:**
| المعامل | الوصف |
|--------|--------|
| `options.recursive` | حذف المجلدات وفحواها |
| `options.force` | تجاهل الأخطاء إذا الملف غير موجود |
| `options.maxRetries` | عدد محاولات إعادة المحاولة |
| `options.retryDelay` | التأخير بين محاولات إعادة المحاولة (بالملي ثانية) |

**مثال عملي - تنظيف الملفات المؤقتة:**

```javascript
const fs = require('fs').promises;
const path = require('path');

async function cleanTempDirectory(tempDir = './temp') {
  try {
    const stats = await fs.stat(tempDir);
    
    if (stats.isDirectory()) {
      // حذف المجلد وجميع محتوياته
      await fs.rm(tempDir, { 
        recursive: true,
        force: true,    // لا تخطأ إذا لم يكن موجود
        maxRetries: 3,
        retryDelay: 100
      });
      console.log(`✓ تم حذف المجلد المؤقت: ${tempDir}`);
    }
  } catch (err) {
    console.error('خطأ في حذف المجلد:', err);
  }
}

// الاستخدام - قم بتنظيف الملفات المؤقتة كل ساعة
setInterval(() => cleanTempDirectory(), 60 * 60 * 1000);
```

---

### 3.4 نسخ الملفات

#### `fsPromises.copyFile(src, dest, [mode])` - نسخ ملف واحد

```javascript
const fs = require('fs').promises;
const { COPYFILE_EXCL } = fs.constants;

async function backupFile(originalPath, backupPath) {
  try {
    // نسخ مع رفع خطأ إذا كان الهدف موجود
    await fs.copyFile(originalPath, backupPath, COPYFILE_EXCL);
    console.log(`✓ تم إنشاء نسخة احتياطية: ${backupPath}`);
  } catch (err) {
    if (err.code === 'EEXIST') {
      console.warn('النسخة الاحتياطية موجودة بالفعل');
    } else {
      throw err;
    }
  }
}
```

#### `fsPromises.cp(src, dest, [options])` - نسخ متقدم (Node.js 16.7+)

**التوقيع:**
```javascript
fsPromises.cp(src, dest, [options])
```

**المعاملات:**
| المعامل | الوصف |
|--------|--------|
| `options.recursive` | نسخ المجلدات بمحتوياتها |
| `options.force` | استبدال الملفات الموجودة |
| `options.errorOnExist` | رفع خطأ إذا كان المصدر موجود |
| `options.preserveTimestamps` | الحفاظ على تواريخ التعديل |
| `options.filter` | دالة تصفية (أي الملفات يتم نسخها) |

**مثال عملي - نسخ مشروع بتصفية:**

```javascript
const fs = require('fs').promises;
const path = require('path');

async function copyProjectWithFiltering(sourceDir, destDir) {
  const filter = (src, dest) => {
    const basename = path.basename(src);
    
    // تخطي هذه الملفات
    const skipPatterns = ['node_modules', '.git', 'dist', 'build', '.env'];
    for (const pattern of skipPatterns) {
      if (src.includes(pattern)) {
        return false;
      }
    }
    return true;
  };

  try {
    await fs.cp(sourceDir, destDir, {
      recursive: true,
      force: false,  // لا تستبدل الملفات الموجودة
      filter,
      preserveTimestamps: true
    });
    console.log(`✓ تم نسخ المشروع من ${sourceDir} إلى ${destDir}`);
  } catch (err) {
    console.error('فشل النسخ:', err);
  }
}

// الاستخدام
await copyProjectWithFiltering('./my-project', './my-project-backup');
```

---

### 3.5 معلومات الملفات

#### `fsPromises.stat(path, [options])` و `lstat(path, [options])`

**الفرق:**
- `stat()`: تتبع الرموز المرجعية (symlinks)
- `lstat()`: معلومات الرابط نفسه بدون تتبع

**مثال عملي - تحليل حجم المشروع:**

```javascript
const fs = require('fs').promises;
const path = require('path');

class ProjectAnalyzer {
  async getDirectorySize(dirPath) {
    let totalSize = 0;
    const fileStats = {};

    async function traverse(dir) {
      const entries = await fs.readdir(dir, { withFileTypes: true });

      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);

        if (entry.isFile()) {
          const stats = await fs.stat(fullPath);
          const ext = path.extname(entry.name);
          
          totalSize += stats.size;
          
          if (!fileStats[ext]) {
            fileStats[ext] = { count: 0, size: 0 };
          }
          fileStats[ext].count++;
          fileStats[ext].size += stats.size;
        } else if (entry.isDirectory()) {
          if (!['node_modules', '.git'].includes(entry.name)) {
            await traverse(fullPath);
          }
        }
      }
    }

    await traverse(dirPath);

    return {
      totalSize,
      totalSizeInMB: (totalSize / 1024 / 1024).toFixed(2),
      byType: fileStats
    };
  }
}

// الاستخدام
const analyzer = new ProjectAnalyzer();
const stats = await analyzer.getDirectorySize('.');
console.log('📊 إحصائيات المشروع:', stats);
```

**الخصائص المهمة للـ Stats:**
| الخاصية | النوع | الوصف |
|--------|-------|--------|
| `size` | number | حجم الملف بالبايت |
| `mtime` | Date | آخر وقت تعديل |
| `atime` | Date | آخر وقت وصول |
| `birthtime` | Date | وقت الإنشاء |
| `mode` | number | الأذونات والنوع |

---

## 4. Streams - للملفات الكبيرة

### 4.1 مقارنة الطرق الثلاث

```javascript
const fs = require('fs');

// ❌ readFile - يحمل الملف كاملاً في الذاكرة
// للملفات > 100MB قد يسبب مشاكل في الذاكرة
fs.readFile('huge-file.bin', (err, data) => {
  // data يحتوي على كل الملف في الذاكرة
});

// ✓ createReadStream - يقرأ على دفعات
// مثالي للملفات الكبيرة جداً
const readStream = fs.createReadStream('huge-file.bin', {
  highWaterMark: 64 * 1024  // 64KB في كل مرة
});

readStream.on('data', chunk => {
  console.log(`استقبال جزء: ${chunk.length} بايت`);
});

readStream.on('end', () => {
  console.log('انتهت القراءة');
});

// ✓ استخدام Streams مع Pipes (الطريقة الأفضل)
fs.createReadStream('input.txt')
  .pipe(fs.createWriteStream('output.txt'))
  .on('finish', () => {
    console.log('تم نسخ الملف بنجاح');
  });
```

### 4.2 مثال عملي: معالج ملفات CSV ضخمة

```javascript
const fs = require('fs');
const { Transform } = require('stream');

class CSVProcessor extends Transform {
  constructor(options = {}) {
    super({ ...options, objectMode: true });
    this.lineNumber = 0;
  }

  _transform(chunk, encoding, callback) {
    try {
      const lines = chunk.toString().split('\n');
      
      lines.forEach((line, index) => {
        if (line.trim()) {
          this.lineNumber++;
          const fields = line.split(',');
          
          // معالجة السجل
          const record = {
            lineNumber: this.lineNumber,
            fields,
            processedAt: new Date()
          };
          
          this.push(record);
        }
      });
      
      callback();
    } catch (err) {
      callback(err);
    }
  }
}

// الاستخدام
const csvProcessor = new CSVProcessor();

fs.createReadStream('data.csv', { encoding: 'utf8', highWaterMark: 16 * 1024 })
  .pipe(csvProcessor)
  .on('data', record => {
    console.log(`معالجة السجل ${record.lineNumber}:`, record.fields[0]);
  })
  .on('error', err => {
    console.error('خطأ في المعالجة:', err);
  })
  .on('end', () => {
    console.log('انتهت معالجة الملف');
  });
```

---

## 5. الأذونات والعلاقات الخاصة

### 5.1 تغيير الأذونات

#### `fsPromises.chmod(path, mode)`

```javascript
const fs = require('fs').promises;

async function setExecutablePermissions(scriptPath) {
  try {
    // 0o755 = rwxr-xr-x (owner: read+write+execute, others: read+execute)
    await fs.chmod(scriptPath, 0o755);
    console.log(`✓ تم جعل الملف قابلاً للتنفيذ: ${scriptPath}`);
  } catch (err) {
    console.error('فشل تغيير الأذونات:', err);
  }
}

// الاستخدام
await setExecutablePermissions('./deploy.sh');
```

**أكواد الأذونات الشائعة:**
| الكود | الوصف | الاستخدام |
|------|--------|----------|
| `0o644` | rw-r--r-- | ملفات عادية |
| `0o755` | rwxr-xr-x | ملفات قابلة للتنفيذ |
| `0o600` | rw------- | ملفات حساسة (مفاتيح SSH) |
| `0o775` | rwxrwxr-x | ملفات المشروع |

---

### 5.2 التحقق من إمكانية الوصول

#### `fsPromises.access(path, [mode])`

```javascript
const fs = require('fs').promises;
const { R_OK, W_OK, X_OK } = fs.constants;

async function checkFileAccess(filePath) {
  try {
    // التحقق من القراءة
    await fs.access(filePath, R_OK);
    console.log('✓ يمكن قراءة الملف');
  } catch (err) {
    console.error('لا يمكن قراءة الملف');
    return;
  }

  try {
    // التحقق من الكتابة
    await fs.access(filePath, W_OK);
    console.log('✓ يمكن كتابة الملف');
  } catch (err) {
    console.error('لا يمكن كتابة الملف');
  }
}

// الاستخدام
await checkFileAccess('./config.json');
```

---

## 6. العلاقات الرمزية (Symlinks)

#### `fsPromises.symlink(target, path, [type])`

```javascript
const fs = require('fs').promises;
const path = require('path');

async function createSymlink(originalPath, linkPath) {
  try {
    // إنشاء رابط رمزي
    await fs.symlink(originalPath, linkPath);
    console.log(`✓ تم إنشاء رابط: ${linkPath} -> ${originalPath}`);

    // التحقق من أنه رابط رمزي
    const stats = await fs.lstat(linkPath);
    if (stats.isSymbolicLink()) {
      const target = await fs.readlink(linkPath);
      console.log(`الهدف الفعلي: ${target}`);
    }
  } catch (err) {
    console.error('فشل إنشاء الرابط:', err);
  }
}

// الاستخدام
await createSymlink('/usr/local/node', './node-link');
```

---

## 7. المراقبة والمزامنة

### 7.1 مراقبة تغييرات الملفات

#### `fs.watch(filename, [options], listener)`

**مثال عملي - مراقب ملفات الإعدادات:**

```javascript
const fs = require('fs');
const path = require('path');

class ConfigWatcher {
  constructor(configPath) {
    this.configPath = configPath;
    this.currentConfig = null;
  }

  watch() {
    fs.watch(this.configPath, { persistent: true, recursive: false }, (eventType, filename) => {
      console.log(`📝 حدث تغيير: ${eventType} - ${filename}`);
      
      // تحديث الإعدادات
      this.reloadConfig();
    });
  }

  reloadConfig() {
    try {
      const data = fs.readFileSync(this.configPath, 'utf8');
      this.currentConfig = JSON.parse(data);
      console.log('✓ تم تحديث الإعدادات');
      
      // تشغيل callback للتحديث
      if (this.onConfigChange) {
        this.onConfigChange(this.currentConfig);
      }
    } catch (err) {
      console.error('خطأ في قراءة الإعدادات:', err);
    }
  }
}

// الاستخدام
const watcher = new ConfigWatcher('./app-config.json');
watcher.onConfigChange = (newConfig) => {
  console.log('الإعدادات الجديدة تم تطبيقها:', newConfig);
};
watcher.watch();
```

**مشاكل معروفة مع fs.watch:**
- قد يشتعل الحدث عدة مرات للتغيير الواحد
- قد لا يعمل على جميع أنظمة الملفات (NFS، FAT32)
- استخدم معالج deduplication أو `fs.watchFile()` كبديل

---

## 8. مقارنة الأداء والاختيار الصحيح

### 8.1 جدول المقارنة الشامل

```
┌─────────────────────────────────────────────────────────────────┐
│              مقارنة طرق العمل مع fs Module                     │
├──────────────┬─────────────┬─────────────┬─────────────────────┤
│   الطريقة    │  الأداء     │  الكود      │  الحالات الموصى بها  │
├──────────────┼─────────────┼─────────────┼─────────────────────┤
│ Callback     │ الأفضل      │ معقد        │ أنظمة عالية الأداء   │
│ Promise      │ -5-10%      │ نظيف        │ معظم التطبيقات       │
│ Sync         │ يحجب Event  │ بسيط جداً   │ بدء البرنامج فقط     │
│ Streams      │ ممتاز       │ متقدم       │ ملفات ضخمة > 100MB   │
└──────────────┴─────────────┴─────────────┴─────────────────────┘
```

### 8.2 قائمة قرار الاختيار

```
هل الملف صغير (< 10MB) وبسيط؟
├─ نعم → استخدم fs.readFile() / writeFile()
└─ لا → استمر

هل تحتاج أداء عالية جداً (> 10,000 ops/sec)?
├─ نعم → استخدم Callback API
└─ لا → استمر

هل تكتب كود حديث (Node.js 14+)?
├─ نعم → استخدم fs/promises مع async/await
└─ لا → استمر

هل الملف كبير جداً (> 100MB) أو يأتي من الشبكة?
├─ نعم → استخدم createReadStream() / createWriteStream()
└─ لا → استمر

هل تحتاج للمزامنة التام (synchronous)?
├─ نعم → استخدم readFileSync() (فقط في البداية!)
└─ لا → استمر

→ استخدم fs/promises (هي الخيار الأفضل عموماً!)
```

---

## 9. أمثلة واقعية متقدمة

### 9.1 نظام إدارة الملفات الشامل

```javascript
const fs = require('fs').promises;
const path = require('path');
const crypto = require('crypto');

class FileManager {
  constructor(baseDir) {
    this.baseDir = baseDir;
  }

  // حساب checksum الملف
  async getFileHash(filePath) {
    const content = await fs.readFile(filePath);
    return crypto.createHash('sha256').update(content).digest('hex');
  }

  // البحث عن الملفات بالنمط
  async findFiles(pattern, searchDir = this.baseDir) {
    const results = [];
    const regex = new RegExp(pattern);

    async function traverse(dir) {
      const entries = await fs.readdir(dir, { withFileTypes: true });
      
      for (const entry of entries) {
        const fullPath = path.join(dir, entry.name);
        
        if (entry.isDirectory()) {
          await traverse(fullPath);
        } else if (regex.test(entry.name)) {
          const stats = await fs.stat(fullPath);
          results.push({
            path: fullPath,
            name: entry.name,
            size: stats.size,
            modified: stats.mtime
          });
        }
      }
    }

    await traverse(searchDir);
    return results;
  }

  // إنشاء نسخة احتياطية ذكية
  async smartBackup(sourceDir, backupDir) {
    const backupLog = [];

    async function syncDirectory(src, dest) {
      await fs.mkdir(dest, { recursive: true });

      const entries = await fs.readdir(src, { withFileTypes: true });

      for (const entry of entries) {
        const srcPath = path.join(src, entry.name);
        const destPath = path.join(dest, entry.name);

        if (entry.isDirectory()) {
          await syncDirectory(srcPath, destPath);
        } else {
          try {
            const srcHash = await this.getFileHash(srcPath);
            
            let needsBackup = true;
            try {
              const destHash = await this.getFileHash(destPath);
              needsBackup = srcHash !== destHash;
            } catch (e) {
              // الملف لم يكن موجود في النسخة الاحتياطية
            }

            if (needsBackup) {
              await fs.copyFile(srcPath, destPath);
              backupLog.push(`✓ تم نسخ: ${srcPath}`);
            } else {
              backupLog.push(`- تم تجاهل (متطابق): ${srcPath}`);
            }
          } catch (err) {
            backupLog.push(`✗ فشل: ${srcPath} - ${err.message}`);
          }
        }
      }
    }

    await syncDirectory(sourceDir, backupDir);
    return backupLog;
  }
}

// الاستخدام
const manager = new FileManager('./project');
const backup = await manager.smartBackup('./project', './backup');
console.log('سجل النسخة الاحتياطية:', backup);
```

---

## 10. أفضل الممارسات والأخطاء الشائعة

### ✓ أفضل الممارسات

```javascript
// ✓ استخدم fs/promises مع async/await
const fs = require('fs').promises;
async function readConfig() {
  try {
    const data = await fs.readFile('config.json', 'utf8');
    return JSON.parse(data);
  } catch (err) {
    console.error('خطأ:', err);
    throw err;
  }
}

// ✓ استخدم try/finally للتنظيف الآمن
let handle;
try {
  handle = await fs.open('file.txt', 'r');
  const data = await handle.readFile('utf8');
  console.log(data);
} finally {
  if (handle) await handle.close();
}

// ✓ استخدم streams للملفات الكبيرة
fs.createReadStream('large.txt')
  .pipe(fs.createWriteStream('copy.txt'))
  .on('finish', () => console.log('Done!'));

// ✓ تحقق من نوع الخطأ (الأكواد)
try {
  await fs.readFile('file.txt');
} catch (err) {
  if (err.code === 'ENOENT') {
    console.log('الملف غير موجود');
  } else if (err.code === 'EACCES') {
    console.log('لا توجد أذونات');
  } else {
    throw err;
  }
}
```

### ❌ الأخطاء الشائعة

```javascript
// ❌ عدم الانتظار للعملية المتزامنة
fs.readFile('file.txt', (err, data) => {
  // قد تحدث race condition
  console.log(data);
});
// سيطبع undefined لأن العملية لم تنته بعد

// ❌ تحميل ملف ضخم بـ readFile
fs.readFile('1GB-file.bin', (err, data) => {
  // سيستهلك 1GB من الذاكرة!
  // استخدم streams بدلاً من ذلك
});

// ❌ عدم إغلاق FileHandle
const handle = await fs.open('file.txt');
const data = await handle.readFile();
// نسيان: await handle.close(); - قد يسبب memory leak

// ❌ استخدام Sync في معالج الطلب
app.get('/read', (req, res) => {
  const data = fs.readFileSync('file.txt'); // سيحجب جميع الطلبات!
  res.send(data);
});

// ✓ الطريقة الصحيحة:
app.get('/read', async (req, res) => {
  const data = await fs.promises.readFile('file.txt', 'utf8');
  res.send(data);
});
```

---

## الخلاصة والإرشادات النهائية

| الحالة | الاختيار الأفضل | السبب |
|--------|-----------------|------|
| Web Server | `fs/promises` + async/await | أداء ممتازة + كود نظيف |
| البرنامج يحتاج أداء عالية | Callback API | أقل overhead |
| Startup script | `fs.readFileSync()` | آمن وبسيط في البدء |
| ملفات > 100MB | `createReadStream()` | تحكم في الذاكرة |
| تطبيق حديث (Node 14+) | `fs/promises` | هي الأفضل عموماً |

**خط النهاية:** استخدم `fs/promises` مع `async/await` في 95% من الحالات. وفقط استخدم الطرق الأخرى عند الحاجة لحالات خاصة (أداء عالية جداً، أو ملفات ضخمة جداً).
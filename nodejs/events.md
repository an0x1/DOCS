## نظرة عامة على الموديول (نظرة عامة)

**موديول `node:events`** هو أحد أساسيات عمارة Node.js، حيث يوفر فئة `EventEmitter` التي تمكّن من بناء تطبيقات حقيقية تعتمد على الأحداث. معظم واجهات برمجة التطبيقات (APIs) الأساسية في Node.js مثل `net.Server`، `fs.ReadStream`، و`http.Server` ترث من هذه الفئة. النمط الحدث مركزي في Node.js لأنه يسمح بالفصل المنطقي بين مُنتج الحدث والمستهلكين، مما يجعل التطبيقات أكثر قابلية للتوسع والصيانة.[^1_1][^1_2]

### متى تستخدم Events Module؟

يجب استخدام `EventEmitter` عندما تحتاج إلى:

- **الاتصالات المتعددة**: تعريف عدة مستمعين لنفس الحدث أو أحداث مختلفة
- **الحياة الدورية المعقدة**: إصدار أحداث في نقاط متعددة من دورة حياة الكائن
- **الفصل بين المكونات**: فصل منطقي بين مُنتج ومستهلك الأحداث
- **معالجة الأخطاء المتخصصة**: التعامل مع أحداث خطأ بطريقة منظمة

بدلاً من:

- **Callbacks** (رد النداء): للاستجابات البسيطة لمرة واحدة
- **Promises/Async-Await**: للعمليات التي تحدث مرة واحدة فقط
- **Streams**: بدلاً من ذلك، الـ Streams نفسها ترث من `EventEmitter`

[^1_3][^1_4][^1_1]

---

## تفاصيل الدوال والخصائص (API Deep Dive)

### فئة EventEmitter

#### الدالة البناء (Constructor)

```javascript
new EventEmitter([options]);
```

**الخيارات**:

| الخيار              | النوع   | قيمة افتراضية | الوصف                                 |
| :------------------ | :------ | :------------ | :------------------------------------ |
| `captureRejections` | boolean | `false`       | تفعيل التقاط رفض الـ Promise تلقائياً |

**مثال عملي - معالج أحداث معقد للملف**:

```javascript
const { EventEmitter } = require("node:events");
const fs = require("node:fs");

class FileProcessor extends EventEmitter {
  constructor(filePath) {
    super({ captureRejections: true });
    this.filePath = filePath;
    this.lines = 0;

    // معالج رفض الـ Promise
    this[Symbol.for("nodejs.rejection")] = (err, event) => {
      console.error(`خطأ في معالجة الملف (${event}):`, err.message);
      this.emit("processingFailed", err);
    };
  }

  async processLargeFile() {
    try {
      const stream = fs.createReadStream(this.filePath, { encoding: "utf-8" });

      stream.on("data", async (chunk) => {
        this.lines += chunk.split("\n").length;
        this.emit("lineProcessed", { totalLines: this.lines });
      });

      stream.on("end", () => {
        this.emit("processingComplete", { totalLines: this.lines });
      });
    } catch (err) {
      this.emit("error", err);
    }
  }
}

// الاستخدام
const processor = new FileProcessor("./large-data.txt");
processor.on("lineProcessed", (data) => {
  console.log(`تم معالجة السطور: ${data.totalLines}`);
});
processor.processLargeFile();
```

**الأخطاء الشائعة**:

- عدم التقاط أحداث الخطأ، مما يؤدي إلى توقف العملية
- نسيان تنظيف المستمعين في الفئات المشتقة

---

### الدوال الأساسية للتسجيل

#### `emitter.on(eventName, listener)` / `emitter.addListener(eventName, listener)`

**التوقيع**:

```javascript
emitter.on(eventName, listener) → EventEmitter
```

**المعاملات**:

| المعامل     | النوع            | مطلوب؟ | الوصف                              |
| :---------- | :--------------- | :----- | :--------------------------------- |
| `eventName` | string \| symbol | نعم    | اسم الحدث (عادة يكون camelCase)    |
| `listener`  | Function         | نعم    | دالة رد النداء التي سيتم استدعاؤها |

**الإرجاع**: نفس كائن `EventEmitter` لتسلسل العمليات

**الوصف**: تضيف دالة مستمع إلى نهاية قائمة المستمعين. يمكن تسجيل نفس الدالة عدة مرات، وستُستدعى كل مرة.[^1_5][^1_1]

**مثال عملي - نظام معالجة الطلبات**:

```javascript
const { EventEmitter } = require("node:events");

class OrderProcessor extends EventEmitter {}

const processor = new OrderProcessor();

// تسجيل عدة مستمعين لنفس الحدث
processor.on("orderCreated", (order) => {
  console.log(`📦 تم إنشاء الطلب: ${order.id}`);
});

processor.on("orderCreated", (order) => {
  console.log(`💳 معالجة الدفع لـ: $${order.amount}`);
});

processor.on("orderCreated", (order) => {
  console.log(`📧 إرسال بريد تأكيد إلى: ${order.email}`);
});

// التسلسل
processor
  .on("orderShipped", (order) => {
    console.log(`✈️ تم شحن الطلب ${order.id}`);
  })
  .on("orderDelivered", (order) => {
    console.log(`🎉 تم استلام الطلب ${order.id}`);
  });

// الإصدار
processor.emit("orderCreated", {
  id: "ORD-123",
  amount: 99.99,
  email: "customer@example.com",
});

/* الإخراج:
📦 تم إنشاء الطلب: ORD-123
💳 معالجة الدفع لـ: $99.99
📧 إرسال بريد تأكيد إلى: customer@example.com
*/
```

**الأخطاء الشابة**:

- استخدام دوال السهم (Arrow Functions) إذا كنت تحتاج إلى قيمة `this`
- تسجيل نفس الدالة عدة مرات عن غير قصد

---

#### `emitter.once(eventName, listener)`

**التوقيع**:

```javascript
emitter.once(eventName, listener) → EventEmitter
```

**الوصف**: تضيف مستمع يُستدعى مرة واحدة فقط. بعد استدعاؤه، يتم حذف المستمع تلقائياً.[^1_1][^1_5]

**مثال عملي - المصادقة والتهيئة**:

```javascript
class DatabaseConnection extends EventEmitter {}

const db = new DatabaseConnection();

// المصادقة - تحدث مرة واحدة فقط
db.once("authenticated", (user) => {
  console.log(`✅ تم المصادقة بنجاح: ${user.name}`);
  console.log("جاهز لتنفيذ الاستعلامات");
});

// محاولة الاتصال 3 مرات
db.emit("authenticated", { name: "Ahmed" }); // سيتم استدعاء المستمع
db.emit("authenticated", { name: "Fatima" }); // سيتم تجاهله
db.emit("authenticated", { name: "Maryam" }); // سيتم تجاهله

// الإخراج:
// ✅ تم المصادقة بنجاح: Ahmed
// جاهز لتنفيذ الاستعلامات
```

**الأخطاء الشائعة**:

- الاعتقاد بأن المستمع سيبقى مسجلاً بعد الاستدعاء الأول

---

#### `emitter.prependListener(eventName, listener)` و `emitter.prependOnceListener(eventName, listener)`

**التوقيع**:

```javascript
emitter.prependListener(eventName, listener) → EventEmitter
emitter.prependOnceListener(eventName, listener) → EventEmitter
```

**الوصف**: مثل `on()` و `once()` لكن يضيف المستمع إلى **بداية** قائمة المستمعين بدلاً من النهاية. هذا يضمن تنفيذ هذا المستمع أولاً.[^1_1]

**مثال عملي - نظام تسجيل الأحداث مع الأولويات**:

```javascript
const { EventEmitter } = require("node:events");

class AlertSystem extends EventEmitter {}

const alerts = new AlertSystem();

// مستمع عادي - سيتم تنفيذه ثانياً
alerts.on("criticalError", (error) => {
  console.log(`⚠️  تسجيل الخطأ: ${error.message}`);
});

// مستمع بأولوية عالية - سيتم تنفيذه أولاً
alerts.prependListener("criticalError", (error) => {
  console.log(`🚨 تنبيه فوري: إيقاف النظام!`);
  // أوقف الخدمة على الفور
});

// آخر مستمع لم يتم تسجيله بعد
alerts.on("criticalError", (error) => {
  console.log(`📞 الاتصال بفريق الدعم`);
});

alerts.emit("criticalError", new Error("فشل قاعدة البيانات"));

/* الإخراج:
🚨 تنبيه فوري: إيقاف النظام!
⚠️  تسجيل الخطأ: فشل قاعدة البيانات
📞 الاتصال بفريق الدعم
*/
```

**الأخطاء الشائعة**:

- الخلط بين ترتيب التنفيذ عند استخدام `prepend` و`append` معاً

---

### دالة الإصدار

#### `emitter.emit(eventName[, ...args])`

**التوقيع**:

```javascript
emitter.emit(eventName[, ...args]) → boolean
```

**المعاملات**:

| المعامل     | النوع            | مطلوب؟  | الوصف                       |
| :---------- | :--------------- | :------ | :-------------------------- |
| `eventName` | string \| symbol | نعم     | اسم الحدث                   |
| `...args`   | any              | اختياري | معاملات تُمرر إلى المستمعين |

**الإرجاع**: `true` إذا كان هناك مستمعون، `false` وإلا

**الوصف**: يُصدر حدثاً بشكل **متزامن** (synchronously)، مما يعني أن جميع المستمعين يتم استدعاؤهم فوراً قبل أن تعود الدالة.[^1_2][^1_1]

**مثال عملي - معالج الفعاليات الحية (Live Events)**:

```javascript
class LiveEventSystem extends EventEmitter {}

const events = new LiveEventSystem();

// معالج العرض
events.on("concertStarted", (artist, song) => {
  console.log(`🎤 بدء حفل ${artist}`);
  console.log(`🎵 الأغنية الأولى: "${song}"`);
});

// معالج الإحصائيات
events.on("concertStarted", (artist, song) => {
  console.log(`📊 تحديث الإحصائيات - الفنان: ${artist}`);
});

// تطبيق على التوازي
events.on("concertStarted", (artist, song) => {
  console.log(`💰 إضافة إيرادات للفنان: ${artist}`);
});

// إصدار الحدث
const hasListeners = events.emit("concertStarted", "أم كلثوم", "ليلة الحنين");
console.log(`هناك مستمعون: ${hasListeners}`);

/* الإخراج:
🎤 بدء حفل أم كلثوم
🎵 الأغنية الأولى: "ليلة الحنين"
📊 تحديث الإحصائيات - الفنان: أم كلثوم
💰 إضافة إيرادات للفنان: أم كلثوم
هناك مستمعون: true
*/
```

**الأخطاء الشابة**:

- نسيان أن الإصدار متزامن - إذا ألقى مستمع استثناء، فسيتم إيقاف باقي المستمعين
- الاعتقاد بأن المعاملات المُمررة يمكن تعديلها في المستمعين (يتم نسخ المراجع فقط، وليس النسخ العميقة)

---

### دوال الاستعلام

#### `emitter.listenerCount(eventName[, listener])`

**التوقيع**:

```javascript
emitter.listenerCount(eventName[, listener]) → integer
```

**الوصف**: تعيد عدد المستمعين المسجلين لحدث معين. إذا تم توفير `listener` محدد، فتُرجع عدد مرات تسجيل هذا المستمع.[^1_2][^1_1]

**مثال عملي - مراقبة الأداء**:

```javascript
class PerformanceMonitor extends EventEmitter {}

const monitor = new PerformanceMonitor();

const logger = () => console.log("تسجيل");
const alerter = () => console.log("تنبيه");

monitor.on("cpuHigh", logger);
monitor.on("cpuHigh", alerter);
monitor.on("cpuHigh", logger); // تسجيل نفس الدالة مرتين

console.log(monitor.listenerCount("cpuHigh")); // 3
console.log(monitor.listenerCount("cpuHigh", logger)); // 2
console.log(monitor.listenerCount("cpuHigh", alerter)); // 1
```

---

#### `emitter.listeners(eventName)`

**التوقيع**:

```javascript
emitter.listeners(eventName) → Function[]
```

**الوصف**: ترجع **نسخة** من مصفوفة المستمعين. أي تعديل على المصفوفة المرجعة لن يؤثر على المستمعين الفعليين.[^1_2][^1_1]

**مثال عملي - فحص المستمعين**:

```javascript
const { EventEmitter } = require('node:events');

const emitter = new EventEmitter();

const handler1 = () => console.log('Handler 1');
const handler2 = () => console.log('Handler 2');

emitter.on('data', handler1);
emitter.on('data', handler2);

const listeners = emitter.listeners('data');
console.log(listeners.length); // 2
console.log(listeners[^1_0] === handler1); // true

// التعديل على النسخة لا يؤثر على المستمعين الحقيقيين
listeners.pop();
console.log(emitter.listenerCount('data')); // لا يزال 2
```

---

#### `emitter.rawListeners(eventName)`

**التوقيع**:

```javascript
emitter.rawListeners(eventName) → Function[]
```

**الوصف**: مثل `listeners()` لكن **تشمل الـ Wrappers** (الدوال الملفوفة) مثل تلك التي تُنشأ بواسطة `once()`.[^1_1][^1_2]

**مثال عملي - فحص المستمعين المتقدم**:

```javascript
const { EventEmitter } = require('node:events');

const emitter = new EventEmitter();

const regularHandler = () => console.log('Regular');
const onceHandler = () => console.log('Once');

emitter.on('event', regularHandler);
emitter.once('event', onceHandler);

// listeners() - يعيد الدوال الأصلية
console.log(emitter.listeners('event').length); // 2

// rawListeners() - يعيد الـ wrappers
const raw = emitter.rawListeners('event');
console.log(raw.length); // 2
console.log(raw[^1_1].listener === onceHandler); // true (مغلفة)
```

---

#### `emitter.eventNames()`

**التوقيع**:

```javascript
emitter.eventNames() → (string | symbol)[]
```

**الوصف**: ترجع مصفوفة بأسماء جميع الأحداث التي يوجد لها مستمعون مسجلون.[^1_2][^1_1]

**مثال عملي - فحص حالة النظام**:

```javascript
class SystemMonitor extends EventEmitter {}

const monitor = new SystemMonitor();

const memSymbol = Symbol("memory");
const cpuSymbol = Symbol("cpu");

monitor.on("diskSpace", () => {});
monitor.on("networkLatency", () => {});
monitor.on(memSymbol, () => {});
monitor.on(cpuSymbol, () => {});

const events = monitor.eventNames();
console.log(events);
// ['diskSpace', 'networkLatency', Symbol(memory), Symbol(cpu)]

// فحص الأحداث
if (events.includes("error")) {
  console.log("❌ لا توجد معالجة للأخطاء!");
} else {
  console.log("⚠️  تحذير: لا توجد معالجة للأخطاء");
}
```

---

### دوال الإزالة

#### `emitter.removeListener(eventName, listener)` / `emitter.off(eventName, listener)`

**التوقيع**:

```javascript
emitter.removeListener(eventName, listener) → EventEmitter
emitter.off(eventName, listener) → EventEmitter
```

**الوصف**: تزيل مستمع محدد من قائمة المستمعين. تزيل **أحدث** نسخة مضافة من الدالة. يجب استدعاؤها عدة مرات لإزالة نفس الدالة المسجلة عدة مرات.[^1_1][^1_2]

**مثال عملي - إدارة دورة الحياة**:

```javascript
class WebSocketConnection extends EventEmitter {}

const ws = new WebSocketConnection();

const onMessage = (data) => {
  console.log("رسالة:", data);
};

const onError = (error) => {
  console.error("خطأ:", error.message);
};

// التسجيل
ws.on("message", onMessage);
ws.on("error", onError);

console.log(`المستمعون: ${ws.listenerCount("message")}`); // 1

// إزالة المستمع
ws.removeListener("message", onMessage);

console.log(`المستمعون بعد الإزالة: ${ws.listenerCount("message")}`); // 0

// محاولة الإرسال - لن يحدث شيء
ws.emit("message", "Hello");
```

**الأخطاء الشابة**:

- استخدام دالة سهم معرّفة مباشرة - لا يمكن إزالتها لاحقاً (لأنها ليست نفس المرجع)

```javascript
// ❌ خطأ
emitter.on("event", () => console.log("handler"));
emitter.removeListener("event", () => console.log("handler")); // لن تعمل!

// ✅ صحيح
const handler = () => console.log("handler");
emitter.on("event", handler);
emitter.removeListener("event", handler); // تعمل
```

---

#### `emitter.removeAllListeners([eventName])`

**التوقيع**:

```javascript
emitter.removeAllListeners([eventName]) → EventEmitter
```

**الوصف**: تزيل **جميع** المستمعين لحدث معين، أو **جميع** المستمعين لجميع الأحداث إذا لم يتم تحديد اسم الحدث.[^1_2][^1_1]

**مثال عملي - تنظيف الموارد**:

```javascript
class DataCache extends EventEmitter {
  shutdown() {
    console.log("🛑 إيقاف الذاكرة المؤقتة");
    // تنظيف جميع المستمعين
    this.removeAllListeners();
    console.log("✅ تم تنظيف جميع المستمعين");
  }
}

const cache = new DataCache();

cache.on("hit", () => console.log("Hit"));
cache.on("miss", () => console.log("Miss"));
cache.on("evict", () => console.log("Evict"));

console.log(`الأحداث: ${cache.eventNames().length}`); // 3

cache.removeAllListeners();

console.log(`الأحداث بعد التنظيف: ${cache.eventNames().length}`); // 0
```

---

### إدارة الذاكرة

#### `emitter.setMaxListeners(n)` و `emitter.getMaxListeners()`

**التوقيع**:

```javascript
emitter.setMaxListeners(n) → EventEmitter
emitter.getMaxListeners() → number
```

**الوصف**: تحدد/تحصل على الحد الأقصى للمستمعين. القيمة الافتراضية هي **10**. إذا تم تجاوزها، يطبع Node.js تحذير إلى `stderr`.[^1_1][^1_2]

**مثال عملي - نظام الإشعارات**:

```javascript
class NotificationHub extends EventEmitter {
  constructor() {
    super();
    // زيادة الحد الأقصى للمستمعين
    this.setMaxListeners(50);
  }

  addNotificationChannel(channelName, handler) {
    this.on("notification", handler);
    console.log(`تم إضافة قناة: ${channelName}`);
  }
}

const hub = new NotificationHub();

console.log(`الحد الأقصى: ${hub.getMaxListeners()}`); // 50

// إضافة 20 قناة إشعارات
for (let i = 1; i <= 20; i++) {
  hub.addNotificationChannel(`channel-${i}`, () => {
    console.log(`إشعار إلى القناة ${i}`);
  });
}

console.log(`المستمعون: ${hub.listenerCount("notification")}`); // 20
```

**الأخطاء الشابة**:

- تعيين الحد الأقصى إلى 0 أو `Infinity` بدون سبب وجيه (قد يؤدي لتسريب ذاكرة)

```javascript
// ❌ خطر
emitter.setMaxListeners(0); // سيعطل التحذيرات تماماً

// ✅ أفضل
emitter.setMaxListeners(Infinity); // إذا كنت متأكداً من عدد المستمعين
```

---

### أحداث داخلية خاصة

#### Event: `'newListener'`

**المعاملات**:

- `eventName` {string|symbol}
- `listener` {Function}

**الوصف**: يُصدر **قبل** إضافة مستمع جديد. يمكن استخدام هذا للتقاط المستمعين قبل تسجيلهم.[^1_2][^1_1]

**مثال عملي - تتبع المستمعين**:

```javascript
const { EventEmitter } = require("node:events");

class AuditedEmitter extends EventEmitter {
  constructor() {
    super();
    this.listenerLog = [];

    // تتبع جميع المستمعين الجدد
    this.on("newListener", (eventName, listener) => {
      this.listenerLog.push({
        timestamp: new Date(),
        event: eventName,
        listenerName: listener.name || "anonymous",
      });
      console.log(`📝 تم تسجيل مستمع جديد: ${eventName}`);
    });
  }

  getAuditLog() {
    return this.listenerLog;
  }
}

const audited = new AuditedEmitter();

audited.on("data", () => {});
audited.on("data", function namedHandler() {});
audited.on("error", () => {});

console.log(audited.getAuditLog());
```

---

#### Event: `'removeListener'`

**المعاملات**:

- `eventName` {string|symbol}
- `listener` {Function}

**الوصف**: يُصدر **بعد** إزالة مستمع.[^1_1][^1_2]

---

#### Event: `'error'`

**المعاملات**:

- `err` {Error}

**الوصف**: حدث خاص جداً. إذا لم يكن هناك مستمع مسجل للحدث `'error'` وتم إصدار حدث خطأ، فسيطرح Node.js استثناء ويوقف العملية.[^1_2][^1_1]

**مثال عملي - معالجة الأخطاء الضرورية**:

```javascript
class DataProcessor extends EventEmitter {
  process(data) {
    if (!data) {
      // خطر! سيوقف العملية إذا لم يكن هناك مستمع للخطأ
      this.emit("error", new Error("لا توجد بيانات"));
      return;
    }
    this.emit("processed", data);
  }
}

const processor = new DataProcessor();

// ❌ خطر - بدون معالج أخطاء
// processor.process(null); // سيوقف العملية

// ✅ آمن - مع معالج أخطاء
processor.on("error", (err) => {
  console.error("❌ خطأ:", err.message);
  // تنظيف أو إعادة محاولة
});

processor.on("processed", (data) => {
  console.log("✅ تمت المعالجة:", data);
});

processor.process(null);
processor.process({ value: 42 });
```

**أفضل الممارسات**:

- **دائماً** سجل مستمع للأخطاء
- استخدم `errorMonitor` لمراقبة الأخطاء بدون التأثير على السلوك

```javascript
const { ErrorMonitor } = require("node:events");

emitter.on(events.errorMonitor, (err) => {
  // تسجيل بدون استهلاك الخطأ
  logger.error(err);
  // العملية ستستمر في الإيقاف إذا لم يكن هناك معالج عادي
});
```

---

## المقارنات واتخاذ القرارات (مقارنات)

### EventEmitter vs Callbacks

| المعيار             | EventEmitter                  | Callback                |
| :------------------ | :---------------------------- | :---------------------- |
| **استخدام الحالات** | أحداث متعددة / متكررة         | استجابة واحدة فقط       |
| **عدد المستمعين**   | عدة مستمعين للحدث نفسه        | عادة مستمع واحد         |
| **الفصل المنطقي**   | عالي (فصل تام)                | منخفض (ارتباط مباشر)    |
| **سهولة الاستخدام** | متوسطة                        | بسيطة جداً              |
| **إدارة الذاكرة**   | تحتاج انتباه (removeListener) | تلقائية في معظم الحالات |

**مثال المقارنة**:

```javascript
// Callback - للعمليات البسيطة
function readConfig(callback) {
  setTimeout(() => {
    callback(null, { apiUrl: "https://api.example.com" });
  }, 100);
}

readConfig((err, config) => {
  if (err) console.error(err);
  console.log(config);
});

// EventEmitter - للعمليات المعقدة
const { EventEmitter } = require("node:events");

class ConfigLoader extends EventEmitter {
  load() {
    setTimeout(() => {
      this.emit("loaded", { apiUrl: "https://api.example.com" });
      this.emit("ready");
    }, 100);
  }
}

const loader = new ConfigLoader();
loader.on("loaded", (config) => {
  console.log("تم تحميل الإعدادات:", config);
});
loader.on("ready", () => {
  console.log("جاهز للعمل");
});
loader.load();
```

---

### EventEmitter vs Promises/Async-Await

| المعيار               | EventEmitter       | Promises/Async-Await |
| :-------------------- | :----------------- | :------------------- |
| **الاستخدام**         | أحداث متعددة       | عملية واحدة فقط      |
| **سلسلة الاستدعاءات** | معقدة جداً         | بسيطة جداً           |
| **معالجة الأخطاء**    | emit('error')      | try-catch            |
| **الألغاء**           | صعب                | AbortController      |
| **الحالة**            | يجب إدارتها يدوياً | في Promise state     |

**مثال المقارنة**:

```javascript
// Promise - للعملية الواحدة
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    return await response.json();
  } catch (err) {
    console.error("فشل جلب المستخدم:", err);
  }
}

// EventEmitter - للعمليات المتعددة
const { EventEmitter } = require("node:events");

class UserStream extends EventEmitter {
  async fetchMany(ids) {
    for (const id of ids) {
      try {
        const response = await fetch(`/api/users/${id}`);
        const user = await response.json();
        this.emit("userLoaded", user); // قد يكون لعدة مستمعين
      } catch (err) {
        this.emit("error", err);
      }
    }
    this.emit("complete");
  }
}

const stream = new UserStream();
stream.on("userLoaded", (user) => {
  console.log("مستخدم:", user.name);
});
stream.on("complete", () => {
  console.log("تم تحميل جميع المستخدمين");
});
stream.fetchMany([1, 2, 3]);
```

---

## دوال المساعدة الثابتة (Utility Methods)

### `events.once(emitter, eventName[, options])`

**التوقيع**:

```javascript
events.once(emitter, eventName[, options]) → Promise
```

**الوصف**: تُرجع Promise تُحل عند إصدار الحدث.[^1_1][^1_2]

**مثال عملي**:

```javascript
const { once, EventEmitter } = require("node:events");

const emitter = new EventEmitter();

async function waitForData() {
  console.log("انتظار البيانات...");
  const [data] = await once(emitter, "data");
  console.log("البيانات المستلمة:", data);
}

waitForData();

setTimeout(() => {
  emitter.emit("data", { message: "مرحباً" });
}, 1000);
```

---

### `events.on(emitter, eventName[, options])`

**التوقيع**:

```javascript
events.on(emitter, eventName[, options]) → AsyncIterator
```

**الوصف**: ترجع `AsyncIterator` يمكن استخدامها مع `for-await-of`.[^1_2][^1_1]

**مثال عملي - معالجة دفق من الأحداث**:

```javascript
const { on, EventEmitter } = require("node:events");

const emitter = new EventEmitter();

async function processEvents() {
  for await (const [data] of on(emitter, "message")) {
    console.log("رسالة:", data);
  }
}

// إطلاق المعالجة
processEvents().catch(console.error);

// محاكاة الرسائل
emitter.emit("message", "مرحباً");
emitter.emit("message", "كيف حالك");
emitter.emit("message", "وداعاً");
```

---

### `events.getEventListeners(emitterOrTarget, eventName)`

**التوقيع**:

```javascript
events.getEventListeners(emitterOrTarget, eventName) → Function[]
```

**الوصف**: ترجع المستمعين لـ `EventEmitter` أو `EventTarget` (مثل عناصر DOM).[^1_1][^1_2]

---

## المزايا والعيوب والحالات الاستخدامية (المزايا والعيوب والحالات)

### المزايا ✅

1. **الفصل المنطقي العالي**: المنتج والمستهلك منفصلان تماماً
2. **قابلية التوسع**: يمكن إضافة مستمعين جدد بسهولة بدون تعديل الكود الموجود
3. **الأداء**: معالجة متزامنة بدون أي overhead إضافي
4. **التوافقية**: معظم واجهات Node.js الأساسية ترث منها
5. **المرونة**: يمكن إضافة/إزالة مستمعين ديناميكياً
6. **سهولة الفحص**: يمكن معرفة عدد المستمعين والأحداث بسهولة

---

### العيوب ❌

1. **تسريب الذاكرة**: إذا لم يتم حذف المستمعين بشكل صحيح
2. **صعوبة التتبع**: يصعب متابعة حركة البيانات بين المستمعين
3. **الأخطاء الصامتة**: إذا لم يكن هناك مستمع، الحدث يُتجاهل بدون تحذير
4. **الأخطاء أثناء الإصدار**: إذا ألقى مستمع واحد استثناء، يتم إيقاف باقي المستمعين
5. **تعقيد التصحيح**: يصعب تتبع الأخطاء عندما تكون هناك أحداث متعددة

---

### أفضل 3 حالات استخدام

| الحالة                     | المثال                      | السبب                          |
| :------------------------- | :-------------------------- | :----------------------------- |
| **أنظمة الإشعارات**        | نظام إشعارات بريد/SMS/Push  | أحداث متعددة، مستمعون مختلفون  |
| **معالجات الملفات الضخمة** | قراءة ملفات CSV كبيرة الحجم | أحداث متكررة (كل سطر/كل chunk) |
| **الأنظمة الموزعة**        | Pub/Sub، الرسائل المرسلة    | فصل تام بين الخدمات            |

**مثال عملي - نظام إشعارات حقيقي**:

```javascript
const { EventEmitter } = require("node:events");

class NotificationService extends EventEmitter {
  async sendNotification(userId, message, channels = ["email", "sms", "push"]) {
    console.log(`📢 إرسال إشعار للمستخدم ${userId}`);

    // إصدار حدث واحد
    this.emit("notification", {
      userId,
      message,
      timestamp: new Date(),
      channels,
    });
  }
}

const service = new NotificationService();

// مستمع 1: إرسال بريد
service.on("notification", (notif) => {
  if (notif.channels.includes("email")) {
    console.log(`📧 إرسال بريد إلى: ${notif.userId}`);
    // sendEmail(notif.userId, notif.message);
  }
});

// مستمع 2: إرسال SMS
service.on("notification", (notif) => {
  if (notif.channels.includes("sms")) {
    console.log(`📱 إرسال SMS إلى: ${notif.userId}`);
    // sendSMS(notif.userId, notif.message);
  }
});

// مستمع 3: تسجيل في قاعدة البيانات
service.on("notification", (notif) => {
  console.log(`💾 حفظ في قاعدة البيانات: ${notif.userId}`);
  // saveToDatabase(notif);
});

// استخدام
service.sendNotification(123, "مرحباً بك!", ["email", "push"]);
```

---

![Node.js EventEmitter Lifecycle and Event Flow](https://ppl-ai-code-interpreter-files.s3.amazonaws.com/web/direct-files/f9111e19dceadf035674475ae9dc55d7/83107656-8bb2-4f12-a483-95ade4411c93/e7985e7a.png)

Node.js EventEmitter Lifecycle and Event Flow

---

## ملخص أفضل الممارسات

### لتجنب تسريب الذاكرة

```javascript
// ❌ خطر
class DataProcessor extends EventEmitter {}
const processor = new DataProcessor();

// في حلقة
for (let i = 0; i < 100; i++) {
  processor.on("data", () => {}); // سيؤدي لـ warning
}

// ✅ آمن
const processor = new DataProcessor();
processor.setMaxListeners(100); // أخبر Node.js بأنك تعرف ما تفعل

// أو إزالة المستمعين عند الانتهاء
const handler = () => {};
processor.on("data", handler);
// لاحقاً...
processor.removeListener("data", handler);
```

### للأخطاء الآمنة

```javascript
// ❌ خطر - قد يوقف العملية
const processor = new EventEmitter();
processor.emit("error", new Error("خطأ"));

// ✅ آمن
const processor = new EventEmitter();
processor.on("error", (err) => {
  console.error("حدث خطأ:", err);
});
processor.emit("error", new Error("خطأ"));
```

### لتجنب الازدواجية

```javascript
// ❌ خطأ - arrow function تُنشئ دالة جديدة كل مرة
componentDidMount() {
  this.emitter.on('data', () => this.setState({...}));
}

componentWillUnmount() {
  this.emitter.removeListener('data', () => this.setState({...})); // لن تعمل!
}

// ✅ صحيح
componentDidMount() {
  this.handler = () => this.setState({...});
  this.emitter.on('data', this.handler);
}

componentWillUnmount() {
  this.emitter.removeListener('data', this.handler);
}
```

---

## الخلاصة

**موديول `node:events`** هو من أساسيات برمجة Node.js. بينما يبدو بسيطاً، فإن إتقان إدارة المستمعين والأخطاء ضروري لبناء تطبيقات موثوقة. المفتاح هو:

- ✅ **دائماً** احتفظ بمراجع للمستمعين لإزالتهم لاحقاً
- ✅ **دائماً** سجل معالج أخطاء
- ✅ استخدم `setMaxListeners` فقط بعد فهمك لعدد المستمعين
- ✅ فكر في `Promises` أو `Callbacks` للعمليات البسيطة
- ✅ استخدم `EventEmitter` للعمليات المعقدة والمتكررة

مع هذه الممارسات، ستكون قادراً على بناء أنظمة حدثية آمنة وفعالة.[^1_1][^1_2][^1_6][^1_7]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_60][^1_61][^1_62][^1_63][^1_64][^1_65][^1_66][^1_67][^1_8][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/nodejs/node/blob/main/doc/api/events.md
[^1_2]: https://nodejs.org/api/events.html
[^1_3]: https://www.yld.io/blog/using-an-event-emitter-common-use-and-edge-cases
[^1_4]: https://www.geeksforgeeks.org/javascript/callbacks-vs-promises-vs-async-await/
[^1_5]: https://www.w3schools.com/nodejs/nodejs_events.asp
[^1_6]: https://infinitejs.com/posts/prevent-memory-leaks-nodejs-event-emitters/
[^1_7]: https://stackoverflow.com/questions/9768444/possible-eventemitter-memory-leak-detected
[^1_8]: https://dl.acm.org/doi/10.1145/3297280.3297456
[^1_9]: https://www.semanticscholar.org/paper/f0d99e7a592b2108f8a13238ec4c43554374684f
[^1_10]: https://www.semanticscholar.org/paper/cd57de2e8838433c5b705d62d1c1194b83905ae1
[^1_11]: https://www.semanticscholar.org/paper/8a23be61506b8c64361cd90e6a024ff628671e79
[^1_12]: https://arxiv.org/pdf/2107.13708.pdf
[^1_13]: http://arxiv.org/pdf/2405.12117.pdf
[^1_14]: http://arxiv.org/pdf/2401.08595.pdf
[^1_15]: https://arxiv.org/pdf/1604.00691.pdf
[^1_16]: https://arxiv.org/pdf/1512.07067.pdf
[^1_17]: https://arxiv.org/pdf/1008.0823.pdf
[^1_18]: https://arxiv.org/pdf/2109.02382.pdf
[^1_19]: https://arxiv.org/pdf/2103.10881.pdf
[^1_20]: https://www.w3schools.com/nodejs/ref_eventemitter.asp
[^1_21]: https://angular.dev/events/v21
[^1_22]: https://www.geeksforgeeks.org/node-js/node-js-eventemitter/
[^1_23]: https://stackoverflow.com/questions/8898399/node-js-inheriting-from-eventemitter
[^1_24]: https://stackoverflow.com/questions/77906647/ensuring-node-js-v20-compatibility-in-a-net-workflow
[^1_25]: https://www.gyata.ai/javascript/events-and-eventemitter
[^1_26]: https://nodejs.org/en/about/previous-releases
[^1_27]: https://www.geeksforgeeks.org/node-js/what-is-eventemitter-in-node-js/
[^1_28]: https://www.csharp.com/UploadFile/f50501/eventemitter-object-in-nodejs/
[^1_29]: https://dev.to/sovannaro/understanding-the-events-module-in-nodejs-3i5o
[^1_30]: https://dev.to/imsushant12/mastering-event-driven-programming-with-the-eventemitter-in-nodejs-38kd
[^1_31]: https://www.elearningsolutions.co.in/nodejs-eventemitter-complete-guide/
[^1_32]: https://github.com/nodejs/node/blob/master/lib/events.js/
[^1_33]: https://softm.tistory.com/entry/Nodejs-events-Module-EventEmitter-Class
[^1_34]: https://nodejs.org/api/process.html
[^1_35]: https://nodejs.org/en/learn/asynchronous-work/the-nodejs-event-emitter
[^1_36]: https://www.synology.com/en-us/dsm/packages/Node.js_v20?os_ver=7.3
[^1_37]: https://linkinghub.elsevier.com/retrieve/pii/S0021929018307309
[^1_38]: https://pubs.acs.org/doi/10.1021/acs.est.3c00026
[^1_39]: https://iopscience.iop.org/article/10.1149/1945-7111/ad7f91
[^1_40]: https://www.ssrn.com/abstract=4703243
[^1_41]: https://link.springer.com/10.1007/s10618-022-00894-5
[^1_42]: https://dirjournal.org/articles/doi/dir.2022.211297
[^1_43]: https://pubs.acs.org/doi/10.1021/acs.macromol.4c02526
[^1_44]: https://arxiv.org/abs/2503.23424
[^1_45]: https://ijaidsml.org/index.php/ijaidsml/article/view/258/
[^1_46]: https://ijcaonline.org/archives/volume187/number22/mhaskey-2025-ijca-925366.pdf
[^1_47]: https://www.mdpi.com/2076-3417/10/20/7338/pdf
[^1_48]: https://arxiv.org/pdf/2408.00440.pdf
[^1_49]: https://aclanthology.org/2023.findings-acl.586.pdf
[^1_50]: https://arxiv.org/ftp/arxiv/papers/0806/0806.1100.pdf
[^1_51]: https://arxiv.org/pdf/1702.06764.pdf
[^1_52]: https://www.aclweb.org/anthology/2020.acl-main.522.pdf
[^1_53]: https://arxiv.org/pdf/1702.08051.pdf
[^1_54]: https://www.linkedin.com/posts/thealiraza_nodejs-javascript-backend-activity-7350452189221289984-Bqaa
[^1_55]: https://www.youtube.com/watch?v=dRToKpcS4Ho
[^1_56]: https://www.dhiwise.com/post/leveraging-react-event-emitter-for-component-communication
[^1_57]: https://uberbrady.com/2016/10/javascript-callbacks-eventemitters-and-promises-which-one-to-use/
[^1_58]: https://stackoverflow.com/questions/34874079/event-emitter-causing-memory-leak
[^1_59]: https://stackoverflow.com/questions/58965412/event-emitters-vs-promises-for-parallel-execution
[^1_60]: https://stackoverflow.com/questions/79152107/why-using-eventemitter-in-angular-is-considered-bad-practice
[^1_61]: https://mcculloughwebservices.com/2016/10/30/possible-eventemitter-memory-leak-detected/
[^1_62]: https://www.digitalocean.com/community/tutorials/understanding-the-event-loop-callbacks-promises-and-async-await-in-javascript
[^1_63]: https://www.reddit.com/r/node/comments/xd2wek/what_is_a_correct_use_case_for_eventemitter_class/
[^1_64]: https://www.freecoderteam.com/post/fixing-nodejs-eventemitter-memory-leaks
[^1_65]: https://www.reddit.com/r/learnjavascript/comments/1ew0a0g/facing_problem_to_understand_callbacks_promises/
[^1_66]: https://github.com/caomingkai/Front-End-Common-Problems/blob/master/Event-Emitter.md
[^1_67]: https://forums.meteor.com/t/debugging-eventemitter-memory-leaks-in-production/42039

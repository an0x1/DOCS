# توثيق Node.js crypto Module - نسخة متعمقة شاملة

في هذا الدليل، سأقدم لك مرجعاً كاملاً وشاملاً لوحدة `crypto` في Node.js. هذه الوحدة توفر وظائف تشفيرية قوية تغطي التجزئة (Hashing)، والتشفير/فك التشفير (Encryption/Decryption)، والمفاتيح غير المتماثلة (Asymmetric Keys)، والتوقيعات الرقمية، وتبادل المفاتيح الآمن. الدليل مخصص للمطورين المتقدمين الذين يريدون فهماً عميقاً لكل دالة وخاصية في الوحدة.

## 1. نظرة عامة على الوحدة

### ما هي وحدة crypto؟

وحدة `node:crypto` توفر تغليفات آمنة حول مكتبة OpenSSL لتنفيذ عمليات تشفيرية. تستخدم OpenSSL كمحرك تشفيري أساسي، مما يعني أن الأداء والأمان يعتمدان على نسخة OpenSSL المثبتة على النظام. تتضمن الوحدة وظائف للتجزئة (Hash)، و HMAC، والتشفير (Cipher)، وفك التشفير (Decipher)، والتوقيعات الرقمية (Sign/Verify)، وإنشاء المفاتيح.[^1_1][^1_2]

### متى تستخدم crypto مقابل المكتبات الخارجية؟

استخدم `crypto` المدمجة عندما:

- تحتاج لأداء عالي وتقليل التبعيات الخارجية
- تعمل مع تشفير متقياس أو HMAC بسيط
- تحتاج لتشفير تناظري (AES) أو غير تناظري (RSA، ECDH)
- تريد توقيعات رقمية من دون مكتبات إضافية
- تطبق معايير أمنية تتطلب استخدام OpenSSL

استخدم مكتبات خارجية عندما:

- تحتاج إلى دعم خوارزميات بديلة (مثل bcrypt أو Argon2)
- تريد واجهة برمجية أكثر سهولة في الاستخدام
- تحتاج إلى سلوك موثق بشكل أفضل للحالات الحدية

chart:63

## 2. تفاصيل الدوال والخصائص الرئيسية

### الجزء الأول: التجزئة (Hashing)

#### **`crypto.createHash(algorithm[, options])`**

**الصيغة:**```
crypto.createHash(algorithm, options) → Hash

````

**الوصف:**
تنشئ كائن `Hash` يستخدم لحساب بصمة تجزئة للبيانات. لا يمكن عكس هذه العملية - يمكنك فقط التحقق من توافق البيانات الجديدة مع الهاش القديم.

**جدول المعاملات:**

| المعامل | النوع | مطلوب/اختياري | الوصف |
|--------|-------|----------|-------|
| `algorithm` | string | مطلوب | اسم خوارزمية التجزئة (مثل `'sha256'`, `'sha512'`, `'sha1'`) |
| `options` | Object | اختياري | خيارات إضافية (مثل `outputLength` للدوال XOF) |

**القيمة المرجعة:**
كائن `Hash` يمكن تسلسل عمليات `update()` و `digest()` عليه.

**مثال عملي - حساب بصمة SHA-256 لملف JSON:**

```javascript
const crypto = require('crypto');
const fs = require('fs');

// حالة واقعية: التحقق من سلامة بيانات مستخدم بعد تحميلها من قاعدة البيانات
function verifyDataIntegrity(userData, storedHash) {
  const hash = crypto.createHash('sha256');

  // تمرير بيانات المستخدم (JSON بعد تنظيفها)
  hash.update(JSON.stringify(userData), 'utf8');

  const computedHash = hash.digest('hex');

  // مقارنة آمنة ضد timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(computedHash),
    Buffer.from(storedHash)
  );
}

// الاستخدام
const user = { id: 123, email: 'user@example.com', role: 'admin' };
const expectedHash = 'a1b2c3...'; // مخزنة في قاعدة البيانات

if (verifyDataIntegrity(user, expectedHash)) {
  console.log('بيانات المستخدم آمنة لم تتعدل');
} else {
  console.log('تحذير: بيانات المستخدم قد تكون معدلة!');
}
````

**الأخطاء الشائعة:**

1. **عدم استخدام `timingSafeEqual` عند المقارنة**: قد يؤدي لهجمات التوقيت
2. **مقارنة الهاشات بطول مختلف**: تأكد من أن كلا الهاشين بنفس الطول
3. **استخدام خوارزميات ضعيفة**: تجنب `md5` و `sha1` للبيانات الحساسة
4. **نسيان تمرير encoding**: إذا أرسلت string، حدد `'utf8'` أو encoding آخر

---

#### **`crypto.createHmac(algorithm, key[, options])`**

**الصيغة:**

```
crypto.createHmac(algorithm, key, options) → Hmac
```

**الوصف:**
تنشئ كائن `Hmac` لحساب رمز مصادقة الرسالة (HMAC). بخلاف Hash العادي، تستخدم HMAC مفتاحاً سراً، مما يجعلها مناسبة للتحقق من أصل البيانات وسلامتها.

**جدول المعاملات:**

| المعامل     | النوع                     | مطلوب/اختياري | الوصف                                         |
| :---------- | :------------------------ | :------------ | :-------------------------------------------- |
| `algorithm` | string                    | مطلوب         | خوارزمية التجزئة (مثل `'sha256'`, `'sha512'`) |
| `key`       | string, Buffer, KeyObject | مطلوب         | المفتاح السري المستخدم لحساب HMAC             |
| `options`   | Object                    | اختياري       | خيارات إضافية (مثل `encoding` للمفتاح)        |

**القيمة المرجعة:**
كائن `Hmac` جاهز لـ `update()` و `digest()`.

**مثال عملي - التحقق من توقيع webhook من API خارجي:**

```javascript
const crypto = require("crypto");

// حالة واقعية: التحقق من أن webhook قادم من Stripe (أو أي خدمة دفع)
function verifyWebhookSignature(payload, signature, webhookSecret) {
  // حساب HMAC-SHA256 باستخدام المفتاح السري
  const hmac = crypto.createHmac("sha256", webhookSecret);

  // تحديث مع نص الطلب الكامل
  hmac.update(payload);

  // الحصول على التوقيع المتوقع بصيغة hexadecimal
  const expectedSignature = hmac.digest("hex");

  // مقارنة آمنة ضد timing attacks
  try {
    crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expectedSignature)
    );
    return true;
  } catch {
    return false; // التوقيعات غير متطابقة
  }
}

// الاستخدام مع Express middleware
const express = require("express");
app.post("/webhook/payment", (req, res) => {
  const signature = req.headers["x-signature-256"];
  const payload = req.rawBody; // يجب حفظ raw body

  if (verifyWebhookSignature(payload, signature, process.env.WEBHOOK_SECRET)) {
    console.log("Webhook موثوق - معالجة الدفع...");
    // معالجة الطلب
  } else {
    console.error("توقيع webhook غير صحيح - رفض الطلب");
    res.status(401).send("Unauthorized");
  }
});
```

**الأخطاء الشابعة:**

1. **استخدام المفتاح بطول قصير**: استخدم مفاتيح على الأقل 32 بايت (256 بت)
2. **عدم التحقق من الهوية**: HMAC تتحقق من السلامة، لكن ليس من الهوية بدون سياق إضافي
3. **إعادة استخدام نفس المفتاح**: يفضل دوران المفاتيح بشكل دوري
4. **الخلط بين الخوارزميات**: تأكد من استخدام نفس الخوارزمية في الحساب والتحقق

---

### الجزء الثاني: التشفير المتماثل (Symmetric Encryption)

#### **`crypto.createCipheriv(algorithm, key, iv[, options])`**

**الصيغة:**

```
crypto.createCipheriv(algorithm, key, iv, options) → Cipheriv
```

**الوصف:**
تنشئ كائن `Cipheriv` لتشفير البيانات باستخدام خوارزمية متماثلة. هذه هي الطريقة الآمنة الحديثة (بدلاً من `createCipher` المهجورة). تتطلب مفتاح وmsg)Initialization Vector (IV) عشوائي.

**جدول المعاملات:**

| المعامل     | النوع                     | مطلوب/اختياري | الوصف                                                   |
| :---------- | :------------------------ | :------------ | :------------------------------------------------------ |
| `algorithm` | string                    | مطلوب         | خوارزمية التشفير (مثل `'aes-256-cbc'`, `'aes-256-gcm'`) |
| `key`       | Buffer, string, KeyObject | مطلوب         | المفتاح (يجب أن يطابق طول الخوارزمية)                   |
| `iv`        | Buffer, string            | مطلوب         | Initialization Vector (عشوائي، لا يجب أن يكون سراً)     |
| `options`   | Object                    | اختياري       | `authTagLength` للأوضاع المصادقة (GCM, CCM)             |

**القيمة المرجعة:**
كائن `Cipheriv` يدعم streaming أو الطرق `update()` و `final()`.

**مثال عملي - تشفير بيانات حساسة للعميل قبل التخزين:**

```javascript
const crypto = require("crypto");
const fs = require("fs");

class DataEncryption {
  constructor(masterKey) {
    // المفتاح الرئيسي (يجب 32 بايت لـ AES-256)
    this.masterKey = crypto.scryptSync(masterKey, "salt", 32);
  }

  // تشفير بيانات نصية (مثل أرقام بطاقات ائتمان أو SSN)
  encryptSensitiveData(plaintext) {
    // إنشاء IV عشوائي جديد لكل تشفير
    const iv = crypto.randomBytes(16);

    // إنشاء cipher
    const cipher = crypto.createCipheriv("aes-256-cbc", this.masterKey, iv);

    // تحديث مع البيانات النصية
    let encrypted = cipher.update(plaintext, "utf8", "hex");
    encrypted += cipher.final("hex");

    // دائماً أرسل IV مع البيانات المشفرة (IV ليس سراً)
    return {
      iv: iv.toString("hex"),
      encrypted: encrypted,
    };
  }

  // فك تشفير البيانات
  decryptSensitiveData(encryptedData) {
    const iv = Buffer.from(encryptedData.iv, "hex");

    // إنشاء decipher باستخدام نفس المفتاح والـ IV
    const decipher = crypto.createDecipheriv("aes-256-cbc", this.masterKey, iv);

    let decrypted = decipher.update(encryptedData.encrypted, "hex", "utf8");
    decrypted += decipher.final("utf8");

    return decrypted;
  }
}

// الاستخدام
const encryption = new DataEncryption(process.env.MASTER_KEY);

// تشفير رقم بطاقة ائتمان
const creditCard = "4532-1488-0343-6467";
const encrypted = encryption.encryptSensitiveData(creditCard);

console.log("بيانات مشفرة:", encrypted);
// { iv: 'a1b2c3...', encrypted: 'x9y8z7...' }

// فك التشفير
const decrypted = encryption.decryptSensitiveData(encrypted);
console.log("بيانات فك تشفيرها:", decrypted); // '4532-1488-0343-6467'
```

**الأخطاء الشائعة:**

1. **استخدام IV ثابت**: كل تشفير يجب أن يستخدم IV عشوائي جديد، وإلا تظهر أنماط في البيانات المشفرة
2. **عدم حفظ IV**: بدون IV لا يمكنك فك التشفير - احفظه مع البيانات المشفرة
3. **استخدام مفتاح ضعيف**: لا تستخدم كلمات مرور مباشرة كمفاتيح - استخدم `scrypt` أو `pbkdf2`
4. **عدم استخدام مود مصادقة**: استخدم GCM أو CCM للتحقق من أن البيانات لم تُعدل

---

#### **`crypto.createDecipheriv(algorithm, key, iv[, options])`**

**الصيغة:**

```
crypto.createDecipheriv(algorithm, key, iv, options) → Decipheriv
```

**الوصف:**
عكس `createCipheriv` - تفك التشفير. يجب استخدام نفس الخوارزمية والمفتاح والـ IV المستخدمة في التشفير.

**مثال عملي - فك تشفير ملف:**

```javascript
const crypto = require("crypto");
const fs = require("fs");
const { pipeline } = require("stream");

function decryptFile(encryptedFilePath, outputPath, key, iv) {
  const input = fs.createReadStream(encryptedFilePath);
  const output = fs.createWriteStream(outputPath);

  // إنشاء decipher
  const decipher = crypto.createDecipheriv("aes-256-cbc", key, iv);

  // Streaming من الملف المشفر مباشرة إلى الملف المفك تشفيره
  pipeline(input, decipher, output, (err) => {
    if (err) {
      console.error("خطأ في فك التشفير:", err);
    } else {
      console.log("تم فك تشفير الملف بنجاح");
    }
  });
}
```

---

### الجزء الثالث: التوقيعات الرقمية (Digital Signatures)

#### **`crypto.createSign(algorithm[, options])`**

**الصيغة:**

```
crypto.createSign(algorithm, options) → Sign
```

**الوصف:**
تنشئ كائن `Sign` لإنشاء توقيع رقمي للبيانات باستخدام مفتاح خاص (Private Key). يتم بعدها التحقق من التوقيع بمفتاح عام (Public Key).

**جدول المعاملات:**

| المعامل     | النوع  | مطلوب/اختياري | الوصف                                         |
| :---------- | :----- | :------------ | :-------------------------------------------- |
| `algorithm` | string | مطلوب         | خوارزمية التجزئة (مثل `'sha256'`, `'sha512'`) |
| `options`   | Object | اختياري       | خيارات الـ stream                             |

**القيمة المرجعة:**
كائن `Sign` جاهز للـ `update()` و `sign()`.

**مثال عملي - توقيع شهادة فاتورة رقمية:**

```javascript
const crypto = require("crypto");
const fs = require("fs");

// حالة واقعية: شركة توقع الفواتير الرقمية
class InvoiceSigner {
  constructor() {
    // إنشاء أو تحميل المفاتيح (في بيئة الإنتاج، حمّلها من مكان آمن)
    const { privateKey, publicKey } = crypto.generateKeyPairSync("rsa", {
      modulusLength: 2048,
      publicKeyEncoding: {
        type: "spki",
        format: "pem",
      },
      privateKeyEncoding: {
        type: "pkcs8",
        format: "pem",
      },
    });

    this.privateKey = privateKey;
    this.publicKey = publicKey;
  }

  // توقيع فاتورة
  signInvoice(invoiceData) {
    const signer = crypto.createSign("sha256");

    // تحديث مع تفاصيل الفاتورة
    signer.update(JSON.stringify(invoiceData));

    // التوقيع بالمفتاح الخاص
    const signature = signer.sign(this.privateKey, "base64");

    return {
      invoice: invoiceData,
      signature: signature,
      timestamp: new Date().toISOString(),
    };
  }

  // التحقق من صحة التوقيع
  verifyInvoice(signedInvoice) {
    const verifier = crypto.createVerify("sha256");

    // تحديث مع تفاصيل الفاتورة الأصلية
    verifier.update(JSON.stringify(signedInvoice.invoice));

    // التحقق من التوقيع
    return verifier.verify(this.publicKey, signedInvoice.signature, "base64");
  }
}

// الاستخدام
const signer = new InvoiceSigner();

const invoice = {
  id: "INV-2024-001",
  amount: 1500.0,
  date: "2024-01-15",
  customer: "شركة النور التجارية",
};

// التوقيع على الفاتورة
const signedInvoice = signer.signInvoice(invoice);
console.log(
  "الفاتورة موقعة:",
  signedInvoice.signature.substring(0, 20) + "..."
);

// التحقق من التوقيع
const isValid = signer.verifyInvoice(signedInvoice);
console.log("التوقيع صحيح:", isValid); // true

// إذا حاول شخص تعديل الفاتورة...
signedInvoice.invoice.amount = 5000.0; // محاولة احتيال!
console.log("بعد التعديل:", signer.verifyInvoice(signedInvoice)); // false
```

**الأخطاء الشائعة:**

1. **نسيان تمرير private key للـ sign()**: يجب استخدام المفتاح الخاص، ليس العام
2. **استخدام خوارزمية مختلفة في التحقق**: يجب استخدام نفس الخوارزمية
3. **عدم حفظ التوقيع بشكل آمن**: التوقيع نفسه ليس سراً، لكن اعتمد عليه في الإثبات
4. **الخلط بين الـ padding**: RSA قد يتطلب padding options معينة

---

#### **`crypto.createVerify(algorithm[, options])`**

**الصيغة:**

```
crypto.createVerify(algorithm, options) → Verify
```

**الوصف:**
عكس `createSign` - التحقق من التوقيع باستخدام المفتاح العام. يجب أن تستخدم نفس الخوارزمية المستخدمة في التوقيع.

**انظر المثال أعلاه للاستخدام الكامل.**

---

### الجزء الرابع: إنشاء المفاتيح (Key Generation)

#### **`crypto.generateKeyPair(type, options, callback)`**

**الصيغة:**

```
crypto.generateKeyPair(type, options, callback)
```

**الوصف:**
إنشاء زوج مفاتيح (عام وخاص) بشكل غير متزامن. تستخدم callback بدلاً من الانتظار.

**جدول المعاملات:**

| المعامل    | النوع    | مطلوب/اختياري | الوصف                                                                                        |
| :--------- | :------- | :------------ | :------------------------------------------------------------------------------------------- |
| `type`     | string   | مطلوب         | نوع المفتاح (`'rsa'`, `'ec'`, `'dsa'`, `'dh'`, `'ed25519'`, `'ed448'`, `'x25519'`, `'x448'`) |
| `options`  | Object   | مطلوب         | خيارات حسب النوع (modulusLength, namedCurve، إلخ)                                            |
| `callback` | Function | مطلوب         | يستدعى مع `(err, publicKey, privateKey)`                                                     |

**مثال عملي - إنشاء مفاتيح RSA لخادم HTTPS:**

```javascript
const crypto = require("crypto");
const fs = require("fs");
const https = require("https");
const express = require("express");

// إنشاء مفاتيح RSA لخادم ويب آمن
crypto.generateKeyPair(
  "rsa",
  {
    modulusLength: 2048, // طول المفتاح بالبت
    publicKeyEncoding: {
      type: "spki", // قياسي PKIX
      format: "pem",
    },
    privateKeyEncoding: {
      type: "pkcs8", // قياسي PKCS#8
      format: "pem",
      // يمكنك إضافة cipher و passphrase لتشفير المفتاح الخاص
      // cipher: 'aes-256-cbc',
      // passphrase: process.env.KEY_PASSPHRASE
    },
  },
  (err, publicKey, privateKey) => {
    if (err) {
      console.error("خطأ في إنشاء المفاتيح:", err);
      return;
    }

    // حفظ المفاتيح
    fs.writeFileSync("public.pem", publicKey);
    fs.writeFileSync("private.pem", privateKey);

    console.log("تم إنشاء المفاتيح بنجاح");

    // إعداد خادم HTTPS
    const app = express();
    app.get("/", (req, res) => {
      res.send("مرحباً بك في خادم آمن!");
    });

    const httpsOptions = {
      key: privateKey,
      cert: publicKey, // ملاحظة: في الإنتاج تحتاج شهادة موقعة
    };

    https.createServer(httpsOptions, app).listen(3000, () => {
      console.log("الخادم يعمل على https://localhost:3000");
    });
  }
);
```

---

#### **`crypto.generateKeyPairSync(type, options)`**

**الصيغة:**

```
crypto.generateKeyPairSync(type, options) → { publicKey, privateKey }
```

**الوصف:**
نفس `generateKeyPair` لكن متزامن (blocking). استخدمها فقط في startup scripts، لا في request handlers.

**مثال عملي - إنشاء مفاتيح Elliptic Curve للتوقيع:**

```javascript
const crypto = require("crypto");

// إنشاء مفاتيح EC للتوقيع السريع (أسرع من RSA)
const { publicKey, privateKey } = crypto.generateKeyPairSync("ec", {
  namedCurve: "secp256r1", // معايير NIST P-256
  publicKeyEncoding: {
    type: "spki",
    format: "pem",
  },
  privateKeyEncoding: {
    type: "pkcs8",
    format: "pem",
  },
});

console.log("Public Key (EC):", publicKey);
console.log("Private Key (EC):", privateKey);

// الاستخدام في JWT tokens
const jwt = require("jsonwebtoken");

const token = jwt.sign(
  { userId: 123, email: "user@example.com" },
  privateKey,
  { algorithm: "ES256" } // Elliptic Curve Signature Algorithm
);

console.log("JWT Token:", token);
```

---

### الجزء الخامس: الدوال الأساسية (Utilities)

#### **`crypto.randomBytes(size[, callback])`**

**الصيغة:**

```
crypto.randomBytes(size, callback) → Buffer | void
```

**الوصف:**
توليد بيانات عشوائية آمنة من الناحية التشفيرية. تستخدم `/dev/urandom` في Unix أو CryptGenRandom في Windows.

**جدول المعاملات:**

| المعامل    | النوع    | مطلوب/اختياري | الوصف                                                 |
| :--------- | :------- | :------------ | :---------------------------------------------------- |
| `size`     | number   | مطلوب         | عدد البايتات العشوائية المطلوبة                       |
| `callback` | Function | اختياري       | يستدعى مع `(err, buffer)` - إذا لم يُمرر، ترجع buffer |

**القيمة المرجعة:**
`Buffer` يحتوي على بيانات عشوائية آمنة.

**مثال عملي - إنشاء token آمن لإعادة تعيين كلمة المرور:**

```javascript
const crypto = require("crypto");
const bcrypt = require("bcrypt");

class PasswordReset {
  // إنشاء رمز إعادة تعيين آمن
  generateResetToken() {
    // 32 بايت = 256 بت من العشوائية
    const token = crypto.randomBytes(32).toString("hex");

    // حفظ hash الرمز (ليس الرمز نفسه)
    const hashedToken = crypto.createHash("sha256").update(token).digest("hex");

    return {
      token: token, // أرسل هذا للمستخدم
      hashedToken: hashedToken, // احفظ هذا في قاعدة البيانات
      expiresAt: Date.now() + 3600000, // ساعة واحدة
    };
  }

  // التحقق من الرمز
  verifyResetToken(userProvidedToken, storedHashedToken) {
    const hashedProvided = crypto
      .createHash("sha256")
      .update(userProvidedToken)
      .digest("hex");

    try {
      crypto.timingSafeEqual(
        Buffer.from(hashedProvided),
        Buffer.from(storedHashedToken)
      );
      return true;
    } catch {
      return false;
    }
  }
}

// الاستخدام
const reset = new PasswordReset();
const { token, hashedToken } = reset.generateResetToken();

console.log("أرسل هذا الرمز للمستخدم:", token);
console.log("احفظ هذا في قاعدة البيانات:", hashedToken);
```

---

#### **`crypto.scrypt(password, salt, keylen, [options], callback)`**

**الصيغة:**

```
crypto.scrypt(password, salt, keylen, options, callback)
```

**الوصف:**
دالة اشتقاق المفاتيح المقاومة للذاكرة (Memory-hard KDF). توفر حماية أقوى ضد هجمات البحث الشامل مقارنة بـ PBKDF2.

**جدول المعاملات:**

| المعامل    | النوع          | مطلوب/اختياري | الوصف                           |
| :--------- | :------------- | :------------ | :------------------------------ |
| `password` | string, Buffer | مطلوب         | كلمة المرور                     |
| `salt`     | string, Buffer | مطلوب         | salt عشوائي (16 بايت على الأقل) |
| `keylen`   | number         | مطلوب         | طول المفتاح المشتق بالبايت      |
| `options`  | Object         | اختياري       | `N`, `r`, `p` (معاملات التكلفة) |
| `callback` | Function       | مطلوب         | يستدعى مع `(err, derivedKey)`   |

**مثال عملي - تجزئة كلمات المرور بأمان:**

```javascript
const crypto = require("crypto");

class PasswordManager {
  // تجزئة كلمة مرور جديدة
  async hashPassword(password) {
    // إنشاء salt عشوائي
    const salt = crypto.randomBytes(16);

    return new Promise((resolve, reject) => {
      // استخدام scrypt مع معاملات قوية
      crypto.scrypt(
        password,
        salt,
        64, // 64 بايت من الطول
        {
          N: 16384, // معامل التكلفة (الافتراضي)
          r: 8, // حجم الكتلة
          p: 1, // التوازي
        },
        (err, derivedKey) => {
          if (err) reject(err);

          // احفظ salt + hash معاً
          resolve({
            salt: salt.toString("hex"),
            hash: derivedKey.toString("hex"),
          });
        }
      );
    });
  }

  // التحقق من كلمة مرور
  async verifyPassword(password, storedData) {
    const salt = Buffer.from(storedData.salt, "hex");

    return new Promise((resolve, reject) => {
      crypto.scrypt(
        password,
        salt,
        64,
        {
          N: 16384,
          r: 8,
          p: 1,
        },
        (err, derivedKey) => {
          if (err) reject(err);

          // مقارنة آمنة ضد timing attacks
          try {
            crypto.timingSafeEqual(
              Buffer.from(derivedKey.toString("hex")),
              Buffer.from(storedData.hash)
            );
            resolve(true);
          } catch {
            resolve(false);
          }
        }
      );
    });
  }
}

// الاستخدام
const passwordManager = new PasswordManager();

// تسجيل مستخدم جديد
passwordManager
  .hashPassword("mySecurePassword123!")
  .then((hashedData) => {
    console.log("احفظ هذا في قاعدة البيانات:", hashedData);

    // محاكاة تسجيل الدخول
    return passwordManager.verifyPassword("mySecurePassword123!", hashedData);
  })
  .then((isValid) => {
    console.log("كلمة المرور صحيحة:", isValid); // true
  });
```

---

#### **`crypto.pbkdf2(password, salt, iterations, keylen, digest, callback)`**

**الصيغة:**

```
crypto.pbkdf2(password, salt, iterations, keylen, digest, callback)
```

**الوصف:**
دالة اشتقاق المفاتيح القائمة على كلمة المرور (PBKDF2). أقل أماناً من scrypt لكن أسرع وموثوقة.

**جدول المعاملات:**

| المعامل      | النوع          | مطلوب/اختياري | الوصف                             |
| :----------- | :------------- | :------------ | :-------------------------------- |
| `password`   | string, Buffer | مطلوب         | كلمة المرور                       |
| `salt`       | string, Buffer | مطلوب         | salt عشوائي (32 بايت على الأقل)   |
| `iterations` | number         | مطلوب         | عدد التكرارات (100,000 على الأقل) |
| `keylen`     | number         | مطلوب         | طول المفتاح المشتق                |
| `digest`     | string         | مطلوب         | دالة التجزئة (مثل `'sha256'`)     |
| `callback`   | Function       | مطلوب         | يستدعى مع `(err, derivedKey)`     |

**مثال عملي - API keys مشتقة من كلمة المرور الرئيسية:**

```javascript
const crypto = require("crypto");

class APIKeyManager {
  // اشتقاق API key من كلمة مرور رئيسية
  async deriveAPIKey(masterPassword, userId) {
    // استخدام معرّف المستخدم كـ salt
    const salt = Buffer.from(userId);

    return new Promise((resolve, reject) => {
      crypto.pbkdf2(
        masterPassword,
        salt,
        100000, // 100,000 تكرار (معيار OWASP)
        32, // 32 بايت للمفتاح
        "sha256",
        (err, derivedKey) => {
          if (err) reject(err);
          resolve(derivedKey.toString("hex"));
        }
      );
    });
  }
}

// الاستخدام
const apiManager = new APIKeyManager();

apiManager.deriveAPIKey("master-password-123", "user-456").then((apiKey) => {
  console.log("API Key:", apiKey);
  // استخدم هذا كـ API key
});
```

---

#### **`crypto.timingSafeEqual(a, b)`**

**الصيغة:**

```
crypto.timingSafeEqual(a, b) → boolean
```

**الوصف:**
مقارنة آمنة ضد **timing attacks**. الدالة تأخذ نفس الوقت بغض النظر عن أين تختلف البيانات، مما يمنع المهاجمين من معرفة البيانات الصحيحة من خلال قياس الوقت.

**جدول المعاملات:**

| المعامل | النوع  | مطلوب/اختياري | الوصف            |
| :------ | :----- | :------------ | :--------------- |
| `a`     | Buffer | مطلوب         | البيانات الأولى  |
| `b`     | Buffer | مطلوب         | البيانات الثانية |

**القيمة المرجعة:**
`true` إذا كانت البيانات متطابقة، `false` خلاف ذلك.

**مثال عملي - التحقق الآمن من رموز المصادقة:**

```javascript
const crypto = require("crypto");

class SecureAuth {
  // مقارنة رمز المصادقة بشكل آمن
  verifyAuthCode(userProvidedCode, storedCode) {
    const userBuffer = Buffer.from(userProvidedCode);
    const storedBuffer = Buffer.from(storedCode);

    // تأكد من نفس الطول لتجنب تسرب المعلومات
    if (userBuffer.length !== storedBuffer.length) {
      return false;
    }

    try {
      // مقارنة آمنة ضد timing attacks
      crypto.timingSafeEqual(userBuffer, storedBuffer);
      return true;
    } catch (err) {
      // الأخطاء تعني عدم المطابقة
      return false;
    }
  }
}

// الاستخدام
const auth = new SecureAuth();

const storedCode = "123456";
const userInput1 = "123456"; // صحيح
const userInput2 = "123457"; // خطأ
const userInput3 = "12"; // طول مختلف

console.log("كود صحيح:", auth.verifyAuthCode(userInput1, storedCode)); // true
console.log("كود خطأ:", auth.verifyAuthCode(userInput2, storedCode)); // false
console.log("طول مختلف:", auth.verifyAuthCode(userInput3, storedCode)); // false
```

---

## 3. مقارنات واختيار الدالة الصحيحة

### مقارنة دوال الهاش والتجزئة

| الميزة              | `Hash` (createHash) | `HMAC` (createHmac) | `scrypt`         | `pbkdf2`         |
| :------------------ | :------------------ | :------------------ | :--------------- | :--------------- |
| **الاستخدام**       | تجزئة عام           | مصادقة الرسائل      | تجزئة كلمات مرور | تجزئة كلمات مرور |
| **يتطلب مفتاح سري** | لا                  | نعم                 | نعم              | نعم              |
| **قابل للعكس**      | لا                  | لا                  | لا               | لا               |
| **مقاومة للذاكرة**  | لا                  | لا                  | نعم (قوية)       | لا               |
| **السرعة**          | سريج جداً           | سريع                | بطيء (آمن)       | متوسط            |
| **الحالات الأمثل**  | checksum، بصمات     | توثيق API، webhook  | تجزئة كلمات مرور | قوائم كلمات مرور |

**القرار: متى تستخدم أي واحدة؟**

- **Hash**: تحقق بسيط من السلامة (مثل checksum)
- **HMAC**: توثيق الرسائل من مصدر معروف
- **scrypt**: كلمات مرور المستخدمين (الخيار الأول)
- **pbkdf2**: المشاريع الموجودة أو إذا لم تتمكن من استخدام scrypt

---

### مقارنة دوال التشفير المتماثل

| الميزة            | AES-CBC                | AES-GCM                | ChaCha20-Poly1305  |
| :---------------- | :--------------------- | :--------------------- | :----------------- |
| **الحجم**         | أسرع قليلاً            | أسرع على معالجات حديثة | سريع جداً          |
| **المصادقة**      | NO - استخدم HMAC منفصل | YES - مدمج             | YES - مدمج         |
| **التعقيد**       | بسيط                   | متوسط                  | متوسط              |
| **الدعم**         | عام جداً               | موصى به                | موصى به            |
| **الحالة الأمثل** | ملفات قديمة            | تطبيقات جديدة          | أنظمة عالية الأداء |

---

### مقارنة دوال إنشاء المفاتيح

| الخاصية           | RSA          | ECDH         | Ed25519   | DH           |
| :---------------- | :----------- | :----------- | :-------- | :----------- |
| **السرعة**        | بطيء         | سريع         | سريع جداً | متوسط        |
| **حجم المفتاح**   | 2048-4096    | 256-521      | 256       | 2048+        |
| **الاستخدام**     | توقيع، تشفير | تبادل مفاتيح | توقيع     | تبادل مفاتيح |
| **الأمان الحالي** | جيد          | ممتاز        | ممتاز     | جيد          |
| **قياسي**         | موثوق        | موثوق        | حديث      | قديم         |

---

## 4. المزايا والعيوب والحالات الاستخدام

### مزايا وحدة crypto

1. **لا تبعيات خارجية**: مدمجة في Node.js، لا حاجة لتثبيت حزم
2. **أداء عالي**: تستخدم OpenSSL المترجمة الأصلية
3. **معايير صناعية**: تدعم الخوارزميات الموثوقة والمعتمدة
4. **دعم شامل**: هاش، تشفير، توقيع، مفاتيح - كل شيء في مكان واحد
5. **توثيق جيد**: توثيق رسمي Node.js شامل

### عيوب وحدة crypto

1. **منحنى تعليمي شديد**: واجهة برمجية معقدة نسبياً
2. **سهل الخطأ**: يمكن للمطورين إساءة استخدام الدوال بسهولة
3. **لا دعم للخوارزميات الحديثة**: مثل Argon2 و bcrypt (خارج crypto)
4. **معاملات كثيرة**: بعض الدوال تحتاج معاملات كثيرة وخيارات
5. **بطء في بعض العمليات**: مثل scrypt و pbkdf2 (بطء عن قصد للأمان)

### أفضل 3 حالات استخدام

**الحالة الأولى: تطبيق ويب آمن مع مصادقة المستخدم**

```javascript
// تجزئة كلمات المرور: scrypt
// التحقق من الهوية: HMAC على session tokens
// توقيع الرسائل: Sign/Verify
// تشفير بيانات حساسة: AES-GCM
```

**الحالة الثانية: خدمة ويب REST تتطلب توقيع الطلبات**

```javascript
// إنشاء مفاتيح عام/خاص: generateKeyPair
// توقيع الطلب: createSign
// التحقق من الطلب: createVerify
// HMAC للرؤوس: createHmac
```

**الحالة الثالثة: نظام تخزين آمن (مثل خدمة سحابة)**

````javascript
// مفتاح تشفير من كلمة مرور: scrypt
// تشفير الملفات: AES-256-GCM
// بصمة الملف: createHash('sha256')
// Integrity check: HMAC على الملف
```---

## 5. الخلاصة والتوصيات

وحدة `crypto` في Node.js هي أداة قوية وضرورية لأي تطبيق يتطلب أماناً تشفيرياً. المفاتيح للاستخدام الآمن هي:

1. **استخدم دائماً IVs عشوائية** في التشفير
2. **استخدم scrypt أو bcrypt** لتجزئة كلمات المرور
3. **استخدم timingSafeEqual** عند مقارنة البيانات الحساسة
4. **استخدم أوضاع مصادقة** (GCM) بدلاً من CBC البسيط
5. **أنشئ و أدر المفاتيح بحكمة** - احفظها في متغيرات البيئة أو vaults

مع اتباع هذه الممارسات، يمكنك بناء تطبيقات Node.js آمنة وموثوقة.
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_3][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39][^1_4][^1_40][^1_41][^1_42][^1_43][^1_44][^1_45][^1_46][^1_47][^1_48][^1_49][^1_5][^1_50][^1_51][^1_52][^1_53][^1_54][^1_55][^1_56][^1_57][^1_58][^1_59][^1_6][^1_60][^1_61][^1_62][^1_7][^1_8][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://www.semanticscholar.org/paper/a2e5899a88a246ddc69c6d2495b043c8e4d952ff
[^1_2]: https://ieeexplore.ieee.org/document/10778454/
[^1_3]: https://www.semanticscholar.org/paper/23cac78121a77ac997ccc504a126589d65b4d6a7
[^1_4]: http://link.springer.com/10.1007/978-1-4842-0037-7
[^1_5]: https://www.semanticscholar.org/paper/c90a33514853cd99d37fd89a336cc2ba007ba3c3
[^1_6]: https://proceedings.elseconference.eu/index.php?paper=f9a5d930857f71858c232ae88a9aae3b
[^1_7]: https://www.semanticscholar.org/paper/d9d2d6eaf986af3e3aabb9453359e6869215a669
[^1_8]: https://www.semanticscholar.org/paper/725bc2fcd301b4bf837ba073ee6d20f2eb74f585
[^1_9]: https://arxiv.org/pdf/2109.01002.pdf
[^1_10]: https://arxiv.org/pdf/2312.17689.pdf
[^1_11]: https://arxiv.org/pdf/2108.11867.pdf
[^1_12]: https://academic.oup.com/bioinformatics/article/doi/10.1093/bioinformatics/btae763/7943493
[^1_13]: https://wjcm.uowasit.edu.iq/index.php/wjcm/article/download/41/18
[^1_14]: https://arxiv.org/pdf/1810.09065.pdf
[^1_15]: http://arxiv.org/pdf/1807.01095.pdf
[^1_16]: http://arxiv.org/pdf/2409.05128.pdf
[^1_17]: https://www.w3schools.com/nodejs/nodejs_crypto.asp
[^1_18]: https://stackoverflow.com/questions/78687009/how-to-encrypt-decrypt-with-node-js-crypto-module
[^1_19]: https://stackoverflow.com/questions/7480158/how-do-i-use-node-js-crypto-to-create-a-hmac-sha1-hash
[^1_20]: https://github.com/nodejs/node/blob/main/doc/api/crypto.md
[^1_21]: https://www.w3schools.com/nodejs/ref_decipher.asp
[^1_22]: https://compile7.org/hashing/how-to-use-hmac-sha256-in-nodejs/
[^1_23]: https://www.cs.unb.ca/~bremner/teaching/cs2613/books/nodejs-api/crypto/
[^1_24]: https://www.youtube.com/watch?v=iH54fek9xc4
[^1_25]: https://www.geeksforgeeks.org/node-js/node-js-crypto-createhmac-method/
[^1_26]: https://blog.logrocket.com/node-js-crypto-module-a-tutorial/
[^1_27]: https://blog.stackademic.com/encrypt-and-decrypt-data-in-node-js-2a02d8dbd5ff?gi=6c9163fe1d80
[^1_28]: https://ssojet.com/hashing/hmac-sha256-in-nodejs/
[^1_29]: https://www.geeksforgeeks.org/node-js/what-is-crypto-module-in-node-js-and-how-it-is-used/
[^1_30]: https://mrebi.com/en/nodejs/nodejs-crypto/
[^1_31]: https://ssojet.com/hashing/hmac-sha512-in-nodejs/
[^1_32]: https://nodejs.org/api/crypto.html
[^1_33]: https://www.tatvasoft.com/outsourcing/2024/01/nodejs-cryptography.html
[^1_34]: https://stackoverflow.com/questions/72763485/how-do-i-validate-the-hmac-using-nodejs
[^1_35]: https://nodejs.org/download/release/v10.24.1/docs/api/crypto.html
[^1_36]: https://nodejs.org/download/release/v4.4.1/docs/api/crypto.html
[^1_37]: https://dl.acm.org/doi/pdf/10.1145/3583780.3615090
[^1_38]: http://arxiv.org/pdf/2406.15596.pdf
[^1_39]: https://onlinelibrary.wiley.com/doi/pdfdirect/10.1049/ise2.12026
[^1_40]: https://arxiv.org/pdf/2207.09319.pdf
[^1_41]: http://arxiv.org/pdf/2308.06797.pdf
[^1_42]: https://dl.acm.org/doi/pdf/10.1145/3576915.3623149
[^1_43]: https://arxiv.org/pdf/2204.06447.pdf
[^1_44]: https://stackoverflow.com/questions/8520973/how-to-create-a-pair-private-public-keys-using-node-js-crypto
[^1_45]: https://www.geeksforgeeks.org/node-js/node-js-crypto-generatekeypairsync-method/
[^1_46]: https://javascript.plainenglish.io/how-to-handle-password-hashing-in-node-js-3448b24c0de8
[^1_47]: https://bun.com/reference/node/crypto/Sign
[^1_48]: https://stackoverflow.com/questions/76369898/default-options-for-crypto-modules-function-generatekeypairsync-or-generatekeypa
[^1_49]: https://stackoverflow.com/questions/17218089/salt-and-hash-using-pbkdf2
[^1_50]: https://www.geeksforgeeks.org/node-js/node-js-crypto-createverify-method/
[^1_51]: https://www.geeksforgeeks.org/node-js/node-js-crypto-generatekeypair-method/
[^1_52]: https://nodejs.org/download/rc/v8.12.0-rc.2/docs/api/crypto.html
[^1_53]: https://stackoverflow.com/questions/48611041/using-node-js-crypto-to-verify-signatures
[^1_54]: https://zanechua.com/blog/generate-ec-private-public-key-pair-node/
[^1_55]: https://gist.github.com/wuriyanto48/414e5415c85b2208bb18ed055439761c
[^1_56]: https://www.w3schools.com/nodejs/ref_sign.asp
[^1_57]: https://docs.denohub.com/api/node/crypto/~/generateKeyPairSync
[^1_58]: https://wiki.hsoub.com/Node.js/crypto
[^1_59]: https://www.tutorialspoint.com/crypto-generatekeypair-method-in-node-js
[^1_60]: https://guptadeepak.com/the-complete-guide-to-password-hashing-argon2-vs-bcrypt-vs-scrypt-vs-pbkdf2-2026/
[^1_61]: https://www.geeksforgeeks.org/node-js/node-js-crypto-privatedecrypt-method/
[^1_62]: https://www.tutorialspoint.com/crypto-generatekeypairsync-method-in-node-js```

````

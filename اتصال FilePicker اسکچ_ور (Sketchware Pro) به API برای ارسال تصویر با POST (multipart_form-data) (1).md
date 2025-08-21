# اتصال FilePicker اسکچ‌ور (Sketchware Pro) به API برای ارسال تصویر با POST (multipart/form-data)

این راهنما قدم‌به‌قدم تمام کدهای جاوا و محل قرار دادن آن‌ها را توضیح می‌دهد تا با FilePicker تصاویر را به API زیر ارسال کنید:

- Endpoint: `https://api2.api-code.ir/gpt-save-v2/`
- روش: `POST` با `multipart/form-data`
- پارامترها (طبق مستندات):
  - `userid` (ضروری)
  - `prompt` (اختیاری وقتی فایل تصویر دارید)
  - `model` (اختیاری – پیش‌فرض openai؛ در پروژه‌ی شما قبلاً `deepseek` استفاده شده بود)
  - `image_file` (فایل تصویر)

نکته: نیازی به کلید API نیست.

---

## 0) پیش‌نیازها در Sketchware

- **Component ها:**
  - FilePicker با فیلتر `image/*` (در تصاویر شما با نام `mahdi` وجود دارد)
  - (اختیاری) RequestNetwork برای درخواست‌های قبلی – اما برای آپلود فایل از کد سفارشی زیر استفاده می‌کنیم.
- **مجوزها:**
  - «Internet» فعال باشد (به‌صورت پیش‌فرض با RequestNetwork فعال است؛ اگر نبود از Project → Permissions فعال کنید).
  - برای اندروید 13 به بالا: `READ_MEDIA_IMAGES`
  - برای اندروید 12 و پایین: `READ_EXTERNAL_STORAGE`

کد درخواست مجوز را پایین‌تر گذاشته‌ام.

---

## 1) درخواست مجوز خواندن تصاویر (onCreate یا initializeLogic)

این کد برای درخواست مجوزهای لازم برای خواندن تصاویر از حافظه دستگاه است. این کد باید هنگام شروع اکتیویتی شما اجرا شود.

**محل قرارگیری:**
در Sketchware Pro، به تب «Event» بروید → رویداد `onCreate` (یا `initializeLogic`) اکتیویتی اصلی خود را باز کنید → سپس یک بلوک «Add source directly» (بلوک سبز رنگ) را به آن اضافه کنید و کد زیر را داخل آن قرار دهید.

**کد جاوا:**
```java
if (android.os.Build.VERSION.SDK_INT >= 33) {
    if (checkSelfPermission(android.Manifest.permission.READ_MEDIA_IMAGES) != android.content.pm.PackageManager.PERMISSION_GRANTED) {
        requestPermissions(new String[]{android.Manifest.permission.READ_MEDIA_IMAGES}, 1001);
    }
} else {
    if (checkSelfPermission(android.Manifest.permission.READ_EXTERNAL_STORAGE) != android.content.pm.PackageManager.PERMISSION_GRANTED) {
        requestPermissions(new String[]{android.Manifest.permission.READ_EXTERNAL_STORAGE}, 1002);
    }
}
```

**نکته:** می‌توانید در «Add source directly» بخش `onRequestPermissionsResult` هم (اختیاری) پیامی برای موفقیت/ناموفق بودن درخواست مجوز نمایش دهید.

---

## 2) رویداد کلیک روی آیکن تصویر (imageview10 → onClick)

این بخش مربوط به زمانی است که کاربر روی دکمه یا `ImageView` مربوط به انتخاب فایل کلیک می‌کند تا FilePicker باز شود.

**محل قرارگیری:**
در Sketchware Pro، به تب «View» بروید → `ImageView` مربوط به انتخاب فایل (در تصاویر شما `imageview10` نام دارد) را انتخاب کنید → به رویداد `onClick` آن بروید → یک بلوک «Add source directly» (بلوک سبز رنگ) را به آن اضافه کنید و کد زیر را داخل آن قرار دهید.

**کد جاوا:**
```java
mahdi.pickFiles(); // فرض بر این است که FilePicker شما با نام \'mahdi\' است
```

**توضیح:** اگر از بلوک‌های پیش‌فرض Sketchware برای فراخوانی FilePicker استفاده می‌کنید، نیازی به این کد نیست. این کد فقط برای فراخوانی FilePicker از طریق کد جاوا است.

---

## 3) دریافت فایل‌های انتخاب‌شده (FilePicker → onFilesPicked)

این مهم‌ترین بخش است که پس از انتخاب فایل توسط کاربر، فایل را برای ارسال به API آماده می‌کند و درخواست POST را می‌فرستد. تمام توابع و متغیرهای مورد نیاز برای ارسال فایل، در همین بخش تعریف شده‌اند.

**محل قرارگیری:**
در Sketchware Pro، به تب «Component» بروید → `FilePicker` خود (در تصاویر شما `mahdi` نام دارد) را انتخاب کنید → به رویداد `onFilesPicked(ArrayList<String> filePath)` بروید → یک بلوک «Add source directly» (بلوک سبز رنگ) را به آن اضافه کنید و **کل کد زیر را داخل آن قرار دهید**.

**کد جاوا:**
```java
// ==== Config API ====
final String API_URL = "https://api2.api-code.ir/gpt-save-v2/"; // انتهای / را نگه دارید
final String DEFAULT_MODEL = "deepseek"; // یا "openai" مطابق نیاز شما

// نخ برای کارهای شبکه
java.util.concurrent.ExecutorService executor = java.util.concurrent.Executors.newSingleThreadExecutor();

// نام فایل را از URI استخراج می‌کند
class FileUtils {
    String getFileNameFromUri(android.net.Uri uri) {
        String result = null;
        if ("content".equals(uri.getScheme())) {
            android.database.Cursor cursor = getContentResolver().query(uri, null, null, null, null);
            try {
                if (cursor != null && cursor.moveToFirst()) {
                    int idx = cursor.getColumnIndex(android.provider.OpenableColumns.DISPLAY_NAME);
                    if (idx >= 0) result = cursor.getString(idx);
                }
            } finally {
                if (cursor != null) cursor.close();
            }
        }
        if (result == null) {
            result = uri.getLastPathSegment();
        }
        return result == null ? "image.jpg" : result;
    }

    // MIME نوع محتوا را از URI می‌گیرد
    String getMimeFromUri(android.net.Uri uri) {
        String mime = getContentResolver().getType(uri);
        if (mime == null) mime = "image/jpeg"; // مقدار پیش‌فرض
        return mime;
    }

    // کپی داده از InputStream به OutputStream
    void copyStream(java.io.InputStream in, java.io.OutputStream out) throws java.io.IOException {
        byte[] buffer = new byte[8192];
        int read;
        while ((read = in.read(buffer)) != -1) {
            out.write(buffer, 0, read);
        }
    }

    /**
     * ارسال تصویر به صورت multipart/form-data
     * @param imageUri   URI تصویر (می‌تواند content:// یا file:// باشد)
     * @param userId     مقدار userid
     * @param prompt     متن دلخواه (اختیاری)
     * @param model      نام مدل (اختیاری)
     * @return پاسخ متنی سرور (JSON)
     */
    String postImageMultipart(android.net.Uri imageUri, String userId, String prompt, String model) throws Exception {
        String boundary = "----SketchwareFormBoundary" + java.util.UUID.randomUUID().toString();
        java.net.URL url = new java.net.URL(API_URL);
        java.net.HttpURLConnection conn = (java.net.HttpURLConnection) url.openConnection();
        conn.setDoInput(true);
        conn.setDoOutput(true);
        conn.setUseCaches(false);
        conn.setRequestMethod("POST");
        conn.setRequestProperty("Connection", "Keep-Alive");
        conn.setRequestProperty("Accept", "application/json");
        conn.setRequestProperty("Content-Type", "multipart/form-data; boundary=" + boundary);

        java.io.DataOutputStream out = new java.io.DataOutputStream(conn.getOutputStream());

        // helper برای نوشتن یک فیلد معمولی
        java.util.function.BiConsumer<String, String> writeField = (name, value) -> {
            try {
                out.writeBytes("--" + boundary + "\r\n");
                out.writeBytes("Content-Disposition: form-data; name=\"" + name + "\"\r\n\r\n");
                out.writeBytes(value + "\r\n");
            } catch (java.io.IOException e) {
                throw new RuntimeException(e);
            }
        };

        // فیلدهای متنی
        writeField.accept("userid", userId);
        if (prompt != null && prompt.length() > 0) writeField.accept("prompt", prompt);
        if (model != null && model.length() > 0) writeField.accept("model", model);

        // بخش فایل
        String fileName = getFileNameFromUri(imageUri);
        String mime = getMimeFromUri(imageUri);
        out.writeBytes("--" + boundary + "\r\n");
        out.writeBytes("Content-Disposition: form-data; name=\"image_file\"; filename=\"" + fileName + "\"\r\n");
        out.writeBytes("Content-Type: " + mime + "\r\n\r\n");

        java.io.InputStream in = getContentResolver().openInputStream(imageUri);
        copyStream(in, out);
        in.close();
        out.writeBytes("\r\n");

        // پایان بادی
        out.writeBytes("--" + boundary + "--\r\n");
        out.flush();
        out.close();

        int code = conn.getResponseCode();
        java.io.InputStream is = (code >= 200 && code < 300) ? conn.getInputStream() : conn.getErrorStream();
        java.io.BufferedReader br = new java.io.BufferedReader(new java.io.InputStreamReader(is, "UTF-8"));
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = br.readLine()) != null) sb.append(line);
        br.close();
        conn.disconnect();
        return sb.toString();
    }
}

// filePath: لیست رشته‌هایی که توسط FilePicker برمی‌گردد
if (filePath == null || filePath.size() == 0) {
    SketchwareUtil.showMessage(getApplicationContext(), "هیچ فایلی انتخاب نشد");
    return;
}

final FileUtils fileUtils = new FileUtils();

String first = filePath.get(0);
android.net.Uri uri = first.startsWith("content://") ? android.net.Uri.parse(first) : android.net.Uri.fromFile(new java.io.File(first));

// مقداردهی به پارامترها
String userId = "api_codeapi_gpt22"; // همان که در پروژه‌تان استفاده کرده‌اید
String promptText = edittext1.getText().toString(); // اگر EditText دارید که متن را از آن می‌گیرید
String model = DEFAULT_MODEL; // یا هر مدلی که می‌خواهید

// (اختیاری) نمایش پیام کاربر در لیست قبل از ارسال
// این بخش مشابه کاری است که بلوک‌های شما برای نمایش پیام کاربر انجام می‌دهند.
java.util.HashMap<String, Object> you = new java.util.HashMap<>();
you.put("Result", promptText.length() > 0 ? promptText : "[ارسال تصویر]");
you.put("key", "you");
list.add(you);
_refresh(); // فرض بر این است که تابع _refresh() برای به‌روزرسانی ListView وجود دارد

// ارسال در نخ جداگانه (برای جلوگیری از فریز شدن UI)
executor.execute(() -> {
    try {
        String resp = fileUtils.postImageMultipart(uri, userId, promptText, model);
        // بازگشت به UI Thread برای به‌روزرسانی رابط کاربری
        runOnUiThread(() -> {
            try {
                org.json.JSONObject obj = new org.json.JSONObject(resp);
                String result = obj.optString("Result", resp); // استخراج \'Result\' یا کل پاسخ

                // نمایش پاسخ سرور در ListView
                java.util.HashMap<String, Object> server = new java.util.HashMap<>();
                server.put("Result", result);
                server.put("key", "bot");
                list.add(server);
                _refresh(); // به‌روزرسانی ListView
                listview1.smoothScrollToPosition((int) (list.size())); // اسکرول به پایین
                linear11.setVisibility(android.view.View.GONE); // مخفی کردن linear11 (اگر برای نمایش وضعیت بود)
            } catch (Exception je) {
                SketchwareUtil.showMessage(getApplicationContext(), "JSON خطا: " + je.getMessage());
            }
        });
    } catch (Exception e) {
        runOnUiThread(() -> SketchwareUtil.showMessage(getApplicationContext(), "ارسال ناموفق: " + e.getMessage()));
    }
});
```

**توضیح:**
- این کد ابتدا بررسی می‌کند که آیا فایلی انتخاب شده است یا خیر.
- سپس `URI` فایل انتخاب شده را به دست می‌آورد (که می‌تواند مسیر `content://` یا `file://` باشد).
- پارامترهای `userid`، `prompt` و `model` را آماده می‌کند. `userid` را از پروژه شما (`api_codeapi_gpt22`) و `prompt` را از `edittext1` (اگر دارید) می‌گیرد.
- یک پیام پیش‌نمایش (اختیاری) در `ListView` شما اضافه می‌کند.
- تابع `postImageMultipart` را در یک `Thread` جداگانه اجرا می‌کند تا رابط کاربری مسدود نشود.
- پس از دریافت پاسخ از API، آن را تجزیه کرده و `Result` را در `ListView` نمایش می‌دهد.

---

## 4) اگر می‌خواهید پارامترهای بیشتری بفرستید

برای ارسال پارامترهای اضافی مانند `clear` (برای پاکسازی تاریخچه چت) یا تغییر مدل، می‌توانید تغییرات زیر را اعمال کنید:

**محل قرارگیری:**
این تغییرات را باید در تابع `postImageMultipart` که در بخش 3 (onFilesPicked) قرار دادید، اعمال کنید.

**کد جاوا (مثال برای `clear`):**
```java
// در تابع postImageMultipart، قبل از بخش فایل (// بخش فایل)
// اگر می‌خواهید چت را پاک کنید:
writeField.accept("clear", "true");
```

**تغییر مدل:**
مقدار `DEFAULT_MODEL` را در بخش 3 تغییر دهید، یا هنگام فراخوانی `postImageMultipart` در بخش 3، پارامتر `model` را با مقدار دلخواه خود جایگزین کنید.

---

## 5) نکات اشکال‌زدایی

اگر با مشکل مواجه شدید، این نکات می‌توانند به شما کمک کنند:

- **پاسخ خالی از API:**
  - **محل قرارگیری:** در تابع `postImageMultipart` (بخش 3)، بعد از خط `int code = conn.getResponseCode();`.
  - **کد جاوا:**
  ```java
  SketchwareUtil.showMessage(getApplicationContext(), "HTTP Status Code: " + code);
  ```
  این کد وضعیت HTTP را به شما نشان می‌دهد که می‌تواند در تشخیص مشکل کمک کند.

- **خطای «Permission denied» هنگام باز کردن `InputStream`:**
  - مطمئن شوید که مجوزهای `READ_MEDIA_IMAGES` یا `READ_EXTERNAL_STORAGE` (بخش 1) به درستی درخواست و اعطا شده‌اند.

- **مشکلات شبکه (اینترنت ضعیف یا قطع شدن):**
  - می‌توانید `timeout` برای اتصال و خواندن تنظیم کنید.
  - **محل قرارگیری:** در تابع `postImageMultipart` (بخش 3)، بعد از خط `java.net.HttpURLConnection conn = (java.net.HttpURLConnection) url.openConnection();`.
  - **کد جاوا:**
  ```java
  conn.setConnectTimeout(20000); // 20 ثانیه برای اتصال
  conn.setReadTimeout(60000);    // 60 ثانیه برای خواندن داده
  ```

---

## 6) نمایش پاسخ در همان ساختار فعلی شما

کدی که در بخش 3 ارائه شد، مستقیماً `list` (لیست داده‌های `ListView` شما) را به‌روزرسانی می‌کند و نیازی به بلوک‌های `RequestNetwork.receive` برای این عملیات آپلود خاص نیست. این روش با ساختار نمایش چت شما سازگار است.

---

## 7) خلاصه‌ی نهایی محل قرارگیری کدها

برای جمع‌بندی، کدهای جاوا را به شرح زیر در Sketchware Pro قرار دهید:

- **رویداد `onCreate` یا `initializeLogic` اکتیویتی اصلی (تب Event → onCreate/initializeLogic → بلوک سبز Add source directly):**
  - کدهای بخش 1 (درخواست مجوزها).

- **رویداد `onClick` دکمه/ImageView انتخاب فایل (تب View → imageview10 → onClick → بلوک سبز Add source directly):**
  - کدهای بخش 2 (فراخوانی FilePicker).

- **رویداد `onFilesPicked` کامپوننت FilePicker (تب Component → mahdi → onFilesPicked → بلوک سبز Add source directly):**
  - کدهای بخش 3 (شامل تمام توابع و منطق ارسال فایل به API).

با انجام این مراحل، FilePicker شما به درستی به API متصل شده و تصاویر را ارسال خواهد کرد. اگر سوال دیگری داشتید، بپرسید.

lity(android.view.View.GONE); // مخفی کردن linear11 (اگر برای نمایش وضعیت 


بود)
            } catch (Exception je) {
                SketchwareUtil.showMessage(getApplicationContext(), "JSON خطا: " + je.getMessage());
            }
        });
    } catch (Exception e) {
        runOnUiThread(() -> SketchwareUtil.showMessage(getApplicationContext(), "ارسال ناموفق: " + e.getMessage()));
    }
});
```

**توضیح:**
- این کد ابتدا بررسی می‌کند که آیا فایلی انتخاب شده است یا خیر.
- سپس `URI` فایل انتخاب شده را به دست می‌آورد (که می‌تواند مسیر `content://` یا `file://` باشد).
- پارامترهای `userid`، `prompt` و `model` را آماده می‌کند. `userid` را از پروژه شما (`api_codeapi_gpt22`) و `prompt` را از `edittext1` (اگر دارید) می‌گیرد.
- یک پیام 


پیش‌نمایش (اختیاری) در `ListView` شما اضافه می‌کند.
- تابع `postImageMultipart` را در یک `Thread` جداگانه اجرا می‌کند تا رابط کاربری مسدود نشود.
- پس از دریافت پاسخ از API، آن را تجزیه کرده و `Result` را در `ListView` نمایش می‌دهد.

---

## 5) اگر می‌خواهید پارامترهای بیشتری بفرستید

برای ارسال پارامترهای اضافی مانند `clear` (برای پاکسازی تاریخچه چت) یا تغییر مدل، می‌توانید تغییرات زیر را اعمال کنید:

**محل قرارگیری:**
این تغییرات را باید در تابع `postImageMultipart` که در بخش 1 (Class Globals) قرار دادید، اعمال کنید.

**کد جاوا (مثال برای `clear`):**
```java
// در تابع postImageMultipart، قبل از بخش فایل (// بخش فایل)
// اگر می‌خواهید چت را پاک کنید:
writeField.accept("clear", "true");
```

**تغییر مدل:**
مقدار `DEFAULT_MODEL` را در بخش 1 تغییر دهید، یا هنگام فراخوانی `postImageMultipart` در بخش 4، پارامتر `model` را با مقدار دلخواه خود جایگزین کنید.

---

## 6) نکات اشکال‌زدایی

اگر با مشکل مواجه شدید، این نکات می‌توانند به شما کمک کنند:

- **پاسخ خالی از API:**
  - **محل قرارگیری:** در تابع `postImageMultipart` (بخش 1)، بعد از خط `int code = conn.getResponseCode();`.
  - **کد جاوا:**
  ```java
  SketchwareUtil.showMessage(getApplicationContext(), "HTTP Status Code: " + code);
  ```
  این کد وضعیت HTTP را به شما نشان می‌دهد که می‌تواند در تشخیص مشکل کمک کند.

- **خطای «Permission denied» هنگام باز کردن `InputStream`:**
  - مطمئن شوید که مجوزهای `READ_MEDIA_IMAGES` یا `READ_EXTERNAL_STORAGE` (بخش 2) به درستی درخواست و اعطا شده‌اند.

- **مشکلات شبکه (اینترنت ضعیف یا قطع شدن):**
  - می‌توانید `timeout` برای اتصال و خواندن تنظیم کنید.
  - **محل قرارگیری:** در تابع `postImageMultipart` (بخش 1)، بعد از خط `java.net.HttpURLConnection conn = (java.net.HttpURLConnection) url.openConnection();`.
  - **کد جاوا:**
  ```java
  conn.setConnectTimeout(20000); // 20 ثانیه برای اتصال
  conn.setReadTimeout(60000);    // 60 ثانیه برای خواندن داده
  ```

---

## 7) نمایش پاسخ در همان ساختار فعلی شما

کدی که در بخش 4 ارائه شد، مستقیماً `list` (لیست داده‌های `ListView` شما) را به‌روزرسانی می‌کند و نیازی به بلوک‌های `RequestNetwork.receive` برای این عملیات آپلود خاص نیست. این روش با ساختار نمایش چت شما سازگار است.

---

## 8) خلاصه‌ی نهایی محل قرارگیری کدها

برای جمع‌بندی، کدهای جاوا را به شرح زیر در Sketchware Pro قرار دهید:

- **Class Globals (منوی سه‌نقطه → افزودن سورس مستقیم → Class Globals):**
  - کدهای بخش 1 (متغیرهای سراسری و توابع کمکی).

- **رویداد `onCreate` یا `initializeLogic` اکتیویتی اصلی (تب Event → onCreate/initializeLogic → بلوک سبز Add source directly):**
  - کدهای بخش 2 (درخواست مجوزها).

- **رویداد `onClick` دکمه/ImageView انتخاب فایل (تب View → imageview10 → onClick → بلوک سبز Add source directly):**
  - کدهای بخش 3 (فراخوانی FilePicker).

- **رویداد `onFilesPicked` کامپوننت FilePicker (تب Component → mahdi → onFilesPicked → بلوک سبز Add source directly):**
  - کدهای بخش 4 (پردازش فایل انتخاب شده و ارسال به API).

با انجام این مراحل، FilePicker شما به درستی به API متصل شده و تصاویر را ارسال خواهد کرد. اگر سوال دیگری داشتید، بپرسید.


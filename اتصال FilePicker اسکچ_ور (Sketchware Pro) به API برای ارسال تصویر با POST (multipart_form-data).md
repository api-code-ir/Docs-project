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

- Component ها:
  - FilePicker با فیلتر `image/*` (در تصاویر شما با نام `mahdi` وجود دارد)
  - (اختیاری) RequestNetwork برای درخواست‌های قبلی – اما برای آپلود فایل از کد سفارشی زیر استفاده می‌کنیم.
- مجوزها:
  - «Internet» فعال باشد (به‌صورت پیش‌فرض با RequestNetwork فعال است؛ اگر نبود از Project → Permissions فعال کنید).
  - برای اندروید 13 به بالا: `READ_MEDIA_IMAGES`
  - برای اندروید 12 و پایین: `READ_EXTERNAL_STORAGE`

کد درخواست مجوز را پایین‌تر گذاشته‌ام.

---

## 1) متغیرهای سراسری و توابع کمکی (Class Globals)

در Sketchware Pro مسیر زیر را باز کنید:

- منوی سه‌نقطه → «افزودن سورس مستقیم» (Add source directly) → قسمت «Class Globals»
- کد زیر را عیناً قرار دهید:

```java
// ==== Config API ====
private static final String API_URL = "https://api2.api-code.ir/gpt-save-v2/"; // انتهای / را نگه دارید
private static final String DEFAULT_MODEL = "deepseek"; // یا "openai" مطابق نیاز شما

// نخ برای کارهای شبکه
private java.util.concurrent.ExecutorService executor = java.util.concurrent.Executors.newSingleThreadExecutor();

// نام فایل را از URI استخراج می‌کند
private String getFileNameFromUri(android.net.Uri uri) {
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
private String getMimeFromUri(android.net.Uri uri) {
    String mime = getContentResolver().getType(uri);
    if (mime == null) mime = "image/jpeg"; // مقدار پیش‌فرض
    return mime;
}

// کپی داده از InputStream به OutputStream
private void copyStream(java.io.InputStream in, java.io.OutputStream out) throws java.io.IOException {
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
private String postImageMultipart(android.net.Uri imageUri, String userId, String prompt, String model) throws Exception {
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
```

---

## 2) درخواست مجوز خواندن تصاویر (onCreate یا initializeLogic)

در «Add source directly» بخش `initializeLogic()` یا در رویداد `onCreate` اکتیویتی اصلی این کد را قرار دهید:

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

و در «Add source directly» بخش `onRequestPermissionsResult` هم (اختیاری) پیام موفق/ناموفق بگذارید.

---

## 3) رویداد کلیک روی آیکن تصویر (imageview10 → onClick)

شما قبلاً این کار را انجام داده‌اید: `FilePicker: mahdi → choose file`.

اگر می‌خواهید با کد جاوا صدا بزنید، در رویداد `imageview10_onClick` این یک خط را (در صورت نیاز) اضافه کنید:

```java
mahdi.pickFiles(); // اگر از بلوک خود FilePicker استفاده نمی‌کنید
```

---

## 4) دریافت فایل‌های انتخاب‌شده (FilePicker → onFilesPicked)

به رویداد `mahdi_onFilesPicked(ArrayList<String> filePath)` بروید و حالت «Add source directly inside this event» را باز کنید و کل کد زیر را جایگزین/اضافه کنید:

```java
// filePath: لیست رشته‌هایی که توسط FilePicker برمی‌گردد
if (filePath == null || filePath.size() == 0) {
    SketchwareUtil.showMessage(getApplicationContext(), "هیچ فایلی انتخاب نشد");
    return;
}

String first = filePath.get(0);
android.net.Uri uri = first.startsWith("content://") ? android.net.Uri.parse(first) : android.net.Uri.fromFile(new java.io.File(first));

// مقداردهی به پارامترها
String userId = "api_codeapi_gpt22"; // همان که در پروژه‌تان استفاده کرده‌اید
String promptText = edittext1.getText().toString(); // اگر EditText دارید
String model = DEFAULT_MODEL; // یا هر مدلی که می‌خواهید

// (اختیاری) نمایش پیام کاربر در لیست قبل از ارسال
java.util.HashMap<String, Object> you = new java.util.HashMap<>();
you.put("Result", promptText.length() > 0 ? promptText : "[ارسال تصویر]");
you.put("key", "you");
list.add(you);
_refresh();

// ارسال در نخ جداگانه
executor.execute(() -> {
    try {
        String resp = postImageMultipart(uri, userId, promptText, model);
        runOnUiThread(() -> {
            try {
                org.json.JSONObject obj = new org.json.JSONObject(resp);
                String result = obj.optString("Result", resp);

                java.util.HashMap<String, Object> server = new java.util.HashMap<>();
                server.put("Result", result);
                server.put("key", "bot");
                list.add(server);
                _refresh();
                listview1.smoothScrollToPosition((int) (list.size()));
                linear11.setVisibility(android.view.View.GONE);
            } catch (Exception je) {
                SketchwareUtil.showMessage(getApplicationContext(), "JSON خطا: " + je.getMessage());
            }
        });
    } catch (Exception e) {
        runOnUiThread(() -> SketchwareUtil.showMessage(getApplicationContext(), "ارسال ناموفق: " + e.getMessage()));
    }
});
```

این کد:
- URI تصویر را (چه `content://` و چه `file://`) پشتیبانی می‌کند.
- درخواست را به صورت multipart می‌سازد و به API ارسال می‌کند.
- پاسخ JSON را خوانده و مقدار `Result` را مثل قبل داخل `list` اضافه می‌کند تا در `ListView` نمایش داده شود.

---

## 5) اگر می‌خواهید پارامترهای بیشتری بفرستید

- پاکسازی چت: `writeField.accept("clear", "true");` را قبل از بخش فایل در تابع `postImageMultipart` اضافه کنید.
- تغییر مدل: مقدار `DEFAULT_MODEL` را عوض کنید یا هنگام فراخوانی پارامتر `model` را مقداردهی کنید.

---

## 6) نکات اشکال‌زدایی

- اگر پاسخ خالی بود، لاگ وضعیت HTTP را چاپ کنید:
  - بعد از `int code = conn.getResponseCode();` یک `SketchwareUtil.showMessage(..., "HTTP " + code);` موقت اضافه کنید.
- اگر FilePicker مسیر فیزیکی برگرداند و باز کردن `InputStream` با خطای «Permission denied» مواجه شد، اطمینان بگیرید مجوزهای بخش 2 صادر شده باشد.
- برای فایل‌های بزرگ، اینترنت ضعیف یا قطع‌شدن اتصال ممکن است رخ دهد؛ می‌توانید timeout تنظیم کنید:

```java
conn.setConnectTimeout(20000); // 20s
conn.setReadTimeout(60000);    // 60s
```

(دو خط بالا را بعد از `openConnection()` اضافه کنید.)

---

## 7) نمایش پاسخ در همان ساختار فعلی شما

کدی که در بخش 4 گذاشته شد مستقیماً `list` را به‌روزرسانی می‌کند (مشابه بلوک‌های رویداد `receive_onResponse` که در تصاویر دیده می‌شود). بنابراین نیازی به `RequestNetwork.receive` برای این آپلود خاص نیست. اگر مایل باشید می‌توانم همین منطق را به صورت «More Block» مجزا هم آماده کنم تا در چند صفحه استفاده شود.

---

## 8) نسخه‌ی بسیار خلاصه‌ی رویدادها

- imageview10_onClick → `mahdi.pickFiles();` (یا همان بلوک انتخاب فایل شما)
- mahdi_onFilesPicked → کد بخش 4
- Class Globals → کدهای بخش 1
- initializeLogic/onCreate → کد بخش 2 (مجوزها)

تمام. با این تغییرات، ارسال تصویر از FilePicker به API انجام می‌شود و پاسخ در ListView نمایش داده خواهد شد.



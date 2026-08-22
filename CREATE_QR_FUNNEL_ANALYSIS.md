# Phân tích Create QR success rate 21.47% — Báo cáo & Phương án

**Branch:** `DinhPV/v1.6x_optimize_qr`
**Ngày:** 2026-08-22
**Số liệu đầu vào:** Create success rate 21.47% theo lượt (15,246 → 3,274)
**Mẫu số (đã xác nhận với PO):** đếm mỗi lần user vào màn hình Create QR Code

---

## 0. TL;DR

**Con số 21.47% KHÔNG phải là "79% lượt tạo QR bị lỗi kỹ thuật".** Với mẫu số = "vào màn Create", đây là tỷ lệ *browse → convert*, và phần lớn hao hụt là hành vi bình thường (user mở tab Create, xem grid 20+ loại QR, rồi thoát).

Tuy nhiên có 3 vấn đề thật:

1. **Metric hiện tại không đo được điều nó tưởng đang đo.** Mẫu số thiếu lượt vào bằng cách *vuốt* ViewPager; tử số bị đếm trùng (log 2 chỗ) và bị trộn thêm 3 luồng không đi qua tab Create. Vì vậy 21.47% không dùng được để tìm điểm rơi. → **Mục 1**
2. **Có bug làm luồng tạo QR thất bại thật** — quan trọng nhất: một lỗi đảo ngược điều kiện 1 ký tự trong module `qr_generator` đang **hạ trần nội dung QR từ 2953 byte xuống 1663 byte**, khiến nội dung dài bị báo "data too long" oan. → **Mục 2**
3. **Các thất bại ở bước cuối hiện đang vô hình trong analytics** — event `ui_create_result` được log ở `onCreate`, *trước* khi biết bitmap có render được hay không. Nếu render fail, app toast lỗi rồi tự back ra, nhưng funnel vẫn ghi nhận "thành công". → **Mục 2.3**

Tin tốt: **có thể tách funnel thành 4 bước ngay hôm nay bằng event đã có sẵn, không cần sửa code, không cần chờ release.** → **Mục 4, Giai đoạn 0**

> ⚠️ **Về tính kiểm chứng:** mọi khẳng định trong Mục 2 đều có `file:line` và/hoặc đối chiếu source gốc (ZXing 3.5.3 sources jar, upstream `alexzhirkevich/custom-qr-generator`). Các giả thuyết chưa kiểm chứng được tách riêng ở **Mục 3** và **không** được đưa vào phương án bắt buộc.

---

## 1. Metric: 21.47% đang đo cái gì

### 1.1 Mẫu số (15,246)

```kotlin
// MainActivity.kt:329 — trong openTab()
R.id.navigation_create -> { Tracker.log(Screens.CREATE, Events.ui_create); TabIndex.CREATE_QR.index }
```

`openTab()` chỉ được gọi từ `bottomNavigationView.setOnItemSelectedListener` (`MainActivity.kt:270-282`).

**Lỗ hổng:** khi user **vuốt** ViewPager sang tab Create, `onPageSelected` chỉ làm:

```kotlin
// MainActivity.kt:524
binding!!.bottomNavigationView.menu.findItem(getTabIdByPosition(position)).isChecked = true
```

`MenuItem.isChecked = true` **không** trigger `OnItemSelectedListener`. → **Mọi lượt vào tab Create bằng vuốt đều không được đếm vào mẫu số.** Số lượt vuốt có thể đo được qua `click_btn_ev` = `SwipeTabCreate` (`MainActivity.kt:530`) — nên so sánh 2 con số này để biết mẫu số hụt bao nhiêu.

### 1.2 Tử số (3,274)

`Tracker.log(Screens.CREATE_RESULT, Events.ui_create_result)` được gọi ở **hai** chỗ trong cùng một fragment:

```kotlin
// BaseCreateQRResultFragment.kt:133 — trong onCreate()
Tracker.log(Screens.CREATE_RESULT, Events.ui_create_result)

// BaseCreateQRResultFragment.kt:628-632 — trong onResume()
if (isFirstResume) { isFirstResume = false }
else { Tracker.log(Screens.CREATE_RESULT, Events.ui_create_result) }
```

→ Mỗi lần user quay lại màn kết quả (back từ Edit QR, từ Share, từ Print, sau khi app về foreground) đều **cộng thêm 1** vào tử số.

Ngoài ra `BaseCreateQRResultFragment` còn được mở từ 3 luồng **không đi qua tab Create**:

| Nguồn | File:line | showFrom |
|---|---|---|
| Tạo QR mới từ kết quả scan / item History | `BaseScanResultFragment.kt:1295` | (tuỳ luồng) |
| Nhận data share từ app khác | `GetDataScanActivity.kt:490` | `GetDataScan` |
| Tạo QR từ clipboard | `ListCreateQRCodeFragment.kt:102` | `ListCreateQRCode_Clipboard` |

### 1.3 Kết luận về metric

Tử số bị **thổi lên** (đếm trùng + nguồn ngoài), mẫu số bị **hụt** (thiếu lượt vuốt). Hai sai lệch này ngược chiều nhau nên không biết 21.47% lệch về phía nào, và quan trọng hơn: **nó không cho biết user rơi ở bước nào**.

---

## 2. Các lỗi kỹ thuật đã kiểm chứng

### 2.1 🔴 P0 — Lỗi đảo cực điều kiện làm trần nội dung QR giảm 44%

**Vị trí:** `qr_generator/src/main/java/com/qr/generator/vector/QrCodeDrawable.kt:970`

```kotlin
// FORK (hiện tại) — SAI
val hasLogo = size > Float.MIN_VALUE && logo.drawable == null ||
        logo.padding != QrVectorLogoPadding.Empty
```

Upstream `alexzhirkevich/custom-qr-generator` (branch `main`) dùng:

```kotlin
val hasLogo = size > Float.MIN_VALUE && logo.drawable != EmptyDrawable ||
        logo.padding != QrVectorLogoPadding.Empty
```

Khi port sang fork, team đã bỏ hẳn sentinel `EmptyDrawable` và chuyển `drawable` sang nullable `Drawable?` — nhưng viết `== null` thay vì `!= null`, **đảo ngược ý nghĩa của biến `hasLogo`**.

**Hệ quả với QR không logo** (mặc định `QrVectorLogo(drawable = null, size = 0.2f, padding = Empty)`, `QrShape.Default.shapeSizeIncrease = 1f`):

- `size = 0.2f * (1 + 0f) * 1f = 0.2f`
- `hasLogo = (0.2f > MIN_VALUE && null == null) || false` = **`true`** (sai — không có logo!)
- `fit(hasLogo = true, logoSize = 0.2f)` → nhánh `logoSize in .2 .. .3 && lvl < Q` → **`MediumHigh` = `ErrorCorrectionLevel.Q`**

Lẽ ra phải là `!hasLogo` → `Low` = `L`.

**Trần dung lượng byte-mode ở QR version 40** (dẫn xuất từ hằng số ZXing 3.5.3, `Version.java:565-573`; tổng codewords cả 4 mức đều = 3706, khớp ISO/IEC 18004 Table 7):

| EC level | Data codewords | Max payload (byte mode) |
|---|---|---|
| **L** | 2956 | **2953 byte** |
| M | 2334 | 2331 byte |
| **Q** ← đang dùng | 1666 | **1663 byte** |
| H | 1276 | 1273 byte |

→ **App đang từ chối nội dung từ 1664 byte trở lên với thông báo "data too long", trong khi chuẩn cho phép tới 2953 byte.** Sửa 1 ký tự nâng trần nội dung **+77.6%**.

Ảnh hưởng cụ thể tới các loại QR nội dung dài: **Text dài, VCard có note/địa chỉ đầy đủ, Event có description, Email có body**. Đây chính là nhóm dễ chạm trần nhất.

**Tác động phụ:** cùng một payload, EC=Q cần matrix lớn hơn EC=L → nhiều module hơn trên cùng số pixel → module nhỏ hơn → **khó quét hơn**, đặc biệt khi in ra hoặc bị resize xuống.

**Lưu ý để không sửa sai:** nhánh **có logo** của app *không* bị lỗi này. `QRCodeGenerateHelper.encodeQRCodeToBitmap` (`:199-209`) ghi tường minh:

```kotlin
mLogo?.let {
    logo {
        size = .25f
        padding = QrVectorLogoPadding.Natural(.008f)
        errorCorrectionLevel = QrErrorCorrectionLevel.High   // ← set EC toàn cục
        ...
    }
}
```

`QrVectorLogoBuilderScope` **không** có property `errorCorrectionLevel` (xem `dsl/QrVectorLogoBuilderScope.kt`), nên Kotlin resolve nó lên receiver ngoài — `QrVectorOptionsBuilderScope.errorCorrectionLevel` (`dsl/QrVectorOptionsBuilderScope.kt:21`). Tức khi có logo, EC = H tường minh, không đi qua `Auto`/`fit()`. Vì vậy chiều nguy hiểm của bug (logo che 20% nhưng EC rơi về L → QR không quét được) **không xảy ra ở app này**. Nhưng nó là bẫy chờ: chỉ cần ai đó thêm một call site dùng logo mà để EC = `Auto` là ăn ngay.

**Fix:**
```kotlin
val hasLogo = size > Float.MIN_VALUE && logo.drawable != null ||
        logo.padding != QrVectorLogoPadding.Empty
```

> Ghi chú về upstream: bug này **chưa từng được report** lên repo gốc (đã soát toàn bộ 16 issue của repo). Upstream cũng đã ngừng phát triển code từ **2023-10** (5 commit gần nhất chỉ sửa README); tác giả chuyển sang [`qrose`](https://github.com/alexzhirkevich/qrose) cho Compose Multiplatform — **không phù hợp** với app này (View system). Kết luận: fork nội bộ là "maintainer" duy nhất của code path này, **phải tự fix, không chờ upstream**. License upstream là **MIT** (không phải Apache 2.0 như thường giả định).

---

### 2.2 🔴 P0 — Validate bằng cách render cả bitmap rồi bỏ đi

**Pattern này lặp ở 20+ ViewModel.** Ví dụ `CreateTextViewModel.kt:26-37`:

```kotlin
val content = QRCodeEncodeUtils.getRawText(mText)
val bitmap = try {
    QRCodeGenerateHelper.getInstance().setContent(content).generate()  // render full bitmap
} catch (ex: Exception) { ex.printStackTrace(); null }

if (bitmap == null) {                                    // ...chỉ để check null
    data.emit(Resource.error(Constants.Validate.DATA_LONG))
    return@launch
}
makeQREncode(mText, BarcodeFormat.QR_CODE, mText)        // ...rồi bỏ bitmap luôn
```

Cùng pattern tại: `CreateUrlViewModel:36`, `CreateWifiViewModel:38`, `CreateEmailViewModel:32`, `CreateEventViewModel:66`, `CreateContactViewModel:43`, `CreateVCardViewModel:48`, `CreatePhoneViewModel:33`, `CreateLocationViewModel:37`, `CreateMessengerViewModel:29`, `CreateFacebookViewModel:34`, `CreateInstagramViewModel:34`, `CreateTwitterViewModel:35`, `CreateViberViewModel:19`, `CreateWhatsappViewModel:19`, `CreateSpotifyViewModel:46`, `CreatePaypalViewModel:29,54`, `CreateYoutubeViewModel:33,60,87`, `CreateAppViewModel:119`, `ListCreateQRCodeFragment:237`.

**Chi phí bộ nhớ mỗi lần `generate()`:**

- Kích thước: `QRUtils.getWidthHeightBarcode()` (`QRUtils.kt:1662-1724`) = `screenWidth − qrcode_total_padding`, cap 2048 → thực tế ~900–1400 px.
- Trong lib, `QrCodeDrawableImpl.resize()` (`QrCodeDrawable.kt:945`) alloc **1 bitmap** `w×h` `ARGB_8888`.
- `QRCodeGenerateHelper.kt:224` gọi `.toBitmap(width, height)`. Vì `QrCodeDrawableImpl` không phải `BitmapDrawable`, androidx alloc **thêm 1 bitmap** `w×h` nữa.
- Bitmap cũ **không được `recycle()`** trước khi gán mới; `setBounds()` → `resize()` chạy lại **mỗi lần layout**.

→ ~2 bitmap mỗi `generate()`. Ở 1200px: `1200² × 4B ≈ 5.8 MB`/bitmap. Một lượt tạo QR gọi `generate()` **2 lần** (validate ở ViewModel + render ở màn kết quả) → **~23 MB churn cho một lần bấm nút Create**.

**Và OOM không được xử lý:**

```kotlin
// QRCodeGenerateHelper.kt:126, 132
} catch (ex: WriterException) { ... }
} catch (ex: Exception) { ... }
```

`OutOfMemoryError` extends `Error`, **không** phải `Exception` → lọt qua cả `generate()` lẫn `catch (ex: Exception)` ở ViewModel → exception không bắt được trong coroutine `Dispatchers.Default` → **crash app**. User bấm Create, app chết, không có event nào ghi lại.

**Đúng cách:** check capacity **thuần số học**, không render gì. `Encoder.willFit()` trong ZXing 3.5.3 là `package-private` nên không gọi trực tiếp được từ `com.tohsoft.*`, nhưng tính lại được bằng public API:

```kotlin
val v = Version.getVersionForNumber(40)
val ecb = v.getECBlocksForLevel(ecLevel)
val dataCodewords = v.totalCodewords - ecb.totalECCodewords
val header = 4 + Mode.BYTE.getCharacterCountBits(v) + if (needsEci) 12 else 0
val maxBytes = (dataCodewords * 8 - header) / 8
```

---

### 2.3 🔴 P0 — Thất bại ở màn kết quả bị analytics ghi nhận là thành công

`BaseCreateQRResultFragment.fillDataToView()` (`:519-542`):

```kotlin
if (bitmap == null) { bitmap = genBitmapFromContent() }   // chạy trên MAIN THREAD
if (bitmap != null) {
    ivEditQR.setImageBitmap(bitmap)
} else {
    showToast(requireActivity(), getString(R.string.msg_created_failed))
    handleBackPressed()      // đẩy user ra khỏi màn hình
    return
}
```

Hai vấn đề:

1. `genBitmapFromContent()` là **synchronous trên main thread**. Với QR version lớn + logo dùng `QrVectorLogoPadding.Natural(.008f)` (`QRCodeGenerateHelper.kt:202`), lib chạy `applyNaturalLogo()` — vòng lặp O(n²) gọi `Path.and()` cho từng pixel trong vùng logo. Đây là **ANR risk** đã được chính doc của lib cảnh báo (*"WARNING: this padding can cause performance issues for qr codes with large amount out data"*).
2. `Tracker.log(Screens.CREATE_RESULT, Events.ui_create_result)` đã được bắn ở `onCreate` (`:133`) — **trước** khi biết bitmap render được hay không. Nên khi nhánh `else` chạy (toast lỗi + back ra), funnel vẫn tính là **thành công**.

→ Tỷ lệ thất bại thật ở bước cuối hiện **không thể đo được**.

---

### 2.4 🟠 P1 — `QRCodeGenerateHelper` là singleton có state mutable, bị gọi đa luồng

```kotlin
// QRCodeGenerateHelper.kt:30-42
private object Holder { val INSTANCE = QRCodeGenerateHelper() }
private var mLogo: Bitmap? = null
private var mForegroundColor: Int? = null
private var mBackgroundColor: Int? = null
private var mContent: String? = null
```

Được gọi đồng thời từ:
- `Dispatchers.Default` — 20+ ViewModel (validate)
- **Main thread** — `QREditImageView.genBitmap()` (`:276, 293, 316, 332`) trong Glide callback, và `BaseCreateQRResultFragment.fillDataToView()`

`QRUtils.genBitmapFromContent()` (`QRUtils.kt:2102-2116`) set cả `logo`/`foregroundColor`/`backgroundColor` lên singleton. `reset()` (`:83-87`) **không** clear `mContent`.

→ Race thật: hai luồng interleave giữa `setContent()` và `generate()` sẽ render QR với **nội dung hoặc màu của luồng khác**. Trường hợp xấu nhất: `mForegroundColor == mBackgroundColor` → QR trắng hoàn toàn.

**Fix:** chuyển sang request object immutable, dựng per-call, không state chia sẻ.

---

### 2.5 🟠 P1 — `setContent()` biến đổi âm thầm nội dung người dùng

```kotlin
// QRCodeGenerateHelper.kt:63-70
fun setContent(content: String?): QRCodeGenerateHelper {
    mContent = try { content?.unescape() } catch (ex: Exception) { content }
    return this
}
```

`String.unescape()` (`extension/String.kt:138-193`) diễn giải escape sequence kiểu Java: `\n`, `\r`, `\t`, `\b`, octal, `\x`, `\u`.

Ví dụ user nhập `C:\temp\new` → QR encode `C:` + TAB + `emp` + LF + `ew`. Nhưng entity lưu vào DB và text hiển thị trên màn kết quả là **nội dung gốc**. → Quét lại QR của chính mình ra chữ khác chữ đang hiện trên màn hình.

Ảnh hưởng hẹp (chỉ nội dung chứa `\`) nhưng là data corruption âm thầm. Nội dung không nên bị transform trước khi encode.

---

### 2.6 🟠 P1 — Quiet zone theo tỷ lệ cố định: dưới chuẩn ở v1, thừa 47% diện tích ở version lớn

Đã kiểm chứng: `Encoder.encode(...).matrix` **không có quiet zone** — `Version.getDimensionForVersion()` trả đúng `17 + 4v` (v1 = 21, v40 = 177). Hằng `QUIET_ZONE_SIZE = 4` chỉ tồn tại trong `QRCodeWriter`, mà lib này **không dùng** (nó gọi `Encoder` trực tiếp).

Nên margin duy nhất là `options.padding = Constants.PADDING_QR = 0.27f` (`Constants.kt:14`), với `size = min(w,h) * (1 - padding)` và code được căn giữa.

Quy đổi ra số module: `q = n × p / (2(1−p))` = `n × 0.1849`

| Version | n (module) | Quiet zone mỗi bên |
|---|---|---|
| v1 | 21 | **3.88** ❌ (ISO yêu cầu ≥ 4) |
| v2 | 25 | 4.62 |
| v4 | 33 | 6.10 |
| v10 | 57 | 10.54 |
| v20 | 97 | 17.94 |
| v40 | 177 | **32.73** (thừa 8×) |

Vùng code chỉ chiếm `(1 − 0.27)² = 53%` ảnh → **mất 47% diện tích bitmap**, làm module nhỏ hơn mức cần thiết.

**Fix:** tính padding từ số module mong muốn thay vì hard-code tỷ lệ:
```kotlin
padding = 2f * quietZoneModules / (codeMatrixSize + 2f * quietZoneModules)   // q = 4
```

Lưu ý: dùng `n` **sau** khi apply `QrShape` (`shapeSizeIncrease` có thể phình matrix), không phải `n` của matrix ZXing.

---

### 2.7 🟡 P2 — Nhóm lỗi payload & nhất quán

| # | Vấn đề | Vị trí |
|---|---|---|
| a | `getRawTextWifi()` **không escape** `\` `;` `:` `,` theo format `WIFI:`. Password chứa `;` hoặc `:` → payload sai, máy khác quét ra sai | `QRCodeEncodeUtils.kt:46-53` |
| b | Từ chối password Wi-Fi non-ASCII: `Regex("\\A[\\x20-\\x7e]+\\Z")` | `CreateWifiViewModel.kt:26` |
| c | `Charset.defaultCharset()` — trên Android luôn UTF-8 nên ECI 26 *được* chèn đúng cho tiếng Việt (verified `ZXing Encoder.java:123-128`: ECI chỉ append khi `mode == BYTE && hasEncodingHint`). Hành vi đúng, nhưng nên khai báo tường minh `StandardCharsets.UTF_8` thay vì phụ thuộc default. Chi phí ECI = 12 bit | `QRCodeGenerateHelper.kt:223` |
| d | `getRawTextUrl()` prepend `http://` thay vì `https://` | `QRCodeEncodeUtils.kt:37-44` |
| e | `mErrorCorrectionLevel = ErrorCorrectionLevel.M` là **dead code** — không được truyền vào lib ở đâu (sót lại từ code path ZXing đã comment) | `QRCodeGenerateHelper.kt:38` |
| f | `CreateAppFragment.getScreenName() = "App"` — lệch quy ước PascalCase `Create*`, phá funnel lọc theo prefix screen name | `CreateAppFragment.kt:34` |
| g | `isShowInterAds` được đọc vào field nhưng **không dùng ở đâu** trong result fragment — dead flag | `BaseCreateQRResultFragment.kt:102, 131` |
| h | `MultiClickUtils` là singleton dùng chung `mLastClickTime` cho ~50 click handler toàn app. Getter `isAvailableClick` dùng `System.currentTimeMillis()`, còn overload `isAvailableClick(timeDelay)` dùng `SystemClock.elapsedRealtime()` — **trộn 2 hệ thời gian vào cùng 1 field**. Hiện đang latent (overload chỉ còn 1 call site đã comment ở `PopupUtils.kt:204`), nhưng nếu ai đó dùng lại overload thì button đó **chết vĩnh viễn** | `MultiClickUtils.kt:13-35` |

---

### 2.8 🟠 P1 — Ba code path generate độc lập, quiet zone và EC không nhất quán

App có **ba** đường sinh ảnh mã, mỗi đường tự quyết định margin và EC riêng:

| # | Path | Dùng cho | Margin / quiet zone | EC level |
|---|---|---|---|---|
| 1 | `QRCodeGenerateHelper` → fork `custom-qr-generator` | Tất cả loại QR | `Constants.PADDING_QR = 0.27f` (tỷ lệ ảnh) | `Auto` → **Q** (do bug 2.1), hoặc `High` khi có logo |
| 2 | `QRUtils.getBitmapWithZXingCpp()` (`:1783`) → `ZxingCpp.encodeAsBitmap(..., -1, 2, ...)` rồi `drawRoundWhite()` thêm viền `Constants.PADDING_BARCODE = 0.3f` | Barcode 1D/2D, product | magic number `2` + 0.3f tỷ lệ | magic number `-1` |
| 3 | `QRUtils.genBitmapBarcode()` (`:1647`) → ZXing `BarcodeEncoder` | Fallback khi path 2 trả null | mặc định của `MultiFormatWriter` | mặc định `L` |

Ba vấn đề:

- **Magic number không giải thích được**: `ZxingCpp.encodeAsBitmap(content, format, w, h, -1, 2, fg, bg)` — cần xác định `-1` và `2` là tham số gì (margin? ecLevel?). Nếu `2` là margin (module) thì **dưới chuẩn ISO 4 module**, và phần bù lại đến từ `drawRoundWhite` — một cách gián tiếp, khó kiểm soát.
- **`drawRoundWhite()` (`:1801-1819`) alloc thêm 1 bitmap full-size** và không recycle bitmap nguồn → path barcode có 2 bitmap mỗi lần generate, chồng lên vấn đề ở Mục 2.2.
- Sửa quiet zone/EC phải sửa **cả ba chỗ**, dễ bỏ sót.

**Điểm tích cực đáng học:** `CreateBarcodeViewModel.validateBarCode()` (`:43-85`) làm **đúng thứ tự** — gọi `barCodeEngine.isValid(query)` (validator riêng theo từng format, `IBarCodeType`) **trước**, rồi mới encode, và message lỗi phân biệt được 2 ca (`txt_error_create_barcode_invalid` vs `txt_error_create_barcode`). Đây chính là pattern cần nhân bản cho 20+ ViewModel của QR (Mục 2.2 + 1.4).

*Lỗi nhỏ cùng chỗ:* validate dùng `query` (chưa trim) nhưng entity lưu `content = getRawText(query)` (đã trim) — với format có độ dài cố định (EAN-13 = 13 số) thì hai chuỗi này có thể cho kết quả validate khác nhau.

---

## 3. Giả thuyết CHƯA kiểm chứng (không đưa vào phương án bắt buộc)

Những điểm dưới đây đọc code thì thấy đáng ngờ nhưng **chưa có bằng chứng** — cần đo trên máy thật hoặc lấy số liệu trước khi hành động. Ghi ra để không bị quên, **không** để làm căn cứ sửa code.

**H1 — Nút "Create" có thể nằm ngoài vùng nhìn khi bàn phím bung.**

`fragment_base_create_qr.xml`: root là `LinearLayoutCompat` dọc gồm AppBar → `NestedScrollView` (weight 1) → ad container cố định ở đáy (`:162`). `btnCreateQR` (`:101`) nằm **bên trong** scroll view, **phía dưới** các input field. Đồng thời form tự bung keyboard khi vào (`BaseCreateQRFragment.onResume():406-414`).

Với form nhiều field (VCard, Event, Contact, Wifi, Email) + keyboard + native/banner ad ở đáy, vùng scroll còn lại rất hẹp → user phải scroll xuống mới thấy nút Create.

*Cách kiểm chứng:* so `click_btn_ev` = `CreateQr` giữa các loại QR ít field (Url, Text, Phone) vs nhiều field (VCard, Event, Contact). Nếu nhóm nhiều field tụt rõ rệt → giả thuyết đúng. App **đã có** hạ tầng A/B: remote config `create_qr_layout_34` chọn 1 trong 4 layout (`BaseCreateQRFragment.kt:183-188`).

**H2 — Tỷ lệ crash/OOM ở màn Create.** Mục 2.2 chỉ ra đường dẫn crash, nhưng chưa có số Crashlytics để biết nó chiếm bao nhiêu %.

**H3 — Phân bố độ dài nội dung.** Mục 2.1 nói trần 1663 byte thay vì 2953. Cần biết bao nhiêu % lượt tạo thực tế vượt 1663 byte để định lượng tác động.

---

## 4. Phương án

### Giai đoạn 0 — Dựng lại funnel (0 dòng code, làm được ngay)

Ba bước trung gian **đã có event sẵn**. Chỉ cần dựng lại dashboard:

| Bước | Event Firebase | Param | Vị trí log |
|---|---|---|---|
| 1. Vào tab Create | `CreateScreen` | `item_name` = `ui_create` | `MainActivity.kt:329` |
| 1b. Vào bằng vuốt | `click_btn_ev` | `click_btn_ev_name` = `SwipeTabCreate` | `MainActivity.kt:530` |
| 2. Chọn 1 loại → mở form | `CreateScreen` | `item_name` ∈ `website`/`wifi`/`text`/`vcard`/… | `ListCreateQRCodeFragment.kt:211` |
| 3. Bấm nút Create | `click_btn_ev` | `click_btn_ev_name` = `CreateQr` | `BaseCreateQRFragment.kt:494` |
| 4. Ra màn kết quả | `create_result_screen` | `item_name` = `ui_create_result` | `BaseCreateQRResultFragment.kt:133` |

Bước 3 chỉ log **sau** khi qua debounce (`MultiClickUtils`), nên nó là mẫu số sạch cho "thực sự có ý định tạo".

**Ba tỷ lệ cần đọc:**
- `2/1` — grid 20+ loại có dẫn được user vào form không (vấn đề discovery/IA)
- `3/2` — vào form rồi có bấm Create không (vấn đề form UX → H1)
- `4/3` — bấm Create rồi có ra kết quả không → **đây mới là "technical success rate" thật**

Breakdown thêm theo `item_name` (loại QR) để biết loại nào rơi nhiều nhất.

**Đầu ra:** biết chính xác nên đầu tư vào đâu, thay vì đoán.

---

### Giai đoạn 1 — Fix rủi ro thấp / giá trị cao

| # | Việc | File | Tiêu chí nghiệm thu |
|---|---|---|---|
| 1.1 | Sửa `== null` → `!= null` | `qr_generator/.../QrCodeDrawable.kt:970` | Text 2900 byte tạo được QR (trước đây fail ở >1663). QR không logo dùng EC = L; QR có logo vẫn EC = H |
| 1.2 | Thay validate-bằng-render bằng **capacity check số học**; encode **1 lần**, cache matrix/bitmap dùng cho cả màn kết quả | 20+ ViewModel + `QRCodeGenerateHelper` | Bấm Create alloc **0** bitmap thừa; thời gian từ bấm Create → thấy QR giảm rõ rệt |
| 1.3 | `catch (Throwable)` ở biên generate, + `CoroutineExceptionHandler`; OOM → thông báo lỗi cụ thể thay vì crash | `QRCodeGenerateHelper.generate()`, các ViewModel | Không còn crash path OOM ở màn Create |
| 1.4 | Tách message lỗi theo nguyên nhân thật (vượt capacity / ký tự không hỗ trợ / lỗi hệ thống) — hiện tất cả đều ra `error_data_long` | `Constants.Validate`, các `Create*Fragment` | User biết phải sửa gì |
| 1.5 | **Thêm event thất bại**: bắn khi validate fail và khi generate fail, kèm lý do + loại QR + độ dài nội dung. Đăng ký hằng số vào enum trong `toh-analytics` (`Event`/`ParameterName`) theo đúng quy ước CLAUDE.md | `toh-analytics`, `BaseCreateViewModel` | Bước `4/3` ở Giai đoạn 0 có thể phân rã thành từng nguyên nhân |
| 1.6 | Move `ui_create_result` từ `onCreate` sang **sau khi render bitmap thành công**; xoá bản log trùng ở `onResume` | `BaseCreateQRResultFragment.kt:133, 628-632` | Tử số = số QR **thực sự hiển thị được**, không đếm trùng |
| 1.7 | `CreateAppFragment.getScreenName()`: `"App"` → `"CreateApp"` | `CreateAppFragment.kt:34` | Funnel lọc theo prefix `Create*` không bỏ sót loại App |

> 1.6 sẽ **làm con số 21.47% tụt xuống** vì bỏ đếm trùng. Đây là điều mong muốn — cần thông báo trước cho team để không bị hiểu là "regression".

---

### Giai đoạn 2 — Thread & memory

| # | Việc | Tiêu chí nghiệm thu |
|---|---|---|
| 2.1 | Render bitmap **off main thread**, hiển thị placeholder trong lúc chờ | Không còn frame drop / ANR khi mở màn kết quả với QR version lớn |
| 2.2 | Bỏ singleton state của `QRCodeGenerateHelper` → request object immutable per-call | Chạy 100 lần generate song song từ nhiều luồng, decode ngược đều ra đúng nội dung |
| 2.3 | Tính kích thước bitmap theo **số module × pixel-per-module** (bội số nguyên) thay vì theo screen width; giới hạn theo `ActivityManager.getMemoryClass()`; `recycle()` bitmap cũ | Peak memory khi tạo QR giảm; không OOM trên máy RAM thấp |
| 2.4 | Cache theo `(width, height)` trong `resize()` để bỏ qua re-render khi bounds không đổi | `resize()` không chạy lại khi layout pass lặp |

---

### Giai đoạn 3 — Chuẩn hoá payload & output

| # | Việc | Tiêu chí nghiệm thu |
|---|---|---|
| 3.1 | Escape đúng `\` `;` `:` `,` trong `getRawTextWifi()`; nới validate password non-ASCII | QR Wi-Fi có password chứa `;` `:` kết nối được; password Unicode tạo được |
| 3.2 | Bỏ `unescape()` khỏi `setContent()` | Nội dung chứa `\` encode y nguyên; quét lại ra đúng chữ user nhập |
| 3.3 | `Charset.defaultCharset()` → `StandardCharsets.UTF_8` tường minh | Không đổi hành vi hiện tại, chỉ bỏ phụ thuộc ngầm |
| 3.4 | Quiet zone tính theo module (`q = 4`) thay vì tỷ lệ cố định `0.27f` | v1 đạt ≥ 4 module (hiện 3.88); v40 không còn thừa 32 module → module lớn hơn ~1.9× |
| 3.5 | Xoá dead code: `mErrorCorrectionLevel`, `isShowInterAds` ở result fragment | — |
| 3.6 | **Round-trip verify** ở debug/tester build: decode ngược QR vừa tạo bằng ZXing/ML Kit, assert khớp payload | Bộ test bao 20 loại QR × 4 EC level × nội dung ASCII/tiếng Việt/emoji đều pass |
| 3.7 | Hợp nhất 3 code path generate (Mục 2.8) về **một** entry point duy nhất, quiet zone và EC khai báo ở một chỗ; thay magic number `-1, 2` bằng hằng số có tên | Sửa quiet zone chỉ cần sửa 1 nơi; QR và barcode có quiet zone nhất quán ≥ 4 module |
| 3.8 | Nhân bản pattern `IBarCodeType.isValid()` của `CreateBarcodeViewModel` sang các ViewModel QR; thống nhất chuỗi dùng để validate và chuỗi lưu DB (`query` vs `getRawText(query)`) | Validate và encode luôn chạy trên cùng một chuỗi |
| 3.9 | Thống nhất version ZXing về một chỗ khai báo (`ext` ở root `build.gradle`); sửa CLAUDE.md ghi 3.5.3 → 3.5.4 | `app` và `qr_generator` compile + runtime cùng một version |

---

### Giai đoạn 4 — UX form (chỉ làm sau khi Giai đoạn 0 xác nhận H1)

Nếu số liệu cho thấy `3/2` thấp và tụt theo số field, thử qua remote config `create_qr_layout_34`:
- Ghim nút Create ra ngoài `NestedScrollView` (cố định trên ad container)
- Không tự bung keyboard với form nhiều field
- Ẩn ad khi keyboard đang mở

---

## 5. Phần nghiên cứu còn thiếu

Đã chạy 8 research agent song song; **2 hoàn thành**, 6 dừng giữa do hết session limit.

**Đã kiểm chứng xong:**
- ZXing 3.5.3: điều kiện append ECI, hành vi `CHARACTER_SET`, `QR_COMPACT` (và bẫy `QR_VERSION` bị ignore âm thầm), `willFit` là package-private, matrix không có quiet zone, bảng capacity byte-mode L/M/Q/H tại v40. *(Nguồn: sources jar chính thức của đúng version, giải nén từ Gradle cache)*
- Upstream `custom-qr-generator`: xác nhận bug đảo cực ở `fit()`, defaults của `QrVectorOptions`/`QrVectorLogo`, số bitmap alloc mỗi `resize()`, tình trạng repo (bỏ hoang từ 2023-10, MIT license)

**Đang chạy lại (6 hướng, song song):**

| # | Chủ đề | Câu hỏi cốt lõi |
|---|---|---|
| R1 | Module resolution & scannability | Pixel tối thiểu mỗi module cho màn hình / share / in; integer module scaling; khoan dung quiet zone của ZXing–zxing-cpp–ML Kit; ngưỡng contrast; anti-alias |
| R2 | Bitmap memory & OOM trên Android | `getMemoryClass()` budget và native heap từ API 26; `ARGB_8888` vs `RGB_565` vs `ALPHA_8` vs `HARDWARE` cho ảnh 2 màu; giới hạn `Canvas`; `recycle()` còn cần không; catch `OutOfMemoryError` có phải anti-pattern; hành vi `Error` chưa bắt trong coroutine |
| R3 | EC level, X-dimension, payment QR | Quy tắc chọn EC theo use case; đánh đổi EC ↔ module count (khi nào EC cao lại khó quét hơn); logo che bao nhiêu % thì cần EC nào; X-dimension GS1 khi in; ràng buộc EMVCo/VietQR; format payload chuẩn (`WIFI:` escape, MECARD vs vCard, MATMSG vs mailto); Micro QR/rMQR |
| R4 | Engine encode landscape 2025–2026 | Trạng thái ZXing, diff 3.5.3→3.5.4; zxing-cpp có encode API gì; ML Kit có encoder không (chốt dứt điểm); alternatives; **khuyến nghị: giữ fork / tách ZXing-matrix + renderer tự viết / đổi lib** |
| R5 | ECI & Unicode tiếng Việt | `Charset.defaultCharset()` có đảm bảo UTF-8 trên Android không; decoder nào hỗ trợ ECI 26 (ML Kit, iOS Vision, Google Lens, Zalo/Momo/banking); mojibake khi decoder bỏ qua ECI; heuristic `StringUtils.guessCharset`; **chốt: luôn UTF-8 hay chỉ khi cần** |
| R6 | Capacity check & fallback & round-trip verify | Công thức capacity đầy đủ 4 mode (đối chiếu ISO Table 7); capacity thực tế khác lý thuyết do ZXing chỉ chọn 1 mode; có nên bật `QR_COMPACT`; Structured Append có encode được / scanner có ghép được; taxonomy lỗi; ràng buộc validate từng format 1D; chi phí và giá trị của round-trip verify |

### 5.1 Phát hiện phụ đã xác nhận — hai module khai báo ZXing lệch version

| Module | Khai báo |
|---|---|
| `app/build.gradle:229` | `com.google.zxing:core:**3.5.4**` |
| `qr_generator/build.gradle:40` | `com.google.zxing:core:**3.5.3**` |

Gradle resolve theo version cao nhất → **runtime thực tế là 3.5.4**, nhưng `qr_generator` được **compile** với API 3.5.3. Với patch version thì thường không sao, nhưng:

- **CLAUDE.md ghi sai** ("ZXing Core 3.5.3") — cần sửa thành 3.5.4.
- Toàn bộ số liệu capacity ở Mục 2.1 lấy từ sources jar 3.5.3. Cần đối chiếu lại `Version.java` của 3.5.4 để chắc chắn hằng số ECBlocks không đổi (khả năng rất cao là không đổi, nhưng phải kiểm).
- Nên thống nhất về một version duy nhất, khai báo ở một chỗ (`ext` ở root `build.gradle`) để không lệch nữa.

Ngoài ra `app/build.gradle:189` còn `implementation files('libs/zxingcpp-release.aar')` (native encode/decode) và `:273` có dòng comment `zxing-cpp:v2.0.0.2` — cần xác định AAR này đang được dùng cho decode, encode, hay cả hai, và có phải là đường thoát tốt hơn cho phần encode không.

---

## 6. Thứ tự đề xuất

1. **Giai đoạn 0 ngay** — 0 code, cho biết rơi ở đâu. Không có nó thì mọi việc sau đều là đoán.
2. **1.1 tách riêng thành 1 commit** — 1 ký tự, tác động lớn nhất, dễ verify, dễ revert.
3. **1.5 + 1.6 vào cùng release với 1.1** — để release sau có số liệu sạch mà đo.
4. Giai đoạn 2 và 3 theo sau.
5. Giai đoạn 4 chỉ khi số liệu xác nhận.

**Chưa sửa dòng code nào.** Chờ xác nhận phương án trước khi bắt đầu.

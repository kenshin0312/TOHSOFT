# Nghiên cứu công nghệ Scan QR/Barcode — Tối ưu tỷ lệ quét thành công trong điều kiện bất thường

- **Ngày thực hiện:** 2026-08-18
- **Phạm vi:** com.tohsoft.qrcode2023 (app Android quét & tạo QR/Barcode, TOHSoft)
- **Người yêu cầu:** dinhpv@tohsoft.com
- **Thực hiện bởi:** Agent `android-qr-researcher` + khảo sát code trực tiếp

---

## Mục lục

1. [Tóm tắt kết luận](#1-tóm-tắt-kết-luận)
2. [Baseline — Hiện trạng pipeline scan của app](#2-baseline--hiện-trạng-pipeline-scan-của-app)
3. [Tổng quan bài toán](#3-tổng-quan-bài-toán)
4. [Cách hoạt động / Pipeline lý tưởng](#4-cách-hoạt-động--pipeline-lý-tưởng)
5. [Kiến trúc đề xuất](#5-kiến-trúc-đề-xuất)
6. [So sánh công nghệ](#6-so-sánh-công-nghệ)
7. [Performance](#7-performance)
8. [Khả năng mở rộng](#8-khả-năng-mở-rộng)
9. [Best Practice (theo Google chính thức)](#9-best-practice-theo-google-chính-thức)
10. [Anti-pattern](#10-anti-pattern)
11. [Production Recommendation — Lộ trình theo ROI](#11-production-recommendation--lộ-trình-theo-roi)
12. [Sample code minh họa](#12-sample-code-minh-họa)
13. [Migration](#13-migration)
14. [Xu hướng công nghệ](#14-xu-hướng-công-nghệ)
15. [Tài liệu tham khảo](#15-tài-liệu-tham-khảo)

---

## 1. Tóm tắt kết luận

**Nên cập nhật công nghệ scan, nhưng theo hướng tối ưu nhẹ (Pha 1 + Pha 2) — không cần chuyển sang SDK thương mại (Scandit/Dynamsoft).**

Gap có ROI cao nhất: app đã vendor sẵn `zxingcpp-release.aar` (zxing-cpp) nhưng **chỉ dùng để encode**, trong khi decode camera vẫn dùng ZXing Core Java 3.5.3 — bản cũ, ít cải tiến thuật toán. Chuyển decode sang zxing-cpp gần như miễn phí (không tăng APK, thư viện đã có sẵn) nhưng mở khóa các tùy chọn decode mạnh hơn (`try_rotate`, `try_downscale`, `try_invert`) mà ZXing Java không có.

Ngoài ra, ML Kit 17.3.0 và CameraX 1.4.2 (đều đã có sẵn trong app) đã hỗ trợ `enableAutoZoom()` và Camera Extensions `EXTENSION_AUTO` (Night mode tự động) — hai tính năng miễn phí, sẵn có nhưng **chưa được bật**.

SDK thương mại (Scandit/Dynamsoft) cho accuracy tốt nhất ở các tình huống cực khó (mã cong nặng, mờ chuyển động mạnh, rách/che lớn) nhưng mô hình license theo năm/scan không phù hợp với app free có ads, và phần lớn khoảng cách accuracy có thể thu hẹp bằng lộ trình 2 pha gần như miễn phí bên dưới.

---

## 2. Baseline — Hiện trạng pipeline scan của app

*(Khảo sát trực tiếp từ source code, dùng làm mốc đối chiếu)*

### 2.1 Camera stack

- CameraX **1.4.2** (core/camera2/lifecycle/view/extensions) — `app/build.gradle:240-249`.
- Bind camera chính: `ScanFragment.kt:1226-1330` (`bindPreview`). `ui/scan/` chỉ chứa luồng nhận ảnh/text share từ app khác (`GetDataScanActivity.kt`), không phải camera scan chính.
- `ImageAnalysis`: `setTargetResolution` bằng kích thước preview (hình vuông), `STRATEGY_KEEP_ONLY_LATEST` (tự drop frame cũ). `Preview` dùng `AspectRatio.RATIO_16_9` cố định.
- Analyzer chạy trên 1 thread nền riêng: `Executors.newSingleThreadExecutor()` (`ScanFragment.kt:216`).

### 2.2 Decode logic

- `BaseImageAnalyzer`/`MLKitBarcodeAnalyzer` (`analysis/`) chỉ forward `ImageProxy`, không tự decode. Logic decode thật nằm ở `ScanViewModel.kt:162-394`.
- Throttle: cờ `isAnalyzing: AtomicBoolean` — frame tới khi đang bận thì bị đóng ngay, chạy trên `Dispatchers.IO`.
- **Luồng camera:** chạy song song (race, `async`) `processUsingZxing` và `processUsingMLKit` — job nào xong trước thắng, cancel job còn lại. Chỉ lấy 1 kết quả đầu tiên (không xử lý multi-QR trong 1 khung hình).
- **Luồng ảnh tĩnh (Gallery/Share-intent):** ZXing chạy trước, ML Kit là fallback nếu ZXing fail.

### 2.3 Thư viện decode/encode

| Thư viện | Version | Vai trò hiện tại |
|---|---|---|
| ML Kit Barcode Scanning | 17.3.0 | Decode, `setBarcodeFormats(FORMAT_ALL_FORMATS)` — bật tất cả format |
| ZXing Core (Java) | 3.5.3 | Decode qua `MultiFormatReader` + `RGBLuminanceSource` + `HybridBinarizer` |
| `zxingcpp-release.aar` (zxing-cpp, vendor) | không rõ version | **Chỉ dùng để encode** (generate barcode hiển thị) — chưa dùng để decode |

### 2.4 Preprocessing hiện có / chưa có

**Đã có:**
- Crop theo vùng khung ngắm (viewfinder rect) — `QRUtils.getRectCrop`.
- Xoay ảnh theo `rotationDegrees`/EXIF — `BitmapUtils.rotateBitmap`.
- Resize/scale để tránh OOM khi ảnh Gallery quá lớn — `QRUtils.handleBitmap`.

**Chưa có:**
- Tăng sáng/tương phản, khử nhòe (deblur), phát hiện góc nghiêng/deskew, perspective correction tự động.
- Auto-torch khi thiếu sáng (chỉ có nút bật/tắt thủ công).
- Auto-zoom tự động theo khoảng cách mã (chỉ có pinch-to-zoom/nút/slider thủ công).
- Xử lý multi-QR trong 1 khung hình.

### 2.5 Camera control

- Tap-to-focus: `FocusMeteringAction` (auto-cancel 3s) — `ScanFragment.kt:1619-1642`.
- Pinch-to-zoom + nút/slider zoom thủ công — `ScanFragment.kt:1590-1616, 771-828, 1061-1087, 1204-1215`.
- Torch: bật/tắt thủ công qua nút bấm, không đo độ sáng môi trường — `ScanFragment.kt:1298-1318`.

### 2.6 Tài nguyên có sẵn có thể tái dùng

- Module `smartcropperlib` (vendor, submodule SmartCropper) — auto-crop tài liệu/thẻ ở tính năng khác, công nghệ crop có sẵn trong app.

---

## 3. Tổng quan bài toán

Bài toán không phải "app chưa quét được" mà là **long-tail accuracy** — nhóm 5-15% trường hợp khó (mã cong trên lon/chai, ngược sáng, mờ, nghiêng góc lớn, bị che, tương phản thấp, mã nhỏ/xa) mà pipeline phổ thông (ML Kit + ZXing decode thuần) thường bỏ lỡ. Đây chính là phân khúc mà các SDK thương mại (Scandit, Dynamsoft, Scanbot) bán giá trị gia tăng, qua tổ hợp: camera control chủ động (exposure/torch/zoom) + tiền xử lý ảnh (deskew, adaptive binarization) + AI-based detection/tracking đa khung hình.

Ba trục công nghệ liên quan trực tiếp:

- **Camera capture layer** (CameraX/Camera2): quyết định ảnh đầu vào có "khả dụng để decode" hay không — nơi rẻ nhất để cải thiện vì không cần thay decoder.
- **Decode engine** (ML Kit / ZXing / zxing-cpp / thương mại): quyết định khả năng chịu nhiễu của thuật toán decode.
- **Preprocessing** (OpenCV hoặc thủ công): cầu nối biến ảnh "khó" thành ảnh "dễ" trước khi decode.

---

## 4. Cách hoạt động / Pipeline lý tưởng

```
Camera sensor
  ↓ (đo AE/AWB, luminance histogram)
Adaptive capture control
  ├─ Auto-torch khi luminance dưới ngưỡng
  ├─ Exposure compensation khi backlight/glare (giảm exposure vùng trung tâm)
  └─ Auto-zoom khi barcode nhỏ/xa (ML Kit đã có sẵn)
  ↓
ImageAnalysis (YUV420/NV21)
  ↓
[Preprocessing – lớp còn thiếu ở baseline]
  ├─ Contrast stretch / CLAHE (bù tương phản thấp)
  ├─ Adaptive threshold / Otsu (bù ngược sáng cục bộ)
  ├─ Corner/quad detection → perspective (homography) warp (bù nghiêng góc lớn, bề mặt cong cục bộ)
  └─ Multi-frame fusion / consensus (bù mờ tức thời 1 frame)
  ↓
Decode engine (ML Kit + zxing-cpp, chạy song song hoặc theo tầng)
  ↓
Parser → Business logic
```

**Nguyên lý cốt lõi khi mã QR "cong"** (trên lon nước): QR chuẩn ISO/IEC 18004 giả định mặt phẳng — 3 Finder Pattern + 1 Alignment Pattern dùng để suy ra ma trận biến đổi affine/homography. Khi bề mặt cong, biến đổi thực tế là non-linear (cong theo trục trụ) — homography phẳng chỉ xấp xỉ đúng cục bộ. Không có thuật toán "duỗi thẳng hình trụ" real-time nào trên mobile được Google hay ZXing công bố chính thức. Các SDK thương mại xử lý bằng cách **tối đa hoá tolerance của model phát hiện góc/méo bằng AI-trained model**, chứ không "un-warp" hình học tường minh.

---

## 5. Kiến trúc đề xuất

```
CameraX Preview + ImageAnalysis (giữ nguyên STRATEGY_KEEP_ONLY_LATEST)
        ↓
Camera2Interop hook (MỚI)
  - Đọc CONTROL_AE_STATE / lux ước lượng từ luminance histogram của Y-plane
  - enableTorch(true) tự động khi luminance < ngưỡng (thay vì chỉ nút bấm thủ công)
  - Set CONTROL_AE_EXPOSURE_COMPENSATION âm khi phát hiện vùng sáng gắt cục bộ (glare/backlight)
        ↓
ImageProxy → Bitmap/luminance array (giữ nguyên bước crop theo viewfinder + rotation)
        ↓
[MỚI] Lightweight preprocessing (không cần OpenCV):
  - Contrast stretch tuyến tính trên Y-plane (rẻ, không cần thư viện ngoài)
  - Nếu vẫn fail: retry decode với ảnh đã tăng tương phản
        ↓
Decode song song (giữ pattern race hiện tại, đổi 1 nhánh):
  ├─ processUsingMLKit (giữ nguyên, FORMAT_ALL_FORMATS)
  └─ processUsingZxingCpp (THAY vì ZXing Java) — dùng chính AAR đã vendor
        ↓
Kết quả đầu tiên thắng → Parser → Business logic
```

**Nếu đầu tư thêm (Pha 2, có OpenCV):** chèn bước `detectAndDecodeMulti` của `cv::QRCodeDetector` (hoặc module `wechat_qrcode` — detector CNN nhẹ) **trước khi** giao ML Kit/zxing-cpp, tìm 4 góc mã khi nghiêng > 25-30°, rồi `warpPerspective` trước khi decode.

---

## 6. So sánh công nghệ

### 6.1 ML Kit vs ZXing (Java) vs zxing-cpp

| Tiêu chí | ML Kit Barcode Scanning 17.3.0 | ZXing Core (Java) 3.5.3 — đang decode | zxing-cpp (đã vendor, chỉ encode) |
|---|---|---|---|
| Nền tảng thuật toán | Model Google huấn luyện, cải tiến liên tục (recall, độ trễ đuôi dài, ổn định bounding box) | Port Java gốc ~2010s, ít đổi thuật toán lõi | Viết lại C++, nhánh phát triển độc lập, tối ưu liên tục |
| Auto-zoom | Có (từ 17.2.0, `setZoomSuggestionOptions`) | Không | Không (chỉ decode) |
| Trả mã "chưa chắc decode" hỗ trợ zoom | `enableAllPotentialBarcodes()` (17.1.0+) | Không | Không |
| Tùy chọn tiền xử lý decode | Không phơi ra (black-box) | Không | Có: `try_rotate`, `try_downscale`, `try_invert`, chọn binarizer |
| Offline | Có | Có | Có |
| Chi phí | Miễn phí | Miễn phí (Apache 2.0) | Miễn phí (Apache 2.0) |
| Đã có trong app | Có, đang dùng | Có, đang dùng | Có nhưng chưa dùng để decode |

### 6.2 CameraX thuần vs CameraX + Camera2Interop

| Tiêu chí | CameraX API thuần (baseline) | CameraX + Camera2Interop |
|---|---|---|
| Auto exposure | Google quản lý, không can thiệp trực tiếp | Set `CONTROL_AE_EXPOSURE_COMPENSATION`, đọc `CONTROL_AE_STATE` |
| Torch | Nhị phân qua `enableTorch()` (thủ công) | Tự động hoá theo luminance đo được; một số thiết bị chỉnh được `FLASH_STRENGTH_LEVEL` |
| HDR 10-bit | Có từ CameraX 1.4.0 (app đã ở 1.4.2 — có sẵn, chưa bật) | — |
| Camera Extensions (Night mode) | `ExtensionsManager`, `EXTENSION_NIGHT`/`EXTENSION_AUTO` | — |
| Độ phức tạp tích hợp | Thấp | Trung bình (tự viết đo luminance + ngưỡng, khác nhau theo chipset) |

### 6.3 Bảng so sánh 4 hướng chính

| Tiêu chí | (a) Giữ nguyên | (b) Tối ưu preprocessing, không thêm SDK | (c) Thêm OpenCV | (d) SDK thương mại (Scandit/Dynamsoft) |
|---|---|---|---|---|
| **Accuracy điều kiện khó** | Thấp nhất | Cải thiện: thiếu sáng, ngược sáng nhẹ, tương phản thấp, nghiêng nhẹ-vừa. Chưa xử lý: cong nặng, mờ chuyển động mạnh | Thêm: nghiêng góc lớn, tương phản cực thấp, che một phần. Vẫn hạn chế: cong nặng trên bề mặt trụ nhỏ, motion blur mạnh | Tốt nhất toàn diện (cong, tối, mờ, rách, theo công bố Scandit) |
| **Tốc độ decode** | Nhanh | Gần như không đổi | Chậm hơn (+10-40ms/frame) | Nhanh (tối ưu native riêng) |
| **CPU/RAM** | Thấp | Thấp-trung bình | Trung bình-cao | Trung bình (black-box) |
| **APK size** | Không đổi | ~0 | +8-15MB | +5-20MB |
| **Chi phí license** | 0đ | 0đ | 0đ (Apache/BSD) | Trả phí theo năm/thiết bị/scan — không có tier free, không hợp app free-ads |
| **Effort tích hợp** | 0 | Thấp (vài ngày) | Trung bình-cao (1-3 tuần) | Cao (nhiều tuần-tháng, vendor lock-in) |
| **Rủi ro dài hạn** | Thấp | Thấp | Trung bình (thêm native dependency) | Cao với app free-ads (chi phí theo scale) |

---

## 7. Performance

- **CPU:** contrast stretch/exposure logic (b) là O(n) trên mảng byte, không đáng kể so với decode. zxing-cpp theo benchmark bên thứ ba (môi trường Python, không phải Android chính thức) nhanh hơn ZXing Java — cần tự đo trên thiết bị thật.
- **RAM:** OpenCV Android AAR đầy đủ nặng (~30-50MB native lib) — nếu dùng (c) cần build tùy biến (`core`, `imgproc`, `objdetect`), tránh kéo `opencv_contrib`/`dnn`.
- **Startup:** (a)/(b)/(c) không ảnh hưởng thời gian khởi động camera. SDK thương mại (d) thường có bước khởi tạo license/model riêng, có thể cộng thêm độ trễ.
- **Battery:** auto-torch cần debounce/hysteresis (bật khi lux < ngưỡng thấp, chỉ tắt khi > ngưỡng cao hơn) để tránh nhấp nháy và hao pin.
- **FPS/Latency:** `STRATEGY_KEEP_ONLY_LATEST` + throttle bằng `AtomicBoolean` hiện tại vẫn hợp lý ở cả 4 phương án, không cần đổi.

---

## 8. Khả năng mở rộng

- **Modular:** (b) và (c) có thể cài dưới dạng "preprocessing step" độc lập, bật/tắt qua Firebase Remote Config (pattern đã có sẵn) — cho phép A/B test tỷ lệ scan thành công theo cohort mà không cần release mới.
- **Maintainability:** (b) tận dụng module đã vendor, rủi ro thấp nhất. (c) thêm native dependency ngoài kiểm soát team (nhịp release riêng, cần theo dõi tương thích NDK). (d) toàn quyền phụ thuộc vendor.
- **Offline:** cả 4 phương án hoạt động offline hoàn toàn.
- **Cross-platform:** zxing-cpp và OpenCV có binding iOS nếu sau này làm bản iOS; SDK thương mại có sẵn cả 2 nền tảng nhưng license riêng từng nền tảng.
- **Future support:** ML Kit là hướng Google tiếp tục đầu tư (auto-zoom, enableAllPotentialBarcodes là tính năng 1-2 năm gần đây); ZXing Java gần như "đóng băng" thuật toán so với zxing-cpp.

---

## 9. Best Practice (theo Google chính thức)

Từ tài liệu ML Kit Barcode Scanning:

- Không capture ở độ phân giải camera gốc — chỉ dùng độ phân giải cần thiết, thường **không quá 2 MP** (khuyến nghị 1280x720 hoặc 1920x1080 khi cần quét xa).
- Module nhỏ nhất của mã phải **rộng tối thiểu 2 pixel**; ví dụ EAN-13 nên rộng ≥190px, PDF417 ≥1156px.
- Giới hạn `setBarcodeFormats()` xuống đúng format cần, thay vì `FORMAT_ALL_FORMATS` (baseline hiện đang dùng `FORMAT_ALL_FORMATS`) — tối ưu tốc độ nếu biết trước người dùng chủ yếu quét QR.
- Với streaming video, nên đợi vài frame liên tiếp cùng giá trị trước khi coi là kết quả tin cậy (baseline hiện lấy kết quả đầu tiên thắng race — cân nhắc thêm xác nhận 2 frame, đánh đổi tốc độ).
- Bật `enableAutoZoom()`/`setZoomSuggestionOptions()` — có sẵn miễn phí trong ML Kit 17.2.0+ nhưng **baseline chưa dùng**.
- Camera Extensions `EXTENSION_AUTO` tự chuyển `EXTENSION_NIGHT` khi thiếu sáng — cơ chế chính thức Google cho low-light.

---

## 10. Anti-pattern

- Dùng `FORMAT_ALL_FORMATS` khi chỉ cần QR: tốn thời gian decode, tăng false positive.
- Race 2 decoder song song không giới hạn timeout: ở ảnh khó (mờ, tối) có thể kéo dài độ trễ cảm nhận — nên có ngưỡng timeout để gợi ý người dùng (bật đèn/tiến gần/giữ yên máy) thay vì treo vô hạn.
- Bật torch cưỡng bức toàn thời gian khi thiếu sáng thay vì theo ngưỡng đo thực tế — gây chói khi mã gần, hao pin.
- Thêm OpenCV "cho chắc" nhưng chỉ dùng 1-2 hàm đơn giản — lãng phí 8-15MB APK cho việc có thể làm bằng vài dòng thao tác mảng byte thủ công.
- Kỳ vọng OpenCV/zxing-cpp giải quyết được mã cong nặng trên bề mặt trụ nhỏ như SDK thương mại — giới hạn vật lý của homography phẳng, không phải giới hạn effort code.

---

## 11. Production Recommendation — Lộ trình theo ROI

### Pha 1 (ưu tiên cao nhất, effort thấp — vài ngày)

1. Chuyển decode luồng camera từ ZXing Java → **zxing-cpp** (AAR đã có sẵn, không tăng APK size).
2. Bật `enableAutoZoom()` của ML Kit (miễn phí, đã có sẵn trong version đang dùng).
3. Auto-torch dựa trên luminance đo từ Y-plane của `ImageProxy` + hysteresis chống nhấp nháy.
4. Thử `EXTENSION_AUTO` của CameraX (đã hỗ trợ ở 1.4.2, không cần nâng version).
5. Thêm contrast stretch làm retry pass thứ 2 khi decode lần đầu thất bại.

### Pha 2 (nếu đo được Pha 1 chưa đủ — effort trung bình, 1-3 tuần)

- Thêm module OpenCV rút gọn (`core`+`imgproc`+`objdetect` cho `QRCodeDetector`), chỉ dùng cho bước phát hiện 4 góc + perspective warp khi decode thất bại — chạy như retry pass thứ 3, không chạy mặc định mọi frame.

### Không khuyến nghị SDK thương mại (d) ở thời điểm hiện tại

- Mô hình license theo năm/scan không phù hợp doanh thu ads.
- Effort tích hợp cao, rủi ro vendor lock-in.
- Phần lớn gap có thể thu hẹp bằng Pha 1+2 với chi phí gần bằng 0.
- Chỉ nên cân nhắc lại nếu: có nguồn thu B2B/enterprise riêng, hoặc dữ liệu thực tế cho thấy tỷ lệ fail trên mã cong/damaged là điểm rời bỏ người dùng nghiêm trọng.

---

## 12. Sample code minh họa

### Bước 1 — Auto-torch theo luminance (không cần thư viện ngoài)

```kotlin
// Trong analyzer đang forward ImageProxy (BaseImageAnalyzer/MLKitBarcodeAnalyzer)
private fun estimateLuminance(image: ImageProxy): Int {
    // Y-plane của YUV_420_888 chứa độ sáng — không cần convert màu
    val yPlane = image.planes[0].buffer
    var sum = 0L
    val sampleStep = 16 // sample thưa để rẻ CPU, không cần duyệt hết ảnh
    var count = 0
    var i = 0
    while (i < yPlane.remaining()) {
        sum += yPlane.get(i).toInt() and 0xFF
        count++
        i += sampleStep
    }
    return if (count > 0) (sum / count).toInt() else 128
}

// Trong ScanFragment, nơi đang giữ CameraControl để enableTorch thủ công:
private var torchAutoOn = false
private val TORCH_ON_THRESHOLD = 50   // ngưỡng bật (tối)
private val TORCH_OFF_THRESHOLD = 90  // ngưỡng tắt (đủ sáng), hysteresis tránh nhấp nháy

fun onLuminanceMeasured(lux: Int) {
    if (!torchAutoOn && lux < TORCH_ON_THRESHOLD) {
        camera.cameraControl.enableTorch(true)
        torchAutoOn = true
    } else if (torchAutoOn && lux > TORCH_OFF_THRESHOLD) {
        camera.cameraControl.enableTorch(false)
        torchAutoOn = false
    }
}
```

Giải thích: (1) tận dụng plane Y đã có sẵn trong `ImageProxy` mà pipeline hiện tại đang xử lý để crop/xoay — không cần đọc thêm dữ liệu; (2) sample thưa (stride 16) giữ chi phí CPU gần như 0; (3) hysteresis 2 ngưỡng khác nhau tránh bật/tắt liên tục khi lux dao động quanh 1 ngưỡng.

### Bước 2 — Đổi decode sang zxing-cpp

```kotlin
// Thay processUsingZxing hiện tại (MultiFormatReader + RGBLuminanceSource + HybridBinarizer)
private val zxingCppReader = BarcodeReader(
    BarcodeReader.Options(
        tryRotate = true,     // bù nghiêng trục 90/180/270 mà baseline chưa có
        tryDownscale = true,  // bù ảnh quá lớn/nhiễu
        tryInvert = true,     // bù mã in ngược màu (nền tối/chữ sáng) — thường gặp trên bao bì
        format = BarcodeReader.Format.QR_CODE // hoặc mở rộng theo nhu cầu
    )
)

suspend fun processUsingZxingCpp(imageProxy: ImageProxy): String? {
    return zxingCppReader.read(imageProxy)?.text
}
```

> **Lưu ý:** API mẫu trên minh hoạ theo tài liệu wrapper Android chính thức của zxing-cpp trên GitHub — cần đối chiếu lại với đúng API thực tế của AAR đang vendor trong repo (`libs/zxingcpp-release.aar`, không rõ version) trước khi code thật, vì đây là bản vendor không có version tag rõ ràng.

---

## 13. Migration

Từ ZXing Java decode → zxing-cpp decode:

1. Xác nhận API thực tế của `zxingcpp-release.aar` đang vendor (đọc trực tiếp file AAR/README đi kèm nếu có) trước khi code thật.
2. Giữ nguyên interface đầu ra (`String?`) để không phải sửa lớp gọi race `async` trong `ScanViewModel.kt` — chỉ thay thân hàm `processUsingZxing`.
3. Chạy song song cả 2 decoder (ZXing Java cũ + zxing-cpp mới) trên tập ảnh test thật (bao gồm ảnh mã cong/mờ/nghiêng sưu tầm từ báo cáo lỗi người dùng) trước khi gỡ ZXing Java hẳn.
4. Với luồng ảnh tĩnh (Gallery/Share-intent, đang là "ZXing trước, ML Kit fallback") — áp dụng đổi tương tự.
5. Rollout qua Firebase Remote Config flag (pattern đã có sẵn) để bật/tắt zxing-cpp decode theo % người dùng, đo tỷ lệ scan thành công thực tế trước khi bật 100%.

---

## 14. Xu hướng công nghệ

- **Google đang ưu tiên:** ML Kit tiếp tục cải tiến incremental (auto-zoom, enableAllPotentialBarcodes, unbundled performance parity) hơn là thay đổi kiến trúc lớn; CameraX đầu tư mạnh vào Camera Extensions (Night, HDR 10-bit, Feature Group API ở 1.5) — hướng rõ ràng là "cải thiện input ảnh trước decoder" chứ không phải "decoder AI nặng hơn". Điều này củng cố khuyến nghị Pha 1 (đầu tư camera capture control).
- **Nguy cơ deprecated:** ZXing Core Java không có dấu hiệu bị khai tử chính thức, nhưng nhịp cải tiến thuật toán gần như dừng so với nhánh zxing-cpp — rủi ro là "tụt hậu accuracy" chứ không phải "bị deprecate".
- **Nên áp dụng cho dự án mới:** zxing-cpp cho mọi nhu cầu decode mới; CameraX Camera2Interop cho camera control nâng cao; cân nhắc `wechat_qrcode` (module OpenCV contrib, detector CNN nhẹ chuyên biệt QR) nếu đầu tư Pha 2.

---

## 15. Tài liệu tham khảo

**Nguồn chính thức:**

- [Scan barcodes with ML Kit on Android](https://developers.google.com/ml-kit/vision/barcode-scanning/android) — Google for Developers
- [Release notes | ML Kit](https://developers.google.com/ml-kit/release-notes) — Google for Developers
- [Google code scanner (Android only)](https://developers.google.com/ml-kit/vision/barcode-scanning/code-scanner) — Google for Developers
- [ML Kit Analyzer | Android media](https://developer.android.com/media/camera/camerax/mlkitanalyzer) — Android Developers
- [Camera extensions | Android media](https://developer.android.com/media/camera/camera-extensions) — Android Developers
- [What's new in CameraX 1.4.0](https://android-developers.googleblog.com/2024/12/whats-new-in-camerax-140-and-jetpack-compose-support.html) — Android Developers Blog
- [Introducing CameraX 1.5](https://developer.android.com/blog/posts/introducing-camera-x-powerful-video-recording-and-pro-level-image-capture) — Android Developers Blog
- [Configuration options | CameraX](https://developer.android.com/media/camera/camerax/configuration) — Android Developers
- [cv::QRCodeDetector Class Reference](https://docs.opencv.org/4.x/de/dc3/classcv_1_1QRCodeDetector.html) — OpenCV chính thức
- [zxing-cpp Android wrapper README](https://github.com/zxing-cpp/zxing-cpp/blob/master/wrappers/android/README.md) — Official GitHub
- [Scandit Barcode Scanner SDK](https://www.scandit.com/products/barcode-scanner-sdk/) — trang sản phẩm chính thức Scandit
- [Scandit AI-Powered Barcode Scanning docs](https://docs.scandit.com/sdks/net/ios/ai-powered-barcode-scanning/) — Scandit Developer Documentation
- [Dynamsoft Barcode Reader Pricing](https://www.dynamsoft.com/barcode-reader/pricing/) — trang giá chính thức Dynamsoft

**Nguồn không chính thức (chỉ đối chiếu số liệu, không lấy làm căn cứ kỹ thuật):** ComponentSource/Capterra/G2/SaaSworthy (giá tham khảo Dynamsoft/Scanbot — số liệu thị trường bên thứ ba), blog Scanbot so sánh ML Kit vs ZXing (quan điểm thương mại của một đối thủ).

> **Khác biệt cần lưu ý:** giá Dynamsoft "$1,469.02/năm" lấy từ ComponentSource (reseller bên thứ ba) — trang giá chính thức của Dynamsoft không công khai số cụ thể mà yêu cầu liên hệ sales. Coi đây là **tham khảo mức sàn**, không phải giá chính thức cuối cùng.

---

*Báo cáo được tạo bởi Claude Code, sử dụng agent `android-qr-researcher` kết hợp khảo sát source code trực tiếp trong repo.*

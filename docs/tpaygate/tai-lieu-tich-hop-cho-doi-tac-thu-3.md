# Tài liệu tích hợp cho đối tác thứ 3

# T-PayGate — Tài liệu tích hợp cho đối tác thứ 3

> **Phiên bản 2.1 — 2026-07-30.**
>
> Tài liệu này được đối chiếu trực tiếp với **source code của chính T-PayGate** (`TMT.TPayGate.HttpApi.Host` + `TMT.TPayment.Application`), không phải suy đoán từ hành vi quan sát được. Bản v1.3 trước đó có nhiều điểm **sai** — đặc biệt là cấu trúc response, phạm vi áp dụng chữ ký, tên field khi tạo bill, mã trạng thái HTTP, và payload phản hồi webhook. Những điểm đó được đánh dấu ⚠️ kèm "tài liệu cũ ghi gì / thực tế là gì" để đối tác đang tích hợp dở sửa nhanh.
>
> Tài liệu áp dụng cho **bất kỳ đối tác thứ 3 nào** (POS, ERP, eCommerce, ứng dụng quản lý bán hàng…), không phụ thuộc ngôn ngữ hay nền tảng.

**Chú thích trạng thái xác minh:**

| Ký hiệu | Ý nghĩa |
|----|----|
| ✅ | **Đã đối chiếu source code T-PayGate** |
| ⚠️ | **Khác với tài liệu v1.3** — dùng mô tả trong tài liệu này |
| ❓ | Không tìm thấy trong source, hoặc phụ thuộc cấu hình vận hành — cần T-PayGate xác nhận |


---

## 0. Mục lục


 1. [Tổng quan & môi trường](#1-t%E1%BB%95ng-quan--m%C3%B4i-tr%C6%B0%E1%BB%9Dng)
 2. [Quy trình chính](#2-quy-tr%C3%ACnh-ch%C3%ADnh)
 3. [Quy ước response chung](#3-quy-%C6%B0%E1%BB%9Bc-response-chung-quan-tr%E1%BB%8Dng)
 4. [Bước 1 — OAuth](#4-b%C6%B0%E1%BB%9Bc-1--oauth-l%E1%BA%A5y-access_token)
 5. [Chữ ký HMAC-SHA256 — áp dụng cho HẦU HẾT endpoint](#5-ch%E1%BB%AF-k%C3%BD-hmac-sha256--%C3%A1p-d%E1%BB%A5ng-cho-h%E1%BA%A7u-h%E1%BA%BFt-endpoint)
 6. [Bước 2 — Connect](#6-b%C6%B0%E1%BB%9Bc-2--connect-k%E1%BA%BFt-n%E1%BB%91i-ng%C3%A2n-h%C3%A0ng)
 7. [Bước 3 — Config](#7-b%C6%B0%E1%BB%9Bc-3--config-qu%E1%BA%A3n-l%C3%BD-k%E1%BA%BFt-n%E1%BB%91i)
 8. [Bước 4 — Bill](#8-b%C6%B0%E1%BB%9Bc-4--bill-t%E1%BA%A1o--v%E1%BA%A5n-tin-h%C3%B3a-%C4%91%C6%A1n)
 9. [Webhook (T-PayGate → đối tác)](#9-webhook-t-paygate--%C4%91%E1%BB%91i-t%C3%A1c)
10. [Bảo mật](#10-b%E1%BA%A3o-m%E1%BA%ADt)
11. [Mã lỗi](#11-m%C3%A3-l%E1%BB%97i)
12. [Khuyến nghị triển khai phía đối tác](#12-khuy%E1%BA%BFn-ngh%E1%BB%8B-tri%E1%BB%83n-khai-ph%C3%ADa-%C4%91%E1%BB%91i-t%C3%A1c)
13. [Testing & Go-live](#13-testing--go-live)
14. [Phụ lục](#14-ph%E1%BB%A5-l%E1%BB%A5c)
15. [Lịch sử thay đổi](#15-l%E1%BB%8Bch-s%E1%BB%AD-thay-%C4%91%E1%BB%95i)


---

## 1. Tổng quan & môi trường

### 1.1 T-PayGate là gì

Cổng kết nối thanh toán trung gian, đứng giữa đối tác thứ 3 và các ngân hàng đối tác. Đối tác chỉ tích hợp **một bộ API duy nhất** của T-PayGate, không phải làm việc kỹ thuật trực tiếp với từng ngân hàng.

Luồng nghiệp vụ: đối tác tạo hóa đơn (bill) → T-PayGate sinh mã VietQR gắn với tài khoản định danh (VA) → khách quét QR chuyển khoản → ngân hàng ghi có → T-PayGate gọi webhook về đối tác để xác nhận.

### 1.2 Hai môi trường

| Mục | Staging (UAT) | Production |
|----|----|----|
| **Base URL** | `https://t-paygate.tpos.dev` | `https://t-paygate.tpos.app` |
| OAuth token | `/api/v1/oauth/token` | `/api/v1/oauth/token` |
| Public API prefix | `/api/v1/public-api/...` | `/api/v1/public-api/...` |
| Connect-bank UI | `/api/v1/public-api/view/connect` | `/api/v1/public-api/view/connect` |
| Webhook timeout ✅ | **30 giây** | **30 giây** |
| Webhook retry ❓ | (xem 9.7) | (xem 9.7) |

> ⚠️ **Sửa so với v1.3:** timeout webhook là **30 giây cho cả hai môi trường** — `HttpClient.Timeout = TimeSpan.FromSeconds(30)` trong `TPayGateCallbackSender`, không phân biệt UAT/PROD. Con số "10 giây ở PROD" trong v1.3 là sai.
>
> Khuyến nghị đối tác không cho nhập base URL tự do, mà chỉ cho chọn `Staging` / `Production` rồi map cứng — tránh trỏ nhầm môi trường thanh toán thật.

### 1.3 Cảnh báo hiệu năng đã đo được

`POST /api/v1/oauth/token` đã được ghi nhận **mất tới \~32 giây rồi trả HTTP 500**, lỗi mang tính gián đoạn (cùng một call thành công trong <200 ms ngay sau đó).

Hệ quả bắt buộc với thiết kế phía đối tác:

* **Đặt timeout HTTP client rõ ràng**, nhỏ hơn ngân sách chờ của client cuối. Timeout mặc định 100 giây của phần lớn HTTP client là quá dài.
* **Bắt buộc cache access token** (TTL 55 phút). Cache là thứ giúp đi qua các đợt gián đoạn nhiều phút của endpoint OAuth.
* **Tuyệt đối không retry API tạo bill** — xem [mục 8.2](#82-refTransactionId-kh%C3%B4ng-h%E1%BB%81-idempotent-quan-tr%E1%BB%8Dng), lý do nghiêm trọng hơn nhiều so với những gì v1.3 mô tả.

### 1.4 Đối tác cần chuẩn bị

| Mục | Khi nào cần |
|----|----|
| `clientId`, `tenantId`, `source`, `clientSecret` (T-PayGate cấp) | OAuth + ký **mọi** request public-api + verify webhook |
| **Tên tenant (**`Name`) trong hệ thống T-PayGate ⚠️ | Verify chữ ký webhook — xem [mục 9.4](#94-ch%E1%BB%AF-k%C3%BD-webhook--%C4%91o%E1%BA%A1n-th%E1%BB%A9-3-kh%C3%B4ng-ph%E1%BA%A3i-source-%E2%9A%A0%EF%B8%8F) |
| `configId` (`x-config-id`) — hoặc tự kết nối bank để lấy `configBankId` | Bước 4 Bill |
| Thông tin ngân hàng đối tác (số tài khoản, mã định danh do bank cấp…) | Bước 2 Connect |
| URL webhook public HTTPS | Trước go-live |
| IP outbound để T-PayGate whitelist | Trước go-live PROD |


---

## 2. Quy trình chính

```
   ┌───────────────────────────────────────────────────────────────────┐
   │                                                                   │
   │   [1] OAUTH      [2] CONNECT       [3] CONFIG       [4] BILL      │
   │                                                                   │
   │   POST /token →  POST /connect →   GET /list   →   POST /bill     │
   │   (cache 55p)    POST /confirm     POST /dis-     (mỗi đơn hàng)  │
   │                  (1 lần/merchant)  connect                        │
   │                                                                   │
   │   ─────────── mọi request từ đây đều phải KÝ ──────────           │
   │                                                                   │
   │                                                  ↓ webhook        │
   └───────────────────────────────────────────────────────────────────┘
```

| Bước | Tần suất | Output | Lưu lại |
|----|----|----|----|
| **1. OAuth** | Mỗi 55 phút (cache) | `Data.AccessToken` (JWT) | Cache phía đối tác |
| **2. Connect** | 1 lần / merchant / bank | `ConfigBankId`, `VaNumber` | DB phía đối tác |
| **3. Config** | Khi cần xem/hủy | List / disconnect | — |
| **4. Bill** | Mỗi đơn hàng | `BillCode`, `QRBase64` (chuỗi EMV) | DB phía đối tác |

### 2.1 Hai mô hình định tuyến tiền

Bước 2 và 3 **chỉ cần khi đối tác tự kết nối tài khoản ngân hàng**:

| Mô hình | Cách làm | Khi nào dùng |
|----|----|----|
| **A.** `configId` cấp sẵn | T-PayGate cấu hình sẵn bank ở phía họ và cấp một `configId`. Đối tác **bỏ qua bước 2 và 3**, dùng thẳng giá trị này ở header `x-config-id` | Một merchant duy nhất, tiền về một tài khoản cố định |
| **B. Tự connect bank** | Đối tác chạy bước 2 để lấy `ConfigBankId`, rồi dùng nó làm `x-config-id` | Nhiều merchant, mỗi merchant một tài khoản nhận tiền |

Khuyến nghị hỗ trợ **cả hai**: ưu tiên `configId` nếu đã cấu hình, nếu trống thì fallback sang `ConfigBankId` của bank đầu tiên có `IsConnected = true`.


---

## 3. Quy ước response chung (QUAN TRỌNG)

⚠️ Tài liệu v1.3 mô tả response dạng:

```jsonc
// ❌ SAI — không tồn tại trên đường truyền
{ "success": true, "results": { ... } }
```

Thực tế **toàn bộ** endpoint đi qua một resource filter chung (`TResourceFilter`) bọc mọi kết quả vào envelope thống nhất, field name **PascalCase**:

```jsonc
// ✅ ĐÚNG
{
  "Error": null,              // null = thành công; object = thất bại
  "Data": { ... } | [ ... ]   // payload, hoặc mảng với các endpoint list
}
```

Khi lỗi:

```json
{
  "Error": { "Code": "TMT.TPayment.Code:300", "Message": "..." },
  "Data": null
}
```

> Nội bộ T-PayGate có kiểu `ResponseApiSingleDto` mang `Success` / `Results` / `Message`, nhưng filter **luôn dịch sang** `{Error, Data}` trước khi ghi ra body. `success`/`results` không bao giờ lộ ra ngoài. Đây chính là nguồn gốc nhầm lẫn của v1.3.

Quy tắc phía đối tác:

* Điều kiện thành công là `Error == null`, không phải `success == true`.
* **Không** dựa vào HTTP status để phân biệt lỗi nghiệp vụ.
* Cấu hình JSON parser **case-insensitive** — request body dùng camelCase, response dùng PascalCase.

| Chiều | Quy ước đặt tên | Ví dụ |
|----|----|----|
| Request body (JSON) | camelCase | `bankCode`, `refTransactionId`, `configBankId` |
| Request body (OAuth form) | camelCase | `clientId`, `tenantId`, `source` |
| Response body | PascalCase | `ConfigBankId`, `BillCode`, `QRBase64` |
| Webhook body | PascalCase | `RefTransactionId`, `BillCode`, `PaymentTime` |
| HTTP header | kebab-case, viết thường | `x-client-id`, `x-config-id`, `x-signature` |


---

## 4. Bước 1 — OAuth (lấy `access_token`)

### 4.1 Endpoint

```
POST /api/v1/oauth/token

Content-Type: application/x-www-form-urlencoded
```

Đây là endpoint **duy nhất không cần chữ ký**.

### 4.2 Request ✅

Phải gửi **đồng thời header và form body**:

* **Header** — `PublicApiOauthMiddleware` kiểm tra, thiếu là chặn ngay.
* **Form body** — controller bind vào DTO.

| Vị trí | Field | Bắt buộc | Mô tả |
|----|----|----|----|
| Header | `x-client-id` | ✅ | T-PayGate cấp khi onboarding |
| Header | `x-tenant-id` | ✅ | T-PayGate cấp |
| Header | `x-source` | ✅ | Phân kênh |
| Body | `clientId` | ✅ | Trùng giá trị `x-client-id` |
| Body | `tenantId` | ✅ | Trùng giá trị `x-tenant-id` |
| Body | `source` | ✅ | Trùng giá trị `x-source` |

```http

POST /api/v1/oauth/token HTTP/1.1

Host: t-paygate.tpos.dev

Content-Type: application/x-www-form-urlencoded

x-client-id: demo_client

x-tenant-id: abc-123

x-source: WEB

clientId=demo_client&tenantId=abc-123&source=WEB
```

> ⚠️ **v1.3 thiếu hoàn toàn 3 header.** Chỉ gửi form body như v1.3 mô tả sẽ **bị chặn ở middleware**, không lấy được token.
>
> ✅ **Về hoa/thường của form key:** DTO khai báo `[FromForm(Name = "clientId")]` — tức camelCase là tên chuẩn. Model binding của [ASP.NET](http://ASP.NET) Core **không phân biệt hoa thường**, nên `ClientId=...` cũng bind được. Dùng camelCase cho đúng đặc tả.

### 4.3 Lỗi khi thiếu / sai header ⚠️

Toàn bộ đều trả **HTTP 403 Forbidden**, *không phải 401*:

| Tình huống | HTTP | `Error.Message` |
|----|----|----|
| Thiếu `x-tenant-id` | 403 | `#TPayGate: x-tenant-id is required` |
| Thiếu `x-client-id` | 403 | `#TPayGate: x-client-id is required.` |
| Thiếu `x-source` | 403 | `#TPayGate: x-source is required.` |
| Bộ ba không khớp tenant nào | 403 | `#TPayGate: x-tenant-id không hợp lệ.` |

> ⚠️ v1.3 ghi "OAuth sai `clientId` → HTTP 401". Sai — là **403**. Đối tác bắt nhầm 401 sẽ không nhận diện được lỗi cấu hình credentials.

### 4.4 Response (200) ⚠️

```json
{
  "Error": null,
  "Data": {
    "AccessToken": "eyJhbGciOiJIUzI1NiIsInR...",
    "TokenType": "Bearer",
    "ExpiresIn": 3600
  }
}
```

v1.3 mô tả `{"accessToken": "...", "expiresIn": 3600}` ở root — **sai**, token nằm trong `Data.AccessToken`.

Phòng thủ bắt buộc:

* Endpoint có lúc trả **nội dung không phải JSON** (HTML error page từ reverse proxy). Bắt lỗi parse riêng, log nguyên văn body.
* `Data.AccessToken` rỗng ⇒ coi như thất bại, kể cả khi HTTP 200 và `Error == null`.

### 4.5 Sử dụng token

```
Authorization: Bearer <AccessToken>
```

Token mang 3 claim `TenantId` / `ClientId` / `Source`; các middleware sau đọc từ claim chứ không đọc lại header.

### 4.6 Lưu ý vận hành

* Token TTL = **3600 giây**. Cache phía đối tác **55 phút**.
* **Key cache phải gồm cả base URL** ngoài `clientId`/`tenantId`, để Staging và Production không dùng nhầm token của nhau.
* **Không gọi** `/token` mỗi request.
* ❓ Hành vi khi token hết hạn (401 từ JWT middleware) chưa quan sát được trong thực tế vì cache 55 phút luôn hết trước token. Vẫn nên hiện thực nhánh xử lý 401 + refresh.


---

## 5. Chữ ký HMAC-SHA256 — áp dụng cho HẦU HẾT endpoint

⚠️ **Đây là sai sót nghiêm trọng nhất của v1.3**, vốn khẳng định "mọi chữ ký số do T-PayGate tự xử lý nội bộ, đối tác không phải tự ký hoặc verify". Hoàn toàn ngược lại.

### 5.1 Phạm vi áp dụng ✅

`PublicApiMiddleware` verify chữ ký cho **mọi request tới** `/api/v1/public-api/*` khi request đó **đã xác thực** (có Bearer token hợp lệ), **trừ** `/api/v1/public-api/view/*`.

| Endpoint | Cần ký? |
|----|----|
| `POST /api/v1/oauth/token` | ❌ Không |
| `GET /api/v1/public-api/view/connect` | ❌ Không (miễn trừ tường minh) |
| `POST /api/v1/public-api/view/otp` | ❌ Không |
| `GET /api/v1/public-api/bank` | ⚠️ Xem [mục 7.3](#73-danh-s%C3%A1ch-ng%C3%A2n-h%C3%A0ng-%C4%91%C6%B0%E1%BB%A3c-h%E1%BB%97-tr%E1%BB%A3-%E2%9A%A0%EF%B8%8F) — endpoint ẩn danh |
| `GET /api/v1/public-api/config-bank/list` | ✅ **Có** |
| `POST /api/v1/public-api/config-bank/connect` | ✅ **Có** |
| `POST /api/v1/public-api/config-bank/confirm` | ✅ **Có** |
| `POST /api/v1/public-api/config-bank/disconnect` | ✅ **Có** |
| `POST /api/v1/public-api/order/bill` | ✅ **Có** |
| `GET /api/v1/public-api/order/get-refTransactionId` | ✅ **Có** |
| `GET /api/v1/public-api/order/get-billCode` | ✅ **Có** |

> ⚠️ v1.3 (và cả bản v2.0 của tài liệu này) chỉ nói `POST /order/bill` cần ký. **Sai.** Nếu chỉ ký mỗi endpoint tạo bill, toàn bộ chức năng quản lý kết nối ngân hàng sẽ trả **403** `#TPayGate: Invalid signature.`

### 5.2 Công thức — KHÁC NHAU giữa POST và GET ✅

```
POST:  message = "{clientId}_{tenantId}_{source}_{timestamp}_{rawBody}"
GET:   message = "{clientId}_{tenantId}_{source}_{timestamp}"          ← KHÔNG có body

key       = clientSecret

signature = HMAC-SHA256(message, key) → hex chữ THƯỜNG, 64 ký tự
```

Trong đó:

* `clientId`, `tenantId`, `source` = giá trị lấy từ **claim của token**, tức đúng bộ đã dùng ở OAuth.
* `timestamp` = giá trị header `x-api-time`, **Unix seconds**.
* `rawBody` = chuỗi JSON body **nguyên văn** (T-PayGate đọc lại từ stream, không re-serialize).
* Dấu phân cách `_`: **4 dấu với POST, 3 dấu với GET**.

Chi tiết bắt buộc lưu ý:

| Điểm | Chi tiết |
|----|----|
| So sánh chuỗi | `StringComparison.Ordinal` — **phân biệt hoa thường**. T-PayGate sinh hex chữ thường (`{0:x2}`), nên đối tác **phải gửi chữ thường** |
| Cửa sổ thời gian | `> 5 phút` lệch (theo trị tuyệt đối) ⇒ từ chối |
| `x-api-time` | Được parse bằng `long.Parse` **không TryParse** — thiếu hoặc không phải số sẽ gây lỗi phía T-PayGate. Luôn gửi, luôn là số nguyên |
| Serialize | Serialize **một lần**, ký trên chuỗi đó, gửi **chính chuỗi đó** |

### 5.3 Tenant cha / tenant con ✅

Nếu tenant của đối tác là **tenant con** (có `TenantMatchingParentId`), T-PayGate verify bằng `ClientSecret` của tenant cha, không phải secret của chính nó:

```
signSecret = tenant.ParentId rỗng ? tenant.ClientSecret : tenant.Parent.ClientSecret
```

❓ Đối tác cần hỏi T-PayGate lúc onboarding: **tenant của mình là cha hay con**, và `clientSecret` được cấp là của cấp nào. Nhầm cấp ⇒ mọi request đều 403.

### 5.4 Ví dụ

```csharp
// POST

var bodyJson  = JsonSerializer.Serialize(apiBody);                  // MỘT LẦN

var timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString();
var message   = $"{clientId}_{tenantId}_{source}_{timestamp}_{bodyJson}";

using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(clientSecret));
var signature  = Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(message)))
                        .ToLowerInvariant();                        // BẮT BUỘC chữ thường

request.Headers.Add("x-api-time",  timestamp);
request.Headers.Add("x-signature", signature);
request.Content = new StringContent(bodyJson, Encoding.UTF8, "application/json");

// GET — bỏ đoạn body

var messageGet = $"{clientId}_{tenantId}_{source}_{timestamp}";
```

```js
// Node.js — POST

const bodyJson  = JSON.stringify(apiBody);
const timestamp = Math.floor(Date.now() / 1000).toString();
const message   = `${clientId}_${tenantId}_${source}_${timestamp}_${bodyJson}`;
const signature = crypto.createHmac("sha256", clientSecret).update(message, "utf8").digest("hex");
// GET: bỏ `_${bodyJson}`
```

### 5.5 Mã lỗi khi ký sai ⚠️

Tất cả là **403**, không phải 401:

| Tình huống | HTTP | `Error.Message` |
|----|----|----|
| Thiếu hoặc sai `x-signature` | 403 | `#TPayGate: Invalid signature.` |
| `x-api-time` lệch > 5 phút | 403 | `#TPayGate: Request expired.` |
| Token thiếu claim TenantId/ClientId/Source | 403 | `#TPayGate: TenantID is required` … |
| Bộ ba tenant/client/source không khớp | 403 | `#TPayGate: Access denied.` |
| Thiếu `x-config-id` (endpoint `/order/*`) | **400** | `#TPayGate: configBankId is required` |
| `x-config-id` không tồn tại / đã ngắt kết nối | 403 | `#TPayGate: configBankId does not exist or is disconnected.` |


---

## 6. Bước 2 — Connect (kết nối ngân hàng)

Mục tiêu: liên kết tài khoản ngân hàng của merchant với T-PayGate **một lần duy nhất**, sau đó dùng `ConfigBankId` cho mọi giao dịch ở bước 4.

> Bỏ qua mục 6 và 7 nếu dùng mô hình A (`configId` cấp sẵn).
>
> **Mọi endpoint** `/config-bank/*` đều phải ký — xem [mục 5](#5-ch%E1%BB%AF-k%C3%BD-hmac-sha256--%C3%A1p-d%E1%BB%A5ng-cho-h%E1%BA%A7u-h%E1%BA%BFt-endpoint).

### 6.1 Hai cách kết nối

| Cách | Ưu điểm | Phù hợp |
|----|----|----|
| **A. Embedded UI** (`/view/connect`) | T-PayGate lo UI, OTP, validate; **không cần ký** | Web app, ít custom |
| **B. API trực tiếp** (`/config-bank/connect`) | Đối tác tự design UI; **phải ký** | Mobile app, native |

### 6.2 Cách A — Embedded UI ✅

```
GET /api/v1/public-api/view/connect?bankCode={bankCode}&token={AccessToken}
```

* Token truyền qua **query string**, không phải header `Authorization`.
* Cả `bankCode` và `token` phải được **URL-encode**.
* **Không cần** `x-signature` — `/view/*` được miễn trừ tường minh trong middleware.
* T-PayGate render form phù hợp theo từng ngân hàng; đối tác không cần biết trước các field.

Sau khi user submit, T-PayGate tự gọi `/config-bank/connect` ở backend của họ, hiển thị màn OTP nếu cần (`/view/otp`), rồi báo thành công.

#### Đóng tab / postMessage ✅

Trang T-PayGate gửi `postMessage` về `window.opener` với payload `"tabClosed"` — có thể ở dạng **chuỗi thuần** hoặc **object** `{ type: "tabClosed" }`. Xử lý **cả hai**:

```js

window.addEventListener("message", (event) => {
  if (event.origin !== allowedOrigin) return;            // BẮT BUỘC: chặn origin lạ
  if (event.data?.type === "tabClosed" || event.data === "tabClosed") {
    refreshConnectedBanks();                             // gọi lại /config-bank/list
  }
});
```

> ⚠️ **Bắt buộc kiểm tra** `event.origin`. v1.3 không nêu; bỏ qua sẽ để trang web bất kỳ giả lập được sự kiện "kết nối thành công".
>
> `tabClosed` **chỉ là tín hiệu refresh, không phải bằng chứng thành công.** Phải gọi `/config-bank/list` để xác nhận `IsConnected`.
>
> Nên có phương án dự phòng khi popup bị chặn: mở URL trong tab mới.

### 6.3 Cách B — API trực tiếp

```
POST /api/v1/public-api/config-bank/connect

Authorization: Bearer <token>
x-api-time: <unix seconds>
x-signature: <hmac hex thường>
Content-Type: application/json
```

Body — camelCase (`ConnectConfigBankDto`):

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `bankCode` | string | ✅ | Mã ngân hàng, lấy động từ `GET /bank` |
| `merchantName` | string | ✅ | Tên cửa hàng / merchant |
| `accountName` | string | ✅ | Tên chủ tài khoản |
| `accountNo` | string | ✅ | Số tài khoản nhận tiền |
| `identity` | string | — | CMND/CCCD (một số bank yêu cầu) |
| `phone` | string | — | SĐT đăng ký dịch vụ |
| `email` | string | — | Email |
| `prefix` | string | — | Mã merchant do ngân hàng cấp |
| `clientId` | string | — | Mã định danh **do ngân hàng cấp** |
| `encryptKey` | string | — | Khóa do ngân hàng cấp (nếu yêu cầu) |
| `secretKey` | string | — | Khóa do ngân hàng cấp (nếu yêu cầu) |

> ⚠️ **Xung đột tên:** `clientId` trong body này là **mã ngân hàng cấp cho merchant**, không liên quan tới `clientId` OAuth ở header `x-client-id`. Đừng dùng chung biến cấu hình.

Response:

```json
{
  "Error": null,
  "Data": {
    "ConfigBankId": "abc-uuid",
    "IsConnected": false,
    "IsOTPConfirmation": true
  }
}
```

#### Xác thực OTP (chỉ khi `IsOTPConfirmation = true`)

```
POST /api/v1/public-api/config-bank/confirm
```

```json
{ "configBankId": "abc-uuid", "otpNumber": "123456" }
```

Response:

```json
{
  "Error": null,
  "Data": {
    "ConfigBankId": "abc-uuid",
    "VaNumber": "9XYZ200309052356",
    "IsConnected": true
  }
}
```

> ❓ **Độ dài OTP do ngân hàng quy định**, T-PayGate không đặc tả. Các tích hợp hiện tại validate 6 chữ số — xác nhận lại khi mở rộng sang bank mới.

#### Lấy `VaNumber` khi không cần OTP ⚠️

Khi `IsConnected = true` ngay ở response `/connect`, response đó **không chứa** `VaNumber`. Phải gọi thêm `GET /config-bank/list` rồi lọc theo `ConfigBankId`.

#### Idempotency phía đối tác

Trước khi tạo record trong DB, kiểm tra `ConfigBankId` trả về đã tồn tại chưa — tránh sinh bản ghi trùng khi gọi `/connect` nhiều lần.

### 6.4 Sequence (cách B với OTP)

```
3rd Party             T-PayGate           Bank
   │                     │                  │
   │ POST /connect       │                  │
   ├────────────────────>│                  │
   │                     │ Register         │
   │                     ├─────────────────>│
   │                     │<─────────────────┤ OTP gửi SMS đến KH
   │ {IsOTPConfirmation} │                  │
   │<────────────────────┤                  │
   │ KH nhập OTP         │                  │
   │ POST /confirm       │                  │
   ├────────────────────>│                  │
   │                     │ Verify           │
   │                     ├─────────────────>│
   │                     │<─────────────────┤ OK + VA number
   │ {IsConnected:true}  │                  │
   │<────────────────────┤                  │
```


---

## 7. Bước 3 — Config (quản lý kết nối)

### 7.1 Liệt kê kết nối hiện có ✅

```
GET /api/v1/public-api/config-bank/list?bankCode={optional}
Authorization: Bearer <token>
x-api-time / x-signature       ← BẮT BUỘC (dạng GET, không có body)
```

Response — `Data` là **mảng**:

```json
{
  "Error": null,
  "Data": [
    {
      "ConfigBankId": "abc-uuid",
      "BankCode": "XXX",
      "AccountNo": "108800888060",
      "AccountName": "NGUYEN VAN A",
      "VaNumber": "9XYZ200309052356",
      "MerchantId": "INTERNAL_ID",
      "ClientId": "provider_xxx",
      "Phone": "0987654321",
      "IsConnected": true,
      "UrlLogo": "https://.../logo.png",
      "DateCreated": "2026-05-01T08:00:00Z"
    }
  ]
}
```

> Response **không có tên ngân hàng** (`BankName`) — phải tự join `BankCode` với danh sách từ `GET /bank`.
>
> `VaNumber`, `MerchantId`, `ClientId`, `Phone`, `UrlLogo` đều **nullable**.

### 7.2 Hủy kết nối ✅

```
POST /api/v1/public-api/config-bank/disconnect
```

```json
{ "configBankId": "abc-uuid" }
```

T-PayGate gọi API hủy phía bank rồi set `IsConnected = false` (**giữ record**, không xóa cứng). Đối tác nên làm tương tự để không mất khả năng đối soát.

> Sau khi disconnect, `x-config-id` tương ứng sẽ bị từ chối với **403** `configBankId does not exist or is disconnected.`

### 7.3 Danh sách ngân hàng được hỗ trợ ⚠️

```
GET /api/v1/public-api/bank
```

✅ `BankController` **không có** `[Authorize]` → endpoint này **gọi được ẩn danh, không cần token và không cần ký**.

> ⚠️ **Cạm bẫy quan trọng.** Middleware chỉ verify chữ ký khi request **đã xác thực**. Do đó:
>
> * Gọi **không kèm** `Authorization` ⇒ chạy bình thường ✅
> * Gọi **có** `Authorization: Bearer` nhưng **không ký** ⇒ **403** `Invalid signature.` ❌
>
> Nghĩa là thêm Bearer token "cho chắc" sẽ **làm hỏng** request. Hoặc gọi ẩn danh, hoặc gửi kèm cả `x-api-time` + `x-signature`. Cả v1.3 ("không cần auth") lẫn bản v2.0 ("bắt buộc Bearer") đều mô tả thiếu vế còn lại.

Response — `Data` là mảng ngân hàng đang `IsActive`:

```json
{
  "Error": null,
  "Data": [
    { "Code": "XXX", "Name": "Ngân hàng TMCP ...", "ShortName": "XXXBank",
      "Logo": "https://.../logo.png", "Bin": "970xxx" }
  ]
}
```

⚠️ Field ảnh là `Logo` ở đây nhưng `UrlLogo` ở `/config-bank/list`.

Dùng `Code` (chuẩn NAPAS) làm `bankCode` ở các bước sau. **Không hardcode danh sách.**


---

## 8. Bước 4 — Bill (tạo & vấn tin hóa đơn)

Mọi endpoint `/order/*` yêu cầu `x-config-id` (thiếu ⇒ 400) **và chữ ký** (sai ⇒ 403).

### 8.1 Tạo hóa đơn

```
POST /api/v1/public-api/order/bill

Content-Type: application/json
```

#### Header — toàn bộ bắt buộc ✅

| Header | Mô tả |
|----|----|
| `Authorization` | `Bearer <AccessToken>` |
| `x-config-id` | `configId` được cấp sẵn, hoặc `ConfigBankId` từ bước 2. **Phải đang kết nối** |
| `x-api-time` | Unix timestamp (giây) |
| `x-signature` | HMAC-SHA256 hex chữ thường (dạng POST, có body) |

> ✅ `x-client-id` / `x-tenant-id` / `x-source` KHÔNG cần ở endpoint này — middleware đọc chúng từ **claim của JWT**, không đọc header. Gửi thêm cũng vô hại nhưng không bắt buộc. (Bản v2.0 của tài liệu này liệt kê chúng là bắt buộc — thừa.)

#### Body — camelCase

| Field | Type | Bắt buộc | Mô tả |
|----|----|----|----|
| `refTransactionId` | string | ✅ | Mã giao dịch tham chiếu phía đối tác |
| `amount` | decimal | ✅ | Số tiền (VND) |
| `description` | string | — | Nội dung giao dịch |

```json
{
  "refTransactionId": "ORDER-2026-0001",
  "amount": 250000,
  "description": "Thanh toan don hang 0001"
}
```

### 8.2 `refTransactionId` KHÔNG HỀ idempotent (QUAN TRỌNG)

⚠️ **Sửa sai nghiêm trọng so với v1.3.**

v1.3 nói: HTTP `409` khi trùng `refTransactionId`, và "trùng → trả response cũ, không double-process", `refTransactionId` được "cache 24h".

Thực tế trong source:

* DTO ghi rõ: *"Mã giao dịch tham chiếu của đối tác **(Có thể tạo được nhiều lần)**"*.
* Toàn bộ khối kiểm tra trùng lặp trong `PaymentGate_OrderBillService` **đã bị comment out** — gồm cả check "đã thanh toán", check lệch số tiền, và check khác tài khoản.
* Mỗi lần gọi đều đi thẳng vào `CreateOrderBillAsync` → **sinh một** `BillCode` mới và một QR mới.

**Không có HTTP 409. Không có cache idempotency. Không có dedupe.**

Hệ quả bắt buộc:

* **Tuyệt đối không retry** `POST /order/bill` — kể cả khi timeout. Một lần retry = một bill thứ hai hợp lệ = khách có thể trả tiền hai lần.
* Nếu request timeout mà không rõ kết quả: **dùng** `GET /order/get-refTransactionId` để kiểm tra trước khi tạo lại (endpoint này trả về *danh sách* bill cùng `refTransactionId` — chính vì trùng là chuyện bình thường).
* Đối tác phải **tự đảm bảo tính duy nhất** phía mình.

### 8.3 Response ⚠️ — khác hoàn toàn cả v1.3 lẫn suy đoán trước đây

`PayGateOrderBillResponseDto` **kế thừa** request DTO, nên có cả `RefTransactionId` / `Amount` / `Description`:

```json
{
  "Error": null,
  "Data": {
    "RefTransactionId": "ORDER-2026-0001",
    "Amount": 250000,
    "Description": null,
    "BillCode": "B2026050500001",
    "QRBase64": "00020101021238...",
    "QrDataURL": "data:image/png;base64,iVBORw0KGgo...",
    "InfoPayment": {
      "Bank": "Ngân hàng TMCP ...",
      "Amount": 250000,
      "AccountNumber": "9XYZ200309052356",
      "AccountName": "NGUYEN VAN A",
      "Description": "B2026050500001"
    }
  }
}
```

| Field | Ghi chú |
|----|----|
| `BillCode` | Mã bill T-PayGate sinh |
| `QRBase64` | ⚠️ **Tên gây hiểu nhầm — KHÔNG phải base64.** Là **chuỗi EMV VietQR** (`00020101021238...`). Dùng để tự render QR |
| `QrDataURL` | Ảnh phiếu thanh toán dạng data URI. Sinh bởi service render ngoài (template `print`) — **có thể rỗng** khi service đó lỗi |
| `InfoPayment` | ⚠️ **Hoàn toàn không có trong v1.3.** Chứa thông tin để hiển thị/chuyển khoản thủ công |
| `InfoPayment.AccountNumber` | ⚠️ **Đây mới là số VA.** |
| `InfoPayment.Description` | Bằng `BillCode` — đây là **nội dung chuyển khoản**, không phải `description` đối tác gửi lên |
| `RefTransactionId`, `Amount` | Echo lại từ request |
| `Description` | **Luôn** `null` — service không gán lại |

> ⚠️ **KHÔNG tồn tại** `VirtualAccount` ở cấp gốc. ⚠️ **KHÔNG tồn tại** `ExpiredAt`. Cả hai đều được v1.3 mô tả (dưới tên `vaNumber` / `expiredAt`) nhưng **không có trong response**. Đối tác nào đang map hai field này sẽ luôn nhận `null`:
>
> * Số VA lấy ở `InfoPayment.AccountNumber`.
> * **Không có thời điểm hết hạn nào được trả về** — đối tác phải tự quản lý thời hạn hiển thị QR phía mình.

**Chiến lược render QR:**


1. `QrDataURL` có giá trị → hiển thị trực tiếp (xử lý cả trường hợp có/không có prefix `data:`).
2. `QrDataURL` rỗng → **fallback** render QR từ chuỗi EMV `QRBase64`.
3. Cả hai rỗng → coi như thất bại; **không hiển thị khung QR trống**.

### 8.4 Vấn tin theo `refTransactionId` ✅

```
GET /api/v1/public-api/order/get-refTransactionId?refTransactionId=ORDER-2026-0001

Authorization + x-config-id + x-api-time + x-signature (dạng GET)
```

Trả **mảng** bill cùng `refTransactionId` (vì trùng là hợp lệ — xem 8.2).

### 8.5 Vấn tin theo `billCode` ✅

```
GET /api/v1/public-api/order/get-billCode?billCode=B2026050500001
```

Trả **một** bill.

### 8.6 Shape response vấn tin ✅

Cả hai endpoint dùng `PayGateGetOrderBillResponse`:

```json
{
  "RefTransactionId": "ORDER-2026-0001",
  "BillCode": "B2026050500001",
  "Amount": 250000,
  "AmountPaid": 250000,
  "ConfigBankId": "abc-uuid",
  "State": "Đã thanh toán"
}
```

> `AmountPaid` cho biết **số tiền đã thực nhận** — dùng để phát hiện thanh toán thiếu/một phần.

### 8.7 Vòng đời hóa đơn ⚠️ — hoàn toàn khác v1.3

v1.3 mô tả `CREATED` → `WAITING_PAYMENT` → `PAID` / `EXPIRED` / `CANCELED`.

**Không có giá trị nào trong số đó tồn tại.** `State` là **chuỗi tiếng Việt** sinh từ cờ `PaymentStatusFlags`:

| `State` thực tế | Ý nghĩa |
|----|----|
| `N/A` | Chưa có cờ trạng thái nào |
| `Đơn mới` | Vừa tạo, chưa thanh toán |
| `Thanh toán một phần` | Đã nhận một phần tiền |
| `Đã thanh toán` | Đã nhận đủ |

Chi tiết kỹ thuật: `PaymentStatusFlags` là kiểu **cờ (flags)**; T-PayGate lấy **phần tử cuối** trong danh sách cờ đang bật, nên `State` phản ánh trạng thái tiến xa nhất.

> ⚠️ **Không tồn tại** `EXPIRED`, không tồn tại `CANCELED`, không có API hủy bill. Cơ chế "hết hạn 24h theo tenant" mà v1.3 mô tả không tìm thấy trong luồng public-api.
>
> **Khuyến nghị:** đối tác **không** dựa vào `State` để điều khiển nghiệp vụ, và **không** so khớp chuỗi tiếng Việt trong code (rất dễ đổi). Quản lý trạng thái bằng state machine riêng, do **webhook** đẩy tiến; dùng vấn tin chỉ để đối soát và xử lý trường hợp timeout.


---

## 9. Webhook (T-PayGate → đối tác)

### 9.1 Đăng ký endpoint ✅

* 1 URL public HTTPS, đăng ký lúc onboarding (lưu ở `UrlWebhook` của tenant).
* T-PayGate lấy URL theo thứ tự: `tenant.UrlWebhook` → `tenant.Parent.UrlWebhook` → rỗng (không gửi).
* Endpoint không cần authentication truyền thống — xác thực hoàn toàn bằng **chữ ký HMAC**.

### 9.2 Header ✅

| Header | Mô tả |
|----|----|
| `x-signature` | HMAC-SHA256 hex chữ thường của raw body |
| `x-api-time` | Unix timestamp (giây), sinh bằng `DateTimeOffset.Now` |
| `Content-Type` | `application/json; charset=utf-8` |

### 9.3 Body ✅ — chỉ có MỘT format, phẳng

```json
{
  "RefTransactionId": "TXN202606030001",
  "BillCode": "HD20260603001",
  "Amount": 1500000.00,
  "VirtualAccount": "970422000123456789",
  "ActualAccount": "1234567890123",
  "PaymentTime": "2026-06-03T14:30:00"
}
```

| Field | Type | Mô tả |
|----|----|----|
| `RefTransactionId` | string | Giá trị đối tác gửi khi tạo bill — **khóa đối chiếu** |
| `BillCode` | string | Mã bill T-PayGate sinh |
| `Amount` | decimal | Số tiền đã ghi có |
| `VirtualAccount` | string | Số VA khách chuyển vào |
| `ActualAccount` | string | Số tài khoản thật nhận tiền |
| `PaymentTime` | DateTime **không nullable** | Thời điểm thanh toán |

> ⚠️ **Không tồn tại "webhook v2" với envelope** `EventType` / `Timestamp` / `Data`. Tìm toàn bộ source T-PayGate: **không có** chuỗi `EventType`, **không có** `PAYMENT_RECEIVED`, và cũng **không có** `BDSD` / `VA_REGISTERED` / `RECONCILIATION_READY`. Bảng "các loại sự kiện" trong v1.3 là **giả định, chưa được hiện thực**.
>
> Đối tác **có thể** dựng sẵn endpoint nhận envelope để phòng tương lai, nhưng **không được** thiết kế phụ thuộc vào nó, và đừng chờ T-PayGate "release v2" như một mốc đã có kế hoạch.

### 9.4 Chữ ký webhook — đoạn thứ 3 KHÔNG phải `source` ⚠️

Đây là điểm **rất dễ sai** và không có ở bất kỳ tài liệu nào trước đây.

```
message = "{tenantRoot.ClientId}_{tenant.TenantId}_{tenantRoot.Name}_{timestamp}_{rawBody}"
key     = tenantRoot.ClientSecret
```

Ba khác biệt so với chữ ký **chiều đi** (mục 5.2):

| Đoạn | Chiều đi (đối tác → T-PayGate) | Chiều về (webhook) |
|----|----|----|
| 1 | `clientId` (từ claim) | `ClientId` của **tenant gốc** |
| 2 | `tenantId` (từ claim) | `TenantId` của **tenant đang giao dịch** (có thể là tenant con) |
| 3 | `source` | ⚠️ `Name` của tenant gốc — KHÔNG phải `source` |

`tenantRoot` = chính tenant nếu không có cha, ngược lại là tenant cha. Nếu tenant gốc thiếu `ClientSecret`, T-PayGate **ném lỗi và không gửi webhook**.

> ⚠️ **Bắt buộc hỏi T-PayGate 3 giá trị chính xác dùng để ký webhook**: `ClientId` của tenant gốc, `TenantId` của tenant mình, và `Name` của tenant gốc. Nhiều tích hợp đang dùng `source` cho đoạn thứ 3 và chỉ tình cờ chạy được khi `source` trùng với `Name`. Khi T-PayGate đổi tên tenant, **mọi webhook sẽ 401/403 hàng loạt** mà không có cảnh báo trước.

Yêu cầu triển khai verify:

| Yêu cầu | Chi tiết |
|----|----|
| Đọc **raw body trước** khi parse JSON | Ký lại trên object đã parse rồi serialize sẽ **sai**. Bật buffering, đọc raw, rewind |
| Chống replay | Từ chối khi `\|now − x-api-time\| > 300 giây` |
| So sánh timing-safe | Kiểm tra độ dài trước (64 ký tự hex), rồi so sánh hằng thời gian |
| Chuẩn hóa hoa/thường | Hạ chữ ký nhận được về chữ thường |
| Thiếu header / body rỗng | Từ chối, không xử lý |
| Đồng hồ | Đồng bộ NTP |

### 9.5 Phản hồi từ đối tác ⚠️ — chỉ `MessageError` được đọc

T-PayGate deserialize body phản hồi thành:

```csharp

public class WebhookResponseDto
{
    public string MessageError { get; set; }
    public bool Success => string.IsNullOrEmpty(MessageError);   // computed, KHÔNG đọc từ JSON
}
```

Nghĩa là:

* **Chỉ** `MessageError` được đọc từ JSON. `Success` là property chỉ-đọc tính từ `MessageError` — có gửi `"Success": true` hay không **không ảnh hưởng gì**.
* **Thành công ⇔** `MessageError` là null hoặc chuỗi rỗng.
* Body **không parse được** thành object ⇒ `res == null` ⇒ T-PayGate coi là **lỗi**.

Phản hồi đúng — bất kỳ dạng nào dưới đây:

```json
{ "MessageError": null }
```

```json
{ "Success": true, "MessageError": null }
```

Báo lỗi về cho T-PayGate (để họ retry):

```json
{ "MessageError": "Không tìm thấy giao dịch" }
```

> ⚠️ **v1.3 ghi phản hồi là** `{"MessageError": true}` — đây là hướng dẫn SAI VÀ NGUY HIỂM. Giá trị `true` khi deserialize vào `string MessageError` sẽ thành `"True"` (chuỗi khác rỗng) ⇒ T-PayGate hiểu là **đối tác báo lỗi** ⇒ ném exception và **retry webhook** dù đối tác đã xử lý thành công. Bất kỳ ai làm đúng theo v1.3 đều đang tạo ra vòng lặp retry vô ích.

### 9.6 Điều kiện T-PayGate coi là thất bại ✅

T-PayGate ném exception (⇒ kích hoạt retry) khi:


1. HTTP status **không thành công** (ngoài 2xx), **hoặc**
2. Body phản hồi có `MessageError` khác rỗng, **hoặc**
3. Body không deserialize được.

Do đó nguyên tắc phía đối tác:

> **Đã verify chữ ký xong thì luôn trả HTTP 200 kèm** `MessageError` rỗng, kể cả khi xử lý nghiệp vụ ném exception. Chỉ trả lỗi khi thật sự muốn T-PayGate gửi lại.
>
> Lý do: một lỗi nội bộ nhất thời (DB timeout) mà trả 5xx sẽ tạo retry; nhưng nghiêm trọng hơn — trả 200 với `MessageError` khác rỗng cũng **bị coi là lỗi**, điều mà rất dễ vô tình làm nếu bê nguyên hướng dẫn v1.3.

### 9.7 Retry ❓

`TPayGateCallbackSender` chỉ **ném exception**; việc gửi lại do lớp gọi (hàng đợi Kafka / job) quyết định — không cấu hình được từ file này. Con số **"3 lần × 5 phút"** của v1.3 **không xác minh được trong source**.

Đối tác **phải thiết kế idempotent bất kể số lần retry là bao nhiêu**, và nên hỏi T-PayGate chính sách retry thực tế trước go-live.

### 9.8 Idempotency & guard bắt buộc

Tra giao dịch theo `RefTransactionId`, áp guard theo thứ tự:

| Điều kiện | Hành động |
|----|----|
| Không tìm thấy giao dịch | Log cảnh báo, trả 200 rỗng lỗi — **không tạo mới** |
| Giao dịch đã hoàn tất | Bỏ qua (retry/trùng), trả 200 |
| Trạng thái không hợp lệ để hoàn tất | Log, bỏ qua, trả 200 |
| `Amount` không khớp | **Từ chối xử lý**, log cảnh báo, trả 200 |
| Xung đột concurrency | Bỏ qua, trả 200 |

> ⚠️ Guard **đối chiếu số tiền** đặc biệt quan trọng ở T-PayGate vì [mục 8.2](#82-refTransactionId-kh%C3%B4ng-h%E1%BB%81-idempotent-quan-tr%E1%BB%8Dng): một `refTransactionId` có thể ứng với **nhiều bill khác số tiền**. Không có guard này, một webhook của bill 10.000đ có thể đánh dấu hoàn tất đơn 10.000.000đ.

**Cập nhật UI:** sau khi commit DB, nên phát sự kiện realtime (WebSocket/SSE), **kèm polling định kỳ (\~5 giây) làm dự phòng**.


---

## 10. Bảo mật

### 10.1 Lớp 1 — Kênh

* HTTPS bắt buộc (TLS 1.2+), cert hợp lệ.
* Whitelist IP (PROD).

### 10.2 Lớp 2 — Identity

* OAuth 2.0 Client Credentials qua `clientId` / `tenantId` / `source`.
* JWT TTL = 60 phút, mang claim `TenantId` / `ClientId` / `Source`; cache phía đối tác 55 phút.

### 10.3 Lớp 3 — Toàn vẹn dữ liệu ⚠️

⚠️ **Đảo ngược hoàn toàn khẳng định của v1.3.**

| Chặng | Ai ký | Ai verify |
|----|----|----|
| T-PayGate ↔ ngân hàng | T-PayGate | T-PayGate — đối tác không tham gia |
| Đối tác → T-PayGate (**mọi** `/public-api/*` trừ `/view/*`) | **Đối tác** | T-PayGate |
| T-PayGate → webhook đối tác | T-PayGate | **Đối tác** |

Chi tiết: [mục 5](#5-ch%E1%BB%AF-k%C3%BD-hmac-sha256--%C3%A1p-d%E1%BB%A5ng-cho-h%E1%BA%A7u-h%E1%BA%BFt-endpoint) và [mục 9.4](#94-ch%E1%BB%AF-k%C3%BD-webhook--%C4%91o%E1%BA%A1n-th%E1%BB%A9-3-kh%C3%B4ng-ph%E1%BA%A3i-source-%E2%9A%A0%EF%B8%8F).

### 10.4 Lớp 4 — Idempotency ⚠️

* ⚠️ **T-PayGate KHÔNG cache** `refTransactionId`, KHÔNG dedupe — xem [mục 8.2](#82-refTransactionId-kh%C3%B4ng-h%E1%BB%81-idempotent-quan-tr%E1%BB%8Dng). Khẳng định "cache 24h" của v1.3 là sai.
* Toàn bộ gánh nặng idempotency nằm ở **phía đối tác**, cả chiều tạo bill lẫn chiều nhận webhook.
* Chống replay webhook: cửa sổ `x-api-time` ±300 giây.

### 10.5 Lớp 5 — Bí mật

* `clientId` / `clientSecret` **không commit Git, không log**.
* Lưu `clientSecret` ở nơi xoay vòng được lúc runtime (secret manager / DB mã hóa), không phải file cấu hình build-time.
* API cấu hình nội bộ **không bao giờ trả** `clientSecret` ra ngoài — chỉ trả cờ "đã cấu hình / chưa".
* ⚠️ **Log khi tích hợp:** log raw body + header rất hữu ích để chẩn đoán lệch chữ ký, nhưng **phải hạ mức hoặc mask trước khi lên production** — raw body chứa số tài khoản, header chứa chữ ký.
* Khi rotate: T-PayGate thông báo trước **30 ngày** ❓.
* Lộ credentials → báo T-PayGate trong 1 giờ; revoke trong 4 giờ ❓.


---

## 11. Mã lỗi

### 11.1 HTTP code ⚠️

| Code | Ý nghĩa |
|----|----|
| 200 | Request tới được endpoint — **vẫn phải kiểm tra** `Error` trong body |
| 400 | Sai format / validate; **hoặc thiếu** `x-config-id` |
| 401 | ❓ Chỉ từ tầng JWT (token hỏng/hết hạn) |
| **403** | ⚠️ **Phần lớn lỗi xác thực & phân quyền**: thiếu header OAuth, tenant không hợp lệ, **chữ ký sai**, **request quá hạn**, `x-config-id` không tồn tại/đã ngắt, access denied |
| 404 | Resource không tồn tại |
| ~~409~~ | ⚠️ **KHÔNG tồn tại.** v1.3 ghi 409 khi trùng `refTransactionId` — thực tế trùng là **hợp lệ**, tạo bill mới |
| 429 | ❓ Rate limit — không thấy cấu hình trong source |
| 5xx | Lỗi nội bộ T-PayGate |

> ⚠️ **Điểm cần sửa gấp nếu đang tích hợp theo v1.3:** hầu hết lỗi bảo mật là **403**, không phải 401. Code bắt riêng 401 để refresh token sẽ **không** nhận diện được lỗi chữ ký, và sẽ refresh token vô ích trong vòng lặp.

### 11.2 Mã lỗi nghiệp vụ ✅ — tập hữu hạn, chỉ 3 giá trị

⚠️ v1.3 liệt kê bảng mã `00`/`01`/`02`/`03`/`05`/`99`. **Không mã nào tồn tại.**

`Error.Code` chỉ nhận đúng **3 giá trị**:

| `Error.Code` | Ý nghĩa |
|----|----|
| `TMT.TPayment.Code:400` | Lỗi input / validate |
| `TMT.TPayment.Code:300` | Lỗi nghiệp vụ (kể cả lỗi xác thực từ middleware) |
| `TMT.TPayment.Code:500` | Lỗi server T-PayGate |

Vì `Code` quá thô (mọi lỗi xác thực đều là `:300`), **phân loại chi tiết phải dựa vào** `Error.Message`. Các message của middleware có tiền tố ổn định `#TPayGate: ` — xem [mục 5.5](#55-m%C3%A3-l%E1%BB%97i-khi-k%C3%BD-sai-%E2%9A%A0%EF%B8%8F).

### 11.3 Khuyến nghị gói lỗi trước khi hiển thị

Không đẩy thẳng message thô ra người dùng cuối. Map sang tập mã ổn định của riêng đối tác:

| Mã đề xuất | Nguyên nhân |
|----|----|
| `TPAYGATE_NOT_CONFIGURED` | Chưa cấu hình credentials |
| `TPAYGATE_NO_BANK` | Không có `configId`, cũng không có bank nào `IsConnected` |
| `TPAYGATE_SIGNATURE` | 403 kèm message `Invalid signature` / `Request expired` — lỗi **cấu hình secret hoặc lệch đồng hồ**, không phải lỗi người dùng |
| `GATEWAY_ERROR` | Lỗi mạng, timeout, response sai shape |
| `PAYMENT_ORDER_FAILED` | Lỗi không xác định |


---

## 12. Khuyến nghị triển khai phía đối tác

| Chủ đề | Khuyến nghị | Vì sao |
|----|----|----|
| **Ký mọi request** | Ký tất cả `/public-api/*` trừ `/view/*` ngay từ đầu | Chỉ ký `/order/bill` sẽ làm hỏng toàn bộ chức năng quản lý bank với 403 |
| **Không gửi Bearer cho** `/bank` | Gọi ẩn danh, hoặc gửi kèm chữ ký | Bearer không kèm chữ ký ⇒ 403 |
| **Không retry tạo bill** | Timeout thì **vấn tin** `get-refTransactionId`, không gọi lại | Không idempotent — retry sinh bill thứ hai |
| Timeout gọi gateway | Đặt cứng, nhỏ hơn ngân sách chờ của client cuối | OAuth đã đo \~32s rồi trả 500 |
| Cache token | TTL 55 phút, key gồm cả base URL | Vượt gián đoạn OAuth; tách Staging/Production |
| Circuit breaker | Nếu dùng, tách theo endpoint | Breaker chung khiến lỗi tạo bill chặn luôn đường lấy token |
| Đồng hồ | Đồng bộ NTP mọi node | Cửa sổ ±5 phút áp dụng **cả hai chiều** |
| Đọc webhook body | Buffering, đọc raw **trước** khi parse, rồi rewind | Parse trước làm rỗng stream; ký lại trên object đã parse sẽ sai |
| Verify webhook | Xác nhận đoạn thứ 3 là `Name` tenant gốc, không phải `source` | Trùng nhau chỉ là tình cờ; đổi tên tenant sẽ gãy hàng loạt |
| Guard số tiền | Bắt buộc khi xử lý webhook | Một `refTransactionId` có thể ứng nhiều bill khác số tiền |
| Trạng thái thanh toán | State machine riêng, do webhook đẩy tiến | `State` là chuỗi tiếng Việt, không ổn định; không có `EXPIRED`/`CANCELED` |
| Hạn QR | **Tự quản lý phía đối tác** | Response tạo bill **không trả** `ExpiredAt` |
| Số VA | Đọc `InfoPayment.AccountNumber` | Không có `VirtualAccount` ở cấp gốc |
| Cập nhật UI | Realtime + polling dự phòng (\~5s) | Webhook về server, không về browser |


---

## 13. Testing & Go-live

### 13.1 Bộ test data UAT

* `clientId` + `tenantId` + `source` + `clientSecret` UAT
* ⚠️ `Name` của tenant gốc và biết tenant mình là **cha hay con**
* `configId` dựng sẵn (nếu dùng mô hình A)
* 1 tài khoản test ngân hàng đã đăng ký sẵn
* Số CMND + SĐT đã đăng ký dịch vụ test
* OTP cố định (nếu áp dụng)
* Postman collection **kèm pre-request script sinh chữ ký**

### 13.2 Testcase tối thiểu

| # | Kịch bản | Pass criteria |
|----|----|----|
| 1 | OAuth thành công | `Data.AccessToken` có giá trị |
| 2 | OAuth thiếu 1 trong 3 header | **403**, message `#TPayGate: x-... is required` |
| 3 | OAuth sai bộ ba | **403** `x-tenant-id không hợp lệ` |
| 4 | OAuth trả non-JSON | Client bắt được, log body, **không crash** |
| 5 | `GET /bank` **không** Bearer | 200, trả list |
| 6 | `GET /bank` **có** Bearer, **không** ký | **403** `Invalid signature` (xác nhận hiểu đúng cạm bẫy) |
| 7 | `/config-bank/list` ký dạng **GET** (không body) | 200 |
| 8 | `/config-bank/list` ký nhầm dạng POST (kèm body) | 403 — xác nhận công thức GET khác POST |
| 9 | Connect bank có OTP, OTP đúng | `IsConnected=true`, có `VaNumber` |
| 10 | Connect bank **không** OTP | `VaNumber` lấy qua `/config-bank/list` |
| 11 | Gọi `/connect` 2 lần | Không tạo record trùng phía đối tác |
| 12 | Tạo bill thiếu `x-config-id` | **400** `configBankId is required` |
| 13 | Tạo bill với `x-config-id` đã disconnect | **403** `does not exist or is disconnected` |
| 14 | Tạo bill, chữ ký đúng | `Data.BillCode` + `Data.QRBase64` có giá trị |
| 15 | Tạo bill, `x-api-time` lệch > 5 phút | **403** `Request expired` |
| 16 | **Tạo bill 2 lần cùng** `refTransactionId` | ⚠️ **Trả về 2** `BillCode` KHÁC NHAU — xác nhận không idempotent |
| 17 | Đọc số VA từ response | Lấy đúng ở `InfoPayment.AccountNumber` |
| 18 | Kiểm tra `ExpiredAt` | **Không tồn tại** — đối tác tự quản lý hạn |
| 19 | `QrDataURL` rỗng | UI fallback render từ `QRBase64` |
| 20 | Vấn tin `get-refTransactionId` sau khi tạo 2 bill | Trả **mảng 2 phần tử** |
| 21 | Quét & thanh toán | Webhook về đúng `RefTransactionId` |
| 22 | Webhook chữ ký sai | Đối tác từ chối, không đổi trạng thái |
| 23 | Webhook `x-api-time` lệch > 5 phút | Đối tác từ chối |
| 24 | **Webhook xử lý OK** | Trả 200 với `MessageError` **rỗng/null** |
| 25 | **Trả** `{"MessageError": true}` (kiểu v1.3) | Xác nhận T-PayGate coi là **LỖI** và retry — để chứng minh phải sửa |
| 26 | Webhook **sai số tiền** | Giao dịch **không** hoàn tất; log; vẫn ack thành công |
| 27 | Gửi lại đúng webhook lần 2 | Không double-process |
| 28 | Webhook gây exception nội bộ | Vẫn ack thành công, có log lỗi |
| 29 | Thanh toán thiếu tiền | `State` = `Thanh toán một phần`, `AmountPaid` < `Amount` |
| 30 | Disconnect | `IsConnected=false` |
| 31 | Đối soát T+1 | File khớp webhook đã nhận |

### 13.3 Go-live checklist

#### Phía đối tác

- [ ] Hoàn tất 100% testcase UAT
- [ ] **Chữ ký áp dụng cho mọi endpoint** `/public-api/*` trừ `/view/*`, đúng công thức GET vs POST
- [ ] **Không gửi Bearer kèm** `/bank` nếu không ký
- [ ] Endpoint webhook PROD chạy + cert SSL hợp lệ
- [ ] **Xác nhận đúng 3 đoạn ký webhook** (ClientId gốc / TenantId / **Name gốc**)
- [ ] **Phản hồi webhook có** `MessageError` rỗng khi thành công (không dùng mẫu `{"MessageError": true}` của v1.3)
- [ ] Đồng hồ server đồng bộ NTP
- [ ] **Không có đường code nào retry** `POST /order/bill`
- [ ] Guard đối chiếu số tiền khi xử lý webhook
- [ ] Tự quản lý thời hạn QR (không trông chờ `ExpiredAt`)
- [ ] Đọc số VA từ `InfoPayment.AccountNumber`
- [ ] Xử lý **403** như lỗi cấu hình/chữ ký, không phải lỗi token
- [ ] IP outbound PROD đã gửi cho T-PayGate
- [ ] Log ở tầng verify webhook đã hạ mức / mask
- [ ] Email/group oncall xác nhận; plan rollback

#### Phía T-PayGate — cần trả lời trước go-live

- [ ] Cấp `clientId`/`tenantId`/`clientSecret` PROD (riêng UAT)
- [ ] **Xác nhận tenant là cha hay con, và** `ClientSecret` được cấp thuộc cấp nào
- [ ] **Cung cấp chính xác** `Name` của tenant gốc (dùng ký webhook)
- [ ] Cấp `configId` PROD (nếu dùng mô hình A)
- [ ] **Chính sách retry webhook thực tế** (số lần, khoảng cách) — không xác minh được từ source
- [ ] **Có rate limit không, ngưỡng bao nhiêu** (HTTP 429)
- [ ] **Có cơ chế hết hạn / hủy bill không** — public-api không có
- [ ] **Kế hoạch (nếu có) cho webhook envelope** `EventType` — hiện chưa hiện thực
- [ ] Whitelist IP đối tác; bật monitoring/alert; job đối soát hàng ngày


---

## 14. Phụ lục

### 14.1 Tóm tắt endpoints

| Bước | Method | Path | Auth | Ký? | Header thêm |
|----|----|----|----|----|----|
| OAuth | POST | `/api/v1/oauth/token` | — | ❌ | `x-client-id`, `x-tenant-id`, `x-source` |
| Connect (UI) | GET | `/api/v1/public-api/view/connect` | Token qua query | ❌ | — |
| Connect (UI OTP) | POST | `/api/v1/public-api/view/otp` | Token qua query | ❌ | — |
| Bank list | GET | `/api/v1/public-api/bank` | **Ẩn danh** | ❌ nếu ẩn danh / ✅ nếu gửi Bearer | — |
| Connect (API) | POST | `/api/v1/public-api/config-bank/connect` | Bearer | ✅ POST | `x-api-time`, `x-signature` |
| Confirm OTP | POST | `/api/v1/public-api/config-bank/confirm` | Bearer | ✅ POST | `x-api-time`, `x-signature` |
| List config | GET | `/api/v1/public-api/config-bank/list` | Bearer | ✅ **GET** | `x-api-time`, `x-signature` |
| Disconnect | POST | `/api/v1/public-api/config-bank/disconnect` | Bearer | ✅ POST | `x-api-time`, `x-signature` |
| Tạo bill | POST | `/api/v1/public-api/order/bill` | Bearer | ✅ POST | `x-config-id`, `x-api-time`, `x-signature` |
| Vấn tin theo ref | GET | `/api/v1/public-api/order/get-refTransactionId` | Bearer | ✅ **GET** | `x-config-id`, `x-api-time`, `x-signature` |
| Vấn tin theo billCode | GET | `/api/v1/public-api/order/get-billCode` | Bearer | ✅ **GET** | `x-config-id`, `x-api-time`, `x-signature` |

### 14.2 Bảng đối chiếu v1.3 → thực tế

| Ngữ cảnh | v1.3 ghi | Thực tế trong source |
|----|----|----|
| Envelope response | `{success, results}` | `{Error, Data}` |
| OAuth request | chỉ form body | **3 header bắt buộc** + form camelCase |
| OAuth response | `accessToken` (root) | `Data.AccessToken` |
| OAuth sai credentials | HTTP 401 | **HTTP 403** |
| Phạm vi ký | "đối tác không phải ký" | **mọi** `/public-api/*` trừ `/view/*` |
| Công thức ký GET | (không nêu) | **bỏ đoạn body** — 3 dấu `_` |
| Chữ ký sai | (không nêu) | **HTTP 403** |
| `GET /bank` | không cần auth | **ẩn danh**; kèm Bearer mà không ký ⇒ 403 |
| Bill — số VA | `vaNumber` (root) | `InfoPayment.AccountNumber` |
| Bill — chuỗi EMV | `qrContent` | `QRBase64` (không phải base64) |
| Bill — ảnh QR | `qrImageBase64` | `QrDataURL` |
| Bill — hết hạn | `expiredAt` | ⚠️ **không tồn tại** |
| Bill — thông tin CK | (không nêu) | `InfoPayment` {Bank, AccountNumber, AccountName, Description} |
| Trùng `refTransactionId` | HTTP 409, không double-process | ⚠️ **hợp lệ, sinh bill mới** |
| Idempotency cache 24h | có | ⚠️ **không có** |
| Trạng thái bill | `CREATED`/`PAID`/`EXPIRED`/`CANCELED` | `N/A`/`Đơn mới`/`Thanh toán một phần`/`Đã thanh toán` |
| Webhook envelope v2 | `EventType`/`Data` | ⚠️ **không hiện thực** |
| Webhook — đoạn ký thứ 3 | (không nêu) | ⚠️ `Name` tenant gốc, không phải `source` |
| Webhook ack | `{"MessageError": true}` | ⚠️ `MessageError` phải RỖNG khi thành công |
| Webhook timeout | 10s PROD / 30s UAT | **30s cả hai** |
| Mã lỗi nghiệp vụ | `00`…`99` | `TMT.TPayment.Code:300/400/500` |

### 14.3 Mã ngân hàng

Lấy động qua `GET /api/v1/public-api/bank`, dùng `Code` (chuẩn NAPAS) làm `bankCode`. **Không hardcode.**

### 14.4 Postman collection

Environment variable cần có: `baseUrl`, `clientId`, `tenantId`, `source`, `clientSecret`, `configId`, `accessToken`.

> ⚠️ Collection **phải kèm pre-request script** sinh `x-api-time` + `x-signature` cho **mọi** request `/public-api/*` (trừ `/view/*`), với **hai biến thể GET và POST**. Thiếu script này, gần như mọi request sẽ trả 403 và đối tác dễ kết luận nhầm là sai credentials.

### 14.5 Hỗ trợ

Liên hệ qua kênh đã đăng ký lúc onboarding (email / group chat / hotline).


---

## 15. Lịch sử thay đổi

| Ngày | Phiên bản | Thay đổi |
|----|----|----|
| 2026-05-05 | 1.0 | Khởi tạo outline |
| 2026-05-05 | 1.1 | Cập nhật URL prod, restructure theo 4 bước |
| 2026-05-05 | 1.2 | Tài liệu standalone, ẩn thông tin bank cụ thể |
| 2026-05-05 | 1.3 | Bỏ chi tiết verify chữ ký webhook (T-PayGate tự xử lý nội bộ) |
| 2026-07-30 | 2.0 | Hiệu chỉnh theo contract quan sát được từ một tích hợp production |
| 2026-07-30 | **2.1** | **Đối chiếu trực tiếp source code T-PayGate.** Mở rộng phạm vi chữ ký ra **mọi** `/public-api/*` trừ `/view/*`, bổ sung **công thức GET khác POST** và quy tắc tenant cha/con (mục 5); sửa mã lỗi bảo mật từ 401 thành **403**; xác định `Error.Code` chỉ có **3 giá trị**; ghi nhận `GET /bank` **ẩn danh** và cạm bẫy Bearer-không-ký; sửa response tạo bill — **không có** `VirtualAccount`/`ExpiredAt`, số VA nằm ở `InfoPayment.AccountNumber`; xác định `refTransactionId` **không idempotent, không có 409** (dedupe đã bị comment out); thay bảng trạng thái bill bằng **4 chuỗi tiếng Việt** thực tế; xác nhận **không tồn tại webhook envelope v2 /** `EventType`; phát hiện đoạn thứ 3 của chữ ký webhook là `Name` tenant gốc chứ không phải `source`; sửa payload ack webhook — `MessageError` phải rỗng, mẫu `{"MessageError": true}` của v1.3 gây retry sai |
# Cấp phát thông số cho đối tác thứ 3

# T-PayGate — Quy trình cấp phát thông số cho đối tác thứ 3

> Phiên bản: 1.0 Cập nhật: 2026-07-30 Dành cho: người vận hành nền tảng (`system_admin`) và dev hỗ trợ onboarding đối tác

Tài liệu này trả lời: **đối tác cần 6 thông số để cấu hình, bên mình cấp chúng ở đâu và theo thứ tự nào.**

Góc nhìn ngược lại — đối tác dùng 6 thông số đó ra sao — nằm ở `docs/tpaygate-integration-guide.md`. Tài liệu này là mặt trong: màn hình nào, endpoint nào, cột DB nào.

**Đọc [mục 6](#6-c%E1%BA%A1m-b%E1%BA%ABy) trước khi cấp bộ đầu tiên.** Có 3 chỗ sai là phải xoá tenant tạo lại, và 1 chỗ sai là webhook chết âm thầm.


---

## 0. Mục lục


1. [Sáu thông số và nguồn gốc](#1-s%C3%A1u-th%C3%B4ng-s%E1%BB%91-v%C3%A0-ngu%E1%BB%93n-g%E1%BB%91c)
2. [Quy trình cấp phát](#2-quy-tr%C3%ACnh-c%E1%BA%A5p-ph%C3%A1t)
3. [Bàn giao cho đối tác](#3-b%C3%A0n-giao-cho-%C4%91%E1%BB%91i-t%C3%A1c)
4. [Nghiệm thu](#4-nghi%E1%BB%87m-thu)
5. [Sửa / cấp lại sau khi đã cấp](#5-s%E1%BB%ADa--c%E1%BA%A5p-l%E1%BA%A1i-sau-khi-%C4%91%C3%A3-c%E1%BA%A5p)
6. [Cạm bẫy](#6-c%E1%BA%A1m-b%E1%BA%ABy)


---

## 1. Sáu thông số và nguồn gốc

Tất cả nằm ở **hai bảng**: `TenantMatchings` (danh tính đối tác) và `ConfigBanks` (kết nối ngân hàng của merchant).

| Thông số đối tác cấu hình | Cột DB | Ai sinh | Sửa sau được? |
|----|----|----|----|
| **Client ID** | `TenantMatchings.ClientId` | Code tự set `= Name` của tenant gốc | Không — phải tạo tenant mới |
| **Tenant ID** | `TenantMatchings.TenantId` | Id của ABP `Tenant` tạo kèm | Không |
| **Source** | `TenantMatchings.Name` | Mình đặt khi tạo tenant gốc / tenant con | Không — có unique index |
| **Client Secret** | `TenantMatchings.ClientSecret` | Tự sinh `RandomSecretApp()` | Có, bằng SQL ([mục 5](#5-s%E1%BB%ADa--c%E1%BA%A5p-l%E1%BA%A1i-sau-khi-%C4%91%C3%A3-c%E1%BA%A5p)) |
| **Webhook URL** | `TenantMatchings.UrlWebhook` | **Mình nhập** khi tạo | Có, bằng SQL |
| **Config ID** | `ConfigBanks.Id` | Tự sinh khi đối tác gọi Connect | Không nhập tay — mỗi kết nối một Id |

Ba thông số dùng để **định danh** (`ClientId` + `TenantId` + `Source`) phải khớp **đúng một dòng** `TenantMatchings`, nếu không thì 403 ngay ở middleware:

```
BasicAuthMiddlewareService.GetTenantMatchingAsync(tenantId, clientId, name)
    → WHERE ClientId = clientId AND TenantId = tenantId AND Name = source AND NOT IsDeleted
```

`Source` **chính là cột** `Name` — đây là chỗ dễ hiểu sai nhất, xem [mục 6.1](#61-source-kh%C3%B4ng-ph%E1%BA%A3i-nh%C3%A3n-k%C3%AAnh-t%E1%BB%B1-do).

`ClientSecret` **không dùng để lấy token** — endpoint `/api/v1/oauth/token` không kiểm secret (đoạn kiểm bị comment trong `OauthController`). Secret chỉ dùng để ký HMAC-SHA256:

| Chiều | Message ký | Khoá |
|----|----|----|
| Đối tác → mình (`/api/v1/public-api/*`) | `{clientId}_{tenantId}_{source}_{timestamp}_{body}` (POST)<br>`{clientId}_{tenantId}_{source}_{timestamp}` (GET) | `ClientSecret` của tenant **gốc** |
| Mình → đối tác (webhook) | `{rootClientId}_{tenantId}_{rootName}_{timestamp}_{body}` | `ClientSecret` của tenant **gốc** |

Cả hai chiều gửi kèm `x-signature` + `x-api-time`; chiều vào có cửa sổ chống replay **±5 phút**.

Code liên quan:

| Việc | Vị trí |
|----|----|
| Tạo tenant gốc / tenant con | `src/TMT.TPayment.Application/TenantMatching/TenantMatchingService.cs` |
| API quản trị | `src/TMT.TPayment.HttpApi/Controllers/V1/TenantMatchingController.cs` — route ghim cứng `api/v1/TenantMaching` |
| Tra cứu định danh | `src/TMT.TPayment.Application/BasicAuthMiddlewareService.cs` |
| Gác `/api/v1/oauth` | `src/TMT.TPayGate.HttpApi.Host/Authorization/PublicApiOauthMiddleware.cs` |
| Gác `/api/v1/public-api` + verify sign | `src/TMT.TPayGate.HttpApi.Host/Authorization/PublicApiMiddleware.cs` |
| Sinh Config ID | `src/TMT.TPayment.Application/PaymentGate/ConfigBanks/PayGate_ConfigBankService.cs` |
| Gửi webhook + ký | `src/TMT.TPayment.Application/PaymentGate/Callback/TPayGateCallbackSender.cs` |
| Màn hình Angular | `angular/src/app/modules/main-app/pages/tenant/`, `.../tenant-child/` |


---

## 2. Quy trình cấp phát

### 2.0 Thu thập từ đối tác (làm trước khi bấm gì)

| Cần | Vì sao |
|----|----|
| **Tên client** (ví dụ `wicloud`) | Thành `Name` + `ClientId`, **không sửa được sau đó**, unique toàn hệ thống |
| **URL webhook** HTTPS public | Điền ngay lúc tạo — bỏ trống sẽ làm callback chết, xem [mục 6.4](#64-tenant-g%E1%BB%91c-thi%E1%BA%BFu-urlwebhook-l%C3%A0m-callback-ch%E1%BA%BFt) |
| **Email tài khoản quản trị** | Phải **đã tồn tại trên identity server**; màn hình tạo tenant chỉ chọn từ danh sách có sẵn |
| Danh sách ngân hàng cần dùng | Chỉ ngân hàng có `Bank.IsConnectProvider = true` mới kết nối được |
| IP outbound của đối tác | Whitelist trước go-live PROD |
| Có cần nhiều kênh / nhiều shop? | Quyết định có tạo tenant con hay không ([mục 2.3](#23-tu%E1%BB%B3-ch%E1%BB%8Dn--t%E1%BA%A1o-tenant-con-cho-t%E1%BB%ABng-k%C3%AAnh--shop)) |

Làm **riêng cho từng môi trường**: Staging và Production là hai DB khác nhau, hai bộ credential khác nhau.

### 2.1 Bảo đảm tài khoản quản trị tồn tại

Đăng nhập admin portal bằng tài khoản `system_admin` → màn hình **User app** (`/user-app`). Nếu email đối tác chưa có trong danh sách thì tạo trước; tenant tạo sau đó sẽ gắn user này làm admin.

### 2.2 Tạo tenant gốc

Màn hình **Tenant** (`/tenant`, role `system_admin`) → nút **Thêm**. Endpoint: `POST api/v1/TenantMaching`, yêu cầu quyền ABP `AbpTenantManagement.Tenants.Create`.

Ba field:

| Field trên form | Ý nghĩa thật |
|----|----|
| **Tenant Name** | Thành `Name` **và** `ClientId`. Với tenant gốc thì `Source ≡ Client ID` |
| **URL Callback** | Thành `UrlWebhook` — form không bắt buộc nhưng **thực tế là bắt buộc** |
| **Management Account** | User trên identity server sẽ làm admin của tenant |

Một lần bấm Lưu chạy 6 việc, không có transaction bao ngoài:


1. Tạo ABP `Tenant` (Id của nó chính là **Tenant ID** cấp cho đối tác)
2. Add user vào app
3. Gán role `admin` cho user trong tenant
4. Tạo `AccountUsersMatching`
5. Insert `TenantMatchings` — sinh `ClientSecret`, set `ClientId = Name`
6. Grant \~18 permission cho user (`OnSetPermissionRoleAdmin`)

Modal sau khi thành công hiện `ClientId` và `ClientSecret`.

> ⚠️ **Copy** `ClientSecret` ngay tại đây. Không màn hình nào hiện lại nó — danh sách tenant chỉ có `Id` + `AppName`, danh sách tenant con chỉ có `Id`/`ClientName`/`ClientId`. Mất là phải đọc DB.

Lấy **Tenant ID**: cột `Id` trên danh sách là Id của `TenantMatchings`, **không phải** Tenant ID. Tenant ID là `TenantMatchings.TenantId`:

```sql

SELECT "Name", "ClientId", "ClientSecret", "TenantId", "UrlWebhook"
FROM "TenantMatchings"
WHERE "Name" = 'wicloud' AND "TenantMatchingParentId" IS NULL;
```

### 2.3 (Tuỳ chọn) Tạo tenant con cho từng kênh / shop

Dùng khi đối tác cần tách nhiều kênh hoặc nhiều shop dưới **cùng một** `ClientId`.

Màn hình **Tenant child** (`/tenant-child`) — role `admin`/`user` + feature key `TenantChild`, tức **phải đăng nhập bằng tài khoản của chính tenant đó**, không phải `system_admin`. Lý do: `CreateChildAsync` tìm tenant cha qua `TMTCurrentTenant.Id`, sai context là gắn con vào cha khác.

Endpoint: `POST api/v1/TenantMaching/child`, hai field `Name` + `UrlWebhook`.

Tenant con thừa hưởng:

| Thông số | Giá trị |
|----|----|
| `ClientId` | **của cha** |
| `TenantId` | **của cha** |
| `Name` | tên con → đây là **Source** đối tác truyền lên |
| `UrlWebhook` | riêng của con, rỗng thì fallback về của cha |
| `ClientSecret` | có sinh riêng nhưng **không được dùng** — ký luôn bằng secret của cha |

### 2.4 Kiểm tra trước khi bàn giao

```sql
-- Cha + toàn bộ con, kèm những cột hay quên điền

SELECT t."Name" AS source, t."ClientId", t."TenantId", t."UrlWebhook",
       CASE WHEN t."TenantMatchingParentId" IS NULL THEN 'gốc' ELSE 'con' END AS loai

FROM "TenantMatchings" t

WHERE (t."ClientId" = 'wicloud') AND NOT t."IsDeleted"
ORDER BY t."TenantMatchingParentId" NULLS FIRST;
```

Checklist: tenant gốc có `UrlWebhook` khác NULL · `Name` không phải `TPOS` · đúng môi trường · secret đã lưu vào nơi quản lý bí mật của team.


---

## 3. Bàn giao cho đối tác

Gửi qua kênh bảo mật (không Slack/email thường cho secret). Mẫu:

| Thông số | Giá trị |
|----|----|
| Base URL | `https://t-paygate.tpos.dev` (Staging) / `https://t-paygate.tpos.app` (Production) |
| Client ID | `wicloud` |
| Tenant ID | `3a21bca8-…` |
| Source | `wicloud` (hoặc tên tenant con nếu có cấp) |
| Client Secret | `abcd1234EF` |
| Webhook URL bên mình sẽ gọi | `https://…` (nhắc lại để đối tác đối chiếu) |

Kèm 4 lưu ý:


1. `Source` là **giá trị cố định** do mình cấp, không phải nhãn kênh đối tác tự đặt.
2. `/api/v1/oauth/token` cần **cả** header `x-client-id`, `x-tenant-id`, `x-source` **và** form body `clientId`, `tenantId`, `source`. Thiếu header là 403 trước khi vào controller.
3. Mọi call `/api/v1/public-api/*` (trừ `/view/*`) cần `x-signature` + `x-api-time`; lệch giờ > 5 phút là 403.
4. **Config ID** đối tác tự lấy được sau khi Connect ngân hàng — mình không cấp.


---

## 4. Nghiệm thu

Chạy ngay sau khi cấp, trước khi đối tác vào việc.

```bash

BASE=https://t-paygate.tpos.dev

CLIENT_ID=wicloud

TENANT_ID=3a21bca8-...
SOURCE=wicloud

SECRET=abcd1234EF

# 1) OAuth — kỳ vọng accessToken + expiresIn 3600

TOKEN=$(curl -s -X POST "$BASE/api/v1/oauth/token" \
  -H "x-client-id: $CLIENT_ID" -H "x-tenant-id: $TENANT_ID" -H "x-source: $SOURCE" \
  -d "clientId=$CLIENT_ID" -d "tenantId=$TENANT_ID" -d "source=$SOURCE" \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["accessToken"])')

# 2) Gọi thử 1 API GET có kiểm chữ ký — kỳ vọng danh sách ngân hàng, KHÔNG phải 403

TS=$(date -u +%s)
SIGN=$(printf '%s_%s_%s_%s' "$CLIENT_ID" "$TENANT_ID" "$SOURCE" "$TS" \
  | openssl dgst -sha256 -hmac "$SECRET" -hex | sed 's/^.*= //')

curl -s "$BASE/api/v1/public-api/bank" \
  -H "Authorization: Bearer $TOKEN" -H "x-signature: $SIGN" -H "x-api-time: $TS"
```

Đọc lỗi:

| Lỗi | Nguyên nhân |
|----|----|
| `x-tenant-id không hợp lệ` | Bộ ba ClientId/TenantId/Source không khớp dòng nào — thường là Source sai |
| `Invalid signature` | Sai secret, hoặc tenant con mà ký bằng secret của con thay vì của cha |
| `Request expired` | Đồng hồ lệch > 5 phút |
| `configBankId is required` | Gọi nhóm `/order` mà thiếu header `x-config-id` |

Bước cuối là **webhook chiều ra**: cho đối tác Connect một ngân hàng UAT, tạo bill, chuyển tiền thử, xác nhận đối tác nhận được callback và verify được `x-signature`. Nếu tenant con nhận callback, nhắc đối tác message ký dùng `Name` của tenant **gốc** ở vị trí source ([mục 6.6](#66-source-trong-sign-webhook-kh%C3%A1c-sign-request)).


---

## 5. Sửa / cấp lại sau khi đã cấp

**Không có endpoint update.** `TenantMatchingController` chỉ có `GET`, `POST`, `DELETE`, `POST/GET/DELETE child`. Mọi thay đổi sau khi tạo là SQL trực tiếp.

```sql
-- Đổi webhook URL

UPDATE "TenantMatchings" SET "UrlWebhook" = 'https://...'
WHERE "Name" = 'wicloud';

-- Rotate secret (dùng chuỗi do team sinh, đừng bắt chước format 10 ký tự của code)
UPDATE "TenantMatchings" SET "ClientSecret" = '<secret mới>'
WHERE "Name" = 'wicloud' AND "TenantMatchingParentId" IS NULL;
```

Rotate secret **không có grace period**: đổi xong là mọi request đang ký bằng secret cũ trả 403 ngay, và webhook mình gửi ra cũng ký bằng secret mới. Phải hẹn giờ với đối tác.

Tạm khoá một đối tác thì **đừng** dùng `DELETE api/v1/TenantMaching/{tenantId}` — nó xoá luôn ABP tenant, gỡ user khỏi app, revoke permission và xoá `AccountUsersMatching`. Đó là thao tác off-board, không phải tạm khoá. Xoá tenant con thì an toàn hơn: `DELETE api/v1/TenantMaching/child/{id}` chỉ soft-delete.


---

## 6. Cạm bẫy

### 6.1 `Source` không phải nhãn kênh tự do

`docs/tpaygate-integration-guide.md` mục 3.2 mô tả `source` là "phân kênh (VD: `WEB`, `MOBILE`, `POS`)". Đọc vậy dễ tưởng đối tác muốn truyền gì cũng được. Thực tế `source` bị so **bằng** với `TenantMatchings.Name`; truyền `WEB` khi không có dòng nào tên `WEB` thuộc đối tác đó là **403 tại middleware**.

Muốn đối tác dùng nhiều source thì phải tạo đúng số lượng tenant con tương ứng.

### 6.2 `Name` unique toàn hệ thống

`TPaymentDbContext` đặt unique index trên `TenantMatchings.Name`. Nghĩa là nếu đối tác A đã lấy tên `POS` thì **không đối tác nào khác** đặt được tên đó nữa.

Luôn đặt tên có namespace: `wicloud`, `wicloud.pos`, `wicloud.web` — đúng tinh thần cách tenant con hiện hữu đang đặt theo domain (`example.tpos.vn`).

### 6.3 `ClientSecret` chỉ hiện một lần

Modal tạo tenant là **chỗ duy nhất** hiển thị secret. Cả hai màn hình danh sách đều không có cột secret (trong `tenant-list.component.html` cột đó đã bị comment). Lưu vào nơi quản lý bí mật của team ngay khi tạo.

### 6.4 Tenant gốc thiếu `UrlWebhook` làm callback chết

`TPayGateCallbackSender.GetUrlCallback` (dòng 59) đọc:

```csharp

return tenantDto.UrlWebhook ?? tenantDto.TenantMatchingParent.UrlWebhook ?? string.Empty;
```

Với tenant **gốc**, `TenantMatchingParent` là `null`. `UrlWebhook` để trống → NullReferenceException lúc gửi callback, không phải lỗi cấu hình rõ ràng. Form "URL Callback" không bắt buộc nên rất dễ bỏ qua. **Luôn điền.**

### 6.5 Tenant con ký bằng secret của cha

`PublicApiMiddleware` dòng 67-70: con có parent thì verify bằng `TenantMatchingParent.ClientSecret`. Secret riêng của con vẫn được sinh và lưu trong DB nhưng **không dùng ở đâu cả** — đưa nó cho đối tác là gây 403 hàng loạt.

Chỉ bàn giao **một** secret cho mỗi đối tác: secret của tenant gốc.

### 6.6 `source` trong sign webhook khác sign request

| Chiều | Vị trí thứ 3 trong message |
|----|----|
| Request vào | `source` của request = `Name` của tenant con |
| Webhook ra | `Name` của tenant **gốc** (`TPayGateCallbackSender` dòng 126) |

Với đối tác chỉ dùng tenant gốc thì hai giá trị trùng nhau nên không ai nhận ra. Đối tác có tenant con mà verify webhook bằng source của mình sẽ luôn lệch chữ ký.

### 6.7 Staging và Production cấp riêng

Hai DB độc lập → `TenantId` (Id của ABP tenant) khác nhau, secret khác nhau. Đối tác phải giữ hai bộ cấu hình. Đừng copy credential UAT sang PROD.

### 6.8 Đừng đặt tên đối tác là `TPOS`

`GetUrlCallback` có nhánh đặc biệt: nếu tenant cha tên `TPOS`, URL webhook **không** lấy từ `UrlWebhook` mà suy ra từ `Name` của tenant con — `https://{Name}/webhooks/tpaygate/receive`, và chỉ khi tên con match `.tpos.dev|localhost|tpos.vn`. Đây là đường riêng cho hệ thống TPOS nội bộ.

### 6.9 Secret sinh tự động có entropy thấp

`RandomSecretApp()` tạo 10 ký tự (4 chữ thường + 4 số + 2 chữ hoa) bằng `System.Random`, không phải RNG mã hoá. Với đối tác production nên **rotate ngay sau khi tạo** bằng chuỗi tự sinh đủ mạnh ([mục 5](#5-s%E1%BB%ADa--c%E1%BA%A5p-l%E1%BA%A1i-sau-khi-%C4%91%C3%A3-c%E1%BA%A5p)), và luôn kết hợp whitelist IP.

### 6.10 Route quản trị viết sai chính tả có chủ đích

`api/v1/TenantMaching` (thiếu chữ `t`). Ghim cứng, Angular gọi thẳng vào. Đừng "sửa lỗi typo" — xem `docs/onboarding-overview.md` mục 14.1.


---

## 7. Đọc tiếp

| Tài liệu | Khi nào cần |
|----|----|
| `docs/tpaygate-integration-guide.md` | Hợp đồng API phía đối tác: OAuth → Connect → Config → Bill, webhook, mã lỗi |
| `docs/onboarding-overview.md` mục 10 | Toàn cảnh 3 cơ chế xác thực của repo |
| `docs/vault-configuration-guide.md` | `JwtPayGate:Key` và các secret hạ tầng |
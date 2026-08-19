---
type: wireframe-html-index
feature: split-order
status: draft
device: mobile
device_width: 375
updated: 2026-07-21
links:
  - docs/split-order/srs/spec.md
  - docs/split-order/srs/screens.md
  - docs/split-order/srs/flows.md
changelog:
  - 2026-07-21 | /wireframe-html | [all] generated 2 flows: split-order-flow-1, split-order-flow-2
---

# split-order — HTML Wireframes

> HTML wireframe files (B&W, static) cho tính năng Tách đơn. Device: Mobile 375px.
> **Cửa vào:** mở `split-order-wireframe.html` trong trình duyệt (double-click).

## Flows

| # | Flow | File | Screens (theo thứ tự) | Status | Updated |
|---|------|------|-----------------------|--------|---------|
| 1 | Chọn sản phẩm tách | [split-order-flow-1.html](split-order-flow-1.html) | S1 Chi tiết đơn → S1b Disabled edge → S2 Chọn SP → S2b Lỗi toàn bộ | draft | 2026-07-21 |
| 2 | Preview & Kết quả | [split-order-flow-2.html](split-order-flow-2.html) | S3 Preview TC → S3b Dialog xác nhận → S3c Loading → S4 Thành công → S4b Thất bại | draft | 2026-07-21 |

**Tổng:** 9 screens (4 happy path + 3 edge/error + 2 loading/confirm states)

## Cách xem

1. **Double-click** `split-order-wireframe.html` → mở trong trình duyệt
2. Sidebar trái: TOC điều hướng theo flow → screen
3. Tab "Flow Map": sơ đồ điều hướng click được
4. Tab "Wireframe": iframe load file flow tương ứng

## Links upstream

- [spec.md](../srs/spec.md) — Business Rules (16 BR)
- [screens.md](../srs/screens.md) — Screen Specifications
- [flows.md](../srs/flows.md) — Activity & Sequence Diagrams

## Changelog

> Newest on top.
- 2026-07-21 | /wireframe-html | [split-order-flow-1] 4 screens — Chi tiết đơn + Chọn SP tách
- 2026-07-21 | /wireframe-html | [split-order-flow-2] 5 screens — Preview TC + Kết quả

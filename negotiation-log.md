# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: Pair 02 (Core Business ↔ AI Vision)
- Product: Smart Campus Operations Platform
- Provider: AI Vision (Nhóm A4/B4)
- Consumer: Core Business (Nhóm A2)
- Phiên: v1.0
- Ngày: 18/05/2026

---

## Issue #1

- Raised by: Consumer
- Endpoint: `POST /vision/face-match`
- Concern: Định dạng dữ liệu truyền nhận cho khuôn mặt khi Face-Match để tối ưu hiệu năng.
- Proposal: Core đề xuất gửi đường dẫn ảnh (`imageRef` dạng URL string) thay vì gửi mảng vector toán học phức tạp (`faceEmbedding`).
- Resolution: Accepted
- Rationale: AI Vision đã tích hợp sẵn module trích xuất đặc trưng ảnh (feature extraction). Việc gửi URL ảnh giúp giảm đáng kể kích thước request payload truyền qua mạng và giảm tải tính toán cho Core Backend.
- Impact: Hệ thống AI Vision cần bổ sung thêm bước tự tải ảnh từ URL do Core cung cấp về bộ nhớ trước khi nạp vào model AI để xử lý.

---

## Issue #2

- Raised by: Provider
- Endpoint: `POST /vision/face-match`
- Concern: Cấu trúc phản hồi lỗi khi ảnh tải lên không có thực thể người hoặc bị mờ nhòe nghiêm trọng.
- Proposal: AI Vision đề xuất trả về HTTP 200 kèm thuộc tính `matched: false`. Core Business không đồng ý, đề xuất phải trả về lỗi `4xx` để phân biệt với việc so khớp thất bại của một người dùng hợp lệ.
- Resolution: Modified
- Rationale: Bản chất của việc ảnh không có mặt người là lỗi dữ liệu đầu vào không thể xử lý (Unprocessable Content), chứ không phải là một phiên đối sánh hợp lệ nhưng không trùng khớp. Phải dùng lỗi 4xx để Core nhận biết ngay và bắt người dùng chụp lại ảnh.
- Impact: Hệ thống thống nhất trả về mã lỗi `422 Unprocessable Entity` và cấu trúc body lỗi phải tuân thủ chuẩn Problem Details (RFC 7807).

---

## Issue #3

- Raised by: Consumer
- Endpoint: `POST /vision/face-match`
- Concern: Quản lý tính định danh của trường `personId` trong trường hợp kết quả so khớp trả về là thất bại (`matched: false`).
- Proposal: Thiết lập trường `personId` thành kiểu dữ liệu lai (Union Type) cho phép nhận giá trị chuỗi hoặc giá trị null.
- Resolution: Accepted
- Rationale: Tận dụng tính năng mới của OpenAPI 3.1. Việc cho phép trường này nhận giá trị `null` giúp cấu trúc JSON phản hồi tường minh, tránh việc Core Business parse nhầm một chuỗi rỗng `""` thành một ID nhân sự hợp lệ.
- Impact: Trong file `openapi.yaml`, trường `personId` sẽ được khai báo là `type: [string, "null"]`. Phía Core Backend cần viết code kiểm tra điều kiện `if (personId == null)`.

---

## Issue #4

- Raised by: Consumer
- Endpoint: Tất cả các endpoint (`POST /vision/face-match`, `GET /vision/detections/{detectionId}`)
- Concern: Nguy cơ trùng lặp request xử lý do mạng chập chờn khiến Consumer bấm nút gọi lại nhiều lần (Retry).
- Proposal: Bắt buộc đính kèm mã định danh vết cuộc gọi duy nhất cho mỗi request.
- Resolution: Accepted
- Rationale: Đảm bảo tính Idempotency (gọi nhiều lần không sinh ra kết quả mới). AI Vision sẽ cache lại các `traceId` đang xử lý trong 5 phút. Nếu nhận được `traceId` trùng, hệ thống sẽ trả ngay kết quả cũ trong cache mà không chạy lại model phân tích.
- Impact: Consumer bắt buộc phải truyền mã UUID thông qua Request Header `X-Correlation-ID` hoặc trường `traceId` trong Body.

---

## Issue #5

- Raised by: Provider
- Endpoint: `POST /vision/face-match`
- Concern: Ai sẽ là bên kiểm soát và quyết định ngưỡng tin cậy (Confidence Threshold) để xác định một người là trùng khớp.
- Proposal: AI Vision đề xuất cố định một ngưỡng duy nhất `0.85` ngay trong mã nguồn của dịch vụ AI.
- Resolution: Accepted
- Rationale: Để đảm bảo tính an ninh nghiêm ngặt cho toàn bộ khu vực Smart Campus, việc hạ thấp ngưỡng an toàn không được phép thực hiện tùy tiện từ phía Core nghiệp vụ. AI Vision nắm giữ cấu hình này là hợp lý nhất.
- Impact: AI Vision thiết lập cấu hình mặc định là `0.85`, đồng thời response trả về cho Core bắt buộc phải bao gồm cả trường `matched` (boolean) và trường tỷ lệ phần trăm `confidence` (number) để đối chiếu.

---

## Issue #6

- Raised by: Consumer
- Endpoint: `GET /vision/detections/{detectionId}`
- Concern: Khả năng mở rộng API trong tương lai khi Smart Campus tích hợp thêm hệ thống nhận diện Biển số xe bãi đỗ (License Plate) bên cạnh việc nhận diện khuôn mặt hiện tại.
- Proposal: Sử dụng mô hình dữ liệu đa hình polymorphism (oneOf + discriminator) ngay từ đầu cho endpoint lấy kết quả phân tích.
- Resolution: Accepted
- Rationale: Giúp hệ thống API đạt tính Extensibility (mở rộng cao), sau này khi bổ sung thêm các loại thực thể nhận diện mới sẽ không làm gãy cấu trúc code hiện tại của Core Business.
- Impact: Cấu trúc lại phân đoạn `components/schemas` trong `openapi.yaml`. Sử dụng trường phân loại `detectionType` làm `discriminator`.

---

# Chốt hợp đồng v1.0

Provider sign-off: Đã ký duyệt (Nhóm A4/B4)  
Consumer sign-off: Đã ký duyệt (Luong Vu - Nhóm A2)  
Witness (GV/TA):   
Date: 18/05/2026

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có | File openapi.yaml cam kết pass 100% các quy tắc của campus-spectral.yaml | Sẽ chạy kiểm tra định kỳ trước khi bàn giao |
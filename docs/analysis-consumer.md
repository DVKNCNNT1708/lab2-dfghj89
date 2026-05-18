# Phân tích độc lập từ phía Consumer (Core Business) - Pair 02

## 1. Yêu cầu Nghiệp vụ & Các Endpoint Kỳ vọng
Dựa trên User Story, phía Consumer cần tương tác với AI Vision qua 4 endpoint trọng tâm sau:

*   `POST /vision/face-match`: Gửi thông tin ảnh/embedding để so khớp khuôn mặt phục vụ điểm danh hoặc kiểm soát ra vào.
*   `GET /vision/detections/{detectionId}`: Lấy kết quả phân tích chi tiết của một lượt nhận diện thông qua ID.
*   `GET /vision/results/recent`: Lấy danh sách các kết quả nhận diện gần đây để hiển thị trên dashboard giám sát.
*   `GET /health`: Kiểm tra trạng thái hoạt động (healthcheck) của dịch vụ AI Vision.

## 2. Cấu trúc Dữ liệu Kỳ vọng (Schemas)
Để phục vụ việc audit và ra quyết định chính xác, Consumer yêu cầu Provider phải cấu trúc dữ liệu phản hồi (Response) đáp ứng các tiêu chí sau:

### Schema: FaceMatchResponse (Trả về từ POST /vision/face-match)
*   `traceId` (string, UUID): Mã vết cuộc gọi để phục vụ việc trace log hệ thống.
*   `matched` (boolean): Kết quả so khớp thành công hay thất bại.
*   `confidence` (number, float): Độ tin cậy của kết quả (Ví dụ: 0.95).
*   `modelVersion` (string): Phiên bản AI model đang chạy để đối chiếu khi có sai sót.
*   `timestamp` (string, date-time): Thời gian thực hiện phân tích.
*   `personId`: Trường này áp dụng tính năng **Union Type** của OpenAPI 3.1. Nếu khớp thành công thì trả về chuỗi ID, nếu không khớp (matched = false) thì bắt buộc phải trả về `null`.
    *   Cú pháp định nghĩa kỳ vọng: `type: [string, "null"]`

### Schema: DetectionResult (Dành cho việc phân loại object/loại thực thể)
Áp dụng **oneOf + discriminator** để phân biệt giữa nhận diện Khuôn mặt (Face) và nhận diện Biển số xe (LicensePlate) nếu có mở rộng hệ thống:
*   Trường phân loại: `detectionType`
*   Nếu `detectionType: "face"` -> Áp dụng `FaceDetectionPayload`
*   Nếu `detectionType: "license_plate"` -> Áp dụng `PlateDetectionPayload`

## 3. Các kịch bản lỗi kỳ vọng (Problem Details)
Tất cả các phản hồi lỗi 4xx và 5xx từ AI Vision phải tuân thủ chuẩn RFC 7807 (Problem Details) để Core Business dễ dàng parse lỗi tự động:
*   `400 Bad Request`: Định dạng ảnh không hợp lệ hoặc thiếu dữ liệu bắt buộc (như thiếu `traceId`).
*   `422 Unprocessable Entity`: Dùng khi dữ liệu đúng format nhưng không thể xử lý (Ví dụ: Ảnh quá mờ, không tìm thấy khuôn mặt để đối sánh).
*   `504 Gateway Timeout`: Hệ thống AI downstream bị nghẽn, không trả kết quả đúng hạn.

## 4. Đề xuất cho phiên đàm phán với AI Vision
Consumer sẽ mang 3 câu hỏi này lên bàn đàm phán để chốt phương án:
1. Phía Core sẽ gửi dữ liệu dưới dạng đường dẫn ảnh (`imageRef` - string URL) hay chuỗi vector khuôn mặt (`faceEmbedding` - array number)? *Đề xuất của Core: Gửi `imageRef` để giảm tải tính toán cho Core.*
2. Ngưỡng `confidence` bao nhiêu thì hệ thống AI Vision tự động tính là `matched: true`? (Ví dụ: cố định ở mức 0.85 hay cho phép Core truyền cấu hình lên?).
3. Khi model không chắc chắn (low confidence), AI Vision nên trả về code `200` kèm trạng thái `low_confidence` hay trả về lỗi `422`? *Đề xuất của Core: Trả về 200 kèm trạng thái để Core tự đưa ra quyết định nghiệp vụ tiếp theo.*
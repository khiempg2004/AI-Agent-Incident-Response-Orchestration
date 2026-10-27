# AI-Agent-Incident-Response-Orchestration

 Xây dựng AI Agent điều phối ứng phó sự cố hạ tầng dựa trên mô hình ngôn ngữ lớn (AI Incident Response Orchestration Agent)
---

## 1. Đặt vấn đề

Khi một dịch vụ phần mềm gặp sự cố (sập, độ trễ tăng vọt, lỗi 5xx hàng loạt), quy trình ứng phó tại hầu hết các tổ chức hiện nay vẫn **hoàn toàn thủ công**: người trực (on-call engineer) phải tập hợp đội ngũ, dò log từng service, đoán nguyên nhân, cập nhật trạng thái cho khách hàng bằng tay, và sau sự cố lại dành thêm thời gian viết báo cáo tổng kết (postmortem).

Điểm mấu chốt là: **phần lớn thời gian đầu của một sự cố không dành cho việc khắc phục, mà dành cho việc thu thập bối cảnh.** Người trực phải trả lời các câu hỏi "chuyện gì đang xảy ra, ở đâu, từ bao giờ, có liên quan đến lần deploy gần nhất không" trước khi chạm được vào bước sửa lỗi — đặc biệt tốn kém vào ban đêm khi đội ngũ phụ trách không có mặt đầy đủ.

Các công cụ giám sát hiện có (Prometheus, Grafana, Datadog...) chỉ giải quyết được một phần nhỏ của bài toán: chúng báo **"có gì đó sai"**. Chúng không:

- tổng hợp bối cảnh xuyên nhiều service;
- đề xuất giả thuyết nguyên nhân kèm bằng chứng số liệu;
- điều phối hành động khắc phục có kiểm soát;
- tự động cập nhật trạng thái cho người dùng;
- sinh báo cáo postmortem sau sự cố.

Bốn khoảng trống này chính là phạm vi mà đề tài hướng đến giải quyết.

## 2. Mục tiêu đề tài

**Mục tiêu tổng quát:** Xây dựng một AI Agent (tác tử thông minh) tự động điều phối trọn vòng đời ứng phó sự cố — từ phát hiện, chẩn đoán đến khắc phục — trong đó **con người giữ quyền quyết định cuối cùng** đối với mọi hành động ảnh hưởng đến môi trường production.

**Mục tiêu cụ thể:**

1. Phát hiện và phân loại mức độ nghiêm trọng (severity) của sự cố tự động dựa trên số liệu giám sát (metrics) và so sánh với baseline;
2. Tự động mở "war room" (phòng họp khẩn) và cập nhật diễn biến theo thời gian thực;
3. Thu thập bối cảnh tự động: metrics, logs, lịch sử deploy gần đây;
4. Chẩn đoán nguyên nhân gốc **có bằng chứng**: mỗi giả thuyết đều kèm số liệu dẫn chứng, truy vết ngược được về câu truy vấn cụ thể;
5. Thực thi cơ chế **Human-in-the-Loop (HITL) bắt buộc**: mọi hành động khắc phục production phải qua bước con người phê duyệt; Agent không có đường nào tự duyệt hành động cho chính mình;
6. Phát hiện sự cố lan truyền sang các service phụ thuộc và liên kết các incident liên quan;
7. Tự động sinh báo cáo postmortem (timeline, nguyên nhân, action item) khi đóng incident;
8. Cung cấp status page công khai, tự cập nhật theo trạng thái sự cố.

## 3. Đối tượng người dùng và phạm vi

| | Đối tượng | Nhu cầu |
|---|---|---|
| **Chính** | Kỹ sư on-call / Incident Commander | Rút ngắn thời gian từ lúc có tín hiệu đến lúc bắt đầu khắc phục; có sẵn bối cảnh được tổng hợp khi tiếp nhận sự cố |
| **Phụ** | Người dùng nội bộ / khách hàng | Biết được dịch vụ nào đang bị ảnh hưởng mà không phải hỏi ai (status page) |

**Trong phạm vi (MVP):** phát hiện & phân loại severity · war room tự động · thu thập bối cảnh · đối chiếu lịch sử deploy · chẩn đoán có bằng chứng + gợi ý runbook · HITL bắt buộc · phát hiện lan truyền · escalation khi chờ duyệt quá lâu · status page · war-room view nội bộ · postmortem tự động.

**Ngoài phạm vi:** tự động sửa chữa không cần người duyệt (trái nguyên tắc an toàn của đề tài); nhận diện bất thường bằng machine learning (ngưỡng tĩnh + baseline là đủ và đo lường được); chạy trên production thật với lưu lượng thật.

## 4. Giải pháp đề xuất

### 4.1. Luồng xử lý tổng quát

Hệ thống gồm một cặp service demo tự sinh ra sự cố **thật** khi bị tải cao (thay vì giả lập lỗi), một lớp giám sát thu thập số liệu, một AI Agent điều phối, và một hệ quản lý vòng đời incident với cổng phê duyệt của con người:

```
Load test → Service bị nghẽn thật (connection pool cạn)
    → Prometheus ghi nhận metric
    → Agent phát hiện vượt ngưỡng (xác nhận kéo dài ≥ 30s, chống mở incident trùng)
    → Phân loại severity → Mở incident/war room, cập nhật bối cảnh
    → Đối chiếu deploy history → Sinh giả thuyết + bằng chứng số liệu + runbook
    → Phát hiện lan truyền sang service phụ thuộc → Liên kết các incident
    → CHỜ NGƯỜI DUYỆT (Agent dừng hoàn toàn tại đây)
    → Người duyệt chấp nhận → Agent thực thi khắc phục
    → Kiểm chứng metric hồi phục → Đóng incident → Status page cập nhật
    → Sinh postmortem + action item
```

### 4.2. Ba quyết định thiết kế chính

1. **Chẩn đoán có bằng chứng, không để LLM đoán.** Rule engine chạy **trước**, sinh ra các con số (ví dụ: `p95 = 5.1s so với baseline 0.2s`, `connection pool 5/5`, `deploy v1.4.2 cách đây 4 phút`). LLM chỉ diễn giải bằng chứng đó thành ngôn ngữ tự nhiên và lựa chọn runbook phù hợp. Mọi giả thuyết đều truy vết ngược được về một câu truy vấn PromQL cụ thể.

2. **HITL được thực thi bằng cấu trúc, không bằng thiện chí.** Agent không có đường nào tự phê duyệt: hệ thống kiểm tra chéo với dữ liệu audit (changelog) của hệ quản lý incident để xác minh *ai* là người phê duyệt — nếu là chính account của Agent thì từ chối thi hành và cảnh báo. Mọi đường dẫn tới bước thực thi đều bắt buộc đi qua cổng kiểm tra này.

3. **Sự cố trong môi trường demo là sự cố thật.** Không mô phỏng lỗi giả: tải thật làm cạn connection pool thật, và service phụ thuộc timeout theo một cách tự nhiên. Nhờ vậy Agent phải chẩn đoán thực sự, không phải đọc kịch bản có sẵn.

## 5. Tech Stack

| Thành phần | Công nghệ | Vai trò |
|---|---|---|
| AI Agent | **LangGraph + LangChain** | State machine điều phối vòng đời incident, có cổng chờ phê duyệt của con người |
| LLM | **Qwen** (Alibaba Cloud Model Studio) qua endpoint tương thích OpenAI | Diễn giải bằng chứng, sinh giả thuyết, chọn runbook, viết postmortem |
| Backend | **FastAPI + Python 3.11** | REST API, webhook, điều phối trung tâm |
| Giám sát | **Prometheus** + `prometheus-fastapi-instrumentator` | Thu thập metrics; logic ngưỡng nằm trong Agent (không dùng Alertmanager) |
| Quản lý incident | **Jira Cloud REST API** (workflow + automation) | War room, phê duyệt HITL, audit trail, action item |
| Cơ sở dữ liệu | **PostgreSQL** | Dữ liệu nghiệp vụ demo, bảng deploy history |
| Kiểm thử tải | **k6 / Locust** | Tạo sự cố thật bằng load test tăng dần |
| DevOps | **Docker + GitHub Actions** | Đóng gói, CI/CD, deploy |

## 6. Kết quả mong đợi và tiêu chí đánh giá


**Bộ chỉ số đánh giá định lượng:**

| # | Chỉ số | Ngưỡng đạt |
|---|---|---|
| M1 | Thời gian từ lúc metric vượt ngưỡng → incident được mở | < 60 giây |
| M2 | Thời gian từ lúc phát hiện → giả thuyết đầu tiên có bằng chứng | < 3 phút |
| M3 | Độ chính xác chẩn đoán nguyên nhân gốc (so với ground truth của 3 kịch bản sự cố) | ≥ 2/3 kịch bản |
| M4 | Phát hiện lan truyền sang service phụ thuộc | Có, và liên kết được các incident |
| M5 | HITL không bị Agent vượt qua | 100% (Agent tự duyệt phải bị từ chối) |
| M6 | Status page phản ánh đúng trạng thái sau khi incident mở/đóng | ≤ 30 giây |


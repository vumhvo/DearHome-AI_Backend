---
title: ChatAgent BusinessInterior
emoji: 🏆
colorFrom: pink
colorTo: red
sdk: docker
app_port: 8000
pinned: false
---

# DearHome – AI Backend

Dịch vụ AI của **DearHome**, nền tảng thương mại điện tử nội thất, viết bằng **FastAPI**. Dịch vụ cung cấp ba nhóm chức năng:

1. **Chatbot tư vấn bán hàng** qua WebSocket: hiểu ý định của khách, tra cứu sản phẩm, giá, khuyến mãi, đơn hàng và trả lời bằng Google Gemini.
2. **Sinh mô hình 3D từ ảnh** (`.glb`) bằng Hunyuan3D-2.1 trên Hugging Face, rồi lưu lên Firebase Storage.
3. **Gợi ý bố trí phòng** (virtual room layout) từ kích thước phòng, cửa, cửa sổ và danh sách nội thất.

> Hệ thống DearHome gồm 3 repo:
>
> | Repo | Vai trò |
> |------|---------|
> | [DearHome-Frontend](https://github.com/vumhvo/DearHome-Frontend) | Web app React cho khách hàng và quản trị viên |
> | [DearHome_Backend](https://github.com/vumhvo/DearHome_Backend) | REST API chính (ASP.NET Core, SQL Server) |
> | **DearHome-AI_Backend** (repo này) | Chatbot AI, sinh mô hình 3D, gợi ý bố trí phòng (FastAPI) |

## Kiến trúc

```
 Frontend ──WebSocket /api/chat/ws──▶ ┌──────────────────────────┐ ──▶ Google Gemini (gemini-2.0-flash)
          ──REST /api/...───────────▶ │   DearHome-AI_Backend    │ ──▶ Hugging Face Space Hunyuan3D-2.1
                                      │        (FastAPI)         │ ──▶ Firebase Storage (file .glb)
 DearHome_Backend ──NATS events─────▶ └────────────┬─────────────┘
 (product/variant/order/promotion)                 │
                                       ┌───────────┴───────────┐
                                       ▼                       ▼
                                  ChromaDB (vector)     SQL Server (dùng chung DB
                                                        INTERIOR_SHOPPING với Backend)
```

### Luồng xử lý của chatbot

1. Frontend mở WebSocket tới `/api/chat/ws?token=<JWT>`. Token do DearHome_Backend cấp và được xác thực bằng cùng khoá bí mật.
2. Câu hỏi được nhúng (embedding) bằng `sentence-transformers` (`all-MiniLM-L6-v2`) rồi **phân loại ý định** bằng k-NN + cosine similarity so với các câu mẫu trong [`src/core/intent_embeddings.json`](src/core/intent_embeddings.json).
3. Gemini trích xuất tham số cần thiết (tên sản phẩm, mã khuyến mãi…) cho ý định đó.
4. Hàm tương ứng trong [`FunctionCallingManager`](src/managers/function_calling_manager.py) truy vấn **ChromaDB** / SQL Server để lấy dữ liệu thật.
5. Gemini tổng hợp câu trả lời tự nhiên và gửi lại qua WebSocket.

Các ý định được hỗ trợ: chào hỏi, tìm sản phẩm, thông tin / kích thước / chất liệu / màu sắc / giá / tình trạng còn hàng của sản phẩm, khuyến mãi, trạng thái đơn hàng, vận chuyển, phương thức thanh toán, chính sách đổi trả, tư vấn thiết kế nội thất và phối màu, cảm ơn, tạm biệt.

### Đồng bộ dữ liệu qua NATS

Khi khởi động, ứng dụng đăng ký lắng nghe các chủ đề `product.*`, `variant.*`, `order.*`, `promotion.*` do DearHome_Backend phát ra, rồi cập nhật các collection tương ứng trong ChromaDB (`products`, `variants`, `orders`, `promotions`) và gửi lại thông điệp `*.ack`. Nếu không kết nối được NATS, ứng dụng **sẽ dừng khởi động**.

## Công nghệ

- **FastAPI** + Uvicorn, WebSocket
- **Google Gemini** (`google-genai`, model `gemini-2.0-flash`)
- **sentence-transformers**, **scikit-learn** để phân loại ý định
- **ChromaDB** (lưu trữ persistent) để tìm kiếm ngữ nghĩa
- **SQLAlchemy** + `pyodbc` (SQL Server)
- **nats-py** để nhận sự kiện từ Backend
- **gradio_client** để gọi Hugging Face Space Hunyuan3D-2.1
- **firebase-admin** để upload mô hình 3D
- **PyJWT** để xác thực token

## Cấu trúc thư mục

```
.
├── app.py                    # Khởi tạo FastAPI, CORS, lifespan (khởi động NATS handlers)
├── Dockerfile                # Image dùng cho Hugging Face Spaces / Docker
├── requirements.txt
├── dataset/                  # Dữ liệu mẫu hội thoại
└── src/
    ├── api/                  # Gemini client, cấu hình
    ├── authentication/       # Xác thực JWT cho HTTP và WebSocket
    ├── config/               # Cấu hình ChromaDB (collections, thư mục lưu)
    ├── core/                 # Dữ liệu ý định và embedding mẫu
    ├── database/             # Kết nối SQL Server, model SQLAlchemy, khởi tạo bảng
    ├── handlers/             # NATS subscribers đồng bộ product/variant/order/promotion
    ├── managers/             # ChatbotManager, FunctionCallingManager
    ├── models/               # Pydantic models
    ├── routers/              # chat_router, shape_gen_router, virtual_room_router
    ├── services/             # Truy vấn sản phẩm, đơn hàng, khuyến mãi, Chroma, Firebase…
    ├── websockets/           # Quản lý kết nối WebSocket
    └── shape_gen.py          # Gọi Hunyuan3D-2.1 qua gradio_client
```

## Bắt đầu

### Yêu cầu

- **Python 3.10 trở lên** (mã nguồn dùng cú pháp `str | None`)
- [Microsoft ODBC Driver 18 for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server)
- SQL Server có database `INTERIOR_SHOPPING` do DearHome_Backend tạo
- NATS server (chạy cùng instance với DearHome_Backend)
- Gemini API key, file service account của Firebase
- (Tuỳ chọn) Hugging Face token để gọi Space sinh 3D

### Cài đặt

```bash
git clone https://github.com/vumhvo/DearHome-AI_Backend.git
cd DearHome-AI_Backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install fastapi uvicorn pyjwt gradio_client
```

> `fastapi` và `uvicorn` đang được comment trong `requirements.txt` (chúng được cài kèm theo `gradio`). Lệnh thứ hai ở trên giúp chắc chắn đủ các gói cần thiết.

### Biến môi trường

Tạo file `.env` ở thư mục gốc:

```dotenv
# Server
HOST=127.0.0.1
PORT=8000
ENVIRONMENT=development          # development: bật auto-reload

# Google Gemini
GEMINI_API_KEY=<gemini-api-key>

# JWT – phải trùng với Jwt:Key / Jwt:Issuer / Jwt:Audience của DearHome_Backend
JWT_SECRET_KEY=<cùng-khoá-với-backend>
JWT_ISSUER=http://localhost:5000/
JWT_AUDIENCE=http://localhost:5000/

# SQL Server
SQL_SERVER=127.0.0.1
SQL_DATABASE=INTERIOR_SHOPPING
SQL_USERNAME=sa
SQL_PASSWORD=<mật-khẩu>

# NATS
NATS_SERVER_URL=nats://localhost:4222
NATS_USER=<user>
NATS_PASSWORD=<password>

# Firebase
FIREBASE_CREDENTIAL_PATH=./credentials/firebase-service-account.json
FIREBASE_STORAGE_BUCKET=<project-id>.appspot.com

# Hugging Face (sinh mô hình 3D)
HF_TOKEN=<hf-token>

# ChromaDB
CHROMA_DB_DIR=./data/chroma_db
```

> Lưu ý: trong [`src/database/db_connection.py`](src/database/db_connection.py), chuỗi kết nối `pyodbc` hiện đang cố định server `127.0.0.1` và database `INTERIOR_SHOPPING`. Nếu SQL Server nằm ở máy khác, hãy sửa trực tiếp tại đây.

### Chạy

```bash
python app.py
```

Hoặc:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

Tài liệu API (Swagger) ở `http://localhost:8000/api/docs`, ReDoc ở `/api/redoc`.

## API

| Phương thức | Đường dẫn | Mô tả |
|-------------|-----------|-------|
| `WS` | `/api/chat/ws?token=<JWT>` | Chatbot. Client gửi text thuần; server trả JSON `{ "type": "message", "client_id", "response", "timestamp" }`. Tin nhắn đầu tiên có `type: "connection"`. |
| `POST` | `/api/generate_shape` | Sinh mô hình 3D từ ảnh. Body: `caption`, `image_path` (bắt buộc); `front_image_path`, `back_image_path`, `left_image_path`, `right_image_path` (tuỳ chọn). Kết quả `.glb` được upload lên Firebase Storage. |
| `GET` | `/api/download_glb?file_path=…` | Tải file `.glb` đã sinh. |
| `POST` | `/api/virtual_room/generate_virtual_layout` | Gợi ý bố trí phòng. Body gồm `room` (loại phòng, kích thước, màu tường, sàn, cửa, cửa sổ), `furniture` (danh sách id biến thể), `prompt`, `options` (`number_of_designs`, `style_preference`, `priority`: `functionality` / `aesthetics` / `balance`). |

Ví dụ gọi sinh mô hình 3D:

```bash
curl -X POST http://localhost:8000/api/generate_shape \
  -H "Content-Type: application/json" \
  -d '{"caption": "ghế gỗ", "image_path": "https://example.com/chair.png"}'
```

## Triển khai

### Docker

```bash
docker build -t dearhome-ai .
docker run --env-file .env -p 8000:8000 dearhome-ai
```

### Hugging Face Spaces

Phần front matter ở đầu file này (`sdk: docker`, `app_port: 8000`) là cấu hình cho Hugging Face Spaces, **đừng xoá**. Khi đẩy repo lên một Space kiểu Docker, Space sẽ build theo `Dockerfile` và chạy trên cổng 8000. Khai báo các biến môi trường ở mục *Settings → Variables and secrets* của Space.

Xem thêm: <https://huggingface.co/docs/hub/spaces-config-reference>

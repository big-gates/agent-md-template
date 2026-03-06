# API Design Guide

> **트리거**: REST API, 스트리밍 API 엔드포인트를 설계할 때 이 문서를 참조한다.

---

## URL 설계

```
/api/v{version}/{resource}
/api/v{version}/{resource}/{id}
/api/v{version}/{resource}/{id}/{sub-resource}
```

### 예시

```
GET    /api/v1/orders                    # 주문 목록
POST   /api/v1/orders                    # 주문 생성
GET    /api/v1/orders/{id}               # 주문 상세
PUT    /api/v1/orders/{id}               # 주문 수정
DELETE /api/v1/orders/{id}               # 주문 삭제
POST   /api/v1/orders/{id}/confirm       # 주문 확정 (행위)
GET    /api/v1/orders/{id}/items         # 주문 항목 목록

# 스트리밍
GET    /api/v1/sse/notifications/{userId}     # SSE 알림 스트림
WS     /ws/chat                               # WebSocket 채팅
GET    /api/v1/orders/stream                  # NDJSON 스트림
```

| 규칙 | 설명 |
|------|------|
| 복수형 명사 | `/orders`, `/products` (동사 아님) |
| 소문자 케밥 | `/order-items` (camelCase 아님) |
| 버전 포함 | `/api/v1/` |
| 행위 URL | 동사가 필요한 경우 하위 경로로: `/orders/{id}/confirm` |

---

## HTTP 메서드

| 메서드 | 용도 | 멱등성 |
|--------|------|--------|
| `GET` | 조회 | O |
| `POST` | 생성 / 행위 | X |
| `PUT` | 전체 수정 | O |
| `PATCH` | 부분 수정 | X |
| `DELETE` | 삭제 | O |

---

## 상태 코드

| 코드 | 용도 |
|------|------|
| `200 OK` | 조회/수정 성공 |
| `201 Created` | 생성 성공 |
| `204 No Content` | 삭제 성공 (응답 본문 없음) |
| `400 Bad Request` | 요청 본문 오류, 검증 실패 |
| `404 Not Found` | 리소스 없음 |
| `409 Conflict` | 비즈니스 규칙 충돌 (재고 부족 등) |
| `422 Unprocessable Entity` | 상태 전환 불가 |
| `500 Internal Server Error` | 서버 내부 오류 |

---

## 요청 / 응답 형식

### 성공 응답

```json
// 단일 리소스
{
    "id": "order-1",
    "customerId": "customer-1",
    "status": "CONFIRMED",
    "totalAmount": 50000,
    "items": [
        {
            "productId": "product-1",
            "quantity": 2,
            "unitPrice": 25000
        }
    ],
    "createdAt": "2025-01-01T00:00:00"
}

// 목록 (페이지네이션)
{
    "content": [...],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
}
```

### 에러 응답

```json
{
    "code": "ORDER_NOT_FOUND",
    "message": "주문을 찾을 수 없습니다: order-1",
    "timestamp": "2025-01-01T12:00:00"
}
```

---

## 스트리밍 API

### SSE (Server-Sent Events)

```
GET /api/v1/sse/notifications/{userId}
Accept: text/event-stream

Response:
event: notification
data: {"id":"n1","message":"새 주문 접수","type":"ORDER"}

event: heartbeat
data:
```

### NDJSON (Newline-Delimited JSON)

```
GET /api/v1/orders/stream
Accept: application/x-ndjson

Response:
{"id":"order-1","status":"CONFIRMED","totalAmount":50000}
{"id":"order-2","status":"DRAFT","totalAmount":30000}
...
```

### WebSocket

```
WS /ws/chat?userId=u1

Client → Server:
{"type":"MESSAGE","roomId":"room-1","content":"안녕하세요"}

Server → Client:
{"type":"MESSAGE","roomId":"room-1","senderId":"u2","content":"반갑습니다"}
```

---

## 페이지네이션

```kotlin
// Router
GET("/api/v1/orders") { request ->
    val page = request.queryParam("page").orElse("0").toInt()
    val size = request.queryParam("size").orElse("20").toInt()
    handler.getOrders(page, size)
}
```

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `page` | 0 | 페이지 번호 (0-based) |
| `size` | 20 | 페이지 크기 (max: 100) |
| `sort` | `createdAt,desc` | 정렬 기준 |

---

## Content-Type 가이드

| 용도 | Content-Type |
|------|-------------|
| 일반 REST | `application/json` |
| SSE | `text/event-stream` |
| NDJSON 스트림 | `application/x-ndjson` |
| WebSocket | - (프로토콜 업그레이드) |

---

## 변경 시 체크리스트

- [ ] URL이 복수형 명사인가?
- [ ] 적절한 HTTP 메서드를 사용하는가?
- [ ] 상태 코드가 올바른가? (201 for 생성, 204 for 삭제 등)
- [ ] 에러 응답이 `code` + `message` 형식인가?
- [ ] 스트리밍 API에 올바른 Content-Type이 설정되어 있는가?
- [ ] 페이지네이션이 일관된 형식인가?

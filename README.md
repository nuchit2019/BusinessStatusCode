# API Error Handling Standard

มาตรฐานกลางสำหรับการออกแบบ API response, exception mapping และ BusinessStatusCode ของทีม

---

## 1. Purpose

เอกสารนี้กำหนดมาตรฐานกลางสำหรับ:

- HTTP Status Code
- BusinessStatusCode
- ErrorCode
- API Response Contract
- Exception Mapping

เป้าหมายคือให้ทุก service ตอบกลับในรูปแบบเดียวกัน
เพื่อให้ frontend, backend, integration และ monitoring ใช้งานได้สม่ำเสมอ

---

## 2. Scope

ใช้กับ:
- Internal APIs
- Backend Services
- Microservices
- Integration APIs ภายในทีม

ไม่ใช้ BusinessStatusCode แทน HTTP Status Code
ทั้งสองอย่างต้องอยู่ร่วมกันอย่างชัดเจน

---

## 3. Core Principles

### 3.1 HTTP Status Code
ใช้สำหรับ protocol/transport outcome

ตัวอย่าง:
- 200 OK
- 400 Bad Request
- 401 Unauthorized
- 403 Forbidden
- 404 Not Found
- 409 Conflict
- 500 Internal Server Error
- 502 Bad Gateway

### 3.2 BusinessStatusCode
ใช้สำหรับ business/application outcome

ตัวอย่าง:
- 1000 Success
- 1001 NoData
- 1400 ValidationError
- 1404 NotFound
- 2001 DuplicateData
- 3002 IntegrationError

### 3.3 ErrorCode
ใช้สำหรับ machine-readable code ที่ frontend / consumer เอาไป handle ได้

ตัวอย่าง:
- PREMIUM_NOT_FOUND
- POLICY_ALREADY_CANCELLED
- VALIDATION_ERROR
- DUPLICATE_DATA

---

## 4. Standard Response Contract

ทุก API ต้องตอบกลับด้วย schema นี้

```json
{
  "httpStatusCode": 200,
  "businessStatusCode": 1000,
  "errorCode": null,
  "success": true,
  "message": "Request processed successfully.",
  "data": {},
  "errors": null,
  "traceId": "00-abc123xyz"
}

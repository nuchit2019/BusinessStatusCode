# API Design Best Practices Guide
## HTTP Status Code & Business Status Code

> **Version 1.0 | March 2026**  
> Reference: [BusinessStatusCode on GitHub](https://github.com/nuchit2019/BusinessStatusCode/blob/main/README.md)

---

## สารบัญ

1. [แนวคิดและหลักการ](#1-แนวคิดและหลักการ-core-concepts)
2. [โครงสร้างโปรเจกต์](#2-โครงสร้างโปรเจกต์-project-structure)
3. [Business Status Code Design](#3-business-status-code-design)
4. [ApiResponse Model](#4-apiresponset-model)
5. [Exception Hierarchy](#5-exception-hierarchy)
6. [Error Catalog](#6-error-catalog-domain-based)
7. [ApiResponseFactory](#7-apiresponsefactory)
8. [Exception Middleware](#8-exception-middleware)
9. [ตัวอย่างการใช้งาน](#9-ตัวอย่างการใช้งาน-usage-examples)
10. [ตัวอย่าง API Response](#10-ตัวอย่าง-api-response)
11. [HTTP vs Business Status Code Mapping](#11-http-vs-business-status-code-mapping)
12. [Best Practices Summary](#12-best-practices-summary)
13. [Implementation Checklist](#13-implementation-checklist)

---

## 1. แนวคิดและหลักการ (Core Concepts)

การออกแบบ API ที่ดีต้อง **แยกการสื่อสาร 2 ระดับ** ออกจากกันให้ชัดเจน

### 🔵 HTTP Status Code — Protocol Layer
ใช้สื่อสารสถานะของ HTTP Request/Response ตามมาตรฐาน RFC 7231

```
200  OK
400  Bad Request
401  Unauthorized
403  Forbidden
404  Not Found
409  Conflict
500  Internal Server Error
502  Bad Gateway
```

### 🟢 Business Status Code — Application Layer
ใช้สื่อสาร Business Logic และ Application Error ใน **Response Body**

```
1000  Success
1400  Validation Error
1401  Unauthorized
1403  Forbidden
1404  Not Found
1409  Conflict
2000  Business Rule Violation
2001  Duplicate Data
2002  Invalid State Transition
3000  External Service Error
3001  Database Error
3002  Integration Error
5000  System Error
```

### ตัวอย่าง Response Structure

HTTP Response เป็น `404 Not Found` แต่ใน Body ระบุ `businessStatusCode: 1404` เพื่อบอก context ที่ละเอียดกว่า

```json
// HTTP Response: 404 Not Found
{
  "statusCode": 404,
  "businessStatusCode": 1404,
  "success": false,
  "message": "Premium compulsory not found",
  "data": null,
  "errors": null
}
```

---

## 2. โครงสร้างโปรเจกต์ (Project Structure)

```
Shared/
├── Constants/
│   └── BusinessStatusCode.cs       // Enum ของ Business Status Code ทั้งหมด
├── Errors/
│   ├── ErrorDefinition.cs          // Model กลางของ Error
│   ├── ErrorCatalog.cs             // Factory สร้าง ErrorDefinition
│   ├── CommonErrors.cs             // Errors ทั่วไป
│   ├── PremiumErrors.cs            // Errors ของ Premium Domain
│   ├── PolicyErrors.cs             // Errors ของ Policy Domain
│   └── OrderErrors.cs              // Errors ของ Order Domain
├── Exceptions/
│   ├── AppException.cs             // Abstract base exception
│   ├── NotFoundException.cs
│   ├── BusinessRuleException.cs
│   ├── ValidationException.cs
│   ├── UnauthorizedException.cs
│   ├── ForbiddenException.cs
│   ├── ConflictException.cs
│   ├── DatabaseException.cs
│   ├── ExternalServiceException.cs
│   └── IntegrationException.cs
├── Models/
│   └── ApiResponse.cs              // Generic response wrapper
└── Middlewares/
    ├── ExceptionMiddleware.cs      // Global exception handler
    └── MiddlewareExtensions.cs
Factories/
└── ApiResponseFactory.cs           // Helper สร้าง ApiResponse
```

---

## 3. Business Status Code Design

```csharp
// Shared/Constants/BusinessStatusCode.cs
namespace YourProject.Shared.Constants;

public enum BusinessStatusCode
{
    // 1xxx — HTTP-aligned general codes
    Success                = 1000,
    ValidationError        = 1400,
    Unauthorized           = 1401,
    Forbidden              = 1403,
    NotFound               = 1404,
    Conflict               = 1409,

    // 2xxx — Business rule violations
    BusinessRuleViolation  = 2000,
    DuplicateData          = 2001,
    InvalidStateTransition = 2002,

    // 3xxx — External / integration errors
    ExternalServiceError   = 3000,
    DatabaseError          = 3001,
    IntegrationError       = 3002,

    // 5xxx — System-level errors
    SystemError            = 5000
}
```

**เหตุผลการออกแบบ prefix:**
- `1xxx` — สอดคล้องกับ HTTP Status (อ่านง่าย, คาดเดาได้)
- `2xxx` — Business Rule violations (logic ภายในระบบ)
- `3xxx` — External/Integration errors (ปัญหาจากภายนอก)
- `5xxx` — System-level unhandled errors

---

## 4. ApiResponse\<T\> Model

```csharp
// Shared/Models/ApiResponse.cs
namespace YourProject.Shared.Models;

public sealed class ApiResponse<T>
{
    public int     StatusCode { get; init; }
    public bool    Success    { get; init; }
    public string  Message    { get; init; } = string.Empty;
    public T?      Data       { get; init; }
    public object? Errors     { get; init; }

    public static ApiResponse<T> Create(
        int statusCode,
        bool success,
        string message,
        T? data = default,
        object? errors = null)
    {
        return new ApiResponse<T>
        {
            StatusCode = statusCode,
            Success    = success,
            Message    = message,
            Data       = data,
            Errors     = errors
        };
    }
}
```

> **จุดสำคัญ:** `StatusCode` ใน Body = Business Status Code (ไม่ใช่ HTTP Status Code)  
> `Errors` field รองรับ Validation detail แบบ structured ได้

---

## 5. Exception Hierarchy

### 5.1 AppException (Abstract Base)

```csharp
// Shared/Exceptions/AppException.cs
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public abstract class AppException : Exception
{
    public HttpStatusCode     HttpStatusCode     { get; }
    public BusinessStatusCode BusinessStatusCode { get; }
    public string             ErrorCode          { get; }
    public object?            Errors             { get; }

    protected AppException(
        string message,
        string errorCode,
        HttpStatusCode httpStatusCode,
        BusinessStatusCode businessStatusCode,
        object? errors = null) : base(message)
    {
        ErrorCode          = errorCode;
        HttpStatusCode     = httpStatusCode;
        BusinessStatusCode = businessStatusCode;
        Errors             = errors;
    }
}
```

### 5.2 NotFoundException

```csharp
public sealed class NotFoundException : AppException
{
    public NotFoundException(
        string message,
        string errorCode = "NOT_FOUND",
        object? errors = null)
        : base(message, errorCode,
               HttpStatusCode.NotFound,
               BusinessStatusCode.NotFound, errors) { }
}
```

### 5.3 ValidationException

```csharp
public sealed class ValidationException : AppException
{
    public ValidationException(
        string message   = "Validation failed",
        string errorCode = "VALIDATION_ERROR",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.BadRequest,
               BusinessStatusCode.ValidationError, errors) { }
}
```

### 5.4 BusinessRuleException

```csharp
public sealed class BusinessRuleException : AppException
{
    public BusinessRuleException(
        string message,
        string errorCode = "BUSINESS_RULE_VIOLATION",
        BusinessStatusCode businessStatusCode = BusinessStatusCode.BusinessRuleViolation,
        object? errors = null)
        : base(message, errorCode,
               HttpStatusCode.BadRequest, businessStatusCode, errors) { }
}
```

### 5.5 UnauthorizedException

```csharp
public sealed class UnauthorizedException : AppException
{
    public UnauthorizedException(
        string message   = "Unauthorized access",
        string errorCode = "UNAUTHORIZED",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.Unauthorized,
               BusinessStatusCode.Unauthorized, errors) { }
}
```

### 5.6 ForbiddenException

```csharp
public sealed class ForbiddenException : AppException
{
    public ForbiddenException(
        string message   = "Access denied",
        string errorCode = "FORBIDDEN",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.Forbidden,
               BusinessStatusCode.Forbidden, errors) { }
}
```

### 5.7 ConflictException

```csharp
public sealed class ConflictException : AppException
{
    public ConflictException(
        string message = "Conflict occurred",
        string errorCode = "CONFLICT",
        BusinessStatusCode businessStatusCode = BusinessStatusCode.Conflict,
        object? errors = null)
        : base(message, errorCode,
               HttpStatusCode.Conflict, businessStatusCode, errors) { }
}
```

### 5.8 DatabaseException

```csharp
public sealed class DatabaseException : AppException
{
    public DatabaseException(
        string message   = "Database error occurred",
        string errorCode = "DATABASE_ERROR",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.InternalServerError,
               BusinessStatusCode.DatabaseError, errors) { }
}
```

### 5.9 ExternalServiceException

```csharp
public sealed class ExternalServiceException : AppException
{
    public ExternalServiceException(
        string message   = "External service error",
        string errorCode = "EXTERNAL_SERVICE_ERROR",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.BadGateway,
               BusinessStatusCode.ExternalServiceError, errors) { }
}
```

### 5.10 IntegrationException

```csharp
public sealed class IntegrationException : AppException
{
    public IntegrationException(
        string message   = "Integration error occurred",
        string errorCode = "INTEGRATION_ERROR",
        object? errors   = null)
        : base(message, errorCode,
               HttpStatusCode.BadGateway,
               BusinessStatusCode.IntegrationError, errors) { }
}
```

---

## 6. Error Catalog (Domain-based)

### 6.1 ErrorDefinition

```csharp
// Shared/Errors/ErrorDefinition.cs
namespace YourProject.Shared.Errors;

public sealed class ErrorDefinition
{
    public string             Code       { get; init; } = string.Empty;
    public BusinessStatusCode StatusCode { get; init; }
    public string             Message    { get; init; } = string.Empty;
}
```

### 6.2 ErrorCatalog

```csharp
// Shared/Errors/ErrorCatalog.cs
namespace YourProject.Shared.Errors;

public static class ErrorCatalog
{
    public static ErrorDefinition Create(
        string code,
        Constants.BusinessStatusCode statusCode,
        string message)
    {
        return new ErrorDefinition
        {
            Code       = code,
            StatusCode = statusCode,
            Message    = message
        };
    }
}
```

### 6.3 CommonErrors

```csharp
// Shared/Errors/CommonErrors.cs
public static class CommonErrors
{
    public static readonly ErrorDefinition Success =
        ErrorCatalog.Create("SUCCESS",
            BusinessStatusCode.Success,
            "Request processed successfully");

    public static readonly ErrorDefinition ValidationFailed =
        ErrorCatalog.Create("VALIDATION_ERROR",
            BusinessStatusCode.ValidationError,
            "Validation failed");

    public static readonly ErrorDefinition Unauthorized =
        ErrorCatalog.Create("UNAUTHORIZED",
            BusinessStatusCode.Unauthorized,
            "Unauthorized access");

    public static readonly ErrorDefinition Forbidden =
        ErrorCatalog.Create("FORBIDDEN",
            BusinessStatusCode.Forbidden,
            "Access denied");

    public static readonly ErrorDefinition NotFound =
        ErrorCatalog.Create("NOT_FOUND",
            BusinessStatusCode.NotFound,
            "Requested resource was not found");

    public static readonly ErrorDefinition Conflict =
        ErrorCatalog.Create("CONFLICT",
            BusinessStatusCode.Conflict,
            "Conflict occurred");

    public static readonly ErrorDefinition ExternalServiceError =
        ErrorCatalog.Create("EXTERNAL_SERVICE_ERROR",
            BusinessStatusCode.ExternalServiceError,
            "External service error");

    public static readonly ErrorDefinition IntegrationError =
        ErrorCatalog.Create("INTEGRATION_ERROR",
            BusinessStatusCode.IntegrationError,
            "Integration error occurred");

    public static readonly ErrorDefinition SystemError =
        ErrorCatalog.Create("SYSTEM_ERROR",
            BusinessStatusCode.SystemError,
            "Unexpected system error");
}
```

### 6.4 PremiumErrors

```csharp
// Shared/Errors/PremiumErrors.cs
public static class PremiumErrors
{
    public static readonly ErrorDefinition PremiumCompulsoryNotFound =
        ErrorCatalog.Create("PREMIUM_COMPULSORY_NOT_FOUND",
            BusinessStatusCode.NotFound,
            "Premium compulsory not found");

    public static readonly ErrorDefinition PremiumAlreadyApproved =
        ErrorCatalog.Create("PREMIUM_ALREADY_APPROVED",
            BusinessStatusCode.BusinessRuleViolation,
            "Premium already approved");

    public static readonly ErrorDefinition PremiumDuplicate =
        ErrorCatalog.Create("PREMIUM_DUPLICATE",
            BusinessStatusCode.DuplicateData,
            "Duplicate premium data");
}
```

### 6.5 PolicyErrors

```csharp
// Shared/Errors/PolicyErrors.cs
public static class PolicyErrors
{
    public static readonly ErrorDefinition PolicyNotFound =
        ErrorCatalog.Create("POLICY_NOT_FOUND",
            BusinessStatusCode.NotFound,
            "Policy not found");

    public static readonly ErrorDefinition PolicyAlreadyCancelled =
        ErrorCatalog.Create("POLICY_ALREADY_CANCELLED",
            BusinessStatusCode.InvalidStateTransition,
            "Policy already cancelled");
}
```

### 6.6 OrderErrors

```csharp
// Shared/Errors/OrderErrors.cs
public static class OrderErrors
{
    public static readonly ErrorDefinition OrderNotFound =
        ErrorCatalog.Create("ORDER_NOT_FOUND",
            BusinessStatusCode.NotFound,
            "Order not found");

    public static readonly ErrorDefinition OrderAlreadyProcessed =
        ErrorCatalog.Create("ORDER_ALREADY_PROCESSED",
            BusinessStatusCode.BusinessRuleViolation,
            "Order already processed");
}
```

---

## 7. ApiResponseFactory

```csharp
// Factories/ApiResponseFactory.cs
using YourProject.Shared.Errors;
using YourProject.Shared.Models;

namespace YourProject.Factories;

public static class ApiResponseFactory
{
    public static ApiResponse<T> Success<T>(T data, ErrorDefinition? error = null)
    {
        var definition = error ?? CommonErrors.Success;

        return ApiResponse<T>.Create(
            statusCode: (int)definition.StatusCode,
            success:    true,
            message:    definition.Message,
            data:       data,
            errors:     null);
    }

    public static ApiResponse<T> Failure<T>(ErrorDefinition error, object? errors = null)
    {
        return ApiResponse<T>.Create(
            statusCode: (int)error.StatusCode,
            success:    false,
            message:    error.Message,
            data:       default,
            errors:     errors);
    }

    public static ApiResponse<T> Failure<T>(
        int statusCode, string message, object? errors = null)
    {
        return ApiResponse<T>.Create(
            statusCode: statusCode,
            success:    false,
            message:    message,
            data:       default,
            errors:     errors);
    }
}
```

---

## 8. Exception Middleware

### 8.1 ExceptionMiddleware

```csharp
// Shared/Middlewares/ExceptionMiddleware.cs
using System.Net;
using System.Text.Json;
using YourProject.Factories;
using YourProject.Shared.Constants;
using YourProject.Shared.Exceptions;

namespace YourProject.Shared.Middlewares;

public sealed class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;

    public ExceptionMiddleware(
        RequestDelegate next,
        ILogger<ExceptionMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (AppException ex)
        {
            _logger.LogWarning(ex,
                "Application exception. ErrorCode: {ErrorCode}, BusinessCode: {BusinessCode}",
                ex.ErrorCode, ex.BusinessStatusCode);

            context.Response.ContentType = "application/json";
            context.Response.StatusCode  = (int)ex.HttpStatusCode;

            var response = ApiResponseFactory.Failure<object>(
                statusCode: (int)ex.BusinessStatusCode,
                message:    ex.Message,
                errors:     ex.Errors);

            await context.Response.WriteAsync(
                JsonSerializer.Serialize(response));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");

            context.Response.ContentType = "application/json";
            context.Response.StatusCode  = (int)HttpStatusCode.InternalServerError;

            var response = ApiResponseFactory.Failure<object>(
                statusCode: (int)BusinessStatusCode.SystemError,
                message:    "Unexpected system error");

            await context.Response.WriteAsync(
                JsonSerializer.Serialize(response));
        }
    }
}
```

### 8.2 MiddlewareExtensions

```csharp
// Shared/Middlewares/MiddlewareExtensions.cs
namespace YourProject.Shared.Middlewares;

public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseGlobalExceptionMiddleware(
        this IApplicationBuilder app)
    {
        return app.UseMiddleware<ExceptionMiddleware>();
    }
}
```

### 8.3 Program.cs Registration

```csharp
using YourProject.Shared.Middlewares;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// ✅ Register ก่อน middleware อื่นทุกตัว
app.UseGlobalExceptionMiddleware();

app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## 9. ตัวอย่างการใช้งาน (Usage Examples)

### 9.1 การ Throw Exception จาก Service Layer

**ตัวอย่างที่ 1 — Not Found**

```csharp
var premium = await _repository.GetByIdAsync(id);

if (premium is null)
{
    throw new NotFoundException(
        PremiumErrors.PremiumCompulsoryNotFound.Message,
        PremiumErrors.PremiumCompulsoryNotFound.Code);
}
```

**ตัวอย่างที่ 2 — Validation Error**

```csharp
throw new ValidationException(
    message:   CommonErrors.ValidationFailed.Message,
    errorCode: CommonErrors.ValidationFailed.Code,
    errors: new
    {
        PolicyNo      = new[] { "PolicyNo is required" },
        PremiumAmount = new[] { "Must be greater than zero" }
    });
```

**ตัวอย่างที่ 3 — Business Rule Violation**

```csharp
if (policy.Status == PolicyStatus.Cancelled)
{
    throw new BusinessRuleException(
        message:            PolicyErrors.PolicyAlreadyCancelled.Message,
        errorCode:          PolicyErrors.PolicyAlreadyCancelled.Code,
        businessStatusCode: PolicyErrors.PolicyAlreadyCancelled.StatusCode);
}
```

**ตัวอย่างที่ 4 — Conflict / Duplicate**

```csharp
var exists = await _repository.ExistsAsync(dto.PolicyNo);

if (exists)
{
    throw new ConflictException(
        message:            PremiumErrors.PremiumDuplicate.Message,
        errorCode:          PremiumErrors.PremiumDuplicate.Code,
        businessStatusCode: PremiumErrors.PremiumDuplicate.StatusCode);
}
```

### 9.2 การ Return Response จาก Controller

```csharp
// ✅ Controller ส่งแค่ Success — Error ปล่อยให้ Middleware จัดการ

[HttpGet("{id}")]
public async Task<IActionResult> GetPremium(int id)
{
    var result = await _service.GetPremiumAsync(id);
    return Ok(ApiResponseFactory.Success(result));
}

[HttpPost]
public async Task<IActionResult> CreatePremium(CreatePremiumDto dto)
{
    var result = await _service.CreateAsync(dto);
    return Created($"/api/premiums/{result.Id}",
        ApiResponseFactory.Success(result));
}

[HttpPut("{id}")]
public async Task<IActionResult> UpdatePremium(int id, UpdatePremiumDto dto)
{
    var result = await _service.UpdateAsync(id, dto);
    return Ok(ApiResponseFactory.Success(result));
}
```

---

## 10. ตัวอย่าง API Response

### ✅ Success — HTTP 200

```json
// HTTP/1.1 200 OK
{
  "statusCode":         200,
  "businessStatusCode": 1000,
  "success":            true,
  "message":            "Request processed successfully",
  "data": {
    "id":       101,
    "policyNo": "PL-2026-0001",
    "status":   "Active"
  },
  "errors": null
}
```

### ❌ Not Found — HTTP 404

```json
// HTTP/1.1 404 Not Found
{
  "statusCode":         404,
  "businessStatusCode": 1404,
  "success":            false,
  "message":            "Premium compulsory not found",
  "data":               null,
  "errors":             null
}
```

### ❌ Validation Error — HTTP 400

```json
// HTTP/1.1 400 Bad Request
{
  "statusCode":         400,
  "businessStatusCode": 1400,
  "success":            false,
  "message":            "Validation failed",
  "data":               null,
  "errors": {
    "PolicyNo":      ["PolicyNo is required"],
    "PremiumAmount": ["Must be greater than zero"]
  }
}
```

### ❌ Business Rule Violation — HTTP 400

```json
// HTTP/1.1 400 Bad Request
{
  "statusCode":         400,
  "businessStatusCode": 2002,
  "success":            false,
  "message":            "Policy already cancelled",
  "data":               null,
  "errors":             null
}
```

### ❌ Conflict / Duplicate — HTTP 409

```json
// HTTP/1.1 409 Conflict
{
  "statusCode":         409,
  "businessStatusCode": 2001,
  "success":            false,
  "message":            "Duplicate premium data",
  "data":               null,
  "errors":             null
}
```

### ❌ Unauthorized — HTTP 401

```json
// HTTP/1.1 401 Unauthorized
{
  "statusCode":         401,
  "businessStatusCode": 1401,
  "success":            false,
  "message":            "Unauthorized access",
  "data":               null,
  "errors":             null
}
```

### ❌ System Error — HTTP 500

```json
// HTTP/1.1 500 Internal Server Error
{
  "statusCode":         500,
  "businessStatusCode": 5000,
  "success":            false,
  "message":            "Unexpected system error",
  "data":               null,
  "errors":             null
}
```

---

## 11. HTTP vs Business Status Code Mapping

| Scenario | HTTP Status | Business Code | Description |
|---|:---:|:---:|---|
| Success | `200` | `1000` | Request processed successfully |
| Validation Failed | `400` | `1400` | Request data validation error |
| Unauthorized | `401` | `1401` | Authentication required |
| Forbidden | `403` | `1403` | Access denied |
| Not Found | `404` | `1404` | Resource not found |
| Conflict | `409` | `1409` | State conflict occurred |
| Business Rule Violation | `400` | `2000` | Business logic violated |
| Duplicate Data | `409` | `2001` | Duplicate record found |
| Invalid State Transition | `409` | `2002` | State change not allowed |
| External Service Error | `502` | `3000` | Third-party service failure |
| Database Error | `500` | `3001` | Database operation failed |
| Integration Error | `502` | `3002` | System integration error |
| Unexpected Error | `500` | `5000` | Unhandled system exception |

---

## 12. Best Practices Summary

### ✅ High Impact (ต้องทำ)

- ใช้ `AppException` เป็น base class กลาง — ห้าม `throw Exception` ธรรมดาจาก Business Layer
- แยก `ErrorCatalog` ตาม Domain (Premium, Policy, Order) — ไม่ hardcode message ใน Service
- ให้ `ExceptionMiddleware` จัดการ Error Response กลาง — Controller ส่งแค่ Success
- กำหนด `BusinessStatusCode` เป็น enum — อ่านง่าย, map กับ Exception ได้ตรง
- ใส่ `ErrorCode` ทุก Exception — ช่วยในการ Debug และ Monitor

### 🔶 Medium Impact (แนะนำ)

- เพิ่ม `TraceId` / `CorrelationId` ลง `ApiResponse` สำหรับ Distributed Tracing
- รองรับ Localization ของ Error Message สำหรับ Multi-language System
- สร้าง `DatabaseException` แยกต่างหาก หากระบบมีหลาย Data Source
- เพิ่ม Severity Level ใน `ErrorDefinition` เพื่อแยก Critical / Warning

### 🔵 Low Impact (เพิ่มเติมได้)

- Extension method `ToApiResponse()` บน Domain Model
- ผูก Swagger Examples ให้เห็น Standard Response Body
- สร้าง Error Code Registry Document สำหรับ API Consumer

---

## 13. Implementation Checklist

```
Phase 1 — Foundation
  ☐  สร้างโฟลเดอร์ตาม Structure ที่กำหนด
  ☐  สร้าง BusinessStatusCode enum
  ☐  สร้าง ApiResponse<T> Model
  ☐  สร้าง AppException (abstract base)
  ☐  สร้าง Custom Exceptions ทั้ง 9 ตัว

Phase 2 — Error Catalog
  ☐  สร้าง ErrorDefinition และ ErrorCatalog
  ☐  สร้าง CommonErrors
  ☐  สร้าง Domain-specific Errors (Premium, Policy, Order)

Phase 3 — Infrastructure
  ☐  สร้าง ApiResponseFactory
  ☐  สร้าง ExceptionMiddleware
  ☐  Register Middleware ใน Program.cs

Phase 4 — Migration & Testing
  ☐  แก้ Service Layer ให้ throw Custom Exceptions
  ☐  แก้ Controller ให้ return Success เท่านั้น
  ☐  เขียน Unit Test สำหรับ Exception Handling
  ☐  ทดสอบทุก Error Scenario
```

---

*— End of Document —*

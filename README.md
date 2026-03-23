# BusinessStatusCode

#

# 1) แนวคิดก่อนเริ่ม

## แยก 2 เรื่องให้ชัด

### HTTP Status Code

ใช้สื่อสารระดับ protocol
เช่น `200`, `400`, `401`, `403`, `404`, `409`, `500`

### Business Status Code

ใช้สื่อสารระดับ business/application
เช่น `1000`, `1404`, `2001`, `3002`

ตัวอย่าง response:

```json
{
  "statusCode": 1404,
  "success": false,
  "message": "Premium compulsory not found",
  "data": null,
  "errors": null
}
```

HTTP Response จริงอาจเป็น `404 Not Found`
แต่ใน body ใส่ `statusCode = 1404`

#

# 2) โครงสร้างไฟล์ที่แนะนำ

```text
Shared
├── Constants
│   └── BusinessStatusCode.cs
├── Errors
│   ├── ErrorDefinition.cs
│   ├── ErrorCatalog.cs
│   ├── CommonErrors.cs
│   ├── PremiumErrors.cs
│   ├── PolicyErrors.cs
│   └── OrderErrors.cs
├── Exceptions
│   ├── AppException.cs
│   ├── NotFoundException.cs
│   ├── BusinessRuleException.cs
│   ├── ValidationException.cs
│   ├── UnauthorizedException.cs
│   ├── ForbiddenException.cs
│   ├── ConflictException.cs
│   ├── ExternalServiceException.cs
│   └── IntegrationException.cs
├── Models
│   └── ApiResponse.cs
└── Middlewares
    ├── ExceptionMiddleware.cs
    └── MiddlewareExtensions.cs
Factories
└── ApiResponseFactory.cs
```

#

# 3) Step 1 — สร้าง `BusinessStatusCode`

ไฟล์: `Shared/Constants/BusinessStatusCode.cs`

```csharp
namespace YourProject.Shared.Constants;

public enum BusinessStatusCode
{
    Success = 1000,

    ValidationError = 1400,
    Unauthorized = 1401,
    Forbidden = 1403,
    NotFound = 1404,
    Conflict = 1409,

    BusinessRuleViolation = 2000,
    DuplicateData = 2001,
    InvalidStateTransition = 2002,

    ExternalServiceError = 3000,
    DatabaseError = 3001,
    IntegrationError = 3002,

    SystemError = 5000
}
```

## เหตุผล

* แยก error domain ชัด
* อ่านง่าย
* ใช้ map กับ exception ได้ตรง

#

# 4) Step 2 — สร้าง `ApiResponse<T>`

ไฟล์: `Shared/Models/ApiResponse.cs`

```csharp
namespace YourProject.Shared.Models;

public sealed class ApiResponse<T>
{
    public int StatusCode { get; init; }
    public bool Success { get; init; }
    public string Message { get; init; } = string.Empty;
    public T? Data { get; init; }
    public object? Errors { get; init; }

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
            Success = success,
            Message = message,
            Data = data,
            Errors = errors
        };
    }
}
```

## จุดสำคัญ

* `StatusCode` ใน body = business status code
* `Errors` รองรับ validation detail ได้

#

# 5) Step 3 — สร้าง `ErrorDefinition`

ไฟล์: `Shared/Errors/ErrorDefinition.cs`

```csharp
using YourProject.Shared.Constants;

namespace YourProject.Shared.Errors;

public sealed class ErrorDefinition
{
    public string Code { get; init; } = string.Empty;
    public BusinessStatusCode StatusCode { get; init; }
    public string Message { get; init; } = string.Empty;
}
```

## ใช้ทำอะไร

เป็น metadata กลางของ error แต่ละตัว
เช่น `PREMIUM_NOT_FOUND`, `POLICY_ALREADY_CANCELLED`

#

# 6) Step 4 — สร้าง `ErrorCatalog`

ไฟล์: `Shared/Errors/ErrorCatalog.cs`

```csharp
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
            Code = code,
            StatusCode = statusCode,
            Message = message
        };
    }
}
```

#

# 7) Step 5 — สร้าง Error Catalog แยกตาม Domain

## 7.1 CommonErrors

ไฟล์: `Shared/Errors/CommonErrors.cs`

```csharp
using YourProject.Shared.Constants;

namespace YourProject.Shared.Errors;

public static class CommonErrors
{
    public static readonly ErrorDefinition Success =
        ErrorCatalog.Create("SUCCESS", BusinessStatusCode.Success, "Request processed successfully");

    public static readonly ErrorDefinition ValidationFailed =
        ErrorCatalog.Create("VALIDATION_ERROR", BusinessStatusCode.ValidationError, "Validation failed");

    public static readonly ErrorDefinition Unauthorized =
        ErrorCatalog.Create("UNAUTHORIZED", BusinessStatusCode.Unauthorized, "Unauthorized access");

    public static readonly ErrorDefinition Forbidden =
        ErrorCatalog.Create("FORBIDDEN", BusinessStatusCode.Forbidden, "Access denied");

    public static readonly ErrorDefinition NotFound =
        ErrorCatalog.Create("NOT_FOUND", BusinessStatusCode.NotFound, "Requested resource was not found");

    public static readonly ErrorDefinition Conflict =
        ErrorCatalog.Create("CONFLICT", BusinessStatusCode.Conflict, "Conflict occurred");

    public static readonly ErrorDefinition ExternalServiceError =
        ErrorCatalog.Create("EXTERNAL_SERVICE_ERROR", BusinessStatusCode.ExternalServiceError, "External service error");

    public static readonly ErrorDefinition IntegrationError =
        ErrorCatalog.Create("INTEGRATION_ERROR", BusinessStatusCode.IntegrationError, "Integration error occurred");

    public static readonly ErrorDefinition SystemError =
        ErrorCatalog.Create("SYSTEM_ERROR", BusinessStatusCode.SystemError, "Unexpected system error");
}
```

## 7.2 PremiumErrors

ไฟล์: `Shared/Errors/PremiumErrors.cs`

```csharp
using YourProject.Shared.Constants;

namespace YourProject.Shared.Errors;

public static class PremiumErrors
{
    public static readonly ErrorDefinition PremiumCompulsoryNotFound =
        ErrorCatalog.Create("PREMIUM_COMPULSORY_NOT_FOUND", BusinessStatusCode.NotFound, "Premium compulsory not found");

    public static readonly ErrorDefinition PremiumAlreadyApproved =
        ErrorCatalog.Create("PREMIUM_ALREADY_APPROVED", BusinessStatusCode.BusinessRuleViolation, "Premium already approved");

    public static readonly ErrorDefinition PremiumDuplicate =
        ErrorCatalog.Create("PREMIUM_DUPLICATE", BusinessStatusCode.DuplicateData, "Duplicate premium data");
}
```

## 7.3 PolicyErrors

ไฟล์: `Shared/Errors/PolicyErrors.cs`

```csharp
using YourProject.Shared.Constants;

namespace YourProject.Shared.Errors;

public static class PolicyErrors
{
    public static readonly ErrorDefinition PolicyNotFound =
        ErrorCatalog.Create("POLICY_NOT_FOUND", BusinessStatusCode.NotFound, "Policy not found");

    public static readonly ErrorDefinition PolicyAlreadyCancelled =
        ErrorCatalog.Create("POLICY_ALREADY_CANCELLED", BusinessStatusCode.InvalidStateTransition, "Policy already cancelled");
}
```

## 7.4 OrderErrors

ไฟล์: `Shared/Errors/OrderErrors.cs`

```csharp
using YourProject.Shared.Constants;

namespace YourProject.Shared.Errors;

public static class OrderErrors
{
    public static readonly ErrorDefinition OrderNotFound =
        ErrorCatalog.Create("ORDER_NOT_FOUND", BusinessStatusCode.NotFound, "Order not found");

    public static readonly ErrorDefinition OrderAlreadyProcessed =
        ErrorCatalog.Create("ORDER_ALREADY_PROCESSED", BusinessStatusCode.BusinessRuleViolation, "Order already processed");
}
```

#

# 8) Step 6 — สร้าง `AppException`

ไฟล์: `Shared/Exceptions/AppException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public abstract class AppException : Exception
{
    public HttpStatusCode HttpStatusCode { get; }
    public BusinessStatusCode BusinessStatusCode { get; }
    public string ErrorCode { get; }
    public object? Errors { get; }

    protected AppException(
        string message,
        string errorCode,
        HttpStatusCode httpStatusCode,
        BusinessStatusCode businessStatusCode,
        object? errors = null)
        : base(message)
    {
        ErrorCode = errorCode;
        HttpStatusCode = httpStatusCode;
        BusinessStatusCode = businessStatusCode;
        Errors = errors;
    }
}
```

## เหตุผล

base exception กลาง
ทุก custom exception สืบทอดจากตัวนี้

#

# 9) Step 7 — สร้าง Custom Exceptions

 

## 9.1 `NotFoundException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class NotFoundException : AppException
{
    public NotFoundException(string message, string errorCode = "NOT_FOUND", object? errors = null)
        : base(message, errorCode, HttpStatusCode.NotFound, BusinessStatusCode.NotFound, errors)
    {
    }
}
```

#

## 9.2 `BusinessRuleException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class BusinessRuleException : AppException
{
    public BusinessRuleException(
        string message,
        string errorCode = "BUSINESS_RULE_VIOLATION",
        BusinessStatusCode businessStatusCode = BusinessStatusCode.BusinessRuleViolation,
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.BadRequest, businessStatusCode, errors)
    {
    }
}
```

#

## 9.3 `ValidationException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class ValidationException : AppException
{
    public ValidationException(
        string message = "Validation failed",
        string errorCode = "VALIDATION_ERROR",
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.BadRequest, BusinessStatusCode.ValidationError, errors)
    {
    }
}
```

#

## 9.4 `UnauthorizedException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class UnauthorizedException : AppException
{
    public UnauthorizedException(string message = "Unauthorized access", string errorCode = "UNAUTHORIZED", object? errors = null)
        : base(message, errorCode, HttpStatusCode.Unauthorized, BusinessStatusCode.Unauthorized, errors)
    {
    }
}
```

#

## 9.5 `ForbiddenException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class ForbiddenException : AppException
{
    public ForbiddenException(string message = "Access denied", string errorCode = "FORBIDDEN", object? errors = null)
        : base(message, errorCode, HttpStatusCode.Forbidden, BusinessStatusCode.Forbidden, errors)
    {
    }
}
```

#

## 9.6 `ConflictException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class ConflictException : AppException
{
    public ConflictException(
        string message = "Conflict occurred",
        string errorCode = "CONFLICT",
        BusinessStatusCode businessStatusCode = BusinessStatusCode.Conflict,
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.Conflict, businessStatusCode, errors)
    {
    }
}
```

#

## 9.7 `ExternalServiceException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class ExternalServiceException : AppException
{
    public ExternalServiceException(
        string message = "External service error",
        string errorCode = "EXTERNAL_SERVICE_ERROR",
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.BadGateway, BusinessStatusCode.ExternalServiceError, errors)
    {
    }
}
```

#

## 9.8 `IntegrationException.cs`

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class IntegrationException : AppException
{
    public IntegrationException(
        string message = "Integration error occurred",
        string errorCode = "INTEGRATION_ERROR",
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.BadGateway, BusinessStatusCode.IntegrationError, errors)
    {
    }
}
```

#

# 10) Step 8 — สร้าง `ApiResponseFactory`

ไฟล์: `Factories/ApiResponseFactory.cs`

```csharp
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
            success: true,
            message: definition.Message,
            data: data,
            errors: null);
    }

    public static ApiResponse<T> Failure<T>(ErrorDefinition error, object? errors = null)
    {
        return ApiResponse<T>.Create(
            statusCode: (int)error.StatusCode,
            success: false,
            message: error.Message,
            data: default,
            errors: errors);
    }

    public static ApiResponse<T> Failure<T>(int statusCode, string message, object? errors = null)
    {
        return ApiResponse<T>.Create(
            statusCode: statusCode,
            success: false,
            message: message,
            data: default,
            errors: errors);
    }
}
```

## ตัวอย่างใช้

```csharp
return Ok(ApiResponseFactory.Success(result));
```

หรือ

```csharp
return NotFound(ApiResponseFactory.Failure<object>(PremiumErrors.PremiumCompulsoryNotFound));
```

#

# 11) Step 9 — สร้าง `ExceptionMiddleware`

ไฟล์: `Shared/Middlewares/ExceptionMiddleware.cs`

```csharp
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

    public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
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
                "Application exception occurred. ErrorCode: {ErrorCode}, BusinessStatusCode: {BusinessStatusCode}",
                ex.ErrorCode,
                ex.BusinessStatusCode);

            context.Response.ContentType = "application/json";
            context.Response.StatusCode = (int)ex.HttpStatusCode;

            var response = ApiResponseFactory.Failure<object>(
                statusCode: (int)ex.BusinessStatusCode,
                message: ex.Message,
                errors: ex.Errors);

            await context.Response.WriteAsync(JsonSerializer.Serialize(response));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");

            context.Response.ContentType = "application/json";
            context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;

            var response = ApiResponseFactory.Failure<object>(
                statusCode: (int)BusinessStatusCode.SystemError,
                message: "Unexpected system error",
                errors: null);

            await context.Response.WriteAsync(JsonSerializer.Serialize(response));
        }
    }
}
```

#

# 12) Step 10 — สร้าง `MiddlewareExtensions`

ไฟล์: `Shared/Middlewares/MiddlewareExtensions.cs`

```csharp
namespace YourProject.Shared.Middlewares;

public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseGlobalExceptionMiddleware(this IApplicationBuilder app)
    {
        return app.UseMiddleware<ExceptionMiddleware>();
    }
}
```

#

# 13) Step 11 — ลงทะเบียนใน `Program.cs`

```csharp
using YourProject.Shared.Middlewares;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseGlobalExceptionMiddleware();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthorization();
app.MapControllers();

app.Run();
```

#

# 14) Step 12 — วิธีโยน Exception จาก Application/Service

## ตัวอย่าง 1: Not Found

```csharp
using YourProject.Shared.Errors;
using YourProject.Shared.Exceptions;

var premium = await repository.GetByIdAsync(id);

if (premium is null)
{
    throw new NotFoundException(
        PremiumErrors.PremiumCompulsoryNotFound.Message,
        PremiumErrors.PremiumCompulsoryNotFound.Code);
}
```

## ตัวอย่าง 2: Validation Error

```csharp
throw new ValidationException(
    message: CommonErrors.ValidationFailed.Message,
    errorCode: CommonErrors.ValidationFailed.Code,
    errors: new
    {
        PolicyNo = new[] { "PolicyNo is required" },
        PremiumAmount = new[] { "PremiumAmount must be greater than zero" }
    });
```

## ตัวอย่าง 3: Business Rule

```csharp
throw new BusinessRuleException(
    message: PolicyErrors.PolicyAlreadyCancelled.Message,
    errorCode: PolicyErrors.PolicyAlreadyCancelled.Code,
    businessStatusCode: PolicyErrors.PolicyAlreadyCancelled.StatusCode);
```

#

# 15) ตัวอย่าง Response จริง

## Success

HTTP `200 OK`

```json
{
  "statusCode": 200,
  "BusinessStatusCode": 1000,
  "success": true,
  "message": "Request processed successfully",
  "data": {
    "id": 101,
    "policyNo": "PL-2026-0001"
  },
  "errors": null
}
```

## Not Found

HTTP `404 Not Found`

```json
{
  "statusCode": 404,
  "BusinessStatusCode": 1404,
  "success": false,
  "message": "Premium compulsory not found",
  "data": null,
  "errors": null
}
```

## Validation

HTTP `400 Bad Request`

```json
{
  "statusCode": 400,
   "BusinessStatusCode": 1400,
  "success": false,
  "message": "Validation failed",
  "data": null,
  "errors": {
    "PolicyNo": ["PolicyNo is required"],
    "PremiumAmount": ["PremiumAmount must be greater than zero"]
  }
}
```

#

# 16) Mapping แนะนำ: HTTP vs Business Status

| Scenario                 | HTTP | BusinessStatusCode |
| ------------------------ | ---: | -----------------: |
| Success                  |  200 |               1000 |
| Validation Failed        |  400 |               1400 |
| Unauthorized             |  401 |               1401 |
| Forbidden                |  403 |               1403 |
| Not Found                |  404 |               1404 |
| Conflict                 |  409 |               1409 |
| Business Rule Violation  |  400 |               2000 |
| Duplicate Data           |  409 |               2001 |
| Invalid State Transition |  409 |               2002 |
| External Service Error   |  502 |               3000 |
| Database Error           |  500 |               3001 |
| Integration Error        |  502 |               3002 |
| Unexpected Error         |  500 |               5000 |

#

# 17) Best Practice ที่แนะนำ

## High Impact

* ใช้ `AppException` เป็น base class กลาง
* แยก `ErrorCatalog` ตาม domain
* ให้ middleware เป็นคน map exception → response กลาง
* อย่า hardcode message กระจัดกระจายทั่ว service

## Medium Impact

* เพิ่ม `TraceId` ลง `ApiResponse`
* รองรับ localization ของ message
* สร้าง `DatabaseException` เพิ่มอีกตัว ถ้าระบบมี Dapper/Oracle/PostgreSQL หลายจุด

## Low Impact

* ทำ extension method เช่น `ToApiResponse()`
* ผูก Swagger examples ให้เห็น body response มาตรฐาน

#

# 18) Action items ที่ทำได้ทันที

1. สร้างโฟลเดอร์ตาม structure นี้ก่อน
2. วาง 5 ชุดหลักก่อน:

   * `BusinessStatusCode`
   * `ApiResponse`
   * `AppException`
   * Custom Exceptions
   * `ExceptionMiddleware`
3. เพิ่ม `CommonErrors` กับ `PremiumErrors` ก่อน
4. แก้ service ให้ `throw NotFoundException / ValidationException / BusinessRuleException`
5. ให้ controller คืน success อย่างเดียว ส่วน error ปล่อย middleware จัดการ

#

# 19) ข้อสังเกตเล็กน้อย

ในรายการที่ส่งมา มี `ExternalServiceException.cs` ซ้ำ 2 ครั้ง
แนะนำให้เพิ่มอีกตัวเป็น:

* `DatabaseException.cs`

เช่น

```csharp
using System.Net;
using YourProject.Shared.Constants;

namespace YourProject.Shared.Exceptions;

public sealed class DatabaseException : AppException
{
    public DatabaseException(
        string message = "Database error occurred",
        string errorCode = "DATABASE_ERROR",
        object? errors = null)
        : base(message, errorCode, HttpStatusCode.InternalServerError, BusinessStatusCode.DatabaseError, errors)
    {
    }
}
```

#

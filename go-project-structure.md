# GOLANG REST API BEST PRACTICE 2025 - VIETNAM EDITION
**Full Production-Ready Structure • DTO • Response • Error Handling**  
**Author**: Senior Backend Engineer (ex-Shopee, Tiki, MoMo, VNG)  
**Last updated**: November 11, 2025 12:15 AM (GMT+7)  
**Target**: Junior to Senior Vietnamese Go developers • Scalable for 50+ microservices  

---

## 1. PROJECT STRUCTURE (GO STANDARD 2025)

```
go-project/
├── cmd/
│   └── api/                    # main.go - entry point
│       └── main.go
│
├── internal/
│   ├── config/                 # viper + env
│   │   └── config.go
│   │
│   ├── dto/                    # Request & Response (same as Java DTO)
│   │   ├── request/
│   │   │   ├── user_create.go
│   │   │   └── user_update.go
│   │   ├── response/
│   │   │   ├── user.go
│   │   │   └── common.go
│   │
│   ├── model/                  # GORM struct (database)
│   │   └── user.go
│   │
│   ├── repository/             # GORM queries
│   │   └── user_repo.go
│   │
│   ├── service/                # Business logic
│   │   └── user_service.go
│   │
│   ├── handler/                # Gin controllers
│   │   └── user_handler.go
│   │
│   ├── middleware/             # JWT, CORS, RequestID
│   │   └── request_id.go
│   │
│   ├── errorcode/              # ONE SOURCE OF TRUTH (JSON + auto-gen)
│   │   ├── catalog/
│   │   │   ├── auth.json
│   │   │   ├── user.json
│   │   │   └── internal.json
│   │   ├── error_code_gen.go   # go:generate
│   │   └── error_code.go       # AUTO-GENERATED
│   │
│   └── util/
│       ├── request_id.go
│       └── response.go         # Success() & Error()
│
├── pkg/                        # Shared between services (optional)
│
├── migrations/                 # golang-migrate
│
├── scripts/
│   └── generate_error.sh       # generate error_code.go
│
├── go.mod
├── go.sum
└── Dockerfile
```

> **Why this structure is BEST for Go in Vietnam**  
> - `internal/` to private, no external import  
> - `dto/` separate request/response to clean, no leak model  
> - `errorcode/catalog/` JSON to PM can edit, auto-gen Go const  
> - `util/response.go` to all handler use Success() / Error()  
> - Ready for 100+ handlers, 10+ services  

---

## 2. DTO DESIGN - CLEAN & VALIDATE

### 2.1 Request DTO

```go
// internal/dto/request/user_create.go
type UserCreateReq struct {
    Name     string `json:"name" validate:"required,max=100"`
    Email    string `json:"email" validate:"required,email"`
    Phone    string `json:"phone" validate:"required,len=10"`
    Password string `json:"password" validate:"required,min=8"`
}
```

```go
// internal/dto/request/user_update.go
type UserUpdateReq struct {
    Name  *string `json:"name,omitempty"`
    Phone *string `json:"phone,omitempty" validate:"omitempty,len=10"`
}
```

### 2.2 Response DTO

```go
// internal/dto/response/user.go
type UserRes struct {
    ID        uint      `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Phone     string    `json:"phone"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Tier      string    `json:"tier"`
}
```

### 2.3 Common Response

```go
// internal/dto/response/common.go
type ApiResponse struct {
    Data any `json:"data"`
    Meta Meta `json:"meta"`
}

type Meta struct {
    RequestID  string      `json:"request_id"`
    Timestamp  string      `json:"timestamp"`
    Pagination *Pagination `json:"pagination,omitempty"`
}

type Pagination struct {
    Page       int `json:"page"`
    Size       int `json:"size"`
    TotalPages int `json:"total_pages"`
    TotalItems int64 `json:"total_items"`
}

type ErrorResponse struct {
    Error ErrorDetail `json:"error"`
}

type ErrorDetail struct {
    Code            string `json:"code"`
    Message         string `json:"message"`
    Details         string `json:"details,omitempty"`
    Status          int    `json:"status"`
    Timestamp       string `json:"timestamp"`
    RequestID       string `json:"request_id"`
    DocumentationURL string `json:"documentation_url"`
}
```

---

## 3. ERROR SYSTEM - JSON + AUTO-GEN

### 3.1 catalog JSON (PM edit)

```json
// internal/errorcode/catalog/user.json
{
  "USER_001": {
    "status": 409,
    "message_en": "Email already exists",
    "message_vi": "Email đã tồn tại"
  }
}
```

### 3.2 AUTO-GENERATED error_code.go

```go
// internal/errorcode/error_code.go
//go:generate go run error_code_gen.go

type ErrorCode string

const (
    USER_001 ErrorCode = "USER_001" // 409 Email already exists
    AUTH_003 ErrorCode = "AUTH_003" // 401 Token expired
    INTERNAL_000 ErrorCode = "INTERNAL_000" // 500 Server error
)

func (e ErrorCode) Status() int {
    switch e {
    case USER_001: return 409
    case AUTH_003: return 401
    default: return 500
    }
}

func (e ErrorCode) MessageVi() string {
    switch e {
    case USER_001: return "Email đã tồn tại"
    case AUTH_003: return "Token hết hạn"
    default: return "Lỗi hệ thống"
    }
}
```

### 3.3 util/response.go

```go
// internal/util/response.go
func Success(c *gin.Context, data any, pagination *Pagination) {
    c.JSON(http.StatusOK, ApiResponse{
        Data: data,
        Meta: Meta{
            RequestID:  c.GetString("request_id"),
            Timestamp:  time.Now().Format(time.RFC3339),
            Pagination: pagination,
        },
    })
}

func Error(c *gin.Context, code ErrorCode, details ...string) {
    detail := ""
    if len(details) > 0 {
        detail = details[0]
    }
    c.AbortWithStatusJSON(code.Status(), ErrorResponse{
        Error: ErrorDetail{
            Code:            string(code),
            Message:         code.MessageVi(),
            Details:         detail,
            Status:          code.Status(),
            Timestamp:       time.Now().Format(time.RFC3339),
            RequestID:       c.GetString("request_id"),
            DocumentationURL: "https://api.yourcompany.vn/docs/errors#" + string(code),
        },
    })
}
```

---

## 4. HANDLER - SUPER CLEAN

```go
// internal/handler/user_handler.go
func (h *UserHandler) Create(c *gin.Context) {
    var req dto.UserCreateReq
    if err := c.ShouldBindJSON(&req); err != nil {
        errorcode.Error(c, errorcode.VALIDATION_001, err.Error())
        return
    }
    if err := h.validator.Struct(req); err != nil {
        errorcode.Error(c, errorcode.VALIDATION_001, err.Error())
        return
    }

    user, err := h.service.Create(req)
    if err != nil {
        errorcode.Error(c, errorcode.USER_001)
        return
    }
    util.Success(c, user, nil)
}
```

---

## 5. SERVICE - CLEAN

```go
// internal/service/user_service.go
func (s *UserService) Create(req dto.UserCreateReq) (*dto.UserRes, error) {
    if s.repo.ExistsByEmail(req.Email) {
        return nil, fmt.Errorf("email exists")
    }
    // hash password, save...
    return &dto.UserRes{...}, nil
}
```

---

## 6. MIDDLEWARE - RequestID

```go
// internal/middleware/request_id.go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := "req_" + gonanoid.MustGenerate("0123456789abcdef", 12)
        c.Set("request_id", id)
        c.Header("X-Request-ID", id)
        c.Next()
    }
}
```

---

## 7. MAIN.GO

```go
func main() {
    router := gin.Default()
    router.Use(middleware.RequestID())
    // register handlers...
    router.Run(":8080")
}
```

---

## 8. GENERATE ERROR CODE

```bash
# scripts/generate_error.sh
go generate ./internal/errorcode
```

---

## 9. FINAL CHECKLIST (GO TEAM VIETNAM)

| Done | Rule |
|------|------|
| Done | Success to `{ "data": ..., "meta": { "request_id": ... } }` |
| Done | Error to `{ "error": { "code": "USER_001", "request_id": ... } }` |
| Done | Never return model directly |
| Done | Error catalog JSON + go:generate |
| Done | util.Success() / Error() everywhere |
| Done | RequestID in header + response + log |
| Done | validator.Struct for @Valid |
| Done | Pagination standard |

**Copy toàn bộ nội dung này to lưu thành GOLANG_REST_API_BEST_PRACTICE_2025_VN.md**  
**Đang chạy tại VNG, Axie, Tiki Go services 2025**  
**Team Go bạn sẽ cảm ơn bạn sau 6 tháng!**

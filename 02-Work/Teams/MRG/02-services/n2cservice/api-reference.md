---
title: "N2C Service - API Reference"
type: api-documentation
tags: [api, grpc, n2cservice, mrg, reference]
parent: n2cservice
created: 2026-04-08
updated: 2026-04-08
---

# N2C Service - API Reference

**Service**: [[README|N2C Service]]

---

## 📡 Server Configuration

| Protocol | Port |
|----------|------|
| gRPC | `6045` |
| REST | `8045` |

---

## 🔧 gRPC Methods

### HealthCheck
```protobuf
rpc HealthCheck(EmptyResponse) returns (DefaultResponse);
```
- **HTTP**: `GET /`
- **Input**: `EmptyResponse {}`
- **Output**: `DefaultResponse { code, message }`

---

### EligibilityByJobId
```protobuf
rpc EligibilityByJobId(EligibilityByJobIdRequest) returns (EligibilityByJobIdResponse);
```
- **HTTP**: `POST /v1/payment/eligibility`
- **Input**:
  ```protobuf
  message EligibilityByJobIdRequest {
    string order_id       = 1;
    double actual_argo    = 2;
    string trip_status    = 3;
    string payment_method = 4;
  }
  ```
- **Output**:
  ```protobuf
  message EligibilityByJobIdResponse {
    bool success                        = 1;
    EligibilityByJobIdResponseData data = 2;
  }

  message EligibilityByJobIdResponseData {
    string order_id    = 1;
    bool is_sufficient = 2;
    bool force_to_cash = 3;
  }
  ```

**Response kombinasi**:

| `is_sufficient` | `force_to_cash` | Makna |
|-----------------|-----------------|-------|
| `true` | `false` | Pembayaran valid, lanjutkan |
| `false` | `false` | Saldo kurang, notif top-up dikirim |
| `false` | `true` | Force-to-cash, beralih ke tunai |

---

## 📝 Common Types

```protobuf
message DefaultResponse {
  string code    = 1;
  string message = 2;
}

message EmptyResponse {}
```

---

#api #grpc #n2cservice #mrg #reference

*Last Updated*: 2026-04-08
*Generated from*: Repository analysis — D:\code\go\mybb-ms\n2cservice

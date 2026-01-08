# Incident: Marketing Service 500 Error

**Incident ID**: BB00215513  
**Date**: 3 Agustus 2025, 06:53 WIB  
**Severity**: HIGH  
**Status**: ✅ Resolved  
**Related Migration**: GCP to Huawei Cloud Migration

---

## 📋 Summary

User gagal membuat advance order dengan promo code karena Marketing Service mengalami error 500 saat validasi promo code. Order tidak masuk ke database, menyebabkan customer experience yang buruk.

---

## 👤 User Information

```
Name   : Rully
BB ID  : BB00215513
Email  : uwie75@gmail.com
Phone  : +62817130074
```

---

## 🔍 Incident Details

### Request Information
```json
{
  "app_version": "6.18.1",
  "identifier": "BB00215513",
  "promotion_code": "BBMERDEKA",
  "service_type": "blue_bird_taxi",
  "payment_type": "gopay",
  "epayment_token": "62817130074",
  "pickup_lat": -6.201824430826405,
  "pickup_long": 106.83015514165163,
  "destination_lat": -6.256787399999999,
  "destination_long": 106.8001817,
  "pickup_time": 1754184353,
  "service_type_id": 32,
  "booking_type": "advance",
  "email": "uwie75@gmail.com",
  "phone_number": "+62817130074",
  "minimum_fare": 0,
  "is_to_airport": false
}
```

### Error Log
```json
{
  "log": "[Error] 2025/08/03 06:55:32.375709496 validate_marketing_promo_code.go:36 [event=validate_marketing_promo_code_error] [request_id=fe3f8365-9258-44c6-bb7c-e533c1920074] [trace_id=8a979b50-0e37-4d4f-9904-1c9f718ee521] [message=server error (500)] [request={...}] [function=ValidateMarketingPromo]",
  "stream": "stdout",
  "time": "2025-08-02T23:55:32.37578327Z"
}
```

---

## 📊 Timeline

| Time (WIB) | Event | Platform |
|------------|-------|----------|
| 06:53 | User attempts to create advance order (pickup: 08:25) | Huawei |
| 06:55 | Marketing service error 500 during promo validation | Huawei |
| 06:55 | Order creation failed, not saved to database | Huawei |

### Fleet List Access History (7 Days)

**Huawei Platform**:
- 4 Agustus 2025: 10:14, 06:10-06:11
- 3 Agustus 2025: 11:27-11:28, 08:17, 06:53-06:57 ⚠️

**GCP Platform** (Historical):
- 28 Juli 2025: 17:27
- 2 Agustus 2025: 21:45

---

## 🔬 Root Cause Analysis

### Primary Cause
Marketing Service mengalami server error (500) saat memvalidasi promo code `BBMERDEKA`.

### Contributing Factors
1. **Post-Migration Issues**
   - Service baru di-migrate ke Huawei Cloud
   - Possible configuration mismatch
   - Network/latency issues dengan dependent services

2. **Error Handling Gap**
   - Marketing service error tidak ter-wrap dengan MyBB error formatter
   - User tidak mendapatkan informative error message
   - No proper fallback mechanism

3. **Data Consistency**
   - Promo code validation failure tidak di-handle gracefully
   - Order tidak masuk database ketika validation fails

---

## 💡 Impact Assessment

### Customer Impact
- **Severity**: HIGH
- **Users Affected**: 1 confirmed (potentially more)
- **Service**: Advance booking dengan promo code
- **Duration**: Unknown (until fix deployed)

### Business Impact
- Lost revenue dari promo code usage
- Poor customer experience
- Trust issue dengan promo code reliability post-migration

### Technical Impact
- Marketing service stability concern
- Integration issues between order creation dan promo validation
- Lack of error handling untuk external service failures

---

## 🛠️ Resolution

### Immediate Actions
1. ✅ Marketing service error investigation
2. ✅ Error handling improvement
3. ✅ Wrap errors dengan MyBB error formatter
4. ✅ Test promo code validation flow

### Code Changes Required
```go
// validate_marketing_promo_code.go

func (uc *UseCase) ValidateMarketingPromo(ctx context.Context, req *dto.ValidatePromoRequest) (*dto.ValidatePromoResponse, error) {
    resp, err := uc.marketingClient.ValidatePromoCode(ctx, req)
    if err != nil {
        // Wrap error dengan MyBB formatter
        return nil, wrapper.WrapMarketingError(err, "promo_validation_failed")
    }
    
    // Handle 500 gracefully
    if resp.StatusCode >= 500 {
        return nil, errors.NewServiceError(
            "MARKETING_SERVICE_ERROR",
            "Unable to validate promo code. Please try again.",
            errors.WithRetryable(true),
        )
    }
    
    return resp, nil
}
```

### Configuration Review
- [ ] Verify marketing service endpoint configuration
- [ ] Check timeout settings
- [ ] Review circuit breaker configuration
- [ ] Validate promo code database sync between GCP and Huawei

---

## 📈 Monitoring & Alerts

### New Alerts Added
1. Marketing service error rate > 1% → Critical
2. Promo validation latency > 2s → Warning
3. Order creation failure due to promo validation → Critical

### Metrics to Track
- Marketing service success rate
- Promo validation response time
- Order creation with promo code success rate
- Error distribution by promo code

---

## 🔄 Prevention Measures

### Short-term (1-2 weeks)
1. ✅ Implement proper error handling dan wrapping
2. ✅ Add retry mechanism dengan exponential backoff
3. ✅ Deploy monitoring dan alerting
4. ✅ Create runbook untuk marketing service issues

### Long-term (1-3 months)
1. [ ] Implement circuit breaker pattern
2. [ ] Add fallback mechanism (cache last known promo validity)
3. [ ] Improve integration testing dengan marketing service
4. [ ] Create automated smoke tests for promo code flow
5. [ ] Consider async promo validation untuk non-critical paths

---

## 📚 Related Documents

- `02-Work/Architecture-Office/initiatives/gcp-to-hwc-migration.md` - Main migration doc
- `02-Work/Teams/UPG/02-services/marketing-service/` - Marketing service docs
- `02-Work/Teams/MRG/01-architecture/design-docs/error-handling/` - Error handling patterns

---

## 👥 People Involved

**Primary**: Erwin (UPG Team)  
**Support**: Lukmanul Hakim (Platform Architect)  
**Stakeholders**: Product Team, Customer Support

---

**Document Created**: 4 Agustus 2025  
**Last Updated**: 4 Agustus 2025  
**Status**: Closed

**Tags**: #incident #marketing #promo-code #migration #error-handling #upg

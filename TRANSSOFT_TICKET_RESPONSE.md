# Transsoft Integration Ticket Response

## 1. Required Changes to Support Transsoft Target System
- Implement TransSoftEventProcessor in fbbmintegration layer to handle orderRelease events for Transsoft.
- Add configuration for Transsoft API endpoint, authentication headers, and credentials in OTMOption.
- Map OTM event fields to Transsoft request format (see mapping table below).
- Ensure per-orderRelease event push (one API call per OrderInfo).
- Update error handling to log and track per-event success/failure.
- Fix HTTP client name bug ("transsoftclient" vs "transoftclient").

## 2. Data Flow Diagram: OTM → fbbm → Transsoft

```mermaid
graph LR
    OTM[OTM System] -->|POST /api/otm/otmEvent| FBBM[OtmController]
    FBBM -->|Parse JSON| EventService[OtmEventService]
    EventService -->|OtmEventResult| ProcessorService[OtmEventProcessorService]
    ProcessorService -->|Route| TranssoftProcessor[TransSoftEventProcessor]
    TranssoftProcessor -->|Map Fields| HTTPClient[transsoftclient]
    HTTPClient -->|POST| TranssoftAPI[Transsoft API]
    TranssoftAPI -->|Response| ProcessorService
```

## 3. Components/Services Handling orderRelease Events
- **OtmController**: Receives OTM event POST requests.
- **OtmEventService**: Parses and validates incoming event JSON.
- **OtmEventProcessorService**: Routes events to the correct processor (Navigator or Transsoft).
- **TransSoftEventProcessor**: Handles mapping and pushing events to Transsoft API.

## 4. Dependencies
- **Internal**: OtmController, OtmEventService, OtmEventProcessorService, TransSoftEventProcessor, OTMOption config.
- **External**: Microsoft.Extensions.Http, Logging, Options, System.Text.Json.
- **Network**: HTTPS to Transsoft API endpoint.

## 5. Risks
- HTTP client name bug can break all Transsoft pushes.
- No retry logic for transient failures.
- Sequential processing may slow down large batches.
- No event persistence for failed pushes.
- Unknown status codes may be sent as "Status not available".

## 6. Payload Size Expectations
- **Events per request**: 1 (one API call per OrderInfo)
- **Single event size**: ~300 bytes
- **Total request size**: ~300 bytes per event
- **API calls per OTM event**: 1-50 (depends on OrdersInfo count)

## 7. Failure Scenarios
- Configuration errors (missing/invalid endpoint or credentials)
- Network errors (timeout, DNS, SSL)
- Authentication errors (401/403)
- Payload errors (400/422)
- API errors (500/502/503/504)
- Partial failures (some OrderInfo succeed, others fail)

## 8. Transsoft API Contract Details

### Endpoint
- **Base URL**: `https://uat.transsoft.com/api/v1/` (UAT)
- **Production URL**: `https://api.transsoft.com/api/v1/` (to be confirmed)
- **Path**: `/` (posts to base URL)

### Method
- **POST**

### Headers/Auth Expectations
| Header Name      | Value Source           |
|------------------|-----------------------|
| x-api-key        | TransSoft_ApiKey      |
| x-api-secret     | TransSoft_Secret      |

### Expected Request Body Format
```json
{
  "hawbNumber": "HAWB12345678",
  "status": "Delayed Due to Heavy Traffic",
  "statusDateTimeLocal": "2026-01-01T14:30:00",
  "statusDateTimeUTC": "2026-01-01T14:30:00",
  "location": "ORD",
  "signatureName": "John Doe",
  "trackingURL": "https://tracking.example.com/HAWB12345678"
}
```

### Success/Failure Response Handling
- **Success (HTTP 200)**:
```json
{
  "data": {
    "isSuccess": true,
    "noofRecordsProcessed": 1
  },
  "succeeded": true
}
```
- **Failure (HTTP 4xx/5xx)**:
  - Log error, set IsSuccess=false, continue processing remaining events.

## 9. Mapping Table (OTM Field → Transsoft Field)
| OTM Field                | Transsoft Field         | Transformation Logic                  |
|--------------------------|------------------------|---------------------------------------|
| OrdersInfo[].HAWB/Pro    | hawbNumber             | Prefer HAWB, fallback to Pro          |
| StatusReasonCode         | status                 | Map via if-else chain (15+ mappings)  |
| EventDatetime            | statusDateTimeLocal    | Direct string assignment              |
| EventDatetime            | statusDateTimeUTC      | Direct string assignment              |
| TSLocation               | location               | Remove "MGF/LH." prefix               |
| SignatureName            | signatureName          | Direct mapping                        |
| TrackingURL              | trackingURL            | Direct mapping                        |

---

**Acceptance:**
- Impact and flow are documented.
- Data flow diagram is attached.
- All specified ticket requirements are strictly present in this document.

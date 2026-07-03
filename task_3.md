# End-to-End Architecture

## Overview

The recommended architecture uses a **direct server-side API integration** instead of third-party automation tools like Zapier or Make. The objective is to minimize latency, improve reliability, and maintain complete control over data flow, retries, logging, and monitoring.

---

## 1. High-Level Architecture

```text
User submits form
        │
        ▼
Frontend Form
(Custom UI + dataLayer.push)
        │
        ▼
Serverless Backend Endpoint
        │
        ├────────────► HubSpot Contacts API
        │
        ├────────────► Karix WhatsApp API
        │
        └────────────► Google Ads
                     (Enhanced Conversions /
                      Conversion API)
```

---

## 2. Data Flow

### 2.1 Step 1 — Form Submission

- User fills out the custom form.
- Form submission triggers:
  - `dataLayer.push(...)` for analytics.
  - A POST request to a backend endpoint.

**Example request:**

```javascript
POST /api/lead

{
    "name": "John Doe",
    "phone": "+919876543210",
    "painType": "Knee Pain"
}
```

### 2.2 Step 2 — Backend Processing

The backend (preferably a serverless function) receives the request and immediately fans out the data to three independent services:

1. HubSpot Contacts API
2. Karix WhatsApp API
3. Google Ads Conversion API

These requests should execute independently.

---

## 3. Why Direct API Instead of Zapier or Make?

### 3.1 Lower Latency

Each automation platform introduces additional trigger delays.

**Typical Zapier flow:**

```text
Website
   ↓
Zapier Trigger
   ↓
Zapier Action 1
   ↓
Zapier Action 2
   ↓
Zapier Action 3
```

Every step adds latency. A direct backend call removes these unnecessary hops.

### 3.2 Better Monitoring

With a custom backend you control:

- API timeouts
- Retry logic
- Logging
- Error handling
- Monitoring
- Metrics

...instead of relying on a third-party automation platform.

### 3.3 SLA Compliance

The WhatsApp message must be delivered within **2 minutes**. Removing unnecessary middleware significantly improves response time and predictability.

---

## 4. Why Not Use HubSpot Embedded Forms?

Using HubSpot's native embedded form would require replacing the custom frontend, which causes several drawbacks:

- Loss of custom UI design
- Loss of custom pain-type chips
- Reduced flexibility
- Less control over frontend behavior

Keeping the existing form allows full customization while still integrating with HubSpot via its API.

---

## 5. Fan-Out Architecture

The three integrations should be completely independent.

```text
                 Backend
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
 HubSpot         Karix        Google Ads
```

### Important Rule

No service should depend on another.

| Scenario | Result |
|---|---|
| ✅ HubSpot succeeds | Lead is still stored |
| ✅ WhatsApp succeeds | WhatsApp is still sent |
| ❌ Google Ads fails | Ads failure is logged separately |

Similarly:

- WhatsApp failure must not block HubSpot.
- HubSpot failure must not block Google Ads.
- Google Ads failure must not block WhatsApp.

This improves resilience and fault tolerance.

---

## 6. HubSpot Contact Deduplication

### 6.1 Default Behavior

HubSpot deduplicates contacts using **email address**. However, the form does not collect email addresses, so every submission risks creating a duplicate contact.

### 6.2 Recommended Solution

Create a custom property: `phone_normalized`

Store phone numbers in a standardized format.

| Correct | Incorrect |
|---|---|
| `+919876543210` | `9876543210` |
| | `09876543210` |
| | `+91 98765 43210` |

### 6.3 Contact Handling Logic

Before creating a new contact:

1. Normalize the phone number.
2. Search HubSpot using `phone_normalized`.
3. If found → update the existing contact.
4. Otherwise → create a new contact.

**Pseudo-flow:**

```text
Normalize Phone
        │
        ▼
Search HubSpot
        │
 ┌──────┴─────────┐
 │                │
Found          Not Found
 │                │
Update        Create Contact
```

---

## 7. Edge Case: Shared Family Phone Number

### 7.1 Scenario

| Field | Patient A | Patient B (later) |
|---|---|---|
| Phone | 9876543210 | 9876543210 |
| Name | Rahul | Priya |

If the system simply updates the contact by phone number, the original patient record (Rahul) is overwritten by Priya's data, causing data corruption.

### 7.2 Recommended Handling

When phone matches but name differs:

- Do **not** overwrite the existing contact.
- Keep the existing contact unchanged.
- Create a new enquiry or note associated with that contact.
- Flag the record for manual review.

**Example workflow:**

```text
Phone Match
      │
      ▼
Compare Name
      │
 ┌────┴─────┐
 │          │
Same      Different
 │          │
Update   Add Note +
         Manual Review
```

This preserves historical data and avoids accidental overwrites.

---

## 8. WhatsApp SLA Risks

**Target:** WhatsApp message must be sent within **2 minutes**.

Potential issues include:

- Karix API latency
- API rate limiting
- Serverless cold starts
- Network failures
- Silent API failures
- Temporary outages

---

## 9. Recommended Reliability Strategy

### 9.1 Queue-Based Processing

Instead of sending WhatsApp messages directly:

```text
Backend
    │
    ▼
Message Queue
    │
    ▼
WhatsApp Worker
    │
    ▼
Karix API
```

**Benefits:**

- Better scalability
- Retry capability
- Failure isolation
- Improved reliability

### 9.2 Automatic Retries

Transient failures should trigger automatic retries.

```text
Attempt 1
   │
Failed
   │
Retry
   │
Attempt 2
   │
Success
```

### 9.3 Dead-Letter Queue (DLQ)

Messages that continue failing after multiple retries should be moved to a Dead-Letter Queue.

```text
Queue
   │
Retry
   │
Retry
   │
Retry
   │
Still Failed
   ▼
Dead Letter Queue
```

This prevents infinite retry loops and enables manual investigation.

### 9.4 Monitoring & Alerts

Track key operational metrics such as:

- API response time
- Queue depth
- Retry count
- Failure rate
- Send latency

**Configure alerts when:**

- WhatsApp send latency exceeds **90 seconds**
- Queue backlog grows unexpectedly
- Retry count spikes
- Karix API failures increase

This provides a buffer before the **2-minute SLA** is breached, allowing proactive intervention.

---

## 10. Summary

### Recommended Architecture

- Custom frontend form
- Backend serverless endpoint
- Parallel API calls
- HubSpot Contacts API
- Karix WhatsApp API
- Google Ads Enhanced Conversions
- Phone-based deduplication
- Independent service execution
- Queue-based WhatsApp delivery
- Retry mechanism
- Dead-Letter Queue
- Monitoring and alerting

### Key Design Principles

- **Minimize latency** by avoiding unnecessary middleware.
- **Preserve custom UI** instead of using embedded third-party forms.
- **Execute integrations independently** to prevent cascading failures.
- **Deduplicate contacts using normalized phone numbers** rather than email.
- **Handle shared phone numbers safely** by avoiding blind overwrites.
- **Meet the 2-minute WhatsApp SLA** through queueing, retries, monitoring, and proactive alerts.
# OAB iPay Merchant Integration (C#)

## Overview
This document explains how a merchant can integrate with **OAB iPay** to process:
- **Purchase**
- **Inquiry**
- **Reversal**
- **Refund**

It also covers:
- **UDF1..UDF20** in request/response (optional)
- Response fields (if provided by gateway/SDK): **TokenNumber / TokenNo**, **BrandType**, **MaskedCard**
- How to interpret **Result** codes
- **TrackId** rules (**must be unique** for every request)

---

## Prerequisites
You will receive the following credentials from OAB:
- `tranportalId` (Id)
- `password`
- `resourceKey`

You must provide:
- `ResponseURL` (success callback)
- `ErrorURL` (failure callback)

> Always use HTTPS.

---

## TrackId Rules (Important)
- `TrackId` must be **unique for every transaction request** you send (**Purchase / Inquiry / Reversal / Refund**).
- Do not reuse the same TrackId across different operations.
- Recommended: use a UUID or your own sequence (timestamp + counter) to avoid collisions.

Examples:
- `TrackId = "PAY-20260204-000001"`
- `TrackId = "PAY-20260204-000002"`

> Note: Some SDKs use `Trackid` (lowercase `i`) instead of `TrackId`. Use the exact property name available in your model.

---

## Request Fields (Minimum)
For most transactions, you will use:
- `Id`, `Password`, `ResourceKey`
- `Mode` (`SANDBOX` or `PRODUCTION`) — mainly for **Purchase**
- `TrackId` (**unique**)
- `Amt`, `CurrencyCode` (commonly required for **Inquiry / Reversal / Refund** as per gateway expectation)
- `ResponseURL`, `ErrorURL` (Purchase)

---

## UDF Fields (UDF1..UDF20) — Optional
UDF fields are **merchant-defined metadata**. Common examples include:
- Customer **Email**
- **Student Number**
- **Mobile Number**
- Internal reference IDs, session IDs, etc.

> Recommendation: keep the meaning of each UDF consistent (example: `Udf1=email`, `Udf2=studentNo`, `Udf3=mobile`).

### Example (set individually as needed)
```csharp
// UDF fields are optional. Set only the fields you need.
req.Udf1 = "student1@college.edu";   // Email
req.Udf2 = "STU-2026-000123";        // Student Number
req.Udf3 = "+9689XXXXXXX";           // Mobile Number

// Up to Udf20 is supported when your SDK model includes them:
// req.Udf4 = "...";
// ...
// req.Udf20 = "...";
```

---

## Token Payments (Using TokenNumber / TokenNo)

### When to use
After a successful **Purchase**, the gateway may return a token:
- `reply.TokenNumber` (or `reply.TokenNo` depending on SDK)

Store this token securely and use it for future token-based purchases.

### How to send token in the next Purchase
- Set `TokenFlag = "2"`
- Set `TokenNumber/TokenNo` to the token value received earlier

```csharp
// From a previous successful purchase response:
string storedTokenNumber = reply.TokenNumber; // store securely

// In the next purchase request:
req.TokenFlag   = "2";
req.TokenNumber = storedTokenNumber; // or req.TokenNo based on your SDK
```

---

## Purchase Transaction

### 1) Request Processing
```csharp
// Define key details
string tranportalId = "your_tranportal_id";
string password     = "your_password";
string resourceKey  = "your_resource_key";
string mode         = "SANDBOX"; // SANDBOX / PRODUCTION
string currency     = "512";
string language     = "EN";
string receiptURL   = "https://merchant.com/responseurl/";
string errorURL     = "https://merchant.com/errorurl/";
string trackid      = "PAY-20260204-000001"; // must be unique
string amount       = "10.00";

// Create a new request instance
Request req = new Request();

// Set required fields
req.Id          = tranportalId;
req.Password    = password;
req.ResourceKey = resourceKey;
req.Mode        = mode;

req.CurrencyCode = currency;
req.LangId       = language;
req.ResponseURL  = receiptURL;
req.ErrorURL     = errorURL;

req.Amt     = amount;
req.TrackId = trackid;

// Optional: UDF fields (Udf1..Udf20)
req.Udf1 = "User Defined value 1";
req.Udf2 = "User Defined value 2";
req.Udf3 = "User Defined value 3";
req.Udf4 = "User Defined value 4";
req.Udf5 = "User Defined value 5";

// Optional: Token transactions
// req.TokenNo   = tokenNo;   // or req.TokenNumber
// req.TokenFlag = tokenFlag; // "2" to use stored token

// Optional: Enable split payment
req.SplitPaymentIndicator = "1";

// Optional: Create and configure split payment payload
SplitPaymentPayload splitPayLoad = new SplitPaymentPayload();
splitPayLoad.AliasName   = "account001";
splitPayLoad.Notes       = "Salary";
splitPayLoad.Type        = "1";
splitPayLoad.Reference   = "87234234234932334";
splitPayLoad.SplitAmount = "10";

// Optional: Add split payment details
req.AddSplitPaymentPayload(splitPayLoad);

// Prepare the request transaction data
RequestTranData reqTranData = OabIpayRequestBuilder.PrepareRequestTranData(req);
```

### 2) Form Submission
```html
<form action="<%= reqTranData.WebAddress %>" method="post">
  <input type="hidden" name="tranportalId" value="<%= reqTranData.TranportalId %>" />
  <input type="hidden" name="responseURL"  value="<%= reqTranData.ResponseURL %>" />
  <input type="hidden" name="errorURL"     value="<%= reqTranData.ErrorURL %>" />
  <input type="hidden" name="trandata"     value="<%= reqTranData.TranData %>" />
  <button type="submit">Pay Now</button>
</form>
```

---

## Response Handling (Purchase / Refund / Reversal)

### Decrypt and Parse Reply
```csharp
// Define key details
string tranportalId = "your_tranportal_id";
string password     = "your_password";
string resourceKey  = "your_resource_key";

// Retrieve parameters from callback
string replyTranData = request["trandata"];
string trackId       = request["trackId"];

// Create a new ReplyTranData instance and set values
ReplyTranData tranData = new ReplyTranData();
tranData.Id          = tranportalId;
tranData.Password    = password;
tranData.ResourceKey = resourceKey;
tranData.TranData    = replyTranData;
tranData.TrackId     = trackId;

// Process the transaction response
Reply reply = OabIpayReplyBuilder.PrepareReply(tranData);
```

### Fields to Read From the Response
**Common fields (recommended)**
- `reply.Result`
- `reply.PaymentId`
- `reply.TranId`
- `reply.Ref`
- `reply.Auth`
- `reply.TrackId`
- `reply.Amt`

**UDF fields (if sent)**
- `reply.Udf1` ... `reply.Udf20`

**Token / card fields (if provided by gateway/SDK)**
- `reply.TokenNumber` (or `reply.TokenNo`)
- `reply.BrandType`
- `reply.MaskedCard`

---

## Result Codes (Success / Failure)
The primary status field is:
- `reply.Result`

### Common result values (typical)
- **Success:** `CAPTURED`, `SUCCESS`, `APPROVED`
- **Failure:** `NOT CAPTURED`, `NOT APPROVED`

### Transaction-specific guidance
- **Purchase**
  - `CAPTURED` = success
  - `NOT CAPTURED` = failure
- **Inquiry**
  - `SUCCESS` = success

> Always treat any unexpected/unknown `Result` value as **non-success** and handle it safely.

---

## Inquiry Transaction

### Inquiry by TrackId (if supported by your SDK)
```csharp
Request request = new Request();
request.TransId      = "original transaction track id"; // reference to original purchase track id (if required by your SDK)
request.Trackid      = "PAY-20260204-000002";            // must be unique for this inquiry request
request.Amt          = "10";
request.CurrencyCode = "512";

Reply reply = new OabIpayConnection().ProcessInquiryByTrackId(request);
var result = reply.Result;
```

### Inquiry by TranId (Transaction Id)
```csharp
Request request = new Request();
request.TransId      = "original transaction transaction id";
request.Trackid      = "PAY-20260204-000003"; // must be unique for this inquiry request
request.Amt          = "10";
request.CurrencyCode = "512";

Reply reply = new OabIpayConnection().ProcessInquiryByTranId(request);
var result = reply.Result;
```

---

## Reversal Transaction (by TranId)
```csharp
Request request = new Request();
request.TransId      = "original transaction transaction id";
request.Trackid      = "PAY-20260204-000004"; // must be unique for this reversal request
request.Amt          = "10";
request.CurrencyCode = "512";

Reply reply = new OabIpayConnection().ProcessReversalByTranId(request);
var result = reply.Result;
```

---

## Refund Transaction (by TranId)
```csharp
Request request = new Request();
request.TransId      = "original transaction transaction id";
request.Trackid      = "PAY-20260204-000005"; // must be unique for this refund request
request.Amt          = "10";
request.CurrencyCode = "512";

Reply reply = new OabIpayConnection().ProcessRefundByTranId(request);
var result = reply.Result;
```

---

## Security Guidelines
- Use HTTPS for all callbacks and API communication.
- Never expose `Password` / `ResourceKey` in frontend code.
- Store TokenNumber securely (do not expose in client-side code).
- Avoid logging sensitive values; prefer masked/tokenized values.

---

## Support
For any issues or questions, contact our support team at **pg-support@oman-arabbank.com**.

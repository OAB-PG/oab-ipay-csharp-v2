# OAB iPay .NET Merchant Integration Guide

## Overview

This document explains how to integrate a merchant .NET application with the OAB iPay plugin.

The plugin is provided as:

```text
OabIpay.Net.dll
```

The plugin supports keystore/resource based merchant credentials. The merchant application passes the key path, alias, and key-file password from server-side configuration into each request.

## Prerequisites

- .NET 8 runtime or later
- `OabIpay.Net.dll`
- OAB merchant key file folder
- Terminal alias provided by OAB
- Key-file password
- HTTPS response and error callback URLs

Do not expose key path, alias, key password, card data, wallet data, or decrypted payment data to browser-side code.

## Add DLL To Merchant Project

Create a `lib` folder in the merchant project and place:

```text
lib\OabIpay.Net.dll
```

Add the DLL reference to the merchant `.csproj`:

```xml
<ItemGroup>
  <Reference Include="OabIpay.Net">
    <HintPath>lib\OabIpay.Net.dll</HintPath>
  </Reference>
</ItemGroup>
```

Use these namespaces:

```csharp
using OabIpay.Net;
using OabIpay.Net.Models;
```

## Common Configuration

Keep these values in server-side configuration such as `appsettings.json`, environment-specific config, database configuration, or another protected configuration store.

```csharp
var keyPath = @"D:\oabpgkeys\";
var alias = "merchant-terminal-aliasname";
var keyPassword = "merchant-key-password";
```

Every request that needs encryption/decryption must include:

```csharp
KeyPath = keyPath,
Alias = alias,
OabIpayKeysPassword = keyPassword
```

## Hosted Payment Request

Use this flow when the customer must be redirected to the OAB hosted payment page.

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "10.000",
    Trackid = "87234234234",
    ResponseURL = "https://merchant.com/callback/response",
    ErrorURL = "https://merchant.com/callback/error",
    Udf1 = "User Defined value 1",
    Udf2 = "User Defined value 2",
    Udf3 = "User Defined value 3",
    Udf4 = "User Defined value 4",
    Udf5 = "User Defined value 5"
};

RequestTranData requestTranData = OabIpayRequestBuilder.PrepareRequestTranData(request);
```

Submit the returned values to `requestTranData.WebAddress` using HTTP POST:

```html
<form action="@requestTranData.WebAddress" method="post">
  <input type="hidden" name="tranportalId" value="@requestTranData.TranportalId" />
  <input type="hidden" name="responseURL" value="@requestTranData.ResponseURL" />
  <input type="hidden" name="errorURL" value="@requestTranData.ErrorURL" />
  <input type="hidden" name="trandata" value="@requestTranData.Trandata" />
  <button type="submit">Pay Now</button>
</form>
```

## Split Payment

Set `SplitPaymentIndicator` and add one or more split payload rows.

```csharp
request.SplitPaymentIndicator = "1";
request.AddSplitPaymentPayload(new SplitPaymentPayload
{
    AliasName = "account001",
    Reference = "87234234234932334",
    SplitAmount = "10.000",
    Notes = "Split payment",
    Description = "Merchant split",
    Type = "1"
});
```

## Token Registration

Use token registration when the merchant needs a reusable card token.

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "0",
    Trackid = "87234234234",
    ResponseURL = "https://merchant.com/callback/response",
    ErrorURL = "https://merchant.com/callback/error"
};

RequestTranData requestTranData = OabIpayRequestBuilder.BuildTokenRegistrationTranData(request);
```

Submit the returned `RequestTranData` fields to the hosted payment page using the same POST form structure.

## Merchant Hosted Card Transaction

Use this only when the merchant is approved to collect card details and is PCI-compliant.

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "10.000",
    Trackid = "87234234234",
    CardNo = "4761340000000019",
    CardName = "CARD HOLDER",
    ExpMonth = "12",
    ExpYear = "2030",
    Cvv2 = "123",
    BrowserAcceptHeader = "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    BrowserIP = "148.64.17.32",
    BrowserJavaEnabled = false,
    BrowserJavascriptEnabled = true,
    BrowserLanguage = "en-GB",
    BrowserColorDepth = "24",
    BrowserScreenHeight = "800",
    BrowserScreenWidth = "1280",
    BrowserTZ = "-240",
    BrowserUserAgent = "Mozilla/5.0 ..."
};

using var connection = new OabIpayConnection();
Reply reply = await connection.InitiateTransactionAsync(request);
```

If `reply.RedirectUrl` is returned, redirect the customer to that URL for authentication.

## Inquiry

Use inquiry to check payment status.

```csharp
using var connection = new OabIpayConnection();

var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Trackid = "87234234234"
};

Reply reply = await connection.ProcessInquiryByTrackIdAsync(request);
```

Available inquiry methods:

```csharp
Reply byTrackId = await connection.ProcessInquiryByTrackIdAsync(request);
Reply byPaymentId = await connection.ProcessInquiryByPaymentIdAsync(request);
Reply byTranId = await connection.ProcessInquiryByTranIdAsync(request);
Reply byRefNo = await connection.ProcessInquiryByRefNoAsync(request);
```

## Refund By Transaction ID

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Amt = "10.000",
    Transid = "202513211360225"
};

Reply reply = await new OabIpayConnection().ProcessRefundByTranIdAsync(request);
```

For split refund, add split payload rows:

```csharp
request.AddSplitPaymentPayload(new SplitPaymentPayload
{
    SplitTranId = "splitTransactionId",
    Reference = "refundReference",
    SplitAmount = "10.000",
    Notes = "Refund",
    Type = "1"
});
```

## Reversal

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Trackid = "87234234234",
    Transid = "202513211360225"
};

Reply byTranId = await new OabIpayConnection().ProcessReversalByTranIdAsync(request);
Reply byTrackId = await new OabIpayConnection().ProcessReversalByTrackIdAsync(request);
```

## Refund To Customer Account

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Transid = "202527112707175",
    ReceiverAccount = "3160798104502",
    ReceiverName = "Customer Name",
    Amt = "10.000",
    SwiftBankId = "OAB",
    Branch = "XXX",
    Purpose = "100",
    Country = "OM",
    Location = "RUW",
    Trackid = Guid.NewGuid().ToString("N")
};

Reply reply = await new OabIpayConnection().RefundToCustomerAccountAsync(request);
```

## Tokenized Purchase

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "10.000",
    Trackid = "87234234234",
    TokenNumber = "4395845272623928180011"
};

Reply reply = await new OabIpayConnection().TokenizedCardPurchaseAsync(request);
```

## Token Deregistration

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Trackid = "87234234234",
    TokenNumber = "4395845272623928180011"
};

Reply reply = await new OabIpayConnection().DeleteRegisteredCardTokenAsync(request);
```

## Apple Pay Direct Purchase

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "10.000",
    Trackid = "87234234234",
    Tranidentifer = "875adfbd9f017612156ccbc4c3f5cb7c3b1e56a58f92e350e7d6650b0eda5a23",
    ThreeDSPayload = new ThreeDSPayload
    {
        ApplicationPrimaryAccountNumber = "4636048640000017",
        ApplicationExpirationDate = "291231",
        CurrencyCode = "512",
        TransactionAmount = "1000000000000",
        DeviceManufacturerIdentifier = "040010030273",
        PaymentDataType = "3DSecure",
        PaymentData = new PaymentData
        {
            OnlinePaymentCryptogram = "AwAAAAAAH2FakPAAAAAAgVpgE4A=",
            EciIndicator = "5"
        }
    },
    MrchAuthData = new MrchAuthData
    {
        DisplayName = "Visa8478",
        Network = "Visa",
        Type = "debit"
    }
};

Reply reply = await new OabIpayConnection().ApplePayDirectPurchaseAsync(request);
```

## Samsung Pay Direct Purchase

```csharp
var request = new Request
{
    KeyPath = keyPath,
    Alias = alias,
    OabIpayKeysPassword = keyPassword,
    Currencycode = "512",
    Langid = "EN",
    Amt = "10.000",
    Trackid = "87234234234",
    ThreeDSPayload = new ThreeDSPayload
    {
        TokenPAN = "4636048570000037",
        TokenPanExpiration = "0929",
        Utc = "1727068862578",
        Amount = "100",
        Cryptogram = "AwAAAAAAH2FakPAAAAAAgVpgE4A=",
        CurrencyCodeSnakeCase = "OMR",
        EciIndicator = "05"
    },
    MrchAuthData = new MrchAuthData
    {
        Method = "3DS",
        RecurringPayment = "false",
        CardBrand = "visa",
        CardLast4Digits = "2301"
    }
};

Reply reply = await new OabIpayConnection().SamsungPayDirectPurchaseAsync(request);
```

## Callback Response Handling

For hosted payment and token registration, OAB posts or redirects encrypted `trandata` to the merchant `ResponseURL` or `ErrorURL`.

```csharp
[HttpPost("/callback/response")]
public IActionResult ResponseCallback([FromForm] string trandata)
{
    var replyTranData = new ReplyTranData
    {
        KeyPath = keyPath,
        Alias = alias,
        OabIpayKeysPassword = keyPassword,
        Trandata = trandata
    };

    Reply reply = OabIpayReplyBuilder.PrepareReply(replyTranData);

    if (reply.Result == "CAPTURED")
    {
        // Mark payment successful.
    }
    else
    {
        // Treat as failed or pending according to merchant rules.
    }

    return Ok();
}
```

## RequestTranData Fields

| Field | Description |
|---|---|
| `Trandata` | Encrypted request payload to submit to OAB |
| `WebAddress` | OAB payment gateway URL |
| `TranportalId` | Terminal ID from merchant credentials |
| `ResponseURL` | Merchant success callback URL |
| `ErrorURL` | Merchant error callback URL |
| `PaymentId` | Payment identifier, when returned by the gateway |

## Reply Fields

| Field | Description |
|---|---|
| `Result` | Gateway result status |
| `PaymentId` | OAB payment identifier |
| `TranId` | Gateway transaction identifier |
| `TrackId` | Merchant track ID |
| `Ref` | Reference or retrieval number |
| `Auth` | Authorization code, when available |
| `Amt` | Transaction amount |
| `Currency` | Transaction currency |
| `Date` | Gateway post date |
| `TranDate` | Transaction processing date |
| `TranRequestDate` | Gateway request timestamp |
| `TranResponseDate` | Gateway response timestamp |
| `CardName` | Cardholder name, when returned |
| `MaskedCard` | Masked card number |
| `BrandType` | Card brand |
| `TranType` | Transaction type |
| `TokenNo` | Returned card token number/customer token value |
| `TokenCustId` | Customer token identifier, when returned |
| `RedirectUrl` | Authentication redirect URL for card/3DS flows |
| `Error` | Error description or code |
| `ErrorText` | Error description text |
| `Udf1` to `Udf20` | Merchant user-defined fields returned by OAB |
| `SplitPaymentPayload` | Split payment response list, when returned |

## Result Codes

| Result Code | Description | Merchant Action |
|---|---|---|
| `CAPTURED` | Transaction captured successfully | Mark payment/refund as successful after validating amount and track ID |
| `NOT CAPTURED` | Transaction was not captured | Treat as failed; show failure message or retry option |
| `REGISTERED` | Card token registration successful | Store returned token/customer token securely |
| `DEREGISTERED` | Card token deletion successful | Remove/deactivate token in merchant system |
| `SUCCESS` | Inquiry or status request completed successfully | Read returned transaction fields and reconcile |
| `VOIDED` | Reversal completed successfully | Mark transaction as reversed |
| `INITILIZED` | Card transaction initialized and may require redirect | Redirect customer to `RedirectUrl` when present |

Any result code other than the expected success code for the transaction type must be treated as failed or pending according to merchant business rules.

## Expected Success Codes By Feature

| Feature | Expected Success Result |
|---|---|
| Hosted purchase | `CAPTURED` |
| Merchant hosted card purchase | `CAPTURED` or `INITILIZED` before 3DS redirect |
| Token registration | `REGISTERED` |
| Tokenized purchase | `CAPTURED` |
| Token deregistration | `DEREGISTERED` |
| Inquiry | `SUCCESS` |
| Refund | `CAPTURED` |
| Reversal | `VOIDED` |
| Apple Pay direct purchase | `CAPTURED` |
| Samsung Pay direct purchase | `CAPTURED` |
| Refund to customer account | `CAPTURED` |

## Error Handling

Check these fields when a request fails:

```csharp
if (!string.IsNullOrWhiteSpace(reply.Error) || !string.IsNullOrWhiteSpace(reply.ErrorText))
{
    var message = reply.ErrorText ?? reply.Error;
    // Log internally and show a safe message to the customer.
}
```

Recommended validation before fulfilling an order:

- Confirm `reply.Result` is the expected success code.
- Confirm `reply.TrackId` matches the merchant order.
- Confirm `reply.Amt` matches the merchant order amount.
- Confirm duplicate callbacks are ignored safely.
- Run inquiry for uncertain, timeout, or pending cases.

## Security Guidelines

- Use HTTPS for all merchant callback URLs.
- Keep credentials and key passwords in server-side configuration only.
- Never expose key files, key path, alias, key password, or decrypted credentials to the browser.
- Never log full card numbers, CVV, wallet cryptograms, or decrypted sensitive payloads.
- Store key files outside publicly served folders.
- Restrict filesystem permissions on the key folder.
- Validate transaction status server-side before fulfilling an order.

## Support

Contact OAB payment gateway support:

[pg-support@oman-arabbank.com](mailto:pg-support@oman-arabbank.com)

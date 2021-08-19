# Cloud API Reference

The Fusion Cloud API allows the Sale System to communicate with a POI terminal via a Websocket connected to the DataMesh Unify switch. 

## Reference code  

- A DotNet NuGet package implementing the DataMesh Fusion API is available [on Nuget](https://www.nuget.org/packages/DataMeshGroup.Fusion.FusionClient/)
- Source is on [GitHub](https://github.com/datameshgroup/sdk-dotnet)
- Demo applications implementing the sdk are also available on [GitHub](https://github.com/datameshgroup/sdk-dotnet-purchasedemo)

## Security requirements

Unify utilises secure websockets for communication between Sale System and POI Server.

- The Sale System and merchant environment must support outgoing TCP connections to `*.datameshgroup.io:443` and `*.datameshgroup.io:5000` for both the terminal and Sale System
- As a cloud service, Unify may run on several IP addresses. 
  - The Sale System must always use the DNS endpoints provided by DataMesh and never limit connectivity to a specific IP address
- The Sale System websocket connection must use TLS v1.2 or v1.3 with the SNI extension and one of the following ciphers:
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (TLS v1.2)
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (TLS v1.2)
  - TLS_AES_128_GCM_SHA256 (TLS v1.3)
  - TLS_CHACHA20_POLY1305_SHA256 (TLS v1.3)
  - TLS_AES_256_GCM_SHA384 (TLS v1.3)
- The Sale System must ensure the connection to Unify is trusted by validating the server-side certificate chain
  - Unify utilises a self-signed root CA provided by DataMesh which can be downloaded from [here](files/rootca.datameshgroup.io.zip)
  - Intermediate certificates in the chain will be provided by the TLS session
  - Contact [integrations@datameshgroup.com](mailto:integrations@datameshgroup.com) for alternative connection methods if the supporting the DataMesh root CA is unmanageable (for example, in a browser-based POS)
  - If certificate validation fails the Sale System must display an error and drop the connection.
- The Sale System must resolve the DNS address before each connection attempt, and never hard code IP addresses
- The Sale System should manage SSL certificate revocation lists and ensure OS security updates are applied
- The Sale System should store the `SaleID`, `POIID`, and `KEK` in a secure location. These values are used to identify the Sale System and authenticate the [SecurityTrailer](#securitytrailer)

<!--  - Unify utilises one of the root CA's provided by [Sectigo](https://sectigo.com/resource-library/sectigo-root-intermediate-certificate-files) and [Digicert](https://www.digicert.com/kb/digicert-root-certificates.htm). The Sale System must ensure all certificate authorities from these sources are loaded into the certificate store used to validate the server. 
  - A full list of ca files can be downloaded from [here](files/root-ca-list.zip). -->


<aside class="success">
Over time, Data Mesh may require the Sale System to update the TLS requirements provided above if a security risk is identified. In this situation Data Mesh will provide the new requirements and allow a reasonable amount of time to allow the Sale System to meet the new requirements. 
</aside>

## Endpoints

All endpoints are on port 443 and valid for secure websocket connections.


Production environment

`wss://prod1.datameshgroup.io:5000`

<!--
`wss://nexo.aus.datameshgroup.io/nexocloudpos`
-->

Test environment

`wss://www.cloudposintegration.io/nexouat1`

<!--
`wss://nexo.aus.datameshgroup.io/nexocloudpos`


`wss://nexo.nonprod.aus.datameshgroup.io/nexocloudpos`
-->


## Message format

> SaleToPOIRequest

```json
{
  "SaleToPOIRequest": {
    "MessageHeader":{...},
    "PaymentRequest":{...},
    "SecurityTrailer":{...}
  }
}
```

> SaleToPOIResponse

```json
{
  "SaleToPOIResponse": {
    "MessageHeader":{...},
    "PaymentResponse":{...},
    "SecurityTrailer":{...}
  }
}
```

All messages use JSON format with UTF-8 encoding. 

Supported primitive data elements are:

1. **String** text string of variable length
1. **Boolean** true or false
1. **Number** defined in this document as either `integer` or `decimal`. For all number fields the Sale System _MUST_ remove the digits equal to zero on left and the right of the value and any useless 
decimal point (eg. 00320.00 is expressed 320, and 56.10 is expressed 56.1). This simplifies parsing and MAC calculations.
1. **Null** optional types can be represented as null

<aside class="success">
Additional fields will be added to the message specification over time. To ensure forwards compatibility 
the Sale System must ignore when extra objects and fields are present in response messages. 
</aside>

The base of every message is the `SaleToPOIRequest` object for requests, and `SaleToPOIResponse` object for responses. 

The `SaleToPOIRequest` and `SaleToPOIResponse` contain three objects: 

1. A [MessageHeader](#messageheader) object.
1. A [Payload](#payload) object of variable types.
1. A [SecurityTrailer](#securitytrailer) object. 

### MessageHeader

> MessageHeader

```json
"MessageHeader":{
  "ProtocolVersion":"3.1-dmg",
  "MessageClass":"",
  "MessageCategory":"",
  "MessageType":"",
  "ServiceID":"",
  "SaleID":"",
  "POIID":""
}
```

A `MessageHeader` is included with each request and response. It defines the protocol, message type, sale and POI id.


Attribute                             |Requ.| Format | Description |
-----------------                     |----| ------ | ----------- |
[ProtocolVersion](#data-dictionary-protocolversion)   | ✔ | String | Version of the Sale to POI protocol specifications. Set to "3.1-dmg". Present when `MessageCategory` is "Login" otherwise absent.
[MessageClass](#data-dictionary-messageclass)         | ✔ | String | Informs the receiver of the class of message. Possible values are "Service", "Device", or "Event"
[MessageCategory](#data-dictionary-messagecategory)   | ✔ | String | Indicates the category of message. Possible values are "CardAcquisition", "Display", "Login", "Logout", "Payment" 
[MessageType](#data-dictionary-messagetype)           | ✔ | String | Type of message. Possible values are "Request", "Response", or "Notification"
[ServiceID](#data-dictionary-serviceid)               | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                     | ✔ | String | Uniquely identifies the Sale System. The `SaleID` is provided by DataMesh, and must match the SaleID configured in Unify.
[POIID](#data-dictionary-poiid)                       | ✔ | String | Uniquely identifies the POI Terminal. The `POIID` is provided by DataMesh, and must match the POIID configured in Unify. For Sale Systems that do not need a POI Terminal, the value must be "POI Server"

 
### Payload

An object which defines fields for the request/response. The object name depends on the [MessageCategory](#data-dictionary-messagecategory) defined in the `MessageHeader`

e.g. a login will include `LoginRequest`/`LoginResponse`, and a payment will include a `PaymentRequest`/`PaymentResponse`.

The [Cloud API Reference](#cloud-api-reference) outlines the expected payload for each supported request.  

### SecurityTrailer

> SecurityTrailer

```json
"SecurityTrailer":{
  "ContentType":"id-ctauthData",
  "AuthenticatedData":{
   "Version":"v0",
 "Recipient":{
   "KEK":{
  "Version":"v4",
  "KEKIdentifier":{
    "KeyIdentifier":"SpecV2TestMACKey",
    "KeyVersion":"20191122164326.594"
  },
  "KeyEncryptionAlgorithm":{
    "Algorithm":"des-ede3-cbc"
  },
  "EncryptedKey":"834EAB305DD18724B9ADF361FC698CE0"
   },
   "MACAlgorithm":{
  "Algorithm":"id-retail-cbc-mac-sha-256"
   },
   "EncapsulatedContent":{
  "ContentType":"iddata"
   },
   "MAC":"C5142F4DB828AA1C"
 }
  }
}
```


A `SecurityTrailer` object is included with each request and response. 

Unify authenticates requests from the Sale System by examining the `SecurityTrailer`, along with the `SaleID`, `POIID`, and `CertificationCode`

Session Keys are used to generate/verify a Message Authentication Code (MAC) to prove the authenticity of transactions. They are also used to protect Sensitive Card Data if sent from the Sale System. Session keys must change for every message.


**SecurityTrailer**

<div style="width:300px">Attribute</div>   |Requ.| Format | Description |
-----------------                          |----| ------ | ----------- |
[ContentType](#contenttype)                | ✔ | String | Set to "id-ctauthData"
*AuthenticatedData*                        | ✔ | Object |  
 [Version](#version)                       | ✔ | String | Set to "v0"
 *Recipient*                               | ✔ | Object | 
  *KEK*                                    | ✔ | Object | 
   [Version](#version)                     | ✔ | String | Set to "v4"
   *KEKIdentifier*                         | ✔ | Object |
    [KeyIdentifier](#data-dictionary-keyidentifier)        | ✔ | String | "SpecV2TestMACKey" for test environment, and "SpecV2ProdMACKey" for production
    [KeyVersion](#data-dictionary-keyversion)              | ✔ | String | An incrementing value. Either a counter or date formatted as YYYYMMDDHHmmss.mmm. See [KeyVersion](#data-dictionary-keyversion)
   *KeyEncryptionAlgorithm*                | ✔ | Object | 
    [Algorithm](#data-dictionary-algorithm)                | ✔ | String | Set to "des-ede3-cbc". 
   [EncryptedKey](#data-dictionary-encryptedkey)           | ✔ | String | A double length 3DES key. See [EncryptedKey](#data-dictionary-encryptedkey)
  *MACAlgorithm*                           | ✔ | Object | 
   [Algorithm](#data-dictionary-algorithm)                 | ✔ | String | Set to "id-retail-cbc-mac-sha-256"
   *EncapsulatedContent*                   | ✔ | Object | 
    [ContentType](#contenttype)            | ✔ | String | Set to "iddata"
  [MAC](#data-dictionary-mac)                              | ✔ | String | MAC of message content. See [MAC](#data-dictionary-mac)


<aside class="notice">
For brevity the <code>SecurityTrailer</code> has been excluded from examples.
</aside>

## Perform a purchase

To perform a purchase the Sale System will need to implement requests, and handle responses outlined in the [payment lifecycle](#payment-lifecycle).


- If a login hasn't already been sent for the session, send a login request as detailed in [login request](#login) 
  - Ensure "PrinterReceipt" is included in [SaleTerminalData.SaleCapabilities](#data-dictionary-salecapabilities) if payment receipts are to be redirected to the Sale System
- Await the a login response and
  - Ensure the [ServiceID](#data-dictionary-serviceid) in the result matches the request
  - Record the [POISerialNumber](#data-dictionary-poiserialnumber) to be sent in subsequent login requests
- Send a payment request, including all required fields, as detailed in [payment request](#payment) 
  - Set [PaymentData.PaymentType](#data-dictionary-paymenttype) to "Normal"
  - Set the purchase amount in [PaymentTransaction.AmountsReq.RequestedAmount](#requestedamount)
  - Set [SaleTransactionID](#data-dictionary-saletransactionid) to a unique value for the sale on this Sale System
  - Populate the [SaleItem](#data-dictionary-saleitem) array with the product basket for the transaction 
- If configured in [SaleTerminalData.SaleCapabilities](#data-dictionary-salecapabilities), handle any [display](#display), [print](#print), and [input](#input) events the POI System sends
  - The expected user interface handling is outlined in [user interface](#user-interface)
  - The expected payment receipt handling is outlined in [receipt printing](#receipt-printing)
- Await the payment response 
  - Ensure the [ServiceID](#data-dictionary-serviceid) in the result matches the request
  - Check [Response.Result](#data-dictionary-result) for the transaction result 
  - If [Response.Result](#data-dictionary-result) is "Success", record the following to enable future matched refunds:
    - [SaleID](#data-dictionary-saleid)
	- [POIID](#data-dictionary-poiid)
	- [POITransactionID](#data-dictionary-poitransactionid)
  - Check [PaymentResult.AmountsResp.AuthorizedAmount](#authorizedamount) (it may not equal the `RequestedAmount` in the payment request)
  - If the Sale System is handling tipping or surcharge, check the [PaymentResult.AmountsResp.TipAmount](#tipamount), and [PaymentResult.AmountsResp.SurchargeAmount](#surchargeamount)
  - Print the receipt contained in `PaymentReceipt`
- Implement error handling outlined in [error handling](#error-handling)


## Perform a refund

<!-- TODO need to add support here for matched refunds -->

To perform a refund the Sale System will need to implement requests, and handle responses outlined in the [payment lifecycle](#payment-lifecycle).

In most cases the Sale System should attempt to 
//

- If a login hasn't already been sent for the session, send a login request as detailed in [login request](#login) 
  - Ensure "PrinterReceipt" is included in [SaleTerminalData.SaleCapabilities](#data-dictionary-salecapabilities) if payment receipts are to be redirected to the Sale System
- Await the a login response and
  - Ensure the [ServiceID](#data-dictionary-serviceid) in the result matches the request
  - Record the [POISerialNumber](#data-dictionary-poiserialnumber) to be sent in subsequent login requests
- Send a payment request, including all required fields, as detailed in [payment request](#payment) 
  - Set [PaymentData.PaymentType](#data-dictionary-paymenttype) to "Refund"
  - Set the refund amount in [PaymentTransaction.AmountsReq.RequestedAmount](#requestedamount)
  - Set [SaleTransactionID](#data-dictionary-saletransactionid) to a unique value for the sale on this Sale System
  - If refunding a previous purchase, set the following fields in [PaymentTransaction.OriginalPOITransaction](#data-dictionary-originalpoitransaction)
    - Set [SaleID](#data-dictionary-saleid) to the `SaleId` of the original purchase payment request 
	- Set [POIID](#data-dictionary-poiid) to the `POIID` of the original purchase payment request 
	- Set [POITransactionID](#data-dictionary-poitransactionid) to the value returned in [POIData.POITransactionID](#data-dictionary-poitransactionid) of the original purchase payment response 
    - The product basket is not required for refunds
- If configured in [SaleTerminalData.SaleCapabilities](#data-dictionary-salecapabilities), handle any [display](#display), [print](#print), and [input](#input) events the POI System sends
  - The expected user interface handling is outlined in [user interface](#user-interface)
  - The expected payment receipt handling is outlined in [receipt printing](#receipt-printing)
- Await the payment response 
  - Ensure the [ServiceID](#data-dictionary-serviceid) in the result matches the request
  - Check [Response.Result](#data-dictionary-result) for the transaction result 
  - Check [PaymentResult.AmountsResp.AuthorizedAmount](#authorizedamount) (it may not equal the `RequestedAmount` in the payment request)
  - Print the receipt contained in `PaymentReceipt`
- Implement error handling outlined in [error handling](#error-handling)


## Methods

### Login 

The Sale System sends a NEXO Login request when it is ready to pair with a POI terminal and 
before any Reconciliation Request. The Sale System can pair with multiple POI terminals by 
sending another NEXO Login request.

#### Login request

> Login request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "ProtocolVersion":"3.1-dmg",
         "MessageClass":"Service",
         "MessageCategory":"Login",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "LoginRequest":{
         "DateTime":"xxx",
         "SaleSoftware":{
            "ProviderIdentification":"xxx",
            "ApplicationName":"xxx",
            "SoftwareVersion":"xxx",
            "CertificationCode":"xxx"
         },
         "SaleTerminalData":{
            "TerminalEnvironment":"xxx",
            "SaleCapabilities":[
               "xxx",
               "xxx",
               "xxx"
            ],
            "TotalsGroupID":"xxx"
         },
         "OperatorLanguage":"en",
         "OperatorID":"xxx",
         "ShiftNumber":"xxx",
         "POISerialNumber":"xxx"
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[ProtocolVersion](#data-dictionary-protocolversion)       | ✔ | String | "3.1-dmg"       
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Login"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Uniquely identifies the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**LoginRequest**

<div style="width:180px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
[DateTime](#data-dictionary-datetime)                        | ✔ | String | Current Sale System time, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) DateTime. e.g. "2019-09-02T09:13:51.0+01:00"   
**SaleSoftware**                            | ✔ | Object | Object containing Sale System identification
 [ProviderIdentification](#data-dictionary-provideridentification)| ✔ | String | The name of the company supplying the Sale System 
 [ApplicationName](#data-dictionary-applicationname)          | ✔ | String | The name of the Sale System application
 [SoftwareVersion](#data-dictionary-softwareversion)          | ✔ | String | The software version of the Sale System
 [CertificationCode](#data-dictionary-certificationcode)      | ✔ | String | Certification code for this Sale System provided by DataMesh
**SaleTerminalData**                          | ✔ | Object | Object containing Sale System configuration 
 [TerminalEnvironment](#data-dictionary-terminalenvironment)  | ✔ | String | "Attended", "SemiAttended", or "Unattended"
 [SaleCapabilities](#data-dictionary-salecapabilities)        | ✔ | Array | Advises the POI System of the Sale System capabilities. See [SaleCapabilities](#data-dictionary-salecapabilities) 
 [TotalsGroupId](#data-dictionary-totalsgroupid)              |  | String | Groups transactions in a login session
[OperatorLanguage](#operatorlanguage)         | ✔ | String | Operator language. Set to 'en'
[OperatorId](#data-dictionary-operatorid)                     |  | String | Groups transactions under this operator id
[ShiftNumber](#data-dictionary-shiftnumber)                   |  | String | Groups transactions under this shift number
[POISerialNumber](#data-dictionary-poiserialnumber)           |  | String | The POISerialNumber from the last login response, or absent if this is the first login 

#### Login response

> Login response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "ProtocolVersion":"3.1-dmg",
         "MessageClass":"Service",
         "MessageCategory":"Login",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "LoginResponse":{
         "Response":{
            "Result":"xxx",
            "ErrorCondition":"xxx",
            "AdditionalResponse":"xxx"
         },
         "POISystemData":{
            "DateTime":"xxx",
            "POISoftware":{
               "ProviderIdentification":"xxx",
               "ApplicationName":"xxx",
               "SoftwareVersion":"xxx"
            },
            "POITerminalData":{
               "TerminalEnvironment":"xxx",
               "POICapabilities":[
                  "xxx",
                  "xxx",
                  "xxx"
               ],
               "POIProfile":{
                  "GenericProfile":"Custom"
               },
               "POISerialNumber":"xxx"
            },
            "POIStatus":{
               "GlobalStatus":"xxx",
               "PEDOKFlag":"true or false",
               "CardReaderOKFlag":"true or false",
               "PrinterStatus":"xxx",
               "CommunicationOKFlag":"true or false",
               "FraudPreventionFlag":"true or false"
            },
            "TokenRequestStatus":"true or false"
         }
      },
      "SecurityTrailer":{...}
   }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[ProtocolVersion](#data-dictionary-protocolversion)       | ✔ | String | "3.1-dmg"       
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Login"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from the request
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from the request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from the request

**LoginResponse**

<div style="width:180px">Attribute</div>     |Requ.| Format | Description |
-----------------                            | -------- | ------ | ----------- |
**Response**                                 | ✔ | Object | Object indicating the result of the login
 [Result](#data-dictionary-result)                           | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)           |    | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)   |    | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
**POISystemData**                            |    | Object | Only present when `Result` is "Success"
 [DateTime](#data-dictionary-datetime)                       | ✔ | String | Time on the POI System, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) DateTime. e.g. "2019-09-02T09:13:51.0+01:00"   
 [TokenRequestStatus](#tokenrequeststatus)   | ✔ | Boolean| True if POI tokenisation of PANs is available and usable
 **POITerminalData**                         | ✔ | Object | Object representing the POI Terminal 
  [TerminalEnvironment](#data-dictionary-terminalenvironment)| ✔ | String | Mirrored from the request
  [POICapabilities](#poicapabilities)        | ✔ | Array  | An array of strings which reflect the hardware capabilities of the POI Terminal. "MagStripe", "ICC", and "EMVContactless" 
  [GenericProfile](#genericprofile)          | ✔ | String | Set to "Custom"
  [POISerialNumber](#data-dictionary-poiserialnumber)        | ✔ | String | If POIID is "POI Server", then a virtual POI Terminal Serial Number. Otherwise the serial number of the POI Terminal
 **POIStatus**                               | ✔ | String | Object representing the current status of the POI Terminal
  [GlobalStatus](#globalstatus)              | ✔ | String | The current status of the POI Terminal. "OK" when the terminal is available. "Maintenance" if unavailable due to maintenance processing. "Unreachable" if unreachable or not responding
  [SecurityOKFlag](#securityokflag)          | ✔ | Boolean| True if the security module is present 
  [PEDOKFlag](#pedokflag)                    | ✔ | Boolean| True if PED is available and usable for PIN entry
  [CardReaderOKFlag](#cardreaderokflag)      | ✔ | Boolean| True if card reader is available and usable
  [PrinterStatus](#printerstatus)            | ✔ | String | Indicates terminal printer status. Possible values are "OK", "PaperLow", "NoPaper", "PaperJam", "OutOfOrder" 
  [CommunicationOKFlag](#communicationokflag)| ✔ | Boolean| True if terminal's communication is available and usable
  [FraudPreventionFlag](#fraudpreventionflag)| ✔ | Boolean| True if the POI detects possible fraud



### Logout 

Logging out is optional.

If sent, it tells the POI system that it won’t send new transactions to the POI Terminal 
and unpairs the Sale Terminal from the POI Terminal. Any further transactions to that POI Terminal 
will be rejected by the POI System until the next Login.

The Sale System may send multiple Login requests without a Logout request.

#### Logout request

> Logout request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"Logout",
         "MessageType":"Request",
         "ServiceID":"xxxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "LogoutRequest":{
         "MaintenanceAllowed":"true or false"
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Logout"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Uniquely identifies the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**LogoutRequest**

<div style="width:180px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
[MaintenanceAllowed](#data-dictionary-maintenanceallowed)    |  | Boolean| Indicates if the POI Terminal can enter maintenance mode. Default to true if not present.    

#### Logout response

> Logout response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"Logout",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "LogoutResponse":{
         "Response":"xxx",
         "ErrorCondition":"xxx",
         "AdditionalResponse":"xxx xxxx xxxx xxxx xxxx"
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Logout"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Uniquely identifies the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**LogoutResponse**

<div style="width:180px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
**Response**                                 | ✔ | Object | Object which represents the result of the response
 [Result](#data-dictionary-result)                           | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)           |  | String | Indicates the reason an error occurred. Only present when result is "Failure". Possible values are "MessageFormat", "Busy", "DeviceOut", "UnavailableService" and others. Note the Sale System should handle error conditions outside the ones documented in this specification.
 [AdditionalResponse](#data-dictionary-additionalresponse)   |  | String | Provides additional error information. Only present when result is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information of possible values. 


### Payment 

The payment message is used to perform purchase, purchase + cash out, cash out only, and refund requests. 


#### Payment request

> Payment request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"Payment",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "PaymentRequest":{
         "SaleData":{
            "OperatorID":"xxx",
            "OperatorLanguage":"en",
            "ShiftNumber":"xxx",
            "SaleTransactionID":{
               "TransactionID":"xxx",
               "TimeStamp":"xxx"
            },
            "SaleReferenceID":"xxx",
            "SaleTerminalData":{
               "TerminalEnvironment":"xxx",
               "SaleCapabilities":[
                  "xxx",
                  "xxx",
                  "xxx"
               ],
               "TotalsGroupID":"xxx"
            },
            "TokenRequestedType":"Customer | Transaction"
         },
         "PaymentTransaction":{
            "AmountsReq":{
               "Currency":"AUD",
               "RequestedAmount":"x.xx",
               "CashBackAmount":"x.xx",
               "TipAmount":"x.xx",
               "PaidAmount":"x.xx",
               "MaximumCashBackAmount":"x.xx",
               "MinimumSplitAmount":"x.xx"
            },
            "OriginalPOITransaction":{
               "SaleID":"xxx",
               "POIID":"xxx",
               "POITransactionID":{
                  "TransactionID":"xxx",
                  "TimeStamp":"xxx"
               },
               "ReuseCardDataFlag":true,
               "ApprovalCode":"xxx",
               "LastTransactionFlag":true
            },
            "TransactionConditions":{
               "AllowedPaymentBrands":[
                  "xxx",
                  "xxx",
                  "xxx"
               ],
               "AcquirerID":[
                  "xxx",
                  "xxx",
                  "xxx"
               ],
               "DebitPreferredFlag":true,
               "ForceOnlineFlag":true,
               "MerchantCategoryCode":"xxx"
            },
            "SaleItem":[
               {
                  "ItemID":"xxx",
                  "ProductCode":"xxx",
                  "EanUpc":"xxx",
                  "UnitOfMeasure":"xxx",
                  "Quantity":"xx.x",
                  "UnitPrice":"xx.x",
                  "ItemAmount":"xx.x",
                  "TaxCode":"xxx",
                  "SaleChannel":"xxx",
                  "ProductLabel":"xxx",
                  "AdditionalProductInfo":"xxx",
				  "CostBase":"xxx",
				  "Discount":"xxx",
				  "Category":"xxx",
				  "SubCategory":"xxx",
				  "Brand":"xxx",
				  "QuantityInStock":"xxx",
				  "Tags":["xxx","xxx","xxx"]
               }
            ]
         },
         "PaymentData":{
            "PaymentType":"xxx",
            "PaymentInstrumentData":{
               "PaymentInstrumentType":"xxx",
               "CardData":{
                  "EntryMode":"xxx",
                  "ProtectedCardData":{
                     "ContentType":"id-envelopedData",
                     "EnvelopedData":{
                        "Version":"v0",
                        "Recipient":{
                           "KEK":{
                              "Version":"v4",
                              "KEKIdentifier":{
                                 "KeyIdentifier":"xxxDATKey",
                                 "KeyVersion":"xxx"
                              },
                              "KeyEncryptionAlgorithm":{
                                 "Algorithm":"des-ede3-cbc"
                              },
                              "EncryptedKey":"xxx"
                           }
                        },
                        "EncryptedContent":{
                           "ContentType":"id-data",
                           "ContentEncryptionAlgorithm":{
                              "Algorithm":"des-ede3-cbc",
                              "Parameter":{
                                 "InitialisationVector":"xxx"
                              }
                           },
                           "EncryptedData":"xxx"
                        }
                     }
                  },
                  "SensitiveCardData":{
                     "PAN":"xxx",
                     "ExpiryDate":"xxx",
                     "CCV":"xxx"
                  },
                  "PaymentToken":{
                     "TokenRequestedType":"xxx",
                     "TokenValue":"xxx"
                  }
               }
            }
         }
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Payment"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Uniquely identifies the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**PaymentRequest**

<div style="width:200px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
**SaleData**                                 | ✔ | Object | Sale System information attached to this payment
 [OperatorID](#data-dictionary-operatorid)                   |   | String | Only required if different from Login Request
 [OperatorLanguage](#operatorlanguage)       | ✔ | String | Set to "en"
 [ShiftNumber](#data-dictionary-shiftnumber)                 |   | String | Only required if different from Login Request
 [SaleReferenceID](#data-dictionary-salereferenceid)         |  | String | Mandatory for pre-authorisation and completion, otherwise optional. See [SaleReferenceID](#data-dictionary-salereferenceid)
 [TokenRequestedType](#data-dictionary-tokenrequestedtype)   |  | String | If present, indicates which type of token should be created for this payment. See [TokenRequestedType](#data-dictionary-tokenrequestedtype)
 **SaleTransactionID**                       | ✔ | Object |
  [TransactionID](#data-dictionary-transactionid)            | ✔ | String | Unique reference for this sale ticket. Not necessarily unique per payment request; for example a sale with split payments will have a number of payments with the same [TransactionID](#data-dictionary-transactionid)
  [TimeStamp](#data-dictionary-timestamp)                    | ✔ | String | Time of initiating the payment request on the POI System, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) DateTime. e.g. "2019-09-02T09:13:51.0+01:00"   
 **SaleTerminalData**                        |  | Object | Define Sale System configuration. Only include if elements within have different values to those in Login Request
  [TerminalEnvironment](#data-dictionary-terminalenvironment)|  | String | "Attended", "SemiAttended", or "Unattended"
  [SaleCapabilities](#data-dictionary-salecapabilities)      |  | Array  | Advises the POI System of the Sale System capabilities. See [SaleCapabilities](#data-dictionary-salecapabilities) 
  [TotalsGroupId](#data-dictionary-totalsgroupid)            |  | String | Groups transactions in a login session
**PaymentTransaction**                       | ✔ | Object | 
 **AmountsReq**                               | ✔ | Object | Object which contains the various components which make up the payment amount
  [Currency](#currency)                      | ✔ | String | Three character currency code. Set to "AUD"
  [RequestedAmount](#requestedamount)        | ✔ | Decimal| The requested amount for the transaction sale items, including cash back and tip requested
  [CashBackAmount](#cashbackamount)          |  | Decimal | The Cash back amount. Only if cash back is included in the transaction by the Sale System
  [TipAmount](#tipamount)                    |  | Decimal | The Tip amount. Only if tip is included in the transaction
  [PaidAmount](#paidamount)                  |  | Decimal | Sum of the amount of sale items – `RequestedAmount`. Present only if an amount has already been paid in the case of a split payment.
  [MaximumCashBackAmount](#maximumcashbackamount)|  | Decimal | Available if `CashBackAmount` is not present. If present, the POI Terminal prompts for the cash back amount up to a maximum of `MaximumCashBackAmount`
  [MinimumSplitAmount](#minimumsplitamount)  |   | Decimal | Present only if the POI Terminal can process an amount < `RequestedAmount` as a split amount. Limits the minimum split amount allowed.
 **[OriginalPOITransaction](#data-dictionary-originalpoitransaction)** |  | Object | Identifies a previous POI transaction. Mandatory for Refund and Completion. See [OriginalPOITransaction](#data-dictionary-originalpoitransaction)
  [SaleID](#data-dictionary-saleid)                          | ✔ | String | `SaleID` which performed the original transaction
  [POIID](#data-dictionary-poiid)                            | ✔ | String | `POIID` which performed the original transaction
  **POITransactionID**                       | ✔ | Object | 
   [TransactionID](#data-dictionary-transactionid)           | ✔ | String | `TransactionID` from the original transaction
   [TimeStamp](#data-dictionary-timestamp)                   | ✔ | String | `TimeStamp` from the original transaction
  [ReuseCardDataFlag](#reusecarddataflag)    |  | Boolean| If 'true' the POI Terminal will retrieve the card data from file based on the `PaymentToken` included in the request. Otherwise the POI Terminal will read the same card again.
  [ApprovalCode](#approvalcode)              |  | String | Present if a referral code is obtained from an Acquirer
  [LastTransactionFlag](#lasttransactionflag)| ✔ | Boolean| Set to true to process the Last Transaction with a referral code
 **TransactionConditions**                   |  | Object | Optional transaction configuration. Present only if any of the JSON elements within are present.
  [AllowedPaymentBrands](#data-dictionary-allowedpaymentbrands)|  | Array  | Restricts the request to specified card brands. See [AllowedPaymentBrands](#data-dictionary-allowedpaymentbrands)
  [AcquirerID](#data-dictionary-paymenttransaction.transactionconditions.acquirerid) |  | Array  | Used to restrict the payment to specified acquirers. See [AcquirerID](#data-dictionary-paymenttransaction.transactionconditions.acquirerid)
  [DebitPreferredFlag](#debitpreferredflag)  |  | Boolean| If present, debit processing is preferred to credit processing.
  [ForceOnlineFlag](#data-dictionary-forceonlineflag)        |  | Boolean| If 'true' the transaction will only be processed in online mode, and will fail if there is no response from the Acquirer.
  [MerchantCategoryCode](#data-dictionary-merchantcategorycode)|  | String | If present, overrides the MCC used for processing the transaction if allowed. Refer to ISO 18245 for available codes.
 **[SaleItem](#data-dictionary-saleitem)**                   | ✔ | Array  | Array of [SaleItem](#data-dictionary-saleitem) objects which represent the product basket attached to this transaction. See [SaleItem](#data-dictionary-saleitem) for examples.
  [ItemID](#data-dictionary-itemid)                          | ✔ | Integer | A unique identifier for the sale item within the context of this payment. e.g. a 0..n integer which increments by one for each sale item.
  [ProductCode](#data-dictionary-productcode)                | ✔ | String | A unique identifier for the product within the merchant. For example if two customers purchase the same product at two different stores owned by the merchant, both purchases should contain the same `ProductCode`.
  [EanUpc](#data-dictionary-eanupc)                          |  | String | A standard unique identifier for the product. Either the UPC, EAN, or ISBN.
  [UnitOfMeasure](#data-dictionary-unitofmeasure)            | ✔ | String | Unit of measure of the `Quantity`. See [UnitOfMeasure](#data-dictionary-unitofmeasure)
  [Quantity](#data-dictionary-quantity)                      | ✔ | Decimal| Item unit quantity
  [UnitPrice](#data-dictionary-unitprice)                    | ✔ | Decimal| Price per item unit.
  [ItemAmount](#data-dictionary-itemamount)                  | ✔ | Decimal| Total amount of the item
  [TaxCode](#data-dictionary-taxcode)                        |  | String | Type of tax associated with the item. Default = "GST"
  [SaleChannel](#data-dictionary-salechannel)                |  | String | Commercial or distribution channel of the item. Default = "Unknown"
  [ProductLabel](#data-dictionary-productlabel)              | ✔ | String | Product name of the item. See [ProductLabel](#data-dictionary-productlabel)
  [AdditionalProductInfo](#data-dictionary-additionalproductinfo)|  | String | Additional information, or more detailed description of the product item
  [CostBase](#costbase)                      |  | Decimal| Cost of the product to the merchant 
  [Discount](#discount)                      |  | Decimal| If applied, the amount this sale item was discounted by
  [Category](#category)                      |  | String | Product item category 
  [SubCategory](#subcategory)                |  | String | Product item sub category 
  [Brand](#brand)                            |  | String | Brand name - typically visible on the product packaging or label
  [QuantityInStock](#data-dictionary-quantityinstock)        |  | Decimal| Remaining number of this item in stock
  [Tags](#sale-item-tags)                    |  | Array  | Array of string with descriptive tags for the product
 **PaymentData**                             | ✔ | Object | Object representing the payment method. Present only if any of the JSON elements within are present.
  [PaymentType](#data-dictionary-paymenttype)                | ✔ | String | Defaults to "Normal". Indicates the type of payment to process. "Normal", "Refund", or "CashAdvance". See [PaymentType](#data-dictionary-paymenttype)
  **[PaymentInstrumentData](#data-dictionary-paymentinstrumentdata)** |  | Object | Object with represents card details for token or manually enter card details. See  for object structure


#### Payment response

> Payment response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"Payment",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "PaymentResponse":{
         "Response":{
            "Result":"Success| Partial | Failure",
            "ErrorCondition":"xxx",
            "AdditionalResponse":"xxx"
         },
         "SaleData":{
            "SaleTransactionID":{
               "TransactionID":"xxx",
               "TimeStamp":"xxx"
            },
            "SaleReferenceID":"xxxx"
         },
         "POIData":{
            "POITransactionID":{
               "TransactionID":"xxx",
               "TimeStamp":"xxx"
            },
            "POIReconciliationID":"xxx"
         },
         "PaymentResult":{
            "PaymentType":"xxx",
            "PaymentInstrumentData":{
               "PaymentInstrumentType":"xxx",
               "CardData":{
                  "EntryMode":"xxx",
                  "PaymentBrand":"xxx",
				  "Account":"xxx",
                  "MaskedPAN":"xxxxxx…xxxx",
                  "PaymentToken":{
                     "TokenRequestedType":"xxx",
                     "TokenValue":"xxx",
                     "ExpiryDateTime":"xxx"
                  }
               }
            },
            "AmountsResp":{
               "Currency":"AUD",
               "AuthorizedAmount":"x.xx",
               "TotalFeesAmount":"x.xx",
               "CashBackAmount":"x.xx",
               "TipAmount":"x.xx",
			   "SurchargeAmount":"x.xx"
            },
            "OnlineFlag":true,
            "PaymentAcquirerData":{
               "AcquirerID":"xxx",
               "MerchantID":"xxx",
               "AcquirerPOIID":"xxx",
               "AcquirerTransactionID":{
                  "TransactionID":"xxx",
                  "TimeStamp":"xxx"
               },
               "ApprovalCode":"xxx",
               "ResponseCode":"xxx",
               "HostReconciliationID":"xxx"
            }
         },
         "AllowedProductCodes":[
            "1",
            "2",
            "3"
         ],
         "PaymentReceipt":[
            {
               "DocumentQualifier":"xxx",
               "RequiredSignatureFlag":true,
               "OutputContent":{
                  "OutputFormat":"XHTML",
                  "OutputXHTML":"xxx"
               }
            }
         ]
      },
      "SecurityTrailer":{...}
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)                | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)          | ✔ | String | "Payment"
[MessageType](#data-dictionary-messagetype)                  | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid)                      | ✔ | String | Mirrored from the request
[SaleID](#data-dictionary-saleid)                            | ✔ | String | Mirrored from the request
[POIID](#data-dictionary-poiid)                              | ✔ | String | Mirrored from the request

**PaymentResponse**

<div style="width:200px">Attribute</div>     |Requ.| Format | Description |
-----------------                            |----| ------ | ----------- |
**Response**                                 | ✔ | Object | Object indicating the result of the payment
 [Result](#data-dictionary-result)                           | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)           |  | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)   |  | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
**SaleData**                                 | ✔ | Object | 
 **SaleTransactionID**                       | ✔ | Object | 
  [TransactionID](#data-dictionary-transactionid)            | ✔ | String | Mirrored from the request
  [TimeStamp](#data-dictionary-timestamp)                    | ✔ | String | Mirrored from the request
 [SaleReferenceID](#data-dictionary-salereferenceid)         |  | String | Mirrored from the request
**POIData**                                  | ✔ | Object | 
 **POITransactionID**                        | ✔ | Object | 
  [TransactionID](#data-dictionary-transactionid)            | ✔ | String | A unique transaction id from the POI system
  [TimeStamp](#data-dictionary-timestamp)                    | ✔ | String | Time on the POI system, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)
 [POIReconciliationID](#data-dictionary-poireconciliationid) |  | String | Present if `Result` is "Success" or "Partial". See [POIReconciliationID](#data-dictionary-poireconciliationid)
**PaymentResult**                            |  | Object | Object related to a processed payment
 [PaymentType](#data-dictionary-paymenttype)                 |  | String | Mirrored from the request
 **PaymentInstrumentData**                   |  | Object 
  [PaymentInstrumentType](#data-dictionary-paymentinstrumenttype) |  | String | "Card" or "Mobile"
  **CardData**                               |  | Object
   [EntryMode](#data-dictionary-entrymode)                   | ✔ | String | Indicates how the card was presented. See [EntryMode](#data-dictionary-entrymode)
   [PaymentBrand](#data-dictionary-paymentbrand)             | ✔ | String | Indicates the card type used. See [PaymentBrand](#data-dictionary-paymentbrand)
   [MaskedPAN](#data-dictionary-maskedpan)                   | ✔ | String | PAN masked with dots, first 6 and last 4 digits visible
   [Account](#data-dictionary-account)                       |  | String | Present if `EntryMode` is "MagStripe", "ICC", or "Tapped". Indicates the card account used. See [Account](#data-dictionary-account)
   **PaymentToken**                          |  | Object | Object representing a token. Only present if token was requested
    [TokenRequestedType](#data-dictionary-tokenrequestedtype)| ✔ | String | Mirrored from the request
    [TokenValue](#tokenvalue)                | ✔ | String | The value of the token
    [ExpiryDateTime](#expirydatetime)        | ✔ | String | Expiry of the token, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)
 **AmountsResp**                             |  | Object | Present if `Result` is "Success" or "Partial"
  [Currency](#currency)                      |  | String | "AUD"
  [AuthorizedAmount](#authorizedamount)      | ✔ | Decimal| Authorised amount which could be more, or less than the requested amount
  [TotalFeesAmount](#totalfeesamount)        |  | Decimal| Total of financial fees associated with the payment transaction if known at time of transaction
  [CashBackAmount](#cashbackamount)          |  | Decimal| Cash back paid amount
  [TipAmount](#tipamount)                    |  | Decimal| The amount of any tip added to the transaction
  [SurchargeAmount](#surchargeamount)        |  | Decimal| The amount of any surcharge added to the transaction
 [OnlineFlag](#onlineflag)                   | ✔ | Boolean| True if the transaction was processed online, false otherwise
 **PaymentAcquirerData**                     |  | Object | Data related to the response from the payment acquirer
  [AcquirerID](#data-dictionary-paymentacquirerdata.acquirerid) | ✔ | String | The ID of the acquirer which processed the transaction
  [MerchantID](#merchantid)                  | ✔ | String | The acquirer merchant ID (MID)
  [AcquirerPOIID](#acquirerPOIID)            | ✔ | String | The acquirer terminal ID (TID)
  **AcquirerTransactionID**                  | ✔ | Object | 
   [TransactionID](#data-dictionary-transactionid)           | ✔ | String | The acquirer transaction ID
   [TimeStamp](#data-dictionary-timestamp)                   | ✔ | String | Timestamp from the acquirer, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)
  [ApprovalCode](#approvalcode)              | ✔ | String | The Acquirer Approval Code. Also referred to as the Authentication Code
  [ResponseCode](#responsecode)              | ✔ | String | The Acquirer Response Code. Also referred as the PINPad response code
  [STAN](#stan)                              |  | String | The Acquirer STAN if available
  [RRN](#rrn)                                |  | String | The Acquirer RRN if available
  [HostReconciliationID](#hostreconciliationid)|✔| String | Identifier of a reconciliation period with the acquirer. This normally has a date and time component in it
 [AllowedProductCodes](#allowedproductcodes)  |  | Array | Present if `ErrorCondition` is "PaymentRestriction". Consists of a list of product codes corresponding to products that are purchasable with the given card. Items that exist in the basket but do not belong to this list corresponds to restricted items
 **PaymentReceipt**                           |  | Array | Array of payment receipt objects which represent receipts to be printed
  [DocumentQualifier](#documentqualifier)     | ✔ | String | "CashierReceipt" for a merchant receipt, otherwise "CustomerReceipt"
  [RequiredSignatureFlag](#requiredsignatureflag) | ✔|Boolean| If true, the card holder signature is required on the merchant CashierReceipt.
  **OutputContent**                           |  | Array | Array of payment receipt objects which represent receipts to be printed
   [OutputFormat](#data-dictionary-outputformat)              | ✔ | String | "XHTML"  
   [OutputXHTML](#data-dictionary-outputxhtml)                | ✔ | String | The payment receipt in XHTML format but coded in BASE64 


### Display 

During a payment, the POI System will send status and error display requests to the Sale System, which enables the Sale System to inform the cashier of the current transaction status.

Follow the [user interface](#user-interface) guide for details on how to implement the required UI to handle display and input requests.

#### Display request

> Display request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Display",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "DisplayRequest":{
         "DisplayOutput":[
            {
               "ResponseRequiredFlag":false,
               "Device":"CashierDisplay",
               "InfoQualify":"xxx",
               "OutputContent":{
                  "OutputFormat":"Text",
                  "OutputText":{
                     "Text":"xxx"
                  }
               }
            }
         ]
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Display"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from payment
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment

**DisplayRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format | Description |
-----------------                             |----| ------ | ----------- |
**DisplayOutput**                             | ✔ | Array | Array [1..6] of objects which represents the display 
 [ResponseRequiredFlag](#data-dictionary-responserequiredflag)|  | Boolean | Indicates if the POI System requires a `DisplayResponse` to be sent for this `DisplayRequest`
 [Device](#device)                            | ✔ | String | "CashierDisplay"
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | "Status" or "Error". See [InfoQualify](#data-dictionary-infoqualify)
 [OutputFormat](#data-dictionary-outputformat)                | ✔ | String | "Text"
 [Text](#text)                                | ✔ | String | Single line of text to display

#### Display response

> Display response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Display",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "DisplayResponse":{
         "OutputResult":[
            {
               "Device":"xxx",
               "InfoQualify":"xxx",
               "Response":{
                  "Result":"xxx",
                  "ErrorCondition":"xxx",
                  "AdditionalResponse":"xxx"
               }
            }
         ]
      },
      "SecurityTrailer":{...}
   }
}
```

The Sale System is expected to send a `DisplayResponse` if one or more displays in `DisplayOutput` have [ResponseRequiredFlag](#data-dictionary-responserequiredflag) set to true.


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Display"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from display request
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from display request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from display request

**DisplayResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format | Description |
-----------------                             |----| ------ | ----------- |
*OutputResult*                                | ✔ | Array | Array [1..6]. One object per Device/InfoQualify pair of values where corresponding `ResponseRequiredFlag` in the `DisplayRequest` is set to true.
 [Device](#device)                            | ✔ | String | Mirrored from display request
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | Mirrored from display request
 [Result](#data-dictionary-result)                            | ✔ | String | "Success", "Partial", or "Failure". See [Result](#data-dictionary-result).
 [ErrorCondition](#data-dictionary-errorcondition)            |  | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)    |  | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 




### Input

During a payment, the POI System will input requests to the Sale System if cashier interaction is required (e.g. signature approved yes/no)

Follow the [user interface](#user-interface) guide for details on how to implement the required UI to handle display and input requests.
 
#### Input request

> Input request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Input",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "InputRequest":{
         "DisplayOutput":{
            "Device":"CashierDisplay",
            "InfoQualify":"POIReplication",
            "OutputContent":{
               "OutputFormat":"Text",
               "OutputText":{
                  "Text":"xxx"
               }
            },
            "MenuEntry":[
               {
                  "OutputFormat":"Text",
                  "OutputText":{
                     "Text":"xxx"
                  }
               }
            ]
         },
         "InputData":{
            "Device":"CashierInput",
            "InfoQualify":"xxx",
            "InputCommand":"xxx",
            "MaxInputTime":"xxx",
            "MinLength":"xxx",
            "MaxLength":"xxx",
            "MaskCharactersFlag":"true or false"
         }
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Input"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from payment request
[DeviceID](#deviceid)                     | ✔ | String | Unique message identifier
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment request

**InputRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
**DisplayOutput**                             |  | Object | Information to display and the way to process the display.
 [Device](#device)                            | ✔ | String | "CashierDisplay"
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | "POIReplication". See [InfoQualify](#data-dictionary-infoqualify)
 **OutputContent**                            | ✔ | Object | 
  [OutputFormat](#data-dictionary-outputformat)               | ✔ | String | "Text"
  **OutputText**                              | ✔ | Object | Wrapper for text content
   [Text](#text)                              | ✔ | String | Single line of text. e.g. "Signature Ok?", "Merchant Password", "Select Account Type"
 **MenuEntry**                                |  | Array  | Conditional. Array of items to be presented as a menu. Only present if [InputCommand](#data-dictionary-inputcommand) = "GetMenuEntry"
  [OutputFormat](#data-dictionary-outputformat)               | ✔ | String | "Text"
  [Text](#text)                               | ✔ | String | One of the selection String items for the cashier to select from. For example: "Savings", "Cheque" and "Credit" for an account type selection.
**InputData**                                 | ✔ | Object | Information related to an `Input` request
 [Device](#device)                            | ✔ | String | "CashierInput"
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | "Input" or "CustomerAssistance". See [InfoQualify](#data-dictionary-infoqualify)
 [InputCommand](#data-dictionary-inputcommand)                | ✔ | String | "GetConfirmation", "Password", "TextString", "DigitString", "DecimalString", or "GetMenuEntry". See [InputCommand](#data-dictionary-inputcommand)
 [MaxInputTime](#maxinputtime)                |  | Number | The maximum number of seconds allowed for providing input.  Note the Sale Terminal needs to abort the Input process if it receives a DisplayRequest or InputRequest whilst waiting on input from the Cashier.
 [MinLength](#minlength)                      |  | Number | The minimum number of characters allowed for entry. Present if [InputCommand](#data-dictionary-inputcommand) = "Password", "TextString", "DigitString", or "DecimalString"
 [MaxLength](#maxlength)                      |  | Number | The maximum number of characters allowed for entry. Present if [InputCommand](#data-dictionary-inputcommand) = "Password", "TextString", "DigitString", or "DecimalString"
 [MaskCharactersFlag](#maskcharactersflag)    |  | Boolean| If true, input should be masked with '*'. Present if [InputCommand](#data-dictionary-inputcommand) = "Password"

#### Input response

> Input response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Input",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "InputResponse":{
         "OutputResult":{
            "Device":"CashierDisplay",
            "InfoQualify":"POIReplication",
            "Response":{
               "Result":"xxx",
               "ErrorCondition":"xxx",
               "AdditionalResponse":"xxx"
            }
         },
         "InputResult":{
            "Device":"CashierInput",
            "InfoQualify":"xxx",
            "Response":{
               "Result":"xxx",
               "ErrorCondition":"xxx",
               "AdditionalResponse":"xxx"
            },
            "Input":{
               "InputCommand":"xxx",
               "ConfirmedFlag":"true or false",
               "Password":"xxx",
               "MenuEntryNumber":"xxx"
            }
         }
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Input"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid    )               | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[DeviceID](#deviceid)                     | ✔ | String | Mirrored from input request
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from input request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from input request

**InputResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*OutputResult*                                |    | Object | Present if `DisplayOutput` is present in the request
 [Device](#device)                            | ✔ | String | Mirrored from input request
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | Mirrored from input request
 *Response*                                   | ✔ | Object | 
  [Result](#data-dictionary-result)                           | ✔ | String | "Success", "Partial", or "Failure". See [Result](#data-dictionary-result).
  [ErrorCondition](#data-dictionary-errorcondition)           |    | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
  [AdditionalResponse](#data-dictionary-additionalresponse)   |    | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
*InputResult*                                 | ✔ | Object | Information related to the result the input 
 [Device](#device)                            | ✔ | String | Mirrored from input request
 [InfoQualify](#data-dictionary-infoqualify)                  | ✔ | String | Mirrored from input request
 *Response*                                   | ✔ | Object | 
  [Result](#data-dictionary-result)                           | ✔ | String | "Success", "Partial", or "Failure". See [Result](#data-dictionary-result).
  [ErrorCondition](#data-dictionary-errorcondition)           |    | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
  [AdditionalResponse](#data-dictionary-additionalresponse)   |    | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
 *Input*                                      | ✔ | Object | 
  [InputCommand](#data-dictionary-inputcommand)               |    | String | Mirrored from input request
  [ConfirmedFlag](#confirmedflag)             |    | Boolean| Result of GetConfirmation input request. Present if [InputCommand](#data-dictionary-inputcommand) = "GetConfirmation"
  [Password](#password)                       |    | String | Password entered by the Cashier. Mandatory, if [InputCommand](#data-dictionary-inputcommand) is "Password". Not allowed, otherwise
  [MenuEntryNumber](#menuentrynumber)         |    | Number | A number from 1 to n, when n is total number of objects in `MenuEntry` of `InputRequest`. Mandatory, if [InputCommand](#data-dictionary-inputcommand) is "GetMenuEntry". Not allowed, otherwise
  [TextInput](#textinput)                     |    | String | Value entered by the Cashier. Mandatory, if [InputCommand](#data-dictionary-inputcommand) is "TextString" or "DecimalString". Not allowed, otherwise
  [DigitInput](#digitinput)                   |    | String | Value entered by the Cashier. Mandatory, if [InputCommand](#data-dictionary-inputcommand) is "DigitInput". Not allowed, otherwise


### Print

During a payment, the POI System may send print requests to the Sale System if a receipt is to be printed before the payment response can be finalised (e.g. when a signature is required).

In this case the Sale System should examine the properties in the print request and print the receipt accordingly. 

The final payment receipt, which is to be included in the Sale receipt, is returned in the payment response.

 
#### Print request

> Print request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Print",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "PrintRequest":{
         "PrintOutput":{
            "DocumentQualifier":"CashierReceipt",
            "IntegratedPrintFlag":false,
			"RequiredSignatureFlag":true,
            "OutputContent":{
               "OutputFormat":"XHTML",
               "OutputXHTML":"xxxx"
            },
		},	
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Print"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from payment request
[DeviceID](#deviceid)                     | ✔ | String | Unique message identifier
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment request

**PrintRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
 **PrintOutput**                              | ✔ | Object | 
  [DocumentQualifier](#documentqualifier)     | ✔ | String | "CashierReceipt" for a merchant receipt, otherwise "CustomerReceipt"
  [IntegratedPrintFlag](#integratedprintflag) |  |Boolean| True if the receipt should be included with the Sale receipt, false if the receipt should be printed now and paper cut (e.g. for a signature receipt)
  [RequiredSignatureFlag](#requiredsignatureflag) | ✔|Boolean| If true, the card holder signature is required on the merchant CashierReceipt.
  **OutputContent**                           |  | Array | Array of payment receipt objects which represent receipts to be printed
   [OutputFormat](#data-dictionary-outputformat)              | ✔ | String | "XHTML"  
   [OutputXHTML](#data-dictionary-outputxhtml)                | ✔ | String | The payment receipt in XHTML format but coded in BASE64 


#### Print response

> Print response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Device",
         "MessageCategory":"Print",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "PrintResponse":{
	     "DocumentQualifier":"CashierReceipt",
         "Response":{
            "Result":"xxx",
            "ErrorCondition":"xxx",
            "AdditionalResponse":"xxx"
         },
      },
      "SecurityTrailer":{...}
   }
}
```


**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Device"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Print"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid    )               | ✔ | String | Mirrored from print request 
[DeviceID](#deviceid)                     | ✔ | String | Mirrored from print request
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from print request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from print request

**PrintResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
[DocumentQualifier](#documentqualifier)       | ✔ | String | Mirrored from print request
 *Response*                                   | ✔ | Object | 
  [Result](#data-dictionary-result)                           | ✔ | String | "Success", "Partial", or "Failure". See [Result](#data-dictionary-result).
  [ErrorCondition](#data-dictionary-errorcondition)           |    | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
  [AdditionalResponse](#data-dictionary-additionalresponse)   |    | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 



### Transaction status 

A transaction status request can be used to obtain the status of a previous transaction. Required for error handling. 

#### Transaction status request

> Transaction status request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"TransactionStatus",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxxx"
      },
      "TransactionStatusRequest":{
         "MessageReference":{
            "MessageCategory":"xxx",
            "ServiceID":"xxx",
            "SaleID":"xxx",
            "POIID":"xxx"
         }
      },
      "SecurityTrailer":{...}
   }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "TransactionStatus"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Uniquely identifies the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**TransactionStatusRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*MessageReference*                            |    | Object | Identification of a previous POI transaction. Present if it contains any data. 
 [MessageCategory](#data-dictionary-messagecategory)          |    | String | "Payment"
 [ServiceID](#data-dictionary-serviceid)                      |    | String | The [ServiceID](#data-dictionary-serviceid) of the transaction to retrieve the status of. If not included the last payment status is returned.
 [SaleID](#data-dictionary-saleid)                            |    | String | The [SaleID](#data-dictionary-saleid) of the transaction to retrieve the status of. Only required if different from the [SaleID](#data-dictionary-saleid) provided in the `MessageHeader`
 [POIID](#data-dictionary-poiid)                              |    | String | The [POIID](#data-dictionary-poiid) of the transaction to retrieve the status of. Only required if different from the [POIID](#data-dictionary-poiid) provided in the `MessageHeader`


#### Transaction status response

> Transaction status response

```json
{
   "SaleToPOIResponse":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"TransactionStatus",
         "MessageType":"Response",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "TransactionStatusResponse":{
         "Response":{
            "Result":"xxx",
            "ErrorCondition":"xxx",
            "AdditionalResponse":"xxx"
         },
         "MessageReference":{
            "MessageCategory":"xxx",
            "ServiceID":"xxx",
            "SaleID":"xxx",
            "POIID":"xxx"
         },
         "RepeatedMessageResponse":{
            "MessageHeader":{...},
            "RepeatedResponseMessageBody":{
               "PaymentResponse":{...},
               "ReversalResponse":{...}
            }
         }
      }
   }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "TransactionStatus"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | Mirrored from request
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from request

**TransactionStatusResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*Response*                                    | ✔ | Object | Object indicating the result of the payment
 [Result](#data-dictionary-result)                            | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)            |  | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)    |  | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
*MessageReference*                            |  | Object | Identification of a previous POI transaction. Present if `Result` is "Success", or `Result` is "Failure" and `ErrorCondition` is "InProgress"
 [MessageCategory](#data-dictionary-messagecategory)          | ✔ | String | Mirrored from request
 [ServiceID](#data-dictionary-serviceid)                      | ✔ | String | Mirrored from request, or `ServiceID` of last transaction if not present in request.
 [SaleID](#data-dictionary-saleid)                            |  | String | Mirrored from request, but only if present in the request
 [POIID](#data-dictionary-poiid)                              |  | String | Mirrored from request, but only if present in the request
*RepeatedMessageResponse*                     |  | Object | Present if `Result` is "Success"
 *MessageHeader*                              | ✔ | Object | `MessageHeader` of the requested payment
 *PaymentResponse*                            | ✔ | Object | `PaymentResponse` of the requested payment


### Abort transaction

The Sale System can send an `abort transaction` message to request cancellation of the in-progress transaction. 

<aside class="success">
Cancel transaction is a "request to cancel". Cancellation of the transaction is not guaranteed. There are a number of instances where cancellation is not possible (for example, when the payment has already completed). 

After sending a cancel transaction request, the Sale System should <b>always</b> wait for the payment/card acquisition response and validate the success of the sale by checking the `Result` property.
</aside>



#### Abort transaction request

> Abort transaction request

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Service",
         "MessageCategory":"Abort",
         "MessageType":"Request",
         "ServiceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "AbortRequest":{
         "MessageReference":{
            "MessageCategory":"xxx",
            "ServiceID":"xxxx",
            "SaleID":"xxx",
            "POIID":"xxx"
         },
         "AbortReason":"xxx"
      },
      "SecurityTrailer":{...}
   }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Abort"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Unique identifier for the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**AbortRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*MessageReference*                            | ✔ | Object | Identification of a POI transaction
 [MessageCategory](#data-dictionary-messagecategory)          | ✔ | String | "Payment" or "CardAcquisition"
 [ServiceID](#data-dictionary-serviceid)                      | ✔ | String | The [ServiceID](#data-dictionary-serviceid) of the transaction to cancel
 [SaleID](#data-dictionary-saleid)                            |  | String | The [SaleID](#data-dictionary-saleid) of the transaction to cancel. Only required if different from the [SaleID](#data-dictionary-saleid) provided in the `MessageHeader`
 [POIID](#data-dictionary-poiid)                              |  | String | The [POIID](#data-dictionary-poiid) of the transaction to cancel. Only required if different from the [POIID](#data-dictionary-poiid) provided in the `MessageHeader`
[AbortReason](#abortreason)                   | ✔ | String | Any text describing the reason for cancelling the transaction. For example, "User cancel"


#### Abort transaction response

There is no direct response to an Abort message.

If the transaction can be aborted, a payment response is returned with `Result` = "Failure" and `ErrorCondition` = "Aborted".

If the transaction cannot be aborted, a normal payment response is sent back in time.

However, if the Abort Request cannot be accepted due to a message format error or if it references a transaction that is not found, an Event Notification is returned. 

> Abort transaction response

```json
{
   "SaleToPOIRequest":{
      "MessageHeader":{
         "MessageClass":"Event",
         "MessageCategory":"Event",
         "MessageType":"Notification",
         "DeviceID":"xxx",
         "SaleID":"xxx",
         "POIID":"xxx"
      },
      "EventNotification":{
         "TimeStamp":"xxx",
         "EventToNotify":"xxx",
         "EventDetails":"xxx"
      },
      "SecurityTrailer":{...}
   }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Event"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Event"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Notification"
[DeviceID](#deviceid)                     | ✔ | String | Unique message identifier
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment request

**EventNotification**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
 [TimeStamp](#data-dictionary-timestamp)                      | ✔ | String | Time of the event on the POI System, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)
 [EventToNotify](#data-dictionary-eventtonotify)              | ✔ | String | "Reject" if the abort request cannot be accepted (e.g. message format error, `ServiceId` not found). "CompletedMessage" if payment has already completed.
 [EventDetails](#data-dictionary-eventdetails)                | ✔ | String | Extra detail on the reason for the event


### Reconciliation

#### Reconciliation request

> Abort transaction request

```json
{
  "SaleToPOIRequest":{
    "MessageHeader":{
      "MessageClass":"Service",
      "MessageCategory":"Reconciliation",
      "MessageType":"Request",
      "ServiceID":"xxxx",
      "SaleID":"xxx",
      "POIID":"xxx"
    },
    "ReconciliationRequest":{
      "ReconciliationType":"xxx",
      "POIReconciliationID":"xxx"
    },
    "SecurityTrailer":{...}
  }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Reconciliation"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Unique identifier for the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**ReconciliationRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
[ReconciliationType](#reconciliationtype)     | ✔ | String | "SaleReconciliation" to close the current period, "PreviousReconciliation" to request the result of a previous reconciliation
[POIReconciliationID](#data-dictionary-poireconciliationid)   | ✔ | String | Present if ReconciliationType is "PreviousReconciliation". See [POIReconciliationID](#data-dictionary-poireconciliationid)


#### Reconciliation response

> Reconciliation response

```json
{
  "SaleToPOIResponse":{
    "MessageHeader":{
      "MessageClass":"Service",
      "MessageCategory":"Reconciliation",
      "MessageType":"Response",
      "ServiceID":"xxx",
      "SaleID":"xxx",
      "POIID":"xxx"
    },
    "ReconciliationResponse":{
      "Response":{
        "Result":"xxx",
        "ErrorCondition":"xxx",
        "AdditionalResponse":"xxx"
      },
      "ReconciliationType":"xxx",
      "POIReconciliationID":"xxx",
      "TransactionTotals":[
        {
          "PaymentInstrumentType":"xxx",
          "CardBrand":"xxx",
          "OperatorID":"xxx",
          "ShiftNumber":"xxx",
          "TotalsGroupID":"xxx",
          "PaymentCurrency":"AUD",
          "PaymentTotals":[
            {
              "TransactionType":"xxx",
              "TransactionCount":"xxx",
              "TransactionAmount":"0.00",
			  "TipAmount":"0.00",
			  "SurchargeAmount":"0.00"
            }
          ]
        }
      ]
    },
    "SecurityTrailer":{...}
  }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Reconciliation"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[DeviceID](#deviceid)                     | ✔ | String | Unique message identifier
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment request

**ReconciliationResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*Response*                                    | ✔ | Object | Object indicating the result of the login
 [Result](#data-dictionary-result)                            | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)            |  | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)    |  | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
[ReconciliationType](#reconciliationtype)     | ✔ | String | Mirrored from request
[POIReconciliationID](#data-dictionary-poireconciliationid)   |  | String | Present if `Result` is "Success". The `ReconciliationID` of the period requested
*TransactionTotals*                           |  | Array | Present if `Result` is "Success". An array of totals grouped by card brand, then operator, then shift, then TotalsGroupID, then payment currency.
 [PaymentInstrumentType](#data-dictionary-paymentinstrumenttype)| ✔ | String | "Card" (card payment) or "Mobile" (phone/QR code payments)
 [CardBrand](#data-dictionary-cardbrand)                      |  | String | A card brand used during this reconciliation period 
 [OperatorID](#data-dictionary-operatorid)                    |  | String | An operator id used during this reconciliation period
 [ShiftNumber](#data-dictionary-shiftnumber)                  |  | String | A shift number used during the reconciliation period
 [TotalsGroupID](#data-dictionary-totalsgroupid)              |  | String | A custom grouping of transactions as defined by the Sale System
 [PaymentCurrency](#data-dictionary-paymentcurrency)          |  | String | "AUD"
 *PaymentTotals*                              |  | Array | An array [0..10] of totals grouped by transaction payment type. Present if both `TransactionCount` and `TransactionAmount` are not equal to zero
  [TransactionType](#data-dictionary-transactiontype)         |  | String | Transaction type for this payment. See [TransactionType](#data-dictionary-transactiontype)
  [TransactionCount](#transactioncount)       |  | String | The number of transactions for the transaction type for the current grouping of transactions
  [TransactionAmount](#transactionamount)     |  | Number | The total amount of transactions for the transaction type for the current grouping of transactions
 

### Card acquisition

The card acquisition request allows the Sale System to tokenise a card which can be used in future payment requests.

#### Card acquisition request

> Card acquisition request

```json
{
  "SaleToPOIRequest":{
    "MessageHeader":{
      "MessageClass":"Service",
      "MessageCategory":"CardAcquisition",
      "MessageType":"Request",
      "ServiceID":"xxx",
      "SaleID":"xxx",
      "POIID":"xxx"
    },
    "CardAcquisitionRequest":{
      "SaleData":{
        "OperatorID":"xxx",
        "OperatorLanguage":"en",
        "ShiftNumber":"xxx",
        "CustomerLanguage":"en",
        "SaleTransactionID":{
          "TransactionID":"xxx",
          "TimeStamp":"xxx"
        },
        "SaleTerminalData":{
          "TerminalEnvironment":"xxx",
          "SaleCapabilities":[
            "xxx",
            "xxx",
            "xxx",
            "…"
          ]
        },
        "TokenRequestedType":"xxx"
      },
      "CardAcquisitionTransaction":{
        "AllowedPaymentBrand":[
          "xxx",
          "xxx",
          "xxx",
          "…"
        ],
        "ForceEntryMode":"xxx"
      }
    },
    "SecurityTrailer":{...}
  }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Reconciliation"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Request"
[ServiceID](#data-dictionary-serviceid)                   | ✔ | String | A unique value which will be mirrored in the response. See [ServiceID](#data-dictionary-serviceid).
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Unique identifier for the Sale System
[POIID](#data-dictionary-poiid)                           | ✔ | String | Uniquely identifies the POI Terminal

**CardAcquisitionRequest**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*SaleData*                                    | ✔ | Object | Object Sale System information attached to this payment
 [OperatorID](#data-dictionary-operatorid)                    |  | String | Only required if different from Login Request
 [OperatorLanguage](#operatorlanguage)        |  | String | Set to "en"
 [ShiftNumber](#data-dictionary-shiftnumber)                  |  | String | Only required if different from Login Request
 [CustomerLanguage](#customerlanguage)        |  | String | Set to "en" for English
 [TokenRequestedType](#data-dictionary-tokenrequestedtype)    | ✔ | String | "Customer"
 *SaleTransactionID*                          |  | Object | 
  [TransactionID](#data-dictionary-transactionid)             | ✔ | String | Unique reference for this sale ticket
  [TimeStamp](#data-dictionary-timestamp)                     | ✔ | String | Time of initiating the request on the POI System, formatted as [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) DateTime. e.g. "2019-09-02T09:13:51.0+01:00"   
 *SaleTerminalData*                           |  | Object | Define Sale System configuration. Only include if elements within have different values to those in Login Request
  [TerminalEnvironment](#data-dictionary-terminalenvironment) |  | String | "Attended", "SemiAttended", or "Unattended"
  [SaleCapabilities](#data-dictionary-salecapabilities)       |  | Array  | Advises the POI System of the Sale System capabilities. See [SaleCapabilities](#data-dictionary-salecapabilities) 
*CardAcquisitionTransaction*                  |  | Object | Present if any of the JSON elements within are present
  [AllowedPaymentBrands](#data-dictionary-allowedpaymentbrands)|  | Array  | Restricts the request to specified card brands. See [AllowedPaymentBrands](#data-dictionary-allowedpaymentbrands)
  [ForceEntryMode](#data-dictionary-forceentrymode)           |  | String| If present, restricts card presentment to the specified type. See [ForceEntryMode](#data-dictionary-forceentrymode)

#### Reconciliation response

> Reconciliation response

```json
{
  "SaleToPOIResponse":{
    "MessageHeader":{
      "MessageClass":"Service",
      "MessageCategory":"Reconciliation",
      "MessageType":"Response",
      "ServiceID":"xxx",
      "SaleID":"xxx",
      "POIID":"xxx"
    },
    "ReconciliationResponse":{
      "Response":{
        "Result":"xxx",
        "ErrorCondition":"xxx",
        "AdditionalResponse":"xxx"
      },
      "ReconciliationType":"xxx",
      "POIReconciliationID":"xxx",
      "TransactionTotals":[
        {
          "PaymentInstrumentType":"xxx",
          "CardBrand":"xxx",
          "OperatorID":"xxx",
          "ShiftNumber":"xxx",
          "TotalsGroupID":"xxx",
          "PaymentCurrency":"AUD",
          "PaymentTotals":[
            {
              "TransactionType":"xxx",
              "TransactionCount":"xxx",
              "TransactionAmount":"xxx"
            }
          ]
        }
      ]
    },
    "SecurityTrailer":{...}
  }
}
```

**MessageHeader**

<div style="width:180px">Attribute</div>  |Requ.| Format | Description |
-----------------                         |----| ------ | ----------- |
[MessageClass](#data-dictionary-messageclass)             | ✔ | String | "Service"
[MessageCategory](#data-dictionary-messagecategory)       | ✔ | String | "Reconciliation"
[MessageType](#data-dictionary-messagetype)               | ✔ | String | "Response"
[DeviceID](#deviceid)                     | ✔ | String | Unique message identifier
[SaleID](#data-dictionary-saleid)                         | ✔ | String | Mirrored from payment request
[POIID](#data-dictionary-poiid)                           | ✔ | String | Mirrored from payment request

**ReconciliationResponse**

<div style="width:180px">Attribute</div>      |Requ.| Format  | Description |
-----------------                             |----| ------ | ----------- |
*Response*                                    | ✔ | Object | Object indicating the result of the login
 [Result](#data-dictionary-result)                            | ✔ | String | Indicates the result of the response. Possible values are "Success" and "Failure"
 [ErrorCondition](#data-dictionary-errorcondition)            |  | String | Indicates the reason an error occurred. Only present when `Result` is "Failure". See [ErrorCondition](#data-dictionary-errorcondition) for more information on possible values.
 [AdditionalResponse](#data-dictionary-additionalresponse)    |  | String | Provides additional error information. Only present when `Result` is "Failure". See [AdditionalResponse](#data-dictionary-additionalresponse) for more information on possible values. 
[ReconciliationType](#reconciliationtype)     | ✔ | String | Mirrored from request
[POIReconciliationID](#data-dictionary-poireconciliationid)   |  | String | Present if `Result` is "Success". The `ReconciliationID` of the period requested
*TransactionTotals*                           |  | Array | Present if `Result` is "Success". An array of totals grouped by card brand, then operator, then shift, then TotalsGroupID, then payment currency.
 [PaymentInstrumentType](#data-dictionary-paymentinstrumenttype)| ✔ | String | "Card" (card payment) or "Mobile" (phone/QR code payments)
 [CardBrand](#data-dictionary-cardbrand)                      |  | String | A card brand used during this reconciliation period. See [CardBrand](#data-dictionary-cardbrand)
 [OperatorID](#data-dictionary-operatorid)                    |  | String | An operator id used during this reconciliation period
 [ShiftNumber](#data-dictionary-shiftnumber)                  |  | String | A shift number used during the reconciliation period
 [TotalsGroupID](#data-dictionary-totalsgroupid)              |  | String | A custom grouping of transactions as defined by the Sale System
 [PaymentCurrency](#data-dictionary-paymentcurrency)          |  | String | "AUD"
 *PaymentTotals*                              |  | Array | An array [0..10] of totals grouped by transaction payment type. Present if both `TransactionCount` and `TransactionAmount` are not equal to zero
  [TransactionType](#data-dictionary-transactiontype)         |  | String | Transaction type for this payment. See [TransactionType](#data-dictionary-transactiontype)
  [TransactionCount](#transactioncount)       |  | String | The number of transactions for the transaction type for the current grouping of transactions
  [TransactionAmount](#transactionamount)     |  | Number | The total amount of transactions for the transaction type for the current grouping of transactions

 
## Error handling

When the Sale System sends a request, it will receive a matching response. For example, if the Sale System sends a [payment request](#payment_request) it will receive a [payment response](#payment-response).

The Sale System should handle errors by checking the [Response.Result](#data-dictionary-result) and [Response.ErrorCondition](#data-dictionary-errorcondition) fields in the response.

In the event the Sale System does not receive a response (for example, due to network error or timeout) it must enter an error handling loop.

Error handling due to network error or timeout is outlined in the diagram below. 

1. Cashier initiates a payment
1. Sale System sends a [payment request](#payment)
1. Network error or timeout occurs
1. Sale System awaits Internet availability
1. Sale System enters a loop for a maximum of 90 seconds
1. Sale System sends a [transaction status request](#transaction-status-request) and awaits a [transaction status response](#transaction-status-response)
1. Sale System handles the transaction status response 
  1. If `Response.Result` is "Success", the transaction result will be contained in `RepeatedMessageResponse` and the Sale System can exit the error loop
  1. If `Response.Result` is "Failure" and `Response.ErrorCondition` is "InProgress" the Terminal is still processing the payment. The Sale System should wait 5 seconds and send the [transaction status request](#transaction_status_request) again.
  1. If `Response.Result` is "Failure" and `Response.ErrorCondition` is not "InProgress" the payment failed

<aside class="success">
If the Sale System is unable retrieve a result within 90 seconds the payment has failed.
</aside>

![](images/payment-error-handling.png)


**Power failure handling**

The Sale System can handle a power failure by checking a local transaction status on start up.

1. Cashier initiates a payment
1. Sale System sets txn_in_progress flag
1. Sale System sends a [payment request](#payment)
1. Sale System power failure occurs
1. Sale System start up after power failure
1. Sale System checks if txn_in_progress flag is set. If so, enter error handling

![](images/payment-error-handling-power-failure.png)

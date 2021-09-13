# Appendix

## Glossary of terms

- **API** - application programming interface.
- **Card holder** - the owner of a credit or debit card.
- **Cashier** - operator of the Sale System.
- **CHD** - cardholder data. Personally identifiable information associated with a person who has a credit or debit card.
- **JSON** - JavaScript object notation.
- **MAC** - message authentication code.
- **Merchant** - a business entity which can accept payments.
- **Payment** - a request sent from a `Sale System` to a `POI System` to perform an operation on a `Card holders` card.
- **Payment receipt** - receipt generated by the `POI Terminal` which contains information about the payment
- **PIN** - personal identification number.
- **POI** - point of interaction.
- **POI System** - system which co-ordinates communication between a Sale System and POI Terminal. i.e. DataMesh Unify.
- **POI Terminal** - the physical PIN entry device which processes the card payment. i.e. DataMesh Satellite.
- **POS** - point of sale.
- **Sale** - a sale, created by a `Cashier` on a `Sale System`, with associated `Sale Items`. A Sale may be paid with one or more `Payments`.
- **Sale item** - a item sold to a card holder as a part of a sale.
- **Sale receipt** - the receipt generated by the `Sale System` which contains all sale information. The `Payment receipt` may be embedded into the `Sale receipt`
- **Sale system** - the system used to process merchant sales. The POS.
- **Satellite app** - the DataMesh payment application 
- **Unify** - the DataMesh payment switch
- **UUID** - universally unique identifier.


## ProtocolVersion updates from 3.1 to 3.1-dmg

The DataMesh API supports two versions. 
- "3.1" is based on the Nexo 3.1 standard
- "3.1-dmg" is "3.1" with additional fields added by DataMesh which are outside the Nexo standard (e.g. surcharge)

Breaking changes when moving from "3.1" to "3.1-dmg"

- [MessageHeader.ProtocolVersion](#data-dictionary-protocolversion) should be set to "3.1-dmg"
- Tip amount is now present in [PaymentResponse.PaymentResult.AmountsResp.TipAmount](#tipamount).
- Surchage amount is now present in [PaymentResponse.PaymentResult.AmountsResp.SurchargeAmount](#surchargeamount).
- `PaymentResponse.PaymentResult.PaymentReceipt` is now an array of receipts. This was a single object in "3.1"
- For a successful payment, the acquirer STAN is returned in `PaymentResponse.PaymentResult.PaymentAcquirerData.STAN`. 
- For a successful payment, the acquirer RRN is returned in `PaymentResponse.PaymentResult.PaymentAcquirerData.RRN`. 
- POS should support the "3.1-dmg" `mandatory features checklist`
  - Support for [purchase](#cloud-api-reference-perform-a-purchase) and [refund](#cloud-api-reference-perform-a-refund) payment types
  - Include [product data](#cloud-api-reference-product-data) in each payment request
  - Additional fields will be added to the message specification over time. To ensure forwards compatibility the Sale System must ignore when extra objects and fields are present in response messages. This includes valid MAC handling in the SecurityTrailer.
  - Support for TLS and other [security requirements](#cloud-api-reference-security-requirements)
  - [Settings user interface](#settings-user-interface)
  - [Payments user interface](#payment-user-interface) which handles the `Initial UI`, `Final UI`, and `cancelling a sale in progress`
  - Handle error scenarios as outlined in [error handling](#cloud-api-reference-error-handling)
  - Ensure Sale System provides a unique [payment identification](#payment-identification)




## Terminal configuration 

### PAX terminals

#### Enable access to the home screen

In a production environment a terminal will typically be locked to the payment application, however in development and test it is useful to enable access to the home screen. 

- In the Satellite payment app, tap the settings icon (⚙)

![](images/pinpad-merchant-password-270x480.png)

- Enter `0000` as the merchant password and ensure the following are ticked
  - Navigation bar
  - Status Bar

![](images/pinpad-settings-270x480.png)

- Tap the back icon (⬅) 
- Hit the "home" button on the terminal to view the home page



#### Configure terminal environment

Your development terminal will be conncted to the UAT environment, which is the correct environemnt for testing. 

To confirm or change the environemnt:

- Launch the Satellite payment app
- Tap the settings icon (⚙)
- Enter 8801 as the merchant password
- You'll be presented with a "Change Host URL" dialog
- Ensure the URL is `wss://test.datameshgroup.com.au/peduat1` and tap Enter

![](images/pinpad-host-url-270x480.png)

- Restart the Satellite payment app 
  - Tap the task switcher button
  - Close the Satellite payment app
  - Launch the Satellite payment app again 


#### Connect to Wi-Fi

The terminal can be connected to Wi-Fi where cellular access isn't avaiable. 

- Press the Home button 
- Launch the "Settings" app (⚙icon)

![](images/pinpad-launch-settings-270x480.png)

- If it asks for a password, enter `pax9876@@`
- Select Wi-Fi open, choose your Wi-Fi network and enter the password
- You should see your Wi-Fi network with a "Connected" status underneath 

#### Update settings

The terminal will periodically connect to the host and download updated settings.

To force a settings update:

- Launch the Satellite payment app
- Tap the "info" icon (ⓘ) at the top of the screen
- The terminal will update settings from the host
- Then current software version, terminal ID, and merchant ID will be displayed on screen

![](images/pinpad-info-270x480.png)

#### Update software

The terminal will periodically connect to the host and update to the configured software version. 

To force a software update:

- Launch the Satellite payment app
- Tap the settings icon (⚙) at the top of the screen
- Enter `0000` as the merchant password
- Select the `SOFTWARE UPGRADE` option and select `YES` on the confirmation dialog
- The software will be downloaded, and Satellite will restart

#### Checking Wi-Fi connection

To check Wi-FI connection status at any time:

- Swipe down from the top of the screen to open the notification shade
- Swipe down again to expand the notification shade
- Top left icon will be your Wi-Fi status and connected access point

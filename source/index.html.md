---
title: Fusion API

language_tabs: # must be one of https://git.io/vQNgJ
  - json: JSON
  - java: Java

toc_footers:
  - 2020-08-19
  - <a href='https://www.datameshgroup.com'>DataMesh Group</a>

includes:
  - 00-introduction
  - 01-getting-started
  - 02-cloud-api-reference
  - 03-satellite-api-reference  
  - 04-data-dictionary
  - 05-testing
  - 06-production
  - 07-appendix

search: true

code_clipboard: true
---


<!--

TODO

Matched refunds - we require a POS to match refunds to original purchases. This is to enable support for alternative payments (e.g. Alipay/WeChat). 

At the moment we require a field which isn't a "human readable" field (a GUID) that isn't present on the receipt. This will work when a POS can look 
up a previous sale (i.e. when they have a central database between stores) but not for POS systems which don't have connectivity. 

Fix for this is to add a new "Transaction" status request which will work based on details from the receipt (e.g. TID/MID/DATETIME/STAN etc)


-----------

Not Implemented
In the payment request, nothing under PaymentTransaction.TransactionConditions isn’t implements

-----------

Do we want barcode/QR code in the receipts?

Is basket data required for refunds ?
What is ApprovalCode
What is LastTransactionFlag
SplitPaymentFlag - not included, therefor no special processing for split payments? Why support PaidAmount and MinimumSplitAmount?.
Split - does basket need to match RequestedAmount? How do we deal with baskets when split payments? 
Do we need basket in refund?

Tokenisation

-->
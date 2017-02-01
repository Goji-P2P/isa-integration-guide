# Goji Innovative Finance ISA Integration Guide

      ##### v0.1.0 Monday 11th January 2016
      ##### v0.1.1 Thursday 17th March 2016
      ##### v0.1.2 Tuesday 20th December 2016
      ##### v0.1.3 Wednesday 11th January 2017
      ##### v0.1.4 Wednesday 25th January 2017

      ## Legal Note

      This document is the property of Goji Holdings Limited. It may not be reproduced, stored in a retrieval system or transmitted in any form or by any means, either in whole or in part, without prior written permission of Goji Holdings Limited. Goji Holdings Limited is the owner of the intellectual property rights of this document. All such rights are reserved.
      You may download extracts, of any page(s) from this document for your personal reference. You must not modify the paper or digital copies of this document in any way, and you must not use any illustrations or any graphics separately from any accompanying text.

      ## Table of Contents

       * [Introduction](#introduction)
       * [API Integration](#api-integration)
         * [Changes to Originator's Systems Data Structures](#data-structures)
         * [Lifecycle Integration Points](#lifecycle-integration)
           * [Opening an ISA](#opening-isa)
           * [Cooling Off Period](#cooling-off)
         * [Funding an ISA](#funding-isa)
           * [Validation](#funding-validation)
           * [Cash Deposits](#cash-deposits)
           * [Transferring Funds](#funds-transfer)
           * [Transferring from an existing ISA](#transfers-in-intro)
         * [Investing](#investing)
           * [Earmarking money](#earmarking)
           * [Create Investment](#create-investment)
           * [Repayments](#repayments)
           * [Reinvestments](#reinvestments)
           * [Sale of Investment](#saleofinvestment)
           * [Defaults and Write-Offs](#writeOffs)
         * [Withdrawing Funds](#withdrawing-funds)
         * [Edit Investor Details](#edit-investor)
       * [Admin Console](#admin-console)
         * [ISA Workflows](#isa-workflows)
           * [Transfers In](#transfers-in)
           * [Transfers Out](#transfers-out)
           * [Repairing and Voiding ISAs](#repair-void)
             * [Voiding an ISA](#voiding)
             * [Repairing an ISA](#repairing)
           * [Death of Investor](#investor-death)
       * [Regulatory Reporting](#reporting)
       * [Integrating the Transfer In Pages](#transfer-in-integration)
       * [HMAC Authentication](#hmac-authentication)
       * [API Error Codes](#api-error-codes)

      ## <a name="introduction"></a> Introduction

      The Goji Innovative Finance ISA Platform (Goji Platform) provides the administration, regulatory and compliance functionality to enable the Goji team to efficiently handle all back-office aspects of the Innovative Finance ISA on behalf of a P2P Platform and provide best-of-class 2nd level customer support.

      The Goji Platform provides a number of key features to enable efficient IF ISA administration:

      * Regulatory reporting to HMRC
      * Workflow management for ISA admin tasks eg transferring an ISA in/out, void & repair, death of investor etc.
      * Business rule validation on data input (e.g. check ISA limits, NI number checking)
      * Customer support instant messaging, ticketing and originator FAQs
      * Management information and insights into types of customers enabling improved customer marketing

      This document details how an Originator integrates with the Goji Platform.
      The intended audience of this document are the developers and analysts responsible for building the integration between the Originator's systems and the Goji Platform.

      The Goji Platform requires data from the Originator at a number of points in the Investor lifecycle to facilitate HMRC reporting, process workflow management and data validation (see API Integration below)

      Additionally, the Goji platform provides an admin interface to allow Originators and Goji staff to manage ISA specific workflows eg transfers in/out, repairing/voiding ISAs, death of investor etc.

      ## <a name="api-integration"></a> API Integration

      The Goji Platform exposes a RESTful API over HTTPS. Full details of the API are available in [separate documentation](https://goji-api.herokuapp.com/docs).

      ### <a name="lifecycle-integration"></a> Investor Lifecycle Integration Points

      Data will need to be submitted to the Goji Platform at the following points in an Investor's lifecycle.

      #### <a name="opening-isa"></a> Opening an ISA

      When a customer on the Originator's platform elects to open an ISA, the investor first needs to be shown an ISA declaration and the current set of terms and conditions. The latest version of the declaration and terms and conditions can be retrieved by calling:

          GET /declaration

          GET /terms

      A record of the investor agreeing needs to be stored by calling:

          POST /terms

          POST /declaration

      A token is returned by these endpoints which needs to be included in the following call to create the Investor.

      Once these documents have been agreed to, the Investor can be created on the Goji Platform by calling:

          POST /investors

      Validation will be performed to ensure that the data provided is a) valid and b) includes the data needed to comply with HMRC reporting.

      We also validate that the Investor has not already opened an IF ISA on the Goji Platform with another Originator in the current tax year.

      It is assumed that the Originator has already performed KYC and AML checks by this point.

      ##### <a name="cooling-off"></a> Cooling off period

      Once the declaration and terms have been signed and the Investor details have been recorded, the ISA is considered 'provisionally opened'.
      It is not fully opened until a deposit is made. Some platforms may wish to apply a cooling-off period for an investor before they can invest. This is configurable on the Goji platform for an Originator. Please inform Goji if you wish to apply a cooling off period. This will block investments until the cooling off period has ended but still allow deposits.

      The current state of the ISA, including when the cooling off period ends, can be checked:

           GET /investors/{investorId}/isa

      Summary data for the ISA can be retrieved using:

           GET /investors/{investorId}/isa/summary

      #### <a name="funding-isa"></a> Funding an ISA

      The Investor's ISA can be funded by depositing cash, transferring funds from an existing account with the Originator
      or by transferring in an ISA held with another ISA manager.

      ##### <a name="funding-validation"></a> Validation

      When money is added to the ISA account, validation is applied to ensure the annual subscription amount is not exceeded.
      The transfer will be rejected in whole if it exceeds the limit.
      The remaining maximum deposit amount is returned on the ISA details endpoint:

           GET /investors/{investorId}/isa

      ##### <a name="cash-deposits"></a> Cash deposit

      If an Investor deposits money to their IF ISA account, this needs to be recorded on the Goji Platform:

           POST /investors/{investorId}/cash

      setting the `type` to `CUSTOMER_DEPOSIT`.

      ##### <a name="funds-transfer"></a> Transferring funds from an existing account

      If an Investor transfers money from an existing investment account on the Originator's platform to their IF ISA, this similarly needs to be recorded as a cash transaction:

           POST /investors/{investorId}/cash

      setting the `type` to `CUSTOMER_DEPOSIT`.

      ##### <a name="transfers-in-intro"></a> Transfers in

      If an Investor holds an existing cash/stocks and shares ISA with another ISA manager, the Investor can elect to transfer that balance to a different ISA Manager.
      Please see the section below which details how the Transfers In/Out process works.

      #### <a name="investing"></a> Investing

      Once an Investor has funded their account (and, if configured, the cooling off period has expired), they can invest up to the amount of the cash balance on their IF ISA account.

      ##### <a name="earmarking"></a> Earmarking funds

      Earmarking funds is used if an Investor has committed an amount of money to an investment but the funds have not yet been drawn down.
      This is required if an Investor is bidding on an investment or if they have added their money to a lending queue which will get drawn down when it is matched against an available loan.

      If the platform is already ensuring that the funds cannot be double committed, this call can be skipped.

      Money can be earmarked with the following call:

           POST /investors/{investorId}/earmarkedAmount

      This reduces the available cash balance of the ISA.

      If an Investor is unsuccessful in their investment bid or if they want to remove their money from the offer or queue, the earmarked amount should be deleted:

           DELETE /investors/{investorId}/earmarkedAmount/{earmarkedAmountId}

      If the earmarked amount is converted into an investment, this needs to be converted to an investment on the Goji Platform:

           POST /investors/{investorId}/earmarkedAmount/{earmarkedAmountId}/investment

      ##### <a name="create-investment"></a> Creating an investment

      If an Investor can invest their money without going via an auction or queue, the investment should be created directly.

           POST /investors/{investorId}/investment

      This will reduce the cash balance available on the ISA.

      ##### <a name="repayments"></a> Repayments

      When the borrower makes a repayment against a loan, this repayment needs to be recorded on the Goji Platform.
      A repayment increases the cash balance on an ISA by the interest component of the repayment.
      If the repayment is automatically re-invested, please see the [Reinvestments](#reinvestments) section below.

           POST /investors/{investorId}/investment/{investmentId}/repayment

      ##### <a name="reinvestments"></a> Reinvestments

      When a borrower makes a repayment against a loan, this repayment needs to be recorded on the Goji Platform.
      A reinvestment differs from a repayment in that a reinvestment automatically invests the repaid capital and interest.
      The balance of the ISA is increased by the interest portion of the reinvestment.
      The cash balance remains unchanged.

           POST /investors/{investorId}/investment/{investmentId}/reinvestment

      Alternatively, this can be achieved with two separate calls to record the repayment and the subsequent investment.

      #### <a name="saleofinvestment"></a> Sale of investment

      When an investor sells an investment this is recorded on the Goji Platform by making a repayment equal to the remaining capital balance on the investment.

          POST /investors/{investorId}/investment/{investmentId}/repayment

      Setting the `capitalAmount` to the amount of the sale and the `interestAmount` to zero (assuming there is no interest element to the sale).

      If a loan has been sold for more or less than the capital outstanding amount (eg in an auction), then there is a specific endpoint that can be called:

           POST /investors/{investorId}/investment/{investmentId}/sale

      ##### <a name="writeOffs"></a> Defaults and write offs

      If a loan has defaulted and either all or a portion of the loan should be written off, the write off endpoint can be called. This will reduce the ISA balance by the write off amount.

           POST /investors/{investorId}/investment/{investmentId}/writeOff

      If repayments need to be made after the full balance has been written off (eg recoveries), these should be processed as interest repayments.

      #### <a name="withdrawing-funds"></a> Withdrawing funds

      An Investor can withdraw funds up to the cash balance available on the ISA.
      The amount that can be withdrawn is available from the ISA details:

           GET /investors/{investorId}/isa

      When a user withdraws cash, this needs to be recorded on the Goji Platform as a cash transaction:

           POST /investors/{investorId}/cash

      setting the `type` to `CUSTOMER_WITHDRAWAL`. The amount sent is still a positive amount. The type will indicate to Goji to decrement the cash balance.

      #### <a name="edit-investor"></a> Editing investor details

      If an Investor's details need to be updated eg they change address, this needs to be reflected on the Goji platform:

           PUT /investors/{investorId}

      ## <a name="admin-console"></a> Admin Console

      The Goji Platform Admin Console allows Originator and/or Goji Customer Service Agents (CSA) to perform ISA administration tasks.
      All activity on the console is fully audited and secured by an authentication and authorisation model.

      ### <a name="isa-workflows"></a> ISA Workflows

      The following workflows for ISA management are supported on the Admin Console.

      #### <a name="transfers-in"></a> Transfers In

      A 'transfer in' is the process where an Investor moves the cash balance of an existing Cash and/or stocks and shares ISA from another ISA Manager to the Originator's platform.

      An Investor-facing web application will guide the Investor through the steps necessary to facilitate this process. A link to this web application can be added to the Originator's platform. Please see the <a href="#transfer-in-integration">section below for details</a> on how to integrate the web application.

      Once the process is complete, a workflow will be created and Goji will inform the 3rd party ISA plan manager of the Originator's bank details and the money will be transferred directly.

      Once the 3rd party ISA manager have sent Goji the details of the ISA being transferred, Goji will email the Originator to inform them to expect a transfer in.

      Once the transfer in funds have been received, the Originator would make call to

      			POST /investors/{investorId}/transferIn/{transferInId}/cash

      specifying the `transferAmount` and either the `repairedAmount` or `subscribedAmount`. These amounts can be determined based on the values Goji will provide upon receiving the transfer history form from the prior ISA manager. These values can be retrieved from the Goji Admin Console. The `transferAmount` is the total amount received from the previous ISA manager. The `repairedAmount` is any amount from this total that had to be transferred to a standard investment account to prevent the current year subscription amount from being exceeded. The `subscribedAmount` is the amount that is credited to the ISA account. If no funds have to be repaired, the `subscribedAmount` would equal the `transferAmount`.

      The `repairedAmount` refers to any part of the transferred amount that cannot be applied to the current year ISA as it would breach the subscription allowance. This portion should be credited to the Investor's standard investment account. The amount that was repaired needs to be recorded in the API call to Goji to ensure a complete record of where the money has been applied is created.

      Goji will email the Originator once the Transfer History Form has been received from the previous ISA Manager. This form includes the details of how the transferred amount should be split between current and prior year subscriptions. The Originator must not process the deposit of the transferred funds until these details have been received.

      #### <a name="transfers-out"></a> Transfers Out

      A 'transfer-out' is the process where an Investor elects to move their IF ISA from the Originator to an alternative ISA Manager.

      The new ISA Manager will contact Goji who will perform the checks necessary to validate that the transfer request is genuine.

      It is up to the terms and conditions of the agreement between the Originator and the Investor as to whether loan parts can be liquidated to meet the requirements of a transfer out.

      If this is allowed and the Investor has elected to liquidate their loans, Goji will inform the Originator what portion of loans need to be sold on the secondary market.

      Once there is enough cash in the IF ISA to enable the transfer out to complete, the cash needs to be transferred either directly to the new ISA manager or to Goji who will facilitate the transfer to the new ISA manager.

      Once the cash has been transferred, a call needs to be made to:

           POST /investors/{investorId}/cash

      setting the `type` to `TRANSFER_OUT`.

      The Originator platform will need to expose a mechanism to allow an IF ISA cash balance to be transferred to a designated bank account different to the bank account held by the investor.

      #### <a name="repair-void"></a> Repairing / voiding ISAs

      If HMRC deem that an Investor has breached any ISA regulations eg opened two IF ISAs in one tax year, then an ISA may need to be voided or repaired.

      ##### <a name="voiding"></a> Voiding an ISA

      A voided ISA is one that is no longer eligible to be tax exempt. Goji will complete the verification, admin and tax submissions required for this process.

      The Originator platform will need to expose a mechanism to allow an ISA account (cash balance and investments) to be either transferred to a 'standard' account (preferable) or for the investments to be liquidated and the cash returned to the investor.

      ##### <a name="repairing"></a> Repairing an ISA

      A repaired ISA is one where a portion of the balance is no longer eligible to be tax exempt. Goji will complete the verification, admin and tax submissions required for this process.

      The Originator platform will need to expose a mechanism to allow a portion of an ISA account (cash balance and investments) to be either transferred to a 'standard' account (preferable) or for part of the investments to be liquidated and the required cash returned to the investor.

      #### <a name="investor-death"></a> Death of Investor

      In the event of a death of an Investor, specific tasks needed to be carried out to process the tax liability of the ISA-wrapped investments.

      When you receive the death certificate, log onto the admin console, find the investor and click Actions > Create Workflow and then Death of Investor. You will be prompted to upload the death certificate and will then be shown a valuation of that ISA which should be sent to the executor. Do the same when you receive the Grant of Probate or Grant of Letters of Administration. The executor then has 3 choices:

      1) If the executor changes the account into his/her name change the appropriate name & address on your systems. This should also call the PUT /investors/{investorId} as shown above.

      2) If the ISA is to be transferred to the spouse then the spouse must create an ISA account on the P2P site. The P2P CSA should then inform Goji to transfer the tax wrapper to the spouse's account. Once this is complete the P2P CSA will be informed and any underlying loans in the ISA must be novated to the spouse.

      3) If the executor sells the loans then it is the normal process for a sale.

      ## <a name="reporting"></a> Reporting

      The Goji Platform will automatically generate and submit all reporting required by HMRC based on the data provided.

      ## <a name="transfer-in-integration"></a> Integrating the Transfer In pages

      The Goji Transfer In application exposes an ISA transfer in form intended for an investor to complete.

      It supports ISA transfers of the following types:

      - Cash
      - Stocks and Shares
      - Innovative Finance

      ## Embedding

      The Goji Transfer In application is a JavaScript component which can be integrated in a number of ways.
      For each possible way of integrating with the application, you will need to first obtain a one time security token.

      #### Obtaining the _uiData_ and the security token

      To obtain the application's asset URLs one time security token, make an authenticated request to the following URL:

         `https://api.gojip2p.net/investors/<investorId>/transferIn/uiData`

         The response will be structured like so:

         _(Please note that you should never hard-code the URLs returned since they are subject to change)_

                     {
                       "apiUrl": "https://api.gojip2p.net",
                       "hostedUrl": "https://api.gojip2p.net/transferIn?token=<oneTimeToken>&investorId=<investorId>",
                       "styleSrc": "https://goji-assets-domain/transfer-in/assets/goji-transfer-in-123456.css",
                       "scriptSrc": "https://goji-assets-domain/transfer-in/assets/goji-transfer-in-123456.js",
                       "investorId": "<investorId>",
                       "token": "<oneTimeToken>"
                     }

         With this data you are then able to bootstrap the application using any of the methods outlined below.

        If your front-end application uses Ember, using as an Ember Addon makes sense. Alternatively, the suggested approach would be to
        embed as a standalone JavaScript component on your existing pages - this enables full control over the application's styling.

      #### Application Arguments

      Four arguments are required for the application to function fully, these are described below:

      - `apiUrl`: [Specified in the _uiData_ response] The API URL the front-end application should use when interacting with the Goji service
      - `investorId`: [Specified in the _uiData_ response] The ID of the active investor
      - `token`: [Specified in the _uiData_ response] The security token used to authenticate the active investor's requests
      - `accountUrl`: The URL used when an investor chooses to return to their account page having successfully completed a transfer in request

      ### Using as a Standalone JavaScript Component

      To include the application in your existing page as a JavaScript component, you will need to do the following:

      1. In the body of your HTML include the following:

                 <div id="goji-application">
                   <div data-component="goji-transfer-in"
                        data-attrs='{ "apiUrl": "<uiData.apiUrl>", "accountUrl": "<platform-manage-account-url>",
                                      "investorId": "<uiData.investorId>", "token": "<uiData.token>" }'>
                   </div>
                 </div>

      1. Extract the JavaScript asset's URL from the request above and include it in your page.

         __Please note: The inclusion of the script import must be made after inclusion of the HTML in the previous step.__

         e.g `<script src="{{uiData.scriptSrc}}"></script>`

         Optionally do the same for the CSS file if you wish to have a basic layout.

         e.g `<link rel="stylesheet" href="{{uiData.styleSrc}}">`

      3. The component will then render when the document's body has fully loaded.

      #### Example HTML

      At a minimum, your HTML will look something like the [example.html](example.html) below:

      ![example.html](example-html.png)

      ### Using the Hosted Page

      The above response _uiData_ object includes a `hostedUrl` property which serves the transfer in form with some basic styling.
      This can be used as a link to the standalone transfers in page.

      ## <a name="hmac-authentication"></a> HMAC Authentication

      HMAC authentication is a mechanism where each HTTP request made by the client of the API is cryptographically signed.

      By signing the request with a secret key known only to the API client and the API itself it means that the authenticity of the request can be established.

      ## How to sign the request

      The request is signed by including three headers:

      `x-nonce` A unique string for every request. This will also be included in the signature string (see below) and is used to prevent replay attacks.

      `x-timestamp` Milliseconds since epoch. This will also be included in the signature string (see below) and is used to prevent replay attacks.

      `Authorization` In the format `<api-key>:<signature-string>` See below for how to build the `signature-string` .

      ## Building the signature string

      The following details are concatenated together:

          nonce + \n
          timestamp

      The string is then encrypted using `SHA256` using the private key.

      The result is then Base64 encoded to produce a string.

      The encrypted string is then UTF-8 URL encoded.

      ### Examples

      #### Simple GET request
      `GET https://api.gojip2p.com/user/session/valid`

      With a nonce = 67681625-d7f9-43e3-859a-25e634c203c2

      and timestamp = 1474982268271

      The signature string would be as follows:

          67681625-d7f9-43e3-859a-25e634c203c2
          1474982268271

      Assuming a secret key of `abcd1234`, this will produce a signature equal to: `q0AdIAm6SphhgN/VxjMiE9UEd3uZRca9gjJXQ5+dyNI=` which is then URL encoded to `q0AdIAm6SphhgN%2FVxjMiE9UEd3uZRca9gjJXQ5%2BdyNI%3D`

      ## <a name="api-error-codes"></a> API Error Codes

      Errors are returned in the following JSON format:

          {
             errors: [
               {
                 errorCode: "<ERROR-CODE>",
                 message: "<ERROR-MESSAGE>"
               }
             ]
           }


      The following error codes can be returned by the API:

      |Error Code | Message|
      |:---------|:---------|
      | DATE_TIME_IN_FUTURE | dateTimeSigned cannot be in the future |
      | INVALID_DATA | address.country cannot be null or empty |
      | INVALID_DATA | address.lineOne cannot be null or empty |
      | INVALID_DATA | address.postcode cannot be null or empty |
      | INVALID_DATA | address.postcode must be in valid UK format. e.g. (A9 9ZZ &#124; A99 9ZZ &#124; AB9 9ZZ &#124; AB99 9ZZ &#124; A9C 9ZZ &#124; AD9E 9ZZ) |
      | INVALID_DATA | address.townCity cannot be null or empty |
      | INVALID_DATA | clientId cannot be null or empty |
      | INVALID_DATA | dateOfBirth cannot be null or empty and must be in valid ISO format (YYYY-MM-DD) |
      | INVALID_DATA | declarationToken cannot be null or empty |
      | INVALID_DATA | emailAddress cannot be null or empty |
      | INVALID_DATA | emailAddress must be in valid format |
      | INVALID_DATA | firstName cannot be null or empty |
      | INVALID_DATA | lastName cannot be null or empty |
      | INVALID_DATA | nationalInsuranceNumber is in invalid format. Must be in NI number format e.g. (AB123456C) |
      | INVALID_DATA | nationalInsuranceNumber cannot be null or empty |
      | INVALID_DATA | termsAndConditionsToken cannot be null or empty |
      | UNDER_18 | Cannot create IF-ISA account for person under the age of 18 |
      | CLIENT_ID_TAKEN | This clientId has already been taken by another investor |
      | INVALID_DATA | The declarationToken supplied does not exist |
      | INVALID_DATA | The declarationToken supplied already been used by another investor |
      | TOKEN_EXPIRED | The declarationToken supplied was signed over 1 hour ago |
      | INVALID_NAME | First and last names must be longer than one character |
      | INVESTOR_ISA_EXISTS | Investor with this N.I number already has an IF-ISA account with this originator |
      | TAX_YEAR_ISA_EXISTS | Investor with this N.I number has previously created an IF-ISA within this tax year |
      | FAILS_RESIDENCY | address.postcode cannot be one from any of the Crown Dependencies (Guernsey GY, Jersey JE, Isle of Man IM) |
      | INVALID_ADDRESS | Address cannot be a PO Box |
      | INVALID_DATA | The termsAndConditionsToken supplied does not exist for this originator |
      | INVALID_DATA | The termsAndConditionsToken supplied already been used by another investor |
      | TOKEN_EXPIRED | The termsAndConditionsToken supplied was signed over 1 hour ago |
      | INVALID_COUNTRY | Country is invalid |
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | Amount is greater than Earmarked Amount. |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | ClientEarmarkedAmountId must be specified |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_DATA | Date of Earmark cannot be in the future |
      | INVALID_DATA | ClientInvestmentId must be specified |
      | INVALID_DATA | InvestmentType must be specified |
      | INVALID_DATA | OriginalAmount must be specified |
      | INVALID_DATA | OriginalAmount must be greater than zero |
      | INVALID_DATA | OriginalAmount must be in GBP |
      | INVALID_DATA | DateTimeOfInvestment must be specified |
      | INVALID_DATA | DateTimeOfInvestment must not be in the future |
      | INVALID_DATA | TermOfInvestment must be greater than zero |
      | INVALID_DATA | ClientReinvestmentId must be specified |
      | INVALID_DATA | Capital amount must be provided |
      | INVALID_DATA | Interest amount must be provided |
      | INVALID_DATA | DateTimeOfReinvestment must be provided |
      | INVALID_DATA | At least one new investment must be provided |
      | INVALID_DATA | Capital amount must be non negative |
      | INVALID_DATA | Interest amount must be non negative |
      | INVALID_DATA | DateTimeOfReinvestment cannot be in the future |
      | INVALID_DATA | Client repayment ID must be provided |
      | INVALID_DATA | DateTime of repayment must be provided |
      | INVALID_DATA | DateTime of repayment must not be in the future |
      | INVALID_DATA | Cannot provide repayment with zero capital and interest amount |
      | CAPITAL_AMOUNT_EXCEEDED | This reinvestment exceeds the remaining capital on the investment |
      | BALANCE_NOT_AVAILABLE | This transaction exceeds the current balance on the earmarked amount |
      | BALANCE_NOT_AVAILABLE | Cannot delete more than earmarked balance |
      | INVALID_INVESTMENT | Investment not allowed as investor is no longer eligible |
      | INVALID_SUBSCRIPTION | Subscription not allowed as investor is no longer eligible |
      | EXCEEDS_ANNUAL_SUBSCRIPTION | This transaction exceeds the current remaining subscription amount %s |
      | CASH_BALANCE_EXCEEDED | This transaction exceeds the current cash balance |
      | INVALID_DATA | The date of the repayment cannot be before the investment date |
      | CAPITAL_AMOUNT_EXCEEDED | The date of the repayment cannot be before the investment date |
      | INVALID_DATA | The clientTransactionId must be provided |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |
      | INVALID_DATA | transferAmount must be defined |
      | INVALID_DATA | repairAmount must be defined |
      | INVALID_DATA | The amounts submitted don't match the transfer history. Please resubmit with the correct amounts |

      ## <a name="api-error-codes-sign-terms"></a> Sign Terms Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | DATE_TIME_IN_FUTURE | dateTimeSigned cannot be in the future |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-sign-declaration"></a> Sign Declaration Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | DATE_TIME_IN_FUTURE | dateTimeSigned cannot be in the future |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-create-investor"></a> Create Investor Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | address.country cannot be null or empty |
      | INVALID_DATA | address.lineOne cannot be null or empty |
      | INVALID_DATA | address.postcode cannot be null or empty |
      | INVALID_DATA | address.postcode must be in valid UK format. e.g. (A9 9ZZ &#124; A99 9ZZ &#124; AB9 9ZZ &#124; AB99 9ZZ &#124; A9C 9ZZ &#124; AD9E 9ZZ) |
      | INVALID_DATA | address.townCity cannot be null or empty |
      | INVALID_DATA | clientId cannot be null or empty |
      | INVALID_DATA | dateOfBirth cannot be null or empty and must be in valid ISO format (YYYY-MM-DD) |
      | INVALID_DATA | declarationToken cannot be null or empty |
      | INVALID_DATA | emailAddress cannot be null or empty |
      | INVALID_DATA | emailAddress must be in valid format |
      | INVALID_DATA | firstName cannot be null or empty |
      | INVALID_DATA | lastName cannot be null or empty |
      | INVALID_DATA | nationalInsuranceNumber is in invalid format. Must be in NI number format e.g. (AB123456C) |
      | INVALID_DATA | nationalInsuranceNumber cannot be null or empty |
      | INVALID_DATA | termsAndConditionsToken cannot be null or empty |
      | UNDER_18 | Cannot create IF-ISA account for person under the age of 18 |
      | CLIENT_ID_TAKEN | This clientId has already been taken by another investor |
      | INVALID_DATA | The declarationToken supplied does not exist |
      | INVALID_DATA | The declarationToken supplied already been used by another investor |
      | TOKEN_EXPIRED | The declarationToken supplied was signed over 1 hour ago |
      | INVALID_NAME | First and last names must be longer than one character |
      | INVESTOR_ISA_EXISTS | Investor with this N.I number already has an IF-ISA account with this originator |
      | TAX_YEAR_ISA_EXISTS | Investor with this N.I number has previously created an IF-ISA within this tax year |
      | FAILS_RESIDENCY | address.postcode cannot be one from any of the Crown Dependencies (Guernsey GY, Jersey JE, Isle of Man IM) |
      | INVALID_ADDRESS | Address cannot be a PO Box |
      | INVALID_DATA | The termsAndConditionsToken supplied does not exist for this originator |
      | INVALID_DATA | The termsAndConditionsToken supplied already been used by another investor |
      | TOKEN_EXPIRED | The termsAndConditionsToken supplied was signed over 1 hour ago |
      | INVALID_COUNTRY | Country is invalid |

      ## <a name="api-error-codes-validate-investor"></a> Validate Investor Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | address.country cannot be null or empty |
      | INVALID_DATA | address.lineOne cannot be null or empty |
      | INVALID_DATA | address.postcode cannot be null or empty |
      | INVALID_DATA | address.postcode must be in valid UK format. e.g. (A9 9ZZ &#124; A99 9ZZ &#124; AB9 9ZZ &#124; AB99 9ZZ &#124; A9C 9ZZ &#124; AD9E 9ZZ) |
      | INVALID_DATA | address.townCity cannot be null or empty |
      | INVALID_DATA | dateOfBirth cannot be null or empty and must be in valid ISO format (YYYY-MM-DD) |
      | INVALID_DATA | nationalInsuranceNumber is in invalid format. Must be in NI number format e.g. (AB123456C) |
      | INVALID_DATA | nationalInsuranceNumber cannot be null or empty |
      | UNDER_18 | Cannot create IF-ISA account for person under the age of 18 |
      | INVESTOR_ISA_EXISTS | Investor with this N.I number already has an IF-ISA account with this originator |
      | TAX_YEAR_ISA_EXISTS | Investor with this N.I number has previously created an IF-ISA within this tax year |
      | FAILS_RESIDENCY | address.postcode cannot be one from any of the Crown Dependencies (Guernsey GY, Jersey JE, Isle of Man IM) |
      | INVALID_ADDRESS | Address cannot be a PO Box |
      | INVALID_COUNTRY | Country is invalid |

      ## <a name="api-error-codes-create-investment"></a> Create Investment Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | Amount is greater than Earmarked Amount. |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_DATA | ClientInvestmentId must be specified |
      | INVALID_DATA | InvestmentType must be specified |
      | INVALID_DATA | OriginalAmount must be specified |
      | INVALID_DATA | OriginalAmount must be greater than zero |
      | INVALID_DATA | OriginalAmount must be in GBP |
      | INVALID_DATA | DateTimeOfInvestment must be specified |
      | INVALID_DATA | DateTimeOfInvestment must not be in the future |
      | INVALID_DATA | TermOfInvestment must be greater than zero |
      | INVALID_INVESTMENT | Investment not allowed as investor is no longer eligible |
      | CASH_BALANCE_EXCEEDED | This transaction exceeds the current cash balance |
      | INVALID_DATA | The clientTransactionId must be provided |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-create-repayment"></a> Create Repayment Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | ClientReinvestmentId must be specified |
      | INVALID_DATA | Capital amount must be provided |
      | INVALID_DATA | Interest amount must be provided |
      | INVALID_DATA | Capital amount must be non negative |
      | INVALID_DATA | Interest amount must be non negative |
      | INVALID_DATA | Client repayment ID must be provided |
      | INVALID_DATA | DateTime of repayment must be provided |
      | INVALID_DATA | DateTime of repayment must not be in the future |
      | INVALID_DATA | Cannot provide repayment with zero capital and interest amount |
      | INVALID_DATA | The date of the repayment cannot be before the investment date |
      | CAPITAL_AMOUNT_EXCEEDED | The date of the repayment cannot be before the investment date |
      | INVALID_DATA | The clientTransactionId must be provided |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-create-reinvestment"></a> Create Reinvestment Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | Amount is greater than Earmarked Amount. |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_DATA | Date of Earmark cannot be in the future |
      | INVALID_DATA | ClientInvestmentId must be specified |
      | INVALID_DATA | InvestmentType must be specified |
      | INVALID_DATA | OriginalAmount must be specified |
      | INVALID_DATA | OriginalAmount must be greater than zero |
      | INVALID_DATA | OriginalAmount must be in GBP |
      | INVALID_DATA | DateTimeOfInvestment must be specified |
      | INVALID_DATA | DateTimeOfInvestment must not be in the future |
      | INVALID_DATA | TermOfInvestment must be greater than zero |
      | INVALID_DATA | ClientReinvestmentId must be specified |
      | INVALID_DATA | Capital amount must be provided |
      | INVALID_DATA | Interest amount must be provided |
      | INVALID_DATA | DateTimeOfReinvestment must be provided |
      | INVALID_DATA | At least one new investment must be provided |
      | INVALID_DATA | Capital amount must be non negative |
      | INVALID_DATA | Interest amount must be non negative |
      | INVALID_DATA | DateTimeOfReinvestment cannot be in the future |
      | INVALID_DATA | Client repayment ID must be provided |
      | INVALID_DATA | DateTime of repayment must be provided |
      | INVALID_DATA | DateTime of repayment must not be in the future |
      | INVALID_DATA | Cannot provide repayment with zero capital and interest amount |
      | CAPITAL_AMOUNT_EXCEEDED | This reinvestment exceeds the remaining capital on the investment |
      | INVALID_INVESTMENT | Investment not allowed as investor is no longer eligible |
      | INVALID_DATA | The date of the repayment cannot be before the investment date |
      | INVALID_DATA | The clientTransactionId must be provided |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-cash-transaction"></a> Cash Transaction Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_SUBSCRIPTION | Subscription not allowed as investor is no longer eligible |
      | EXCEEDS_ANNUAL_SUBSCRIPTION | This transaction exceeds the current remaining subscription amount %s |
      | CASH_BALANCE_EXCEEDED | This transaction exceeds the current cash balance |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-earmarked-amount"></a> Earmarked Amount Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | ClientEarmarkedAmountId must be specified |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_DATA | Date of Earmark cannot be in the future |
      | INVALID_DATA | ClientInvestmentId must be specified |
      | CASH_BALANCE_EXCEEDED | This transaction exceeds the current cash balance |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-invest-earmarked-amount"></a> Invest Earmarked Amount Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | Amount must be greater than zero. |
      | INVALID_DATA | Amount cannot be zero |
      | INVALID_DATA | Amount is greater than Earmarked Amount. |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | ClientEarmarkedAmountId must be specified |
      | INVALID_DATA | Amount must be in GBP |
      | INVALID_DATA | ClientInvestmentId must be specified |
      | INVALID_DATA | InvestmentType must be specified |
      | INVALID_DATA | OriginalAmount must be specified |
      | INVALID_DATA | OriginalAmount must be greater than zero |
      | INVALID_DATA | OriginalAmount must be in GBP |
      | INVALID_DATA | DateTimeOfInvestment must be specified |
      | INVALID_DATA | DateTimeOfInvestment must not be in the future |
      | INVALID_DATA | TermOfInvestment must be greater than zero |
      | BALANCE_NOT_AVAILABLE | This transaction exceeds the current balance on the earmarked amount |
      | INVALID_INVESTMENT | Investment not allowed as investor is no longer eligible |
      | INVALID_DATA | The clientTransactionId must be provided |
      | INVALID_DATA | The clientTransactionId=%s has already been used |
      | INVALID_DATA | Cannot parse date. Please ensure it is in the correct format eg 2015-01-01T09:00:00Z |

      ## <a name="api-error-codes-transfer-cash"></a> Transfer Cash Error Codes

      |Error Code | Message|
      |:---------|:---------|
      | INVALID_DATA | transferAmount must be defined |
      | INVALID_DATA | repairAmount must be defined |
      | INVALID_DATA | The amounts submitted don't match the transfer history. Please resubmit with the correct amounts |

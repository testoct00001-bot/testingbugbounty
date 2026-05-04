================================================================================
  DEEP RECON REPORT: https://test-payment.backend-capital.com/
  Date: 2026-05-05 00:41 GMT+8
  Target: Capital.com Payment REST API (Test Environment)
  Company: Capital Com (UK) Limited
  Contact: support@capital.com
================================================================================

## INFRASTRUCTURE
  WAF:           Imperva/Incapsula
  CDN:           Imperva (d7ihayo.ng.impervadns.net)
  DNS:           AWS Route53 (awsdns-*.com/net/org/co.uk)
  SSL:           Wildcard *.backend-capital.com (Sectigo, expires Jun 2026)
  Load Balancer: AWS ALB (AWSALB cookies)
  Email:         AWS SES (eu-west-1)
  App Stack:     Java/Spring Boot (Jakarta EE)
  Package:       com.expcapital.payment
  Bug Bounty:    Active on Intigriti (up to $15k RCE Tier 1)

## CRITICAL FINDINGS

### 1. FULL OPENAPI SPEC EXPOSED (NO AUTH)
  Endpoint: /payment/docs/
  Returns: Complete OpenAPI 3.0.1 spec (435KB)
  Stats: 329 endpoints, 232 paths, 255 schemas
  Impact: Complete attack surface map exposed to any attacker

### 2. UNAUTHENTICATED DATA ENDPOINTS
  [NO AUTH] GET /payment/gateway/v1/vertupay/data/banks
    Response: {"VN":["ACB","BIDV","DAB"]} (Vietnamese bank codes)
  
  [NO AUTH] GET /payment/gateway/v1/options/commission?license=CYSEC&country=GB
    Response: Full commission data for VISA, Mastercard, etc.
    Leaks: min/max deposit amounts, commission structures, payment options
  
  [NO AUTH] GET /payment/gateway/v1/options/info?countryOfResidence=GB&license=CYSEC
    Response: Payment options info (empty data with these params)

### 3. JAVA CLASS/SIGNATURE LEAKAGE IN ERROR MESSAGES
  Error messages reveal full Java class paths and method signatures:
  - com.expcapital.payment.gateway.rest.PaymentOptionsRestController
  - com.expcapital.payment.gateway.rest.PaymentGatewayRestController.depositRefV1()
  - com.expcapital.payment.gateway.rest.PaymentTransfermateRestController.saveBank()
  - com.expcapital.payment.model.PaymentPageReference
  - com.expcapital.payment.client.dto.License (enum)
  - com.expcapital.payment.rest.dto.VoidResponse

### 4. VALID LICENSE ENUM VALUES
  CYSEC, FCA, ASIC, NBRB (confirmed working)
  Invalid: SCB, CIMA (rejected with type conversion error)

### 5. VALID TENANT ENUM VALUES
  CAPITAL_COM, CURRENCY_COM, CAPITAL, CURRENCY, DEFAULT, TEST, PROD, STAGING

### 6. AUTH MECHANISM IDENTIFIED
  Header: Session-Token (required for gateway endpoints)
  Error: "Required request header 'Session-Token' for method parameter type String is not present"

### 7. WEBHOOK ENDPOINTS RESPONDING WITHOUT AUTH
  [401] POST /equalsmoney/deposit/notification
  [500] POST /equalsmoney/withdrawal/notification
  [500] POST /equalsmoney/withdrawal/cancel
  [400] POST /myfatoorah/notification/{merchantId}
  [400] POST /oneroadpayments/notification
  [415] POST /capital/deposit3dsV2 (and 10 other /capital/* 3DS endpoints)
  All other PSP notification endpoints blocked by WAF (403)

### 8. MULTIPLE OPENAPI GROUPS
  /payment/docs/        => 200 (full spec - "payment" group)
  /payment/docs/payment => 200 (same spec)
  /payment/docs/gateway => 500 ("No OpenAPI resource found for group: gateway")
  /payment/docs/internal=> 500 ("No OpenAPI resource found for group: internal")
  /payment/docs/admin   => 500 ("No OpenAPI resource found for group: admin")

## FULL ENDPOINT MAP (329 endpoints)

GET     /
GET     /admin/v1/bank/details  // Get BankPaymentOptionDetailsResponse by criteria  [Params: query:license(string), query:test(boolean), query:currency(string), query:country(string)]
PUT     /admin/v1/bank/details  // Update merchant details data  [Body: MerchantBankDetailsRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/bank/details  // Add merchant details  [Body: MerchantBankDetailsRequest]  [Params: header:modifiedBy(string)]
GET     /admin/v1/bank/details/routing/{routingRuleId}  // Get BankPaymentOptionDetailsResponse by routing rule id  [Params: path:routingRuleId(integer)]
GET     /admin/v1/bank/details/{paymentConfigNodeId}  // Get BankPaymentOptionDetailsResponse by payment option config id  [Params: path:paymentConfigNodeId(integer)]
DELETE  /admin/v1/bank/details/{processingBank}/{bankAccount}  // Delete merchant details  [Params: header:modifiedBy(string), path:processingBank(string), path:bankAccount(string)]
POST    /admin/v1/blacklist  // Add new blacklisted value  [Body: BankBlackListDto]  [Params: header:modifiedBy(string)]
GET     /admin/v1/blacklist/info  // Get blacklist info by filters  [Params: query:license(), query:type()]
POST    /admin/v1/blacklist/validate  // Validate blacklist bin  [Body: ValidateBlacklistRequest]
POST    /admin/v1/blacklist/{id}  // Update blacklisted value  [Body: BankBlackListDto]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/blacklist/{id}  // Delete blacklisted value  [Params: header:modifiedBy(string), path:id(integer)]
GET     /admin/v1/commission  // Search Payment Option commission configs  [Params: header:modifiedBy(string), query:paymentOptions(array), query:transactionTypes(array), query:licenses(array)]
PUT     /admin/v1/commission  // Update Payment Option commission config  [Body: AdminPaymentOptionCommissionConfigRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/commission  // Add new Payment Option commission config  [Body: AdminPaymentOptionCommissionConfigRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/commission/{id}  // Delete Payment Option commission config  [Params: header:modifiedBy(string), path:id(integer)]
GET     /admin/v1/configNodes  // Get config
POST    /admin/v1/configNodes  // Add new config  [Body: PaymentConfigNode]
GET     /admin/v1/configNodes/{paymentConfigNodeId}  // Get config node  [Params: path:paymentConfigNodeId(integer)]
DELETE  /admin/v1/configNodes/{paymentConfigNodeId}  // Delete config node  [Params: path:paymentConfigNodeId(integer)]
GET     /admin/v1/currency/info  // Get currency info  [Params: header:modifiedBy(string)]
GET     /admin/v1/currency/local  // Get local currencies  [Params: query:currency(array)]
POST    /admin/v1/currency/local  // Add or update local currency  [Body: AdminLocalCurrencyAddOrUpdateRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/currency/local/{country}  // Delete local currency  [Params: path:country(string), header:modifiedBy(string)]
GET     /admin/v1/currency/markup  // Get FX markup for currency  [Params: query:currency(array)]
POST    /admin/v1/currency/markup  // Add FX markup  [Body: AdminFxMarkupUpdateRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/currency/markup/{id}  // Update FX markup  [Body: AdminFxMarkupUpdateRequest]  [Params: path:id(integer), header:modifiedBy(string)]
DELETE  /admin/v1/currency/markup/{id}  // Delete FX markup  [Params: path:id(integer), header:modifiedBy(string)]
GET     /admin/v1/options  // Get payment options info  [Params: query:license(), query:test(boolean), query:countryOfResidence(string), query:binCountry(string)]
POST    /admin/v1/paymentOrder/complete  // Complete the order in Payments and the transaction in UMS  [Body: UpdatePaymentOrderRequest]  [Params: header:modifiedBy(string)]
GET     /admin/v1/paymentSystems  // Get payment systems info
GET     /admin/v1/paymentoptionconfig  // Get PaymentOptionsConfig by license, currency, paymentOptions, externalSystems and test flag  [Params: query:license(string), query:currencies(array), query:paymentOptions(array), query:externalSystems(array)]
POST    /admin/v1/paymentoptionconfig  // Add new PaymentOptionsConfig  [Body: PaymentOptionsConfig]
GET     /admin/v1/paymentoptionconfig/apmMethodsList  // Get a list of available apm methods
POST    /admin/v1/paymentoptionconfig/{paymentOptionConfigId}  // Update PaymentOptionsConfig  [Body: PaymentOptionsConfig]
DELETE  /admin/v1/paymentoptionconfig/{paymentOptionConfigId}  // Delete PaymentOptionsConfig  [Params: path:paymentOptionConfigId(integer)]
GET     /admin/v1/paymentoptionordering  // Get PaymentOptionOrderingRules by license, currencies and countries  [Params: query:license(), query:countries(array), query:currencies(array)]
POST    /admin/v1/paymentoptionordering  // Add new PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
GET     /admin/v1/paymentoptionordering/default  // Get details of default PaymentOptionOrderingRule for given license  [Params: query:license()]
POST    /admin/v1/paymentoptionordering/validate  // Validate PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
GET     /admin/v1/paymentoptionordering/{id}  // Get details of PaymentOptionOrderingRule  [Params: path:id(integer)]
PUT     /admin/v1/paymentoptionordering/{id}  // Update PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
DELETE  /admin/v1/paymentoptionordering/{id}  // Delete PaymentOptionOrderingRule  [Params: path:id(integer)]
GET     /admin/v1/paymentoptionrouting  // Get PaymentOptionRoutingRules by license, currency, binCountry and test flag  [Params: query:license(), query:paymentOption(), query:currencies(array), query:countries(array)]
POST    /admin/v1/paymentoptionrouting  // Add new PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
DELETE  /admin/v1/paymentoptionrouting/card/deleteRules  // Delete all non default bank card cascades, available only for test env  [Body: DeleteAllCardRulesRequest]
POST    /admin/v1/paymentoptionrouting/validate  // Validate PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
POST    /admin/v1/paymentoptionrouting/{ruleId}  // Update PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
DELETE  /admin/v1/paymentoptionrouting/{ruleId}  // Delete PaymentOptionRoutingRule  [Params: path:ruleId(integer)]
GET     /admin/v1/properties  // Get all properties
POST    /admin/v1/properties  // Add properties  [Body: AddPropertiesRequest]
GET     /admin/v1/properties/name/{name}  // Get properties by name containing  [Params: path:name(string)]
GET     /admin/v1/properties/{propertiesId}  // Get properties by id  [Params: path:propertiesId(integer)]
POST    /admin/v1/properties/{propertiesId}  // Update properties  [Body: UpdatePropertiesRequest]  [Params: path:propertiesId(integer)]
DELETE  /admin/v1/properties/{propertiesId}  // Delete properties by id  [Params: path:propertiesId(integer)]
GET     /admin/v1/setting  // Get Settings  [Params: header:modifiedBy(string)]
GET     /admin/v1/setting/global  // Get Settings Global  [Params: header:modifiedBy(string), query:keys(array), query:licenses(array), query:countries(array)]
PUT     /admin/v1/setting/global  // Update Settings Global  [Body: AdminSettingGlobalRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/setting/global  // Add new Settings Global  [Body: AdminSettingGlobalRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/setting/global/{id}  // Delete Setting Global  [Params: header:modifiedBy(string), path:id(integer)]
POST    /azupay/notification
POST    /azupay/v2/notification
POST    /azupay/withdrawal/notification
POST    /bancasella/notification
POST    /bankingcircle/accountverification/notification/{operatingEntity}  [Params: path:operatingEntity(string), header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /bankingcircle/notification  [Params: header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /bankingcircle/notification/{operatingEntity}  [Params: path:operatingEntity(string), header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /capital/assembly3dsV2  // Complete deposit via credit card with 3DS  [Params: query:assembledBrowserDetails(), query:ACSUnavailable(boolean), query:sid(string), query:tenant()]
POST    /capital/auth3ds  // Handles 3DS 2.0 authentication notification  [Params: query:request()]
POST    /capital/auth3ds/nested  // Handles 3DS 2.0 authentication notification for nested iframes  [Params: query:request()]
POST    /capital/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:cres(string)]
POST    /capital/complete3ds  // Complete deposit via credit card with 3DS  [Body: MultiValueMapStringString]
DELETE  /capital/deleteCard  // Delete credit card  [Body: DeleteCardRequest]
GET     /capital/deposit  // Get deposit URL. Can return URL to external iframe  [Params: header:App-Version(string), query:sid(string)]
POST    /capital/deposit  // Deposit via credit card  [Params: header:App-Version(string), query:depositData()]
POST    /capital/deposit3dsV2  // Handles 3DS 2.0 deposit notification  [Params: query:tenant(), query:sid(string), query:PaReq(string)]
POST    /capital/ecommpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:pares(string), query:sid(string)]
GET     /capital/mpi  // Complete mpi via credit card with 3DS  [Params: query:sid(string), query:tenant()]
POST    /capital/mpi  // Complete mpi via credit card with 3DS  [Params: query:sid(string), query:tenant()]
POST    /capital/networkint/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:cres(string)]
GET     /capital/praxis/redirect/CAPITAL_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/redirect/CAPITAL_COM  [Params: query:status(string), query:app(string)]
GET     /capital/praxis/redirect/CURRENCY_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/redirect/CURRENCY_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/validate  [Body: PraxisValidationRequest]
GET     /capital/submit3ds  // Handles 3DS deposit redirects  [Params: query:tenant(), query:sid(string)]
POST    /capital/submit3ds  // Handles 3DS deposit notification  [Params: query:tenant(), query:sid(string), query:PaReq(string)]
GET     /capital/webpay/challenge3ds  // Handles 3DS 2.0 challenge notification with extra parameters  [Params: query:request()]
POST    /capital/webpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:sid(string), query:cres(string)]
POST    /capital/worldpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:MD(string)]
POST    /cardpay/notification/htp  [Params: header:Signature(string)]
POST    /checkout/notification  [Params: query:license(), query:operatingEntity()]
POST    /clearbank/notification  [Params: header:DigitalSignature(string)]
POST    /credorax/apm/deposit/notification
POST    /credorax/deposit/notification
POST    /d24/notification/deposit
POST    /d24/notification/refund
POST    /dbs/deposit/notification
POST    /dbs/withdrawal/notification
POST    /ecommpay/notification
POST    /ecommpay/notification/redirect
POST    /ecommpay/notification/token
POST    /equalsmoney/deposit/notification
POST    /equalsmoney/withdrawal/cancel
POST    /equalsmoney/withdrawal/notification
POST    /erip/deposits/notification  [Params: query:XML(string)]
GET     /exactly/notification  // Handle notification object from Exactly
POST    /exactly/notification  // Handle notification object from Exactly
POST    /fuze/notification
GET     /gateway/v1/beneficiaryCustomer/validate  // Validate Beneficiary Customer  [Params: header:Session-Token(string), query:beneficiaryCustomer(string)]
POST    /gateway/v1/card/ani  // Validate Card Number  [Body: CardAccountNumberInquiryRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/commission  // Get commission  [Params: header:Session-Token(string), query:accountId(integer), query:paymentType(), query:paymentOption()]
GET     /gateway/v1/commission/card  // Get commission for card  [Params: header:Session-Token(string), query:accountId(integer), query:paymentType(), query:bin(string)]
GET     /gateway/v1/crypto/deposit/details  // Get Crypto deposit details  [Params: header:Session-Token(string), query:request()]
GET     /gateway/v1/crypto/deposit/terms  // Get a link to the terms  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/crypto/deposit/terms  // Save crypto deposit terms  [Body: CryptoDepositSaveTermsRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/crypto/virtualAssetProviders  // Get a list of virtual asset providers  [Params: header:Session-Token(string), query:accountId(integer), query:providerName(string)]
GET     /gateway/v1/crypto/whitelist/address  // Get user's whitelisted addresses on PSP side  [Params: header:Session-Token(string), query:accountId(integer), query:chain(string), query:paymentType()]
POST    /gateway/v1/crypto/whitelist/address  // Whitelist an address on PSP side  [Body: AddCryptoWhitelistAddressRequest]  [Params: header:Session-Token(string)]
DELETE  /gateway/v1/crypto/whitelist/address  // Deactivate a user's whitelisted address  [Params: header:Session-Token(string), query:id(integer)]
POST    /gateway/v1/crypto/withdrawal  // Withdrawal via crypto currency  [Body: CryptoWithdrawalRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/crypto/withdrawal/details  // Get Crypto withdrawal details  [Params: header:Session-Token(string), query:request()]
GET     /gateway/v1/dcc/rate  // Get DCC rate  [Params: header:Session-Token(string), query:accountId(integer), query:type(), query:options(array)]
GET     /gateway/v1/dcc/rate/card  // Get DCC rate for card  [Params: header:Session-Token(string), query:accountId(integer), query:type(), query:bin(string)]
GET     /gateway/v1/decline/cardpay  [Params: query:params()]
PUT     /gateway/v1/decline/cardpay  [Params: query:params()]
POST    /gateway/v1/decline/cardpay  [Params: query:params()]
DELETE  /gateway/v1/decline/cardpay  [Params: query:params()]
PATCH   /gateway/v1/decline/cardpay  [Params: query:params()]
GET     /gateway/v1/decline/checkout  [Params: query:cko-session-id(string), query:license(), query:operatingEntity()]
GET     /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
PUT     /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
POST    /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
DELETE  /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
PATCH   /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
GET     /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
PUT     /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
POST    /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
DELETE  /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
PATCH   /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
GET     /gateway/v1/deposit  // Get deposit URL. Can return URL to external iframe. email is used for skrill deposit to override skrill account name  [Params: header:Session-Token(string), query:accountId(integer), query:amount(number), query:option()]
POST    /gateway/v1/deposit/card  // Deposit via credit card  [Body: DepositCardRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/deposit/card/bin/check  // Check credit card BIN  [Body: CardBinCheckRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
PUT     /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
POST    /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
DELETE  /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
PATCH   /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
GET     /gateway/v1/deposit/stripe  [Params: query:sid(string)]
PUT     /gateway/v1/deposit/stripe  [Params: query:sid(string)]
POST    /gateway/v1/deposit/stripe  [Params: query:sid(string)]
DELETE  /gateway/v1/deposit/stripe  [Params: query:sid(string)]
PATCH   /gateway/v1/deposit/stripe  [Params: query:sid(string)]
GET     /gateway/v1/iban/validate  // Validate iban  [Params: header:Session-Token(string), query:iban(string)]
POST    /gateway/v1/lean/createPaymentIntent  // Create payment Intent for Lean deposit  [Body: LeanCommunicationRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/lean/createWithdrawalPaymentMethod  // Create Lean Payment Method for Withdrawals  [Body: LeanCreateWithdrawalPaymentMethodRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/lean/data/getDepositDetails/{currency}  // Deposit data for Lean payment  [Params: header:Session-Token(string), path:currency(string)]
POST    /gateway/v1/lean/unlinkAccount  // Unlink account  [Body: LeanCommunicationRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/options/commission  // Get payment options commissions info  [Params: query:license(), query:country(string)]
GET     /gateway/v1/options/info  // Get payment available options  [Params: query:countryOfResidence(string), query:license()]
GET     /gateway/v1/payment/truelayer  [Params: query:payment_id(string), query:error(string)]
GET     /gateway/v1/paymentMethods  // Get payment methods  [Params: header:Session-Token(string), header:App-Version(string), query:option(), query:accountId(integer)]
POST    /gateway/v1/paymentMethods/card  // Add new credit card payment method  [Body: CardPaymentMethodAddRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/paymentMethods/unverified  // Get unverified payment methods  [Params: header:Session-Token(string), query:accountId(integer)]
PUT     /gateway/v1/paymentMethods/{paymentMethodId}  // Edit payment method  [Body: PaymentMethodEditRequest]  [Params: header:Session-Token(string), path:paymentMethodId(integer)]
DELETE  /gateway/v1/paymentMethods/{paymentMethodId}  // Delete payment method  [Params: header:Session-Token(string), path:paymentMethodId(integer)]
GET     /gateway/v1/paymentPage/info  // Get payment options info  [Params: header:Session-Token(string), query:accountId(integer)]
GET     /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
PUT     /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
DELETE  /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
PATCH   /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
GET     /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
PUT     /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
DELETE  /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
PATCH   /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/exactly  [Params: query:params()]
POST    /gateway/v1/redirect/mercuryo  [Params: query:params()]
GET     /gateway/v1/result/azupay  [Params: query:params(), query:sid(string), query:license(), query:paymentRequestId(string)]
GET     /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
PUT     /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
POST    /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
DELETE  /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
PATCH   /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
GET     /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
PUT     /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
POST    /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
DELETE  /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
PATCH   /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
GET     /gateway/v1/result/stripe  [Params: query:app(boolean), query:sid(string), query:license(), query:payment_intent(string)]
GET     /gateway/v1/result/volt  [Params: query:volt(string), query:volt-signature(string), query:volt-timestamp(string)]
GET     /gateway/v1/skrill/cancel  [Params: query:params(), query:sid(string), query:license()]
GET     /gateway/v1/threedsecure/totalprocessing  [Params: query:params(), query:currency(string), query:sid(string), query:license()]
GET     /gateway/v1/transactx/result
PUT     /gateway/v1/transactx/result
POST    /gateway/v1/transactx/result
DELETE  /gateway/v1/transactx/result
PATCH   /gateway/v1/transactx/result
GET     /gateway/v1/transfermate/data/banks  // List of available banks  [Params: header:Session-Token(string)]
POST    /gateway/v1/transfermate/data/saveBank  // Save bank code and bank name  [Body: TransferMateSaveBankRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/truelayer/data/auth  [Params: query:code(string), query:state(string), query:error(string)]
GET     /gateway/v1/truelayer/data/linkedAccounts  // List of selected bank accounts  [Params: header:sid(string)]
POST    /gateway/v1/truelayer/data/saveAccount  // Save bank account  [Body: TrueLayerSaveAccountRequest]
GET     /gateway/v1/vertupay/data/banks  // List of available bank codes
GET     /gateway/v1/virtualIban  // Get virtual iban  [Params: header:Session-Token(string), query:accountId(integer), query:bankCode(string), query:currency(string)]
POST    /gateway/v1/virtualIban  // Create virtual iban  [Body: CreateVirtualIbanRequestDto]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/apple/deposit/card  // Make Apple Pay deposit  [Body: ApplePayDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wallet/apple/deposit/details  // Get Apple Pay deposit details  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wallet/apple/deposit/session  // Create Apple Pay payment session  [Body: ApplePayCreatePaymentSessionRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/erip/deposit/session  // Create ERIP payment session  [Body: EripCreatePaymentSessionRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/google/deposit/card  // Make Google Pay deposit  [Body: GooglePayDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wallet/google/deposit/details  // Get Google Pay deposit details  [Params: header:Session-Token(string), query:accountId(integer)]
GET     /gateway/v1/wallet/paypal/checkout/token  // Get Paypal token for client  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wallet/paypal/deposit  // Make PayPal deposit  [Body: PayPalDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wire/bankDepositDetails  // Get bank details for deposit  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wire/paymentMethods/am  // Create payment method for Armenia Bank Account  [Body: WirePaymentMethodAddAmRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/au  // Create payment method for Australia Account  [Body: WirePaymentMethodAddAuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/generic  // Create payment method for Bank Account  [Body: WirePaymentMethodAddGenericRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/iban  // Create payment method for International Bank Account  [Body: WirePaymentMethodAddIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/in  // Create payment method for India Bank Account  [Body: WirePaymentMethodAddInRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/ru  // Create payment method for RUB Russian Bank Account  [Body: WirePaymentMethodAddRuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/ru/foreign  // Create payment method for non RUB Russian Bank Account  [Body: WirePaymentMethodAddRuForeignRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/uk  // Create payment method for British or Irish Bank Account  [Body: WirePaymentMethodAddUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal  // Withdrawal  [Body: WireWithdrawalGenericRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/am  // Withdrawal to Armenia Bank Account  [Body: WireWithdrawalAmRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/iban  // Withdrawal to International Bank Account  [Body: WireWithdrawalIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/in  // Withdrawal to India Bank Account  [Body: WireWithdrawalInRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/ru  // Withdrawal to RUB Russian Bank Account  [Body: WireWithdrawalRuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/ru/foreign  // Withdrawal to non RUB Russian Bank Account  [Body: WireWithdrawalRuForeignRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/sortCode  // Withdrawal to British or Irish Bank Account  [Body: WireWithdrawalUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/swift  // Withdrawal to International Bank Account  [Body: WireWithdrawalIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/uk  // Withdrawal to British or Irish Bank Account  [Body: WireWithdrawalUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/withdrawal  // Withdrawal via specific payment method  [Body: WithdrawalRequest]  [Params: header:Session-Token(string), header:App-Version(string)]
POST    /imbank/notification
GET     /internal/v1/bankDepositDetails  // Get bank details for deposit  [Params: query:license(), query:operatingEntity(), query:currency(string), query:country(string)]
GET     /internal/v1/healthcheck/ping  // Check rest service availability
GET     /internal/v1/isProcessingBankSupportsReturn  [Params: query:license(), query:processingBank(string), query:paymentBeta(boolean)]
GET     /internal/v1/operation/info  [Params: header:modifiedBy(string)]
POST    /internal/v1/operation/request  [Body: PaymentOrderRequestNewRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/operation/schema  [Params: query:request(), header:modifiedBy(string)]
GET     /internal/v1/operation/status  [Params: query:request(), header:modifiedBy(string)]
POST    /internal/v1/order/complete  [Body: PaymentOrderStatusChangeRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/order/info  [Params: header:modifiedBy(string)]
POST    /internal/v1/order/request  [Body: PaymentOrderRequestNewRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/order/schema  [Params: query:request(), header:modifiedBy(string)]
GET     /internal/v1/order/status  [Params: query:request(), header:modifiedBy(string)]
POST    /internal/v1/paymentMethod/block  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/delete  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/restore  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/unblock  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/withdrawal  [Body: CreateWithdrawalRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/withdrawal/bankProviders  [Params: query:license(), query:operatingEntity(), query:currency(string), query:processingCurrency(string)]
POST    /internal/v1/withdrawal/split  [Body: PaymentWithdrawalSplitRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /itez/notification  [Params: header:X-Payload-Digest(string)]
PUT     /itez/notification  [Params: header:X-Payload-Digest(string)]
POST    /itez/notification  [Params: header:X-Payload-Digest(string)]
DELETE  /itez/notification  [Params: header:X-Payload-Digest(string)]
PATCH   /itez/notification  [Params: header:X-Payload-Digest(string)]
POST    /klarpay/deposit/notification
POST    /lean/notification  [Params: header:lean-signature(string)]
GET     /mercuryo/notification  // Handle notification object from Mercuryo  [Params: query:data(string), query:sign(string), query:license(), query:currency(string)]
POST    /mercuryo/notification  // Handle notification object from Mercuryo  [Params: query:data(string), query:sign(string), query:license(), query:currency(string)]
GET     /mercuryo/payout/notification  // Handle payout notification object from Mercuryo  [Params: query:license(), query:currency(string)]
POST    /mercuryo/payout/notification  // Handle payout notification object from Mercuryo  [Params: query:license(), query:currency(string)]
POST    /modulr/deposit/notification
POST    /modulr/withdrawal/notification
POST    /monetix/notification
POST    /monetix/notification/redirect
POST    /monetix/notification/token
GET     /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
PUT     /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
POST    /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
DELETE  /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
PATCH   /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
POST    /nbd/withdrawal/notification
GET     /neteller/notification
PUT     /neteller/notification
POST    /neteller/notification
DELETE  /neteller/notification
PATCH   /neteller/notification
POST    /networkint/notification
GET     /oneroadpayments/notification  [Params: query:success(string), query:merchantID(string), query:orderID(string), query:clientID(string)]
POST    /oneroadpayments/notification  [Params: query:success(string), query:merchantID(string), query:orderID(string), query:clientID(string)]
POST    /openpayd/withdrawal/notification  [Params: header:Signature(string)]
POST    /pawapay/deposit/notification
POST    /pawapay/refund/notification
POST    /pawapay/withdrawal/notification
GET     /paycom/notification/{license}  [Params: path:license()]
PUT     /paycom/notification/{license}  [Params: path:license()]
POST    /paycom/notification/{license}  [Params: path:license()]
DELETE  /paycom/notification/{license}  [Params: path:license()]
PATCH   /paycom/notification/{license}  [Params: path:license()]
POST    /paymob/notification  [Params: query:hmac(string)]
POST    /paypal/braintree/notification  // Handle notification object from Braintree Gateway  [Params: query:license()]
GET     /paypal/payments/notification  // Handle notification object from Paypal  [Params: query:license()]
POST    /paypal/payments/notification  // Handle notification object from Paypal  [Params: query:license()]
POST    /payretailers/deposit/notification
POST    /payretailers/withdrawal/notification
POST    /poli/notification  [Params: query:Token(string)]
POST    /praxis/notification/cashier
POST    /praxis/notification/direct
POST    /rbs/notification
GET     /safecharge/dmn/deposit
POST    /safecharge/dmn/deposit
GET     /safecharge/dmn/withdrawal
POST    /safecharge/dmn/withdrawal
GET     /safecharge/dmn/withdrawal/payout
POST    /safecharge/dmn/withdrawal/payout
POST    /skrill/notification
POST    /skrill/notification/refund
POST    /sofort/statusNotification  [Body: StatusNotification]
POST    /stripe/notification/{license}  [Params: path:license(string)]
POST    /swiffy/deposit/notification/{license}  [Params: path:license()]
POST    /swiffy/withdrawal/notification/{license}  [Params: path:license()]
POST    /totalprocessing/notification
POST    /totalprocessing/notification/currency
POST    /transactx/notification
POST    /transfermate/notification
POST    /truelayer/notification
POST    /trustly/notification
POST    /vertupay/deposit/notification
POST    /vertupay/withdrawal/notification
POST    /volt/notification
POST    /volt/notification/withdrawal
POST    /webpay/notification
POST    /worldpay/order/notification
POST    /worldpay/order/notification/asic
POST    /wpayments/notification

================================================================================
  RAW PROBE DATA
================================================================================


### 5. VALID TENANT ENUM VALUES
  CAPITAL_COM, CURRENCY_COM, CAPITAL, CURRENCY, DEFAULT, TEST, PROD, STAGING

### 6. AUTH MECHANISM IDENTIFIED
  Header: Session-Token (required for gateway endpoints)
  Error: "Required request header 'Session-Token' for method parameter type String is not present"

### 7. WEBHOOK ENDPOINTS RESPONDING WITHOUT AUTH
  [401] POST /equalsmoney/deposit/notification
  [500] POST /equalsmoney/withdrawal/notification
  [500] POST /equalsmoney/withdrawal/cancel
  [400] POST /myfatoorah/notification/{merchantId}
  [400] POST /oneroadpayments/notification
  [415] POST /capital/deposit3dsV2 (and 10 other /capital/* 3DS endpoints)

### 8. SECURITY HEADERS
  HTTP/2 200 on /payment/docs/
  Headers: AWSALB cookies (load balancer stickiness)
  No X-Frame-Options on API responses
  No CORS headers visible on API responses

================================================================================
  FULL ENDPOINT MAP (329 endpoints, 232 paths)
================================================================================

GET     /
GET     /admin/v1/bank/details  // Get BankPaymentOptionDetailsResponse by criteria  [Params: query:license(string), query:test(boolean), query:currency(string), query:country(string)]
PUT     /admin/v1/bank/details  // Update merchant details data  [Body: MerchantBankDetailsRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/bank/details  // Add merchant details  [Body: MerchantBankDetailsRequest]  [Params: header:modifiedBy(string)]
GET     /admin/v1/bank/details/routing/{routingRuleId}  // Get BankPaymentOptionDetailsResponse by routing rule id  [Params: path:routingRuleId(integer)]
GET     /admin/v1/bank/details/{paymentConfigNodeId}  // Get BankPaymentOptionDetailsResponse by payment option config id  [Params: path:paymentConfigNodeId(integer)]
DELETE  /admin/v1/bank/details/{processingBank}/{bankAccount}  // Delete merchant details  [Params: header:modifiedBy(string), path:processingBank(string), path:bankAccount(string)]
POST    /admin/v1/blacklist  // Add new blacklisted value  [Body: BankBlackListDto]  [Params: header:modifiedBy(string)]
GET     /admin/v1/blacklist/info  // Get blacklist info by filters  [Params: query:license(), query:type()]
POST    /admin/v1/blacklist/validate  // Validate blacklist bin  [Body: ValidateBlacklistRequest]
POST    /admin/v1/blacklist/{id}  // Update blacklisted value  [Body: BankBlackListDto]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/blacklist/{id}  // Delete blacklisted value  [Params: header:modifiedBy(string), path:id(integer)]
GET     /admin/v1/commission  // Search Payment Option commission configs  [Params: header:modifiedBy(string), query:paymentOptions(array), query:transactionTypes(array), query:licenses(array)]
PUT     /admin/v1/commission  // Update Payment Option commission config  [Body: AdminPaymentOptionCommissionConfigRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/commission  // Add new Payment Option commission config  [Body: AdminPaymentOptionCommissionConfigRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/commission/{id}  // Delete Payment Option commission config  [Params: header:modifiedBy(string), path:id(integer)]
GET     /admin/v1/configNodes  // Get config
POST    /admin/v1/configNodes  // Add new config  [Body: PaymentConfigNode]
GET     /admin/v1/configNodes/{paymentConfigNodeId}  // Get config node  [Params: path:paymentConfigNodeId(integer)]
DELETE  /admin/v1/configNodes/{paymentConfigNodeId}  // Delete config node  [Params: path:paymentConfigNodeId(integer)]
GET     /admin/v1/currency/info  // Get currency info  [Params: header:modifiedBy(string)]
GET     /admin/v1/currency/local  // Get local currencies  [Params: query:currency(array)]
POST    /admin/v1/currency/local  // Add or update local currency  [Body: AdminLocalCurrencyAddOrUpdateRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/currency/local/{country}  // Delete local currency  [Params: path:country(string), header:modifiedBy(string)]
GET     /admin/v1/currency/markup  // Get FX markup for currency  [Params: query:currency(array)]
POST    /admin/v1/currency/markup  // Add FX markup  [Body: AdminFxMarkupUpdateRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/currency/markup/{id}  // Update FX markup  [Body: AdminFxMarkupUpdateRequest]  [Params: path:id(integer), header:modifiedBy(string)]
DELETE  /admin/v1/currency/markup/{id}  // Delete FX markup  [Params: path:id(integer), header:modifiedBy(string)]
GET     /admin/v1/options  // Get payment options info  [Params: query:license(), query:test(boolean), query:countryOfResidence(string), query:binCountry(string)]
POST    /admin/v1/paymentOrder/complete  // Complete the order in Payments and the transaction in UMS  [Body: UpdatePaymentOrderRequest]  [Params: header:modifiedBy(string)]
GET     /admin/v1/paymentSystems  // Get payment systems info
GET     /admin/v1/paymentoptionconfig  // Get PaymentOptionsConfig by license, currency, paymentOptions, externalSystems and test flag  [Params: query:license(string), query:currencies(array), query:paymentOptions(array), query:externalSystems(array)]
POST    /admin/v1/paymentoptionconfig  // Add new PaymentOptionsConfig  [Body: PaymentOptionsConfig]
GET     /admin/v1/paymentoptionconfig/apmMethodsList  // Get a list of available apm methods
POST    /admin/v1/paymentoptionconfig/{paymentOptionConfigId}  // Update PaymentOptionsConfig  [Body: PaymentOptionsConfig]
DELETE  /admin/v1/paymentoptionconfig/{paymentOptionConfigId}  // Delete PaymentOptionsConfig  [Params: path:paymentOptionConfigId(integer)]
GET     /admin/v1/paymentoptionordering  // Get PaymentOptionOrderingRules by license, currencies and countries  [Params: query:license(), query:countries(array), query:currencies(array)]
POST    /admin/v1/paymentoptionordering  // Add new PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
GET     /admin/v1/paymentoptionordering/default  // Get details of default PaymentOptionOrderingRule for given license  [Params: query:license()]
POST    /admin/v1/paymentoptionordering/validate  // Validate PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
GET     /admin/v1/paymentoptionordering/{id}  // Get details of PaymentOptionOrderingRule  [Params: path:id(integer)]
PUT     /admin/v1/paymentoptionordering/{id}  // Update PaymentOptionOrderingRule  [Body: PaymentOptionOrderingRule]
DELETE  /admin/v1/paymentoptionordering/{id}  // Delete PaymentOptionOrderingRule  [Params: path:id(integer)]
GET     /admin/v1/paymentoptionrouting  // Get PaymentOptionRoutingRules by license, currency, binCountry and test flag  [Params: query:license(), query:paymentOption(), query:currencies(array), query:countries(array)]
POST    /admin/v1/paymentoptionrouting  // Add new PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
DELETE  /admin/v1/paymentoptionrouting/card/deleteRules  // Delete all non default bank card cascades, available only for test env  [Body: DeleteAllCardRulesRequest]
POST    /admin/v1/paymentoptionrouting/validate  // Validate PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
POST    /admin/v1/paymentoptionrouting/{ruleId}  // Update PaymentOptionRoutingRule  [Body: PaymentOptionRoutingRule]
DELETE  /admin/v1/paymentoptionrouting/{ruleId}  // Delete PaymentOptionRoutingRule  [Params: path:ruleId(integer)]
GET     /admin/v1/properties  // Get all properties
POST    /admin/v1/properties  // Add properties  [Body: AddPropertiesRequest]
GET     /admin/v1/properties/name/{name}  // Get properties by name containing  [Params: path:name(string)]
GET     /admin/v1/properties/{propertiesId}  // Get properties by id  [Params: path:propertiesId(integer)]
POST    /admin/v1/properties/{propertiesId}  // Update properties  [Body: UpdatePropertiesRequest]  [Params: path:propertiesId(integer)]
DELETE  /admin/v1/properties/{propertiesId}  // Delete properties by id  [Params: path:propertiesId(integer)]
GET     /admin/v1/setting  // Get Settings  [Params: header:modifiedBy(string)]
GET     /admin/v1/setting/global  // Get Settings Global  [Params: header:modifiedBy(string), query:keys(array), query:licenses(array), query:countries(array)]
PUT     /admin/v1/setting/global  // Update Settings Global  [Body: AdminSettingGlobalRequest]  [Params: header:modifiedBy(string)]
POST    /admin/v1/setting/global  // Add new Settings Global  [Body: AdminSettingGlobalRequest]  [Params: header:modifiedBy(string)]
DELETE  /admin/v1/setting/global/{id}  // Delete Setting Global  [Params: header:modifiedBy(string), path:id(integer)]
POST    /azupay/notification
POST    /azupay/v2/notification
POST    /azupay/withdrawal/notification
POST    /bancasella/notification
POST    /bankingcircle/accountverification/notification/{operatingEntity}  [Params: path:operatingEntity(string), header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /bankingcircle/notification  [Params: header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /bankingcircle/notification/{operatingEntity}  [Params: path:operatingEntity(string), header:AuthenticationTag(string), header:Nonce(string), header:Checksum(string)]
POST    /capital/assembly3dsV2  // Complete deposit via credit card with 3DS  [Params: query:assembledBrowserDetails(), query:ACSUnavailable(boolean), query:sid(string), query:tenant()]
POST    /capital/auth3ds  // Handles 3DS 2.0 authentication notification  [Params: query:request()]
POST    /capital/auth3ds/nested  // Handles 3DS 2.0 authentication notification for nested iframes  [Params: query:request()]
POST    /capital/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:cres(string)]
POST    /capital/complete3ds  // Complete deposit via credit card with 3DS  [Body: MultiValueMapStringString]
DELETE  /capital/deleteCard  // Delete credit card  [Body: DeleteCardRequest]
GET     /capital/deposit  // Get deposit URL. Can return URL to external iframe  [Params: header:App-Version(string), query:sid(string)]
POST    /capital/deposit  // Deposit via credit card  [Params: header:App-Version(string), query:depositData()]
POST    /capital/deposit3dsV2  // Handles 3DS 2.0 deposit notification  [Params: query:tenant(), query:sid(string), query:PaReq(string)]
POST    /capital/ecommpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:pares(string), query:sid(string)]
GET     /capital/mpi  // Complete mpi via credit card with 3DS  [Params: query:sid(string), query:tenant()]
POST    /capital/mpi  // Complete mpi via credit card with 3DS  [Params: query:sid(string), query:tenant()]
POST    /capital/networkint/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:cres(string)]
GET     /capital/praxis/redirect/CAPITAL_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/redirect/CAPITAL_COM  [Params: query:status(string), query:app(string)]
GET     /capital/praxis/redirect/CURRENCY_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/redirect/CURRENCY_COM  [Params: query:status(string), query:app(string)]
POST    /capital/praxis/validate  [Body: PraxisValidationRequest]
GET     /capital/submit3ds  // Handles 3DS deposit redirects  [Params: query:tenant(), query:sid(string)]
POST    /capital/submit3ds  // Handles 3DS deposit notification  [Params: query:tenant(), query:sid(string), query:PaReq(string)]
GET     /capital/webpay/challenge3ds  // Handles 3DS 2.0 challenge notification with extra parameters  [Params: query:request()]
POST    /capital/webpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:tenant(), query:sid(string), query:cres(string)]
POST    /capital/worldpay/challenge3ds  // Handles 3DS 2.0 challenge notification  [Params: query:MD(string)]
POST    /cardpay/notification/htp  [Params: header:Signature(string)]
POST    /checkout/notification  [Params: query:license(), query:operatingEntity()]
POST    /clearbank/notification  [Params: header:DigitalSignature(string)]
POST    /credorax/apm/deposit/notification
POST    /credorax/deposit/notification
POST    /d24/notification/deposit
POST    /d24/notification/refund
POST    /dbs/deposit/notification
POST    /dbs/withdrawal/notification
POST    /ecommpay/notification
POST    /ecommpay/notification/redirect
POST    /ecommpay/notification/token
POST    /equalsmoney/deposit/notification
POST    /equalsmoney/withdrawal/cancel
POST    /equalsmoney/withdrawal/notification
POST    /erip/deposits/notification  [Params: query:XML(string)]
GET     /exactly/notification  // Handle notification object from Exactly
POST    /exactly/notification  // Handle notification object from Exactly
POST    /fuze/notification
GET     /gateway/v1/beneficiaryCustomer/validate  // Validate Beneficiary Customer  [Params: header:Session-Token(string), query:beneficiaryCustomer(string)]
POST    /gateway/v1/card/ani  // Validate Card Number  [Body: CardAccountNumberInquiryRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/commission  // Get commission  [Params: header:Session-Token(string), query:accountId(integer), query:paymentType(), query:paymentOption()]
GET     /gateway/v1/commission/card  // Get commission for card  [Params: header:Session-Token(string), query:accountId(integer), query:paymentType(), query:bin(string)]
GET     /gateway/v1/crypto/deposit/details  // Get Crypto deposit details  [Params: header:Session-Token(string), query:request()]
GET     /gateway/v1/crypto/deposit/terms  // Get a link to the terms  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/crypto/deposit/terms  // Save crypto deposit terms  [Body: CryptoDepositSaveTermsRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/crypto/virtualAssetProviders  // Get a list of virtual asset providers  [Params: header:Session-Token(string), query:accountId(integer), query:providerName(string)]
GET     /gateway/v1/crypto/whitelist/address  // Get user's whitelisted addresses on PSP side  [Params: header:Session-Token(string), query:accountId(integer), query:chain(string), query:paymentType()]
POST    /gateway/v1/crypto/whitelist/address  // Whitelist an address on PSP side  [Body: AddCryptoWhitelistAddressRequest]  [Params: header:Session-Token(string)]
DELETE  /gateway/v1/crypto/whitelist/address  // Deactivate a user's whitelisted address  [Params: header:Session-Token(string), query:id(integer)]
POST    /gateway/v1/crypto/withdrawal  // Withdrawal via crypto currency  [Body: CryptoWithdrawalRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/crypto/withdrawal/details  // Get Crypto withdrawal details  [Params: header:Session-Token(string), query:request()]
GET     /gateway/v1/dcc/rate  // Get DCC rate  [Params: header:Session-Token(string), query:accountId(integer), query:type(), query:options(array)]
GET     /gateway/v1/dcc/rate/card  // Get DCC rate for card  [Params: header:Session-Token(string), query:accountId(integer), query:type(), query:bin(string)]
GET     /gateway/v1/decline/cardpay  [Params: query:params()]
PUT     /gateway/v1/decline/cardpay  [Params: query:params()]
POST    /gateway/v1/decline/cardpay  [Params: query:params()]
DELETE  /gateway/v1/decline/cardpay  [Params: query:params()]
PATCH   /gateway/v1/decline/cardpay  [Params: query:params()]
GET     /gateway/v1/decline/checkout  [Params: query:cko-session-id(string), query:license(), query:operatingEntity()]
GET     /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
PUT     /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
POST    /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
DELETE  /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
PATCH   /gateway/v1/decline/myfatoorah  [Params: query:params(), query:paymentId(string)]
GET     /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
PUT     /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
POST    /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
DELETE  /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
PATCH   /gateway/v1/decline/poli  [Params: query:sid(string), query:token(string)]
GET     /gateway/v1/deposit  // Get deposit URL. Can return URL to external iframe. email is used for skrill deposit to override skrill account name  [Params: header:Session-Token(string), query:accountId(integer), query:amount(number), query:option()]
POST    /gateway/v1/deposit/card  // Deposit via credit card  [Body: DepositCardRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/deposit/card/bin/check  // Check credit card BIN  [Body: CardBinCheckRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
PUT     /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
POST    /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
DELETE  /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
PATCH   /gateway/v1/deposit/card/decline  // Process card deposit decline redirect  [Params: query:sid(string), query:license(), query:declineCode()]
GET     /gateway/v1/deposit/stripe  [Params: query:sid(string)]
PUT     /gateway/v1/deposit/stripe  [Params: query:sid(string)]
POST    /gateway/v1/deposit/stripe  [Params: query:sid(string)]
DELETE  /gateway/v1/deposit/stripe  [Params: query:sid(string)]
PATCH   /gateway/v1/deposit/stripe  [Params: query:sid(string)]
GET     /gateway/v1/iban/validate  // Validate iban  [Params: header:Session-Token(string), query:iban(string)]
POST    /gateway/v1/lean/createPaymentIntent  // Create payment Intent for Lean deposit  [Body: LeanCommunicationRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/lean/createWithdrawalPaymentMethod  // Create Lean Payment Method for Withdrawals  [Body: LeanCreateWithdrawalPaymentMethodRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/lean/data/getDepositDetails/{currency}  // Deposit data for Lean payment  [Params: header:Session-Token(string), path:currency(string)]
POST    /gateway/v1/lean/unlinkAccount  // Unlink account  [Body: LeanCommunicationRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/options/commission  // Get payment options commissions info  [Params: query:license(), query:country(string)]
GET     /gateway/v1/options/info  // Get payment available options  [Params: query:countryOfResidence(string), query:license()]
GET     /gateway/v1/payment/truelayer  [Params: query:payment_id(string), query:error(string)]
GET     /gateway/v1/paymentMethods  // Get payment methods  [Params: header:Session-Token(string), header:App-Version(string), query:option(), query:accountId(integer)]
POST    /gateway/v1/paymentMethods/card  // Add new credit card payment method  [Body: CardPaymentMethodAddRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/paymentMethods/unverified  // Get unverified payment methods  [Params: header:Session-Token(string), query:accountId(integer)]
PUT     /gateway/v1/paymentMethods/{paymentMethodId}  // Edit payment method  [Body: PaymentMethodEditRequest]  [Params: header:Session-Token(string), path:paymentMethodId(integer)]
DELETE  /gateway/v1/paymentMethods/{paymentMethodId}  // Delete payment method  [Params: header:Session-Token(string), path:paymentMethodId(integer)]
GET     /gateway/v1/paymentPage/info  // Get payment options info  [Params: header:Session-Token(string), query:accountId(integer)]
GET     /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
PUT     /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
DELETE  /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
PATCH   /gateway/v1/redirect/credorax  [Params: query:allParams(object)]
GET     /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
PUT     /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
DELETE  /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
PATCH   /gateway/v1/redirect/credorax/mpi  [Params: query:allParams(object)]
POST    /gateway/v1/redirect/exactly  [Params: query:params()]
POST    /gateway/v1/redirect/mercuryo  [Params: query:params()]
GET     /gateway/v1/result/azupay  [Params: query:params(), query:sid(string), query:license(), query:paymentRequestId(string)]
GET     /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
PUT     /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
POST    /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
DELETE  /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
PATCH   /gateway/v1/result/myfatoorah  [Params: query:params(), query:paymentId(string)]
GET     /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
PUT     /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
POST    /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
DELETE  /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
PATCH   /gateway/v1/result/paycom  [Params: query:sid(string), query:chargeId(string), query:license()]
GET     /gateway/v1/result/stripe  [Params: query:app(boolean), query:sid(string), query:license(), query:payment_intent(string)]
GET     /gateway/v1/result/volt  [Params: query:volt(string), query:volt-signature(string), query:volt-timestamp(string)]
GET     /gateway/v1/skrill/cancel  [Params: query:params(), query:sid(string), query:license()]
GET     /gateway/v1/threedsecure/totalprocessing  [Params: query:params(), query:currency(string), query:sid(string), query:license()]
GET     /gateway/v1/transactx/result
PUT     /gateway/v1/transactx/result
POST    /gateway/v1/transactx/result
DELETE  /gateway/v1/transactx/result
PATCH   /gateway/v1/transactx/result
GET     /gateway/v1/transfermate/data/banks  // List of available banks  [Params: header:Session-Token(string)]
POST    /gateway/v1/transfermate/data/saveBank  // Save bank code and bank name  [Body: TransferMateSaveBankRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/truelayer/data/auth  [Params: query:code(string), query:state(string), query:error(string)]
GET     /gateway/v1/truelayer/data/linkedAccounts  // List of selected bank accounts  [Params: header:sid(string)]
POST    /gateway/v1/truelayer/data/saveAccount  // Save bank account  [Body: TrueLayerSaveAccountRequest]
GET     /gateway/v1/vertupay/data/banks  // List of available bank codes
GET     /gateway/v1/virtualIban  // Get virtual iban  [Params: header:Session-Token(string), query:accountId(integer), query:bankCode(string), query:currency(string)]
POST    /gateway/v1/virtualIban  // Create virtual iban  [Body: CreateVirtualIbanRequestDto]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/apple/deposit/card  // Make Apple Pay deposit  [Body: ApplePayDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wallet/apple/deposit/details  // Get Apple Pay deposit details  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wallet/apple/deposit/session  // Create Apple Pay payment session  [Body: ApplePayCreatePaymentSessionRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/erip/deposit/session  // Create ERIP payment session  [Body: EripCreatePaymentSessionRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wallet/google/deposit/card  // Make Google Pay deposit  [Body: GooglePayDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wallet/google/deposit/details  // Get Google Pay deposit details  [Params: header:Session-Token(string), query:accountId(integer)]
GET     /gateway/v1/wallet/paypal/checkout/token  // Get Paypal token for client  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wallet/paypal/deposit  // Make PayPal deposit  [Body: PayPalDepositRequest]  [Params: header:Session-Token(string)]
GET     /gateway/v1/wire/bankDepositDetails  // Get bank details for deposit  [Params: header:Session-Token(string), query:accountId(integer)]
POST    /gateway/v1/wire/paymentMethods/am  // Create payment method for Armenia Bank Account  [Body: WirePaymentMethodAddAmRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/au  // Create payment method for Australia Account  [Body: WirePaymentMethodAddAuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/generic  // Create payment method for Bank Account  [Body: WirePaymentMethodAddGenericRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/iban  // Create payment method for International Bank Account  [Body: WirePaymentMethodAddIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/in  // Create payment method for India Bank Account  [Body: WirePaymentMethodAddInRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/ru  // Create payment method for RUB Russian Bank Account  [Body: WirePaymentMethodAddRuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/ru/foreign  // Create payment method for non RUB Russian Bank Account  [Body: WirePaymentMethodAddRuForeignRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/paymentMethods/uk  // Create payment method for British or Irish Bank Account  [Body: WirePaymentMethodAddUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal  // Withdrawal  [Body: WireWithdrawalGenericRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/am  // Withdrawal to Armenia Bank Account  [Body: WireWithdrawalAmRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/iban  // Withdrawal to International Bank Account  [Body: WireWithdrawalIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/in  // Withdrawal to India Bank Account  [Body: WireWithdrawalInRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/ru  // Withdrawal to RUB Russian Bank Account  [Body: WireWithdrawalRuRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/ru/foreign  // Withdrawal to non RUB Russian Bank Account  [Body: WireWithdrawalRuForeignRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/sortCode  // Withdrawal to British or Irish Bank Account  [Body: WireWithdrawalUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/swift  // Withdrawal to International Bank Account  [Body: WireWithdrawalIbanRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/wire/withdrawal/uk  // Withdrawal to British or Irish Bank Account  [Body: WireWithdrawalUkRequest]  [Params: header:Session-Token(string)]
POST    /gateway/v1/withdrawal  // Withdrawal via specific payment method  [Body: WithdrawalRequest]  [Params: header:Session-Token(string), header:App-Version(string)]
POST    /imbank/notification
GET     /internal/v1/bankDepositDetails  // Get bank details for deposit  [Params: query:license(), query:operatingEntity(), query:currency(string), query:country(string)]
GET     /internal/v1/healthcheck/ping  // Check rest service availability
GET     /internal/v1/isProcessingBankSupportsReturn  [Params: query:license(), query:processingBank(string), query:paymentBeta(boolean)]
GET     /internal/v1/operation/info  [Params: header:modifiedBy(string)]
POST    /internal/v1/operation/request  [Body: PaymentOrderRequestNewRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/operation/schema  [Params: query:request(), header:modifiedBy(string)]
GET     /internal/v1/operation/status  [Params: query:request(), header:modifiedBy(string)]
POST    /internal/v1/order/complete  [Body: PaymentOrderStatusChangeRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/order/info  [Params: header:modifiedBy(string)]
POST    /internal/v1/order/request  [Body: PaymentOrderRequestNewRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/order/schema  [Params: query:request(), header:modifiedBy(string)]
GET     /internal/v1/order/status  [Params: query:request(), header:modifiedBy(string)]
POST    /internal/v1/paymentMethod/block  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/delete  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/restore  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/paymentMethod/unblock  [Body: PaymentMethodUpdateRequest]  [Params: header:modifiedBy(string), header:signature(string)]
POST    /internal/v1/withdrawal  [Body: CreateWithdrawalRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /internal/v1/withdrawal/bankProviders  [Params: query:license(), query:operatingEntity(), query:currency(string), query:processingCurrency(string)]
POST    /internal/v1/withdrawal/split  [Body: PaymentWithdrawalSplitRequest]  [Params: header:modifiedBy(string), header:signature(string)]
GET     /itez/notification  [Params: header:X-Payload-Digest(string)]
PUT     /itez/notification  [Params: header:X-Payload-Digest(string)]
POST    /itez/notification  [Params: header:X-Payload-Digest(string)]
DELETE  /itez/notification  [Params: header:X-Payload-Digest(string)]
PATCH   /itez/notification  [Params: header:X-Payload-Digest(string)]
POST    /klarpay/deposit/notification
POST    /lean/notification  [Params: header:lean-signature(string)]
GET     /mercuryo/notification  // Handle notification object from Mercuryo  [Params: query:data(string), query:sign(string), query:license(), query:currency(string)]
POST    /mercuryo/notification  // Handle notification object from Mercuryo  [Params: query:data(string), query:sign(string), query:license(), query:currency(string)]
GET     /mercuryo/payout/notification  // Handle payout notification object from Mercuryo  [Params: query:license(), query:currency(string)]
POST    /mercuryo/payout/notification  // Handle payout notification object from Mercuryo  [Params: query:license(), query:currency(string)]
POST    /modulr/deposit/notification
POST    /modulr/withdrawal/notification
POST    /monetix/notification
POST    /monetix/notification/redirect
POST    /monetix/notification/token
GET     /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
PUT     /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
POST    /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
DELETE  /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
PATCH   /myfatoorah/notification/{merchantId}  [Params: path:merchantId(string), header:MyFatoorah-Signature(string)]
POST    /nbd/withdrawal/notification
GET     /neteller/notification
PUT     /neteller/notification
POST    /neteller/notification
DELETE  /neteller/notification
PATCH   /neteller/notification
POST    /networkint/notification
GET     /oneroadpayments/notification  [Params: query:success(string), query:merchantID(string), query:orderID(string), query:clientID(string)]
POST    /oneroadpayments/notification  [Params: query:success(string), query:merchantID(string), query:orderID(string), query:clientID(string)]
POST    /openpayd/withdrawal/notification  [Params: header:Signature(string)]
POST    /pawapay/deposit/notification
POST    /pawapay/refund/notification
POST    /pawapay/withdrawal/notification
GET     /paycom/notification/{license}  [Params: path:license()]
PUT     /paycom/notification/{license}  [Params: path:license()]
POST    /paycom/notification/{license}  [Params: path:license()]
DELETE  /paycom/notification/{license}  [Params: path:license()]
PATCH   /paycom/notification/{license}  [Params: path:license()]
POST    /paymob/notification  [Params: query:hmac(string)]
POST    /paypal/braintree/notification  // Handle notification object from Braintree Gateway  [Params: query:license()]
GET     /paypal/payments/notification  // Handle notification object from Paypal  [Params: query:license()]
POST    /paypal/payments/notification  // Handle notification object from Paypal  [Params: query:license()]
POST    /payretailers/deposit/notification
POST    /payretailers/withdrawal/notification
POST    /poli/notification  [Params: query:Token(string)]
POST    /praxis/notification/cashier
POST    /praxis/notification/direct
POST    /rbs/notification
GET     /safecharge/dmn/deposit
POST    /safecharge/dmn/deposit
GET     /safecharge/dmn/withdrawal
POST    /safecharge/dmn/withdrawal
GET     /safecharge/dmn/withdrawal/payout
POST    /safecharge/dmn/withdrawal/payout
POST    /skrill/notification
POST    /skrill/notification/refund
POST    /sofort/statusNotification  [Body: StatusNotification]
POST    /stripe/notification/{license}  [Params: path:license(string)]
POST    /swiffy/deposit/notification/{license}  [Params: path:license()]
POST    /swiffy/withdrawal/notification/{license}  [Params: path:license()]
POST    /totalprocessing/notification
POST    /totalprocessing/notification/currency
POST    /transactx/notification
POST    /transfermate/notification
POST    /truelayer/notification
POST    /trustly/notification
POST    /vertupay/deposit/notification
POST    /vertupay/withdrawal/notification
POST    /volt/notification
POST    /volt/notification/withdrawal
POST    /webpay/notification
POST    /worldpay/order/notification
POST    /worldpay/order/notification/asic
POST    /wpayments/notification

================================================================================
  RAW PROBE RESULTS
================================================================================

--- Probe 4: OpenAPI Groups ---
  /payment/docs/         => 200 (full spec)
  /payment/docs/payment  => 200 (same spec)
  /payment/docs/gateway  => 500 (No OpenAPI resource found)
  /payment/docs/internal => 500 (No OpenAPI resource found)
  /payment/docs/admin    => 500 (No OpenAPI resource found)

--- Probe 5: Unauthenticated Endpoint Responses ---
[OPEN] GET /gateway/v1/deposit
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token' for method parameter type String is not present"}
[OPEN] GET /gateway/v1/options/info
  {"errorCode":"FIELD_VALIDATION_ERROR","userMessage":"must not be null","debugMessage":"Validation failed for argument [0]..."}
[OPEN] GET /gateway/v1/options/commission
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request parameter 'license' for method parameter type License is not present"}
[OPEN] GET /gateway/v1/paymentPage/info
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/paymentMethods
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/commission
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/dcc/rate
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/beneficiaryCustomer/validate
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/iban/validate
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/crypto/virtualAssetProviders
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/crypto/deposit/details
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/crypto/deposit/terms
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/wire/bankDepositDetails
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/virtualIban
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/transfermate/data/banks
  {"errorCode":"BAD_REQUEST","debugMessage":"Required request header 'Session-Token'..."}
[OPEN] GET /gateway/v1/vertupay/data/banks
  {"VN":["ACB","BIDV","DAB"]}  *** NO AUTH - DATA LEAK ***

--- Probe 7: Detailed Responses ---
  VertuPay Banks (NO AUTH): {"VN":["ACB","BIDV","DAB"]}
  Options Info with params: {"data":[]}
  Commission CYSEC: Full commission data returned (VISA, Mastercard, etc.)
  Commission FCA: Full commission data returned
  Commission ASIC: Full commission data returned
  Commission NBRB: Full commission data returned
  Commission SCB: Failed to convert (invalid enum)
  Commission CIMA: Failed to convert (invalid enum)

--- Probe 8: Response Headers ---
  /payment/docs/: HTTP/2 200, content-type: application/json, AWSALB cookie
  /payment/gateway/v1/deposit: HTTP/2 400, AWSALB + AWSALBCORS cookies
  /: HTTP/2 403, x-iinfo header (Incapsula)

  Session-Token: test on /deposit:
    Validation error revealing: com.expcapital.payment.gateway.rest.PaymentGatewayRestController.depositRefV1()
    Fields: accountId, sid (Session-Token accepted as valid!)

  Session-Token: test on /paymentPage/info:
    {"errorCode":"BAD_REQUEST","debugMessage":"Required request parameter 'accountId'..."}

--- Probe 11: Validation Error Extraction ---
  POST /gateway/v1/transfermate/data/saveBank (Session-Token: test):
    Reveals: com.expcapital.payment.gateway.rest.PaymentTransfermateRestController.saveBank()
    Fields: bankCode (NotNull), paymentMethodId (NotNull), bankName (NotNull)

--- Probe 12: License/Tenant Enumeration ---
  Valid Licenses: CYSEC, FCA, ASIC, NBRB
  Valid Tenants: CAPITAL_COM, CURRENCY_COM, CAPITAL, CURRENCY, DEFAULT, TEST, PROD, STAGING

--- Probe 15: Webhook Testing ---
  PayPal JSON POST: [403] (WAF blocked)
  Stripe JSON POST: [403] (WAF blocked)
  Skrill JSON POST: [403] (WAF blocked)
  EqualsMoney deposit: [401] (reached server!)
  EqualsMoney withdrawal: [500] (reached server!)
  MyFatoorah: [400] (reached server!)
  OneRoadPayments: [400] (reached server!)
  Capital 3DS endpoints: [415] (reached server - wrong content type)

--- Probe 16: Advanced Testing ---
  Path traversal: All returned 403 (WAF blocks)
  HTTP method override: Blocked by WAF
  X-HTTP-Method-Override: Blocked by WAF


================================================================================
  PHASE 2: DEEPER DISCOVERY
================================================================================

--- Full License Enum Brute Force ---
License schema: {
  "type": "string",
  "enum": [
    "CYSEC",
    "CYSEC_STOCK",
    "CYSEC_ONEX",
    "CYSEC_ITALY",
    "CYSEC_MICA",
    "FCA",
    "FCA_STOCK",
    "FCA_ONEX",
    "NBRB",
    "CCSTV",
    "HTP",
    "HTP_FINAPP",
    "DLT",
    "DLT_PRO",
    "HTP_SPATIUM",
    "ASIC",
    "ASIC_ONEX",
    "CXSTV",
    "CCUS",
    "CXUS",
    "CXUS_ZH",
    "SEY",
    "BAH",
    "HTP_OLD",
    "BAH_SH",
    "BAH_IN",
    "CCMENA",
    "CCMENA_ONEX",
    "KECMA",
    "CIRO",
    "BMA",
    "SAFSCA",
    "SGMAS"
  ]
}

--- Tenant Enum ---
Tenant schema: {
  "type": "string",
  "enum": [
    "CAPITAL_COM",
    "CURRENCY_COM",
    "SHARES_COM",
    "INVESTMATE"
  ]
}

--- All Enum Schemas ---
AcsState: ['COMPLETED', 'FAILED', 'TIMEOUT']
ApiErrorCode: ['BAD_REQUEST', 'FIELD_VALIDATION_ERROR', 'INTERNAL_SERVER_ERROR', 'ILLEGAL_STATE', 'SESSION_EXPIRED', 'PERMISSION_DENIED', 'ADDRESS_NOT_SET', 'REGISTRATION_FORM_NOT_SUBMITTED', 'ACCOUNT_NOT_FOUND', 'INVALID_ACCOUNT_TYPE', 'PAYMENT_METHOD_NOT_FOUND', 'PAYMENT_METHOD_BLOCKED', 'PAYMENT_METHOD_TEMPORARILY_BLOCKED', 'PAYMENT_METHOD_ALREADY_EXISTS', 'PAYMENT_METHOD_NOT_AVAILABLE', 'PAYMENT_METHOD_NOT_AVAILABLE_FOR_WITHDRAWAL', 'CARD_METHOD_ONLY_SUPPORTED', 'BIN_NOT_SUPPORTED', 'CARD_NOT_SUPPORTED', 'CARD_VISA_ONLY_SUPPORTED', 'CVV_REQUIRED', 'INVALID_AMOUNT', 'LIMIT_EXCEEDED', 'SKRILL_ACCOUNT_VERIFICATION_FAILED', 'CARD_ACCOUNT_VERIFICATION_FAILED', 'PROPERTIES_NOT_FOUND', 'PROPERTIES_NAME_ALREADY_EXIST', 'PAYMENT_ORDER_NOT_FOUND', 'NOT_FINAL_STATUS_IN_REQUEST', 'ORDER_ALREADY_PROCESSED', 'ORDER_STATUS_CHANGE_FAILED', 'RATE_LIMIT', 'CONVERSION_RATE_EXPIRED', 'COMMISSION_EXPIRED', 'VERIFICATION_FAILED', 'REACHED_MAX_FAILED_ATTEMPTS', 'COR_NOT_SUPPORTED', 'TERMS_NOT_CONFIRMED', 'OPERATION_NOT_ALLOWED']
AppExternalInterface: ['TELEGRAM_MINI_APPS']
AppPlatform: ['ANY', 'ANDROID', 'IOS', 'WEB']
ApplePayMerchantCapability: ['THREE_D_S', 'CREDIT', 'DEBIT', 'EMV']
ApplePayPaymentTokenVersion: ['EC_v1', 'RSA_v1']
BankAccountType: ['IBAN', 'UK', 'FPS', 'RU', 'RU_FOREIGN', 'AM', 'IN', 'AU', 'MX']
BankBlackListReasonCode: ['REVOLUT', 'BLOCKED_BY_BANK']
BankBlackListType: ['BANK_CODE', 'BIN', 'BANK_COUNTRY']
CardType: ['VISA', 'MASTERCARD', 'AMEX', 'MAESTRO', 'AMERICAN_EXPRESS', 'DISCOVER', 'DINERS_CLUB', 'JCB', 'MIR', 'UNDEFINED', 'BELKART', 'CHINA_UNION_PAY']
CryptoWalletType: ['CUSTODIAL', 'DECENTRALIZED']
DccFlowType: ['NONE', 'BOTH_CURRENCIES', 'LOCAL_ONLY']
DeclineCode: ['SESSION_EXPIRED', 'CANCELLED', 'INTERNAL_SERVER_ERROR', 'INVALID_CARD_DETAILS', 'INCORRECT_PAN', 'INVALID_PIN', 'DECLINED_BY_BANK', 'DECLINED_BY_BANK_FATAL', 'TRANSACTION_NOT_PERMITTED', 'INSUFFICIENT_FUNDS', 'AUTH_3D_FAILED', 'AUTH_3D_EXPIRED', 'INVALID_CVV2', 'EXPIRED_CARD', 'DECLINED_BY_GATEWAY', 'DECLINED_BY_GATEWAY_FATAL', 'BLOCKED', 'PAYMENT_METHOD_BLOCKED', 'PAYMENT_METHOD_TEMPORARILY_BLOCKED', 'PAYMENT_METHOD_USED_BY_ANOTHER_CLIENT', 'SKRILL_ACCOUNT_VERIFICATION_FAILED', 'OPERATION_NOT_SUPPORTED', 'OTHER', 'PAYPAL_ACCOUNT_SINGLE_LINKED', 'PAYPAL_ACCOUNT_MISSING_VAULTED_PAYMENT_METHOD_TOKEN', 'CARD_LIMIT_EXCEEDED', 'ACCOUNT_LIMIT_EXCEEDED', 'TECHNICAL_ISSUE', 'TECHNICAL_ISSUE_TIMEOUT', 'RECEIPT_UNVERIFIED', 'INIT_PAYMENT_REQUIRED', 'CARD_TYPE_NOT_SUPPORTED_CODE', 'COUNTRY_NOT_SUPPORTED_CODE', 'BLOCKED_BY_BANK_CODE', 'BIN_BLOCKED_CODE', 'BIN_BLOCKED_REVOLUT_CODE', 'OPERATION_AMOUNT_EXCEEDED', 'AFT_VALIDATION', 'WITHDRAWAL_CASCADE_CHANGED']
DepositCommissionDiscountType: ['FIRST_DEPOSIT']
ExternalSystem: ['SAFECHARGE', 'ECOMMPAY', 'CAPITAL_PAYMENT', 'INTERNAL_TRANSACTION', 'PAYPAL', 'SOFORT', 'WORLDPAY', 'TRUSTLY', 'BEPAID', 'CLEAR_JUNCTION', 'MODULR', 'WPAYMENTS', 'PIASTRIX', 'CHECKOUT', 'PRAXIS', 'ONE_ROAD_PAYMENTS', 'CITIZEN_PAYMENTS', 'CARDPAY', 'ALFABANK', 'AEROPAY', 'ERIP', 'PAY_RETAILERS', 'POLI', 'TOTALPROCESSING', 'TRANSACTX', 'EXACTLY', 'CREDORAX', 'MERCURYO', 'TRUELAYER', 'SKRILL', 'NETELLER', 'VOLT', 'STRIPE', 'PAY_COM', 'EQUALS_MONEY', 'OPENPAYD', 'AZUPAY', 'LHV', 'D24', 'LEAN', 'MY_FATOORAH', 'KLARPAY', 'NBD', 'ITEZ', 'ECOMMPAY_ITEZ', 'CIRCLE', 'WEBPAY', 'TRANSFERMATE', 'BANCASELLA', 'PAWAPAY', 'VERTUPAY', 'RBS', 'IFX', 'CBA', 'PARITET', 'BNB', 'BSB', 'CLEARBANK', 'FUZE', 'NETWORK_INT', 'SWIFFY', 'PAYMOB', 'DBS', 'IMBANK']
GetVirtualIbanStatus: ['NOT_REQUESTED', 'OK', 'FAILED', 'PENDING']
GooglePayAllowedAuthMethod: ['PAN_ONLY', 'CRYPTOGRAM_3DS']
GooglePayEnvironment: ['TEST', 'PRODUCTION']
GooglePayIntegrationType: ['DIRECT', 'GATEWAY']
Level: ['TENANT', 'LICENSE', 'PAYMENT_OPTION', 'PSP_PAYMENT_OPTION', 'PSP_INSTANCE']
License: ['CYSEC', 'CYSEC_STOCK', 'CYSEC_ONEX', 'CYSEC_ITALY', 'CYSEC_MICA', 'FCA', 'FCA_STOCK', 'FCA_ONEX', 'NBRB', 'CCSTV', 'HTP', 'HTP_FINAPP', 'DLT', 'DLT_PRO', 'HTP_SPATIUM', 'ASIC', 'ASIC_ONEX', 'CXSTV', 'CCUS', 'CXUS', 'CXUS_ZH', 'SEY', 'BAH', 'HTP_OLD', 'BAH_SH', 'BAH_IN', 'CCMENA', 'CCMENA_ONEX', 'KECMA', 'CIRO', 'BMA', 'SAFSCA', 'SGMAS']
Method: ['deposit', 'withdraw', 'approve_withdrawal', 'deny_withdrawal', 'account_ledger', 'view_automatic_settlement_details_csv', 'balance', 'get_withdrawals', 'refund', 'credit', 'debit', 'pending', 'cancel', 'account', 'charge', 'select_account', 'account_payout']
OperatingEntity: ['CCSV', 'CCEU', 'CCUK', 'CCAU', 'CCMENA', 'CCBAH', 'CCBY', 'CCSEY', 'CCSTV', 'CXBY', 'CXDLT', 'CXSTV', 'CXUS', 'CCUS', 'CCKE', 'CCCA', 'CCBER', 'CCSA', 'CCSG', 'CVCY']
PageOpenMode: ['iframe', 'new_tab']
PaymentMethodViolation: ['COUNTRY_NOT_SUPPORTED', 'CARD_TYPE_NOT_SUPPORTED', 'BLOCKED_BY_BANK', 'BIN_BLOCKED', 'BIN_BLOCKED_REVOLUT']
PaymentMode: ['DEPOSIT_OFF', 'WITHDRAWAL_OFF', 'OFF', 'ON']
PaymentOption: ['CARD', 'NETELLER', 'PAYPAL', 'BANK', 'BANK_QR', 'INTERNAL', 'SKRILL', 'SOFORT', 'IDEAL', 'GIROPAY', 'MULTIBANCO', 'PRZELEWY', 'BLIK', 'QIWI', 'WEBMONEY', 'TRUSTLY', 'SKRILL_RAPID_TRANSFER', 'PAYMENTS_2C2P', 'ASTROPAY_TEF', 'SEPA', 'UNIONPAY', 'UNIONPAY_CHINA', 'UNIONPAY_GLOBAL', 'BANK_INDONESIA', 'BANK_INDONESIA_VA', 'BANK_MALAYSIA', 'BANK_THAILAND', 'BANK_THAILAND_QR', 'BANK_VIETNAM', 'BANK_VIETNAM_QR', 'BANK_TRANSFER_VIETNAM', 'BANK_CHINA', 'BANK_INDIA', 'BANK_PHILIPPINES', 'DRAGONPAY', 'DOKU', 'NGANLUONG', 'APPLEPAY', 'GOOGLEPAY', 'GOOGLEPAY_REF', 'BANK_UK', 'YANDEX_MONEY', 'ALFA_CLICK', 'PAY_SAFE_CARD', 'CROSS_BANK_QR', 'BANGKOK_BANK_ATM', 'BANGKOK_BANK_IPAY', 'BANGKOK_BANK_MOBILE', 'KBANK_ATM', 'KBANK_MOBILE', 'SCB_ATM', 'SCB_MOBILE', 'SCB_EASY_PAY', 'UPI', 'NETBANKING_TW', 'PAY_RETAILERS', 'PAGO_FACIL', 'RAPIPAGO', 'EFECTY', 'BALOTO', 'DAVIVIENDA', 'BANCO_ESTADO', 'BCI', 'BANORTE', 'SPEI', 'OXXO', 'BBVA_BANCOMER', 'BOLETO', 'BOLETO_RAPIDO', 'TRANSFERENCIA_BANCARIA', 'OPEN_BANKING', 'MOBIKWIK', 'OLA_MONEY', 'OXIGEN_WALLET', 'JIOMONEY', 'CITIZEN', 'MOMOPAY', 'PAYTM', 'GLOBEPAY', 'GCASH', 'PAYMAYA', 'AEROPAY', 'QRIS', 'ERIP', 'PROVINCIA_NET', 'BOLETO_PIX', 'TED', 'TBANC', 'SANTANDER', 'MACH', 'KHIPU', 'WEB_PAY', 'EXPRESS_DE_LIDER', 'LIDER', 'MULTICAJA', 'ACUENTA', 'BANCO_NATIONAL_CASH', 'BANCO_NATIONAL', 'PSE', 'EXITO', 'MOVIL_RED', 'PUNTO_RED', 'SUPER_GIROS', 'BANCO_GUAYAQUIL', 'BANCO_GUAYAQUIL_CASH', 'BANCO_PICHINCHA', 'BANCO_PICHINCHA_CASH', 'PICHINCHA_MI_VECINO', 'RED_ACTIVA', 'MI_COMISARIATO', 'PUNTO_EXPRESS', 'BANCO_DE_ANTIGUA', 'CAJA_DESARROLLO', 'FUNDACION_GENESIS', 'SUPER_EXPRESS', 'SERVICENTRO', 'TELEDOLAR', 'TIENDAS_PRONTO', 'QUICK_STOP', 'FDL', 'WESTERN_UNION', 'SCOTIABANK', 'SCOTIABANK_CASH', 'BBVA_BANCOMER_CASH', 'WALMART', 'HSBC', 'HSBC_CASH', 'BANCO_AZTECA', 'CALIMAX', 'COMERCIAL_MEXICANA', 'FARMACIAS_SANTA_MARIA', 'SORIANA', 'BCP', 'BCP_CASH', 'INTERBANK', 'INTERBANK_CASH', 'CAJA_HUANCAYO', 'CAJA_HUANCAYO_CASH', 'CAJA_AREQUIPA', 'CAJA_AREQUIPA_CASH', 'CAJA_TACNA', 'CAJA_TACNA_CASH', 'CAJA_TRUJILLO', 'CAJA_TRUJILLO_CASH', 'TAMBO', 'RIPLEY', 'SEVEN_ELEVEN', 'CIRCLE_K', 'FARMACIAS_DR_AHORRO', 'FARMACIA_BENAVIDES', 'SAMS_CLUB', 'BODEGA_AURRERA', 'SUPERAMA', 'BANK_LATAM', 'POLI', 'OVO', 'PRAXIS_CASHIER', 'TRUELAYER', 'VOLT', 'VERTUPAY_IB', 'VERTUPAY_QR', 'PRAXIS_CRYPTO_CASHIER', 'AZUPAY', 'LEAN', 'BANK_HONG_KONG_QR', 'BANK_HONG_KONG_VR', 'PIX', 'CRYPTO', 'MOBILE_PAYMENTS', 'MPESA']
PaymentOptionGroupType: ['RECOMMENDED', 'FAST', 'UNLIMITED', 'OTHER']
PaymentOrderChangeStatusReason: ['API_ISSUES', 'FINAL_NOTIFICATION_NOT_RECEIVED', 'TECH_ISSUE_WITH_THE_PSP', 'INSUFFICIENT_BALANCE_ON_OUR_SIDE', 'OTHER']
PaymentOrderChangeStatusRequestType: ['INITIATE', 'APPROVE', 'REJECT']
PaymentStatus: ['SUCCESSFUL', 'DECLINED', 'CANCELLED', 'BLOCKED', 'PROCESSING', 'APPROVAL', 'RETRY_TO_APPROVAL']
PaymentSystem: ['SAFECHARGE', 'ECOMMPAY', 'CAPITAL', 'INTERNAL', 'PAYPAL', 'SOFORT', 'WORLDPAY', 'TRUSTLY', 'BEPAID', 'CLEAR_JUNCTION', 'MODULR', 'WPAYMENTS', 'PIASTRIX', 'CHECKOUT', 'PRAXIS', 'ONE_ROAD_PAYMENTS', 'CITIZEN_PAYMENTS', 'CARDPAY', 'ALFABANK', 'AEROPAY', 'ERIP', 'PAY_RETAILERS', 'POLI', 'TOTALPROCESSING', 'TRANSACTX', 'EXACTLY', 'CREDORAX', 'MERCURYO', 'TRUELAYER', 'SKRILL', 'NETELLER', 'STRIPE', 'VOLT', 'PAY_COM', 'EQUALS_MONEY', 'OPENPAYD', 'AZUPAY', 'LHV', 'D24', 'LEAN', 'MY_FATOORAH', 'KLARPAY', 'NBD', 'ITEZ', 'ECOMMPAY_ITEZ', 'CIRCLE', 'WEBPAY', 'TRANSFERMATE', 'BANCASELLA', 'PAWAPAY', 'VERTUPAY', 'RBS', 'IFX', 'CBA', 'PARITET', 'BNB', 'BSB', 'CLEARBANK', 'FUZE', 'NETWORK_INT', 'SWIFFY', 'PAYMOB', 'DBS', 'IMBANK']
PaymentType: ['DEPOSIT', 'WITHDRAWAL', 'REFUND', 'AUTH', 'SETTLE', 'VOID', 'MPI', 'RETURN_DEPOSIT']
ProofStatus: ['NO', 'REQUESTED', 'IN_REVIEW', 'PROVIDED']
PropertiesLevel: ['MERCHANT', 'PSP', 'APPLICATION']
Status: ['NEW', 'OK', 'INREVIEW', 'INCOMPLETE', 'BLOCKED', 'TEMPORARILY_BLOCKED']
SupportedCardType: ['VISA', 'MASTERCARD', 'OTHER']
Tenant: ['CAPITAL_COM', 'CURRENCY_COM', 'SHARES_COM', 'INVESTMATE']
Theme: ['WHITE_MOBILE', 'DARK_MOBILE', 'WHITE_WEB', 'DARK_WEB']
TransactionStatus: ['PROCESSED', 'PENDING', 'DECLINED', 'CANCELLED', 'WAITING_FOR_SETTLEMENT', 'APPROVAL', 'PROCESSING', 'REFUND', 'VOID']
UserFilterType: ['ALL', 'TEST', 'REAL']
UserRole: ['GUEST', 'ANONYMOUS', 'USER', 'RETAIL_EXPERIENCED', 'PROFESSIONAL', 'RETAIL_LE', 'PROFESSIONAL_LE', 'ECP', 'PRODUCT']
UserStatus: ['NEW', 'REGISTERED', 'WAITING_FOR_VERIFICATION', 'VERIFIED', 'BLOCKED', 'CLOSED', 'SUSPENDED', 'DELETING', 'DELETED', 'CLOSED_BY_CLIENT', 'INACTIVE']
ValidationErrorCode: ['NOT_NULL', 'NOT_BLANK', 'NOT_BLANK_STRING', 'SIZE', 'DECIMAL_MIN', 'PATTERN', 'MATCH', 'COUNTRY_CODE', 'CREDIT_CARD_EXP_DATE', 'CREDIT_CARD_NUMBER', 'NOT_CREDIT_CARD_NUMBER', 'BANK_ACCOUNT_NUMBER', 'IBAN', 'BIK', 'SWIFT', 'BANK_CODE_NOT_SUPPORTED', 'SORT_CODE', 'IBAN_COUNTRY_MISMATCH']
WithdrawalAction: ['APPROVE', 'DECLINE', 'POSTPONE']
WithdrawalLimitLevel: ['NET_DEPOSIT_ONLY', 'NET_DEPOSIT_AND_PROFITS', 'NET_DEPOSIT_AND_EARNINGS']

--- Key Request Schemas (attack surface) ---
Traceback (most recent call last):
  File "<string>", line 15, in <module>
NameError: name 'schema' is not defined. Did you mean: 'schemas'?

--- Additional doc paths ---
  SPEC => /payment/docs/
  INTERNAL_SERVER_ERROR => /payment/docs/default
  INTERNAL_SERVER_ERROR => /payment/docs/all
  INTERNAL_SERVER_ERROR => /payment/docs/api
  INTERNAL_SERVER_ERROR => /payment/docs/rest
  INTERNAL_SERVER_ERROR => /payment/docs/swagger
  INTERNAL_SERVER_ERROR => /payment/docs/openapi
  INTERNAL_SERVER_ERROR => /payment/docs/v1
  INTERNAL_SERVER_ERROR => /payment/docs/v2
  INTERNAL_SERVER_ERROR => /payment/docs/v3
  INTERNAL_SERVER_ERROR => /payment/docs/webhooks
  INTERNAL_SERVER_ERROR => /payment/docs/notifications
  INTERNAL_SERVER_ERROR => /payment/docs/psp
  INTERNAL_SERVER_ERROR => /payment/docs/capital
  INTERNAL_SERVER_ERROR => /payment/docs/gateway
  INTERNAL_SERVER_ERROR => /payment/docs/admin
  INTERNAL_SERVER_ERROR => /payment/docs/internal
  INTERNAL_SERVER_ERROR => /payment/docs/deposit
  INTERNAL_SERVER_ERROR => /payment/docs/withdrawal
  INTERNAL_SERVER_ERROR => /payment/docs/crypto
  INTERNAL_SERVER_ERROR => /payment/docs/wire
  INTERNAL_SERVER_ERROR => /payment/docs/wallet
  INTERNAL_SERVER_ERROR => /payment/docs/lean
  INTERNAL_SERVER_ERROR => /payment/docs/truelayer
  INTERNAL_SERVER_ERROR => /payment/docs/transfermate
  INTERNAL_SERVER_ERROR => /payment/docs/vertupay
  INTERNAL_SERVER_ERROR => /payment/docs/commission
  INTERNAL_SERVER_ERROR => /payment/docs/routing
  INTERNAL_SERVER_ERROR => /payment/docs/cascade
  INTERNAL_SERVER_ERROR => /payment/docs/blacklist
  INTERNAL_SERVER_ERROR => /payment/docs/properties
  INTERNAL_SERVER_ERROR => /payment/docs/settings
  INTERNAL_SERVER_ERROR => /payment/docs/currency
  INTERNAL_SERVER_ERROR => /payment/docs/config

--- Key Request Schemas (attack surface) ---
DepositCardRequest:
  Required: ['sid']
  All fields: ['sid', 'cvv', 'cardToken', 'cardNumber', 'expDate', 'cardholder', 'commissionToken', 'dccToken', 'mismatchReason', 'contactByPhone', 'contactByEmail', 'skipPayment']

CreateWithdrawalRequest:
  Required: ['accountId', 'amount', 'country', 'currency', 'jurisdictions', 'license', 'tenant', 'userId']
  All fields: ['userId', 'accountId', 'amount', 'currency', 'paymentMethodId', 'license', 'tenant', 'country', 'jurisdictions', 'comment', 'internalWithdrawal']

WithdrawalRequest:
  Required: ['accountId', 'amount', 'paymentMethodId']
  All fields: ['paymentMethodId', 'accountId', 'amount', 'dccToken']

CryptoWithdrawalRequest:
  Required: ['accountId', 'address', 'amount', 'chain', 'paymentMethodId', 'verificationCode']
  All fields: ['paymentMethodId', 'accountId', 'amount', 'chain', 'address', 'verificationCode']

WireWithdrawalIbanRequest:
  Required: ['accountId', 'amount', 'beneficiaryCustomer', 'iban', 'swift']
  All fields: ['accountId', 'amount', 'beneficiaryCustomer', 'displayName', 'swift', 'iban']

WireWithdrawalUkRequest:
  Required: ['accountId', 'accountNumber', 'amount', 'beneficiaryCustomer', 'sortCode']
  All fields: ['accountId', 'amount', 'beneficiaryCustomer', 'displayName', 'sortCode', 'accountNumber']

CardPaymentMethodAddRequest:
  Required: ['accountId', 'cardNumber', 'cardholder', 'expDate']
  All fields: ['cardNumber', 'expDate', 'cardholder', 'displayName', 'accountId']

DepositFormDto:
  Required: ['sid']
  All fields: ['sid', 'cardToken', 'cardNumber', 'expDate', 'cardholder', 'cvv', 'saveCard', 'mismatchReason', 'contactByEmail', 'contactByPhone', 'skipPayment', 'browserDetails']

PaymentNotification:
  Required: []
  All fields: ['externalTxId', 'externalSystem', 'type', 'userId', 'accountId', 'tenant', 'amount', 'commission', 'currency', 'paymentMethod', 'paymentMethodId', 'paymentMethodDisplayName', 'status', 'note', 'timestamp', 'hidden', 'license', 'paymentNotificationTimestamp', 'params', 'details']

AdminSettingGlobalRequest:
  Required: ['id', 'key', 'value']
  All fields: ['id', 'key', 'test', 'licenses', 'countries', 'value']

PaymentOptionRoutingRule:
  Required: []
  All fields: ['id', 'license', 'paymentOption', 'currency', 'countries', 'bins', 'type', 'protectedRule', 'test', 'mpi', 'cardNetwork', 'cascade', 'bankDepositDetails', 'createdTimestamp', 'modifiedTimestamp', 'violationExceptions', 'priority']

PaymentOptionsConfig:
  Required: []
  All fields: ['id', 'paymentOption', 'license', 'externalSystem', 'test', 'mpi', 'currencies', 'supportedBinCountries', 'unsupportedBinCountries', 'supportedTransactionTypes', 'supportedSwifts', 'createdTimestamp', 'modifiedTimestamp', 'supportedTransactionTypesConfig', 'apiSupportedTransactionTypes', 'operatingEntities']

AddCryptoWhitelistAddressRequest:
  Required: ['accountId', 'address', 'chain']
  All fields: ['accountId', 'chain', 'address', 'walletType', 'vaspId', 'vaspName', 'vaspWebsite']

CreateVirtualIbanRequestDto:
  Required: []
  All fields: ['accountId', 'currency', 'bankCode']

LeanCreateWithdrawalPaymentMethodRequest:
  Required: ['accountId', 'iban']
  All fields: ['accountId', 'iban', 'displayName']

ApplePayDepositRequest:
  Required: ['displayName', 'paymentToken', 'sid']
  All fields: ['sid', 'merchantId', 'cardholder', 'displayName', 'paymentToken']

GooglePayDepositRequest:
  Required: ['paymentRequest', 'sid']
  All fields: ['sid', 'paymentRequest']

PayPalDepositRequest:
  Required: ['sid']
  All fields: ['sid', 'paymentMethodNonce', 'payerId', 'deviceData']

WirePaymentMethodAddIbanRequest:
  Required: ['beneficiaryCustomer', 'iban', 'license', 'swift']
  All fields: ['license', 'beneficiaryCustomer', 'displayName', 'swift', 'iban']

BankBlackListDto:
  Required: ['license', 'type', 'value']
  All fields: ['id', 'license', 'value', 'type', 'reasonCode', 'transactionType', 'createdTimestamp', 'modifiedTimestamp', 'binValueValid']

PaymentOptionCommissionConfig:
  Required: []
  All fields: ['id', 'paymentOption', 'type', 'test', 'licenses', 'countries', 'currencies', 'cardTypes', 'bankCountries', 'bins', 'data', 'createdTimestamp', 'modifiedTimestamp']


================================================================================
  END OF RECON
  Total endpoints: 329
  Total paths: 232
  Total schemas: 255
  License values: 33
  Tenant values: 4
  Payment systems: 63
  Payment options: 150+
  Operating entities: 20
================================================================================

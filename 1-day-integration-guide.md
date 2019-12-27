# web-sample
Sample of web integration

Web integration is composed of building a calculator on your premise and then launching an Iframe with the values user has put into calculator (you can also bypass the calculator step and launch an iframe outright)

You will need:
Required:
- placementID - a unique marker for each placement of a calculator. This will allow you to track how good the flow performs for every spot you place it at
Optional:
- placementSecret - used for some functions connected with order tracking
- clientOrderID - a unique GUID that you can assign to an order originating at your premise. This is used for tracking the order.

Steps:
1) Get available coins and rates
2) Get percisions and rounding rules
3) Calculate rate and use Iframe to place order
4) Track the order (optional)




* Sampe placementId 8e29e9b1-f3d0-4dd4-9920-08773ebcf0fa - use this for testing. 

1. Getting the rates
- call https://api.cexdirect.com/api/v1/payments/currencies/{placementId} 
You will receive array of currency pairs :
[
  {
    a: 0.0000990154556955063,
    b: 0.0005,
    c: 1,
    crypto: "BTC",
    cryptoPopularValues: ["0.01", "0.05", "0.1"], fiat: "USD",
    fiatPopularValues: ["100", "200", "500"] 
    },
  {
    a: 0.00010979641127605852,
    b: 0.0005,
    c: 1,
    crypto: "BTC",
    cryptoPopularValues: ["0.01", "0.05", "0.1"], fiat: "EUR",
    fiatPopularValues: ["100", "200", "500"]
  }, 
  {
    a: 0.003135035352303995,
    b: 0.001,
    c: 1,
    crypto: "BCH",
    cryptoPopularValues: ["0.01", "0.05", "0.1"], fiat: "USD",
    fiatPopularValues: ["100", "200", "500"] 
    }
 ];
 
Values a, b, c will be used to calculate the rate. You can use preset popular values to have an understanding of what users like to purchase in this currency pair.

2. Getting the percision and rounding rules

Use precisions to correctly round calculation and set min and max values.
- call https://api.cexdirect.com/api/v1/merchant/precisions/{placementId} 
[
  {
    currency: BCH",
    currencyPrecision: 8, maxLimit: "0",
    minLimit: "0.01",
    type: "crypto", 
    visiblePrecision: 4, 
    visibleRoundRule: "trunk"
  }, 
  {
    currency: "BTC",
    currencyPrecision: 8, 
    maxLimit: "0",
    minLimit: "0.01",
    type: "crypto", 
    visiblePrecision: 4, 
    visibleRoundRule: "trunk"
  }, 
  {
    currency: "USD", 
    currencyPrecision: 2, 
    maxLimit: "1000", 
    minLimit: "50",
    type: "fiat", 
    visiblePrecision: 2, 
    visibleRoundRule: "trunk"
  } 
]

3. Calculating and placing an order
To calculate rate, you should know the following things:
Use input value from user and insert it to these formulas using values a,b,c from step 1
For crypto, use - a * value(from user) - b / a
For fiat, use -   c * value(from user) + b / a

Round the result using the data from step 2
- visiblePrecision - number of characters after dot;
- visibleRoundRule - this is a rule how you should round your calculations:
"trunk" - just cut to the specified number; 
"bigger" - round up;
"smaller" - round down;
"dynamic" - the number of zeros entered by user is displayed

To invoke the flow on your page you should add this snippet:

<iframe allow="geolocation" frameborder="0" height="450px" width="600px"
src="https://api.cexdirect.com/?placementId=8e29e9b1-f3d0-4dd4-9920-08773ebcf0fa&calculatorValues={"fiatCurrency":"USD","fiatValue":"250","cryptoCurrency":"BTC","cryptoValue":"0.0240"}&wallet=1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2&clientOrderId=0cf8fc5d-6f03-4217-ab83-ccf0a316afac"
></iframe>

allow="geolocation" - this flag is optional and it gives an access for geolocation of user; 
height and width - you can control widget size on your page;
src - url of iframe with important params:
placementId - this is a required parameter, without it your page won't load;
calculatorValues - this is optional parameter, it decides which step to show first. Here you should pass values from your calculator:
fiatCurrency - selected fiat currency (USD, EUR);
fiatValue - amount of money;
cryptoCurrency - selected crypto currency (BTC, TRX, ETH and etc);
cryptoValue - amount of crypto;
wallet - optional crypto address. If nothing is passed, client will be able to enter it later.
clientOrderId - optional unique GUID, to get order status (if such clientOrderId already exists, flow will be crashed).

This will start the order flow for user within an iframe.

*Notice that, if you don't pass calculatorValues to iframe or calculatorValues data is invalid , widget will load its own calculator. If calculatorValues is valid, widget will move users to step of filling email

4. Tracking the order (optional)

If you have a need to track the order within your system (for example, find out when crypto has been sent, or keep user up-to-date within your application) you may use the following:

  // placementId =  "8e29e9b1-f3d0-4dd4-9920-08773ebcf0fa"
  // placementSecret =  "af087b108d1bc50735951f81a4d39126"
const crypto = require('crypto');
const nonce = new Date().getTime();

Generate a signature:

function generateSignature(params) 
  { 
    const hash = crypto.createHash("sha512"); 
    params.forEach(value => hash.update(value)); 
    return hash.digest("hex").toUpperCase();
  }

Run a request:
{
const signature = generateSignature([placementId, placementSecret, nonce.toString()]);
  //POST https://api.cexdirect.com/api/v1/orders/getOrderInfo
  const req = {
    "serviceData":
      {
        "placementId": "<< placementId >>", 
        "deviceFingerprint": "deviceFingerprint", 
        "signature": signature,
        "signatureType": "msignature512", 
        "nonce": nonce
      }, 
     "data": 
      {
        "clientOrderId": "<< clientOrderId >>" 
      }
}


Success reponse exanple would be:
  
const res = 
{
  code: 200,
  nonce: 1574339717108,
  data:
  {
    placementId: '77777777-7777-7777-7777-777777777777', 
    status: 'settled',
    userEmail: 'test@test.com',
    createdAt: 1574246991678,
    orderId: '332d1824-db60-4456-9ee4-99323f47167a', 
    clientOrderId: '346682a2-cafe-4aeb-8eed-2c8d93187207'
  } 
};

Possible status values:

"new" - Should appear once an order is created (after Email/Country are filled in)
"verification-ready" - After Next on Verification
"verification-in-progress" - Shown very quickly when verification is in progress. Transition status. 
"verification-rejected" - User was rejected by verification process
"verification-failed" - When verification related services are down
"processing-acknowledge" - Additional data is requested from user
"processing-ready" - Shown after additional data is sent
"processing-3ds" - 3DS sceen
"processing-rejected" - Payment was rejected by service provider
"processing-failed" - When processing related service is down
"refund-in-progress" - Transition status after fail in processing, if user was charged
"refunded" - In case of successful refund
"email-confirmation" - Should be updated automatically if user did it earlie 
"crypto-sending" Should be shown on transaction check with loading tx id
"crypto-sent" Should come with tx id. 
"settled" - Fees are settled 
"crashed" - When there is need in manual actions:
 • Refund Failed 
 • Settlement Failed and additionally:
 • Exchange Rate is changed more than allowed percent

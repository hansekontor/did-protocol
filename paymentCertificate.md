# DID:CERT - Payment Certificate Specification
Version: v0.0.1
Date Published: May 4, 2023

## Purpose
The [BUX API](https://github.com/bux-digital/documentation/blob/main/merchant-server-api.md) allows to create permissionless eToken invoices in BUX. Merchants have been onboarded via Affiliates who had to run their own relay servers to add themselves as transaction outputs before relaying the payment request to a BIP-70 server. [JSON Web Token](https://github.com/bux-digital/documentation/blob/main/relay-jwt.md) had to be used to control access to servers as well as reliably calculating fees before the payment request had been made. Both, running relay servers and managing JWTs, are tedious points of friction in the onboarding process. Payment Certificates are intended to skip infrastructure requirements for Affiliates while still ensuring reliability in the outputs of Multi-Recipient-Payments. Using the [DID-Protocol](https://github.com/hansekontor/did-protocol/blob/main/did-cert.md), a Payment Certificate is a template to create a Multi-Recipient-Payment that is usable by a Merchant requesting a BIP-70 server directly. Since the certificate is on-chain it becomes possible for a third party to validate actual transactions according to the Merchant's expectations and thus the process of IPN validation can be simplified. 

## Protocol
1. Payment Certificate is ordered based on shares that the parties of the payment have agreed on in combination with their addresses. The certificate must contain the pro-rata amounts anticipating possible BIP-70 fees so that payment requests using a certificate and an amount will always create an invoice for exactly the requested amount in BUX. 
2. The certificate is broadcasted by a known/trusted address according to the protocol below. 
3. A payment request is made using at a minimum: 
    * the transaction hash of the broadcasted certificate,
    * and the amount in BUX that all the shares in the certificate will be multiplied with
4. After the payment request has been fulfilled, the Merchant receives an IPN from a trusted origin. The Merchant, or anyone else, can now validate the on-chain transaction with three necessary properties:
    * payment transaction hash
    * expected payment amount
    * certificate transaction hash

## Certificate Transaction Details
The claim within the OP_RETURN script contains the payment shares, the recipients of these shares are added as outputs after the OP_RETURN statements.

**Transaction Details**
<table>
<tr>
  <td><b>v<sub>out</sub></b></td>
  <td><b>ScriptPubKey ("Address")</b></td>
  <td><b>sats<br/>amount</b></td>
</tr>
  <tr>
    <td>0</td>
   <td>
   OP_RETURN<br/>
   &lt;lokad_id: 'DID\x00'&gt; (4 bytes, ascii)<br/>
   &lt;method: 'cert'&gt; (4 bytes, ascii)<br/>
   &lt;operation_code: 'C'&gt; (1 byte, ascii)<br/>
   &lt;credential_code: 'paym'&gt; (4 bytes, ascii)<br/>
   &lt;expiration_block: 0&gt; (4 bytes integer)<br/>
   &lt;claims&gt; (0 to 188 bytes)<br/>
   </td>
    <td>typically 0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>payment recipient 1</td>
    <td>546</td>
  </tr>
  <tr>
    <td>2</td>
    <td>payment recipient 2</td>
    <td>546</td>
  </tr>
  <tr>
    <td>3</td>
    <td>other payment recipients...</td>
    <td>546</td>
  </tr>
  <tr>
    <td>...</td>
    <td>any</td>
    <td>any</td>
  </tr>
</table> 

**Credential Details**
<table>
 <tr>
  <td><b>Property</b></td>
  <td><b>Syntax</b></td>
 </tr>
 <tr>
  <td>Credential Type</td>
  <td>PaymentCertificate</td>
 </tr>
 <tr>
  <td>Credential Type Code</td>
  <td>paym</td>
 </tr>
 <tr>
  <td>Claims</td>
  <td>{"shares":[0.60, 0.35]}</td>
 </tr>
</table>

## Certificate Example
1. Two parties have agreed on a split of 80% and 20%. Their BIP-70 server takes fees of 5% of the total amount requested. The two fee shares in the certificate need to be divided by 1.05 so that the amounts will end up as requested after fee addition. The calculation below can be done automatically before the certificate order is confirmed.

```
share_1 = 0.8
share_2 = 0.2

server_fee = 0.05

cert_share_1 = share_1 / (1 + server_fee) = 0.761905
cert_share_2 = share_2 / (1 + server_fee) = 0.190476

claim = "{"amt":[cert_share_1, cert_share_2]}" = "{"amt":[0.761905,0.190476]}"
```

2. The transaction is broadcasted: 
```javascript
const VC = require('ecash-did-cert');

const paymentOptions = {
    issuerAddress: "ecash:qzwaq0yyqc3t6zqm75eqtjz5h3jrzztka5e7ne58nx",
    credentialTypeCode: "paym",
    claims: {
        shares: [ 0.761905, 0.190476 ] 
    },
    expirationBlock: 0
};
const paymentVC = new VC(paymentOptions);
const paymentScript = paymentVC.buildCreationScript();
console.log(paymentScript.raw.toString('hex'));
```
The resulting OP_RETURN script is: 
```
6a046469640004636572740143047061796d04000000001e7b22736861726573223a5b302e3736313930352c302e3139303437365d7d
```
While the two recipients of the specified shares must be the recipients of 546 sats in `vout1`and `vout2` as in [this transaction](https://explorer.be.cash/tx/3670c9a4a6f252d42d1359c6b78711e6036f0960dedcc519c4085deaf824e204).

3. A transaction can be created based on the certificate transaction hash and an amount specified by the Merchant. The amounts per recipient must be calculated by multiplying the certificate shares with the requested amount. The BIP-70 server will add fees according to the fees implicit in the share calculation of the certificate creation. In this scenario, the fee is 5% of the total certificate amounts that is added on top:

<table>
<tr>
  <td><b>v<sub>out</sub></b></td>
  <td><b>Recipient</b></td>
  <td><b>BUX<br/>amount</b></td>
</tr>
  <tr>
    <td>0</td>
   <td>
    SLP SEND script
   </td>
    <td>0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>ecash:qpr9erjh78uct7lc7m2f0ueq2dmnd535du54fvmttx</td>
    <td>76.1905</td>
  </tr>
  <tr>
    <td>2</td>
    <td>ecash:qpguf78k4rllxnar7fwy0rm9cjm77mlh4uneueus2c</td>
    <td>19.0476</td>
  </tr>
  <tr>
    <td>3</td>
    <td>BIP-70 Fees</td>
    <td>4.7619</td>
  </tr>
  <tr>
    <td>...</td>
    <td>any</td>
    <td>any</td>
  </tr>
</table> 
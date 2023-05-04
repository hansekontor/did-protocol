# DID Protocol on eCash
Version: v0.0.1
Date Published: May 4, 2023

A Protocol designed to use Decentralized Identifiers (DID) and Verifiable Credentials (VCs) on the eCash blockchain. 

## Purpose
DIDs are Uniform Resource Identifiers (URI) that can be used for digital identification of any subject using no intermediaries in the process. As presented in the abstract of the W3C [recommendation](https://www.w3.org/TR/2022/REC-did-core-20220719/): 

"In contrast to typical, federated identifiers, DIDs have been designed so that they may be decoupled from centralized registries, identity providers, and certificate authorities. Specifically, while other parties might be used to help enable the discovery of information related to a DID, the design enables the controller of a DID to prove control over it without requiring permission from any other party."

The eCash blockchain allows for a clean implementation of the W3C recommendations due to these features: 
- address-specific public-private key pairs render additional cryptographic infrastructure unnecessary
- transactions act as issuance of credentials
- the subject of the credential is able to "receive" it
- the OP_RETURN code allows for credential content with high degrees of detail

If wallets would collect Verifiable Credentials related to one address they could extend their functionality to become a Decentralized Identity Wallet (DID wallet).

## 1 Decentralized Identifiers
Decentralized Identifiers (DID) contain three parts separated by colons - the URI scheme identifier, the DID method identifier and the DID method-specific identifier. A basic DID looks like this: 

**did:cert:qqewa92l2pudd0dzlz9zmswyc528ccjmr5kksntqlh** 

This protocol is using the method 'cert'. The last part of the DID, the method-specific identifier could be set in various ways, for example using URL like syntax for paths and queries. In the eCash ecosystem, at least two possible types of method-specific-identifiers are easy to resolve: 
* eCash addresses 
* eCash transaction IDs
Thereby, the DID of a subject can be built with addresses and the DID of a Verifiable Credential can be built with the transaction ID of its issuance.

### 1.1 DID Documents
Per [recommendation](https://www.w3.org/TR/2022/REC-did-core-20220719/), DID Documents are defined as follows: 

"A set of data describing the DID subject, including mechanisms, such as cryptographic public keys, that the DID subject or a DID delegate can use to authenticate itself and prove its association with the DID."

Using eCash to bootstrap cryptographic infrastructure, DID Documents are not as relevant within the eCash chain as no layer-2 network specificications or external proof mechanisms have to be specified. Nevertheless, inter-operability with other DID schemes or methods will require a specification of the general DID Document on eCash. 

A DID Document can contain various [properties](https://www.w3.org/TR/did-core/#core-properties) of which only the id is required per recommendation. Other properties are optional and not included in this current version of the protocol.
<table>
    <tr>
        <td>Property</td>
        <td>Syntax</td>
        <td>Description</td>
    </tr>
    <tr>
        <td>id</td>
        <td>did:{method}:{id}</td>
        <td>unique DID</td>
    </tr>
</table>

## 2 Verifiable Credentials
A [Verifiable Credential](https://www.w3.org/TR/vc-data-model/) (VC) is a certificate that contains claims and comes with verifiable proof by means of a digital signature. In this case, the digital signature is provided implicitly by broadcasting a valid transaction containing the VC. Issuers can verify their address variously, for example by providing it on an https website. The [W3 recommendation](https://www.w3.org/TR/vc-data-model/#basic-concepts) lists required and optional properties. All required and other properties used by this current version of the protocol are listed below.

<table>
  <tr>
    <td>Property</td>
    <td>Explanation</td>
    <td>Source</td>
  </tr>
  <tr>
    <td>@context</td>
    <td>link to W3C syntax</td>
    <td>W3C</td>
  </tr>
  <tr>
    <td>id</td>
    <td>DID of the credential itself</td>
    <td>transaction id of the issuance</td>  
  </tr>
  <tr>
    <td>type</td>
    <td>determines the use  & syntax for /claims</td>
    <td>OP_RETURN</td>
  </tr>
  <tr>  
    <td>credentialSubject</td>
    <td>contains properties /id, /claims and /expirationBlock</td>
    <td>-</td>
  </tr>
  <tr>
    <td>/id</td>
    <td>DID of credentialSubject</td>
    <td>eCash address of the subject without prefix</td>
  </tr>
  <tr>
    <td>/claims</td>
    <td>contains a VC's claims</td>
    <td>OP_RETURN</td>
  </tr>
  <tr>
    <td>/expirationBlock</td>
    <td>last block height before VC becomes invalid</td>
    <td>OP_RETURN</td>
  </tr>
  <tr>
    <td>issuer</td>
    <td>DID of the issuer</td>
    <td>eCash address of the sender without prefix</td>
  </tr>
  <tr>
    <td>issuanceDate</td>
    <td>issuance date and time of the VC</td>
    <td>block time of issuance transaction</td>
  </tr>
  </table>
  
Implementing the syntax above, properties of a VC could look like this:

```
"@context": ["https://www.w3.org/2018/credentials/v1"],
"id": "did:cert:58b63d54cafe97fc1d9c45a33c699468763ec7de1bb3f478b6929ac95b87d3c5",
"type": ["VerifiableCredential", "UsernameCredential"],
"issuer": "did:cert:qqp657igtzd0ws76ukftbhef533638670v3krtohyf",
"issuanceDate": 
"credentialSubject": {
    "id": "did:cert:qz92ejgtzd0wstr6qjjy6cef533635cxju8vuzeqye", 
    "claims": {
        username: "rolf3000"
    },
    "expirationBlock": "709200"
},
```
### 2.1 Creating a Verifiable Credential
Verifiable Credentials on eCash are issued by broadcasting a transaction. The CREATE operation script is specified as:

**Transaction outputs**
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
   &lt;credential_code&gt; (4 bytes, ascii)<br/>
   &lt;expiration_block&gt; (8 bytes integer)<br/>
   &lt;claims&gt; (0 to 188 bytes)<br/>
   </td>
    <td>typically 0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>credentialSubject</td>
    <td>546</td>
  </tr>
  <tr>
    <td>...</td>
    <td>any</td>
    <td>any</td>
  </tr>
</table> 

#### 2.1.1 operation_code
An operation code refers to the possible VC operations.
<table>
  <tr>
    <td><b>operation</b></td>    
    <td><b>operation_code</b></td>
  </tr>
  <tr>
    <td>CREATE</td>
    <td>C</td>
  </tr>
  <tr>
    <td>UPDATE</td>
    <td>U</td>
  </tr>
  <tr>
    <td>DELETE</td>
    <td>D</td>
  </tr>
</table>

#### 2.1.2 credential_code
The credential code is a 4-byte abbreviation that can refer to additional information related to the specific VC type. Credential codes express the credential type on-chain and indicate the syntax for the claims. Credential codes default to `0000` if not specified otherwise.

#### 2.1.3 expiration_block
The expiration of a VC is given as the last block that the VC is still valid. Expiration block can be set to `0` to mark indefinite validity.

#### 2.1.4 claims
There are two possibilities to add the same claim to the on-chain script. Either it can be added **with** the keys as a JSON object `{"key1":"value1","key2":"value2"}` or **without** the keys as JSON array `["value1", "value2"]`. The decision which notation should be used falls to the issuer. If the claim structure (the order of values in the JSON array) for a given credential code is known to wallets, recipients or verifiers, the keys could be ommitted which saves on-chain sapce. Keys could also be ommitted if they are supposed to be secret. 

### 2.2 Updating a Verifiable Credential
It might be necessary to update a Verifiable Credential's claims or expiration block. Instead of using CREATE to issue a new VC, UPDATE will allow to review a history of the VC for varying purposes. The issuance time of a VC is not changed by an UPDATE operation. The CREATE operation and the UPDATE operation can be merged together.

**Transaction outputs**:
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
   &lt;lokad_id: 'DID\x00'&gt; (4 bytes, ascii)<sup>1</sup><br/>
   &lt;method: 'cert'&gt; (4 bytes, ascii)<br/>
   &lt;operation_code: 'U'&gt; (1 byte, ascii)<br/>
   &lt;credential_code&gt; (4 bytes, ascii)<br/>
   &lt;reference_id&gt; (4 bytes, integer)<br/>
   &lt;expiration_block&gt; (8 byte integer)<br/>
   &lt;claims&gt; (0 to 188 bytes)<br/>
   </td>
    <td>typically 0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>credentialSubject</td>
    <td>546</td>
  </tr>
    <tr>
    <td>...</td>
    <td>any</td>
    <td>any</td>
  </tr>
</table> 

The two operations UPDATE and DELETE exist in relation to a previous CREATE. This issuance must be referenced by a `reference__ID`. As this reference only needs to be defined after a VC has been issued, the first four bytes of the issuance transaction hash can be used as a natural uid.

## 2.3 Deleting a Verifiable Credential
The DELETE operation is necessary when a VC needs to be made invalid before its expiration block has been reached.

**Transaction outputs**:
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
   &lt;lokad_id: 'DID\x00'&gt; (4 bytes, ascii)<sup>1</sup><br/>
   &lt;method: 'cert'&gt; (4 bytes, ascii)<br/>
   &lt;operation_code:'D'&gt; (1 byte, ascii)<br/>
   &lt;credential_code&gt; (4 bytes, ascii)<br/>
   &lt;reference_id&gt; (4 bytes, integer)<br/>
   </td>
    <td>typically 0</td>
  </tr>
  <tr>
    <td>1</td>
    <td>credentialSubject</td>
    <td>546</td>
  </tr>
    <tr>
    <td>...</td>
    <td>any</td>
    <td>any</td>
  </tr>
</table> 

## 3 Resolving Verifiable Credentials
The resolution of a VC follows the process: using the relevant transaction hash to read the OP_RETURN script and parsing it. If the claims have been set in value notation, the resolver has to compare with a list of VC templates if the claimKeys are known. If not, the default resolution convention would `["value0", "value1"]` resolve to:
```
credentialSubject: {
    id: "did:cert:[subjectAddress], 
    claims: {
        claim0: "value0",
        claim1: "value1"
    }
}
```
Wallets, Verifiers or other Agents can keep a list of VC data as simple as this:
```
{
    "code": "usnm", 
    "name": "UsernameCredential", 
    "claimKeys": ["username"]
},
{
    "code": "tokn", 
    "name": "TokenCredential", 
    "claimKeys": ["id", "name", "ticker", "status"]
}
```
In addition to that it is recommended to also keep an address list of trusted credential issuers.

Since the DID syntax allows for paths, it is useful to add credential paths to the base DID such that **did:cert:qqewa92l2pudd0dzlz9zmswyc528ccjmr5kksntqlh/UsernameCredential** leads to the most recently VC issued credential of the specified credential type.

## 4 Examples
### 4.1 Usernames
A unique username could be assigned to an eCash address. This username could be used to register at websites by providing a digital signature or could simply act as a nickname in transactions. 
```javascript
const VC = require('ecash-did-cert');

const usernameOptions = {
    issuerAddress: "ecash:qzwaq0yyqc3t6zqm75eqtjz5h3jrzztka5e7ne58nx",
    subjectAddress: "ecash:qpr9erjh78uct7lc7m2f0ueq2dmnd535du54fvmttx",
    type: "UsernameCredential",
    credentialTypeCode: "usnm",
    claims: {
        username: "paul4"
    },
    expirationBlock: 987654
};
const usernameVC = new VC(usernameOptions);
console.log("usernameVC", usernameVC);

const usernameScript = usernameVC.buildCreationScript();
console.log("OP_RETURN script", usernameScript.raw.toString('hex'));
```
Running this, will result in an OP_RETURN script, that should be added to the first output of the transaction.
```
6a0464696400046365727401430475736e6d0800000000000f1206147b22757365726e616d65223a227061756c34227d
```
The second output of the transaction should send 546 sats to the `subjectAddress`.

### 4.2 Payment Certificates


## 5 References

[W3 DID Model](https://www.w3.org/TR/2022/REC-did-core-20220719/)

[W3 VC Data Model](https://www.w3.org/TR/vc-data-model/)

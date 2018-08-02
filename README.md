# Open Keystore Protocol
Open Keystore(oks) is a proposed API standard to allow secure remote access 
to private keys in an Ethereum wallet.
This standard does not replace an Ethereum RPC node. Rather, it allows other
types of servers to store encrypted keys on behalf of web and mobile light
clients.

**This is a draft proposal, and subject to change. The API should not be
considered stable at this time.**


## API Documentation
The API is deliberately kept lightweight and easy to implement. A great deal
of functionality is not dictated by the spec and is left to the specific
implementation.

### General
API responses SHOULD be well-formed JSON. The appropriate headers for a JSON
REST API service SHOULD be set on all responses. Specifically, responses SHOULD
have the `Content-Type: application/json` header.

The server SHOULD use TLS and the https:// schema whenever possible.
The server SHOULD set appropriate *Access-Control-Allow-Origin* headers to allow
cross-domain access for Javascript applications.

All API endpoints are REQUIRED. The API does not need to be in a top-level directory.
For example, `GET https://www.example.com/open-keystore/v1/accounts` is a valid request.

### Authentication
All methods except for `GET /info` SHOULD require authentication. 
All authentication is done through server-set HTTP cookies. 

The server SHOULD set *Secure* and *HttpOnly* flags on authentication cookies.

Authentication is out of scope for this protocol. The provider server MAY use
any authentication and authorization strategy for access permissions to the
protected endpoints.

For example, a development keystore server running on localhost may not require
any authorization. Conversely, an enterprise identity server may require SSO and
multi-factor authentication.

### GET /info

This endpoint provides discovery and general metadata about the keystore
provider, allowing automatic configuration for clients.

#### Response Fields
**version** string, OPTIONAL. The version of the Open Keystore protocol this
service supports. Valid versions are: `1.0`. If this field is missing, defaults
to `1.0`.
**providerName** string, OPTIONAL. A friendly name for the keystore provider.
**loginUrl** string, OPTIONAL. If the provider requires authentication, a
well-formed URL to redirect the user to for authentication. Client software MUST
redirect the user to the server-provided loginUrl if requests to the other
endpoints result in authentication failures.

#### HTTP Response Codes
**200 OK** Success

#### Example response
```json
{
 "version": "1.0",
 "providerName": "Acme", 
 "loginUrl": "https://example.com/login"
}
```


### GET /accounts

Returns an array of Ethereum addresses in the given keystore. The server MUST
return all addresses in the given keystore, or an empty array if there are none.
The server SHOULD require authentication to access this endpoint.

Each item in the array MUST be a valid Ethereum address.

#### HTTP Response Codes
**200 OK** Success
**403 Forbidden** The user does not have permission to access this resource.
Typically, this means the user has not authenticated with the provider server. 
**500 Internal Server Error** Error loading accounts list.


#### Example response
```json
[
  "0xde0B295669a9FD93d5F28D9Ec85E40f4cb697BAe",
  "0x1234567890123456789012345678901234567891"
]
```

### GET /account/{address}

Returns the encrypted private key for the corresponding account in [**Ethereum
Key Store Version
3**](https://github.com/ethereum/wiki/wiki/Web3-Secret-Storage-Definition)
format. The server MUST NOT make any changes to the JSON response object if it
is already in valid keystore format.

The server MUST convert the address parameter to lowercase in order to match
both checksummed and non checksummed addresses.

#### HTTP Response Codes
**200 OK** Success
**400 Bad Request** The address parameter in the URL is missing or malformed.
**403 Forbidden** The user does not have permission to access this resource.
Typically, this means the user has not authenticated with the provider server. 
**404 Not Found**. The encrypted private key for the given address 
does not exist in the keystore, OR the user is authenticated, but does not have
permission to access the private key for the given account(for example, in a
view-only wallet). The server MAY choose to return this error instead of 403
Forbidden for unauthenticated users to prevent account enumeration attacks.
**500 Internal Server Error** Error loading keystore.



## Client Implementation
This section is advisory and not part of the official spec.
The following implementation flow is provided only as a rough guideline to
client logic. All logic is pseudocode.

1. User opens client login page.
2. User selects an Open Keystore provider URL in the client interface.
3. Client requests `/accounts`. 
If response code is 200, user is logged in. Proceed to step 5. 
Client parses reponse code and displays the appropriate error to the user, such
as "Not Logged In", "Rate Limit Exceeeded", etc.
4. Client prompts user to visit `loginUrl` and authenticate. After successful
  authentication, go to step 5.
5. Client requests `/accounts` and displays the user's accounts in its
   interface.
6. When the user needs to sign a transaction, client requests the account's
   encrypted keystore from `/accounts/{account}` and prompts the user for a
   password to decrypt it. Client proceeds to sign transaction with the
   decrypted private key.

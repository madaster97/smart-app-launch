### Profile Audience and Scope

This profile describes SMART's
[`client-confidential-asymmetric`](conformance.html) authentication mechanism.  It is intended
for SMART clients that can manage and sign assertions with asymmetric keys.
Specifically, this profile describes the registration-time metadata required for
a client using asymmetric keys, and the runtime process by which a client can
authenticate to an OAuth server's token endpoint. This profile can be
implemented by user-facing SMART apps in the context of the [SMART App Launch](app-launch.html)
flow or by [SMART Backend Services](backend-services.html) that
establish a connection with no user-facing authorization step.

#### Use this profile when the following conditions apply:

* The target FHIR authorization server supports SMART's `client-confidential-asymmetric` capability
* The client can manage asymmetric keys for authentication
* The client is able to protect a private key

*Note*: The FHIR specification includes a set of [security
considerations](http://hl7.org/fhir/security.html) including security, privacy,
and access control. These considerations apply to diverse use cases and provide
general guidance for choosing among security specifications for particular use
cases.

### Underlying Standards

* [HL7 FHIR RESTful API](http://www.hl7.org/fhir/http.html)
* [RFC5246, The Transport Layer Security Protocol, V1.2](https://tools.ietf.org/html/rfc5246)
* [RFC6749, The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
* [RFC7515, JSON Web Signature](https://tools.ietf.org/html/rfc7515)
* [RFC7517, JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.txt)
* [RFC7518, JSON Web Algorithms](https://tools.ietf.org/html/rfc7518)
* [RFC7519, JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
* [RFC7521, Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7521)
* [RFC7523, JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7523)
* [RFC7591, OAuth 2.0 Dynamic Client Registration Protocol](https://tools.ietf.org/html/rfc7591)

<a id="discovery-requirements"></a>

### Advertising server support for this profile
As described in the [Conformance section](conformance.html), a server advertises its support for SMART Confidential Clients with Asymmetric Keys by including the `client-confidential-asymmetric` capability at is `.well-known/smart-configuration` endpoint; configuration properties include `token_endpoint`, `scopes_supported`, `token_endpoint_auth_methods_supported` (with values that include `private_key_jwt`), and `token_endpoint_auth_signing_alg_values_supported` (with values that include at least one of `RS384`, `ES384`).

#### Example `.well-known/smart-configuration` Response
```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token_endpoint": "https://ehr.example.com/auth/token",
  "token_endpoint_auth_methods_supported": ["private_key_jwt"],
  "token_endpoint_auth_signing_alg_values_supported": ["RS384", "ES384"],
  "scopes_supported": ["system/*.rs"]
}
```

### Registering a client (communicting public keys)

Before a SMART client can run against a FHIR server, the client SHALL generate
or obtain an asymmetric key pair and SHALL register its public key set with that
FHIR server’s authorization service (referred to below as the "FHIR authorization server").
SMART does not require a
standards-based registration process, but we encourage FHIR service implementers to
consider using the [OAuth 2.0 Dynamic Client Registration
Protocol](https://tools.ietf.org/html/draft-ietf-oauth-dyn-reg).

No matter how a client registers with a FHIR authorization server, the
client SHALL register the **public key** that the
client will use to authenticate itself to the FHIR authorization server.  The public key SHALL
be conveyed to the FHIR authorization server in a JSON Web Key (JWK) structure presented within
a JWK Set, as defined in
[JSON Web Key Set (JWKS)](https://tools.ietf.org/html/rfc7517).  The client SHALL
protect the associated private key from unauthorized disclosure
and corruption.

For consistency in implementation, FHIR authorization servers SHALL support registration of client JWKs using both of the following techniques (clients SHALL choose a server-supported method at registration time):

  1. URL to JWK Set (strongly preferred). This URL communicates the TLS-protected
  endpoint where the client's public JWK Set can be found.
  This endpoint SHALL be accessible via TLS without client authentication or authorization. Advantages
  of this approach are that
  it allows a client to rotate its own keys by updating the hosted content at the
  JWK Set URL, assures that the public key used by the FHIR authorization server is current, and avoids the
  need for the FHIR authorization server to maintain and protect the JWK Set. The client SHOULD return a “Cache-Control” header in its JWKS response

  2. JWK Set directly (strongly discouraged). If a client cannot host the JWK
  Set at a TLS-protected URL, it MAY supply the JWK Set directly to the FHIR authorization server at
  registration time.  In this case, the FHIR authorization server SHALL protect the JWK Set from corruption,
  and SHOULD remind the client to send an update whenever the key set changes.  Conveying
  the JWK Set directly carries the limitation that it does not enable the client to
  rotate its keys in-band.  Including both the current and successor keys within the JWK Set
  helps counter this limitation.  However, this approach places increased responsibility
  on the FHIR authorization server for protecting the integrity of the key(s) over time, and denies the FHIR authorization server the
  opportunity to validate the currency and integrity of the key at the time it is used.  

The client SHALL be capable of generating a JSON Web Signature in accordance with [RFC7515](https://tools.ietf.org/html/rfc7515). The client SHALL support both `RS384` and `ES384` for the JSON Web Algorithm (JWA) header parameter as defined in [RFC7518](https://tools.ietf.org/html/rfc7518).
The FHIR authorization server SHALL be capable of validating signatures with at least one of `RS384` or `ES384`.
Over time, best practices for asymmetric signatures are likely to evolve. While this specification mandates a baseline of support clients and servers MAY support and use additional algorithms for signature validation.
As a reference, the signature algorithm discovery protocol `token_endpoint_auth_signing_alg_values_supported` property is defined in OpenID Connect as part of the [OAuth2 server metadata](https://tools.ietf.org/html/rfc8414).

No matter how a JWK Set is communicated to the FHIR authorization server, each JWK SHALL represent an
asymmetric key by including `kty` and `kid` properties, with content conveyed using
"bare key" properties (i.e., direct base64 encoding of key material as integer values).
This means that:

* For RSA public keys, each JWK SHALL include `n` and `e` values (modulus and exponent)
* For ECDSA public keys, each JWK SHALL include `crv`, `x`, and `y` values (curve,
x-coordinate, and y-coordinate, for EC keys)

Upon registration, the client SHALL be assigned a `client_id`, which the client SHALL use when
requesting an access token.


### Authenticating to the Token endpoint

This specification describes how a client authenticates using an asymmetric key, e.g., when requesting an access token during: [SMART App Launch](app-launch.html#step-5-access-token) or [SMART Backend Services](backend-services.html#step-3-access-token), authentication is based on the OAuth 2.0 client credentials flow, with a [JWT
assertion](https://tools.ietf.org/html/rfc7523) as the client's authentication mechanism.

To begin the exchange, the client SHALL use the [Transport Layer Security
(TLS) Protocol Version 1.2 (RFC5246)](https://tools.ietf.org/html/rfc5246) or a more recent version of TLS to
authenticate the identity of the FHIR authorization server and to establish an encrypted,
integrity-protected link for securing all exchanges between the client
and the FHIR authorization server's token endpoint.  All exchanges described herein between the client
and the FHIR server SHALL be secured using TLS V1.2 or a more recent version of TLS .

#### Request

Before a client can request an access token, it SHALL generate a
one-time-use JSON Web Token (JWT) that will be used to authenticate the client to
the FHIR authorization server. The authentication JWT SHALL include the
following claims, and SHALL be signed with the client's private
key (which SHOULD be an `RS384` or `ES384` signature). For a practical reference on JWT, as well as debugging
tools and client libraries, see [https://jwt.io](https://jwt.io).

<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Header Values</th>
  </thead>
  <tbody>
    <tr>
      <td><code>alg</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The JWA algorithm (e.g., <code>RS384</code>, <code>ES384</code>) used for signing the authentication JWT.
      </td>
    </tr>
    <tr>
      <td><code>kid</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The identifier of the key-pair used to sign this JWT. This identifier SHALL
          be unique within the client's JWK Set.</td>
    </tr>
    <tr>
      <td><code>typ</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>JWT</code>.</td>
    </tr>
    <tr>
      <td><code>jku</code></td>
      <td><span class="label label-info">optional</span></td>
      <td>The TLS-protected URL to the JWK Set containing the public key(s) accessible without authentication or authorization. When present, this SHALL match the JWKS URL value that the client supplied to the FHIR authorization server at client registration time. When absent, the FHIR authorization server SHOULD fall back on the JWK Set URL or the JWK Set supplied at registration time. See <a href="#signature-verification">Signature Verification</a> for details.</td>
    </tr>
  </tbody>
</table>


<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Claims</th>
  </thead>
  <tbody>
    <tr>
      <td><code>iss</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Issuer of the JWT -- the client's <code>client_id</code>, as determined during registration with the FHIR authorization server
        (note that this is the same as the value for the <code>sub</code> claim)</td>
    </tr>
    <tr>
      <td><code>sub</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The client's <code>client_id</code>, as determined during registration with the FHIR authorization server
      (note that this is the same as the value for the <code>iss</code> claim)</td>
    </tr>
    <tr>
      <td><code>aud</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The FHIR authorization server's "token URL" (the same URL to which this authentication JWT will be posted -- see below)</td>
    </tr>
    <tr>
      <td><code>exp</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Expiration time integer for this authentication JWT, expressed in seconds since the "Epoch" (1970-01-01T00:00:00Z UTC). This time SHALL be no more than five minutes in the future.</td>
    </tr>
    <tr>
      <td><code>jti</code></td>
      <td><span class="label label-success">required</span></td>
      <td>A nonce string value that uniquely identifies this authentication JWT.</td>
    </tr>
  </tbody>
</table>

After generating an authentication JWT, the client requests an access token following either the [SMART App Launch](app-launch.html#step-5-access-token) or the [SMART Backend Services](backend-services.html#step-3-access-token) specification.  Authentication details are conveyed using the following additional properties on the token request:

<table class="table">
  <thead>
    <th colspan="3">Parameters</th>
  </thead>
  <tbody>
    <tr>
      <td><code>client_assertion_type</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>urn:ietf:params:oauth:client-assertion-type:jwt-bearer</code></td>
    </tr>
    <tr>
      <td><code>client_assertion</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Signed authentication JWT value (see above)</td>
    </tr>
  </tbody>
</table>

#### Response

##### Signature Verification

The FHIR authorization server SHALL validate the JWT according to the
processing requirements defined in [Section 3 of RFC7523](https://tools.ietf.org/html/rfc7523#section-3) including validation of the signature on the JWT.

In addition, the authentication server SHALL:
* check that the `jti` value has not been previously encountered for the given `iss` within the maximum allowed authentication JWT lifetime (e.g., 5 minutes). This check prevents replay attacks.
* ensure that the `client_id` provided is known and matches the JWT's `iss` claim.

To resolve a key to verify signatures, a FHIR authorization server SHALL follow this algorithm:

<ol>
  <li>If the <code>jku</code> header is present, verify that the <code>jku</code> is whitelisted (i.e., that it
    matches the JWKS URL value supplied at registration time for the specified <code>client_id</code>).
    <ol type="a">
      <li>If the <code>jku</code> header is not whitelisted, the signature verification fails.</li>
      <li>If the <code>jku</code> header is whitelisted, create a set of potential keys by dereferencing the <code>jku</code> URL. Proceed to step 3.</li>
    </ol>
  </li>
  <li>If the <code>jku</code> header is absent, create a set of potential key sources consisting of all keys found in the registration-time JWKS or found by dereferencing the registration-time JWK Set URL. Proceed to step 3.</li>
  <li>Identify a set of candidate keys by filtering the potential keys to identify the single key where the <code>kid</code> matches the value supplied in the client's JWT header, and the <code>kty</code> is consistent with the signature algorithm supplied in the client's JWT header (e.g., <code>RSA</code> for a JWT using an RSA-based signature, or <code>EC</code> for a JWT using an EC-based signature). If no keys match, or more than one key matches, the verification fails.</li>
  <li>Attempt to verify the JWK using the key identified in step 3.</li>
</ol>

To retrieve the keys from a JWKS URL in step 1 or step 2, a FHIR authorization server issues a HTTP GET request that URL to obtain a JWKS response. For example, if a client has registered a JWKS URL of https://client.example.com/path/to/jwks.json, the server retrieves the client's JWKS with a GET request for that URL, including a header of `Accept: application/json`.

If an error is encountered during the authentication process, the server SHALL
respond with an `invalid_client` error as defined by
the [OAuth 2.0 specification](https://tools.ietf.org/html/rfc6749#section-5.2).

* The FHIR authorization server SHALL NOT cache a JWKS for longer than the client's cache-control header indicates.
* The FHIR authorization server SHOULD cache a client's JWK Set according to the client's cache-control header; it doesn't need to retrieve it anew every time. 

Processing of the access token request proceeds according to either the [SMART App Launch](app-launch.html#step-5-access-token) or the [SMART Backend Services](backend-services.html#step-3-access-token) specification.

### Worked example

Assume that a "bilirubin result monitoring service" client has registered with a FHIR authorization server whose token endpoint is at "https://authorize.smarthealthit.org/token", establishing the following

 * JWT "issuer" URL: `https://bili-monitor.example.com`
 * OAuth2 `client_id`: `https://bili-monitor.example.com`
 * JWK identfier: `kid` value (see [example JWK](RS384.public.json))

The client protects its private key from unauthorized access, use, and modification.  

At runtime, when the bilirubin monitoring service needs to authenticate to the token endpoint, it generates a one-time-use authentication JWT.

**JWT Headers:**

```
{
  "typ": "JWT",
  "alg": "RS384",
  "kid": "eee9f17a3b598fd86417a980b591fbe6"
}
```

**JWT Payload:**

```
{
  "iss": "https://bili-monitor.example.com",
  "sub": "https://bili-monitor.example.com",
  "aud": "https://authorize.smarthealthit.org/token",
  "exp": 1422568860,
  "jti": "random-non-reusable-jwt-id-123"
}
```

Using the client's RSA private key, with SHA-384 hashing (as specified for
an `RS384` algorithm (`alg`) parameter value in RFC7518), the signed token
value is:

```
eyJhbGciOiJSUzM4NCIsImtpZCI6ImVlZTlmMTdhM2I1OThmZDg2NDE3YTk4MGI1OTFmYmU2IiwidHlwIjoiSldUIn0.eyJpc3MiOiJodHRwczovL2JpbGktbW9uaXRvci5leGFtcGxlLmNvbSIsInN1YiI6Imh0dHBzOi8vYmlsaS1tb25pdG9yLmV4YW1wbGUuY29tIiwiYXVkIjoiaHR0cHM6Ly9hdXRob3JpemUuc21hcnRoZWFsdGhpdC5vcmcvdG9rZW4iLCJleHAiOjE0MjI1Njg4NjAsImp0aSI6InJhbmRvbS1ub24tcmV1c2FibGUtand0LWlkLTEyMyJ9.D5kAqNJwaftCqsRdVVQDq6dMBxuGFOF5svQJuXbcYp-oEyg5qOwK9ZE5cGLTHxqwfpUPNzRKgVdIGuhawAA-8g0s1nKQae8CuKs33hhKh4J34xSEwW3MYs1gwI4GHTtR_g3kYSX6QCi14Ed3GIAvYFgqRqt-gD7sewMUXL4SB8I8cXcDbCqVizm7uPVhjw6QaeKZygJJ_AVLhM4Xs9LTy4HAhdCHpN0FrNmCerUIYJvHDpcod7A0jDmxdoeW1KIBYlhdhQNwjtsTvT1ce4qacN_3KIv_fIzCKLIgDv9eWxkjAtxOmIm8aW5gX9xX7X0nbd0QglIyiic_bZVNNEh0kg
```

Note: to inspect this example JWT, you can visit https://jwt.io. Paste the signed
JWT value above into the "Encoded"  field, and paste the [sample public signing key](RS384.public.json) (starting with the `{"kty": "RSA"` JSON object, and excluding the `{ "keys": [` JWK Set wrapping array) into the "Public Key" box.
The plaintext JWT will be displayed in the "Decoded:Payload"  field, and a "Signature Verified" message will appear.

For a complete code example demonstrating how to generate this assertion, see: [rendered Jupyter Notebook](authorization-example-jwks-and-signatures.html), [source .ipynb file](authorization-example-jwks-and-signatures.ipynb).


#### Requesting an Access Token

A `client_assertion` generated in this fashion can be used to request an access token as part of a SMART App Launch authorization flow, or as part of a SMART Backend Services authorization flow. See complete example:

* SMART App Launch: [specification](app-launch.html); [full example](example-app-launch-asymmetric-auth.html#step-5-access-token)
* SMART Backend Services: [specification](backend-services.html); [full example](example-backend-services.html#step-3-access-token)

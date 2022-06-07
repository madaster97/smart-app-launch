This page outlines a protocol for [public applications](app-launch.html#support-for-public-and-confidential-apps) to request persistent API authorization on behalf of a user or patient. It differs from the [UDAP specification](https://www.udap.org/) in that it allows a purely public app (one without any static credential) to request persistant API access.

### Background
With the existing public app workflow, the app is not given a refresh token or any other way to request new access tokens. This means the app is limited to using APIs for the time it's first access token is valid.

This is by design, given that public apps are susceptible to various attacks such as Cross-Site Scripting (XSS). Granting these apps refresh tokens (and allowing them to use the tokens without authentication) opens up the possibility of refresh token theft. This idea of "\[persistent\] bearer tokens in the browser" is not a viable solution for sensitive use cases such as health applications because of this risk.

The only secure way for public apps to be granted persistent API access is to have the app's authorization tied to an un-extractable credential stored on the device the app is running on. There are different ways to acheive tie the authorization and credential together, such as [sender-constrained tokens](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-4.9.1.1.2) with Mutual TLS, but those protocols still rely on a registration step that is left undefined. They also do not build off of existing [asymmetric authentication](client-confidential-asymmetric.html) methods defined in SMART.

### Overview
The _SMART Trusted Dynamic Client Registration_ protocol defined here acheieves the necessary security of per-device credentials while building off of existing OAuth 2.0 workflows.

This protocol is a combination of:
1. The existing public app workflow, secured with PKCE and redirect URI validation
2. Trusted Dynamic Client Registration, which is secured with an initial access token from step 1
3. The JWT Bearer grant type, which has semantics for requesting access tokens issued on behalf of a specific user

### Step 0: The initial app registers with the EHR
The _SMART Trusted Dynamic Client Registration_ protocol requires an initial client_id for use with the public client profile. The EHR will also provision a `software_id` to the app for use in step 2 below. These 2 values must be 1:1, and can optionally be the same value. 

### Step 1: The app obtains an initial access token
The app initiates a normal SMART App Launch after registering as a public client with the EHR. 

The app requests the `system/DynamicClient.register` scope during the launch to indicate it would like to request persistent API access through this workflow.

If succesful, the app will now have an access token with the `system/DynamicClient.register` scope assigned.

### Step 2: The app registers a dynamic client instance
The app registers a new dynamic client instance using the initial access token from step 1 and the `software_id` from step 0.

The app first needs to create a new public-private key pair unique to this session. See [Managing On-Device Keys]()




### Manage On-Device Keys
- Manage your private key in hardware
- Use your initial access token immediately
- Don't re-use private keys


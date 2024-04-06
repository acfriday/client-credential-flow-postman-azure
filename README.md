# Client Credential Flow from OAuth 2.0

The Client Credentials Flow is useful for machine-to-machine communication where no user will be present. This flow involves an application exchanging its application credentials (client ID and client secret), for an access token. When the client app presents a token to a resource, the resource enforces that the app itself has authorization to perform the desired action since there is no user involved in the authentication.

## Prerequisites

1. **Microsoft Azure** - Ensure you have proper permissions to create resources and register applications for your tenant. The licenses I personally possess are `Office 365 E5` and `Enterprise Mobility + Security E5`.
2. **Postman API Platform** - This is a popular API development tool that can downloaded for free at its website https://www.getpostman.com. I'll be using the `64 bit` desktop version `v10.24.15`. We will utilize Postman to make requests to Azure's Microsoft Graph API.

## **The Components of OAuth 2.0**

<details><summary><b>Read Overview</b></summary>

#### **`Resources`**

* The digital assets or services the user grants access to via OAuth 2.0. Resources are hosted by Resource Servers, which require valid access tokens for data access.

#### **`Resource Owners`**

* Individuals or entities that have the authority to grant access to their resources. In most cases, the resource owner is the end-user.

#### **`Clients`**

* Applications requesting access to resources on behalf of the Resource Owner. Clients are authenticated by the Authorization Server and authorized by the Resource Owner to access specified resources.

#### **`Authorization Server`**

* The server that issues access tokens to clients after successfully authenticating the Resource Owner and obtaining authorization. It plays a critical role in the OAuth 2.0 security framework, ensuring that access to resources is granted only to clients with proper authorization from the Resource Owners.
* **Authorization Endpoint** `/auth` initiates the flow. Clients request this endpoint with parameters like `response_type=code`, `client_id`, `redirect_uri`, `scope`, `state`, and `code_challenge`.
* **Token Endpoint** `/token` exchanges the `authorization code` for tokens. The request includes `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, and `code_verifier`.
* **Userinfo Endpoint** `/userinfo` when accessed with an access token, returns `claims` about the authenticated user.

#### **`Tokens`** - Strings representing the granted permissions

* **Access Token**: Enables access to the user's data via the Authorization: `Bearer <token>` header in API requests.
* **Refresh Token**: Used to renew an access token via the token endpoint with `grant_type=refresh_token`, without the user's interaction.
* **ID tokens**: Issued by the authorization server to the client application. Clients use ID tokens when signing in users and to get basic information about them.

#### **`Grants`**

* **Authorization Code Grant**: Involves redirecting the user to the authorization endpoint, obtaining an authorization code, and exchanging the code for tokens at the token endpoint.
* **Client Credentials Grant**: Used for server-to-server communication where the application acts on its own behalf. Access is granted based on the authorization of the client, not the end user.
* **Resource Owner Password Credentials Grant**: Allows direct exchange of user credentials for access tokens. Recommended only for trusted clients, as it exposes the user's password.
* **Implicit Grant**: Optimized for clients implemented in a browser using a scripting language. Deprecated in OAuth 2.1 due to security vulnerabilities.

#### **`Scope`** - Defines the level of access the application requests

* Expressed in `space-delimited strings`, such as `scope=openid profile email`, determining which resources the application can access and actions it can perform.

#### **`Proof Key for Code Exchange (PKCE)`** - Enhances security for public clients

* Uses `code_challenge` and `code_challenge_method` during the authorization request, and `code_verifier` in the token exchange process to mitigate interception attacks.

#### **`OpenID Connect (OIDC)`** - An authentication and authorization layer built on top of OAuth 2.0, incompatible with OAuth 1.0

* Utilizes `ID Tokens`, returned along with the `access token`, containing claims about the authentication of the user.

</details>

## Microsoft Azure - Register a Web Application in Entra ID

<details><summary><b>Instructions</b></summary>

1. Sign in to the `Microsoft Entra admin center` as at least a `Cloud Application Administrator`.
2. Browse to `Identity` > `Applications` > `App registrations` and select `New Registration`.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/f1e15c54-9baa-4423-9825-a816996e5c67)

3. Enter a `Display Name` for your application.
4. We'll select the default `single-tenant` option.
5. Select Web as our platform with `http(s)://localhost` (excluding the parenthesis as seen below) as our redirect URI.
6. Complete this step by selecting `Register`.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/a175dbf2-b83a-4f3f-9dc9-e015e851634f)

</details>

## Microsoft Azure - Client ID, Client Secret, API permissions, Endpoint

<details><summary><b>Instructions</b></summary>

1. We'll need to note our app's `Client ID` from the Entra ID `Overview` tab under `App Registrations` for later use.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/619546f4-9ac6-414f-89e2-2d93349d03fd)

2. Next we'll retrieve our `Client Secret`. Select `Certificates & secrets` > `Client secrets` > `New client secret`.
Click `Add` to save your `Client Secret`. I chose the `default expiry time` after selecting `new client secret`,
we'll need to retrieve this secret again later.

    `I'll be deleting this Client Secret from my account before posting it publicly here.
    Important! Record the secret's value for later use This secret value
    is never displayed again after you leave this webpage.`
    
![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/c8e2a435-a0cf-4bc1-bde8-ebda4b717772)

3. Now let's check the `API permissions` tab. We'll need to apply an `application permission` as opposed to a `delegated permission` which requires user interaction to authenticate. The Microsoft Graph API `Service.Health.Read.All` is the permission we'll go with here for this demonstration.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/32e738d9-65b6-4e8c-a784-2a513a9a5c4c)

4. Make sure after adding the permission to `grant admin consent for {your_domain}` as we the user won't be providing consent ourselves when requesting access tokens with this flow.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/21948e56-7af5-4b3d-bd8e-8b9bb3cb1a38)

5. Finally we'll need to retrieve the Azure REST Endpoints we'll send out requests towards.
The Token Endpoint `OAuth 2.0 token endpoint (v2)` is the only one we'll copy from here.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/f719548c-1544-4fa5-84b2-8bf7367ba3e1)

</details>

## Postman - Request OAuth 2.0 Access Token

<details><summary><b>Instructions</b></summary>

1. In `Postman`, create a new `Request` and navigate to the `Authorization tab` and select `OAuth 2.0` as the auth `type`.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/178e0d76-bb13-47eb-9731-9ce3bfea64a0)

2. This is where we'll input data for the values below:

* Token Name: `Any name of your personal choice`
* Access Token URL: `Entra ID > App Registrations > your app > Overview > Endpoints`
* Grant Type: `Client Credentials`
* Client ID: `Entra ID > App Registrations > your app > Overview`
* Client Secret: `Entra ID > App Registrations > your app > Certificates & secrets`
* Scope: `https://graph.microsoft.com/.default`

  `Note! The Client Credential Flow must have a scope value with /.default suffixed to the resource identifier (application ID URI) you're attempting to access,
  For the Microsoft Graph API that value is: https://graph.microsoft.com/.default. This value informs the token endpoint to include within the access token all of the permissions we as the admin consented to for the app.`

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/9d91ce64-5c15-494d-bbdb-b4abc45e35c4)

3. Scroll to the bottom and click `Get New Access Token`.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/2d2407da-2700-49be-ac3e-7c4b4c3b2b06)

</details>

## Postman - Use The Retrieve Access Token

<details><summary><b>Instructions</b></summary>

1. After successfully authenticating you should receive the following acknowledgement, click `Proceed` here.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/22a0b245-4da1-447d-a26b-0eab0ca47b24)

2. scroll back to the top of `Token Details`, go ahead and `use` the `access token`.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/32d5b2e4-cae4-418f-8658-0bcb812ea1f6)

3. Scroll to the top of `Postman` after selecting `Use Token` to verify our `named` `access token` is being used in our upcoming request.

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/b1ce0c2c-14b1-4af7-942b-fe3273bdeeb3)

</details>

# Query and Response

### `Get serviceHealth`
### Namespace: `microsoft.graph`
* `Retrieve the properties and relationships of a serviceHealth object.
* This operation provides the health information of a specified service for a tenant.

### `Request Format`
GET /admin/serviceAnnouncement/healthOverviews/{ServiceName}

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/9255ce05-f641-4fb8-af96-30346bb8935b)

### `Response`

![image](https://github.com/acfriday/client-credential-flow-postman-azure/assets/82184168/b17d5cde-795e-49ce-b473-c224671086ab)

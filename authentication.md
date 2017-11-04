# Developer Centre
You can create an application in the Ingest Developer Centre (https://developers.ingest.io) where you sign in with your Ingest account and set up your App. From there you can set up your redirect URI and retrieve and generate your tokens.

# Authentication

```shell
  curl -i https://api.ingest.io/videos \
    -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQ..."
```

All authorization with Ingest for our API is accomplished with Bearer tokens. We support a few standard OAuth2 flows for building your client application. When building your integration with Ingest, you need to register your application with us. Your registered application is identified by a few different properties:

* Client Identifier - public identifier of your application
* Client Secret - secret key for your application
* Redirect URI - required for some authorization flows

There are two possible ways your application can interact with Ingest. It can either act on behalf of a user or as the application itself. Most interactions that modify data require that it happen from the context of a user. Our API endpoints document what context they can be called from, and what scopes are required to call them.

## Acting on a user's behalf

Ingest uses the OAuth2 [authorization code grant](https://tools.ietf.org/html/rfc6749#section-4.1) to issue access tokens on the behalf of users and properly federating the security of our user's account to your application.

![Authorization Grant Diagram](/images/auth-code-flow.png "Authorization Code Flow")

### Getting an Authorization Code

```shell
https://login.ingest.io/authorize?response_type=code&client_id=IngestNetwork123&scope=read_videos,write_videos&redirect_uri=https://mywebsite.com/oauth/ingest
```

The first step to obtaining a token is to send your users to the authorization endpoint.

`https://login.ingest.io/authorize`

#### Request Parameters

We accept these parameters in two different locations, either as query parameters from the URL, or in the request body with a content type of `application/x/www-form-urlencoded`.

Parameter     | Description
------------- | -----------
response_type | **(required)** Value should be `code`
client_id     | **(required)** Your unique identifier for your application
scope         | **(required)** Comma-separated list of scopes
state         | **(recommended)** Unique string passed back to you for verification to prevent CSRF
redirect_uri  | URL to redirect user to can be used to validate the stored redirect_uri for your application
network_id    | Ingest network ID for a shorter login flow for multi-network users

When a user arrives at the page, they are required to login if they don't already have a session active with Ingest. Once completing that process, they are shown your applications name, and the permissions you are requesting to operate with on their behalf. It is a best practice to request the least amount of privilege that your application requires because it is always possible to send a user back to ask for more when you deploy a new section to your application. The user is then given the choice to approve access, or reject. In both cases, we will send the result of this interaction to the URL specified as your applications `redirect_uri`.

#### Response Parameters

The data returned to your service will be returned as query parameters on the request that is sent to the `redirect_uri`. The parameters that will be returned for a successful authorization attempt are below.

| Parameter | Description                                                                                           |
| --------- | ------------------------------------------------------------------------------------------------------|
| `code`    | This is a one-time code that is valid and after used is no longer valid for your client application   |
| `state`   | If we were given a state value, we will return the exact value present on your request.               |

When a user rejects the authorization attempt, or some other error occurs, you will receive a response, in the same manner, however, the parameters are different.

| Parameter           | Description |
| ------------------- | ----------- |
| `error`             | Refer to the [Error Types](#auth-error-types) table for more information. |
| `error_description` | A short description of potential reasons you may be receiving the error, intended to help during the development of a client application. |

| Error Types                | |
| -------------------------- | -----------
| invalid_request            | The request is missing a required parameter or is otherwise malformed. |
| unauthorized_client        | The authorizing client is not able to request a token via this method. |
| access_denied              | The client rejected your request. |
| unsupported_response_type  | Ingest doesn't support retrieving a token using this method. |
| invalid_scope              | The requested scope is invalid, or malformed. |
| server_error               | We encountered an error trying to process your request. |
| temporarily_unavailable    | We aren't currently available, try again soon. |

#### Retrieving a token

```shell
curl "https://login.ingest.io/token?grant_type=authorization_code&client_id=IngestNetwork123&client_secret=jhjed88ydjKJDFU77y7dk&redirect_uri=https://mywebsite.com/oauth/ingest" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Accept: application/vnd.ingest.v1+json" \
  -X POST
```

`POST https://login.ingest.io/token`

This brings us to Step 4 in our flow diagram above. Should a user approve permission, you will receive a response containing a "code". This code is a one-time-use that you use to exchange for an access token scoped to that specific user's privilege level. To accomplish this, you send a POST request to the token endpoint.

The client_id and client_secret parameters below can also be sent in the `Authorization` header, in Basic encoding. The expected value would be `Basic base64_encode(client_id:client_secret)`.

#### Request Parameters

We accept these parameters in two different locations, either as query parameters from the URL, or in the request body with a content type of `application/x/www-form-urlencoded`.

Parameter     | Description
------------- | -----------
grant_type    | **(required)** Must be set to `authorization_code`
client_id     | **(required)** Your unique identifier for your application
client_secret | **(required)** Your secret client value to prove it's you
code          | This is the value returned to your service from the authorization step
scope         | The permissions you would like this token to have, it cannot have more permission than the application itself was given by the initializing user.
redirect_uri  | If this was included in the authorizing request, it MUST be included in the token access request

```json
{
  "access_token": "asdkjzuiuqw.sjxuqiopx;jspwie.sjdisjkaqwup;xjw",
  "token_type": "Bearer",
  "expires_in": "86400",
  "refresh_token": "thisisafakerefreshtokenvaluebutitisprettyclose",
  "scope": "read_videos,write_videos"
}
```

Assuming all goes well when sending your request to the token endpoint you will receive a JSON document containing your access token. We use [JSON Web Tokens](https://jwt.io), specifically JWS tokens for our access token format, and include a fair amount of information inside the token to make it easier for application developers to access information about the user.

For instance, your access token will contain a claim `sub`, which will be the UUID for the authorizing user. You don't need to store the user_id separately yourself. We recommend using the website [https://jwt.io](https://jwt.io) to see more parameters that are stored in the token.

##### Refreshing your token

```shell
curl "https://login.ingest.io/token?grant_type=refresh_token&refresh_token=asdaadfaf32fwoisfj&client_id=IngestNetwork123&client_secret=jhjed88ydjKJDFU77y7dk" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Accept: application/vnd.ingest.v1+json" \
  -X POST
```

`POST https://login.ingest.io/token`

The access token that you receive from us is short-lived. The exact lifetime can be determined from the `expires_in` value in the JSON document and a little bit of math, or you can use the `exp` claim in the JWT.

Utilizing either of the mechanisms above or waiting to receive an Unauthorized call from the API, you'll eventually want to get a new access token. The method to do this is to exchange the refresh token the authorization server gave you last time, for a new access token and a new refresh token. To accomplish this, you'll use the token endpoint again.

The `client_id` and `client_secret` parameters below can also be sent in the `Authorization` header, in Basic encoding. The expected value would be `Basic base64_encode(client_id:client_secret)`.

###### Request Parameters

We accept these parameters in two different locations, either as query parameters from the URL, or in the request body with a content type of `application/x/www-form-urlencoded`.

Parameter     | Description
------------- | -----------
grant_type    | **(required)** Must be set to `refresh_token`
refresh_token | **(required)** The refresh token issued when an access token is retrieved
client_id     | **(required)** Your unique identifier for your application
client_secret | **(required)** Your secret client value to prove it's you
scope         | **(optional)** The level of scope the new access token should have, it MUST NOT include any scope not originally granted, and if omitted will be equal to the original token

Once you've exchanged your client credentials for a token, you use it exactly the same way you would as a user access token. Calls which don't accept client credentials as they require a user context are indicated in the specific documentation for that API call.

#### Acting as an application

![Client Credentials Diagram](/images/client-credentials-flow.png "Client Credentials Flow")

It is possible to interact with the Ingest API as your application. This is done with the client credentials flow which allows an application to perform API interactions. The available calls that can be taken as an application are limited mostly to calls that are read-only. However, because the application is directly authenticating itself using the client\_id and client\_secret pair, it is much simpler as the diagram above illustrates.

##### Retrieving a token

```shell
curl "https://login.ingest.io/token?grant_type=client_credentials&client_id=IngestNetwork123&client_secret=jhjed88ydjKJDFU77y7dk" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Accept: application/vnd.ingest.v1+json" \
  -X POST
```

`POST https://login.ingest.io/token`

Retrieving an access token using client credentials is similar to the fourth step in the authorization credentials. You make a POST request to <https://login.ingest.io/token> with the following query parameters.

The `client_id` and `client_secret` parameters below can also be sent in the `Authorization` header, in Basic encoding. The expected value would be `Basic base64_encode(client_id:client_secret)`.

###### Request Parameters

We accept these parameters in two different locations, either as query parameters from the URL, or in the request body with a content type of `application/x/www-form-urlencoded`.

Parameter       | Description
--------------- | -----------
`grant_type`    | **(required)** Must be set to `client_credentials`
`client_id`     | **(required)** Your unique identifier for your application
`client_secret` | **(required)** Your secret client value to prove it's you
`scope`         | **(required)** A comma-separated string of scopes
`redirect_uri`  | **(optional)** Not necessary, but can be sent to verify the `redirect_uri` hasn't changed


The major differences with the client credential flow are that you not operating in a users context, and you never receive a refresh token as you have access to the authorizing credentials.

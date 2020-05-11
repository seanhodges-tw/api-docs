# Connected Apps Integration Guide

This guide describes how your application can connect to TransferWise customer accounts.
It also describes some common operations that are useful for application integrations. 


## API access

<aside class="notice">
Please note that asking TransferWise users to copy and paste their Transferwise API tokens directly into your
application is a violation of our security policy.
Only use the proper OAuth authorization flow described below.
</aside>

TransferWise uses the OAuth 2.0 protocol for authorization.
Once our partnership begins, we'll send you API access credentials for our sandbox environment.
These credentials consist of a *client ID* and a *client secret*.
The client ID and secret are required for an OAuth 2.0 authorization code-based account linking flow.
Later, you will receive a different set of client credentials for our live environment.

Your technical team will need to provide TransferWise with a *redirect URL* for the account linking flow.
This is the URL controlled by your system that we will forward users to after successfully granting your application
access to TransferWise accounts.
Specifying this URL explicitly makes the integration more secure.
We will only use redirect URLs registered with us when redirecting our users to your system.

<a href="https://auth0.com/docs/protocols/oauth2" target="_blank" rel="noopener noreferrer">This article about the OAuth 2.0 framework</a>
is a great way to refresh your knowledge about the protocol itself.


## Sandbox and live environments

* Our sandbox (test) API is located at https://api.sandbox.transferwise.tech

* Our live (production) API is located at https://api.transferwise.com


## User authorization

At a high level, there are five steps to gaining access to an existing TransferWise account:

1. Your application redirects the user to the TransferWise authorization page, which prompts them to log in.
2. The user logs in to TransferWise.
3. The user agrees to give your application access to one of their TransferWise profiles.
4. TransferWise redirects the user back to your preconfigured redirect URL, with an authorization code and selected profile ID appended as URL query parameters.
5. Your application exchanges the authorization code for API tokens.

These steps are explained in more detail below.


###  1. Your application redirects the user to TransferWise authorization page

Your website or app opens the following URL in the user's browser:

Sandbox:  

    `https://sandbox.transferwise.tech/oauth/authorize/?client_id={clientId}&redirect_uri={redirectUri}&state={state}`

Live:  
    
    `https://transferwise.com/oauth/authorize/?client_id={clientId}&redirect_uri={redirectUri}&state={state}`

List of available parameters:

Parameter | Description
--------- | -----------
client_id (required) | The client ID you received from us.
redirect_uri (required) | The preconfigured URL in your application where users will be sent after authorization.
state | An opaque value, used for security purposes. If this parameter is set in the request, then it is returned to the application as part of the redirect URL. <a href="https://auth0.com/docs/protocols/oauth2/oauth-state" target="_blank" rel="noopener noreferrer">More about state parameter</a>.

<aside class="notice">
You should not use "WebView" components in mobile apps to show the authorization page to the users.
They are not secure and will not allow users to log in to TransferWise using Google, which is an option used by our
users.
Your app should instead open the device's full browser application.
</aside>


### 2. The user logs in to TransferWise

Our usual login screens are presented to the user if they are not already logged in on the browser being used.
Users will be prompted to go through our two-factor authentication procedure if that is enabled for their account.


### 3. The user creates or selects a profile

The TransferWise user will create or select a profile that will be linked with your application.
TransferWise profiles are the personal or business entities which will be involved in money transfers and other
TransferWise activities.

Once the authorization process is complete, your app will be authorized to perform actions on the selected profile.
Access to other profiles associated with the TransferWise user must be separately authorized.


### 4. The user agrees to grant access and we forward them to your system

Once a user gives your application authorization to connect to TransferWise and access their data, they will be
redirected back to your *redirect URL* with some additional query parameters.

| Parameter | Description
| --------- | -----------
| code      | Authorization code used to complete the authorization code grant-based OAuth 2.0 flow
| state     | Any state parameter you initially passed to us when initiating the flow
| profileId | The profile ID that the TransferWise user granted you access to

An example, given the redirect URL `https://www.yourapp.com`:

    `https://www.yourapp.com/?code=ABCDEF&state=f6027a42-344d-4a4d-9f8a-39e42acf9887&profileId=12345`


### 5. Your application exchanges the authorization code for API tokens

Your website or service can then use the authorization code to obtain API tokens to act on behalf of the user
account as described in the [get user tokens](#connected-apps-integration-guide-get-user-tokens) section.

If you are building your TransferWise integration as a native mobile phone app then the redirect URL should be able to
handle returning the user to the correct place in the app.

*Please note that authorization codes expire after 30 minutes and are one time use only.*


### Error handling

When errors occur during authorization request handling, we display error details on our web pages.
The user may also see a link back to your application, with `error` and `error_description` parameters in the URL
instead of `code`. Your callback URL handler should be prepared to accept such "error" requests. 


## Get user tokens

Authorization codes can be used to obtain user *access* and *refresh* tokens.
These allow your system to make API calls on behalf of a TransferWise user.

*Access tokens* are short-lived API tokens used to access TransferWise customer API resources.

*Refresh tokens* are long-lived API tokens that are used to generate access tokens.

> Example Request:

```shell
curl \
-u '[your-api-client-id]:[your-api-client-secret]' \
-d 'grant_type=authorization_code' \
-d 'client_id=[your-api-client-id]' \
-d 'code=[code-from-redirect-uri]' \
-d 'redirect_uri=https://www.yourapp.com' \
'https://api.sandbox.transferwise.tech/oauth/token' 
```

You will be returned an access token and a refresh token.

> Example Response:

```json
  {
    "access_token":"ba8k9935-62f2-475a-60d8-6g45377b4062",
    "token_type":"bearer",
    "refresh_token":"a235uu9c-9azu-4o28-a1kn-e15500o151cx",
    "expires_in": 43199,
    "scope":"transfers"
  }
```

You need to include the user's access token in API requests using an `Authorization` HTTP request header: 
 
    `Authorization: Bearer <access token>`

Access tokens are short-lived for security reasons. They are only valid for 12 hours. 
When they expire, you need to use the refresh token to generate a new access token.

You must securely store the user's refresh token in order to generate new access tokens.
Refresh tokens (or your client credentials) should never be transmitted to any of your client applications.


### Request

**`POST https://api.sandbox.transferwise.tech/oauth/token`**

Use Basic Authentication with your api-client-id/api-client-secret as username/password.

Field                 | Description                                   | Format
---------             | -------                                       | -----------
grant_type            | "authorization_code"                          | Text
client_id             | Your API client ID                            | Text
code                  | Authorization code provided to you upon redirect back from the authorization flow. See [User authorization](#connected-apps-integration-guide-user-authorization).  | Text
redirect_uri          | Redirect URL associated with your API client credentials   | Text

### Response

Field                 | Description                                   | Format
---------             | -------                                       | -----------
access_token          | Access token to be used when calling API endpoints on behalf of user. Valid for 12 hours. | uuid
token_type            | "bearer"                                      | Text
refresh_token         | Refresh token which you need to use in order to request new access_token. The lifetime of refresh tokens is 10 years. | uuid
expires_in            | Expiry time in seconds                        | Integer
scope                 | "transfers"                                   | Text


## Refresh user access token

> Example Request:

```shell
      curl \
      'https://api.sandbox.transferwise.tech/oauth/token' \
      -u '[your-api-client-id]:[your-api-client-secret]' \
      -d 'grant_type=refresh_token' \
      -d 'refresh_token=[user's refresh token]'
```

> Example Response:

```json
  {
    "access_token":"be69d566-971e-4e15-9648-85a486195863",
    "token_type":"bearer",
    "refresh_token":"1d0ec7b9-b569-426d-a18d-8dead5b6a3cc",
    "expires_in":43199,
    "scope":"transfers"
  }
```

Access tokens are valid for 12 hours, so upon expiry you need to use the refresh token to generate a new access token. 

In order to maintain an uninterrupted connection, you can request a new access token whenever it’s close to expiring.
There is no need to wait for the actual expiration to happen first. 

Depending on how your application uses the TransferWise API, you may find that requesting a new access token before
attempting a series of API calls on behalf of an individual user will avoid issues with expired access tokens. 

### Request

**`POST https://api.sandbox.transferwise.tech/oauth/token`**

Use Basic Authentication with your api-client-id/api-client-secret as username/password.

Field                 | Description                                   | Format
---------             | -------                                       | -----------
grant_type            | "refresh_token"                               | Text
refresh_token         | User's refresh token obtained in [Get user tokens](#connected-apps-integration-guide-get-user-tokens) step. | uuid

### Response

Field                 | Description                                   | Format
---------             | -------                                       | -----------
access_token          | Access token to be used when calling API endpoints on behalf of user. Valid for 12 hours. | uuid
token_type            | "bearer"                                      | Text
refresh_token         | Refresh token which you need to use in order to request new access token once the existing one expires | uuid
expires_in            | Expiry time in seconds                        | Integer
scope                 | "transfers"                                   | Text


## Refresh token expiry

It is possible that a user's refresh token will become invalid. This could happen for a number of reasons, including:

* The user revokes access for your application to their account.
* The user enables enhanced security on their TransferWise account.
* TransferWise revokes the token due to a security breach of your client secret.

Your application should handle a failing refresh token scenario by sending the user through the authorization code flow
as you initially did.

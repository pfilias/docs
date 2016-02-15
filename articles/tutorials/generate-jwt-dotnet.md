---
title: Generate a JSON Web Token (JWT) using .NET
---

## Introduction

JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with HMAC algorithm) or a public/private key pair using RSA.

::: panel-info More infomation on JSON Web Tokens
For a more detailed background on JSON Web Tokens and how they are used in Auth0, please refer to the documents [Get Started with JSON Web Tokens](https://auth0.com/learn/json-web-tokens/) and [JsonWebTokens in Auth0](https://auth0.com/docs/jwt) on the Auth0 website.
:::

When a user authenticates in your application using the [Auth0 Lock](https://auth0.com/lock), one of the parameters which are returned is a JWT ([see the Lock documentation](https://auth0.com/docs/libraries/lock)). You can also authenticate a user directly [using the Authentication API](https://auth0.com/docs/auth-api#!#post--oauth-ro) and obtain a JWT that way.

There are however times when you may want to generate a JWT manually. Let us take an example where you have [secured your ASP.NET Web API using JWT](https://auth0.com/docs/quickstart/backend/webapi-owin/). Your users sign in using Lock and on every call to the API you pass along the JWT obtained during authentication.

Suppose that you now want to add the ability for another app, or perhaps guest users, to be able to make calls to API methods which are secured. In this case you do not have a JWT since there was no user who authenticated. In instances like these you can easily generate a JWT yourself, and pass that along in the Authentication header to API calls.

## Generating a JWT

In order to generate a JWT we'll make use of a NuGet package called `jose-jwt`, so go ahead and install it:

```
Install-Package jose-jwt
```

And import the namespace for the library:

```
using Jose;
```

Next up you will need to add two helper methods to your code. The first will allow you to decode the **Client Secret** which is BASE64 encoded, but we need to pass it as non-base64 encoded to the JWT generator. The second is to generate a UNIX timestamp from a `DateTime`:

```
private byte[] Base64UrlDecode(string arg)
```

> The code for the `Base64UrlDecode` method was taken from the [following StackOverflow answer](http://stackoverflow.com/a/33113820)

And here is the code to generate the JWT:

```
byte[] secretKey = Base64UrlDecode("${account.cientSecret}");

Walking throught the code about, we first decode the Client Secret. Next we set the issued date to the current date and time, and the expiry date to 10 hours from now.

Then we generate the actual payload of the JWT by setting the following parameters:

* **iss** must be set to your Auth0 domain. Be sure to include the trailing slash at the end.
*  **aud** must be set to your Client ID
*  **sub** must be set to the user's identifier. In this case we use "anonymous", but you can basically use any identifier here. If the token is for another application, then use something which identifies that application.
* **iat** is set to the time the token was issued. We convert the actual `DatetTime` to a Unix timestamp.
* **exp** is set to the time the token expires, once again converted to a Unix timestamp

Finally we generate the actual token by by calling the `Encode` method of the `JWT` class, passing the payload, the secretkey and the JWT Signature Algorithm, in this case **HS256**.

Now you can use that token to pass along as a bearer token in the Auth header for calls made to your API.

## Checking the user ID

Inside your Web API controller you may want to check the user id to see whether it was a user which was authenticated via Auth0, or whether it is the anonymous user for which you generated a token.

This is as simple as retrieving the value of the **NameIdentifier** claim and checking whether the value is "anonymous". If it is you may have different logic executing for the anonymous user.

```
ClaimsPrincipal principal =  User as ClaimsPrincipal;
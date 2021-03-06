# Multifactor Authentication with YubiKey-NEO

This tutorial shows how to implement multifactor with a [YubiKey-NEO](https://www.yubico.com/products/yubikey-hardware/yubikey-neo/).

Enabling this functionality relies on three core features available in Auth0:

* [Rules](/rules)
* The [redirect protocol](/protocols#21)
* [Auth0 Webtasks](https://webtask.io)

__Rules__ are used to evaluate a condition that will trigger the multifactor challenge. The __redirect protocol__ is used to direct the user to a web site that will actually perform the 2nd authentication factor with YubiKey and __Webtasks__ to host that website.

## Configuring the Webtask

Auth0 Webtasks allows you to run arbitrary code on the Auth0 sandbox. It is the same underlying technology used for __rules__ and __custom db connections__.

In this example, we will use Webtasks to implement a simple website that that captures the YubiKey-NEO input, and then backend code that calls the Yubico API. After validation completes, we return the result back to Auth0, resuming the login transaction originally initiated by the user.


```javascript
var request = require('request');
var qs = require('qs');
var jwt = require('jsonwebtoken');

return function (context, req, res) {

  require('async').series([
    /*
     * We only care about POST and GET
     */
    function(callback) {
      if (req.method !== 'POST' && req.method !== 'GET') {
        res.writeHead(404);
        return res.end('Page not found');
      }
      return callback();
    },

    /*
     * 1. GET: Render initial View with OTP.
     */
    function(callback) {
      if (req.method === 'GET') {
        renderOtpView();
      }
      return callback();
    },

    /*
     * 2. Validate OTP
     */
    function(callback) {
      if (req.method === 'POST') {

      yubico_validate(context.data.yubikey_clientid, context.body.otp, function(err,resp) {
        if (err) {
            return callback(err);
        }

        if(resp.status==='OK'){
          //Return result to Auth0 (includes OTP and Status. Only when OK)
          var token = jwt.sign({
                status: resp.status,
                otp: resp.otp
                },
                new Buffer(context.data.yubikey_secret, 'base64'),
                      {
                subject: context.data.user,
                expiresInMinutes: 1,
                audience: context.data.yubikey_clientid,
                issuer: 'urn:auth0:yubikey:mfa'
            });

        res.writeHead(301, {Location: context.data.returnUrl + "?id_token=" + token + "&state=" + context.data.state});
        res.end();
        }
        return callback([resp.status]);
      });

    return callback();
     }
    },
  ], function(err) {

      if (Array.isArray(err)) {
          return renderOtpView(err);
      }

      if (typeof err === 'string') {
          return renderOtpView([err]);
      }

      if (typeof err === 'object') {
          var errors = [];
          errors.push(err.message || err);
          return renderOtpView(errors);
        }
  });

  function yubico_validate(clientId, otp, done){
    var params = {
        id: clientId,
        otp: otp,
        nonce: uid(16)
      };

    request.get('http://api.yubico.com/wsapi/2.0/verify',
    {
      qs: params
    },function(e,r,b){
      if(e) return done(e);
      if(r.statusCode !== 200) return done(new Error('Error: ' + r.statusCode));
      var yubico_response=qs.parse(b.replace(/\r\n/g, '&'));
      if(yubico_response.nonce !== params.nonce) return done(new Error('Invalid response - nonce doesn\'t match'));
      done(null,yubico_response);
    });
  }

  function uid(len) {
      var buf = []
      , chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
      , charlen = chars.length;

      for (var i = 0; i < len; ++i) {
        buf.push(chars[getRandomInt(0, charlen - 1)]);
      }

      return buf.join('');
  }

  function getRandomInt(min, max) {
      return Math.floor(Math.random() * (max - min + 1)) + min;
  }

  function renderOtpView(errors) {
      res.writeHead(200, {
        'Content-Type': 'text/html'
      });
      res.end(require('ejs').render(otpForm.stringify(), {
        user: context.data.user,
        errors: errors || []
      }));
  }

  function otpForm() {
    /*
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta charset="UTF-8">
        <title>Auth0 - Yubikey MFA</title>
        <style>

        <!--

        All styling of this HTML form is omitted for brevity. Please see the full source code here: https://github.com/auth0/rules/blob/master/redirect-rules/yubico-mfa.md
        -->

        </style>
      </head>
      <body>
        <div class="modal-wrapper">
          <div class="modal-centrix">
            <div class="modal">
              <form onsubmit="showSpinner();" action="" method="POST" enctype="application/x-www-form-urlencoded">
                <div class="head"><img src="https://cdn.auth0.com/styleguide/2.0.9/lib/logos/img/badge.png" class="logo auth0"><span class="first-line">Yubikey 2FA</span></div>
                <div class="errors <${'%'}- (errors.length === 0 ? 'hidden' : '') ${'%'}>">
                  <${'%'} errors.forEach(function(error){ ${'%'}>
                  <div class="p"><${'%'}= error ${'%'}></div>
                  <${'%'}})${'%'}>
                </div>
                <div class="body"><span class="description">Hi <strong><${'%'}- user || "" ${'%'}></strong>, please tap your Yubikey.</span><span class="description domain"><span>Yubikey OTP:</span>
                    <input type="text" autocomplete="off" name="otp" required autofocus id="otp"></span></div>
                <div id="ok-button" class="ok-cancel">
                  <button class="ok full-width">
                    <div class="icon icon-budicon-509"></div>
                    <div class="spinner"></div>
                  </button>
                </div>
              </form>
            </div>
          </div>
          <script>
            function showSpinner() {
              document.getElementById('ok-button').className += " auth0-spinner";
            }
          </script>
        </div>
      </body>
    </html>
    */
  }
}
```

> Full source code is available [here](https://github.com/auth0/rules/blob/master/redirect-rules/yubico-mfa.md).

This sample, uses a single Webtask to handle 3 states:

* __Rendering the UI__ for the first time (implemented in the `otpForm` function).
* __Capturing the YubiKey-NEO__ code and validating it with the Yubico API.
* __Returning the result to Auth0__. If validation succeeds, the result is returned to Auth0 to continue with the login transaction.

> Notice the redirect back to Auth0 carries two query string values: an `id_token` and a `state` parameters. `id_token` is a convenient, and secure way of transferring information back to Auth0. `state` is mandatory to play back to protect against CSRF attacks.

Notice __no secrets are hardcoded__ in the Webtask code. They are referred to by the variables `context.data.yubico_clientid` and `context.data.yubico_secret`. When the Webtask is created, these sensitive parameters are embedded in the Webtask token securely.

Save the code above in a file somewhere in your file system. This tutorial assumes `yubico-mfa-wt.js` for the file.

### 1. Initialize Webtask CLI

Code in __rules__ is automatically packaged as Webtasks by Auth0. Because this is a custom Webtask, it needs to be created with the Webtask CLI.

All instructions are available under __Account Settings__ on the [dashboard](${uiURL}/#/account/webtasks).

Once the Webtask CLI is installed, run:

```
wt create --name yubikey-mfa --secret yubikey_secret={YOUR YUBIKEY SECRET} --secret yubikey_clientid={YOUR YUBIKEY CLIENT ID} --secret returnUrl=https://${account.namespace}/continue --output url --profile {WEBTASK PROFILE} yubico-mfa-wt.js
```

The `WEBTASK PROFILE` is the value obtained from the __Account Settings__ above.

The `create` command will generate a URL that will look like this:

```
https://sandbox.it.auth0.com/api/run/${account.tenant}/yubikey-mfa?webtask_no_cache=1
```

Keep this URL handy.

## Configuring the Rule

This sample uses a single rule that handles both the initial redirect to the Webtask, and the returned back result. "Returning" is indicated by the `protocol` property in the `context` object.

```
function (user, context, callback) {

  var yubikey_secret = configuration.YUBIKEY_SECRET;

  //Returning from OTP validation
  if(context.protocol === 'redirect-callback') {
    var decoded = jwt.verify(context.request.query.id_token,
                               new Buffer(yubikey_secret,'base64'));
    if(!decoded) return callback(new Error('Invalid OTP'));
    if(decoded.status !== 'OK') return callback(new Error('Invalid OTP Status'));

    user.otp = decoded.otp;

    return callback(null,user,context);
  }

  //Trigger MFA
  context.redirect = {
        url: config.WEBTASK_URL + "?user=" + user.name
  }

  callback(null,user,context);
}
```

The `context.redirect` statement will instruct Auth0 to redirect the user to the Webtask URL instead of calling back the app.

> Notice that the “returning” section of the rule, validates the JWT issued in the Webtask. This prevents the result of the MFA part of the transaction to be tampered with, because the payload is digitally signed (with a shared secret).

Now, every time the user logs in, they will be redirected to the Webtask and will see this:

![](/media/articles/mfa/yubico-mfa.png)

Of course you can add any logic to the rule to decide the condition under which the challenge will happen:

* The IP address or location of the user.
* The Application the user is logging in to.
* The type of authentication used (e.g. AD, LDAP, social, etc.)

## See also:

* [Rules](/rules)
* [Multi-factor in Auth0](/mfa)
* [Auth0 Webtasks](https://webtask.io/)

# Node JS Security Tips


# 1. Overview

These are generally applied to all the frameworks. Summarizing posts from different references.

Security HTTP Headers -

- **Strict-Transport-Security** enforces secure (HTTP over SSL/TLS) connections to the server.
- **X-Frame-Options** provides clickjacking protection.
- **X-XSS-Protection** enables the Cross-site scripting (XSS) filter built into most recent web browsers.
- **X-Content-Type-Options** prevents browsers from MIME-sniffing a response away from the declared content type.
- **Content-Security-Policy** prevents a wide range of attacks, including Cross-site scripting and other cross-site injections.

In NodeJS it is easy to set these with helmet modules.

```jsx
var express = require('express');
var helmet = require('helmet');
var app = express();
app.use(helmet());
```

These can also be implement on Server level without touching the codebase -

```jsx
# nginx.conf

add_header X-Frame-Options SAMEORIGIN;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header Content-Security-Policy "default-src 'self'";
```

### Sensitive Data on the Client Side

When deploying front-end applications make sure that you never expose API secrets and credentials in your source code, as it will be readable by anyone.

There is no good way to check this automatically, but you have a couple of options to mitigate the risk of accidentally exposing sensitive data on the client side:

- use of pull requests
- regular code reviews
- Code obfuscation

## Ratelimiting

To protect your applications from these kind of attacks you have to implement some kind of rate-limiting. In Node.js you can use the ratelimiter.
 package.

```jsx
var email = req.body.email;
var limit = new Limiter({ id: email, db: db });

limit.get(function(err, limit) {

});
```

It’s not the only way , You can wrap into a middleware and just drop it into any application.Both Express and Koa has great middlewares.

```jsx
var ratelimit = require('koa-ratelimit');
var redis = require('redis');
var koa = require('koa');
var app = koa();

var emailBasedRatelimit = ratelimit({
  db: redis.createClient(),
  duration: 60000,
  max: 10,
  id: function (context) {
    return context.body.email;
  }
});

var ipBasedRatelimit = ratelimit({
  db: redis.createClient(),
  duration: 60000,
  max: 10,
  id: function (context) {
    return context.ip;
  }
});

app.post('/login', ipBasedRatelimit, emailBasedRatelimit, handleLogin);
```

Try with hydra which is a bydefault Kali Linux tool.

## **Cookie Flags**

- **secure** – this attribute tells the browser to only send the cookie if the request is being sent over HTTPS.
- **HttpOnly** – this attribute is used to help prevent attacks such as cross-site scripting, since it does not allow the cookie to be accessed via JavaScript.

### Cookie Scope

- **domain** – this attribute is used to compare against the domain of the server in which the URL is being requested. If the domain matches or if it is a sub-domain, then the path attribute will be checked next.
- **path** – in addition to the domain, the URL path that the cookie is valid for can be specified. If the domain and path match, then the cookie will be sent in the request.
- **expires** – this attribute is used to set persistent cookies, since the cookie does not expire until the set date is exceeded

Using a wrapper for the cookie session - 

```jsx
var cookieSession = require('cookie-session');
var express = require('express');

var app = express();

app.use(cookieSession({
name: 'session',
keys: [
process.env.COOKIE_KEY1,
process.env.COOKIE_KEY2
]
}));

app.use(function (req, res, next) {
var n = req.session.views || 0;
req.session.views = n++;
res.end(n + ' views');
});

app.listen(3000);
```

## CSRF

In Node.js to mitigate this kind of attacks you can use the csrf module. As it is quite low-level, there are wrappers for different frameworks as well. 

It can happen because cookies are sent with every request to a website – even when those requests come from a different site.

### Example

```
<body onload="document.forms[0].submit()">
  <form method="POST" action="http://yoursite.com/user/delete">
    <input type="hidden" name="id" value="123555.">
  </form>
</body>

```

The result of the above snippet can easily result in deleting your user profile.

### How to prevent it?

To prevent CSRF, you should implement the synchronizer token pattern – luckily the Node community has already done it for you. In short, this is how it works:

1. When a `GET` request is being served check for the CSRF token – if it does not exists, create one
2. When a user input is showed, make sure to add a hidden input with the CSRF token’s value
3. When the form is sent, make sure that the value coming from the form and from the session are a match.

One example for this is the csurf module: an express middleware for CSRF protection.

On the route handler level you have to do something like this:

```jsx
var cookieParser = require('cookie-parser');
var csrf = require('csurf');
var bodyParser = require('body-parser');
var express = require('express');
 
// setup route middlewares 
var csrfProtection = csrf({ cookie: true });
var parseForm = bodyParser.urlencoded({ extended: false });
 
// create express app 
var app = express();
 
// we need this because "cookie" is true in csrfProtection 
app.use(cookieParser());
 
app.get('/form', csrfProtection, function(req, res) {
  // pass the csrfToken to the view 
  res.render('send', { csrfToken: req.csrfToken() });
});
 
app.post('/process', parseForm, csrfProtection, function(req, res) {
  res.send('data is being processed');
});
```

**Frontend :-** 

```jsx
<form action="/process" method="POST">
  <input type="hidden" name="_csrf" value="{{csrfToken}}">
  
  Favorite color: <input type="text" name="favoriteColor">
  <button type="submit">Submit</button>
</form>
```

 

## Data Validation -

### OS injection  -

In practice, if you have a URL like:

```
https://example.com/downloads?file=user1.txt

```

it could be turn into:

```
https://example.com/downloads?file=%3Bcat%20/etc/passwd

```

In this example `%3B` becomes the semicolon, so multiple OS commands can be run.

**To defend against these kind of attacks make sure that you always filter/sanitize user input.**

## **SSL Version, Algorithms, Key length**

**Checking for Certificate information**

```
nmap --script ssl-cert,ssl-enum-ciphers -p 443,465,993,995 www.example.com

```

**Testing SSL/TLS vulnerabilities with sslyze**

```
./sslyze.py --regular example.com:443
```

Checking **Strict-Policy** 

```jsx
curl -s -D- [https://twitter.com/](https://twitter.com/) | grep -i Strict
```

*Great tool to check headers* - [https://securityheaders.com/](https://securityheaders.com/)

## DDOS  -

This kind of attack exploits the fact that most Regular Expression implementations may reach extreme situations that cause them to work very slowly. These Regexes are called Evil Regexes:

To check your Regexes against these, you can use a Node.js tool called [safe-regex](https://www.npmjs.com/package/safe-regex). *It may give false-positives, so use it with caution.*

```
$ node safe.js '(beep|boop)*'
true
$ node safe.js '(a+){10}'
false
```

# Error Handling

### Error Codes, Stack Traces

During different error scenarios the application may leak sensitive details about the underlying infrastructure, like: `X-Powered-By:Express`.

Stack traces are not treated as vulnerabilities by themselves, but they often reveal information that can be interesting to an attacker. Providing debugging information as a result of operations that generate errors is considered a bad practice. You should always log them, but do not show them to the users.

# NPM

With great power comes great responsibility – **[NPM](https://blog.risingstack.com/glossary/npm/)**npm is a software registry that serves over 1.3 million packages. npm is used by open source developers from all around the world to share and borrow code, as well as many businesses. There are three components to npm: the website the Command Line Interface (CLI) the registry Use the website to discover and download packages, create user profiles, and... has lots of packages what you can use instantly, but that comes with a cost: you should check what you are requiring to your applications. They may contain security issues that are critical.

### The Node Security Project

Luckily the Node Security project has a great tool that can check your used modules for known vulnerabilities.

```
npm i nsp -g
# either audit the shrinkwrap
nsp audit-shrinkwrap
# or the package.json
nsp audit-package
```

## Eval()

Eval is not the only one you should avoid – in the background each one of the following expressions use `eval`:

- `setInterval(String, 2)`
- `setTimeout(String, 2)`
- `new Function(String)`

But why should you avoid `eval`?

It can open up your code for injections attacks.

# Say no to `sudo node app.js`

I see this a lot: people are running their Node app with superuser rights. Why? Because they want the application to listen on port 80 or 443.

This is just wrong. In case of an error/bug your process can bring down the entire system, as it will have credentials to do anything.

# Content Security Policy

> Content Security Policy (CSP) is an added layer of security that helps to detect and mitigate certain types of attacks, including Cross Site Scripting (XSS) and data injection attacks.
> 

CSP can be enabled by the `Content-Security-Policy` HTTP header.

### Example

`Content-Security-Policy: default-src 'self' *.mydomain.com`

This will allow content from a trusted domain and its subdomains.

---
title: "Finding XSS with source code analysis I"
date: 2022-06-03T19:00:00+07:00
tags: [xss, codeanalysis, owasp10]
description: My way on finding vulnerabilities in new target website, first thing to do are fire up the burp suite (collecting all requests and responses from target).
---
My way on finding vulnerabilities in new target website, first thing to do are fire up the burp suite (collecting all requests and responses from target), and click all the functions and buttons on target website. After that I usually analyze the HTML & javascript from the response.

While reviewing the code.. I found a comment like this.. Well that's interesting...

```js
$(window).on('load', function() {

// old code for handling errors, due to be deprecated
```

I was so curious... So I start analyzing the code.  
It was noted that an event listener is waiting for the page to load via `$(window).on('load'`

```js
$(window).on('load', function() {

    // old code for handling errors, due to be deprecated
    var errorCode = decodeURIComponent(getHashValue("error"));  
    *snipped*
    if (errorCode != "null")
    {
        if (errorCode == '2') {
            // invalid password!
            document.getElementById("errorMsg").innerHTML =  "<div class='alert alert-danger mt-3 d-inline-block' role='alert'> <span class='badge badge-secondary'>Error "+errorCode+"</span> - No match with our records.</div>";
        } else if (errorCode == '1') {
            // invalid email ?
            document.getElementById("errorMsg").innerHTML = "<div class='alert alert-danger mt-3 d-inline-block' role='alert'> <span class='badge badge-secondary'>Error "+errorCode+"</span> - Not valid. </div>";
        } else {
            document.getElementById("errorMsg").innerHTML =  "<div class='alert alert-danger mt-3 d-inline-block' role='alert'> Error "+errorCode+" - An unexpected error occured. </div>";
        }
    }

```

As evident above, when the page is loaded the `getHashValue` function is called with an argument `(key)` consisting of a string `("error")` representing the parameter within the URL fragment who's value will be returned and assigned to the `errorCode` variable. The `getHashValue` function can be seen below:

```js
function getHashValue(key) {
    var matches = location.hash.match(new RegExp(key+'=([^&]*)'))
    return matches ? matches[1] : null;
}
```

Because the `errorCode` variable can be injected and controlled via user input in the URL and the fact that the following JavaScript will take the `errorCode` value and inject it into the DOM as HTML:

```js
document.getElementById("errorMsg").innerHTML =  "<div class='alert alert-danger mt-3 d-inline-block' role='alert'> Error "+errorCode+" - An unexpected error occured. </div>";

```

Can result in XSS as evident by clicking the link below which when when visited by a victim will execute attacker controlled JavaScript in the context of the victim user's browser:

```html
https://redacted.com/#error=</div></div><img src=x onerror=alert(0)>
```

Successful execution of malicious JavaScript in the browser can be seen below:

![xss](/images/001.png)
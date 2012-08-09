Types of Cross-Site Scripting Attacks
=====================================

XSS attacks can be categorised in two ways. The first lies in how malicious input navigates the web application. Input to an application can be included in the output of the current request, stored for inclusion in the output of a later request, or passed to a Javascript based DOM operation. This gives rise to the following categories:

1. Reflected XSS Attack

In a Reflected XSS attack, untrusted input sent to a web application is immediately included in the application's output, i.e. it is reflected from the server back to the browser in the same request. Reflection can occur with error messages, search engine submissions, comment previews, etc. This form of attack can be mounted by persuading a user to click a link or submit a form of the attacker's choosing. Getting a user to click untrusted links may require a bit of persuasion and involve emailing the target, mounting a UI Redress attack, or using a URL Shortener service to disguise the URL. Social services are particularly vulnerable to shortened URLs since they are commonplace in that setting. Be careful of what you click!

2. Stored XSS Attack

A Stored XSS attack is when the payload for the attack is stored somewhere and retrieved as users view the targeted data. While a database is to be expected, other persistent storage mechanisms can include caches and logs which also store information for long periods of time. We've already learned about Log Injection attacks.

3. DOM-based XSS Attack

DOM-based XSS can be either reflected or stored and the differentiation lies in how the attack is targeted. Most attacks will strike at the immediate markup of a HTML document. However, HTML may also be manipulated by Javascript using the DOM. An injected payload, rendered safely in HTML, might still be capable of interfering with DOM operations in Javascript. There may also be security vulnerabilities in Javascript libraries or their usage which can also be targeted.
SSL/TLS From Client (Client/Browser to Server)
==============================================

So far, most of what we've discussed has been related to SSL/TLS connections established from a PHP web application to another server. Of course, there are also quite a few security concerns when our web application is the party exposing SSL/TLS support to client browsers and other applications. At this end of the process we run the risk of suffering security attacks arising from Insufficient Transport Layer Protection vulnerabilities.

This is actually quite basic if you think about it. Let's say I create an online application which a secure login to protect the user's password. The login form is served over HTTPS and the form is submitted over HTTPS. Mission accomplished. The user is then redirected to a HTTP URL to start using their account. Spot the problem?

When a Man-In-The-Middle (MitM) Attack is a concern, we should not simply protect the login form for users and then call it quits. Over HTTP, the user's session cookie and all other data that they submit, and all other HTML markup that they receive, will not be secure. An MitM could steal the session cookie and impersonate the user, inject XSS into the received web pages to perform tasks as the user or manipulate their actions, and the MitM need never know the password to accomplish all of this.

Merely securing authentication with HTTPS will prevent direct password theft but does not prevent session hijacking, other forms of data theft and Cross-Site Scripting (XSS) attack injection. By limiting the protection offered by HTTPS to the user, we are performing insufficient transport layer protection. Our application's users are STILL vulnerable to MitM attacks.
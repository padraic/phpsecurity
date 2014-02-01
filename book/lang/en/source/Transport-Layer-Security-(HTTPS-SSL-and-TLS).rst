Insufficient Transport Layer Security (HTTPS, TLS and SSL)
##########################################################

Communication between parties over the internet is fraught with risk. When you are sending payment instructions to a store using their online facility, the very last thing you ever want to occur is for an attacker to be capable of intercepting, reading, manipulating or replaying the HTTP request to the online application. You can imagine the consequences of an attacker being able to read your session cookie, or to manipulate the payee, product or billing address, or to simply to inject new HTML or Javascript into the markup sent in response to a user request to the store.

Protecting sensitive or private data is serious business. Application and browser users have an extremely high expectation in this regard placing a high value on the integrity of their credit card transactions, their privacy and their identity information. The answer to these concerns when it comes to defending the transfer of data from between any two parties is to use Transport Layer Security, typically involving HTTPS, TLS and SSL.

The broad goals of these security measures is as follows:

* To securely encrypt data being exchanged
* To guarantee the identity of one or both parties
* To prevent data tampering
* To prevent replay attacks

The most important point to notice in the above is that all four goals must be met in order for Transport Layer Security to be successful. If any one of the above are compromised, we have a real problem.

A common misconception, for example, is that encryption is the core goal and the others are non-essential. This is, in fact, completely untrue. Encryption of the data being transmitted requires that the other party be capable of decrypting the data. This is possible because the client and the server will agree on an encryption key (among other details) during the negotiation phase when the client attempts a secure connection. However, an attacker may be able to place themselves between the client and the server using a number of simple methods to trick a client machine into believing that the attacker is the server being contacted, i.e. a Man-In-The-Middle (MitM) Attack. This encryption key will be negotiated with the MitM and not the target server. This would allow the attacker to decrypt all the data sent by the client. Obviously, we therefore need the second goal - the ability to verify the identity of the server that the client is communicating with. Without that verification check, we have no way of telling the difference between a genuine target server and an MitM attacker.

So, all four of the above security goals MUST be met before a secure communication can take place. They each work to perfectly complement the other three goals and it is the presence of all four that provides reliable and robust Transport Layer Security.

Aside from the technical aspects of how Transport Layer Security works, the other facet of securely exchanging data lies in how well we apply that security. For example, if we allow a user to submit an application's login form data over HTTP we must then accept that an MitM is completely capable of intercepting that data and recording the user's login data for future use. If we allow pages loaded over HTTPS to, in turn, load non-HTTPS resources then we must accept that a MitM has a vehicle with which to inject Cross-Site Scripting attacks to turn the user's browser into a pre-programmed weapon that will operate over the browser's HTTPS connection tranparently.

In judging the security quality of any implementation, we therefore have some very obvious measures drawn from the four goals I earlier mentioned:

* Encryption: Does the implementation use a strong security standard and cipher suite?
* Identity: Does the implementation verify the server's identity correctly and completely?
* Data Tampering: Does the implementation fully protect user data for the duration of the user's session?
* Replay Attacks: Does the implementation contain a method of preventing an attacker from recording requests and repetitively send them to the server to repeat a known action or effect?

These questions are your core knowledge for this entire chapter. I will go into far more detail over the course of the chapter, but everything boils down to asking those questions and identifying the vulnerabilities where they fail to hold true.

A second core understanding is what user data must be secured. Credit card details, personally identifiable information and passwords are obviously in need of securing. However, what about the user's session ID? If we protect passwords but fail to protect the session ID, an attacker is still fully capable of stealing the session cookie while in transit and performing a Session Hijacking attack to impersonate the user on their own PC. Protecting login forms alone is NEVER sufficient to protect a user's account or personal information. The best security is obtained by restricting the user session to HTTPS from the time they submit a login form to the time they end their session.

You should now understanding why this chapter uses the phrase "insufficient". The problem in implementing SSL/TLS lies not in failing to use it, but failing to use it to a sufficient degree that user security is maximised.

This chapter covers the issue of Insufficient Transport Layer Security from three angles.

* Between a server-side application and a third-party server.
* Between a client and the server-side application.
* Between a client  and the server-side application using custom defenses.

The first addresses the task of ensuring that our web applications securely connect to other parties. Transport Layer Security is commonly used for web service APIs and many other sources of input required by the application.

The second addresses the interactions of a user with our web applications using a browser or some other client application. In this instance, we are the ones exposing a secured URL and we need to ensure that that security is implemented correctly and is not at risk from being bypassed.

The third is a curious oddity. Since SSL/TLS has a reputation for not being correctly implemented by programmers, there have been numerous approaches developed to secure a connection without the use of the established SSL/TLS standards. An example is OAuth's reliance on signed requests which do not require SSL/TLS but offer some of the defences of those standards (notably, encryption of the request data is omitted so it's not perfect but a better option than a misconfigured SSL/TLS library).

Before we get down to those specific categories, let's first take a look at Transport Layer Security in general as there is some important basic knowledge to be aware of before we get down into nuts and bolts with PHP.

.. include:: _includes/TLS-Definitions-And-Basic-Vulnerabilities.rst
.. include:: _includes/SSL-TLS-From-PHP-From-Server-To-Server.rst
.. include:: _includes/SSL-TLS-From-Client-Client-Browser-To-Server.rst

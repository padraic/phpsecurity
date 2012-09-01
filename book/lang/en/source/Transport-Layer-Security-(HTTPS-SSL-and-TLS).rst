Transport Layer Security (HTTPS, TLS and SSL)
#############################################

Communication between parties over the internet is fraught with risk. When you are sending payment instructions to your bank using their online facility, the very last thing you ever want to occur is for an attacker to be capable of intercepting, reading, manipulating or replaying the HTTP request to the online application. You can imagine the consequences of an attacker being able to read your session cookie, or to manipulate the payee targeted, or simply to inject new HTML or Javascript into the markup sent in response to a user request to the bank.

Protecting sensitive or private data is serious business. Application and browser users have an extremely high expectation in this regard placing a high value on the integrity of their credit card transactions, their privacy and their identity information. The answer to these concerns when it comes to defending the transfer of data from between any two parties is to use Transport Layer Security: HTTPS, TLS and SSL.

The goals of these security measures is as follows:

* To encrypt data being exchanged
* To guarantee the identity of one or both parties
* To prevent data tampering
* To prevent replay attacks

The most important point to notice in the above is that all four goals must be met in order for Transport Layer Security to be successful. If any one of the above are compromised, we have a real problem.

A common misconception, for example, is that encryption is the core goal and the others are non-essential. This is, in fact, completely untrue. Encryption of the data being transmitted requires that the other party be capable of decrypting the data. This is possible because the client and the server will agree on an encryption key during the negotiation phase when the client attempts a secure connection. However, if an attacker (commonly referred to as a Man-In-The-Middle or MitM) masquerades as the server, this encryption key will be negotiated with them and not the target server. This would allow the attacker or MitM to decrypt all the data sent by the client. Obviously, we therefore need the second goal - the ability to verify the identity of the server that the client is communicating with. Without that verification check, we have no way of telling the difference between a genuine target server and an attacker.

So, all four of the above security goals MUST be met before a secure communication can take place. They each work to perfectly complement the other three goals and it is the presence of all four that provides reliable and robust Transport Layer Security.

Aside from the technical aspects of how Transport Layer Security works, the other facet of securely exchanging data lies in how well we apply that security. For example, if we allow a user to submit an application's login form data over HTTP we must then accept that an MitM is completely capable of intercepting that data and recording the user's login data for future use. Knowing when to apply Transport Layer Security and how to ensure that it cannot be compromised or disregarded by the user in error, is the other part of the battle that we address in this chapter.
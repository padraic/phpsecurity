Insufficient Transport Layer Security (HTTPS, TLS and SSL)
##########################################################

Communication between parties over the internet is fraught with risk. When you are sending payment instructions to a store using their online facility, the very last thing you ever want to occur is for an attacker to be capable of intercepting, reading, manipulating or replaying the HTTP request to the online application. You can imagine the consequences of an attacker being able to read your session cookie, or to manipulate the payee, product or billing address, or to simply to inject new HTML or Javascript into the markup sent in response to a user request to the store.

Protecting sensitive or private data is serious business. Application and browser users have an extremely high expectation in this regard placing a high value on the integrity of their credit card transactions, their privacy and their identity information. The answer to these concerns when it comes to defending the transfer of data from between any two parties is to use Transport Layer Security, typically involving HTTPS, TLS and SSL.

The broad goals of these security measures is as follows:

* To encrypt data being exchanged
* To guarantee the identity of one or both parties
* To prevent data tampering
* To prevent replay attacks

The most important point to notice in the above is that all four goals must be met in order for Transport Layer Security to be successful. If any one of the above are compromised, we have a real problem.

A common misconception, for example, is that encryption is the core goal and the others are non-essential. This is, in fact, completely untrue. Encryption of the data being transmitted requires that the other party be capable of decrypting the data. This is possible because the client and the server will agree on an encryption key (among other details) during the negotiation phase when the client attempts a secure connection. However, an attacker may be able to place themselves between the client and the server using a number of simple methods to trick a client machine into believing they are the server being contacted, i.e. a Man-In-The-Middle (MitM) Attack. This encryption key will be negotiated with the MitM and not the target server. This would allow the attacker to decrypt all the data sent by the client. Obviously, we therefore need the second goal - the ability to verify the identity of the server that the client is communicating with. Without that verification check, we have no way of telling the difference between a genuine target server and an MitM attacker.

So, all four of the above security goals MUST be met before a secure communication can take place. They each work to perfectly complement the other three goals and it is the presence of all four that provides reliable and robust Transport Layer Security.

Aside from the technical aspects of how Transport Layer Security works, the other facet of securely exchanging data lies in how well we apply that security. For example, if we allow a user to submit an application's login form data over HTTP we must then accept that an MitM is completely capable of intercepting that data and recording the user's login data for future use.

In judging the security quality of any implementation, we therefore have some very obvious measures drawn from the four goals I earlier mentioned:

* Encryption: Does the implementation use a strong security standard and cipher suite?
* Identity: Does the implementation verify the server's identity correctly and completely?
* Data Tampering: Does the implementation fully protect user data for the duration of the user's session?
* Replay Attacks: Does the implementation contain a method of preventing an attacker from recording requests and repetitively send them to the server to repeat a known action or effect?

These questions are your core knowledge for this entire chapter. I will go into far more detail over the course of the chapter, but everything boils down to asking those questions and identifying the vulnerabilities where they fail to hold true.

A second core understanding is what user data must be secured. Credit card details, personally identifiable information and passwords are obviously in need to securing. However, what about the user's session ID? If we protect passwords but fail to protect the session ID, an attacker is still fully capable of stealing the session cookie while in transit and performing a Session Hijacking attack to impersonate the user on their own PC. Protecting login forms alone is NEVER sufficient to protect a user's account or personal information. The best security is obtained by restricting the user session to HTTPS from the time they submit a login form to the time they end their session.

You should now understanding why this chapter uses the phrase "insufficient". The problem in implementing SSL/TLS lies not in failing to use it, but failing to use it to a sufficient degree that user security is maximised.

This chapter covers the issue of Insufficient Transport Layer Security from three angles.

* Between a server-side application and a third-party server.
* Between a client and the server-side application.
* Between a client  and the server-side application using custom defenses.

The first addresses the task of ensuring that our web applications securely connect to other parties. Transport Layer Security is commonly used for web service APIs and many other sources of input required by the application.

The second addresses the interactions of a user with our web applications using a browser or some other client application. In this instance, we are the ones exposing a secured URL and we need to ensure that that security is implemented correctly and is not at risk from being bypassed.

The third is a curious oddity. Since SSL/TLS has a reputation for not being correctly implemented by programmers, there have been numerous approaches developed to secure a connection without the use of the established SSL/TLS standards. An example is OAuth's reliance on signed requests which do not require SSL/TLS but offer some of the defences of those standards (notably, encryption of the request data is omitted so it's not perfect but a better option than a misconfigured SSL/TLS library).

Before we get down to those specific categories, let's first take a look at Transport Layer Security in general as there is some important basic knowledge to be aware of before we get down into nuts and bolts with PHP.

Definitions & Basic Vulnerabilities
===================================

Transport Layer Security is a generic title for securing the connection between two parties using encryption, identity verification and so on.


SSL/TLS From PHP (Server to Server)
===================================

As much as I love PHP as a programming language, the briefest survey of popular open source libraries makes it very clear that Transport Layer Security related vulnerabilities are extremely common and, by extension, are tolerated by the PHP community for absolutely no good reason other than it's easier to subject users to security violations than fix the underlying problem. This is backed up by PHP itself suffering from a very poor implementation of SSL/TLS in PHP Streams which are used by everything from socket based HTTP clients to the ``file_get_contents()`` and other filesystem functions. This shortcoming is then exacerbated by the fact that the PHP library makes no effort to discuss the security implications of SSL/TLS failures.

If you take nothing else from this section, my advice is to make sure that all HTTPS requests are performed using the CURL extension for PHP. This extension is configured to be secure by default and is backed up, in terms of expert peer review, by its large user base outside of PHP. Take this one simple step towards greater security and you will not regret it. A more ideal solution would be for PHP's internal developers to wake up and apply the Secure By Default principle to its builtin SSL/TLS support.

My introduction to SSL/TLS in PHP is obviously very harsh. Transport Layer Security vulnerabilities are far more basic than most security issues and we are all familiar with the emphasis it receives in browsers. Our server-side applications are no less important in the chain of securing user data.

Let's examine SSL/TLS in PHP in more detail by looking in turn at PHP Streams and the superior CURL extension.

PHP Streams
-----------

For those who are not familiar with PHP's Streams feature, it was introduced to generalise file, network and other operations which shared common functionality and uses. In order to tell a stream how to handle a specific protocol, there are "wrappers" allowing a Stream to represent a file, a HTTP request, a PHAR archive, a Data URI (RFC 2397) and so on. Opening a stream is simply a matter of calling a supporting function with a relevant URL which indicates the wrapper and target resource to use.

.. code-block:: php

    file_get_contents('file:///tmp/file.ext');

Streams default to using a File Wrapper, so you don't ordinarily need to use a file:// URL and can even use relative paths. This should be obvious since most filesystem functions such as ``file()``, ``include()`` and ``require_once`` all accept stream references. So we can rewrite the above example as:

.. code-block:: php

    file_get_contents('/tmp/file.ext');

Besides files, and of relevance to our current topic of discussion, we can also do the following:

.. code-block:: php

    file_get_contents('http://www.example.com');

Since filesystem functions such as ``file_get_contents()`` support HTTP wrapped streams, they bake into PHP a very simple to access HTTP client if you don't feel the need to expand into using a dedicated HTTP client library like Buzz or Zend Framework's ``\Zend\Http\Client`` classes. In order for this to work, you'll need to enable the ``php.ini`` file's ``allow_url_fopen`` configuration option. This option is enabled by default.

However, things get interesting when you try the following:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $result = file_get_contents($url);

The above is a simple unauthenticated request to the Twitter API over HTTPS. It also has a serious flaw. PHP uses an SSL Context (ssl:// transport) for requests made using the HTTPS (https://) and FTPS (ftps://) wrappers. The SSL Context offers a lot of settings for SSL/TLS and their default values are completely insecure. The above example can be rewritten as follows to show how a default SSL Context can be plugged into ``file_get_contents()`` as a parameter:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $contextOptions = array(
        'ssl' => array()
    );
    $sslContext = stream_context_create($contextOptions);
    $result = file_get_contents($url, NULL, $sslContext);

As described earlier in this chapter, failing to securely configure SSL/TLS leaves the application open to a Man-In-The-Middle (MitM) attack. PHP Streams are entirely insecure over SSL/TLS by default. So, let's correct the above example to make it completely secure!

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $contextOptions = array(
        'ssl' => array(
            'verify_peer'   => TRUE,
            'cafile'        => __DIR__ . '/cacert.pem',
            'verify_depth'  => 5,
            'CN_match'      => 'api.twitter.com'
        )
    );
    $sslContext = stream_context_create($contextOptions);
    $result = file_get_contents($url, NULL, $sslContext);

Now we have a secure example! If you contrast this with the earlier example, you'll note that we had to set four options which were, by default, unset or disabled by PHP. Let's examine each in turn to demystify their purpose.

* verify_peer

Peer Verification is the act of verifying that the SSL Certificate presented by the server we sent the HTTPS request to is valid. In order to be valid, the public certificate from the server must be signed by the private key of a trusted Certificate Authority (CA). This can be checked using the CA's public key which will be included in the file set as the ``cafile`` option to the SSL Context we're using. The certificate must also not have expired.

* cafile

The ``cafile`` setting must point to a valid file containing the public keys of trusted CAs. This is not provided automatically by PHP so you need to have the keys in a concatenated certificate formatted file (a PEM file). If you're having any difficulty locating a copy, you can download a copy which is parsed from Mozilla's VCS from http://curl.haxx.se/ca/cacert.pem . Without this file, it is impossible to perform Peer Verification and the request will fail.

* verify_depth

This setting sets the depth of the chain of trust (i.e. how many signing CAs exist before we get to a root trusted CA).

* CN_match

The previous three options focused on verifying the certificate presented by the server. They do not, however, tell us if the verified certificate is valid for the domain name or IP address we are requesting. To ensure that the certificate is tied to the current domain, we need to perform Host Verification. In PHP, this requires setting ``CN_match`` in the SSL Context to the HTTP host value (including subdomain part if present!). PHP performs the matching internally so long as this option is set.

Not performing this check would allow an MitM to present a valid certificate (which they can easily apply for on a domain under their control) and reuse it during an attack to ensure they are presenting a certificate signed by a trusted CA. However, such a certificate would only be valid for their domain - and not the one you are seeking to connect to. Setting the ``CN_match`` option will detect such certificate mismatches and cause the HTTPS request to fail. While such a valid certificate would contain identity information specific to the attacker (a precondition of getting one!), please bear in mind that there are undoubtedly any number of valid CA-signed certificates, complete with matching private keys, available to a knowledgeable attacker. These may have been stolen from another company or slipped passed a trusted CA radar (as happened in 2011 when DigiNotor notoriously (sorry, couldn't resist) issued a certificate for ``google.com`` to an unknown party who went on to employ it in MitM attacks predominantly against Iranian users.

Limitation?
^^^^^^^^^^^

As described above, verifying that the certificate presented by a server is valid for the host in the URL you're using ensures that a MitM cannot simply present any valid certificate copied from the internet. Is an essential step, one of four, to ensuring your connection is absolutely secure.

The ``CN_match`` parameter exposed by the SSL Content in PHP's ssl:// wrapper tells PHP to perform this matching exercise but it has a downside. The matching used will only check the Common Name (CN) of the SSL certificate but ignore the equally valid Subject Alternative Names (SANs) if defined by the certificate. An SAN lets you protect multiple domain names with a single SSL certificate so it's extremely useful and supported by all modern browsers. Since PHP does not currently support SAN matching, connections over SSL/TLS to a domain secured using such a certificate will fail.

The CURL extension, on the other hand, supports SANs out of the box so it is far more reliable and should be used in preference to PHP's built in HTTPS/FTPS wrappers. Using PHP Streams with this issue introduces a greater risk of erroneous behaviour which in turn would tempt impatient programmers to disable host verification altogether which is the last thing we want to see.

CURL Extension
--------------

Unlike PHP Streams, the CURL extension is all about performing data transfers including its most commonly known capability for HTTP requests. Also unlike PHP Streams' SSL context, CURL is configured by default to make requests securely over SSL/TLS. You don't need to do anything special unless it was compiled without the location of a Certificate Authority cert bundle (e.g. a cacert.pem or ca-bundle.pem file containing the certs for trusted CAs).

Since it requires no special treatment, you can perform a similar Twitter API call to what we used earlier for SSL/TLS over a PHP Stream with a minimum of fuss and without worrying about missing options that will make it vulnerable to MitM attacks.

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $req = curl_init($url);
    curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
    $result = curl_exec($req);

This is why my recommendation to you is to prefer CURL for HTTPS requests. It's secure by default whereas PHP Streams is most definitely not. If you feel comfortable setting up SSL context options, then feel free to use PHP Streams. Otherwise, just use CURL and avoid the headache. At the end of the day, CURL is safer, requires less code, and is less likely to suffer a human-error related failure in its SSL/TLS security.

SSL/TLS From Client (Client/Browser to Server)
==============================================

So far, most of what we've discussed has been related to SSL/TLS connections established from a PHP web application to another server. Of course, there are also quite a few security concerns when our web application is the party exposing SSL/TLS support to client browsers and other applications. At this end of the process we run the risk of suffering security attacks arising from Insufficient Transport Layer Protection vulnerabilities.

This is actually quite basic if you think about it. Let's say I create an online application which a secure login to protect the user's password. The login form is served over HTTPS and the form is submitted over HTTPS. Mission accomplished. The user is then redirected to a HTTP URL to start using their account. Spot the problem?

When a Man-In-The-Middle (MitM) Attack is a concern, we should not simply protect the login form for users and then call it quits. Over HTTP, the user's session cookie and all other data that they submit, and all other HTML markup that they receive, will not be secure. An MitM could steal the session cookie and impersonate the user, inject XSS into the received web pages to perform tasks as the user or manipulate their actions, and the MitM need never know the password to accomplish all of this.

Merely securing authentication with HTTPS will prevent direct password theft but does not prevent session hijacking, other forms of data theft and Cross-Site Scripting (XSS) attack injection. By limiting the protection offered by HTTPS to the user, we are performing insufficient transport layer protection. Our application's users are STILL vulnerable to MitM attacks.
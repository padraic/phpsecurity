SSL/TLS From PHP (Server to Server)
===================================

As much as I love PHP as a programming language, the briefest survey of 

popular open source libraries makes it very clear that Transport Layer 

Security related vulnerabilities are extremely common and, by extension, are 

tolerated by the PHP community for absolutely no good reason other than it's 

easier to subject users to privacy-invading security violations than fix the 

underlying problem. This is backed up by PHP itself suffering from a very poor 

implementation of SSL/TLS in PHP Streams which are used by everything from 

socket based HTTP clients to the ``file_get_contents()`` and other filesystem 

functions. This shortcoming is then exacerbated by the fact that the PHP 

library makes no effort to discuss the security implications of SSL/TLS 

failures.

If you take nothing else from this section, my advice is to make sure that all 

HTTPS requests are performed using the CURL extension for PHP. This extension 

is configured to be secure by default and is backed up, in terms of expert 

peer review, by its large user base outside of PHP. Take this one simple step 

towards greater security and you will not regret it. A more ideal solution 

would be for PHP's internal developers to wake up and apply the Secure By 

Default principle to its built-in SSL/TLS support.

My introduction to SSL/TLS in PHP is obviously very harsh. Transport Layer 

Security vulnerabilities are far more basic than most security issues and we 

are all familiar with the emphasis it receives in browsers. Our server-side 

applications are no less important in the chain of securing user data.

Let's examine SSL/TLS in PHP in more detail by looking in turn at PHP Streams 

and the superior CURL extension.

PHP Streams
-----------

For those who are not familiar with PHP's Streams feature, it was introduced 

to generalise file, network and other operations which shared common 

functionality and uses. In order to tell a stream how to handle a specific 

protocol, there are "wrappers" allowing a Stream to represent a file, a HTTP 

request, a PHAR archive, a Data URI (RFC 2397) and so on. Opening a stream is 

simply a matter of calling a supporting file function with a relevant URL 

which indicates the wrapper and target resource to use.

.. code-block:: php

    file_get_contents('file:///tmp/file.ext');

Streams default to using a File Wrapper, so you don't ordinarily need to use a 

file:// URL and can even use relative paths. This should be obvious since most 

filesystem functions such as ``file()``, ``include()``, ``require_once`` and 

``file_get_contents()`` all accept stream references. So we can rewrite the 

above example as:

.. code-block:: php

    file_get_contents('/tmp/file.ext');

Besides files, and of relevance to our current topic of discussion, we can 

also do the following:

.. code-block:: php

    file_get_contents('http://www.example.com');

Since filesystem functions such as ``file_get_contents()`` support HTTP 

wrapped streams, they bake into PHP a very simple to access HTTP client if you 

don't feel the need to expand into using a dedicated HTTP client library like 

Buzz or Zend Framework's ``\Zend\Http\Client`` classes. In order for this to 

work, you'll need to enable the ``php.ini`` file's ``allow_url_fopen`` 

configuration option. This option is enabled by default.

Of course, the ``allow_url_fopen`` setting also carries a separate risk of 

enabling Remote File Execution, Access Control Bypass or Information 

Disclosure attacks. If an attacker can inject a remote URI of their choosing 

into a file function they could manipulate an application into executing, 

storing or displaying the fetched file including from any untrusted remote 

source. It's also worth bearing in mind that such file fetches would originate 

from localhost and thus be capable of bypassing access controls based on local 

server restrictions. As such, while ``allow_url_fopen`` is enabled by default, 

you should disable it without hesitation.


Back to using PHP Streams as a simple HTTP client (which you now know is NOT 

recommended), things get interesting when you try the following:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $result = file_get_contents($url);

The above is a simple unauthenticated request to the Twitter API over HTTPS. 

It also has a serious flaw. PHP uses an ``SSL Context`` for requests made 

using the HTTPS (https://) and FTPS (ftps://) wrappers. The ``SSL Context`` 

offers a lot of settings for SSL/TLS and their default values are wholly 

insecure. The above example can be rewritten as follows to show how a default 

set of ``SSL Context`` options can be plugged into ``file_get_contents()`` as 

a parameter:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $contextOptions = array(
        'ssl' => array()
    );
    $sslContext = stream_context_create($contextOptions);
    $result = file_get_contents($url, NULL, $sslContext);

As described earlier in this chapter, failing to securely configure SSL/TLS 

leaves the application open to a Man-In-The-Middle (MitM) attacks. PHP Streams 

are entirely insecure over SSL/TLS by default. So, let's correct the above 

example to make it completely secure!

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

Now we have a secure example! If you contrast this with the earlier example, 

you'll note that we had to set four options which were, by default, unset or 

disabled by PHP. Let's examine each in turn to demystify their purpose.

* verify_peer

Peer Verification is the act of verifying that the SSL Certificate presented 

by the Host we sent the HTTPS request to is valid. In order to be valid, the 

public certificate from the server must be signed by the private key of a 

trusted Certificate Authority (CA). This can be verified using the CA's public 

key which will be included in the file set as the ``cafile`` option to the SSL 

Context we're using. The certificate must also not have expired.

* cafile

The ``cafile`` setting must point to a valid file containing the public keys 

of trusted CAs. This is not provided automatically by PHP so you need to have 

the keys in a concatenated certificate formatted file (usually a PEM or CRT 

file). If you're having any difficulty locating a copy, you can download a 

copy which is parsed from Mozilla's VCS from http://curl.haxx.se/ca/cacert.pem 

. Without this file, it is impossible to perform Peer Verification and the 

request will fail.

* verify_depth

This setting sets the maximum allowed number of intermediate certificate 

issuers, i.e. the number of CA certificates which are allowed to be followed 

while verifying the initial client certificate.

* CN_match

The previous three options focused on verifying the certificate presented by 

the server. They do not, however, tell us if the verified certificate is valid 

for the domain name or IP address we are requesting, i.e. the host part of the 

URL. To ensure that the certificate is tied to the current domain/IP, we need 

to perform Host Verification. In PHP, this requires setting ``CN_match`` in 

the SSL Context to the HTTP host value (including subdomain part if present!). 

PHP performs the matching internally so long as this option is set.

Not performing this check would allow an MitM to present a valid certificate 

(which they can easily apply for on a domain under their control) and reuse it 

during an attack to ensure they are presenting a certificate signed by a 

trusted CA. However, such a certificate would only be valid for their domain - 

and not the one you are seeking to connect to. Setting the ``CN_match`` option 

will detect such certificate mismatches and cause the HTTPS request to fail. 
While such a valid certificate used by an attacker would contain identity 

information specific to the attacker (a precondition of getting one!), please 

bear in mind that there are undoubtedly any number of valid CA-signed 

certificates, complete with matching private keys, available to a 

knowledgeable attacker. These may have been stolen from another company or 

slipped passed a trusted CA's radar as happened in 2011 when DigiNotor 

notoriously (sorry, couldn't resist) issued a certificate for ``google.com`` 

to an unknown party who went on to employ it in MitM attacks predominantly 

against Iranian users.

Limitations
^^^^^^^^^^^

As described above, verifying that the certificate presented by a server is 

valid for the host in the URL that you're using ensures that a MitM cannot 

simply present any valid certificate they can purchase or illegally obtain. 

This is an essential step, one of four, to ensuring your connection is 

absolutely secure.

The ``CN_match`` parameter exposed by the ``SSL Context`` in PHP's HTTPS 

wrapper tells PHP to perform this matching exercise but it has a downside. At 

the time of writing, the matching used will only check the Common Name (CN) of 

the SSL certificate but ignore the equally valid Subject Alternative Names 

(SANs) field if defined by the certificate. An SAN lets you protect multiple 

domain names with a single SSL certificate so it's extremely useful and 

supported by all modern browsers. Since PHP does not currently support SAN 

matching, connections over SSL/TLS to a domain secured using such a 

certificate will fail.

The CURL extension, on the other hand, supports SANs out of the box so it is 

far more reliable and should be used in preference to PHP's built in 

HTTPS/FTPS wrappers. Using PHP Streams with this issue introduces a greater 

risk of erroneous behaviour which in turn would tempt impatient programmers to 

disable host verification altogether which is the very last thing we want to 

see.

SSL Context in PHP Sockets
^^^^^^^^^^^^^^^^^^^^^^^^^^

Many HTTP clients in PHP will offer both a CURL adapter and a default PHP 

Socket based adapter. The default choice for using sockets reflects the fact 

that CURL is an optional extension and may be disabled on any given server in 

the wild.

PHP Sockets use the same ``SSL Context`` resource as PHP Streams so it 

inherits all of the problems and limitations described earlier. This has the 

side-effect that many major HTTP clients are themselves, by default, likely to 

be unreliable and less safe than they should be. Such client libraries should, 

where possible, be configured to use their CURL adapter if available. You 

should also review such clients to ensure they are not disabling (or 

forgetting to enable) the correct approach to secure SSL/TLS.

CURL Extension
--------------

Unlike PHP Streams, the CURL extension is all about performing data transfers 

including its most commonly known capability for HTTP requests. Also unlike 

PHP Streams' SSL context, CURL is configured by default to make requests 

securely over SSL/TLS. You don't need to do anything special unless it was 

compiled without the location of a Certificate Authority cert bundle (e.g. a 

cacert.pem or ca-bundle.crt file containing the certs for trusted CAs).

Since it requires no special treatment, you can perform a similar Twitter API 

call to what we used earlier for SSL/TLS over a PHP Stream with a minimum of 

fuss and without worrying about missing options that will make it vulnerable 

to MitM attacks.

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $req = curl_init($url);
    curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
    $result = curl_exec($req);

This is why my recommendation to you is to prefer CURL for HTTPS requests. 

It's secure by default whereas PHP Streams is most definitely not. If you feel 

comfortable setting up SSL context options, then feel free to use PHP Streams. 

Otherwise, just use CURL and avoid the headache. At the end of the day, CURL 

is safer, requires less code, and is less likely to suffer a human-error 

related failure in its SSL/TLS security.

Of course, if the CURL extension was enabled without the location of trusted 

certificate bundle being configured, the above example would still fail. For libraries intending to be publicly distributed, the programmer will need to follow a sane pattern which enforces secure behaviour:

.. code-block:: php

    $url = 'https://api.twitter.com/1/statuses/public_timeline.json';
    $req = curl_init($url);
    curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
    $result = curl_exec($req);

    /**
     * Check if an error is an SSL failure and retry with bundled CA certs on
     * the assumption that local server has none configured for ext/curl.
     * Error 77 refers to CURLE_SSL_CACERT_BADFILE which is not defined as
     * as a constant in PHP's manual for some reason.
     */
    $error = curl_errno($req);
    if ($error == CURLE_SSL_PEER_CERTIFICATE || $error == CURLE_SSL_CACERT
    || $error == 77) {
        curl_setopt($req, CURLOPT_CAINFO, __DIR__ . '/cert-bundle.crt');
        $result = curl_exec($req);
    }

    /**
     * Any subsequent errors cannot be recovered from while remaining
     * secure. So do NOT be tempted to disable SSL and try again ;).
     */

The tricky part is obviously distributing the ``cert-bundle.crt`` or ``cafile.pem`` certificate bundle file (filename varies with source!). Given that any Certificate Authority's certificate could be revoked at any time by most browsers should they suffer a breach in their security or peer review processes, it's not really a great idea to allow a certificate file to remain stale for any lengthy period. Nonetheless, the most obvious solution is to distribute a copy of this file with the library or application requiring it.

If you cannot assure tight control over updating a distribute certificate bundle, or you just need a tool that can periodically run this check for you, you should consider using the PHP Sslurp tool: https://github.com/EvanDotPro/Sslurp.
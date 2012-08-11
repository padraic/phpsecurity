XML Injection
=============

Despite the advent of JSON as a lightweight means of communicating data between a server and client, XML remains a viable and popular alternative that is often supported in parallel to JSON by web service APIs. Outside of web services, XML is the foundation of exchanging a diversity of data using XML schemas such as RSS, Atom, and RDF, to name but a few of the more common standards.

XML is so ubiquitous that it can be found in use on the web application server, in browsers as a result of XMLHttpRequest responses, and in browser extensions. Given its widespread use, it can present an attractive target for XML Injection attacks.

XML External Entity Injection
-----------------------------

Vulnerabilities to an XML External Entity Injection (XXE) exist because XML parsing libraries will often support the use of entity references in XML. You'll be familiar with XML's standard complement of entities used to represent special markup characters such as &gt;, &lt;, and &apos;. XML allows you to expand on the standard entity set by defining custom entities within the XML document itself. Custom entities can be defined by including them directly in a DOCTYPE and the expanded value they represent may reference an external resource to be included. For example, let's define a new named entity called "harmless":

.. code-block:: xml

    <!DOCTYPE results [ <!ENTITY harmless "completely harmless"> ]>

An XML document with this entity definition can now refer to the &harmless; entity:

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY harmless "completely harmless">]>
    <results>
        <result>This result is &harmless;</result>
    </results>

An XML parser when interpreting this XML will convert the custom entity to its expanded form as:

    This result is completely harmless

Custom entities obviously have a benefit in representing repetitive text and XML with shorter named entities, however, in keeping with our theme of not trusting outside inputs, we need to be very careful as to what all the XML our application is consuming is really up to. For example, this one is definitely not of the harmless variety:

.. code-block:: xml

    <?xml version="1.0"?>
    <!DOCTYPE results [<!ENTITY harmless SYSTEM "file:///var/www/config.ini">]>
    <results>
        <result>&harmless;</result>
    </results>

Depending on the contents of the requested local file, the content could be used when expanding the &harmless; entity and the expanded content could then be extracted from the XML parser and included in the web application's output for an attacker to examine, i.e. Information Disclosure. The file retrieved will be interpreted as XML unless it avoids the special characters that trigger that interpretation making the scope of local file content disclosure limited. If the file is intepreted as XML but does not contain valid XML, an error will be the likely result preventing disclosure of the contents. PHP, however, has a neat trick available to bypass this scope limitation and remote HTTP requests can still have an impact on the web application even if the returned response cannot be communicated back to the attacker.

PHP offers three frequently used methods of parsing and consuming XML: PHP DOM, SimpleXML and XmlReader. All three of these use the libxml2 extension and external entity support is enabled by default. As a consequence, PHP has a by-default vulnerability to XXE which makes it extremely easy to miss when considering the security of a web application or an XML consuming library. 

Examples of XML External Entity Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File Content And Information Disclosure
"""""""""""""""""""""""""""""""""""""""

Bypassing Access Controls
"""""""""""""""""""""""""

Denial Of Service (DOS)
"""""""""""""""""""""""

Uses of this nature don't look dangerous but let's examine the capability to load custom entities from an external source. This involves the XML parser making a request for a file so fire up your imagination. Take the following example:

.. code-block:: xml

    <!DOCTYPE foo [ <!ENTITY harmless SYSTEM "file:///var/www/config.ini" > ]>
    <results>
        <result>This result is &harmless;</result>
    </results>

In this scenario, the entity is expanded by making a request for the given URI, in this case a target application's configuration file. Who knows, maybe it contains a database credential or two. The entity is expanded with the contents of the file which may then be presented on a web page for an attacker to examine.

Not only could an attacker target the filesystem in such a manner but they can also make HTTP GET requests for an external file:

.. code-block:: xml

    <!DOCTYPE foo [ <!ENTITY harmless SYSTEM "http://127.0.1.1/log.php" > ]>
    <results>
        <result>This result is &harmless;</result>
    </results>

Defenses against XML Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
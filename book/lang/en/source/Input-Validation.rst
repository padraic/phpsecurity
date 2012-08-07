Input Validation
################
 
Input Validation is the outer defensive perimeter for your web application. This perimeter protects the core business logic, processing and output generation. Beyond the perimeter is everything considered potential enemy territory which is...literally everything other than the literal code executed by the current request. All possible entrances and exits on the perimeter are guarded day and night by trigger happy sentries who prefer to shoot first and never ask questions. Connected to this perimeter are separately guarded (and very suspicious looking) "allies" including the Model/Database and Filesystem. Nobody wants to shoot them but if they press their luck...pop. Each of these allies have their own perimeters which may or may not trust ours.
 
Remember what I said about who to trust? As noted in the Introduction, we trust nothing and nobody. The common phrase you will have seen in PHP is to never trust "user input". This is one of those compartmentalising by trust value issues I mentioned. In suggesting that users are untrusted, we imply that everything else is trusted. This is untrue. Users are just the most obvious untrusted source of input since they are known strangers over which we have no control.
 
Validation Considerations
=========================
 
Input validation is both the most fundamental defense that a web application relies upon and the most unreliable. A significant majority of web application vulnerabilities arise from a validation failure, so getting this part of our defenses right is essential. Even where we do appear to have gotten it down, we'll need to be concious of the following considerations.

You should bear these in mind whenever implementing custom validators or adopting a 3rd party validation library. When it comes to 3rd party validators, also consider that these tend to be general in nature and most likely omit key specific validation routines your web application will require. As with any security oriented library, be sure to personally review your preferred library for flaws and limitations. It's also worth bearing in mind that PHP is not above some bizarre arguably unsafe behaviours. Consider the following example from PHP's filter functions:

.. code-block:: php

    filter_var('php://', FILTER_VALIDATE_URL);

The above example passes the filter without issue. The problem with accepting a php:// URL is that it can be passed to PHP functions which expect to retrieve a remote HTTP URL and not to return data from executing PHP (via the PHP wrapper). The flaw in the above is that the filter options have no method of limiting the URI scheme allowed and users' expect this to be one of http, https or mailto rather than some generic PHP specific URI. This is the sort of generic validation approach we should seek to avoid at all costs.
 
Be Wary Of Context
------------------
 
Validating input is intended to prevent the entry of unsafe data into the web application. It has a significant stumbling block in that validation is usually performed to check if data is safe for its first intended use.
 
For example, if I receive a piece of data containing a name, I may validate it fairly loosely to allow for apostrophes, commas, brackets, spaces, and the whole range of alphanumeric Unicode characters (not all of which need literally be alphabetic according to Western languages). As a name, we'd have valid data which can be useful for display purposes (it's first intended use). However, if we use that data elsewhere (e.g. a database query) we will be putting it into a new context. In that new context, some of the characters we allow would still be dangerous - our name might actually be a carefully crafted string intended to perform an SQL Injection attack.
 
The outcome of this is that input validation is inherently unreliable. Input validation works best with extremely restricted values, e.g. when something must be an integer, or an alphanumeric string, or a HTTP URL. Such limited formats and values are least likely to pose a threat if properly validated. Other values such as unrestricted text, GET/POST arrays, and HTML are both harder to validate and far more likely to contain malicious data.
 
Since our application will spend much of its time transporting data between contexts, we can't just validate all input and call it a day. Input validation is our initial defense but never our only one.
 
One of the most common partner defenses used with Input Validation is Escaping (also referred to as Encoding). Escaping is a process whereby data is rendered safe for each new context it enters. While Escaping is usually associated with Cross-Site Scripting, it's also required in many other places where it might be referred to as Filtering instead. Nobody said security terminology was supposed to be consistent, did they?
 
Besides Escaping, which is output oriented to prevent misinterpretation by the receiver, as data enters a new context it should often be greeted by yet another round of context-specific validation.
 
While often perceived as duplication of first-entry validation, additional rounds of input validation are more aware of the current context where validation requirements may differ drastically from the initial round. For example, input into a form might include a percentage integer. At first-entry, we will validate that it is indeed an integer. However, once passed to our application's Model, a new requirement might emerge - the percentage needs to be within a specific range, something only the Model is aware of since the range is a product of the applications business logic. Failing to revalidate in the new context could have some seriously bad outcomes.
 
Never Blacklist; Only Whitelist
-------------------------------
 
The two primary approaches to validating an input are whitelisting and blacklisting. Blacklisting involves checking if the input contains unacceptable data while whitelisting checks if the input contains acceptable data. The reason we prefer whitelisting is that it produces a validation routine that only passes data we expect. Blacklisting, on the other hand, relies on programmers anticipating all possible unexpected data which means it is far easier to run afoul of omissions and errors.
 
A good example here is any validation routine designed to make HTML safe for unescaped output in a template. If we take the blacklisting approach, we need to check that the HTML does not contain dangerous elements, attributes, styles and executable javascript. That accumulates to a large amount of work and all blacklisting oriented HTML sanitisers nearly always tend to forget or omit some dangerous combination of markup. A whitelist based HTML sanitiser dispenses with this uncertainty by only allowing known safe elements and attributes. All other elements and attributes will be stripped out, escaped or deleted regardless of what they are.
 
Since whitelisting tends to be both safer and more robust, it should be preferred for any validation routine.
 
Never Attempt To Fix Input
--------------------------
 
Input validation is frequently accompanied by a related process we call Filtering. Where validation just checks if data is valid (giving either a positive or negative result), Filtering changes the data being validated to meet the validation rules being applied.
 
In many cases, there's little harm in doing this. Common filters might include stripping all but integers out of a telephone number (which may contain extraneous brackets and hyphens), or trimming data of any unneeded horizontal or vertical space. Such use cases are concerned with minimal cleanup of the input to eliminate transcription or transmission type errors. However, it's possible to take Filtering too far into the territory of using Filtering to block the impact of malicious data.
 
One outcome of attempting to fix input is that an attacker may predict the impact your fixes have. For example, let's say a specific string in an input is unacceptable - so you search for it, remove it, and end the filter. What if the attacker created a split string deliberately intended to outwit you?
 
.. code-block:: html

    <scr<script>ipt>alert(document.cookie);</scr<script>ipt>
 
In the above example, naive filtering for a specific tag would achieve nothing since removing the obvious <script> tag actually ensures that the remaining text is now a completely valid HTML script element. The same principle applies to the filtering of any specific format and it underlines also why Input Validation isn't the end of your application's defenses.
 
Rather than attempting to fix input, you should just apply a relevant whitelist validator and reject such inputs - denying them any entry into the web application. Where you must filter, always filter before validation and never after.

Never Trust External Validation Controls But Do Monitor Breaches
----------------------------------------------------------------

In the section on context, I noted that validation should occur whenever data moves into a new context. This applies to validation processes which occur outside of the web application itself. Such controls may include validation or other constraints applied to a HTML form in a browser. Consider the following HTML5 form (labels omitted).

.. code-block:: html
    :linenos:

    <form method="post" name="signup">
        <input name="fname" placeholder="First Name" required />
        <input name="lname" placeholder="Last Name" required />
        <input type="email" name="email" placeholder="someone@example.com" required />
        <input type="url" name="website" required />
        <input name="birthday" type="date" pattern="^d{1,2}/d{1,2}/d{2}$" />
        <select name="country" required>
            <option>Rep. Of Ireland</option>
            <option>United Kingdom</option>
        </select>
        <input type="number" size="3" name="countpets" min="0" max="100" value="1" required />
        <textarea name="foundus" maxlength="140"></textarea> 
        <input type="submit" name="submit" value="Submit" />
    </form>

HTML forms are able to impose constraints on the input used to complete the form. You can restrict choices using a option list, restrict a value using a mininum and maximum allowed number, and set a maximum length for text. HTML5 is even more expressive. Browsers will validate urls and emails, can limit input on date, number and range fields (support for both is sketchy though), and inputs can be validated using a Javascript regular expression included in the pattern attribute.

With all of these controls, it's important to remember that they are intended to make the user experience more consistent. Any attacker can create a custom form that doesn't include any of the constraints in your original form markup. They can even just use a programmed HTTP client to automate form submissions!

Another example of external validation controls may be the constraints applied to the response schema of third-party APIs such as Twitter. Twitter is a huge name and it's tempting to trust them without question. However, since we're paranoid, we really shouldn't. If Twitter were ever compromised, their responses may contain unsafe data we did not expect so we really do need to apply our own validation to defend against such a disaster.

Where we are aware of the external validation controls in place, we may, however, monitor them for breaches. For example, if a HTML form imposes a maxlength attribute but we receive input that exceeds that lenght, it may be wise to consider this as an attempted bypass of validation controls by a user. Using such methods, we could log breaches and take further action to discourage a potential attacker through access denial or request rate limiting.

Evade PHP Type Conversion
-------------------------

PHP is not a strongly typed language and most of its functions and operations are therefore not type safe. This can pose serious problems from a security perspective. Validators are particularly vulnerable to this problem when comparing values. For example:

.. code-block:: php

    assert(0 == '0ABC'); //returns TRUE
    assert(0 == 'ABC'); //returns TRUE (even without starting integer!)
    assert(0 === '0ABC'); //returns NULL/issues Warning as a strict comparison

When designing validators, be sure to prefer strict comparisons and use manual type conversion where input or output values might be strings. Web forms, as an example, always return string data so to work with a resulting expected integer from a form you would have to verify its type:

.. code-block:: php
    :linenos:

    function checkIntegerRange($int, $min, $max)
    {
        if (is_string($int) && !ctype_digit($int)) {
            return false; // contains non digit characters
        }
        if (!is_int((int) $int)) {
            return false; // other non-integer value or exceeds PHP_MAX_INT
        }
        return ($int >= $min && $int <= $max);
    }

You should never do this:

.. code-block:: php
    :linenos:
    
    function checkIntegerRangeTheWrongWay($int, $min, $max)
    {
        return ($int >= $min && $int <= $max);
    }

If you take the second approach, any string which starts with an integer that falls within the expected range would pass validation.

assert(checkIntegerRange("6' OR 1=1", 5, 10)); //issues NULL/Warning correctly
assert(checkIntegerRangeTheWrongWay("6' OR 1=1", 5, 10)); //returns TRUE incorrectly

Type casting naughtiness abounds in many operations and functions such as in_array() which is often used to check if a value exists in an array of valid options.
 
Data Validation Techniques
==========================
 
Failing to validate input can lead to both security vulnerabilities and data corruption. While we are often preoccupied with the former, corrupt data is damaging in its own right. Below we'll examine a number of validation techniques with some examples in PHP.
 
Data Type Check
---------------
 
A Data Type check simply checks whether the data is a string, integer, float, array and so on. Since a lot of data is received through forms, we can't blindly use PHP functions such as is_int() since a single form value is going to be a string and may exceed the maximum integer value that PHP natively supports anyway. Neither should we get too creative and habitually turn to regular expressions since this may violate the KISS principle we prefer in designing security.
 
Allowed Characters Check
------------------------
 
The Allowed Characters check simply ensures that a string only contains valid characters. The most common approaches use PHP's ctype functions and regular expressions for more complex cases. The ctype functions are the best choice where only ASCII characters are allowed.
 
Format Check
------------
 
Format checks ensure that data matches a specific pattern of allowed characters. Emails, URLs and dates are obvious examples here. Best approaches should use PHP's filter_var() function, the DateTime class and regular expressions for other formats. The more complex a format is, the more you should lean towards proven format checks or syntax checking tools.
 
Limit Check
-----------
 
A limit check is designed to test if a value falls within the given range. For example, we may only accept an integer that is greater than 5, or between 0 and 3, or must never be 34. These are all integer limits but a limit check can be applied to string length, file size, image dimensions, date ranges, etc.
 
Presence Check
--------------
 
The presence check ensures that we don't proceed using a set of data if it omits a required value. A signup form, for example, might require a username, password and email address with other optional details. The input will be invalid if any required data is missing.
 
Verification Check
------------------
 
A verification check is when input is required to include two identical values for the purposes of eliminating error. Many signup forms, for example, may require users to type in their requested password twice to avoid any transcription errors. If the two values are identical, the data is valid.
 
Logic Check
-----------
 
The logic check is basically an error control where we ensure the data received will not provoke an error or exception in the application. For example, we may be substituting a search string received into a regular expression. This might provoke an error on compiling the expression. Integers above a certain size may also cause errors, as can zero when we try the divide using it, or when we encounter the weirdness of +0, 0 and -0.
 
Resource Existence Check
------------------------

Resource Existence Checks simply confirms that where data indicates a resource to be used, that the resource actually exists. This is nearly always accompanied by additional checks to prevent the automatic creation of non-existing resources, the diverting of work to invalid resources, and attempts to format any filesystem paths to allow Directory Traversal Attacks.

Validation Of Input Sources
===========================

Despite our best efforts, input validation does not solve all our security problems. Indeed, failures to properly validate input are extremely common. This becomes far more likely in the event that the web application is dealing with a perceived "trusted" source of input data such as a local database. There is not much in the way of additional controls we can place over a database but consider the example of a remote web service protected by SSL or TLS, e.g. by requesting information from the API's endpoints using HTTPS.

HTTPS is a core defense against Man-In-The-Middle (MITM) attacks where an attacker can interject themselves as an intermediary between two parties. As an intermediary, the MITM impersonates a server. Client connections to the server are actually made to the MITM who then makes their own separate connection to the requested server. In this way, a MITM can transfer messages between both parties without their knowledge while still retaining the capacity to read the messages or alter them to the attacker's benefit before they reach their intended destination. To both the server and client, nothing extraordinary has occurred so long as the data keeps flowing.

To prevent this form of attack, it is necessary to prevent an attacker from impersonating the server and from reading the messages they are exchanging. SSL/TLS perform this task with two basic steps:

1. Encrypt all data being transmitted using a shared key that only the server and client have access to.
2. Require that the server prove its identity with a public certificate and a private key that are issued by a trusted Certificate Authority (CA) recognised by the client.

You should be aware that encryption is possible between any two parties using SSL/TLS. In an MITM attack, the client will contact the attacker's server and both will negotiate to enable mutual encryption of the data they will be exchanging. Encryption by itself is useless in this case because we never challenged the MITM server to prove it was the actual server we wanted to contact. That is why Step 2, while technically optional, is actually completely necessary. The web application MUST verify the identity of the server it contacted in order to defend against MITM attacks.

Due to a widespread perception that encryption prevents MITM attacks, many applications and libraries do not apply Step 2. It's both a common and easily detected vulnerability in open source software. PHP itself, due to reasons beyond the understanding of mere mortals, disables server verification by default for its own HTTPS wrapper when using stream_socket_client(), fsockopen() or other internal functions. For example:

.. code-block:: php
    :linenos:

    $body = file_get_contents('https://api.example.com/search?q=sphinx');

The above suffers from an obvious MITM vulnerability and any data resulting from such a HTTPS request can never be considered as representing a response from the intended service. This request should have been made by enabling server verification as follows:

.. code-block:: php
    :linenos:

    $context = stream_context_create(array('ssl' => array('verify_peer' => TRUE)));
    $body = file_get_contents('https://api.example.com/search?q=sphinx', false, $context);

Returning to sanity, the cURL extension does enable server verification out of the box so no option setting is required. However, programmers may demonstrate the following crazy approach to securing their libraries and applications. This one is easy to search for in any libraries your web application will depend on.

.. code-block:: php

    curl_setopt(CURLOPT_SSL_VERIFYPEER, false);

Disabling peer verification in either PHP's SSL context or with curl_setopt() will enable a MITM vulnerability but it's commonly allowed to deal with annoying errors - the sort of errors that may indicate an MITM attack or that the application is attempting to communicate with a host whose SSL certificate is misconfigured or expired.

Web applications can often behave as a proxy for user actions, e.g. acting as a Twitter Client. The least we can do is hold our applications to the high standards set by browsers who will warn their users and do everything possible to prevent users from reaching suspect servers.

Conclusion
==========

TBD
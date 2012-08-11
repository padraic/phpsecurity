Injection Attacks
#################
 
The OWASP Top 10 lists Injection and Cross-Site Scripting (XSS) as the most common security risks to web applications. Indeed, they go hand in hand because XSS attacks are contingent on a successful Injection attack. While this is the most obvious partnership, Injection is not just limited to enabling XSS.
 
Injection is an entire class of attacks that rely on injecting data into a web application in order to facilitate the execution or interpretation of malicious data in an unexpected manner. Examples of attacks within this class include Cross-Site Scripting (XSS), SQL Injection, Header Injection, Log Injection and Full Path Disclosure. I'm scratching the surface here.
 
This class of attacks is every programmer's bogeyman. They are the most common and successful attacks on the internet due to their numerous types, large attack surface, and the complexity sometimes needed to protect against them. All applications need data from somewhere in order to function. Cross-Site Scripting and UI Redress are, in particular, so common that I've dedicated the next chapter to them and these are usually categorised separately from Injection Attacks as their own class given their significance.
 
OWASP uses the following definition for Injection Attacks:
 
Injection flaws, such as SQL, OS, and LDAP injection, occur when untrusted data is sent to an interpreter as part of a command or query. The attacker’s hostile data can trick the interpreter into executing unintended commands or accessing unauthorized data.

.. include:: includes/SQL-Injection.rst
.. include:: includes/Code-Injection.rst
.. include:: includes/Command-Injection.rst
.. include:: includes/Log-Injection.rst
.. include:: includes/Directory-Traversal.rst
.. include:: includes/XML-Injection.rst
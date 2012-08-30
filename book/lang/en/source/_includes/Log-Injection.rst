Log Injection (also Log File Injection)
=======================================

Many applications maintain a range of logs which are often displayable to authorised users from a HTML interface. As a result, they are a prime target for attackers wishing to disguise other attacks, mislead log reviewers, or even mount a subsequent attack on the users of the monitoring application used to read and analyse the logs.

The vulnerability of logs depends on the controls put in place over the writing of logs and ensuring that log data is treated as an untrusted source of data when it comes to performing any monitoring or analysis of the log entries.

A simple log system may write lines of text to a file using file_put_contents(). For example, a programmer might log failed login attempts using a string of the following format:

.. code-block:: php

    sprintf("Failed login attempt by %s", $username);

What if the attacker used a username of the form "Admin\nSuccessful login by Admin\n"?

If this string, from untrusted input were inserted into the log the attacker would have successfully disguised their failed login attempt as an innocent failure by the Admin user to login. Adding a successful retry attempt makes the data even less suspicious.

Of course, the point here is that an attacker can append all manner of log entries. They can also inject XSS vectors, and even inject characters to mess with the display of the log entries in a console.

Goals of Log Injection
----------------------

Injection may also target log format interpreters. If an analyser tool uses regular expressions to parse a log entry to break it up into data fields, an injected string could be carefully constructed to ensure the regex matches an injected surplus field instead of the correct field. For example, the following entry might pose a few problems:

.. code-block:: php

    $username = "iamnothacker! at Mon Jan 01 00:00:00 +1000 2009";
    sprintf("Failed login attempt by $s at $s", $username, )

More nefarious attacks using Log Injection may attempt to build on a Directory Traversal attack to display a log in a browser. In the right circumstances, injecting PHP code into a log message and calling up the log file in the browser can lead to a successful means of Code Injection which can be carefully formatted and executed at will by the attacker. Enough said there. If an attacker can execute PHP on the server, it's game over and time to hope you have sufficient Defense In Depth to minimise the damage.

Defenses Against Log Injection
------------------------------

The simplest defence against Log Injections is to sanitise any outbound log messages using an allowed characters whitelist. We could, for example, limit all logs to alphanumeric characters and spaces. Messages detected outside of this character list may be construed as being corrupt leading to a log message concerning a potential LFI to notify you of the potential attempt. It's a simple method for simple text logs where including any untrusted input in the message is unavoidable.

A secondary defence may be to encode the untrusted input portion into something like base64 which maintains a limited allowed characters profile while still allowing a wide range of information to be stored in text.
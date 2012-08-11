Code Injection (also Remote File Inclusion)
===========================================

Code Injection refers to any means which allows an attacker to inject source code into a web application such that it is interpreted and executed. This does not apply to code injected into a client of the application, e.g. Javascript, which instead falls under the domain of Cross-Site Scripting (XSS).

The source code can be injected directly from an untrusted input or the web application can be manipulated into loading it from the local filesystem or from an external source such a URL. When a Code Injection occurs as the result of including an external resource it is commonly referred to as a Remote File Inclusion though a RFI attack itself need always be intended to inject code.

The primary causes of Code Injection are Input Validation failures, the inclusion of untrusted input in any context where the input may be evaluated as PHP code, failures to secure source code repositories, failures to exercise caution in downloading third-party libraries, and server misconfigurations which allow non-PHP files to be passed to the PHP interpreter by the web server. Particular attention should be paid to the final point as it means that all files uploaded to the server by untrusted users can pose a significant risk.

Examples of Code Injection
--------------------------

PHP is well known for allowing a myriad of Code Injection targets ensuring that Code Injection remains high on any programmer's watch list.

File Inclusion
^^^^^^^^^^^^^^

The most obvious target for a Code Injection attack are the include(), include_once(), require() and require_once() functions. If untrusted input is allowed to determine the path parameter passed to these functions it is possible to influence which local file will be included. It should be noted that the included file need not be an actual PHP file; any included file that is capable of carrying textual data (e.g. almost anything) is allowed.

The path parameter may also be vulnerable to a Directory Traversal or Remote File Inclusion. Using the ../ or ..\ (dot-dot-slash) string in a path allows an attacker to navigate to almost any file accessible to the PHP process. The above functions will also accept a URL in PHP's default configuration unless XXX is disabled.

Evaluation
^^^^^^^^^^

PHP's eval() function accepts a string of PHP code to be executed.

Regular Expression Injection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The PCRE function preg_replace() function in PHP allows for an "e" (PREG_REPLACE_EVAL) modifier which means the replacement string will be evaluated as PHP after subsitution. Untrusted input used in the replacement string could therefore inject PHP code to be executed.

Flawed File Inclusion Logic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Web applications, by definition, will include various files necessary to service any given request. By manipulating the request path or its parameters, it may be possible to provoke the server into including unintended local files by taking advantage of flawed logic in its routing, dependency management, autoloading or other processes.

Such manipulations outside of what the web application was designed to handle can have unforeseen effects. For example, an application might unwittingly expose routes intended only for command line usage. The application may also expose other classes whose constructors perform tasks (not a recommended way to design classes but it happens). Either of these scenarios could interfere with the application's backend operations leading to data manipulation or a potential for Denial Of Service (DOS) attacks on resource intensive operations not intended to be directly accessible.

Server Misconfiguration
^^^^^^^^^^^^^^^^^^^^^^^

Goals of Code Injection
-----------------------

The goal of a Code Injection is extremely broad since it allows the execution of any PHP code of the attacker's choosing.

Defenses against Code Injection
-------------------------------
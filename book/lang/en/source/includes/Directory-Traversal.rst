Directory Traversal (also Path Traversal)
=========================================

Directory Traversal (also Path Traversal) Attacks are attempts to influence the reading from or writing to files by the web application. If untrusted input is used to determine the path to the file, an attacker may inject a ../ or ..\ (dot-dot-slash) string to influence the directory path actually used when the application performs a file operation. If the untrusted input accepted constitutes an absolute path, the dot-dot-slash injection is not required since the input is free to set any path of the attacker's choosing.

In PHP, Directory Traversals also incorporate remote resources since functions such as file_get_contents() may also accept a URI.

Examples of Directory Traversal
-------------------------------

Defenses against Directory Traversal
------------------------------------
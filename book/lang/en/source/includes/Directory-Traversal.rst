Path Traversal (also Directory Traversal)
=========================================

Path Traversal (also Directory Traversal) Attacks are attempts to influence backend operations that read from or write to files in the web application by injecting parameters capable of manipulating the file paths employed by the backend operation. As such, this attack is a stepping stone towards successfully attacking the application by facilitating Information Disclosure and Local/Remote File Injection.

We'll cover these subsequent attack types separately but Path Traversal is one of the root vulnerabilities that enables them all. While the functions described below are specific to the concept of manipulating file paths, it bears mentioning that a lot of PHP functions don't simply accept a file path in the traditional sense of the word. Instead functions like ``include()`` or ``file()`` accept a URI in PHP. This seems completely counterintuitive but it means that the following two function calls using absolute file paths (i.e. not relying on autoloading of relative file paths) are equivalent.

    include('/var/www/vendor/library/Class.php');
    include('file:///var/www/vendor/library/Class.php');

The point here is that relative path handling aside (``include_path`` setting from php.ini and available autoloaders), PHP functions like this are particularly vulnerable to many forms of parameter manipulation including File URI Scheme Substitution where an attacker can inject a HTTP or FTP URI if untrusted data is injected at the start of a file path. We'll cover this in more detail for Remote File Inclusion attacks so, for now, let's focus on filesystem path traversals.

In a Path Traversal vulnerability, the common factor is that the path to a file is manipulated to instead point at a different file. This is commonly achieved by injecting a series of ``../`` (Dot-Dot-Slash) sequences into an argument that is appended to or inserted whole into a function like ``include()``, ``require()``, ``file_get_contents()`` or even less suspicious (for some people) functions such as ``DOMDocument::load()``.

The Dot-Dot-Slash sequence allows an attacker to tell the system to navigate or backtrack up to the parent directory. Thus a path such as ``/var/www/public/../vendor`` actually points to ``/var/www/public/vendor``. The Dot-Dot-Slash sequence after ``/public`` backtracks to that directory's parent, i.e. ``/var/www``. As this simple example illustrates, an attacker can use this to access files which lie outside of the ``/public`` directory that is accessible from the webserver.

Of course, path traversals are not just for backtracking. An attacker can also inject new path elements to access child directories which may be inaccessible from a browser, e.g. due to a ``deny from all`` directive in a ``.htaccess`` in the child directory or one of its parents. Filesystem operations from PHP don't care about how Apache or any other webserver is configured to control access to non-public files and directories.

Examples of Path Traversal
--------------------------



Defenses against Path Traversal
-------------------------------
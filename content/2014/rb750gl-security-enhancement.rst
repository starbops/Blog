==============================
 RB750GL Security Enhancement
==============================

:date: 2014-09-29 15:01
:tags: routeros
:category: memo
:slug: rb750gl-security-enhancement
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: Open the management ports carefully

I found that there are great amount of records about SSH bruteforce from the
Internet in logs. And RouterOS default allow many management ports. To enhance
security of RB750GL, we have to disable some management ports

.. code-block:: sh

    /ip service disable <number>

This will disable both Internet and LAN the ability to access services that
RB750GL provided. If you want to control the accessibility more precisely,
doing this with firewall. By default, all connection from the Internet will be
dropped unless there are rules which accept packets through specific
tranportation layer ports have higher priority than the one which drop packets.


======================
 Vsftpd on Arch Linux
======================

:date: 2014-09-27 01:55
:tags: archlinux, vsftpd, tls
:category: memo
:slug: vsftpd-on-arch-linux
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: Vsftpd is one of the packages in Arch Linux offical repository.

To enable SSL/TLS along with vsftpd, please do the following.

Installation
============

Install vsftpd via pacman

.. code-block:: sh

    # pacman -S vsftpd

Generate an SSL cert

.. code-block:: sh

    # cd /etc/ssl/certs
    # openssl req -x509 -nodes -days 7300 -newkey rsakey:2048 -keyout /etc/ssl/certs/vsftpd.pem -out /etc/ssl/certs/vsftpd.pem
    # chmod 600 /etc/ssl/certs/vsftpd.pem

Make sure the lines below are presented in the ``/etc/vsftpd.conf`` configure file an uncommented

.. code-block:: text

    local_enable=YES

    write_enable=YES

    ssl_enable=YES

    force_local_logins_ssl=YES

    ssl_tlsv1=YES
    ssl_sslv2=YES
    ssl_sslv3=YES
    rsa_cert_file=/etc/ssl/certs/vsftpd.pem
    rsa_private_key_file=/etc/ssl/certs/vsftpd.pem

    require_ssl_reuse=NO

    pasv_min_port=60000
    pasv_max_port=65000

    log_ftp_protocol=YES
    debug_ssl=YES

Fire up vsftpd and make it start at boot time

.. code-block:: sh

    # systemctl enable vsftpd.service
    # systemctl start vsftpd.service

Trouble Shooting
================

Port 21 Occupied by System Default ftpd.service
-----------------------------------------------

In some cases the system provided ftpd.service will be activated. To stop it

.. code-block:: sh

    # systemctl stop ftpd.service
    # systemctl disable ftpd.service

GnuTLS Error -15: An Unexpected TLS Packet Was Received
-------------------------------------------------------

If you uncommented ``chroot_local_user=YES`` in ``/etc/vsftpd.conf``, your FTP client, e.g. FileZilla, will get an error that it cannot explain (decode). I think this is a bug. To workaround this bug, just comment the line and you're done.


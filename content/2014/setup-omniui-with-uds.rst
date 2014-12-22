=======================
 Setup OmniUI with UDS
=======================

:date: 2014-11-25 15:50
:modified: 2014-12-22 16:02
:tags: mininet, omniui, trema, sdn, openflow
:category: memo
:slug: setup-omniui-with-uds
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary:

Mininet
=======

Download Mininet virtual machine image from Mininet's official website. Then
import the uzipped image to your VMware, VirtualBox, etc. Using account/password
both are "mininet" to login into the black box, weeeeee!

OmniUI
======

.. code-block:: bash

    # apt-get install python-pip
    # pip install Flask
    # pip install Flask-Cors
    # pip install pymongo

.. code-block:: bash

    $ git clone https://github.com/dlinknctu/OmniUI.git
    $ cd OmniUI && git checkout dev

Trema-Edge
==========

.. code-block:: bash

    $ cp -R ~/OmniUI/adapter/trema/uds ~/trema-edge/

Install RVM for better ruby and packages management:

.. code-block:: bash

    $ gpg --keyserver hkp://keys.gnupg.net --recv-keys D39DC0E3
    $ \curl -sSL https://get.rvm.io | bash -s stable
    $ echo "source $HOME/.rvm/scripts/rvm" >> ~/.bash_profile

Then re-login you will have rvm.

.. code-block:: bash

    # apt-get install gcc make libpcap-dev libssl-dev
    $ gem install bundler
    $ bundle install
    $ rake

Adapter for Trema-Edge From OmniUI
==================================

Install dependencies:

- json-c

.. code-block:: bash

    $ git clone https://github.com/json-c/json-c.git
    $ cd json-c
    $ sh autogen.sh
    $ ./configure
    $ make
    # make install

- pkg-config

.. code-block:: bash

    # apt-get install pkg-config

.. code-block:: bash

    $ cd ~/trema-edge/uds
    $ make

Trema listen on port 6653. Trema-Edge controller is compatible with OpenFlow 1.3
switch. To turn mininet's OVS to support OpenFlow 1.3, we need to configure all
the OVS spawn by mininet.

.. code-block:: bash

    # ovs-vsctl show | grep Bridge | awk -F "\"" '{print $2}' | xargs -i  ovs-vsctl set bridge {} protocols=OpenFlow10,OpenFlow12,OpenFlow13

References
==========

- `rvm`__
- `json-c`__
- `trema-edge`__

.. __: https://rvm.io/rvm/install
.. __: https://github.com/json-c/json-c
.. __: https://github.com/trema/trema-edge

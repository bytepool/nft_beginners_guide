.. SPDX-License-Identifier: GFDL-1.3-only
   
.. Links, use as netfilter_
.. _netfilter: https://netfilter.org
.. _nftables: https://netfilter.org/projects/nftables/index.html
.. _iptables: https://netfilter.org/projects/iptables/index.html
.. _manpage: https://www.netfilter.org/projects/nftables/manpage.html

.. |disclaimer| replace:: I am in no way an expert on either nftables, Netfilter or any of the affiliated projects, so there is a chance that mistakes have crept in. If you find any mistakes, please let me know and I will fix them as soon as possible.


================================
 A beginner's guide to nftables
================================

.. contents:: Table of contents
    :depth: 4

Introduction
============

Welcome, oh curious one! As the title says, this is a beginner's guide to nftables, the default firewall configuration tool for Linux. If that is of interest to you, read on! If not, but you must work with nftables, read on anyway?

Prerequisite Knowledge
----------------------

I am assuming that you are familiar with Linux in general, comfortable with command line tools, and that you know basic networking. If any of that is not true for you, I strongly recommend that you first read up on those topics and return later; you will not get much out of this tutorial otherwise.


Why this guide?
---------------

While there are a number of beginner's guides for iptables_ and netfilter_, I could not find one that uses ``nftables``. Every ``nftables`` tutorial I have found assumes familiarity with ``iptables``. So if you want to teach students basic firewall concepts with Linux, you either have to accept that there are no introductory texts or start with ``iptables`` before moving on to ``nftables``. This simple guide attempts to bridge this gap. In many ways this is simply a filtered collection of information from various manpages and other sources in a format that aims to be beginner friendly.

.. topic:: Disclaimer

    |disclaimer|

I would also like to point out that the ``nft`` manpage_ is an excellent source of information, and much of the basic concepts presented here are more or less direct reproductions from it. If you are interested in more advanced use cases, I would recommend you start with the manpage_.

Since I borrow heavily from various sources, I have done my best to add appropriate attribution wherever possible. 
    

What is a packet filter?
------------------------

A packet filter, colloquially also known as a firewall, allows an administrator to match network packets with a set of rules that specify how to handle matching packets. For instance you can drop packets, count and log packets, or allow a certain subset of packets. 

A simple rule could be to block all traffic to TCP port 22 except for a specific IP address. Similarly, an administrator could block all outgoing packets to the network 10.0.0.0/8, or if the machine routes packets between networks A and B, the administrator could add a rule that allows all HTTP traffic from network A to B but blocks all other traffic from A to B.

The above are just some simple examples and Netfilter in particular has many more capabilities and features. We will demonstrate several more as we go along. 

What are nftables, nf_tables and Netfilter?
-------------------------------------------

To quote from the ``nft`` manpage, "*nft is the command line tool used to set up, maintain and inspect packet filtering [...] rules in the Linux kernel, in the nftables framework. The Linux kernel subsystem is known as nf_tables, and 'nf' stands for Netfilter.*"

Netfilter_ has been the kernel's packet filtering subsystem since Linux 2.4.x, and for a long time ``iptables``, ``ip6tables``, ``arptables`` and ``ebtables`` have been the main user space utilities to administer the various packet filtering rules. However, their interfaces are less flexible than some people would like, and maintaining larger rulesets can be somewhat cumbersome. Therefore, `nftables`_ was developed and saw its first official release in Linux 3.13.

``nft`` has a completely different syntax than ``iptables``, which might take some getting used to for people accustomed to ``iptables``, but ``nft`` has some nice features that make it worth your while. For instance, ``nft`` can handle all use cases that previously required four separate tools (``iptables``, ``ip6tables``, ``arptables`` and ``ebtables``). Also, instead of having to write two separate rules for a content match in IPv4 and IPv6, ``nft`` can achieve the same match with a single rule.

For more examples why ``nftables`` makes your life easier, see: `Why you will love nftables <https://home.regit.org/2014/01/why-you-will-love-nftables/>`_

My main two complaints about ``nftables`` in its current state are (1) that the documentation is lacking (one of the reasons why I am writing this tutorial), and (2) that the online help of ``nft`` could be improved a lot, meaning the positional arguments to ``nft`` are not always obvious (try for instance ``nft list`` and decide for yourself if the error message is helpful...). It also seems that no one has written a bash completion for ``nftables`` yet [#bash_completion]_...

.. Don't want a footnotes header here .. rubric:: footnotes

.. [#bash_completion] Bored? Always wanted to learn how to write bash_completions? Do the community a favor and write one for ``nft``! See https://github.com/scop/bash-completion/ for more info!




How to enable or reset nftables with systemd
============================================

I am assuming that you already have nftables installed, otherwise install it with your method of choice.

Most GNU/Linux distributions use systemd as their init system now, so I will only cover systemd here. In all likelihood, nftables is already enabled, but if it is not, you can enable it as root with::
		
  systemctl enable nftables.service

If you have changed the config file and want to reload the ruleset, you can run as root::
  
  systemctl restart nftables.service
  

Simple example
==============

Before we get into the basic concepts, let us start with a simple example, just to give you an idea how ``nftables`` works.

Let us start by listing all existing rules::

  nft list ruleset

On a Debian-based system, this might be the output you are seeing::
  
  table inet filter {
	chain input {
		type filter hook input priority filter; policy accept;
	}

	chain forward {
		type filter hook forward priority filter; policy accept;
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
  }

We will discuss this output in detail later, for now note that you have a table called ``filter`` that uses a chain called ``input``.

If you get no output, that means you have no default configuration set up, and you have to add the table and chain first, before you can do anything with it::

  nft add table inet filter
  nft add chain inet filter input { type filter hook input priority 0 \; policy accept\;}

Let us start by adding a rule to block all traffic to port 21 to the input chain::

  nft add rule inet filter input tcp dport 21 counter drop
  
If you are familiar with ``iptables``, note that no interface has been specified, so this rule applies to every network interface on the system. I originally used port 22 here, but I did not want anyone to accidentally lose access to their server if by an off-chance they happened to try this on a remote server with only ssh access...

You could simply test that this rule works by opening a local socket on port 21 (assuming you have no ftp server running), and try to connect::

  # open a listening socket on port 21
  nc -l 21
  # in a new terminal, try to connect to port 21 on localhost
  nc localhost 21

If everything is set up correctly, the client will not be able to connect to the server, so anything you type on the client will not reach the server. If you are not familiar with ``netcat``, you might want to try it without any firewall rules first to see what is supposed to happen.

But there is an even simpler way to test that the rule is being matched: the "counter" keyword in the rule indicates that matches should be counted, so if you run ``nft list ruleset`` before the rule is matched and after the rule is matched, you should see that there is a counter that increases. 

This example was inspired by `theurbanpenguin <https://www.youtube.com/channel/UCFFLP0dKesrKWccYscdAr9A>`_ and `quidsup <https://www.youtube.com/channel/UC0A3ldncnGQ1M_RU2Wb4L2A>`_ on youtube, who each have a nice, short ``nft`` video tutorial:

- `RHCSA 8 - Native Nftables Firewalls on Red Hat Enterprise Linux 8 <https://www.youtube.com/watch?v=Hpfcd7qZUis>`_
- `Getting Started with nftables Firewall in Debian <https://www.youtube.com/watch?v=_A-Q6yTMX0g>`_



Basic concepts
==============


Address families
----------------

Nftables has support for six address families (reproduced from the manpage_):

:ip: IPv4 address family. 
:ip6: IPv6 address family. 
:inet: Internet (IPv4/IPv6) address family. 
:arp: ARP address family, handling IPv4 ARP packets. 
:bridge: Bridge address family, handling packets which traverse a bridge device (switch). 
:netdev: Netdev address family, handling packets from ingress. 

In this guide, we mostly focus on `ip` and `inet`.

.. note::
   
   For some commands, such as chain creation, the address family is optional, but if it is not explicitly specified the address family defaults to **ip**!


Hooks
-----

Hooks allow you to specify at which stage during the packet processing in the network stack you would like to match your rules. The different address families have different hooks, but in this guide we will focus only on IPv4/IPv6/inet and Netdev.


IPv4/IPv6/inet Hooks
~~~~~~~~~~~~~~~~~~~~

There are five hooks you can use for the IPv4, IPv6 and inet address families:

+-------------+--------------------------------------------------------------------------+
| Hook        | Description                                                              |
+=============+==========================================================================+
| prerouting  | All packets entering the system are processed by the prerouting hook.    |
|             | It is invoked before the routing process and is used for early filtering |
|             | or changing packet attributes that affect routing.                       |
+-------------+--------------------------------------------------------------------------+
| input       | Packets delivered to the local system are processed by the input hook.   |
+-------------+--------------------------------------------------------------------------+
| forward     | Packets forwarded to a different host are processed by the forward hook. |
+-------------+--------------------------------------------------------------------------+
| output      | Packets sent by local processes are processed by the output hook.        |
+-------------+--------------------------------------------------------------------------+
| postrouting | All packets leaving the system are processed by the postrouting hook.    |
+-------------+--------------------------------------------------------------------------+

On a regular workstation, you can cover the most common use cases with just the `input` and `output` hooks, and if you use VMs or containers potentially the `forward` hook.



Netdev Hooks
~~~~~~~~~~~~

For the netdev address family, there is a single hook:


+-------------+-------------------------------------------------------------------+
| Hook        | Description                                                       |
+=============+===================================================================+
| ingress     | All packets entering the system are processed by this hook. It is |
|             | invoked before layer 3 protocol handlers and it can be used for   |
|             | early filtering and policing.                                     |
+-------------+-------------------------------------------------------------------+


Tables and chains
-----------------

Tables are containers for chains, and chains are containers for rules, as you have seen in the `simple example`_. 

Tables
~~~~~~

Every table is identified by its name and its address family, so if you have the same table name for different address families, those are separate tables. Try for instance::

  nft add table ip tutorial
  nft add table ip6 tutorial
  nft add table arp tutorial
  nft list ruleset

You should now have three new tables, one for IPv4, one for IPv6 and one for ARP, all named `tutorial`.

You can also list all existing tables with::

  nft list tables

If you do not want the ARP table anymore, you can simply `delete` it::

  nft delete table arp tutorial


Chains
~~~~~~

There are two types of chains:

1. **base chains**, which are hooked into the network stack using the `hooks`_ described earlier, and
2. **regular chains**, which can be used as jump targets from base chains, allowing for better rule organization.

Every chain must belong to a table, so the minimum command to create a new chain looks as follows::

  nft add chain tutorial test-chain

We have now created a new chain called `test-chain` in the table `tutorial`. But wait, if you followed along, you had two tables named `tutorial`, one for the address family `ip` and one for `ip6`! So which one was it added to? The attentive reader will remember that if no address family is specified, it defaults to `ip`. So the above command added `test-chain` only to the `ip tutorial` table. You can easily verify this with::

  nft list ruleset

In order to avoid any confusion, it is probably best to always add the address family as well::

  nft add chain ip6 tutorial test-chain

This successfully added `test-chain` to the `ip6 tutorial` table. So the basic syntax to add a chain is:

.. code-block:: perl
	    
  nft add chain $addr_family $table_name $chain_name

I found this syntax slightly confusing in the beginning: I would have expected the chain name to come directly after `add chain`. But if you think about it, the address family and the table name are necessary to specify the namespace of the chain, so in a way they are part of the chain's name. At the very least, ``nft`` is very consistent, all commands follow this pattern of first specifying the namespace that you want to act upon before any other arguments. 

You may have noticed that we specified no hooks, so `test-chain` is a **regular chain**, and is currently unused.

In order to create a **base chain** that hooks into the network stack, you have to add the desired **hook**, a specific **type** and a desired **priority**. We already covered `hooks`_. The **type** can be one of `filter`, `nat` and `route`. For our purposes in this beginner's guide, the type will always be `filter`. Finally, as the name suggests, **priorities** allow you to specify a priority to determine in which order the chains are traversed.

.. note::

   Since this is a beginner's guide, I will not elaborate any further on types and priorities, but the interested reader is pointed to the manpage_, and for additional background reading to the nice `iptables tutorial <https://rlworkman.net/howtos/iptables/chunkyhtml/index.html>`_, specifically `Chapter 6 Traversing tables and chains <https://rlworkman.net/howtos/iptables/chunkyhtml/c962.html>`_. Just note that the iptables tutorial is outdated, I would not recommend to read it as standalone documentation, make sure to crossreference it with the information in the ``nft`` manpage_. Also note that there are predefined names for certain priority values, so "priority 0" might show up as "priority filter" when you list the chain. 

In the `simple example`_ you already saw how to create a base chain for an input filter. Let us now create an output filter instead::
  
  nft add chain inet filter my-output { type filter hook output priority 0 \; }

This adds a new chain to the table `inet filter`, hooking onto the output hook. In other words, all IP packets that are created on your system and are about to be sent out will traverse this chain. 

.. note::

   When listing the ruleset, you may have noticed that most chains have an entry "policy accept;" . The policy entry denotes the default policy for a base chain, meaning that if no rule match is found for the packet that is being inspected, do what the default policy says. If not specified, the default policy is to accept packets, but you can change the policy to drop if desired. 

Before we move on to rules, let us make one final observation about chains. We already know that chains are containers for rules, but it is important to note that the **rules are ordered and chains are traversed in-order**. 
   
Rules
-----

Rules are the building blocks of any firewall. Rules define conditional actions, such as "if the packet matches pattern 'A' then drop it".

There are generally three parts to a rule:

1. its namespace (address family + table + chain), specifying when and where to match
2. its matching criteria, specifying what to match
3. its action, specifying what to do when a match occurs

Technically, this is not entirely accurate and rules can be much more complex than this, but we will focus on some of the simplest, yet useful, cases. If you want all the nitty-gritty details, see the ``EXPRESSIONS`` (matching criteria) and ``STATEMENTS`` (actions) sections in the manpage_ (covering approximately 3/4 of the manual). 


Adding and deleting a rule
~~~~~~~~~~~~~~~~~~~~~~~~~~

You can add a rule with the ``add rule`` command. Here is a simple example that blocks any packets originating from this machine that are addressed to the network 192.168.0.0/24 (``daddr`` is short for destination address)::

  nft add rule inet filter my-output ip daddr 192.168.0.0/24 drop

If you now try to ping an address in the 192.168.0.0/24 network, you should get an ``Operation not permitted`` error.

Note that ``add`` appends the rule to the chain; you can use ``insert`` instead to insert a new rule at the beginning of the chain. 

What's that? You actually have your router on 192.168.0.1 and you can't reach the internet anymore? Alright, let us delete the rule again. In order to delete a specific rule, you need to know its handle, which you can find using ``nft --handle list`` (short arg: ``-a``)::

  nft -a list chain inet filter my-output

The output should be something like this::

  table inet filter {
	chain my-output { # handle 1
		type filter hook output priority filter; policy accept;
		ip daddr 192.168.0.0/24 drop # handle 2
	}
  }

In my case the handle for the rule is 2, so we can delete the rule like this::

  nft delete rule inet filter my-output handle 2

You can also ``replace`` rules using the handle, please see the manpage_ for details. 


Matching criteria (expressions)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let us examine the previous rule again::

  nft add rule inet filter my-output ip daddr 192.168.0.0/24 drop

Clearly, ``ip daddr 192.168.0.0/24`` is the matching criteria, and of course there are many more possible criteria. I will attempt to highlight some of the most useful matching criteria here:

.. code-block:: bash
		
  ip { protocol | ttl | saddr | daddr }
  ip6 { nexthdr | hoplimit | saddr | daddr }
  icmp { type | code }
  icmp6 { type | code }
  tcp { sport | dport | flags }
  udp { sport | dport }
  iifname
  oifname
  ct state
  
Most of these should be relatively self-explanatory, but here are some pointers:

- ``saddr`` and ``daddr`` stand for source and destination address
- ``sport`` and ``dport`` stand for source port and destination port
- ``iifname`` and ``oifname`` stand for input and output interface name  
- ``ct`` stands for connection tracker

For example, ``iifname "eth0"`` matches on packets that entered through interface ``eth0``, and ``ct state established,related`` matches packets that have a state of established or related in ``nf_tables``' connection tracker. 


Actions (statements)
~~~~~~~~~~~~~~~~~~~~

Once you matched a packet, ``nftables`` needs to know what to do with it. That is where statements come in, they specify the action to be taken. Let me be lazy here and quote from the manpage_ again:

"*Statements represent actions to be performed. They can alter control flow (return, jump to a different chain, accept or drop the packet) or can perform actions, such as logging, rejecting a packet, etc.*"

They go on to say that there are two kind of statements: 

1. "**non-terminal statements** *either only conditionally or never terminate evaluation of the current rule*", whereas
2. "**terminal statements** *unconditionally terminate evaluation of the current rule*"

There can be any number of non-terminal statements in a rule, but there can only be a single terminal statement, which must be the final statement in the rule. If you are familiar with ``iptables``, you may already see the convenience in this. Instead of having to write two rules to log and drop a packet, ``nft`` allows you to write this in a single rule::

  nft add rule inet filter input tcp dport 21 log drop

TODO: CONTINUE HERE with more actions: accept, drop, return, jump, reject, log etc.
  
  
Creating a configuration
========================

There are two main ways of using ``nft`` to configure your firewall:

1. Using the CLI tool ``nft`` (or its interactive mode ``nft -i``) to create tables, chains and rules as you go along.
2. Using a configuration file that you input to ``nft -f`` (system config typically at ``/etc/nftables.conf`` or ``/etc/sysconfig/nftables.conf``). 

Obviously you can first write a basic configuration file, and make dynamic changes with ``nft`` as you see fit.

A nice property of ``nft`` is that the output of ``nft list ruleset`` can be used as valid configuration file, so if you are happy with your current firewall configuration, and you want to override the previous default, you can simply run::

  nft list ruleset > /etc/nftables.conf

and your current configuration will be the new default. You can test this by flushing the ruleset and restarting nftables (see `How to enable or reset nftables with systemd`_). 


Common use cases
================

Blocking ports
--------------

TBD

Blocking specific content
-------------------------

TBD


Allow established connections
-----------------------------

TBD

List of common commands
=======================

TBD

.. code-block:: bash

   ## Listing rules
   # list all rules
   nft list ruleset

   # list all IPv4 rules
   nft list ruleset ip
   
   # list names of all tables
   nft list tables

   # list all chains
   nft list chains

   # list all rules in the given table
   nft list $addr_family $table_name

   ## Tables
   # add a table
   nft add table $table_name

   # delete a table
   nft delete table $table_name

   ## Chains
   # add a chain
   nft add chain $addr_family $table_name $chain_name
   
   # delete a chain
   nft delete chain $addr_family $table_name $chain_name
   
   ## Rules
   # add a rule
   TODO
   
   ## Bulk removal
   # remove (flush) all rules (and all tables and chains)
   nft flush ruleset
  
   # remove (flush) IPv$ rules
   nft flush ruleset ip
  
  
   

Advanced use cases
==================

Just to highlight that you can do much more than the basic cases presented earlier, here are some fun examples of what else is possible with `nftables`.


Port knocking
-------------

TODO: basic description of port knocking.

TODO: port knocking example. 


DDoS protection
---------------

It is difficult to protect against DDoS attacks, but ``nftables`` allows you to filter packets as soon as they arrive to avoid using unnecessary resources, thus giving you a little bit more breathing room. You can use the address family ``netdev`` for this. ``netdev`` includes all traffic directed at your network interface, just after it was passed to the network stack and before any protocol parsing. See also the `nftables Wiki <https://wiki.nftables.org/wiki-nftables/index.php/Nftables_families#netdev>`_.

TODO: example rules here. 


Topics for further exploration
==============================

Since this is a beginner's tutorial, I focused on the most common use cases. However, there are many more features to explore, some I have already hinted at:

- connection tracker (conntrack): `ct helper`, `ct timeout`, `ct expectation`)
- quotas
- `nat` and `route` chain types
- chain priorities
- sets
- maps
- flowtables
- stateful objects
- etc. 


Acknowledgments
===============

A special thanks to the `Linux kernel community <http://www.kernel.org/>`_, as well as the netfilter_ and nftables_ communities, for their much appreciated work! I am also grateful for the people around `Sphinx <https://www.sphinx-doc.org>`_ for having written a great documentation generator. 

Many innovations we see today would not be possible without open source software in general, and together with open standards OSS has an important role to play in democratic societies. Not to mention that computing would be very dull without it! Therefore, **I would also like to thank everyone involved in the open source ecosystem in general** [#oss_thanks]_: please keep up the good work! :)

.. .. rubric:: footnotes

.. [#oss_thanks] If I were to list all the free and open source software I use daily we would still sit here tomorrow. However, I cannot resist listing a few projects and communities especially dear to my heart: `Debian <https://www.debian.org/>`_, `Emacs <https://www.gnu.org/software/emacs/>`_, `GNU <https://www.gnu.org/>`_, `Linux <https://www.kernel.org/>`_, `Python <http://www.python.org/>`_, `LaTeX <https://www.latex-project.org/>`_, `i3 <https://i3wm.org/>`_ and `Mozilla <https://www.mozilla.org/>`_. :)


###############
第1章  基础知识
###############


`官方文档 <http://zguide.zeromq.org/page:all>`_

`PyZMQ Documentation <https://pyzmq.readthedocs.io/en/latest/>`_

API 
    `zmqpp : C++ style API <https://zeromq.github.io/zmqpp/>`_

* Ubuntu 


    .. code-block:: bash

        $ sudo aptitude install libzmq3-dev

修复这个世界
============

本书读者对象
============


获取示例
========

.. code-block:: sh

    $ git clone --depth=1 https://github.com/imatix/zguide.git


问过就必有收获
==============

因此,让我们先从一些代码起步。当然，我们讲以一个"HelloWorld"的例子开始。我们会制作一个客户端和服务器，客户端发送"Hello"到服务器，服务器用"World" 来应答（参见图1-1)。实例给出服务器代码，将5555端口上打开一个ØMQ套接字，读取请求并用"World" 应答每个请求。

.. image:: ../images/fig2.svg
    :scale: 100%
    :alt: alternate text
    :align: center


.. raw:: html

    <p style="text-align:center;">图1-1 请求－应答</p>


**示例 HeloWorld服务器 (hwserver.cpp)**

.. literalinclude:: ../examples/C++/hwserver.cpp
    :language: cpp
    :encoding: utf-8
    :emphasize-lines: 20

示例 HeloWorld客户端代码(hwclient.cpp)


.. literalinclude:: ../examples/C++/hwclient.cpp
    :language: cpp
    :encoding: utf-8

编译

.. code-block:: sh

    $ g++ hwserver.cpp -o hwserver -lzmq
    $ g++ hwclient.cpp -o hwclient -lzmq

REQ-REP套接字对是步调一致的。客户端在一个循环中（或一次，根据需要而定）先发出 send(),然后在发出recv().任何其他序列（例如,一行中发送两个消息）将导致从send或者recv代码返回-1.同样的，服务器先发出recv(),然后再发出send ,按照这个顺序，根据需要多次重复。

在字符串上的小注解
==================

当从ØMQ用C接收字符串数据时，你根本无法相信他是安全的终止的。每一次你读到一个字符串时，都应该分配一个包含一个额外字节的新缓冲区，复制该字符串，并用正确的空字符串来终止它。

版本报告
========

.. literalinclude:: ../examples/C/version.c
    :language: c
    :encoding: utf-8
    :linenos:
    :lines: 1-11
    :emphasize-lines: 3,7-8

获取消息
========

第二个经典的模式，是单向的数据发布，其中一台服务器将更新推送到一组客户端。
让我们看一个天气更新的例子，包括邮政编码，温度和相对湿度的例子。我们将使用随机值，就像真正的气象站那样。

天气更新服务端: wuserver

.. literalinclude:: ../examples/C/wuserver.c
    :language: c
    :encoding: utf-8

这个气象发布，是一个不停广播的死循环.

.. image:: ../images/fig4.svg
    :scale: 100%
    :alt: alternate text
    :align: center

.. raw:: html

    <p style="text-align:center;">图 发布-订阅</p>

天气更新客户端: wuclient

.. literalinclude:: ../examples/C/wuclient.c
    :language: c
    :encoding: utf-8

.. code-block:: bash

    $ g++ wuserver.c -o wuserver -lzmq
    $ g++ wuclient.c -o wuclient -lzmq

请注意，当你使用一个SUB套接字时，必须使用　zmq_setsockopt() 和 SUBSCRIBE设置一个订阅，如上面的代码所示。如果你没有设置任何订阅，就不会得到任何消息。这是初学者常犯的一个错误。订阅者多会接收到它。定业者也可以取消特定的订阅。订阅经常但不一定是一个可打印的字符串。要了解这是如何工作的。请参见 zmq_setsockopt_ ().

.. _zmq_setsockopt: http://api.zeromq.org/4-2:zmq-setsockopt

该PUB-SUB 套接字对是异步的。客户端在一个循环中（或一次，根据需要而定）执行zmq_msg_recv(). 试图将消息发送到一个SUB套接字会导致错误。同样，服务根据需要多次执行zmq_msg_send(). 但不能在PUB套接字上执行zmq_msg_recv().

从理论上讲，使用ZMQ套接字不关心哪端连接和哪端绑定。然而，在实践中有未在文档中记录的差异，我稍后就会提到它。就目前而言，只要绑定PUB并连接SUB就可以了，除非你的网络设计是的这行不通。

关于PUB-SUB套接字还有一件更重要的事情需要了解: 你不知道订阅者开始得到消息的精确时间。即使你启动一个订阅者，稍等片刻，然后再启动发布者， **订阅者也总会错过发布者发送的第一个消息。** 这是因为当订阅者连接到发布者时，（这需要的时间很短，但非零），发布者可能已经将消息发送出去了。 

这种"慢木匠"症状经常会击中足够多的人，那我们要详细解释一下。请记住，ZMQ执行异步I/O(即台后).假设你有两个节点执行此操作，顺序如下:

    * 订阅者连接到一个端点，并接收和清点消息。
    * 发布者绑定到一个端点，并立即发送1000条消息。 

订阅者很可能不会收到任何东西。你会眨眼，检查你是否设定了一个正确的过滤器，然后再试一次，而订阅者任然没有收到任何东西。

* 一个订阅者可以连接到多个发布者，每次使用一个connect 调用。那么数据将交错到达（"公平排队"),因此，没有任何一个发布者能淹没其他发布者。

分而治之
========


.. image:: ../images/fig5.svg
    :scale: 100%
    :alt: alternate text
    :align: center



.. literalinclude:: ../examples/C++/taskvent.cpp
    :language: cpp
    :encoding: utf-8

 
.. literalinclude:: ../examples/C++/taskwork.cpp
    :language: cpp
    :encoding: utf-8


.. literalinclude:: ../examples/C++/tasksink.cpp
    :language: cpp
    :encoding: utf-8
    
.. code-block:: sh

    $ g++ taskvent.cpp -o taskvent -lzmq
    $ g++ taskwork.cpp -o taskwork -lzmq
    $ g++ tasksink.cpp -o tasksink -lzmq
     

用ØMQ编程
=========

获取正确的上下文
================

为什么我们需要ØMQ
=================

套接字的可扩展性
================


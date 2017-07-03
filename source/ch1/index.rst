###############
第1章  基础知识
###############


`官方文档 <http://zguide.zeromq.org/page:all>`_


* Ubuntu 

    .. code-block:: sh

        $ sudo aptitude install libzmq3-dev

获取示例
========

.. code-block:: sh

    $ git clone --depth=1 https://github.com/imatix/zguide.git


问过就必有收获
==============

.. image:: ./images/fig2.png
    :scale: 100%
    :alt: alternate text
    :align: center


示例 HeloWorld服务器(hwserver.cpp)


.. code-block:: cpp

    //
    //  Hello World server in C++
    //  Binds REP socket to tcp://*:5555
    //  Expects "Hello" from client, replies with "World"
    //
    #include <zmq.hpp>
    #include <string>
    #include <iostream>
    #ifndef _WIN32
    #include <unistd.h>
    #else
    #include <windows.h>

    #define sleep(n)	Sleep(n)
    #endif

    int main () {
        //  Prepare our context and socket
        zmq::context_t context (1);
        zmq::socket_t socket (context, ZMQ_REP);
        socket.bind ("tcp://*:5555");

        while (true) {
            zmq::message_t request;

            //  Wait for next request from client
            socket.recv (&request);
            std::cout << "Received Hello" << std::endl;

            //  Do some 'work'
            sleep(1);

            //  Send reply back to client
            zmq::message_t reply (5);
            memcpy (reply.data (), "World", 5);
            socket.send (reply);
        }
        return 0;
    }



示例 HeloWorld客户端代码(hwclient.cpp)


.. code-block:: cpp

    //
    //  Hello World client in C++
    //  Connects REQ socket to tcp://localhost:5555
    //  Sends "Hello" to server, expects "World" back
    //
    #include <zmq.hpp>
    #include <string>
    #include <iostream>

    int main ()
 
    {
        //  Prepare our context and socket
        zmq::context_t context (1);
        zmq::socket_t socket (context, ZMQ_REQ);

        std::cout << "Connecting to hello world server..." << std::endl;
        socket.connect ("tcp://localhost:5555");

        //  Do 10 requests, waiting each time for a response
        for (int request_nbr = 0; request_nbr != 10; request_nbr++) {
            zmq::message_t request (5);
            memcpy (request.data (), "Hello", 5);
            std::cout << "Sending Hello " << request_nbr << "..." << std::endl;
            socket.send (request);

            //  Get the reply.
            zmq::message_t reply;
            socket.recv (&reply);
            std::cout << "Received World " << request_nbr << std::endl;
        }
        return 0;
    }


编译

.. code-block:: sh

    $ g++ hwserver.cpp -o hwserver -lzmq
    $ g++ hwclient.cpp -o hwclient -lzmq





在字符串上的小注解
==================

版本报告
========

分而治之
========

用ØMQ编程
=========

获取正确的上下文
================

为什么我们需要ØMQ
=================

套接字的可扩展性
================














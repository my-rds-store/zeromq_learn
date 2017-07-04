###############
第1章  基础知识
###############


`官方文档 <http://zguide.zeromq.org/page:all>`_

`PyZMQ Documentation <https://pyzmq.readthedocs.io/en/latest/>`_

`zmqpp : C++ style API <https://zeromq.github.io/zmqpp/>`_

* Ubuntu 

    .. code-block:: sh

        $ sudo aptitude install libzmq3-dev

获取示例
========

.. code-block:: sh

    $ git clone --depth=1 https://github.com/imatix/zguide.git


问过就必有收获
==============

因此,让我们先从一些代码起步。当然，我们讲以一个"HelloWorld"的例子开始。我们会制作一个客户端和服务器，客户端发送"Hello"到服务器，服务器用"World" 来应答（参见图1-1)。实例给出服务器代码，将5555端口上打开一个ØMQ套接字，读取请求并用"World" 应答每个请求。

.. image:: ./images/fig2.png
    :scale: 100%
    :alt: alternate text
    :align: center

*图1-1 请求-应答*
 

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

REQ-REP套接字对是步调一致的。客户端在一个循环中（或一次，根据需要而定）先发出 send(),然后在发出recv().任何其他序列（例如,一行中发送两个消息）将导致从send或者recv代码返回-1.同样的，服务器先发出recv(),然后再发出send ,按照这个顺序，根据需要多次重复。

在字符串上的小注解
==================

当从ØMQ用C接收字符串数据时，你根本无法相信他是安全的终止的。每一次你读到一个字符串时，都应该分配一个包含一个额外字节的新缓冲区，复制该字符串，并用正确的空字符串来终止它。

版本报告
========

.. code-block:: cpp

    #include <zmq.hpp>
    #include <iostream>

    void s_version (void)
    {
        int major, minor, patch;
        zmq_version (&major, &minor, &patch);
        std::cout << "Current 0MQ version is " << major << "." << minor << "." << patch << std::endl;
    }

    int main ()
    {
        s_version ();
        return EXIT_SUCCESS;
    }


获取消息
========


.. image:: ./images/fig4.png
    :scale: 100%
    :alt: alternate text
    :align: center

*图 发布-订阅*
 

.. code-block:: cpp

    //  Weather update server in C++
    //  Binds PUB socket to tcp://*:5556
    //  Publishes random weather updates
    //
    //  Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>
    //
    #include <zmq.hpp>
    #include <stdio.h>
    #include <stdlib.h>
    #include <time.h>

    #if (defined (WIN32))
    #include <zhelpers.hpp>
    #endif

    #define within(num) (int) ((float) num * random () / (RAND_MAX + 1.0))

    int main () {

        //  Prepare our context and publisher
        zmq::context_t context (1);
        zmq::socket_t publisher (context, ZMQ_PUB);
        publisher.bind("tcp://*:5556");
        publisher.bind("ipc://weather.ipc");				// Not usable on Windows.

        //  Initialize random number generator
        srandom ((unsigned) time (NULL));
        while (1) {

            int zipcode, temperature, relhumidity;

            //  Get values that will fool the boss
            zipcode     = within (100000);
            temperature = within (215) - 80;
            relhumidity = within (50) + 10;

            //  Send message to all subscribers
            zmq::message_t message(20);
            snprintf ((char *) message.data(), 20 ,
                    "%05d %d %d", zipcode, temperature, relhumidity);
            publisher.send(message);

        }
        return 0;
    }


.. code-block:: cpp

    //  Weather update client in C++
    //  Connects SUB socket to tcp://localhost:5556
    //  Collects weather updates and finds avg temp in zipcode
    //
    //  Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>
    //
    #include <zmq.hpp>
    #include <iostream>
    #include <sstream>

    int main (int argc, char *argv[])
    {
        zmq::context_t context (1);

        //  Socket to talk to server
        std::cout << "Collecting updates from weather server...\n" << std::endl;
        zmq::socket_t subscriber (context, ZMQ_SUB);
        subscriber.connect("tcp://localhost:5556"); 
        // OR : subscriber.connect("ipc://weather.ipc");

        //  Subscribe to zipcode, default is NYC, 10001
        const char *filter = (argc > 1)? argv [1]: "10001 ";
        subscriber.setsockopt(ZMQ_SUBSCRIBE, filter, strlen (filter));

        //  Process 100 updates
        int update_nbr;
        long total_temp = 0;
        for (update_nbr = 0; update_nbr < 100; update_nbr++) {

            zmq::message_t update;
            int zipcode, temperature, relhumidity;

            subscriber.recv(&update);

            std::istringstream iss(static_cast<char*>(update.data()));
            iss >> zipcode >> temperature >> relhumidity ;

            total_temp += temperature;
        }
        std::cout << "Average temperature for zipcode '"<< filter
                  <<"' was "<<(int) (total_temp / update_nbr) <<"F"
                  << std::endl;
        return 0;
    }

.. code-block:: sh

    $ g++ wuserver.cpp -o wuserver -lzmq
    $ g++ wuclient.cpp -o wuclient -lzmq


关于PUB-SUB套接字还有一件更重要的事情需要了解: 你不知道订阅者开始得到消息的精确时间。即使你启动一个订阅者，稍等片刻，然后再启动发布者， **订阅者也总会错过发布者发送的第一个消息。** 这是因为当订阅者连接到发布者时，（这需要的时间很短，但非零），发布者可能已经将消息发送出去了。 

* 一个订阅者可以连接到多个发布者，每次使用一个connect 调用。那么数据将交错到达（"公平排队"),因此，没有任何一个发布者能淹没其他发布者。

分而治之
========

.. code-block:: cpp

    //  Task ventilator in C++
    //  Binds PUSH socket to tcp://localhost:5557
    //  Sends batch of tasks to workers via that socket
    //
    //  Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>
    //
    #include <zmq.hpp>
    #include <stdlib.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <iostream>

    #define within(num) (int) ((float) num * random () / (RAND_MAX + 1.0))

    int main (int argc, char *argv[])
    {
        zmq::context_t context (1);

        //  Socket to send messages on
        zmq::socket_t  sender(context, ZMQ_PUSH);
        sender.bind("tcp://*:5557");

        std::cout << "Press Enter when the workers are ready: " << std::endl;
        getchar ();
        std::cout << "Sending tasks to workers…\n" << std::endl;

        //  The first message is "0" and signals start of batch
        zmq::socket_t sink(context, ZMQ_PUSH);
        sink.connect("tcp://localhost:5558");
        zmq::message_t message(2);
        memcpy(message.data(), "0", 1);
        sink.send(message);

        //  Initialize random number generator
        srandom ((unsigned) time (NULL));

        //  Send 100 tasks
        int task_nbr;
        int total_msec = 0;     //  Total expected cost in msecs
        for (task_nbr = 0; task_nbr < 100; task_nbr++) {
            int workload;
            //  Random workload from 1 to 100msecs
            workload = within (100) + 1;
            total_msec += workload;

            message.rebuild(10);
            memset(message.data(), '\0', 10);
            sprintf ((char *) message.data(), "%d", workload);
            sender.send(message);
        }
        std::cout << "Total expected cost: " << total_msec << " msec" << std::endl;
        sleep (1);              //  Give 0MQ time to deliver

        return 0;
    }


.. code-block:: cpp 

    //  Task worker in C++
    //  Connects PULL socket to tcp://localhost:5557
    //  Collects workloads from ventilator via that socket
    //  Connects PUSH socket to tcp://localhost:5558
    //  Sends results to sink via that socket
    //
    //  Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>
    //
    #include "zhelpers.hpp"
    #include <string>

    int main (int argc, char *argv[])
    {
        zmq::context_t context(1);

        //  Socket to receive messages on
        zmq::socket_t receiver(context, ZMQ_PULL);
        receiver.connect("tcp://localhost:5557");

        //  Socket to send messages to
        zmq::socket_t sender(context, ZMQ_PUSH);
        sender.connect("tcp://localhost:5558");

        //  Process tasks forever
        while (1) {

            zmq::message_t message;
            int workload;           //  Workload in msecs

            receiver.recv(&message);
            std::string smessage(static_cast<char*>(message.data()), message.size());

            std::istringstream iss(smessage);
            iss >> workload;

            //  Do the work
            s_sleep(workload);

            //  Send results to sink
            message.rebuild();
            sender.send(message);

            //  Simple progress indicator for the viewer
            std::cout << "." << std::flush;
        }
        return 0;
    }


.. code-block:: cpp

    //  Task sink in C++
    //  Binds PULL socket to tcp://localhost:5558
    //  Collects results from workers via that socket
    //
    //  Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>
    //
    #include <zmq.hpp>
    #include <time.h>
    #include <sys/time.h>
    #include <iostream>

    int main (int argc, char *argv[])
    {
        //  Prepare our context and socket
        zmq::context_t context(1);
        zmq::socket_t receiver(context,ZMQ_PULL);
        receiver.bind("tcp://*:5558");

        //  Wait for start of batch
        zmq::message_t message;
        receiver.recv(&message);

        //  Start our clock now
        struct timeval tstart;
        gettimeofday (&tstart, NULL);

        //  Process 100 confirmations
        int task_nbr;
        int total_msec = 0;     //  Total calculated cost in msecs
        for (task_nbr = 0; task_nbr < 100; task_nbr++) {

            receiver.recv(&message);
            if ((task_nbr / 10) * 10 == task_nbr)
                std::cout << ":" << std::flush;
            else
                std::cout << "." << std::flush;
        }
        //  Calculate and report duration of batch
        struct timeval tend, tdiff;
        gettimeofday (&tend, NULL);

        if (tend.tv_usec < tstart.tv_usec) {
            tdiff.tv_sec = tend.tv_sec - tstart.tv_sec - 1;
            tdiff.tv_usec = 1000000 + tend.tv_usec - tstart.tv_usec;
        }
        else {
            tdiff.tv_sec = tend.tv_sec - tstart.tv_sec;
            tdiff.tv_usec = tend.tv_usec - tstart.tv_usec;
        }
        total_msec = tdiff.tv_sec * 1000 + tdiff.tv_usec / 1000;
        std::cout << "\nTotal elapsed time: " << total_msec << " msec\n" << std::endl;
        return 0;
    }

    
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


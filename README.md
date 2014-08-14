py_hearbeat
====

This is from http://code.activestate.com/recipes/52302-pyheartbeat-detecting-inactive-computers/ 

Here is the how-to:



How it works

When we have a number of computers, we are often interested in monitoring their working state. It is possible to detect when a computer stops working by using a pair of programs, one client and one server.

The client program, running on any number of computers, periodically sends an UDP packet to the server program, listening on one computer. The server program dinamically builds a dictionary that stores the IP addresses of the client computers, and the time stamp of the last packet received from each one. At the same time it periodically checks the dictionary, checking whether any of the time stamps is older than a defined timeout.

In this kind of application there is no need to use reliable TCP connections, since the loss of a packet now and then does not produce false alarms, given that the server checking timeout is kept suitably larger than the client sending period. On the other hand, if we have hundreds of computers to monitor, it is preferable to keep the bandwith used and the load on the server at a minimum. We obtain this by periodically sending a small UDP packet, instead of setting up a comparably expensive TCP connection each time.

The packets are sent from each client with a period of five seconds, while the server checks the dictionary with a period of twenty seconds, and its timeout is set to fifteen seconds. These parameters, along with the server IP address and port used, may be configured to one's needs.

Threaded server

In the threaded server, one thread listens to the UDP packets coming from the clients, while the main thread periodically checks the recorded heartbeats. The shared data structure, a dictionary, must be locked and released at each access, both while writing and reading, to avoid data corruption on concurrent access. Such data corruption often manifests itself as intermittent, time-dependent bugs that are difficult to reproduce, investigate and correct.

Twisted server

The Twisted server employs an asynchronous, event driven model, being based on the Twisted Matrix framework ( http://www.twistedmatrix.com/ ). The framework is built around a central "reactor" that dispatches events from a queue in a single thread, and monitors network and host resources. The user program is composed of short code fragments invoked by the reactor when dispatching the matching events. Such a working model guarantees that only one user code fragment is being executed at any given time, eliminating at the root all problems of concurrent access to shared data structures.

The server program is composed of an Application and two Services, the UDPServer and the DetectorService. It is invoked by means of the "twistd" command, with the following options:

$ twistd -ony TwistedBeatServer.py

See the Twisted Matrix documentation for further information.

Versions

This program has been tested on Python 2.3.4 and Twisted 1.3.0 . It will work on Python 2.2 by substituting the three occurrences of the "super" keyword in the ThreadedBeatServer.py file with the corresponding old form.

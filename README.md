py_hearbeat
====


Here is the brife code view:
```python
# Filename: HeartbeatClient.py

"""Heartbeat client, sends out an UDP packet periodically"""

import socket, time

SERVER_IP = '127.0.0.1'; SERVER_PORT = 43278; BEAT_PERIOD = 5

print ('Sending heartbeat to IP %s , port %d\n'
    'press Ctrl-C to stop\n') % (SERVER_IP, SERVER_PORT)
while True:
    hbSocket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    hbSocket.sendto('PyHB', (SERVER_IP, SERVER_PORT))
    if __debug__: print 'Time: %s' % time.ctime()
    time.sleep(BEAT_PERIOD)

--- 8< --- snip --- 8< --- snip --- 8< --- snip --- 8< ---

# Filename: ThreadedBeatServer.py

"""Threaded heartbeat server"""

UDP_PORT = 43278; CHECK_PERIOD = 20; CHECK_TIMEOUT = 15

import socket, threading, time

class Heartbeats(dict):
    """Manage shared heartbeats dictionary with thread locking"""

    def __init__(self):
        super(Heartbeats, self).__init__()
        self._lock = threading.Lock()

    def __setitem__(self, key, value):
        """Create or update the dictionary entry for a client"""
        self._lock.acquire()
        super(Heartbeats, self).__setitem__(key, value)
        self._lock.release()

    def getSilent(self):
        """Return a list of clients with heartbeat older than CHECK_TIMEOUT"""
        limit = time.time() - CHECK_TIMEOUT
        self._lock.acquire()
        silent = [ip for (ip, ipTime) in self.items() if ipTime < limit]
        self._lock.release()
        return silent

class Receiver(threading.Thread):
    """Receive UDP packets and log them in the heartbeats dictionary"""

    def __init__(self, goOnEvent, heartbeats):
        super(Receiver, self).__init__()
        self.goOnEvent = goOnEvent
        self.heartbeats = heartbeats
        self.recSocket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.recSocket.settimeout(CHECK_TIMEOUT)
        self.recSocket.bind((socket.gethostbyname('localhost'), UDP_PORT))

    def run(self):
        while self.goOnEvent.isSet():
            try:
                data, addr = self.recSocket.recvfrom(5)
                if data == 'PyHB':
                    self.heartbeats[addr[0]] = time.time()
            except socket.timeout:
                pass

def main():
    receiverEvent = threading.Event()
    receiverEvent.set()
    heartbeats = Heartbeats()
    receiver = Receiver(goOnEvent = receiverEvent, heartbeats = heartbeats)
    receiver.start()
    print ('Threaded heartbeat server listening on port %d\n'
        'press Ctrl-C to stop\n') % UDP_PORT
    try:
        while True:
            silent = heartbeats.getSilent()
            print 'Silent clients: %s' % silent
            time.sleep(CHECK_PERIOD)
    except KeyboardInterrupt:
        print 'Exiting, please wait...'
        receiverEvent.clear()
        receiver.join()
        print 'Finished.'

if __name__ == '__main__':
    main()

--- 8< --- snip --- 8< --- snip --- 8< --- snip --- 8< ---

# Filename: TwistedBeatServer.py

"""Asynchronous events-based heartbeat server"""

UDP_PORT = 43278; CHECK_PERIOD = 20; CHECK_TIMEOUT = 15

import time
from twisted.application import internet, service
from twisted.internet import protocol
from twisted.python import log

class Receiver(protocol.DatagramProtocol):
    """Receive UDP packets and log them in the clients dictionary"""

    def datagramReceived(self, data, (ip, port)):
        if data == 'PyHB':
            self.callback(ip)

class DetectorService(internet.TimerService):
    """Detect clients not sending heartbeats for too long"""

    def __init__(self):
        internet.TimerService.__init__(self, CHECK_PERIOD, self.detect)
        self.beats = {}

    def update(self, ip):
        self.beats[ip] = time.time()

    def detect(self):
        """Log a list of clients with heartbeat older than CHECK_TIMEOUT"""
        limit = time.time() - CHECK_TIMEOUT
        silent = [ip for (ip, ipTime) in self.beats.items() if ipTime < limit]
        log.msg('Silent clients: %s' % silent)

application = service.Application('Heartbeat')
# define and link the silent clients' detector service
detectorSvc = DetectorService()
detectorSvc.setServiceParent(application)
# create an instance of the Receiver protocol, and give it the callback
receiver = Receiver()
receiver.callback = detectorSvc.update
# define and link the UDP server service, passing the receiver in
udpServer = internet.UDPServer(UDP_PORT, receiver)
udpServer.setServiceParent(application)
# each service is started automatically by Twisted at launch time
log.msg('Asynchronous heartbeat server listening on port %d\n'
    'press Ctrl-C to stop\n' % UDP_PORT)
```

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









```sequence
participant Browser
participant HAProxy
participant S1
participant S2

Note left of Browser: First request\nAlice
Browser->HAProxy :GET / HTTP/1.1
HAProxy->S1:GET / HTTP/1.1
S1->HAProxy :HTTP/1.1, NODESESSID=xxx
HAProxy->Browser : HTTP/1.1, NODESESSID=xxx, SERVERID=cs1

Note left of Browser: Second request\nAlice
Browser->HAProxy:GET / HTTP/1.1, NODESESSID=xxx, SERVERID=cs1
Note over HAProxy: With SERVERID I know where\nto send the request
HAProxy->S1: GET / HTTP/1.1, NODESESSID=xxx
S1->HAProxy :HTTP/1.1
HAProxy->Browser : HTTP/1.1

Note left of Browser: Third request\nBob
Browser->HAProxy:GET / HTTP/1.1
HAProxy->S2: GET / HTTP/1.1
S2->HAProxy :HTTP/1.1, NODESESSID=yyy
HAProxy->Browser : HTTP/1.1, NODESESSID=yyy, SERVERID=cs2

```
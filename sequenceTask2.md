





```sequence
participant Browser
participant HAProxy
participant S1
participant S2

Note left of Browser: First request
Browser->HAProxy :GET / HTTP/1.1
HAProxy->S1:GET / HTTP/1.1
S1->HAProxy :HTTP/1.1, NODESESSID=xxx
HAProxy->Browser : HTTP/1.1, NODESESSID=cs1~xxx

Note left of Browser: Second request\nSame user \nthan the first one
Browser->HAProxy:GET / HTTP/1.1, NODESESSID=cs1~xxx
Note over HAProxy: I know this NODESESSID\nIt's linked to S1
HAProxy->S1: GET / HTTP/1.1, NODESESSID=xxx
S1->HAProxy :HTTP/1.1, NODESESSID=xxx
HAProxy->Browser : HTTP/1.1, NODESESSID=cs1~xxx

Note left of Browser: Third request\nNew user
Browser->HAProxy:GET / HTTP/1.1
HAProxy->S2: GET / HTTP/1.1
S2->HAProxy :HTTP/1.1, NODESESSID=yyy
HAProxy->Browser : HTTP/1.1, NODESESSID=cs2~yyy

```
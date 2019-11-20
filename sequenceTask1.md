

```sequence
participant Browser
participant HAProxy
participant S1
participant S2

Note left of Browser: First request
Browser->HAProxy :GET / HTTP/1.1
HAProxy->S1:GET / HTTP/1.1
S1->HAProxy :HTTP/1.1, SESSIONID=xxx
HAProxy->Browser : HTTP/1.1, SESSIONID=xxx

Note left of Browser: Second request
Browser->HAProxy:GET / HTTP/1.1, SESSIONID=xxx
HAProxy->S2: GET / HTTP/1.1, \nSESSIONID=xxx
Note right of S2: I don't know this SESSIONID\nTake this new one: yyy
S2->HAProxy :HTTP/1.1, \nSESSIONID=yyy
HAProxy->Browser : HTTP/1.1, SESSIONID=yyy

```
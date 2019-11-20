```sequence
participant Browser
participant HAProxy
participant S2
participant S1


Note over S1: shutdown
Note over HAProxy: Only S2 is available, so\nI send everything to S2
Note left of Browser: First request
Browser->HAProxy :GET / HTTP/1.1
HAProxy->S2:GET / HTTP/1.1
S2->HAProxy :HTTP/1.1, SESSIONID=xxx
HAProxy->Browser : HTTP/1.1, SESSIONID=xxx

Note left of Browser: Second request
Browser->HAProxy:GET / HTTP/1.1, SESSIONID=xxx
HAProxy->S2: GET / HTTP/1.1, \nSESSIONID=xxx
Note over  S2: I know this SESSIONID
S2->HAProxy :HTTP/1.1, \nSESSIONID=xxx
HAProxy->Browser : HTTP/1.1, SESSIONID=xxx

```
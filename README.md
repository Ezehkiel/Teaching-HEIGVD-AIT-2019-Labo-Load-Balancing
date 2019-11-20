Lab 03 - Load balancing

### Task 1: Install the tools


**Deliverables:**

> 1. Explain how the load balancer behaves when you open and refresh the URL <http://192.168.42.42> in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.

The load balancer seems to use the **roundrobin** policy. In order to verify this statement, we refresh the URL <http://192.168.42.42> in our browser several times.

Here is the result for the first access.

![2](./img/2.png)

Then, the second access.

![3](./img/3.png)

As we repeat the process, we can observe the same pattern over and over again. The load balancer alternated between the two servers. 

It is interesting to look at both the `id` fields. They are not the same, suggesting that each server has opened a different session with the client. But there is something worst : the `id` doesn't remain the same, even on the same server.

![4](./img/4.png)

As we can see, the `id` has changed between refreshes on server 1. In conclusion, session management is close to zero.

We can confirm that the reverse proxy uses roundrobin policy by looking at `/ha/config/haproxy.cfg`.

![8](./img/8.png)

> 2. Explain what should be the correct behavior of the load balancer for session management.

The load balancer should use **sticky sessions**. This means that once a server respond to a request with a session id, the load balancer should route the future requests for this particular session to the same server that serviced the first request.

> 3. Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. Here is an example:
>
> ![Sequence diagram for part 1](assets/img/seq-diag-1.png)

Here is the sequence diagram :

![11](./img/11.png)

> 4. Provide a screenshot of the summary report from JMeter.

As we can see below, on 1000 samples, there is a 50/50 distribution since S1 and S2 have both been reached 500 times.

![1](./img/1.png)

> 5. Run the following command:
>
> ```bash
> $ docker stop s1
> ```
>
> Clear the results in JMeter and re-run the test plan. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.

Here is the result of the test plan run.

![5](./img/5.png)

Now, every GET request reaches S2. This time, the session `id` is correctly reused and the `sessionViews` counter increments correctly.

![6](./img/6.png)

And, after a refresh.

![7](./img/7.png)

Here is the sequence diagram.

![10](./img/10.png)


### Task 2: Sticky sessions

**Deliverables:**

> 1. There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).

* Using SERVERID :

This approach consists of configuring the load-balancer to inject a cookie in the client browser. First, we have to add the following line in the configuration file `ha/config/haproxy.cfg` :

```bash
cookie SERVERID insert
```

It tells HAProxy to setup a cookie called SERVERID only if the user did not come with such cookie.

Then, we modify the following lines :

```bash
server s1 ${WEBAPP_1_IP}:3000 cookie cs1 check
server s2 ${WEBAPP_2_IP}:3000 cookie cs2 check
```

Here, we added `cookie csX` before `check`. It provides the value of the cookie inserted by HAProxy. When the client comes back, then HAProxy knows directly which server to choose for this client.

![13](./img/13.png)

* Using NODESESSID :

This approach consist of configuring the load-balancer to use the cookie setup by the application server to maintain affinity between the server and a client. We add the following lines in `ha/config/haproxy.cfg` :

```bash
cookie NODESESSID prefix
server s1 ${WEBAPP_1_IP}:3000 cookie cs1 check
server s2 ${WEBAPP_2_IP}:3000 cookie cs2 check
```

We use the `NODESESSID` since it is a NodeJS application. 

Here, when the server first responds to a client, a `Set-Cookie:NODESESSID=...` is set in the header. When passing through HAProxy, the cookie is modified like this : `Set-Cookie:NODESESSID=cs1~...`.

Then, for its second request, the client will send a request containing : `Cookie:NODESESSID=cs1~...`. Finally, HAProxy will clean it up on the fly to set it up back like the origin : `Cookie:NODESESSID=...` and forward the request to the right server.

![14](./img/14.png)

> Choose one of the both stickiness approach for the next tasks.

> 2. Provide the modified `haproxy.cfg` file with a short explanation of the modifications you did to enable sticky session management.

We chose to use `NODESESSID`. The modifications we did in the configuration file and the explanations can be found in the previous question.

> 3. Explain what is the behavior when you open and refresh the URL
>    <http://192.168.42.42> in your browser. Add screenshots to
>    complement your explanations. We expect that you take a deeper a
>    look at session management.

...

> 4. Provide a sequence diagram to explain what is happening when one
>    requests the URL for the first time and then refreshes the page. We
>    want to see what is happening with the cookie. We want to see the
>    sequence of messages exchanged (1) between the browser and HAProxy
>    and (2) between HAProxy and the nodes S1 and S2. We also want to see
>    what is happening when a second browser is used.

...

> 5. Provide a screenshot of JMeter's summary report. Is there a
>    difference with this run and the run of Task 1?

  * Clear the results in JMeter.

  * Now, update the JMeter script. Go in the HTTP Cookie Manager and
    <del>uncheck</del><ins>verify that</ins> the box `Clear cookies each iteration?`
    <ins>is unchecked</ins>.

  * Go in `Thread Group` and update the `Number of threads`. Set the value to 2.

7. Provide a screenshot of JMeter's summary report. Give a short
    explanation of what the load balancer is doing.


### Task 3: Drain mode

HAProxy provides a mode where we can set a node to DRAIN state. In
this case, HAProxy will let _current_ sessions continue to make
requests to the node in DRAIN mode and will redirect all other traffic
to the other nodes.

In our case, it means that if we put `s1` in DRAIN mode, all new
traffic will reach the `s2` node and all current traffic directed to
`s1` will continue to communicate with `s1`.

Another mode is MAINT mode which is more intrusive than DRAIN. In this
mode, all current and new traffic is redirected to the other active
nodes even if there are active sessions.

In this task, we will experiment with these two modes. We will base
our next steps on the work done on Task 2. We expect you have a
working Sticky Session configuration with two web app nodes up and
running called `s1` and `s2`.

When all the infra is up and running, perform the following steps:

1. Open a browser on your host

2. Navigate to `http://192.168.42.42`. You will reach one of the two
   nodes. Let's assume it is `s1` but in your case, it could be `s2`
   as the balancing strategy is roundrobin.

3. Refresh the page. You should get the same result except that the
   views counter is incremented.

4. Refresh multiple times the page and verify that you continue to
   reach the same node and see the sessionViews counter increased.

5. In a different tab, open `http://192.168.42.42:1936` and take a
   look. You should have something similar to the following
   screenshot.

![Admin Stat view of HAProxy](assets/img/stats.png)

  You should be able to see the `s1` and `s2` nodes and their state.

For the next operations, you will use HAProxy's built-in command line
to query its status and send commands to it. HAProxy provides the
command line via a TCP socket so that a system administrator is able
to connect to it when HAProxy runs on a remote server. You will use
`socat` to connect to the TCP socket. `socat` is a universal
command-line tool to connect pretty much anything with anything. You may need to install it

To use it type the following:

```bash
$ socat - tcp:192.168.42.42:9999
prompt

> help
Unknown command. Please enter one of the following commands only :
  clear counters : clear max statistics counters (add 'all' for all counters)
  clear table    : remove an entry from a table
  help           : this message
  prompt         : toggle interactive mode with prompt
  quit           : disconnect
  show info      : report information about the running process
[...]
```

After typing `socat - tcp:localhost:9999` and pressing enter you will
see... nothing. You are connected to HAProxy, but it remains silent
for the time being. You have to turn on the prompt by typing
`prompt`. You will see a new line starting with `>`. Now you can enter
commands that will be directly interpreted by HAProxy.

First, increase the client timeout to avoid losing connections.

```bash
> set timeout cli 1d
```

Now, to set a node's state to `ready`, `maint` or `drain`, enter the
following command:

```bash
> set server nodes/<containerName> state <state>
```

**Note:** In fact, the `nodes` is the group of backend nodes
  labelled. You will find the corresponding `backend nodes` in ha
  config.

**Note 2:** The containerName is the label of the node. In fact, in
this lab, we used the same name as Docker container names but both
names are not linked together. We can choose different names if we
want. The name set in this command is the name present in the HAProxy
admin stats interface (or also found in the config file).

**Note 3:** We will use only the three states presented there. Take
  care that the command only accept lower cases states.

**Deliverables:**

> 1. Take a screenshot of the Step 5 and tell us which node is answering.

![30](./img/30.png)

We can see on the zoom that it's the node s1 that is answering. Indeed, under the "Sessions" label we can observe that the column "LbTot" is set to one, this column is a counter of how many time this node was selected. We also have the column "Total" that summerize how many request come to this node.

![31](./img/31.png)

> 2. Based on your previous answer, set the node in DRAIN mode. Take a
>    screenshot of the HAProxy state page.

![33](./img/33.png)

![34](./img/34.png)

> 3. Refresh your browser and explain what is happening. Tell us if you
>    stay on the same node or not. If yes, why? If no, why?

It's still the same server that is responding. It's because the state "drain" mean that the server will not accept any new connections but connection with a session already establish are still available to go on this node.

![35](./img/35.png)

> 4. Open another browser and open `http://192.168.42.42`. What is
>    happening?

It's the other node that is answering us.

![36](./img/36.png)

> 5. Clear the cookies on the new browser and repeat these two steps
>    multiple times. What is happening? Are you reaching the node in
>    DRAIN mode?

No, it's always the node s2 (the one note in drain state) that is answering us. It's because we are creating new connections, so we are always redirect to the node s2

> 6. Reset the node in READY mode. Repeat the three previous steps and
>    explain what is happening. Provide a screenshot of HAProxy's stats
>    page.

The connection that was still on s1 because the session was already set has been redirected to s2 with new session. This could be a problem and cause inconsistent behaviors for the user.

When we create a new connection with an other browser it's the node s1 that is responding us, the roundrobin sticky session policy is functioning again.

![37](./img/37.png)

![38](./img/38.png)

> 7. Finally, set the node in MAINT mode. Redo the three same steps and
>    explain what is happening. Provide a screenshot of HAProxy's stats
>    page.

For every connections, new or already setup, it's the node s2 that is answering. So all session that were on s1 are lost.

![39](./img/39.png)

### Task 4: Round robin in degraded mode.

In this part, we will try to simulate a degraded mode based on the round-robin previously configured.

To help experimenting the balancing when an application started to behave strangely, the web application
has a REST resource to configure a delay in the response. You can set
an arbitrary delay in milliseconds. Once the delay is configured, the
response will take the amount of time configured.

To set the timeout, you have to do a `POST` request with the following
content (be sure the `Content-Type` header is set to
`application/json`. The configuration is applicable on each
node. Therefore, you can do one `POST` request on
`http://192.168.42.42/delay` and taking a look at the response cookies
will tell you which node has been configured.

```json
{
  "delay": 1000
}
```

The previous example will set a delay of 1 second.

Or retrieve the IP of the container you want to
configure and then do the `curl` command to configure the delay.

```bash
$ docker inspect <containerName>

$ curl -H "Content-Type: application/json" -X POST -d '{"delay": 1000}' http://<containerIp>:3000/delay
```

To reset the delay configuration, just do a `POST` with 0 as the delay
value.

Prepare your JMeter script with cookies erased (this will simulate new
clients for each requests) and 10 threads this will simulate 10
concurrent users.

*Remark*: In general, take a screenshot of the summary report in
 JMeter to explain what is happening.

**Deliverables:**

*Remark*: Make sure you have the cookies are kept between two requests.

1. Be sure the delay is of 0 milliseconds is set on `s1`. Do a run to have base data to compare with the next experiments.

2. Set a delay of 250 milliseconds on `s1`. Relaunch a run with the
    JMeter script and explain what it is happening?

3. Set a delay of 2500 milliseconds on `s1`. Same than previous step.

4. In the two previous steps, are there any error? Why?

5. Update the HAProxy configuration to add a weight to your nodes. For
    that, add `weight [1-256]` where the value of weight is between the
    two values (inclusive). Set `s1` to 2 and `s2` to 1. Redo a run with 250ms delay.

6. Now, what happened when the cookies are cleared between each requests and the delay is set to 250ms ? We expect just one or two sentence to summarize your observations of the behavior with/without cookies.

### Task 5: Balancing strategies

In this part of the lab, you will be less guided and you will have more opportunity to play and discover HAProxy. The main goal of this part is to play with various strategies and compare them together.

We propose that you take the time to discover the different strategies in [HAProxy documentation](http://cbonte.github.io/haproxy-dconv/configuration-1.6.html#balance) and then pick two of them (can be round-robin but will be better to chose two others). Once you have chosen your strategies, you have to play with them (change configuration, use Jmeter script, do some experiments).

**Deliverables:**

1. Briefly explain the strategies you have chosen and why you have chosen them.

2. Provide evidences that you have played with the two strategies (configuration done, screenshots, ...)

3. Compare the both strategies and conclude which is the best for this lab (not necessary the best at all).

#### References

* [HAProxy Socket commands (drain, ready, ...)](https://cbonte.github.io/haproxy-dconv/configuration-1.5.html#9.2-set%20server)
* [Socat util to run socket commands](http://www.dest-unreach.org/socat/)
* [Socat command examples](https://stuff.mit.edu/afs/sipb/machine/penguin-lust/src/socat-1.7.1.2/EXAMPLES)

#### Lab due date

Deliver your results at the latest 15 minutes before class Wednesday, November 25.

#### Windows troubleshooting

git It appears that Windows users can encounter a `CRLF` vs. `LF` problem when the repos is cloned without taking care of the ending lines. Therefore, if the ending lines are `CRFL`, it will produce an error message with Docker:

```bash
... no such file or directory
```

(Take a look to this Docker issue: https://github.com/docker/docker/issues/9066, the last post show the error message).

The error message is not really relevant and difficult to troubleshoot. It seems the problem is caused by the line endings not correctly interpreted by Linux when they are `CRLF` in place of `LF`. The problem is caused by cloning the repos on Windows with a system that will not keep the `LF` in the files.

Fortunatelly, there is a procedure to fix the `CRLF` to `LF` and then be sure Docker will recognize the `*.sh` files.

First, you need to add the file `.gitattributes` file with the following content:

```bash
* text eol=lf
```

This will ask the repos to force the ending lines to `LF` for every text files.

Then, you need to reset your repository. Be sure you do not have **modified** files.

```bash
# Erease all the files in your local repository
git rm --cached -r .

# Restore the files from your local repository and apply the correct ending lines (LF)
git reset --hard
```

Then, you are ready to go.

There is a link to deeper explanation and procedure about the ending lines written by GitHub: https://help.github.com/articles/dealing-with-line-endings/

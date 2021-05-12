---
layout: essay
type: essay
title: 'Load testing Python API: Background threads behaviour'
# All dates must be YYYY-MM-DD format!
date: 2021-05-13
labels:
  - Load Testing
  - Performance
  - Node.js
  - Python
---

<figure class="ui image centered">
	<img src="../images/stuck.png">
    <figcaption>Photo by <a href="https://unsplash.com/@octoberroses?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Aubrey Odom</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a></figcaption>
</figure>

I was adding email-confirmation feature to aside project I'm working on. The REST-API is built by Flask, and you know Python is really not good at handling IO operations (not talking about the new ASYNC frameworks) so I thought let's test how many user the registration API can handle in on minute if we sent the email synchronously in the main thread and if I sent them asynchronously in background thread.

## Jmeter configuration

Testing by Jmeter:
I generally used `appache-benchmark` to load test endpoints but in this case I have to generate dynamic payloads with different email and username every request and Apache is not really suitable for that, I googled around and I found this tool called `Jmeter`.

<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/jmeter.png"/>

`number of threads` and `ramp-up period` determine how many concurrent requests Jmeter is going to fore. For example 1000 users in 10 seconds means Jmeter will initiate 100 requests every seconds.
I fixed the ramp-up period on 60 seconds and ran the test with 750, 800 and 850 requests. In each test I generated a response-time graph because I think it's more interesting than just comparing the average response-time.

## Results

### 750 request distributed over one minute:

> Gunicorn server is started with 16 gevent workers

<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/750R-20W-MERGE.png"/>

**average respone time (BLOCKING) = 443ms**

**Average response time (ASYNC) = 307ms**

They are almost close. The average response time when the server is sending the emails synchronously is 443ms. And 307ms for the non blocking implementation.
I thought 16 workers is the best I can get on my 4 CPU's but I tested with 20 workers and the response time was 278 ms (I tested the non-blocking), so all next tests I'm going to use 20 workers for better performance.

### 800 request

> Gunicorn server with 20 gevent workers

   <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/800R-20W-MERGE.png"/>

**Average respone time (BLOCKING) = 790ms**

**Average response time (ASYNC) = 662ms**

**Average response time (without sending emails) = 294ms**

To be honest I was very disappointed by this result, I thought the async version will _almost_ eliminate the latency introduced by the email-sending feature.
Even though I scaled the web server to use 20 workers, then 30 workers but it got worse so 20 is the best we can get.

OK, let's try 850 request to see if it's going to crash

### 850 request

  <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/850R-20W-MERGE.png"/>

**Average response time (ASYNC) = 2026ms**

**Average response time (NO-MAILs) = 537ms**

Almost only 3 seconds for the blocking version and the response time took a more vertical shape. The server started to queue up requests so I didn't wait it to finish all the 850. Even the async version is not performing well the response time is going to surpass the 5s if the test had ran longer. Apparently sending emails asynchronously is not helping to increase server throughput at all.

I think I'm getting close results because the email sending is really a blocking operation when your are using a local STMP server like the one I used for this test, I guess handling emails in background threads will have a great impact if you are using a remote SMTP or API for instance. To mimic this network delay, I decided to add 500ms delay in the email-sending function , which should make the deference between the threaded and blocking version more obvious.
OK let's explore how the response time will react to the introduced delay.

## Network Delay effect

### At 750 request

I didn't finish the test. After 400 request sent I noticed the number of queued requests had gotten over 100 ! and the response time is now 1863ms comparing to 443ms when there was no delay ! .. Why? because there are no free workers to accept more requests, All the 20 workers are forced to wait 500ms for the mail server to respond. So threads really worth the effort !
<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/DELAY-750R.png"/>

## One last thing

What about the 1000 requests? well unfortunately with threads and with no threads the server couldn't handle 1000 requests in a reasonable time, 1000 requests over 60 seconds means the server should has ~17 rps throughput at least but Flask couldn't make more than 13 rps, I think the bottleneck now between Flask and mongodb, I'm using the ODM `mongoengine` but probably it's just because Flask is not an ASYNC framework. Anyways I made one final test with Node.js/Express and mongoose, I have the same endpoint (it register a user and sends an email) and I load tested it with 1000 requests over 60 seconds .. . and it _smoothly_ handled it like it's nothing .. and scored only **73 ms** average response time .. amazingly fast with out any extra effort or tricks my part.

  <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/NODE-PYTHON.png"/>

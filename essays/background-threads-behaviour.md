---
layout: essay
type: essay
title: 'Investigating Python API performance: handling IO operation in background threads'
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

I was adding email-confirmation feature to aside project I'm working on. The REST-API is built by Flask, and you know Python is really not good at handling IO operations -not talking about the new ASYNC based frameworks- so I thought it would be really interesting to investigate the performance behavior when we send the confirmation email synchronously in the main thread VS sending it asynchronously in a background thread. My intuition is saying that when I delegate sending emails to a background thread the latency should be very close to the latency before introducing this feature. But hey! this all is just a theoretical speaking .. let's see what the machine has got to say!.

## Jmeter configuration

Testing by Jmeter:
I generally used `appache-benchmark` to load test endpoints but in this case I have to generate dynamic payloads with different email and username every request and `ab` is not really suitable for that, I googled around and I found this tool called `Jmeter`.

<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/jmeter.png"/>

`number of threads` and `ramp-up period` determines how many concurrent requests Jmeter is going to fire. For example 1000 users in 10 seconds means Jmeter will initiate 100 requests every seconds.
I fixed the ramp-up period on 60 seconds and ran the test with 750, 800 and 850 requests. In each test I generated a response-time graph because I think it's more interesting than just comparing the average response-time.

## Results

### Handling 750 request:

> Gunicorn server is started with 16 gevent workers

<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/750R-20W-MERGE.png"/>

Average respone time (BLOCKING) = **443ms**

Average response time (ASYNC) = **307ms**

They are almost close. The average response time when the server is sending the emails synchronously is 443ms. And 307ms for the non blocking implementation.
I thought 16 workers is the best I can get on my 4 CPU's but I tested with 20 workers and the response time was 278 ms (I tested the non-blocking), so all upcoming tests ran on a server with 20 workers for better performance.

### 800 request

> Gunicorn server with 20 gevent workers

   <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/800R-20W-MERGE.png"/>

Average respone time (BLOCKING) = **790ms**

Average response time (ASYNC) = **662ms**

Average response time (without sending emails at all) = **294ms**

To be honest I was very disappointed by this result, I thought the async version will _almost_ eliminates the latency introduced by the email-sending feature.
Even though I scaled the web server to use 20 workers, then 30 workers but it got worse so 20 is the best we can get.

OK, let's try 850 request to see if it's going to crash

### 850 request

  <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/850R-20W-MERGE.png"/>

Average response time (ASYNC) = **2026ms**

Average response time (NO-MAILs) = **537ms**

Almost only 3 seconds for the blocking version and the response time took a more vertical shape. The server started to queue up requests so I didn't wait it to finish all the 850. Even the async version is not performing well the response time is going to surpass the 5s if the test had ran longer. Apparently sending emails asynchronously is not helping to increase the throughput at all but at least it reduces the response time.

I think the sync and async versions have close results because the email sending is _not_ really a blocking operation when your are using a local STMP server like the one I used for this test, I guess handling emails in background threads will have a noticable impact if you are using a remote SMTP or API for instance. To mimic this network delay, I decided to add a 500ms delay in the email-sending function, which should make the deference between the threaded and the blocking version more obvious.
OK let's explore how the response time will react to the introduced delay.

## Network Delay effect

### At 750 request

I didn't finish the test. After 400 request sent I noticed the number of queued requests had gotten over 100 ! and the response time is now 1863ms comparing to 443ms when there was no delay! .. Why? because there are no free workers to accept more requests, All the 20 workers are forced to wait 500ms for the mail server to respond .. this is the blocking effect we didn't notice when the mail server was local. Contrary -and as expected- the threaded version's response-time is not effected by the delay .. phew! 
<img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/DELAY-750R.png"/>

So threads really worth the effort right ?! Well. Yeah. But. You know. Not so many people are excited about threads and the complexity introduced by threads. So? .. So all I am saying is that when your application is IO bounded maybe you should think about other options.

## One last thing

What about the 1000 requests? well unfortunately with threads and with no threads the server couldn't handle 1000 requests in a reasonable time, 1000 requests over 60 seconds means the server should have at least ~17 rps throughput but Flask couldn't make more than 13 rps, I think the bottleneck now is between Flask and mongodb, I'm using `mongoengine` as an ODM but probably it's just because Flask is not an ASYNC framework. Anyways I made one last test with Node.js/Express and mongoose, I have the same endpoint (it register a user and sends an email) and I load tested it with 1000 requests over 60 seconds .. . and Node _smoothly_ handled it like it's nothing .. and scored only **73 ms** average response time .. amazingly fast with out any extra effort or tricks on my part.

  <img class="ui image" src="{{ site.baseurl }}/images/jmeter-results/NODE-PYTHON.png"/>

## Fighting the techonlogy

I know I could use one last trick to get around the database bottleneck I mentioned. I can use a Redis queue to buffer regeneration requests and then I can use worker(s) to do the actual writing into the database. This may give me a performance boost I know but.. . but it sounds like every time there is an IO problem I'll have to pull some tricks and introduce even more complexity. While as we've seen above Node _naturally_ handled this .. so why to bother?. We are not supposed to fight the technology to do something it is not designed for.

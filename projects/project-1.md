---
layout: project
type: project
image: images/zelite.png
title: Zelite
permalink: projects/zelite
# All dates must be YYYY-MM-DD format!
date: 2020-07-01
labels:
  - Web
  - IoT
  - Python
  - MQTT
  - Mongodb
  - JWT
  - Vue.js
summary: I developed this platform for IoT developer to acclerate building home autoamtion solutions.
---

<img class="ui image" src="{{ site.baseurl }}/images/zelite-arch.jpg">

Zelite is an IoT platform to monitor and manage home devices "Lights, Fans, TV ..etc". Basically you can control any device by connecting it to a relay that has an internet connection. It's composed of a **REST** **API** for AuthN/AuthZ and devices management. The actual messaging between end devices "IoT" and the front end is done through **MQTT** **broker**.

> [Live Demo](https://3omer.github.io/zelite-client/)

<video class="" width="600px" height="240" controls>
  <source type="video/webm" src="{{ site.baseurl }}/images/zelite.webm">
</video>

The **REST API** is built with flask. In the first version the whole project lived togther, I used templating language to build the front end and **cookies**'s based authentication. Eventually I separated the front end and used **JWT** for authentication.

While I could have used HTTP to connect IoT devices too -which I did at early stages- but it has many draw backs: First it's bad for scalling, imagine hundered of IoT devices hitting the API every 3-5 seconds to get real time updates. Plus HTTP and JSON parsing is heavy and inconvienet at these IoT devices anyway. Thats why I opted for the light-weight MQTT portocol. Since MQTT can persists last mesaages we don't needto bother the database with IoT traffic 'huge plus for scalability', and MQTT has client liberaries for most languages throw WebSockets.

## Documentaion

I used **Postman** to write the documentation for this services as well as testing the endpoints.

The Client App is an **SPA** build with **Vue.js**. I learned **Vue.js** specially for this project .. It was SMOOTH. Vue is well documented I didn't have to look for anyother resource to learn it just the docs.

### Screens:

<div class="ui images">
  <img class="ui centered big image" src="../images/zelite-1.png">
  <img class="ui medium image" src="../images/zelite-2.png">
  <img class="ui medium image" src="../images/zelite-3.png">
</div>

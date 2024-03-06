+++
title = "Hash code saga"
description = "Hash code of enums in Java"
date = "2020-08-12"
aliases = ["design", "code"]
author = "Vishwamitra Hegde"
draft = true
+++

In my team, to avoid mid-air collisions on the PUT operations on REST endpoints, we use [ETag and If-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag#avoiding_mid-air_collisions) headers. ETag value is nothing but a hash code generated on the entity fields. During the GET the value is sent in ETag and the client will send the value back in If-Match header on a PUT. The server upon receiving the request, recacluates the hash code on the unmodified entity and matches the it with the value from If-Match header. if no match then the request is rejected with [412 Precondition Failed](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/412).


We built a new service with number of CRUD endpoints in Java. The code was tested and deployed to production, in a K8S cluster with 2 nodes. The initial validations indicated that the service is working as expected. After couple of days when the users started to use it in anger, some have started noticiing frequest 412 errors on a particular endpoints. All the usual remidies like clear cookie, clearning borwser cache didn't really made any difference. It was also made sure they are not using multople browsers whihc was very unlikey but we wanted to rule out all the possiblities.

So I took the issue in my hand and stared to debug. The issue was reporducable, after a while i nocied every time i do a GET it is generating a new ETag (hashcode), though the resource was not modified.
I further delved into the code, but notingh obvioud was wrong. But one partucular thing i noced in the logs is that the request was going to two nodes in round robin fashing. one thing wa clear that they ar generating two different hashcodes. But why?

One of the fields in the entity was an ENUM. So 


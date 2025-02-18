+++
title = "The Enum HashCode Conundrum: Avoiding ETag Issues in RESTful APIs"
description = "Hash code of enums in Java"
date = "2020-08-12"
aliases = ["design", "code"]
author = "Vishwamitra Hegde"
+++

In my team, to avoid mid-air collisions on the PUT operations on REST endpoints, we use [ETag and If-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag#avoiding_mid-air_collisions) headers. ETag value is nothing but a hash code generated based off entity fields. This is how it all works

- During a GET request, the server sends the calculated hash as the ETag header.
- The client then sends this ETag value back in the If-Match header during subsequent PUT requests.
- Upon receiving a PUT request, the server recalculates the hash code based on the current state of the entity and compares it to the value provided in the If-Match header.
- If the calculated hash and the If-Match value don't match, the server rejects the request and returns a [412 Precondition Failed](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/412) error, indicating that the resource has been modified since the client last retrieved it

We developed a new Java service featuring several CRUD endpoints. After testing, we deployed it to a Kubernetes cluster with two nodes. Initial validation indicated that the service was functioning correctly. However, after a few days of heavier use, some users began experiencing frequent 412 errors on a specific endpoint. Standard troubleshooting steps, such as clearing cookies and browser cache, proved ineffective.

After several hours of intense debugging, I managed to reproduce the issue. I observed a pattern: each GET request generated a new ETag (hash code), even when the underlying resource remained unchanged. I meticulously reviewed the code, but nothing appeared obviously wrong. However, while examining the logs, I noticed that requests were being routed to different cluster instances. It became clear that these instances were generating different hash codes for the same entity. The question was: why?

It turned out that one of the entity's fields was an Enum. Java's default hashCode() implementation for enums relies on the object's memory address within the JVM. Since each instance of the service running on different nodes had its own JVM with different memory addresses, the same enum value had different hash codes across instances. This discrepancy led to the inconsistent ETag generation and, consequently, the 412 errors.

The issue was difficult to identify initially because, in a local development environment, the service typically runs within a single JVM, resulting in consistent hash codes. However, the problem manifested when the service was deployed across multiple nodes in the cluster, each with its own JVM.

The fix was to generate the hash code based on ordinal values rather then the enum itself. To mitigate the risks of encountering this issue, we added a helper method in our utility lib to generate hash codes and mandated using it instead.
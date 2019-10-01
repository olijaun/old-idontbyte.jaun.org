---
layout: post
title:  "Idempotence"
---

Idempotence is one of most essential properties of a web service. No matter whether it is SOAP, REST or GraphQL. However in my experience this aspect is often overlooked or ignored. I am aware that there are many articles about this topic already but often these articles are not tackling the topic in all his aspects. So here is my attempt.

## Definition
An elevator button is usually idempotent. It lights up when you press it for the first time. If you press it again it won't change. If you press it a hundred times the elevator will still arrive once.

In mathematics idempotence is defined as follows:

> Idempotence is the property of certain operations in mathematics [...] whereby they can be applied multiple times without changing the result beyond the initial application. ([Wikipedia](https://en.wikipedia.org/w/index.php?title=Idempotence&oldid=917052170))

However we are more interested in the definition used in computer science:

> in imperative programming, a subroutine with side effects is idempotent if the system state remains the same after one or several calls, in other words if the function from the system state space to itself associated to the subroutine is idempotent in the mathematical sense given in the definition;  ([Wikipedia](https://en.wikipedia.org/w/index.php?title=Idempotence&oldid=917052170))

Here is an example in Java:

```java
public class MyString {
    
    private String string;
    
    public MyString(String s) {
        string = s;
    }
    
    /** idempotent method */
    public void toUpperCase() {
        string = s.toUpperCase();
    }
    
    /** NON idempotent method */
    public void append(String s) {
        string = string + s;
    }
    
    public String getValue() {
        return string;
    }
}
```

`toUpperCase()` is idempotent: No matter whether you call `toUpperCase()` once or a hundred times: the result will be the same. `append()` is not idempotent because each time it is being called the string gets longer.


## Part II: Distributed Systems

In the context of a single Java application (as shown above) idempotence is not so exiting... However if you have a distributed system that is communicating over a network then things are getting more interesting.

Firstly we have to understand why distributed systems are difficult. Years ago Peter Deutsch enumerated eight [Fallacies of Distributed Computing](https://en.wikipedia.org/w/index.php?title=Fallacies_of_distributed_computing&oldid=916589690). The first one was "The network is reliable". Let's assume your write a messenger app that communicates with other parties. No matter which fancy programming language you will use: in the end there will be some bits and bytes sent over the wire. So what can probably go wrong? A lot. Let's look at a sequence diagram of two services communication. Service A sends a Request to Service B.

{% plantuml %}
participant "Service A" as A
participant "Service B" as B

A -> B: Request
activate B
A <-- B: Response
deactivate A
{% endplantuml %}

If Service A and Service B communicate over a network then this picture is obviously a simplification. So let's draw the network as well:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B

A -> N: Request
activate N
N -> B: Request
activate B
N <-- B: Response
deactivate B
A <-- N: Response
deactivate A
{% endplantuml %}

Also the last image is implying that there is an synchronous communication going on which is not true. The bits and bytes are travelling from A to B and eventually B might create a response that could get lost on its way back. In a programming language you may get the illusion that a request is synchronous because the libraries you are using is waiting for a response. Look at the following example:

```
curl -v http://idontbyte.jaun.org
* Rebuilt URL to: http://idontbyte.jaun.org/
*   Trying 185.199.111.153...
* TCP_NODELAY set
* Connected to idontbyte.jaun.org (185.199.111.153) port 80 (#0)
> GET / HTTP/1.1
> Host: idontbyte.jaun.org
> User-Agent: curl/7.58.0
> Accept: */*

```

We get the impression that our curl fetches the page at http://idontbyte.jaun.org synchronously. curl terminates as soon as it receives the response. Under the hood what this call does is send some bytes to the server and then wait to see if it ever gets some bytes as a response. The correlation of request and response is done by the underlying protocol.

What happens if the page is responding very slow? I'm using [slowwly](http://slowwly.robertomurray.co.uk) here to simulate a slow page. The parameter `--max-time 1` tells curl to wait one second for response:

```
curl --max-time 1 -v http://slowwly.robertomurray.co.uk/delay/3000/url/http://www.google.ch
*   Trying 34.241.172.109...
* TCP_NODELAY set
* Connected to slowwly.robertomurray.co.uk (34.241.172.109) port 80 (#0)
> GET /delay/3000/url/http://www.google.ch HTTP/1.1
> Host: slowwly.robertomurray.co.uk
> User-Agent: curl/7.58.0
> Accept: */*
> 
* Operation timed out after 1000 milliseconds with 0 bytes received
* stopped the pause stream!
* Closing connection 0
curl: (28) Operation timed out after 1000 milliseconds with 0 bytes received
```

curl returns a timeout error in this case. Now look at what can go wrong when A communicates with B:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B
autonumber
A ->x N: Request
activate N
N ->x B: Request
activate B
N x<-- B: Response
deactivate B
A -> A: Timeout

A x<-- N: Response
deactivate A
{% endplantuml %}

1. A cannot connect to the network
2. A can connect to the network but B cannot be reached
3. B cannot connect to the network when it tries to send the response
4. B can send the response to the network but the response never reaches A
5. A is waiting for the response and after a while decides to not wait any longer (timeout)
6. ...

What can be done if such a problem occurs? One simple solution is to just retry. So if the network cannot be reached we just retry to send the data again. If B is not responding then we just send the request again, maybe we get a response. Assume the following service:

```
POST /money-sender

{
    accountNumber: "CH93 0076 2011 6238 5295 7"
    amount: 42,
    currency: CHF
}
```

If this POST request fails for some reason (e.g. a timeout) and we send it again then the money might be sent twice to the given account. That is because in case of a timeout or other failures you do not know if the recipient actually sent the money or not. Maybe just the response was lost:

{% plantuml %}
participant "Service A" as A
participant "Network" as N
participant "Service B" as B
A -> N: send 42 CHF
activate N
N -> B: send 42 CHF
activate B
N x<-- B: OK, thanks
deactivate B
A -> A: Timeout

deactivate A
{% endplantuml %}

So how can idempotency help? Read on...

## How to design idempotent services

Firstly it's important to note that read-only services are already idempotent.

We assume that this service responds with the newly created person containing an ID that has been generated for the new person:

```json
{
  id: 42,
  name: "Oliver"
}
```

Now assume we call this service with java:
// TODO: curl

```java
Person person = Person person = client.target("http://idontbyte.jaun.org")
    .request(MediaType.APPLICATION_JSON)
    .accept(MediaType.APPLICATION_JSON)
    .post(Entity.json(newPerson), Person.class);
```

If the service or the network is very slow or the service never returns a response because it crashed then you will eventually get a `SocketTimeoutException`. It is not the service that returns this error, its your network stack or the Java API you are using. Usually you simply cannot wait "forever". In theory the service might respond in one week. But this is probably too late for your use case...

We can basically do two things now:

Do Nothing: Because we haven't received the response does not mean that the request has not been processed by the service! Our person might have been created and everything is fine. But how do we know? We could issue a GET call and see whether the person has been created. But wait... for this we would need the ID of the created person which is part of the response we've never received. Damn.

Repeat the call: Maybe there was a temporary problem. Assuming that the person has not been created with the first call this could be the perfect solution. If the second call works then we are fine. But what if it already worked the first time and we just haven't received the response? Then this will create a second person with the same name but a different id. This is usually not what you want.

## Part III: The Solution

How can we solve this problem? By making the service idempotent. There are different possibilities. Firstly I would like to look at some services that are idempotent without having to do something special.

The most obvious kind of services are read only services:

```
GET /persons/42
{
  id: 42,
  name: "Oliver"
}
```

This service is not modifying anything. You can call the service as many times as you wish, it won't change the result. Of course, if someone changes the name of this person you will get another response but that is not what you want to solve by making the service idempotent. The point is that YOU are not changing the result by calling it multiple times. In order to know whether someone changed the data you would have to use some versioning and/or locking techniques. That's a completely other topic.

There are also write services that can be "naturally" idempotent. Assume a service where you can create categories to be used to categorise books in a book shop. You have categories like "fiction", "science" or "fantasy". Before they can be used they have to be registered via the /book-categories resource:

TODO: Bank account
```
POST /book-categories
{
  name: "fiction"
}
```

It makes no sense to have two categories that have "fiction" as their name. So this service is probably implemented in a way that ignores requests that try to add a category with a name that already exists. If you call this service and the first call fails you can simply repeat the call until it works. You don't have to care. *So this service is idempotent*.

Now, what we do about our `/persons` resource? There might exist persons with the same name. Of course in the real world such a system would include probably a first and lastname, an address etc. But there still might be two persons with the same name and living at the same address.

One possibility is to let the caller decide which ID to use for the person:

```
POST /persons
{
  id: 42,
  name: "Oliver"
}
```

The service might just ignore POST requests with an already used ID. So repeated calls with this request will not create multiple persons anymore. One problem with this approach is that the service cannot control the IDs used. So how does the caller know which IDs are still unused? Assuming someone already crated a person with ID 42. Now someone creates a person with same ID. The service will just return "OK" and the caller thinks that the person has been created successfully. 

This solution could be improved by not only checking whether the ID already exists but also by comparing the input whether it is the same as the input that has been stored already. If it is different there might be returned a special error indicating this fact. But then there would still be a problem when you actually want to create a new person although it has the same input but represents an other physical person.

So it might be better to return a special code in the response indicating that the person already exists which is fine according to the HTTP-Specification that does not mandate POST to be idempotent (however GET, PUT, and DELETE should be implemented in an idempotent manner). Just as a side note: I'm using HTTP/REST as an examples here. Of course idempotence is also relevant for SOAP, GraphQL or any other remoting protocol.

Another possibility would be to use PUT requests:

```
PUT /persons/42
{
  name: "Oliver"
}
```

Here we are specifying the id in the URL. However, due to the fact that PUT should be idempotent we cannot return something like "already exists" if the call is performed multiple times. So if a client accidentally picks an ID that has been used by another client then he might think that his call succeeded. It does not help to check whether the ID already has been used before. Because after you have checked and before you are creating the new person someone still might create a person  with exactly this id.

Too make this case very unlikely clients could use UUIDs (preferably type 4). These UUIDs should be unique and there shouldn't be any collisions. Of course if clients do not generate this UUID correctly there still might be collisions.

You can mitigate this problem by providing a service which delivers a person id that has to be used (e.g. a random UUID type 4):

```
POST /person-ids
{
  id: "145b63ec-1440-46b5-b29f-6ae3c948dce4"
}
```

The client then uses this id for the PUT request:

```
PUT /persons/145b63ec-1440-46b5-b29f-6ae3c948dce4
{
  name: "Oliver"
}
```

The service must make sure it is an ID that he actually issued before. This way the ID generation is controlled by the service, clients cannot do anything wrong.

Another solution would be to treat idempotency on the request level. Have a look at the following example:

```
POST /persons
X-idemotence-id: 145b63ec-1440-46b5-b29f-6ae3c948dce4

{
  name: "Oliver"
}
```

So here we are adding a `X-idemotence-id` HTTP header to the request. The service has to remember all `X-idemotence-id` that he successfully processed. If he repeatedly receives the same `X-idemotence-id` he just ignores the request. This solution has the advantage that the service can control the generation of the person ID and no additional service is required. Of course it is again up to the client to use a proper ID that will not clash with IDs used by others.

## Part 4: Implementation

When implementing and idempotent service special care has to be taken when persisting IDs that are used for idempotence. Make sure the ID is "unique" (e.g. using UNIQUE constraints in a database). Violations of this constraint must be handled according to the approach you have taken.

When using the approach with `X-idemotence-id` one might be tempted to implement this in a reusable manner e.g. as a service. Let's look a the following code:

```java

@Inject
private IdempotenceService idempotenceService;

public Person createPerson(UUID idempotenceId, String personId, String personName) {
    
    idempotenceService.markAsUsed(idempotenceId).isFailure());
    
    return createPerson(personId, personName);
    
}
```

`idempotenceService.markAsUsed` will for example throw an exception if the request already exists. This will only work if `idempotenceService.markAsUsed` runs in the same local transaction as `createPerson`. Because otherwise it might be possible that the request is marked as used but then then the actual creation of the person fails. The same "local transaction" means the same database (if you are using a database but the same applies for other data stores). So you cannot provide the IdempotencyService as a reusable REST service.

Some might argue that you could use distributed transactions etc. However this leads to other problems... well basically the same problems as mentioned in Part II. Many people don't know (or just ignore) that a distributed transaction might have a timeout as well! Beside this they can be problem for scaling etc. I won't go into details here.

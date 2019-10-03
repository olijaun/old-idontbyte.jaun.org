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
        string = string.toUpperCase();
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

What can be done if such a problem occurs? One simple solution is to just retry. So if the network cannot be reached we just retry to send the data again. If B is not responding then we just send the request again, maybe we get a response. Assume the following service that adds 42 Swiss Francs (CHF) to account "1".

```
POST /accounts/1/deposits

{
    amount: 42,
    currency: "CHF"
}
```

If this POST request fails for some reason (e.g. a timeout) and we send it again then the money might be deposited twice to the given account. That is because in case of a timeout or other failures you do not know if the recipient actually sent the money or not. Maybe just the response was lost:

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

Of course it might also be the case that the transaction was not successfully processed. If you don't retry the money will never be added. So how can idempotency help? Read on...

## How to design idempotent services

Firstly it's important to note that read-only services are already idempotent. Assume the following:

```
GET /accounts/1

{
    id: 1
    balance: 42,
    currency: "CHF"
}
```

This reads account "1". It does not change the account. Whether this is called once or twice does not matter. It will return the same account. Of course: if someone else modifies the account after the first call then the second call will return the modified account. But that is not what you want to solve by making the service idempotent. The point is that YOU are not changing the result by calling it multiple times. In order to know whether someone changed the data you would have to use some versioning and/or locking techniques. That's a completely different story.

No let's get back to our money-sender:

```
POST /accounts/1/deposits

{
    amount: 42,
    currency: "CHF"
}
```

How can this be made idempotent?

### Make the body itself idempotent

The example service could be made idempotent by changing the body as follows:

```
PUT /accounts/1

{
    balance: 42,
    currency: "CHF"
}
```

Instead of telling which amount to add the client can simply specify the new balance of the account. Now the service is idempotent. This call can made multiple times and the balance will always be 42. Obviously in this concrete case this could lead to problems if multiple clients are running in parallel. Clients would just override the value of the others. Especially in case of a banking application this would be very unfortunate.


### Add an id to the body

```
POST /accounts/1/deposits

{
    id: "123",
    amount: 42,
    currency: "CHF"
}
```

By adding "id' to the body of the POST call the service is able to distinguish whether the caller tries to repeatedly create the same transaction. If the transactions has already be processed then the service has to options. It could just respond with OK and the client will never know whether a previous POST already worked. This is fine because the client does not need to have this information. The client just has to know if it does not work.

The other possibility would be to respond with a special response that indicates that the response has already been processed. This would be OK as well but then the client has to handle this return code correctly.

One thing to consider is also what should happen if you send a different transaction but with the same id:

```
POST /accounts/1/deposits

{
    id: "123",
    amount: 37,
    currency: "CHF"
}
```

The transaction id is the same as in the other call but the amount is different: 37 CHF. The service could return a special "conflict" error code in this case. It could also be fine to ignore this case and just treat it as already processed. But probably this should be documented in the service description.

Another issue with this solution is that the client decides which is the transaction ID to be used. Usually the service would like to choose the ID.


### Use PUT instead of POST

By using PUT it is possible to specify the transaction's ID in the URL:

```
PUT /accounts/1/deposits/123

{
    amount: 37,
    currency: "CHF"
}
```

Similar to the POST example above the service is now able to decide whether he as already processed the transaction or not. Like with the POST example it is again the client who chooses the transaction ID which might be a problem.

### Provide an ID generator service

As mentioned before, usually a service would like to control the IDs used. Also, if the client chooses the transaction id then he could accidentally pick an ID that has been used by another client before. The service would then ignore this request and the client would never know. 

In order to avoid the latter clients could use UUIDs (preferably type 4). These UUIDs should be unique and there shouldn't be any collisions. Of course if clients do not generate this UUID correctly there still might be collisions.

If the service must control the ID to be used by the transaction then an ID generator service could be implemented:

```
POST /id-generator
{
  id: "145b63ec-1440-46b5-b29f-6ae3c948dce4"
}
```

This id generator just generates a new ID each time it is called. This ID is then used when creating a transaction:

```
POST /accounts/1/deposits

{
    id: "145b63ec-1440-46b5-b29f-6ae3c948dce4"
    amount: 37,
    currency: "CHF"
}
```

The service must check whether the transaction id specified has been issued before. If this is true then the transaction will be created.

The disadvantage of this solution is of course that the client has to perform two calls and an additional service (the id generator) has to be implemented.

### Provide an idempotence id as request metadata

Currently I think that probably the best solution is to use a dedicated idempotence id. In case of HTTP this could be implemented as a HTTP header:

```
POST /accounts/1/deposits
x-idempotence-id: 145b63ec-1440-46b5-b29f-6ae3c948dce4

{
    amount: 37,
    currency: "CHF"
}
```

The service has to remember all `x-idemotence-id`s that he successfully processed. If he repeatedly receives the same `x-idemotence-id` he just ignores the request. This solution has the advantage that the service can control the generation of the transaction ID and no additional service is required in order to generate an ID. Of course it is again up to the client to use a proper ID that will not clash with IDs used by others.

This also allows to use POST instead of PUT. I think it's more intuitive to create resources with POST and not with PUT. PUT would be used for updates of an existing resource (which probably makes no sense in our concrete example).

## Service implementation considerations

When implementing and idempotent service special care has to be taken when persisting IDs that are used for idempotence. Make sure the ID is "unique" (e.g. using UNIQUE constraints in a database). Violations of this constraint must be handled according to the approach you have taken.

When using the approach with a dedicated idempotence id (e.g. `x-idemotence-id`) then it must be assured that the business entity (the account transaction in the example) is saved in the same local transaction as the idempotence id:

```java
@Path("/accounts/{id}")
public class AccountResource {

    @Autowired
    private TransactionTemplate transactionTemplate;

    // ...

    @Autowired
    private JdbcTemplate jdbcTemplate;

     @POST
     @Produces({MediaType.TEXT_PLAIN})
        @Path("/deposits")
        @Transactional
        public Response addDeposit(Deposit deposit, @HeaderParam("x-idempotence-id") String idempotenceId) {
    
    
            String newDepositId = UUID.randomUUID().toString();
    
            try {
                transactionTemplate.execute(new TransactionCallbackWithoutResult() {
    
                    @Override
                    protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
    
                        saveIdempotenceId(idempotenceId, newDepositId);
    
                        saveDeposit(newDepositId, deposit);
                    }
                });
    
                return Response.ok(newDepositId).build();
    
            } catch (Exception e) {
    
                String existingDepositId = getDepositIdByIdempotenceId(idempotenceId);
    
                if (existingDepositId != null) {
    
                    Deposit existingDeposit = getDeposit(existingDepositId);
    
                    if (existingDeposit.equals(deposit)) {
                        return Response.ok(existingDepositId).build();
                    } else {
                        return Response.status(Response.Status.CONFLICT).build();
                    }
                }
    
                throw e;
            }
        }
        
     // ...
}
```

In the example application ([see here for full source](http://github.com)) there are two saves performed: First the idempotence id is saved and then the business entity itself (the Deposit) is saved (the example uses a relational database and SQL for this). Both saves are done inside the same transaction (here using Spring programmatic transactions). This is important. Otherwise it could be that the idempotence id is stored but the actual business entity is not. If the client retries the request it would get an "OK" response which is obviously not true.

If two clients are calling this method with the same idempotence id at almost the same time then the first will store the idempotence id. The second one will get a locking error (if the transaction of the first has not finished yet). In my examplethis is e.g.:

> Concurrent update in table "IDEMPOTENT_REQUEST": another transaction has updated or deleted the same row [90131-199]

An exception is also thrown in case the idempotence id already was saved before, because we have a UNIQUE constraint on this column.

Inside the catch block we check whether a deposit id already exists for this idempotence id. If it exists, we check whether the corresponding deposit matches the one the caller tries to save. If this is true the service returns the deposit id of the already stored entity. This only works if we know which was the state of entity at the time it was saved. 

## A note on HTTP

The examples in this articles are REST or at least REST-like webservices. However everything is also valid for SOAP services or any RPC protocol. One thing to note when HTTP is used is the fact that HTTP defines which verbs are supposed to be idempotent and which not. This is defined in [rfc7231](https://tools.ietf.org/html/rfc7231)



https://stackoverflow.com/questions/45016234/what-is-idempotency-in-http-methods#targetText=A%20request%20method%20is%20considered,safe%20request%20methods%20are%20idempotent.

9.1.2 Idempotent Methods
Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical requests is the same as for a single request. The methods GET, HEAD, PUT and DELETE share this property. Also, the methods OPTIONS and TRACE SHOULD NOT have side effects, and so are inherently idempotent.

However, it is possible that a sequence of several requests is non- idempotent, even if all of the methods executed in that sequence are idempotent. (A sequence is idempotent if a single execution of the entire sequence always yields a result that is not changed by a reexecution of all, or part, of that sequence.) For example, a sequence is non-idempotent if its result depends on a value that is later modified in the same sequence.

A sequence that never has side effects is idempotent, by definition (provided that no concurrent operations are being executed on the same set of resources).

9.2 OPTIONS




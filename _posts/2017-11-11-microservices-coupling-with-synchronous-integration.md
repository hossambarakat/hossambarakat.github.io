---
layout: post
title: Microservices Coupling With Synchronous Integration
tags: Microservices
---

A lot of companies have started the journey of splitting their current monolithic applications into Microservices to gain all the benefits from Microservices such as strong module boundaries, independent deployment, hybrid technologies... which is fine, but what I am concerned about the assumption of splitting monolothic application to microservice will automatically lead to a loosely coupled services.

Let’s start with a simple piece of code handles purchasing command that includes create an order, then start the shipping process:

```csharp
public class PurchaseCommand
{
    public List<OrderItem> Items { get; set; }
    public Address ShippingAddress { get; set; }
    //rest of command properties
}
public class PurchaseCommandHandler
{
    public void Handle(PurchaseCommand command)
    {
        using(var transactionScope = new TransactionScope())
        {
            SaveOrderInDB(command);
            SaveShippingInDB(command);
        }
    }
}
```

Now, we decide that this code should be split into two microservices “Orders” and “Shipping” and each one will be hosted in a separate process. but how the two microservices will communicate to each other?

## REST
Usually the answer is using REST for integration between services, so the "Shipping" service will expose an end point that could be called by "Orders" service:

```csharp
//Shipping Service
public class StartShippingCommand
{
    public Guid OrderId { get; set; }
    public List<OrderItem> Items { get; set; }
}
public class StartShippingCommandHandler
{
    public void Handle(StartShippingCommand command)
    {
        SaveShippingInDB(command);
    }
}
```

The orders service will be updated to make a call to the shipping endpoint:

```csharp
//Orders Service
public class PurchaseCommandHandler
{
    public void Handle(PurchaseCommand command)
    {
        SaveOrderInDB(command);
        var shippingServiceProxy = new ShippingServiceProxy();
        shippingServiceProxy.StartShipping(new StartShippingCommand(){});
    }

}
```

So we managed to split the code into two separate services that could be deployed independently...but are the services loosely coupled?

> *Autonomous* microservice is a service that can respond to client requests regardless of the availablity of other services...to an extent.


### What will happen if the shipping service is not available?
The orders service will save the order in the DB but will fail while calling the shipping and the whole purchase will fail, moreover our database will be left with data inconsistencies.

So, Although the "Orders" and "Shipping" services are now hosted in two separate processes they are still coupled because the "Orders" service will not be able to serve the client request unless the "Shipping" service is available.

So how could we remove the coupling?

## Event-Driven Integration
Event-driven integration means the communication between services done though events, service will publish an event when something notable happens and then other services could subscribe to the event and do its own logic which could result in publishing more events.

So in our scenario, Event-driven integrations means that "Orders" service will save the order in the database then public an event `OrderAccepted`, the "Shipping" service will subscribe to `OrderAccepted` event and will save the shipping information in the database:

```csharp
//Orders Service
public class OrderAccepted
{
    public Guid OrderId { get; set; }
    public List<OrderItem> Items { get; set; }
}
public class PurchaseCommandHandler
{
    public void Handle(PurchaseCommand command)
    {
        SaveOrderInDB(command);
        Bus.Publish(new OrderAccepted
        {
            OrderId = OrderId, 
            Items = new List<OrderItem>(command.Items)
        });
    }
}

//Shipping Service
public class StartShippingCommandHandler
{
    public void Handle(OrderAccepted @event)
    {
        SaveShippingInDB(@event);
    }
}
```

The "Shipping" service could publish another event called `ShippingStarted` so "Orders" service could update the the order status.

### Again, what will happen if the "Shipping" service unavailable?
The "Orders" service now can serve the requests while "Shipping" service is down because it can save the data in database then publish an event and respond immediately to the client. the published `OrdeAccepted` event will be processed by the "Shipping" service whenever it is available.

Ofcourse that is assuming that the events has been published successfully inside the bus/queue.

## Summary
Decomposing a monolithic application into a set of services hosted in different processes doesn't guarantee having a loosely coupled microservices. Synchronous communication between microservices will lead to a more coupled microservices however using asynchronous event-driven communication will help in leading to more autonomous loosely coupled microservices.
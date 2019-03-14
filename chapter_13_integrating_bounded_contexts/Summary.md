# Chapter 12 Integrating Bounded Contexts

Summary of Integrating Bounded Contexts chapter.

## Integration Basics

When one Bounded Context (BC) needs to integrate with another BC, options are:

* BC-A -----RPC-API----> BC-B
    * RPC-API: made using procedures that take parameters
    
* BC-A -----Messaging(Publish-Subscribe)----> BC-B

* BC-A -----RESTFul HTTP----> BC-B
    * RESTFul HTTP: exchange and modify resources using URIs

## Principles of Distributed Computing

* Network is <b>not reliable</b>. 
* There is always <b>short/long latency </b>.
* <b>Bandwidth</b> is not infinite.??
* The network is not always <b>secure</b>.
* Network <b>topology changes</b>.
* Knowledge and policies are spread across  <b>multiple administrators</b>.
* Network transport has <b>cost</b>.
* The network is <b>heterogeneous</b>.

## Exchange Information Across System Boundaries

- Using a set of common classes and interfaces [M]<br>
System-A [M] ---> exchangeable information structure --> [M] System-B

- Using a standards based approach where no shared classes/interfaces are needed<br>
System-A --- exchangeable information structure --> System-B

* Exchangeable Information Structure
    * should be of a format that is transferable across the wire
    * should be usable by the consuming system
    * without the use of interfaces and classes
    
Example: Notification -> Event
```json
{
  "notificationId": 11111,
  "typeName": "com.compnay.app.SomeEvent",
  "version": 123,
  "occurredOn": "some date/time format",
  "event": {
     "eventType": "com.compnay.app.SomeEvent",
     "eventVersion": "123",
     "occurredOn": "some date/time format",
     "businessEventId": "456",
     "businessId1": "456",
     "businessId2": "789",
     "businessId3": "135",
     "eventField1": "abc",
     "eventField2": "efg",
     "eventField3": "xyz"
  }
}
```

* we can see that root level fields are standard end hence can provide 
type-safe methods for these fields: notificationId,typeName,version,occurredOn

* special event details can be read like so:

```java
    //java code
    
    //the producer creates and packages the domain-event
    DomainEvent domainEvent = new DomainEvent(100, "testing");
    Notification notification = new Notifiction(1, domainEvent);
    NotificationSerializer serializer = NotificationSerializer.instance();
    
    //this is what is sent across the wire
    String serializedNotificatin = serializer.serialize(notification);
    
    //once recieved on the consumer side
    NotificationReader reader = NotificationReader(serializedNotificatin);
    
    //standard fields
    reader.notificationId();
    reader.typeName();
    reader.version();
    reader.occurredOn();
    
    //speacial event fields
    reader.eventStringValue("eventVersion");
    reader.eventStringValue("/eventVersion");//using slash
    reader.eventIntegerValue("eventVersion");//using type-safe method
    
    reader.eventStringValue("nestedEvent", "eventVersion");
    reader.eventStringValue("nestedEvent/eventVersion");
    reader.eventIntegerValue("nestedEvent", "eventVersion");
    reader.eventIntegerValue("nestedEvent/eventVersion");
        
    reader.eventStringValue("<fieldName>");
    reader.eventStringValue("<nestedEvent>","<fieldName>");
    reader.eventStringValue("<nestedEvent>/<fieldName>");
    
    //events can also hold Value Objects
    reader.eventStringValue("backlogItemId.id");
    reader.eventStringValue("sprintId.id");
    reader.eventStringValue("tenantId.id");
   
```

* Here it can be seen that the notification and event have version numbers.
* Consumers can look for specific versions of notification/event to consume or 
* With careful consideration it is possible that consumers read event fields in the same way even when event structure changes.
* Using this approach consumers can still process older versions of these notification.
* Protocol Buffers: can be a better option when events change more frequently.

## Integration Using RESTful Resources
* TBD 

## Integration Using Messaging
* TBD







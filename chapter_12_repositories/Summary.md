# Chapter 12 Repository

A repository is a storage location. When you put things in you expect that it would be in the same state when you read it afterwards. At some point you want to remove it.

For each type of object that needs global access, create an object that can provide the illusion of in-memory collection. Provide methods to add and remove objects. Provide methods that select objects based on some criteria and return fully instantiated objects or collections of objects whose attribute values meet the criteria. Every persistent Aggregate type will have a Repository.

## Chapter Learning content
* Learn two different kinds of Repositories
* Implement Repositories for Hibernate, TopLink, Coherence, and MongoDB
* Understand why you might need additional behavior on a Repository’s interface.
* Become familiar with the challenges of designing Repositories for type hierarchies.
* Differences between Repositories and Data Access Objects

Only Aggregates have Repositories. If you are not using Aggregates in a given Bounded Context the respository pattern may be less useful.

## Collection-Oriented Repositories
You design a Repository interface that does not hint in any way that there is an underlying persistence mechanism, avoiding any notion of saving or persisting data to a store.

Explanation: 

How it works in common languages(like c#, java, etc.)
Objects are added to a collection, and they remain in the collection until they are removed. There is no need to do anything special to get the collection to recognize changes to the objects that it contains.

Standard collection interface:
```java
public interface Collection ... {
    public boolean add(Object o);
    public boolean addAll(Collection c);
    public boolean remove(Object o);
    public boolean removeAll(Collection c);
}
```

Simple Usage: 

```java
public interface CollectionUsage {
    assertTrue(calendarCollection.add(calendar));

    assertEquals(1, calendarCollection.size());

    assertTrue(calendarCollection.remove(calendar));

    assertEquals(0, calendarCollection.size());
}
```
Simple enough. One special kind of it is Set:
```java
public interface SetUsage {

    public void testSet() {
        Set<Calendar> calendarSet = new HashSet<Calendar>();
        assertTrue(calendarSet.add(calendar));
        assertEquals(1, calendarSet.size());
        assertFalse(calendarSet.add(calendar));
        assertEquals(1, calendarSet.size());
    }
}
```
If you try to add the same object twice it still has one object. The same goes for a Repository designed using a collection orientation. If you try to add an Aggregate a second time it has the same state.

### takeaway
* A Repository should work like a Set in the Collection-Oriented way
* You don’t need to “re-save” modified objects already held by the Repository

Example: 
```java
public interface RepositoryUsage {

    public void modifyCalender() {
       CalendarId calendarId = new CalendarId(...);
        Calendar calendar =  new Calendar(calendarId, "Project Calendar"...);
        CalendarRepository calendarRepository = new CalendarRepository();
        calendarRepository.add(calendar);
        // later ...
        Calendar calendarToRename =
        calendarRepository.findCalendar(calendarId);
        calendarToRename.rename("CollabOvation Project Calendar");
        // even later still ...
        Calendar calendarThatWasRenamed =
        calendarRepository.findCalendar(calendarId);

        assertEquals("CollabOvation Project Calendar",calendarThatWasRenamed.name());
    }
}
```
The entry was modified without asking the CalendarRepository to save changes to the Calendar instance. CalendarRepository doesn’t have a save() method because there is no need for one.

**Collection-oriented Repository truly mimics a collection in that no parts of the persistence mechanisms are surfaced to the client by its public interface.**

To do this the persistence mechanism must support the ability to implicitly track changes made to each persistent object that it manages.
Two strategies:

### Implicit Copy-on-Read
The persistence mechanism implicitly copies each persistent object on read when it is reconstituted from the data store and compares its private copy to the client’s copy on commit. When a change on this object happens, the persistence compares the object and flushes the changes to the data store.

### Implicit Copy-on-Write
The persistence mechanism manages all loaded persistent objects through a proxy. When an object is loaded, a thin proxy is created. The client performs actions on the proxy object which reflects behavior onto real object. When the proxy first receives a method invocation, it makes a copy of the managed object. The proxy tracks changes made to the state of the managed object and marks it dirty. On the commit of the transcation the changes where flushed.

### overall advantage
Persistent object changes are tracked implicitly, requiring no explicit client knowledge or intervention to make changes known to the persistence mechanism.
The bottom line here is that using a persistence mechanism like this, such as Hibernate, allows you to employ a traditional, collection-oriented Repository.

### When not to use
If your requirements demand a very high-performance domain with many, many objects in memory at any given time, this sort of mechanism is going to add gratuitous overhead, in both memory and execution. 


## Hibernate Implementation



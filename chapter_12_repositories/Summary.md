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
You need an interface which represents the collection-oriented design. The implementation can use hibernate. The return value of the add() and remove() method has a void return value. If you return a boolean it is not said, that the operation suceeded. You should carefully use removeAll() method on the interface. If you need to remove items, you should also think about logical remove and not physical remove. 

Another important part is the definition of finder methods like: 

```java
public interface CalendarEntryRepository  {

    public CalendarEntry calendarEntryOfId(
            Tenant aTenant,
            CalendarEntryId aCalendarEntryId);

    public Collection<CalendarEntry> calendarEntriesOfCalendar(
            Tenant aTenant,
            CalendarId aCalendarId);

    public Collection<CalendarEntry> overlappingCalendarEntries(
            Tenant aTenant,
            CalendarId aCalendarId,
            TimeSpan aTimeSpan);
}
```

You could also provide a function to get the next identy:
```java
public interface CalendarEntryRepository  {
    public CalendarEntryId nextIdentity();
}
```
Any code which wants to create a new instance can call the nextIdenty method and assign its new unique id.

Start with the implementation in hibernate. You should put the implementation in another package or module. You can also put all technical implementation classes in the infrastructure layer.
```java
public class HibernateCalendarEntryRepository
        implements CalendarEntryRepository  {
    ...
    @Override
    public void add(CalendarEntry aCalendarEntry) {
        try {
            this.session().saveOrUpdate(aCalendarEntry);
        } catch (ConstraintViolationException e) {
            throw new IllegalStateException(
                    "CalendarEntry is not unique.", e);
        }
    }

    @Override
    public void addAll(
            Collection<CalendarEntry> aCalendarEntryCollection) {
        try {
            for (CalendarEntry instance : aCalendarEntryCollection) {
                this.session().saveOrUpdate(instance);
            }
        } catch (ConstraintViolationException e) {
            throw new IllegalStateException(
                "CalendarEntry is not unique.", e);
        }
    }

    @Override
    public void remove(CalendarEntry aCalendarEntry) {
        this.session().delete(aCalendarEntry);
    }

    @Override
    public void removeAll(
            Collection<CalendarEntry> aCalendarEntryCollection) {
        for (CalendarEntry instance : aCalendarEntryCollection) {
            this.session().delete(instance);
        }
    }
    ...
}
```
The internal usage of saveOrUpdate() method illustrates the set like handling of the collection. To hide information about exceptions from the client the `ConstraintViolationException` is catched and converted to a more client-friendly IllegalStateException. You can also use domain specific exceptions. 

A special thing is the one-to-one bidirectional relation in Identity and Access Context. 
```java
public class HibernateUserRepository implements UserRepository  {
    ...
    @Override
    public void remove(User aUser) {
        this.session().delete(aUser.person());
        this.session().delete(aUser);
    }

    @Override
    public void removeAll(Collection<User> aUserCollection) {
        for (User instance : aUserCollection) {
            this.session().delete(instance.person());
            this.session().delete(instance);
        }
    }
    ...
}
```
Before you can delete the User, you must first delete the person. If you don´t delete the person object, it will be orphaned in its db table. This is a good reason to avoid bidirectional one-to-one mappings. You should use a constrained  unidirectional one-to-many mapping. Vernon is a strong opponent of Aggregate-managed persistence. His advice is to use Repository-only persistence, it should be a rule of thumb to avoid Aggregate-managed persistence.

Finder implementation:
```java
public class HibernateCalendarEntryRepository
        implements CalendarEntryRepository {
    ...
    @Override
    public Collection<CalendarEntry> overlappingCalendarEntries(
        Tenant aTenant, CalendarId aCalendarId, TimeSpan aTimeSpan) {
        Query query =
            this.session().createQuery(
                "from CalendarEntry as _obj_ " +
                "where _obj_.tenant = :tenant and " +
                  "_obj_.calendarId = :calendarId and " +
                  "((_obj_.repetition.timeSpan.begins between " +
                      ":tsb and :tse) or " +
                  " (_obj_.repetition.timeSpan.ends between " +
                      ":tsb and :tse))");

        query.setParameter("tenant", aTenant);
        query.setParameter("calendarId", aCalendarId);
        query.setParameter("tsb", aTimeSpan.begins(), Hibernate.DATE);
        query.setParameter("tse", aTimeSpan.ends(), Hibernate.DATE);

        return (Collection<CalendarEntry>) query.list();
    }
}
```

## Toplink Implementation
TopLink has a Session and a Unit of Work. Example Usage:
```java
public class UnitOfWorkUsage {

    public void usageExample(
        Calendar calendar = session.readObject(...);
        UnitOfWork unitOfWork = session.acquireUnitOfWork();
        Calendar calendarToRename = unitOfWork.registerObject(calendar);
        calendarToRename.rename("CollabOvation Project Calendar");
        unitOfWork.commit();
    }

    public void add(Calendar aCalendar) {
        this.unitOfWork().registerNewObject(aCalendar);
    }

    // to register change listener on object so you can
    // persist it after modify it.
    public Calendar editingCopy(Calendar aCalendar) {
        return (Calendar) this.unitOfWork().registerObject(aCalendar);
    }

    // or

    ublic void useEditingMode();
}
```
The UnitOfWork provides a much more efficient use of memory and processing power. 
You must explicitly inform the UnitOfWork that you intend to modify the object. There is no clone or editing copy created. With the register method the change tracker is activated so you can commit the changes. 


## Collection-Oriented Repositories
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

    public void useEditingMode();
}
```
The UnitOfWork provides a much more efficient use of memory and processing power. 
You must explicitly inform the UnitOfWork that you intend to modify the object. There is no clone or editing copy created. With the register method the change tracker is activated so you can commit the changes. 


## Persistence-Oriented (save-based) Repositories
When a collection-oriented repo does´t work:
* persistence mechanism doesn´t implicitly or explicitly detect and track object changes

This happens, when using an in-memory DataFabric or another NoSQL key-value data store. Every time you modify an object, you need the ```save()``` method. If you plan to use a NoSQL DB instead of a relational db you should also consider using this style. 

### Take-away
We must explicitly ```put()``` both new and changed objects into the store. It simplifies the writes and reads of aggregates, thats why they were called Aggregate Stores or Aggregate-Oriented Databases. The data is saved in a key-value map.

```java
public class SimpleWriteRead {

    public void usageExample() {
       cache.put(product.productId(), product);

        // later ...
        product = cache.get(productId);
    }
}
```
In this example the product is serialized with java standard serialization. You need to think about performance when serialize objects. It is not as simple as using a put in a Java Map.

## Coherence Implementation
Implementation example for a Coherence DB.
Interface:
```java
public interface ProductRepository  {
    public ProductId nextIdentity();
    public Collection<Product> allProductsOfTenant(Tenant aTenant);
    public Product productOfId(Tenant aTenant, ProductId aProductId);
    public void remove(Product aProduct);
    public void removeAll(Collection<Product> aProductCollection);
    public void save(Product aProduct);
    public void saveAll(Collection<Product> aProductCollection);
}
```
The main difference is using ```save()``` and ```saveAll()``` instead of ```add()``` and ```addAll()```. In this style Aggregates need to be added when created and updated:

```java
public class CreateAndUpdate {

    public void usageExample() {
        Product product = new Product(...);

        productRepository.save(product);

        // later ...

        Product product =
        productRepository.productOfId(tenantId, productId);

        product.reprioritizeFrom(backlogItemId, orderOfPriority);

        productRepository.save(product);
    }
}
```

Repository implementation:


```java
public class CoherenceProductRepository implements ProductRepository {
    private Map<Tenant,NamedCache> caches;

    public CoherenceProductRepository() {
        super();
        this.caches = new HashMap<Tenant,NamedCache>();
    }
    ...
    private synchronized NamedCache cache(TenantId aTenantId) {
        NamedCache cache = this.caches.get(aTenantId);

        if (cache == null) {
            cache = CacheFactory.getCache(
                    "agilepm.Product." + aTenantId.id(),
                    Product.class.getClassLoader());

            this.caches.put(aTenantId, cache);
        }

        return cache;
    }
    ...
}
```
In the case of the Agile Project Management Context, the team has chosen to place Repository technical implementations in the Infrastructure Layer. There are various Coherence named cache strategies that could be designed. In this case the team has chosen to cache using the following namespace:
1. First level by the Bounded Context short name: agilepm
2. Second level by the Aggregate simple name: Product
3. Third level by the unique identity of each tenant: TenantId

Benefits:
* model of each Bounded Context, Aggregate, and tenant that is managed by Coherence can be tuned and scaled separately. 
* each tenant is completely segregated from all others.

Further implementations:

```java
public class CoherenceProductRepository implements ProductRepository {
    @Override
    public void save(Product aProduct) {
        this.cache(aProduct.tenantId())
                .put(this.idOf(aProduct), aProduct);
    }

    @Override
    public void saveAll(Collection<Product> aProductCollection) {
        if (!aProductCollection.isEmpty()) {
            TenantId tenantId = null;

            Map<String,Product> productsMap =
                new HashMap<String,Product>(aProductCollection.size());

            for (Product product : aProductCollection) {
                if (tenantId == null) {
                    tenantId = product.tenantId();
                }
                productsMap.put(this.idOf(product), product);
            }

            this.cache(tenantId).putAll(productsMap);
        }
    }
    ...
    private String idOf(Product aProduct) {
        return this.idOf(aProduct.productId());
    }

    private String idOf(ProductId aProductId) {
        return aProductId.id();
    }

    @Override
    public void remove(Product aProduct) {
        this.cache(aProduct.tenant()).remove(this.idOf(aProduct));
    }

    @Override
    public void removeAll(Collection<Product> aProductCollection) {
        for (Product product : aProductCollection) {
            this.remove(product);
        }
    }
}
```
Note the two editions of ```idOf()```. Both methods return a String which is used as unique identity for the cache. To reduce network traffic the ```saveAll()``` does not call the ```save()``` method. It uses batch-processing. The ```removeAll()``` method uses ```remove()``` method because there is no method on HashMap which removes all elements, so you need to iterate.

### Finder Methods
```java
public class CoherenceProductRepository
        implements ProductRepository {
    ...

    @SuppressWarnings("unchecked")
    @Override
    public Collection<Product> allProductsOfTenant(Tenant aTenant) {
        Set<Map.Entry<String, Product>> entries = this.cache(aTenant).entrySet();

        Collection<Product> products =
            new HashSet<Product>(entries.size());

        for (Map.Entry<String, Product> entry : entries) {
            products.add(entry.getValue());
        }

        return products;
    }

    @Override
    public Product productOfId(Tenant aTenant, ProductId aProductId) {
       return (Product) this.cache(aTenant).get(this.idOf(aProductId));
    }
    ...
}
```
Through a good key design, the ```allProductsOfTenant()``` method is really simple. 

## MongoDB Implementation
It is similar to the Coherence Implementation. What we need:
1. Serialization of objects. MongoDB uses a special form of JSON called BSON, which is a binary JSON format.
2. A unique identity generated by MongoDB and assigned to the Aggregate.
3. A reference to the MongoDB node/cluster.
4. A unique collection in which to store each Aggregate type. All instances of each Aggregate type must be stored as a set of serialized documents (key-value pairs) in their own collection.

```java
public class MongoProductRepository
        extends MongoRepository<Product>
        implements ProductRepository {

    public MongoProductRepository() {
        super();

        this.serializer(new BSONSerializer<Product>(Product.class));
    }
    ...
}
```
The BSONSerializer serializes objects using direct field access. So you dont need getters and setters on the object. If you want to migrate, you can simply override mapping for each field on deserialization:


```java
public class MongoProductRepository
        extends MongoRepository<Product>
        implements ProductRepository {

    public MongoProductRepository() {
        super();
        this.serializer(new BSONSerializer<Product>(Product.class));

        Map<String, String> overrides = new HashMap<String, String>();
        overrides.put("description", "summary");
        this.serializer().registerOverrideMappings(overrides);
    }

    public ProductId nextIdentity() {
        return new ProductId(new ObjectId().toString());
    }
    ...
}
```
You need to weigh the trade-offs of lazy migration approach.



# Chapter 10 Aggregates

Clustering Entities and Value Objects into an Aggregate with a carefully crafted consistency boundary. The Aggregate pattern discusses composition and alludes to information hiding.

## Chapter Learning content
* experience negative consequences of bad aggregate design
* learn aggregate rules of thumb
* consider advantages of small aggregates
* see why you should reference aggregates by identifier
* discover advantes of eventual consistency
* learn aggregate implementation techniques


## Bad aggregate design by example
In the book the example of the agile context is described. The core domain is scrum. The bounded context is **Agile Project Management Context** the application named ProjectOvation. The team started with the following statements of the ubiquitous language:
* Products have backlog items, releases, and sprints.
* New product backlog items are planned.
* New product releases are scheduled.
* New product sprints are scheduled.
* A planned backlog item may be scheduled for release.
* A scheduled backlog item may be committed to a sprint.

### First attempt
The team has focused on the word Product for their aggregate design. So the first attempt was a large Product aggregate:
```java
public class Product extends ConcurrencySafeEntity  {
    private Set<BacklogItem> backlogItems;
    private String description;
    private String name;
    private ProductId productId;
    private Set<Release> releases;
    private Set<Sprint> sprints;
    private TenantId tenantId;
}
```
![aggregate1](img/aggregate1.PNG)

The big aggreate was not practical because of transactional failures in the multi-user system. The commit fails because every entity has its own version. If two users work with one version and a third user updates the version, the commit will be rejected. So this is not working. Problem is also artificial constraints imposed by the developer. So a creation of a new backlog item corresponding with Sprint, which is not the sense of the business model.

### second attempt

The second attempt consists of four distinct Aggregates. Each of the dependency is associated by inference using a common ProductId

![aggregate2](img/aggregate2.PNG)

The new code looks like:
```java
public class Product {
    
    public BacklogItem planBacklogItem(
        String aSummary, String aCategory,
        BacklogItemType aType, StoryPoints aStoryPoints) {
            //plan
    }

    public Release scheduleRelease(
        String aName, String aDescription,
        Date aBegins, Date anEnds) {
        //schedule release
    }

    public Sprint scheduleSprint(
        String aName, String aGoals,
        Date aBegins, Date anEnds) {
        //schedule sprint
    }
}
```

All methods have a CQS query contract and act as Factories. If an application service wants to create a BacklogItem it can do it this way:

```java
public class ProductBacklogItemService {
    
    @Transactional
    public void planProductBacklogItem(
        String aTenantId, String aProductId,
        String aSummary, String aCategory,
        String aBacklogItemType,
        String aStoryPoints) {

        Product product =
            productRepository.productOfId(
                    new TenantId(aTenantId),
                    new ProductId(aProductId));

        BacklogItem plannedBacklogItem =
            product.planBacklogItem(
                    aSummary,
                    aCategory,
                    BacklogItemType.valueOf(aBacklogItemType),
                    StoryPoints.valueOf(aStoryPoints));

        backlogItemRepository.add(plannedBacklogItem);
    }
}
```
We solved the transactional failures by modelling it away. You can simultaneous create backlog items.

**RULE: Model True Invariant in Consistency Boundaries**


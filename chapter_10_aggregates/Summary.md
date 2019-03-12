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

### invariants
An invariant is a business rule that must always be consistent. Two kinds of consistency:
* transactional consistency -> immediate and atomic
* eventual consistency 

An invariant can be:
c = a + b

We would model this as a consistent boundary that the rule is always true. The consistency of everything outside the boundary is irrelevant to the aggregate. So **Aggregate is synonymous with transactional consistency boundary**.
A properly designed Aggregate is one that can be modified in any way required by the business with its invariants completely consistent within a single transaction. 

**LEARNING: We cannot correctly reason on Aggregate design without applying transactional analysis.**

Aggregates are chiefly about consistency boundaries and not driven by a desire to design object graphs.

## **Rule: Design Small Aggregates**
If we have to large Aggregates we will have problems with guarantee that all transactions would succeed and it limits performance. If we start with a large aggregate we will face a lot of problems later if the model grows. So don´t do this.

**How to find right size for Aggregates?**

Normally you should design an aggregate with a root entity and some value objects. In the evans book there is an example where it makes sense to use multiple entities. A purchase order is assigned a maximum allowable total, and the sum of all line items must not surpass the total. This rule is tricky to enforce if it´s not in the same aggregate.

## **Rule: Reference Other Aggregates by Identity**
When designing Aggregates, we may desire a compositional structure that allows for traversal through deep object graphs, but that is not the motivation of the pattern. However, we must keep in mind that this does not place the referenced Aggregate inside the consistency boundary of the one referencing it. See the following picture. The Product is still outside the Backlog Aggregate.

![aggregate3](img/aggregate3.PNG)

The code looks like:
```java
public class BacklogItem extends ConcurrencySafeEntity  {
     ...
    private Product product;
    ...
}
```
With this implementation we have the following implications:
* Both the referencing Aggregate (BacklogItem) and the referenced Aggregate (Product) must not be modified in the same transaction. Only one or the other may be modified in a single transaction.
* If you are modifying multiple instances in a single transaction, it may be a strong indication that your consistency boundaries are wrong. If so, it is possibly a missed modeling opportunity; a concept of your Ubiquitous Language has not yet been discovered 
* If you are attempting to previous point, and doing so influences a large-cluster Aggregate with all the previously stated caveats, it may be an indication that you need to use eventual consistency (see later in this chapter) instead of atomic consistency.

We can´t modify another aggregate if we hold any reference. So **Making Aggregates Work Together through Identity References**

![aggregate4](img/aggregate4.PNG)

The new code looks like:
```java
public class BacklogItem extends ConcurrencySafeEntity  {
    ...
    private ProductId productId;
    ...
}
```
advantages:
* aggregates are smaller
* model can perform better 
* need less memory

If you use references, you also need navigation through the model. This should be done by a Repository or Domain Service to to lookup dependend objects.
```java
public class ProductBacklogItemService ... {
    ...
    @Transactional
    public void assignTeamMemberToTask(String aTenantId,
        String aBacklogItemId,
        String aTaskId,
        String aTeamMemberId) {

        BacklogItem backlogItem =
            backlogItemRepository.backlogItemOfId(
                new TenantId(aTenantId),
                new BacklogItemId(aBacklogItemId));

        Team ofTeam =
            teamRepository.teamOfId(
                backlogItem.tenantId(),backlogItem.teamId());

        backlogItem.assignTeamMemberToTask(
                new TeamMemberId(aTeamMemberId),
                ofTeam,
                new TaskId(aTaskId));
    }
    ...
}
```
Limiting a model to using only reference by identity could make it more difficult to serve clients that assemle and render User Interface views. If the query overhead causes performance it may be worth to considering CQRS.
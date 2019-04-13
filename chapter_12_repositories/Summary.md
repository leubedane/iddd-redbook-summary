# Chapter 12 Repository

A repository is a storage location. When you put things in you expect that it would be in the same state when you read it afterwards. At some point you want to remove it.

For each type of object that needs global access, create an object that can provide the illusion of in-memory collection. Provide methods to add and remove objects. Provide methods that select objects based on some criteria and return fully instantiated objects or collections of objects whose attribute values meet the criteria. Every persistent Aggregate type will have a Repository.

## Chapter Learning content
* Learn two different kinds of Repositories
* Implement Repositories for Hibernate, TopLink, Coherence, and MongoDB
* Understand why you might need additional behavior on a Repository’s interface.
* Become familiar with the challenges of designing Repositories for type hierarchies.
* Differences between Repositories and Data Access Objects



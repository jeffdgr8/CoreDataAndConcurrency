## Sample to illustrate the usage of the [Encrypted Core Data](https://github.com/project-imas/encrypted-core-data) SQLCipher library

This sample creates a Core Data stack that consists of two persistent store coordinators. One as the main coordinator, connected to a private queue persisting context, with a main queue context as its child. The second persistent store coordinator has a private queue batch worker context, to be used for batch import jobs, as described in [this 2013 WWDC session](https://developer.apple.com/videos/play/wwdc2013/211/) (~27:50).

![Core Data Stack](https://www.bignerdranch.com/assets/img/blog/2015/09/BNR_Stack.png)

###### _(Image from [Big Nerd Ranch core data stack](https://www.bignerdranch.com/blog/introducing-the-big-nerd-ranch-core-data-stack/))_

`CoreDataManager.swift` has a constant `encrypt` which can be toggled to test the sample using the Encrypted Core Data SQLCipher store or the standard SQLite store.

The sample will continuously create new `Note` objects on the batch import context and then fetch all `Note` objects on the main context while at the same time deleting some random `Note` objects on the batch import context.

Using the standard SQLite store, everything works as expected. ~Using the encrypted store, the parallel operations on multiple persistent store coordinators will often result in failed operations with the error: `Error Domain=NSSQLiteErrorDomain Code=5 "(null)" UserInfo={EncryptedStoreErrorMessage=database is locked}, The operation couldn’t be completed. (NSSQLiteErrorDomain error 5.)` This is likely because Encrypted Core Data does not utilize write-ahead logging (WAL) journal mode, as the default Core Data SQLite implementation does. [Issue #212](https://github.com/project-imas/encrypted-core-data/issues/212) discusses this.~ This is now fixed with my [pull request](https://github.com/project-imas/encrypted-core-data/pull/300) and [specifying the WAL journal mode and busy timeout in the store options](https://github.com/jeffdgr8/CoreDataAndConcurrency/commit/d413f88186a791061a0f3bfa96e8f9f017764bc8).

## Forked Sample Readme

### [Core Data and Concurrency](https://cocoacasts.com/core-data-and-concurrency/)

#### Author: Bart Jacobs

Up to now, we've used a single managed object context, which we created in the `CoreDataManager` class. This works fine, but there will be times when one managed object context won't suffice.

What happens if you access the same managed object context from different threads? What do you expect happens? What happens if you pass a managed object from a background thread to the main thread? Let's start with the basics.

## Concurrency Basics

Before we explore solutions for using Core Data in multithreaded applications, we need to know how Core Data behaves on multiple threads. The documentation is very clear about this. Core Data expects to be run on a single thread. Even though that thread doesn't have to be the main thread, Core Data was not designed to be accessed from different threads.

> Core Data expects to be run on a single thread.

The Core Data team at Apple is not naive, though. It knows that a persistence framework needs to be accessible from multiple threads. A single thread, the main thread, may be fine for many applications. More complex applications need a robust, multithreaded persistence framework.

Before I show you how Core Data can be used across multiple threads, I lay out the basic rules for accessing Core Data in a multithreaded application.

### Managed Objects

`NSManagedObject` instances should never be passed from one thread to another. If you need to pass a managed object from one thread to another, you use a managed object's `objectID` property.

The `objectID` property is of type `NSManagedObjectID` and uniquely identifies a record in the persistent store. A managed object context knows what to do when you hand it an `NSManagedObjectID` instance. There are three methods you need to know about:

- `object(with:)`
- `existingObject(with:)`
- `registeredObject(for:)`

The first method, `object(with:)`, returns a managed object that corresponds to the `NSManagedObjectID` instance. If the managed object context doesn't have a managed object for that object identifier, it asks the persistent store coordinator. This method always returns a managed object.

Know that `object(with:)` throws an exception if no record can be found for the object identifier it receives. For example, if the application deleted the record corresponding with the object identifier, Core Data is unable to hand your application the corresponding record. The result is an exception.

The `existingObject(with:)` method behaves in a similar fashion. The main difference is that the method throws an error if it cannot fetch the managed object corresponding to the object identifier.

The third method, `registeredObject(for:)`, only returns a managed object if the record you're asking for is already registered with the managed object context. In other words, the return value is of type `NSManagedObject?`. The managed object context doesn't fetch the corresponding record from the persistent store if it cannot find it.

The object identifier of a record is similar, but not identical, to the primary key of a database record. It uniquely identifies the record and enables your application to fetch a particular record regardless of what thread the operation is performed on.

```swift
let objectID = managedObject.objectID

DispatchQueue.main.async {
    let managedObject = managedObjectContext?.object(with: objectID)
    ...
}
```

In the example, we ask a managed object context for the managed object that corresponds with `objectID`, an `NSManagedObjectID` instance. The managed object context first looks if a managed object with a corresponding object identifier is registered in the managed object context. If there isn't, the managed object is fetched or returned as a fault.

It's important to understand that a managed object context always expects to find a record if you give it an `NSManagedObjectID` instance. That is why `object(with:)` returns an object of type `NSManagedObject`, not `NSManagedObject?`.

### Managed Object Context

Creating an `NSManagedObjectContext` instance is a cheap operation. You should never share managed object contexts between threads. This is a hard rule you shouldn't break. The `NSManagedObjectContext` class isn't thread safe. Plain and simple.

> You should never share managed object contexts between threads. This is a hard rule you shouldn't break.

### Persistent Store Coordinator

Even though the `NSPersistentStoreCoordinator` class isn't thread safe either, the class knows how to lock itself if multiple managed object contexts request access, even if these managed object contexts live and operate on different threads.

It's fine to use one persistent store coordinator, which is accessed by multiple managed object contexts from different threads. This makes Core Data concurrency a little bit easier ... a little bit.

**Read this article on [Cocoacasts](https://cocoacasts.com/core-data-and-concurrency/)**.

CoreDataMonk
============

CoreDataMonk is a helper library to make using CoreData easier and safer in the concurrent setup.
The main features of CoreDataMonk are:

+ Allow you to setup CoreData in different ways easily
(three tier, two-tier with auto merge, multiple main context with manual reload, etc...)
+ API that is easy to use and understand
+ Swift friendly query expression
+ Serialized update to avoid data consistency problem (optional)
+ Use exception for error handling

Getting started
---------------

Setup CoreDataMonk is easy, and the default provides three-tier NSManagedObjectContext setup,
that is good for most applications. You can do this with:

````swift
// first pick the name you want for the global main context
var World: CoreDataMainContext!

// then in your AppDelegate
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    do {
        let dataStack = try CoreDataStack()
        try dataStack.addDatabaseStore()
        World = try CoreDataMainContext(stack: dataStack)

        ...
    } catch let error {
        fatalError("fail to init core data: \(error)")
    }
    ...
}
````

It is possible to use multiple store by the configuration in Xcode model:

````swift
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    do {
        let dataStack = try CoreDataStack()
        try dataStack.addInMemoryStore(configuration: "InMemory")
        try dataStack.addDatabaseStore(configuration: "Database", resetOnFailure: true)
        World = try CoreDataMainContext(stack: dataStack)

        ...
    } catch let error {
        fatalError("fail to init core data: \(error)")
    }
    ...
}
````

Fetch object
------------

OK now you can fetch data from main context in the view controller:

````swift
// you can add % prefix to tell CoreDataMonk it is for key name
let monk = try World.fetch(Person.self, %"name" == "Monk")

let warrior = try World.fetch(Person.self, %"name" == "Warrior" && %"location" == "Taipei")

````

To fetch a list of objects:

````swift
let monks = try World.fetchAll(Person.self)
````

To give more conditions:

````swift
let monks = try World.fetchAll(Person.self,
    %"age" >= 18 || %"location" == "Taipei",
    orderBy: .Ascending("name") | .Descending("age"),
    options: .Limit(100) | .Offset(20)
)
````

And you can query value too:

````swift
let age = try World.queryValue(Person.self, .Average("age")) as! NSNumber
````

And of course more values:

````swift
let info_list = try World.query(Person.self,
    .Select("location") | .Average("age"),
    %"name" == "Monk",
    orderBy: .Descending("age"),
    groupBy: %"location"
)

for info in info_list {
    let location = info["location"] as! String
    let age = info["age"] as! NSNumber
    ...
}
````

Create and update object
------------------------

To create or update object, you have to call `.beginUpdate()`, and you need to call `.commit()` before the block ends,
otherwise all changes will be discarded.


````swift
World.beginUpdate() {
    update in

    let farmer = try update.create(Person.self)
    farmer.name = "Framer"
    farmer.age = 28

    let knight = try update.fetchOrCreate(Person.self, %"name" == "Knight" && %"age" == 18)
    knight.friend = farmer

    let monk = try update.fetch(Person.self, %"name" == "Monk")
    monk.age = 44
    monk.friend = knight

    try update.commit()
}

// or you prefer to wait for the update to complete
World.beginUpdateAndWait() {
    update in

    ...
}
````

If you already fetch object in the main context, you can use that in the update:

````swift
let warrior = try World.fetch(Person.self, %"name" == "Warrior" && %"location" == "Taipei")

World.beginUpdate() {
    update in

    let warrior = try update.use(warrior)

    let monk = try update.fetch(Person.self, %"name" == "Monk")
    monk.friend = warrior

    try update.commit()
}
````

Update can be used in nested block. The changes are discarded only after it is de-inited.
For example, if you need to fetch data from server to update data:

````swift
World.beginUpdate() {
    update in

    let monk = try update.fetch(Person.self, %"name" == "Monk")

    // this will start network connection and return result in callback block
    remote_server.findAge(monk.name, location: monk.location) {
        age in

        // you need to call .perform() in nested block
        update.perform() {
            update in

            // it is safe to use the object directly in the same update
            monk.age = age

            try update.commit()
        }
    }
}
````

If the update takes multiple steps and can not be done in one block, you can use `.beginTransaction()`:

````swift
let transaction = World.beginTransaction()

transaction.perform() {
    update in

    ...
}

...

transaction.perform() {
    update in

    ...
}

...

transaction.perform() {
    update in

    ...
    try update.commit()
}
````

In fact, `.beginUpdate()` is just a temporary transaction with perform:
````swift
public func beginUpdate(block: (CoreDataUpdate) throws -> Void) {
    beginTransaction().perform(block)
}
````

Predicate expression
--------------------

CoreDataMonk add some syntactic sugar to the predicate expression, so the code looks more natural.

First you need a way specify key name, thus not to confuse with constant value:

Expression              | Example                   | Description
------------------------|---------------------------|-------------
`%String`               | `%"name"`                 | key "name"
`%String%.any`          | `%"friend.age"%.any`      | key "friend.age" with ANY modifier
`%String%.all`          | `%"friend.age"%.all`      | key "friend.age" with ALL modifier

CoreDataMonk has some mappings to the predicate:

Expression                | Example                     | Description
--------------------------|-----------------------------|-------------
`%String == Any`          | `%"name" == "monk"`         | The same as `NSPredicate(format: "%K == %@", "name", "monk")`
`%String == %String`      | `%"name" == %"location"`    | The same as `NSPredicate(format: "%K == %K", "name", "location")`
`!=`, `>`, `<`, `>=`, `<=` |                            | Just like `==`
`.Where(String, Any...)`  | `.Where("name like %@", pattern)` | The same as `NSPredicate(format: "name like %@", pattern)`
`.Predicate(NSPredicate)` | `.Predicate(my_predicate)`  | The same as `my_predicate`

You use `&&`, `||` and `!` operators to combine predicate:

Operator    | Example                               | Description
------------|---------------------------------------|-------------
`&&`        | `%"name" == name && %"age" > age`     | The same as `NSPredicate(format: "name == %@ and age > %@", name, age)`
`||`        | `%"name" == name || %"name" == name + " sam"` | The same as `NSPredicate(format: "name == %@ or name = %@", name, name + " sam")`
`!`         | `!(%"name" == name && %"age" > age)`  | The same as `NSPredicate(format: "not (name == %@ and age > %@)", name, age)`

`orderBy:` expression
---------------------

The `orderBy:` is supported by `.fetchAll`, `.fetchResults` and `.query` methods:

Expression              | Example                   | Description
------------------------|---------------------------|-------------
`.Ascending(String)`    | `.Ascending("name")`      | The same as `NSSortDescriptor(key: "name", ascending: true)`
`.Descending(String)`   | `.Descending("name")`     | The same as `NSSortDescriptor(key: "name", ascending: false)`

You can use `|` operator to combine two or more expressions:

Operator    | Example                                        | Description
------------|------------------------------------------------|-------------
`|`         | `.Ascending("name") | .Descending("location")` | The same as `[NSSortDescriptor(key: "name", ascending: true), NSSortDescriptor(key: "location", ascending: false)]`

`options:` expression
---------------------

You can set options to adjust `NSFetchRequest`, it is supported by all `.fetch` and `.query` methods:

Expression                  | Description
----------------------------|------------
`.NoSubEntities`            | `fetchRequest.includesSubentities = false`
`.NoPendingChanges`         | `fetchRequest.includesPendingChanges = false`
`.NoPropertyValues`         | `fetchRequest.includesPropertyValues = false`
`.Limit(Int)`               | `fetchRequest.fetchLimit = Int`
`.Offset(Int)`              | `fetchRequest.fetchOffset = Int`
`.Batch(Int)`               | `fetchRequest.fetchBatchSize = Int`
`.Prefetch([String])`       | `fetchRequest.relationshipKeyPathsForPrefetching = [String]`
`.PropertiesOnly([String])` | `fetchRequest.propertiesToFetch = [String]` // ignored in .query
`.Distinct`                 | `fetchRequest.returnsDistinctResults = true`
`.Tweak(NSFetchRequest -> Void)` | allow block to modify fetchRequest

`.query` and `.queryValue` select expression
--------------------------------------------

In `.query`, you need to specify the select targets you want to return. You can specify
property, expression or aggregated function. The aggregated function may have optional alias,
if user does not specify one, the default is to use property name. It is important to make sure
each select target having unique alias, as it is used as key in returned dictionary.

Expression                               | Description
-----------------------------------------|-------------
`.Select(String...)`                     | to get value of properties, `.Select("name", "age")` will add two targets
`.Expression(NSExpressionDescription)`   | to get value of  `my_expression`
`.Average(String, alias: String? = nil)` | to get average of property
`.Sum(String, alias: String? = nil)`     | to get sum of property
`.StdDev(String, alias: String? = nil)`  | to get standard deviation of property
`.Min(String, alias: String? = nil)`     | to get minimum value of property
`.Max(String, alias: String? = nil)`     | to get maximum value of property
`.Median(String, alias: String? = nil)`  | to get median value of property
`.Count(String, alias: String? = nil)`   | to get the number of returned values

You can use `|` operator to combine two or more select targets:

Operator    | Example                               | Description
------------|---------------------------------------|-------------
`|`         | `.Average("age") | .Min("age")`       | Combine them into select targets

`groupBy:` expression
---------------------

The same key expression, but only apply to by `.query` method:

Expression              | Example                   | Description
------------------------|---------------------------|-------------
`%String`               | `%"name"`                 | "name" as object property name

You can use `|` operator to combine two or more keys:

Operator    | Example                               | Description
------------|---------------------------------------|-------------
`|`         | `%"name" | %"location"`               | The same as `["name", "location"]`

`having:` expression
--------------------

The same as predicate expression, but only apply to `.query` method with `groupBy:`.



# MongoDB Integration

This document explains how to integrate MongoDB as a database provider to ABP based applications and how to configure it.

## Installation

`Volo.Abp.MongoDB` is the main NuGet package for the MongoDB integration. Install it to your project (for a layered application, to your data/infrastructure layer), You can use the [ABP CLI](../../../cli) to install it to your project. Execute the following command in the folder of the .csproj file of the layer:

```
abp add-package Volo.Abp.MongoDB
```

> If you haven't done it yet, you first need to install the [ABP CLI](../../../cli). For other installation options, see [the package description page](https://abp.io/package-detail/Volo.Abp.MongoDB).

Then add `AbpMongoDbModule` module dependency to your [module](../../architecture/modularity/basics.md):

```c#
using Volo.Abp.MongoDB;
using Volo.Abp.Modularity;

namespace MyCompany.MyProject
{
    [DependsOn(typeof(AbpMongoDbModule))]
    public class MyModule : AbpModule
    {
        //...
    }
}
```

## Creating a Mongo Db Context

ABP introduces **Mongo Db Context** concept (which is similar to Entity Framework Core's DbContext) to make it easier to use collections and configure them. An example is shown below:

```c#
public class MyDbContext : AbpMongoDbContext
{
    public IMongoCollection<Question> Questions => Collection<Question>();

    public IMongoCollection<Category> Categories => Collection<Category>();

    protected override void CreateModel(IMongoModelBuilder modelBuilder)
    {
        base.CreateModel(modelBuilder);

        //Customize the configuration for your collections.
    }
}
```

* It's derived from `AbpMongoDbContext` class.
* Adds a public `IMongoCollection<TEntity>` property for each mongo collection. ABP uses these properties to create default repositories by default.
* Overriding `CreateModel` method allows to configure collection configuration.

### Configure Mapping for a Collection

ABP automatically register entities to MongoDB client library for all `IMongoCollection<TEntity>` properties in your DbContext. For the example above, `Question` and `Category` entities are automatically registered.

For each registered entity, it calls `AutoMap()` and configures known properties of your entity. For instance, if your entity implements `IHasExtraProperties` interface (which is already implemented for every aggregate root by default), it automatically configures `ExtraProperties`.

So, most of times you don't need to explicitly configure registration for your entities. However, if you need it you can do it by overriding the `CreateModel` method in your DbContext. Example:

````csharp
protected override void CreateModel(IMongoModelBuilder modelBuilder)
{
    base.CreateModel(modelBuilder);

    modelBuilder.Entity<Question>(b =>
    {
        b.CollectionName = "MyQuestions"; //Sets the collection name
        b.BsonMap.UnmapProperty(x => x.MyProperty); //Ignores 'MyProperty'
    });
}
````

This example changes the mapped collection name to 'MyQuestions' in the database and ignores a property in the `Question` class.

If you only need to configure the collection name, you can also use `[MongoCollection]` attribute for the collection in your DbContext. Example:

````csharp
[MongoCollection("MyQuestions")] //Sets the collection name
public IMongoCollection<Question> Questions => Collection<Question>();
````

### Configure Indexes and CreateCollectionOptions for a Collection

You can configure indexes and `CreateCollectionOptions` for your collections by overriding the `CreateModel` method. Example:

````csharp
protected override void CreateModel(IMongoModelBuilder modelBuilder)
{
    base.CreateModel(modelBuilder);

    modelBuilder.Entity<Question>(b =>
    {
        b.CreateCollectionOptions.Collation = new Collation(locale:"en_US", strength: CollationStrength.Secondary);
        b.ConfigureIndexes(indexes =>
            {
                indexes.CreateOne(
                    new CreateIndexModel<BsonDocument>(
                        Builders<BsonDocument>.IndexKeys.Ascending("MyProperty"),
                        new CreateIndexOptions { Unique = true }
                    )
                );
            }
        );
    });
}
````

This example sets a collation for the collection and creates a unique index for the `MyProperty` property.

### Configure the Connection String Selection

If you have multiple databases in your application, you can configure the connection string name for your DbContext using the `[ConnectionStringName]` attribute. Example:

````csharp
[ConnectionStringName("MySecondConnString")]
public class MyDbContext : AbpMongoDbContext
{

}
````

If you don't configure, the `Default` connection string is used. If you configure a specific connection string name, but not define this connection string name in the application configuration then it fallbacks to the `Default` connection string.

## Registering DbContext To Dependency Injection

Use `AddAbpDbContext` method in your module to register your DbContext class for [dependency injection](../../fundamentals/dependency-injection.md) system.

```c#
using Microsoft.Extensions.DependencyInjection;
using Volo.Abp.MongoDB;
using Volo.Abp.Modularity;

namespace MyCompany.MyProject
{
    [DependsOn(typeof(AbpMongoDbModule))]
    public class MyModule : AbpModule
    {
        public override void ConfigureServices(ServiceConfigurationContext context)
        {
            context.Services.AddMongoDbContext<MyDbContext>();

            //...
        }
    }
}
```

### Add Default Repositories

ABP can automatically create default [generic repositories](../../architecture/domain-driven-design/repositories.md) for the entities in your DbContext. Just use `AddDefaultRepositories()` option on the registration:

````C#
services.AddMongoDbContext<MyDbContext>(options =>
{
    options.AddDefaultRepositories();
});
````

This will create a repository for each [aggregate root entity](../../architecture/domain-driven-design/entities.md) (classes derived from `AggregateRoot`) by default. If you want to create repositories for other entities too, then set `includeAllEntities` to `true`:

```c#
services.AddMongoDbContext<MyDbContext>(options =>
{
    options.AddDefaultRepositories(includeAllEntities: true);
});
```

Then you can inject and use `IRepository<TEntity, TPrimaryKey>` in your services. Assume that you have a `Book` entity with `Guid` primary key:

```csharp
public class Book : AggregateRoot<Guid>
{
    public string Name { get; set; }

    public BookType Type { get; set; }
}
```

(`BookType` is a simple `enum` here) And you want to create a new `Book` entity in a [domain service](../../architecture/domain-driven-design/domain-services.md):

```csharp
public class BookManager : DomainService
{
    private readonly IRepository<Book, Guid> _bookRepository;

    public BookManager(IRepository<Book, Guid> bookRepository) //inject default repository
    {
        _bookRepository = bookRepository;
    }

    public async Task<Book> CreateBook(string name, BookType type)
    {
        Check.NotNullOrWhiteSpace(name, nameof(name));

        var book = new Book
        {
            Id = GuidGenerator.Create(),
            Name = name,
            Type = type
        };

        await _bookRepository.InsertAsync(book); //Use a standard repository method

        return book;
    }
}
```

This sample uses `InsertAsync` method to insert a new entity to the database.

### Add Custom Repositories

Default generic repositories are powerful enough in most cases (since they implement `IQueryable`). However, you may need to create a custom repository to add your own repository methods.

Assume that you want to delete all books by type. It's suggested to define an interface for your custom repository:

```csharp
public interface IBookRepository : IRepository<Book, Guid>
{
    Task DeleteBooksByType(
        BookType type,
        CancellationToken cancellationToken = default(CancellationToken)
    );
}
```

You generally want to derive from the `IRepository` to inherit standard repository methods. However, you don't have to. Repository interfaces are defined in the domain layer of a layered application. They are implemented in the data/infrastructure layer (`MongoDB` project in a [startup template](https://abp.io/Templates)).

Example implementation of the `IBookRepository` interface:

```csharp
public class BookRepository :
    MongoDbRepository<BookStoreMongoDbContext, Book, Guid>,
    IBookRepository
{
    public BookRepository(IMongoDbContextProvider<BookStoreMongoDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task DeleteBooksByType(
        BookType type,
        CancellationToken cancellationToken = default(CancellationToken))
    {
        var collection = await GetCollectionAsync(cancellationToken);
        await collection.DeleteManyAsync(
            Builders<Book>.Filter.Eq(b => b.Type, type),
            cancellationToken
        );
    }
}
```

Now, it's possible to [inject](../../fundamentals/dependency-injection.md) the `IBookRepository` and use the `DeleteBooksByType` method when needed.

#### Override Default Generic Repository

Even if you create a custom repository, you can still inject the default generic repository (`IRepository<Book, Guid>` for this example). Default repository implementation will not use the class you have created.

If you want to replace default repository implementation with your custom repository, do it inside `AddMongoDbContext` options:

```csharp
context.Services.AddMongoDbContext<BookStoreMongoDbContext>(options =>
{
    options.AddDefaultRepositories();
    options.AddRepository<Book, BookRepository>(); //Replaces IRepository<Book, Guid>
});
```

This is especially important when you want to **override a base repository method** to customize it. For instance, you may want to override `DeleteAsync` method to delete an entity in a more efficient way:

```csharp
public async override Task DeleteAsync(
    Guid id,
    bool autoSave = false,
    CancellationToken cancellationToken = default)
{
    //TODO: Custom implementation of the delete method
}
```

### Access to the MongoDB API

In most cases, you want to hide MongoDB APIs behind a repository (this is the main purpose of the repository). However, if you want to access the MongoDB API over the repository, you can use `GetDatabaseAsync()`, `GetCollectionAsync()` or `GetAggregateAsync()` extension methods. Example:

```csharp
public class BookService
{
    private readonly IRepository<Book, Guid> _bookRepository;

    public BookService(IRepository<Book, Guid> bookRepository)
    {
        _bookRepository = bookRepository;
    }

    public async Task FooAsync()
    {
        IMongoDatabase database = await _bookRepository.GetDatabaseAsync();
        IMongoCollection<Book> books = await _bookRepository.GetCollectionAsync();
        IAggregateFluent<Book> bookAggregate = await _bookRepository.GetAggregateAsync();
    }
}
```

> Important: You must reference to the `Volo.Abp.MongoDB` package from the project you want to access to the MongoDB API. This breaks encapsulation, but this is what you want in that case.

### Transactions

MongoDB supports multi-document transactions starting from the version 4.0 and the ABP supports it. However, the [startup template](../../../solution-templates) **disables** transactions by default. If your MongoDB **server** supports transactions, you can enable them in the *YourProjectMongoDbModule* class:

Remove the following code to enable transactions:

```diff
- context.Services.AddAlwaysDisableUnitOfWorkTransaction();
- Configure<AbpUnitOfWorkDefaultOptions>(options =>
- {
- 	options.TransactionBehavior = UnitOfWorkTransactionBehavior.Disabled;
- });
```

#### Setting up a Transaction-Enabled MongoDB Replica Set in Docker

Use the following `docker-compose.yml` to create a local MongoDB Replica Set that supports transactions. The connection string will be `mongodb://localhost:27017/YourProjectName?replicaSet=rs0`.

```yaml
version: "3.8"

services:
  mongo:
    image: mongo:8.0
    command: ["--replSet", "rs0", "--bind_ip_all", "--port", "27017"]
    ports:
      - 27017:27017
    healthcheck:
      test: echo "try { rs.status() } catch (err) { rs.initiate({_id:'rs0',members:[{_id:0,host:'127.0.0.1:27017'}]}) }" | mongosh --port 27017 --quiet
      interval: 5s
      timeout: 30s
      start_period: 0s
      start_interval: 1s
      retries: 30
```

### Advanced Topics

### Controlling the Multi-Tenancy

If your solution is [multi-tenant](../../architecture/multi-tenancy), tenants may have **separate databases**, you have **multiple** `DbContext` classes in your solution and some of your `DbContext` classes should be usable **only from the host side**, it is suggested to add `[IgnoreMultiTenancy]` attribute on your `DbContext` class. In this case, ABP guarantees that the related `DbContext` always uses the host [connection string](../../fundamentals/connection-strings.md), even if you are in a tenant context.

**Example:**

````csharp
[IgnoreMultiTenancy]
public class MyDbContext : AbpMongoDbContext
{
    ...
}
````

Do not use the `[IgnoreMultiTenancy]` attribute if any one of your entities in your `DbContext` can be persisted in a tenant database.

> When you use repositories, ABP already uses the host database for the entities don't implement the `IMultiTenant` interface. So, most of time you don't need to `[IgnoreMultiTenancy]` attribute if you are using the repositories to work with the database.

#### Set Default Repository Classes

Default generic repositories are implemented by `MongoDbRepository` class by default. You can create your own implementation and use it for default repository implementation.

First, define your repository classes like that:

```csharp
public class MyRepositoryBase<TEntity>
    : MongoDbRepository<BookStoreMongoDbContext, TEntity>
    where TEntity : class, IEntity
{
    public MyRepositoryBase(IMongoDbContextProvider<BookStoreMongoDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }
}

public class MyRepositoryBase<TEntity, TKey>
    : MongoDbRepository<BookStoreMongoDbContext, TEntity, TKey>
    where TEntity : class, IEntity<TKey>
{
    public MyRepositoryBase(IMongoDbContextProvider<BookStoreMongoDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }
}
```

First one is for [entities with composite keys](../../architecture/domain-driven-design/entities.md), second one is for entities with single primary key.

It's suggested to inherit from the `MongoDbRepository` class and override methods if needed. Otherwise, you will have to implement all standard repository methods manually.

Now, you can use `SetDefaultRepositoryClasses` option:

```csharp
context.Services.AddMongoDbContext<BookStoreMongoDbContext>(options =>
{
    options.SetDefaultRepositoryClasses(
        typeof(MyRepositoryBase<,>),
        typeof(MyRepositoryBase<>)
    );
    //...
});
```

#### Set Base MongoDbContext Class or Interface for Default Repositories

If your MongoDbContext inherits from another MongoDbContext or implements an interface, you can use that base class or interface as the MongoDbContext for default repositories. Example:

```csharp
public interface IBookStoreMongoDbContext : IAbpMongoDbContext
{
    Collection<Book> Books { get; }
}
```

`IBookStoreMongoDbContext` is implemented by the `BookStoreMongoDbContext` class. Then you can use generic overload of the `AddDefaultRepositories`:

```csharp
context.Services.AddMongoDbContext<BookStoreMongoDbContext>(options =>
{
    options.AddDefaultRepositories<IBookStoreMongoDbContext>();
    //...
});
```

Now, your custom `BookRepository` can also use the `IBookStoreMongoDbContext` interface:

```csharp
public class BookRepository
    : MongoDbRepository<IBookStoreMongoDbContext, Book, Guid>,
      IBookRepository
{
    //...
}
```

One advantage of using interface for a MongoDbContext is then it becomes replaceable by another implementation.

#### Replace Other DbContextes

Once you properly define and use an interface for a MongoDbContext , then any other implementation can use the following ways to replace it:

#### ReplaceDbContext Attribute

```csharp
[ReplaceDbContext(typeof(IBookStoreMongoDbContext))]
public class OtherMongoDbContext : AbpMongoDbContext, IBookStoreMongoDbContext
{
    //...
}
```

#### ReplaceDbContext Option

```csharp
context.Services.AddMongoDbContext<OtherMongoDbContext>(options =>
{
    //...
    options.ReplaceDbContext<IBookStoreMongoDbContext>();
});
```

In this example, `OtherMongoDbContext` implements `IBookStoreMongoDbContext`. This feature allows you to have multiple MongoDbContext (one per module) on development, but single MongoDbContext (implements all interfaces of all MongoDbContexts) on runtime.

#### Replacing with Multi-Tenancy

It is also possible to replace a DbContext based on the [multi-tenancy](../../architecture/multi-tenancy) side. `ReplaceDbContext` attribute and  `ReplaceDbContext` method can get a `MultiTenancySides` option with a default value of `MultiTenancySides.Both`.

**Example:** Replace DbContext only for tenants, using the `ReplaceDbContext` attribute

````csharp
[ReplaceDbContext(typeof(IBookStoreDbContext), MultiTenancySides.Tenant)]
````

**Example:** Replace DbContext only for the host side, using the `ReplaceDbContext` method

````csharp
options.ReplaceDbContext<IBookStoreDbContext>(MultiTenancySides.Host);
````

### Customize Bulk Operations

If you have better logic or using an external library for bulk operations, you can override the logic via implementing `IMongoDbBulkOperationProvider`.

- You may use example template below:

```csharp
public class MyCustomMongoDbBulkOperationProvider
    : IMongoDbBulkOperationProvider, ITransientDependency
{
    public async Task DeleteManyAsync<TEntity>(
        IMongoDbRepository<TEntity> repository,
        IEnumerable<TEntity> entities,
        IClientSessionHandle sessionHandle,
        bool autoSave,
        CancellationToken cancellationToken)
        where TEntity : class, IEntity
    {
        // Your logic here.
    }

    public async Task InsertManyAsync<TEntity>(
        IMongoDbRepository<TEntity> repository,
        IEnumerable<TEntity> entities,
        IClientSessionHandle sessionHandle,
        bool autoSave,
        CancellationToken cancellationToken)
        where TEntity : class, IEntity
    {
        // Your logic here.
    }

    public async Task UpdateManyAsync<TEntity>(
        IMongoDbRepository<TEntity> repository,
        IEnumerable<TEntity> entities,
        IClientSessionHandle sessionHandle,
        bool autoSave,
        CancellationToken cancellationToken)
        where TEntity : class, IEntity
    {
        // Your logic here.
    }
}
```

## See Also

* [Entities](../../architecture/domain-driven-design/entities.md)
* [Repositories](../../architecture/domain-driven-design/repositories.md)
* [Video tutorial](https://abp.io/video-courses/essentials/abp-mongodb)

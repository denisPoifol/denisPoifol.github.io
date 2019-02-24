---
layout: post
title:  "Type Erasure and Protocol with associated type"
date:   2019-02-23 10:35:32 +0100
categories: ["swift"]
---

# Protocol with associated type:

First let's see what Apple says about protocols with associated types:

> When defining a protocol, it’s sometimes useful to declare one or more associated types as part of the protocol’s definition. An associated type gives a placeholder name to a type that is used as part of the protocol. The actual type to use for that associated type isn’t specified until the protocol is adopted. Associated types are specified with the associatedtype keyword.

This seems quite simple and pretty usefull, let's say I want to define protocol that defines identifiable objects.

~~~swift
protocol Identifiable {
    associatedType Identifier
    var id: Identifier { get }
}
~~~

Let's write some struct conforming to this protocol to see how hard it can be to use.

~~~swift
struct Car: Identifiable {
    let plate: ImmatriculationPlate
    var id: ImmatriculationPlate {
        return plate
    }
}
~~~

Pretty simple and the compiler even infers the actual **Identifier** type for us, but in the case it cannot do it we can always spell it out for it: `typealias Identifier = ImmatriculationPlate`

# The concrete example:

If we are being totally honest the **Identifiable** protocol is not doing much for us. Let's take a look at an actual use case for protocol with associated type. I was recently working on an application that needed to display a catalog of products, so it was pretty classic: my ViewController fetched the products from a repository and displayed it.

~~~swift
class CatalogViewController: UIViewController {

    private let CatalogFetcher: CatalogFetcher
    
    init(catalogFetcher: CatalogFetcher) {
        self.catalogFetcher = catalogFetcher
    }

    override func viewWillAppear(_ animated: Bool)
        fetch()
    }
    
    func productSelected(id: String) {
        // Push detail viewController
    }
    
    // MARK: - Private methods
    
    private func fetch() {
        catalogFetcher.fetch { [weak self] result in
            switch result {
            case let .error(error):
                // Display error message
            case let .value(catalog):
                // Display catalog
            }
        }
    }
}
~~~

But after a while there were three different kind of product A, B and C. Each kind of product had its own characteristics and so was described in my app by a Struct.

So lets make this viewController generic, and for that we need our **CatalogFetcher** protocol to have an associated type and in that context for that type to be our generic product type.

# The generic implementation:

Now that we have our solution we juste have to implement it:
This seems pretty simple `CatalogViewController` becomes `CatalogViewController<Catalog>` and everything falls into place, we just have to make **CatalogFetcher** a PAT (Protocol with Associated Type) and we are all set.

~~~swift
protocol CataLogFetcher {
    associatedtype Catalog
    func fetch(completion: ((Result<Catalog>) -> Void)?)
}
~~~

But now `private let catalogFetcher: CatalogFetcher` cause the error: ***Protocol 'CatalogFetcher' can only be used as a generic constraint because it has Self or associated type requirements***

This was too good and too simple to be true, let's dive in and find out what this is all about.

# Protocol '***' can only be used as a generic constraint because it has Self or associated type requirements:

Swift is a strongly typed language which means every variable should have a type and this type has to be completely defined. Therefore **CatalogFetcher** has no meaning by itself to the compiler, indeed to make sense of a type the compiler should be able to know what variable and function can be called from that variable. In this case the compiler needs to know the actual type of **CatalogFetcher.Catalog**.

# Workarounds for a **CatalogFetcher variable**

In order to have such a variable we have to create a type whose sole purpose is to present the **CatalogFetcher** API.

## Generic wrapper

The easiest way to create such a type is to create a generic wrapper that only display the wanted API.

~~~swift
struct CatalogFetcherWrapper<ImplementingType: CatalogFetcher>: CatalogFetcher {
    private let wrappedInstance: ImplementingType
    
    init(wrappedInstance: ImplementingType) {
        self.wrappedInstance = wrappedInstance
    }
    
    // MARK - CatalogFetcher
    
    func fetch(completion: ((ImplementingType.Catalog) -> Void)?) {
        wrappedInstance.execute(completion: completion)
    }
}
~~~

This way we can wrap any kind of type **ImplementingType** that implement **CatalogFetcher** inside a **CatalogFetcherWrapper<ImplementingType>** that only present **CatalogFetcher** API.

Unfortunately this imply that our variable has to be: 
`let catalogFetcher: CatalogFetcherWrapper<ImplementingType>`
which means for our ViewController to be able to use any implementation of **catalogFetcher** we need to add another generic type parameter to our protocol:

~~~swift
class CatalogViewController<Catalog, CatalogFetcherImplementation: CatalogFetcher>: UIViewController
    where CatalogFetcherImplementation.Catalog == Catalog {
    
    private let catalogFetcher: CatalogFetcherWrapper<CatalogFetcherImplementation>
    
    init(catalogFetcher: CatalogFetcherWrapper<CatalogFetcherImplementation>) {
        self.catalogFetcher = catalogFetcher
    }
}
~~~ 

This actually suits our needs but it is not quite the best way to do it: 
because even though the wrapper does not enable to do anything else than what the API of the **CatalogFetcher** provides, it does not feel right that our viewController knows what is the underlying implementing type, and its awful to understand on the first read.

## Type Erasure

In the wrapper solution the implementing type enables the compiler to infer what kind of catalog we are manipulating.

But now let's try to hide that implementing type ; keep it private and only show the Catalog type used. This pattern is called type erasure since we erase the implementing type to only display the associated type.

In order to avoid mixing thing up here are the definitions of the terms we are going to use:

- **AssociatedType:** 
the placeholder of the protocol which represent a type associated to the protocol
- **ImplementingType:**
the type of the object implementing the protocol
- **ConcreteType:** 
the type associated to the ImplementingType

In order to achieve type erasing we are going to use 3 layers of boilerplate:

- **AbstractBase:**
The abstract base is a generic abstract class that implements protocol and where the associated type of the protocol is its generic parameter type
- **Private Box:**
The private box is the same generic wrapper described in the previous section, but the important difference is that it inherits from the AbstractBase which is the reason we can erase the implementing type using the polymorphism of this object.
- **Public Wrapper:**
The public wrapper is the class were going to use once all is setup, its job is mainly to present a clean interface and be an opaque wrapper for the PrivateBox

Theses classes and their initialisation methods are summed up in the below figure:

![Type Erasure pattern's boiler plate layers](/Assets/TypeErasure.png)

Type Erasure pattern's boiler plate layers

In order to make this as clear as possible lets look at the implementation of our boilerplate classes in our concrete example:

- **Abstract Base**

    Since generic abstract classes does not exist per say in swift we are forced to manufacture one, making init unavailable and providing crashing implementation of the methods that have to be overridden by the subclasses

    ~~~swift
class _AnyGetCatalogFetcherBase<ConcreteType>: CatalogFetcher {
        
    @available(*, unavailable)
    init() {}
        
    func fetch(completion: ((Result<ConcreteType>) -> Void)?) {
        fatalError("Method has to be overriden this is an abstract class")
    }
}
    ~~~

- **Private Box**

    The private box take advantage of inheritance and in order to be able to be seen either as a generic class whose type parameter is the implementing type, or one where the type parameter is the concrete type associated to it.

    ~~~swift
class _AnyCatalogFetcherBox<ImplementingType: CatalogFetcher>: 
    _AnyCatalogFetcherBase<ImplementingType.Catalog> {
        
    private let wrappedInstance: ImplementingType
        
    init(_ fetcher: ImplementingType) {
        wrappedInstance = fetcher
    }
        
    override func fetch(completion: ((Result<ImplementingType.Catalog>) -> Void)?) {
        wrappedInstance.fetch(completion: completion)
    }
}
    ~~~

- **Public Wrapper**

    As mentioned before in the wrapper class we only wrap the box and use polymorphism to see it as a `_AnyCatalogFetcherBase` which is a generic of the concrete type instead of the implementing type

    ~~~swift
final class AnyCatalogFetcher<ConcreteType>: CatalogFetcher {
        
    private let box: _AnyCatalogFetcherBase<ConcreteType>
        
    init<ImplementingType: CatalogFetcher>(_ fetcher: ImplementingType)
        where ImplementingType.Catalog == ConcreteType {
            box = _AnyCatalogFetcherBox(fetcher)
    }
        
    func fetch(completion: ((Result<ConcreteType>) -> Void)?) {
        box.fetch(completion: completion)
    }
}
    ~~~

These 3 classes can be declared in the same file so that the first 2 class can be declared private so that we only expose the final **AnyCatalogFetcher**.

Let's see what our viewController looks like with a type erased wrapper.

~~~swift
class CatalogViewController<Catalog>: UIViewController {
    
    private let catalogFetcher: AnyCatalogFetcher<Catalog>
    
    init(catalogFetcher: AnyCatalogFetcher<Catalog>) {
        self.catalogFetcher = catalogFetcher
    }
    
    [...]
}
~~~

We now can enjoy our generic viewController that will be able to handle any type of Catalog

# TLDR:

Type erasure is a great way to palliate the lack of existential type for Protocol with associated types. In a perfect world, and maybe in a close future, the compiler could figure out all this boilerplate for us. But currently this has to be done by hand.


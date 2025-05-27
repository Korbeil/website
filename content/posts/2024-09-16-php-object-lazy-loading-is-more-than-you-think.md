+++
title = "✍️ PHP Object Lazy-Loading is More Than What You Think"
date = "2023-12-05"
tags = ["php", "symfony", "doctrine"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/php-object-lazy-loading-is-more-than-what-you-think).

We recently attended [a talk about lazy-loading](https://x.com/AFUP_Paris/status/1801306022994137460) by [Nicolas Grekas](https://github.com/nicolas-grekas/) and it inspired me this blogpost!

We can find lazy-loading in all modern PHP applications, in ORMs, for example. **But is there more usage of lazy-loading?**

## What is lazy-loading?

In short: lazy-loading consists of delaying load or initialization of resources or objects until they're actually needed. It's something you will never see directly, the whole objective of lazy-loading is to be invisible so you can use your applications the way you always do.

And behind this invisible thing, we will give you *something* that will act as the object you want but will only load its content when required.

We can see two main pros about using lazy-loading:

- It will save memory and CPU until we load the data
- It may never be called since you sometimes load nested objects that you don't need...

What is lazy-loading? [Martin Fowler](https://martinfowler.com/), in his book "[Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)", explains this pattern very well. There are 4 types of lazy-loading patterns. Each of them have pros &amp; cons:

- **Lazy initialization**, your object contains a null property and whenever you need that property, it will initialize it. In that case, your object needs to be aware that it is lazy-loaded;
- **Value holders**, it's an object that is responsible for initializing another object whenever you need it. It will require you to write a different class but the base class will stays the same;
- **Virtual proxy** is an object that is using the same interface (methods) that is used in the base object and whenever you use one of those methods, it will instantiate the base object and delegate to it;
- **Ghost objects** are objects that are to be loaded in a partial state. It may initially only contain the object's identifier, but it loads its own data the first time one of its properties or methods are accessed.

In PHP we have some libraries that implement this pattern such as [ProxyManager](https://github.com/Ocramius/ProxyManager), [Proxy Manager LTS](https://github.com/FriendsOfPHP/proxy-manager-lts) or even [symfony/var-exporter](https://github.com/symfony/var-exporter). I do personally recommend using the latter when possible.

One of the biggest drawbacks of Ghost objects is that you need to create a class that extends the proxified class. It can block you if you want to proxify a class that is final but will work in most cases and none of the consumers or the proxified class has to know there is a ghost object there.
More recently, we saw a [Lazy Object RFC](https://wiki.php.net/rfc/lazy-objects) done by Nicolas Grekas &amp; [Arnaud Le Blanc](https://github.com/arnaud-lb), merged from the PHP team! Adding this feature to the core means we no longer need to create a proxy class, which is currently almost always required for lazy-loading.

Now that we've seen what lazy-loading is and what libraries can leverage it in PHP, let's see how it is used in well known PHP libraries.

## Symfony DependencyInjection

In Symfony, you can define a service as needing to be lazy-loaded in multiple ways.
You can either define a service as being lazy-loaded or you can define one of the dependencies of a service as being lazy-loaded.

> For instance, imagine you have a `NewsletterManager` and you inject a `mailer` service into it. Only a few methods on your `NewsletterManager` actually use the `mailer`, but even when you don't need it, `mailer` service is always instantiated in order to construct your `NewsletterManager`.
>
> Configuring lazy services is one answer to this. With a lazy service, a "proxy" of the `mailer` service is actually injected. It looks and acts like the `mailer`, except that the `mailer` isn't actually instantiated until you interact with the proxy in some way.

To put it with examples, considering you have the following service:

```php
readonly class NewsletterManager
{
  public function __construct(
	private MailerInterface $mailer,
  ) {
  }
}
```

When we use this service, we could have methods that don't use the `$mailer` property. That's why we don't want to load it directly and we use lazy-loading. When you use the external mailer service, the mailer will be initialized and returned.

Based on the 4 types of lazy-loading we described, Symfony doesn't use a single type but adapts based on what it requires. You can find more about lazy-loading service in Symfony in the [related documentation page](https://symfony.com/doc/current/service_container/lazy_services.html).

## Doctrine

When you use Doctrine, you'll sometimes have a lazy-loaded object so that it can trigger a database query only when the object is accessed. By making a direct request to an object, Doctrine will return the hydrated object, but sometimes our objects have relationships with other objects, so these relationships will be lazy-loaded. By default, the relationships are lazy-loaded (except for OneToOne relations),  a proxy object will be returned instead of the real object, which will be loaded only if this object is accessed, which will also trigger a request to the database. You can avoid this behavior by using the `fetch` parameter [with the value `EAGER`](https://www.doctrine-project.org/projects/doctrine-orm/en/3.2/tutorials/extra-lazy-associations.html), you'll get a real object directly.

In practice, you will have the following code for your entity:

```php
#[ORM\Entity]
#[ORM\Table(name: 'customer')]
class User
{
  // some columns …

  #[ManyToOne(targetEntity: Address::class, nullable: true)]
  private ?Address $address = null;
}
```

And whenever you request an User, the Address object will be proxified. Doctrine will generate the following proxy:

```php
class Address extends \App\Entity\Address implements \Doctrine\ORM\Proxy\InternalProxy
{
  use \Symfony\Component\VarExporter\LazyGhostTrait {
    initializeLazyObject as __load;
    setLazyObjectAsInitialized as public __setInitialized;
    isLazyObjectInitialized as private;
    createLazyGhost as private;
    resetLazyObject as private;
  }

  private const LAZY_OBJECT_PROPERTY_SCOPES = [
    "\0".parent::class."\0".'id' => [parent::class, 'id', null],
    "\0".parent::class."\0".'name' => [parent::class, 'name', null],
    // ... other fields
  ];

  // ... other methods
}
```

When you’re making a database query with Doctrine, the relations of the object you’re fetching will (depending on configuration) be proxified. Since those proxies extend your entity class, you will be able to use them the exact same way you usually use your entity.

Something different from Symfony's DependencyInjection here is that each Doctrine proxy will contain an identifier (usually a numeric id) that will allow Doctrine to link a proxy instance to a row for the given entity in the database. We call that a LazyGhost.

## And in your projects?

Now that we've seen the two most common uses of lazy-loading, how could you use this in your applications?

One of the best use cases I stumbled over upon my 13 years of development is **curl requests**. Not the simple curl request that you do to an external service, I mean thousands of curl requests that you probably will require at the same time… or maybe not.

### The base issue

curl is one of those projects we can't ignore in my developer’s life. There aren't many of them but curl is clearly in that list. And after using curl all this time, we found some strange issues, 5 years ago, when trying to make some HTTP calls to an API.

First of all, why do we have to make thousands of requests to an API? we are working for an e-commerce company that has multiple services, and specifically one that is a [PIM](https://en.wikipedia.org/wiki/Product_information_management). That API is a product catalog. When we show a product page, we reach that API for the specific product we're showing in order to gather data about this product. Also, when we show a collection page, we have multiple products so we have to reach that API for each product (and other stuff like product categories).

And the final encounter, exports. That company is often doing exports of all products that are currently in sale and thus making thousands of requests to that API.

In order to make all of this work, we could just wait for each HTTP call to be done on the pages or exports. But this could mean waiting for hundreds of calls to finish. At the moment our product detail call is between 200 to 300 ms, so for an export of 2000 products it would require 10min to wait **just** for API calls to be done, not counting deserialization of data because I’m using DTOs. For a collection page, we can have up to 200/300 products in the same page for the biggest ones which would take around 1 min for API calls only.

### Linking the PIM

In order to solve this issue, we have to create a link with that PIM API so we can lazy-load its data. As we said before, Doctrine is often using proxies so why not using an entity to make that link? The idea here is that when we load an entity, one of the properties will contain a proxy of how the API response should be, and once you access it we will trigger the API call to return the API model.

We will need something to identify what is the entity that contains the proxy object:

```php
#[ORM\Entity]
#[ORM\Table(name: 'product')]
class Product
{
  #[ORM\Column]
  public string $pimUuid;

  public string $pimObject;

  public function setPimUuid(string $pimUuid): void
  {
    $this->pimUuid = $pimUuid;
  }

  public function getPimUuid(): string
  {
    return $this->pimUuid;
  }

  public function setPimObject(object $pimObject): void
  {
    $this->pimObject = $pimObject;
  }

  public function getPimObject(): object;
  {
    return $this->pimObject;
  }
}
```

In addition to that entity, we need to listen to the loading of the Product entity, which can be done with a Doctrine listener. That listener will handle the proxy generation and insert the proxy into the pimObject property so we can trigger the curl call when it is required:

```php
#[AsEntityListener(event: Events::postLoad)]
class PimListener
{
  public function postLoad(PostLoadEventArgs $args): void
  {
    /** @var object|PimInterface $entity */
    $entity = $args->getObject();

    if (!$entity instanceof Product) {
      return;
    }

    /** @var string $uuid */
    $uuid = $entity->getPimUuid();
    $entity->setPimObject($this->fetchPim($uuid));
  }

  private function fetchPim(string $uuid): object
  {
    $initializer = function (
      GhostObjectInterface $ghostObject,
      string $method,
      array $parameters,
      &$initializer,
      array $properties
    ) use ($uuid) {
      $initializer = null; // disable initialization
      // Checking if we got a result from PIM API
      if (!$ghostObject instanceof ProductDTO) {
        return false;
      }

      // do the actual curl call
      $pimProduct = $this->pimClient->getVariantItem($uuid);

      /**
        * For all object properties, we take property name then we guess the mutator
        * method and we use it for each property in order to hydrate the object
        */
      foreach (\array_keys($properties) as $property) {
        $parts = explode("\x00", $property);
        /** @var string $variable */
        $variable = \array_pop($parts);
        $mutator = sprintf('get%s', \ucfirst($variable));
        $properties[$property] = \call_user_func([$pimProduct, $mutator]);
      }

      return true;
    };

    return $this->factory->createProxy(ProductDTO::class, $initializer);
  }
}
```

With this implementation, whenever we get a Product entity from the database, we inject a GhostProxy into that entity. If we access a property of this object, we trigger the curl call. From this point, we could easily have PIM data within other parts of our application with ease, thanks to that hidden curl call, but that was only the beginning of issues.

![Flow no async](/media/original/2024/no_async.png)

<center>Diag. for a product page</center>

For a simple product page, this would make the job easy but what about a showcase of multiple products or even exporting the whole product list? We would have to wait for the curl call to be made one by one … As explained before, we don’t want that.

### Overcoming curl limits

With this implementation we could see some strange issues: from time to time, some curl requests towards the PIM API were dropping for like, no reason!? We did many investigations but could never reproduce the exact issue. After some time, we could tell that the issues would always happens when we had more than 400/500 curl calls to do in a row and stumbled upon [this issue within Symfony](https://github.com/symfony/symfony/pull/38690).

When you're using `HttpClient`in Symfony, you will, by default, use the `CurlHttpClient` which uses `curl_multi` under the hood to launch HTTP calls. Thanks to `curl_multi`, we have something close to asynchronous HTTP calls because we can trigger a HTTP call and use it later, but in the meantime `curl_multi` could already have received the result?

Since we knew what the issue was (number of concurrent requests when using `curl_multi`), we did the simplest change to fix this issue: batching. The idea was that instead of getting the Products one by one, we would use the product list endpoint onto the PIM API. That way we could get Product 25 per 25. From 500 curl calls, we dropped to 20 calls.

But we still had to **wait** for 20 calls to be made before we could start building our page. One product list call takes around ~120ms, but 20 times that would be 2.4s. And that's only in the optimistic case where we only need products.

That's why we combined the strengths of lazy-loading with the *almost async* Symfony CurlHttpClient to make the "preload" feature. Think about HTTP preload: we will start fetching data before it's even used and when we access that data, it will probably already be there, or it will have started to call for it. That way we have the minimum possible time to wait for PIM data!

Let's see it with some code:

```php
final class Result
{
  private bool $initialized;

  public function __construct(
    private ResponseInterface $response,
    private string $modelClass, // DTO FQCN
    private SerializerInterface $serializer,
  ) {
    $this->initialized = false;
  }

  public function isInitialized(): bool
  {
    return $this->initialized;
  }

  public function getStatusCode(): int
  {
    $this->initialized = true;
    return $this->response->getStatusCode();
  }

  public function toObject(): ?object
  {
    return $this->serializer->deserialize($this->getContents(), $this->modelClass, 'json');
  }

  public function getContents(): string
  {
    $this->initialized = true;

    return $this->response->getContent();
  }
}
```

In the PimListener:

```php
public function preload(array $entities): void
{
  foreach ($entities as $entity) {
    // some configuration pass based on the entity ...
    [$modelClass, $endpointClass] = $entity->getPimModelConfiguration();

    if (\array_key_exists($entity->getPimUuid(), $this->arrayResults[$modelClass])) {
      continue; // Do not fetch data we already have in current transaction
    }

    $endpoint = new $endpointClass($entity->getPimUuid());
    $response = $this->makeRequest($endpoint);
    $this->arrayResults[$modelClass][$entity->getPimUuid()] = new Result($response, $modelClass, $this->serializer);
  }
}
```

With this Result class and the preload method, we can load data before we need it. So now in my page controller, the first line is a query that gets all the products we will use during the rendering part. And we send that product array into the preload method. That way we start the curl queries and later on when we try to, for example, get the image of a product, it will be instantaneous!

![Flow with async](/media/original/2024/with_async.png)

<center>Diag. for a product page</center>

Full `PimListener` code is available on this gist: https://gist.github.com/Korbeil/7451afbff73da572f0500375d6f74805

## Conclusion

Now that you know what lazy-loading is and have an idea on how you can use it, we can't wait to see all the new things the PHP community will find to do with this awesome tool. We wanted to make this post because we saw many comments about the recent RFC being too "niche" and "not something we need". Now we can all see how lazy-loading could help us and make our projects more performant!

## And with that new RFC?

Now that [Lazy Object RFC](https://wiki.php.net/rfc/lazy-objects) was accepted, this code will not be the same anymore. Here is a quick peek on how it should looks based on the simple PimListener we made before:

```php
#[AsEntityListener(event: Events::postLoad)]
class PimListener
{
  public function postLoad(PostLoadEventArgs $args): void
  {
    /** @var object|PimInterface $entity */
    $entity = $args->getObject();

    if (!$entity instanceof Product) {
      return;
    }

    /** @var string $uuid */
    $uuid = $entity->getPimUuid();
    $entity->setPimObject($this->fetchPim($uuid));
  }

  private function fetchPim(string $uuid): object
  {
    $reflector = new ReflectionClass(ProductDTO::class);
    $initializer = static function (ProductDTO $product) use ($reflector): void {
      $pimProduct = $this->pimClient->getVariantItem($product->getUuid());
      foreach ($reflector->getProperties() as $property) {
        $mutator = sprintf('get%s', \ucfirst($property->getName()));
        $reflector->getProperty($property)->setValue($post, \call_user_func([$pimProduct, $mutator]));
      }
    };

    $product = $reflector->newLazyGhost($initializer);
    $reflector->getProperty('uuid')->setRawValueWithoutLazyInitialization($product, $uuid);

    return $product;
  }
}
```

See how simple the implementation is with the RFC! And on top of that, you won’t need any third-party library to do that, it will be embedded into the PHP core! 🎉

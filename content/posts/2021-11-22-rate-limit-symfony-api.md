+++
title = "Rate limit your Symfony APIs!"
date = "2021-11-22"
tags = ["php", "symfony", "rate-limiter"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/rate-limit-your-symfony-apis).

Sometimes, you need to put some custom rate limits on your APIs! In this article I'll show you how you can combine the `symfony/rate-limiter` component and some usual controllers.

### RateLimit configuration

The goal here is to have the following rate limit configuration works thanks to PHP8 attributes on any route you want:

```yaml
framework:
  rate_limiter:
    account_create:
      policy: 'fixed_window'
      limit: 5
      interval: '60 minutes'
    account_modify: # account activate, change profile
      policy: 'fixed_window'
      limit: 30
      interval: '60 minutes'
```

This article isn't about the component itself so I'll recommend you to read the [Symfony's RateLimiter documentation](https://symfony.com/doc/current/rate_limiter.html) if you want to understand how it works and how to create rules.

### Attribute

First of all, we need an attribute that we will use to declare routes that need to be rate limited. We will require a configuration key to identify which rate limit configuration we should take:

```php
#[Attribute(Attribute::TARGET_METHOD)]
class RateLimiting
{
  public function __construct(
    public string $configuration,
  ) {
  }
}
```

### Controller

Now let's use our attribute on some controller:

```php
#[RateLimiting('account_create')]
#[Route('/create', methods: ['POST'])]
public function createAccount(): JsonResponse
{
  // your controller logic ...
}
```

And that's all you need to do to declare a route as rate limited 👌

### CompilerPass

But before it works we need to make Symfony understand these attributes. So we need a CompilerPass to store all routes that have our attribute to avoid reflection at runtime:

```php
class RateLimitingPass implements CompilerPassInterface
{
  public function process(ContainerBuilder $container): void
  {
    if (!$container->hasDefinition(ApplyRateLimitingListener::class)) {
      throw new \LogicException(sprintf('Can not configure non-existent service %s', ApplyRateLimitingListener::class));
    }

    $taggedServices = $container->findTaggedServiceIds('controller.service_arguments');
    /** @var Definition[] $serviceDefinitions */
    $serviceDefinitions = array_map(fn (string $id) => $container->getDefinition($id), array_keys($taggedServices));

    $rateLimiterClassMap = [];

    foreach ($serviceDefinitions as $serviceDefinition) {
      $controllerClass = $serviceDefinition->getClass();
      $reflClass = $container->getReflectionClass($controllerClass);

      foreach ($reflClass->getMethods(\ReflectionMethod::IS_PUBLIC | ~\ReflectionMethod::IS_STATIC) as $reflMethod) {
        $attributes = $reflMethod->getAttributes(RateLimiting::class);
        if (\count($attributes) > 0) {
          [$attribute] = $attributes;

          $serviceKey = sprintf('limiter.%s', $attribute->newInstance()->configuration);
          if (!$container->hasDefinition($serviceKey)) {
            throw new \RuntimeException(sprintf(‘Service %s not found’, $serviceKey));
          }

          $classMapKey = sprintf('%s::%s', $serviceDefinition->getClass(), $reflMethod->getName());
          $rateLimiterClassMap[$classMapKey] = $container->getDefinition($serviceKey);
        }
      }
    }

    $container->getDefinition(ApplyRateLimitingListener::class)->setArgument('$rateLimiterClassMap', $rateLimiterClassMap);
  }
}
```

Here we get all controllers and we check on each method if they have our attribute and then we link the route to the corresponding rate limit service and add it in our cache.

### Listener

Now that Symfony understands our attribute and cache it, we need an event listener to hook on the `kernel.controller` event and check if our rate limit is fine or not.

```php
class ApplyRateLimitingListener implements EventSubscriberInterface
{
  public function __construct(
    private TokenStorageInterface $tokenStorage,
    /** @var RateLimiterFactory[] */
    private array $rateLimiterClassMap,
    private bool $isRateLimiterEnabled,
    private RequestStack $requestStack,
    private RoleHierarchyInterface $roleHierarchy,
  ) {
  }

  public function onKernelController(KernelEvent $event): void
  {
    if (!$this->isRateLimiterEnabled || !$event->isMainRequest()) {
      return;
    }

    $request = $event->getRequest();
    /** @var string $controllerClass */
    $controllerClass = $request->attributes->get('_controller');

    $rateLimiter = $this->rateLimiterClassMap[$controllerClass] ?? null;
    if (null === $rateLimiter) {
      return; // no rate limit service was assigned for this controller
    }

    $token = $this->tokenStorage->getToken();
    if ($token instanceof TokenInterface && in_array('ROLE_GLOBAL_MODERATOR', $this->roleHierarchy->getReachableRoleNames(($token->getRoleNames())))) {
      return; // we ignore rate limit for site moderator & upper roles
    }
    
    $this->ensureRateLimiting($request, $rateLimiter, $request->getClientIp());
  }

  private function ensureRateLimiting(Request $request, RateLimiterFactory $rateLimiter, string $clientIp): void
  {
    $limit = $rateLimiter->create(sprintf('rate_limit_ip_%s', $clientIp))->consume();
    $request->attributes->set('rate_limit', $limit);
    $limit->ensureAccepted();

    $user = $this->tokenStorage->getToken()?->getUser();
    if ($user instanceof User) {
      $limit = $rateLimiter->create(sprintf('rate_limit_user_%s', $user->getId()))->consume();
      $request->attributes->set('rate_limit', $limit);
      $limit->ensureAccepted();
    }
  }

  public static function getSubscribedEvents(): array
  {
    return [KernelEvents::CONTROLLER => ['onKernelController', 1024]];
  }
}
```

In this example, I chose to ignore rate limits for our global moderator roles. For all other users I check the rate limit on two levels: IP then User if they are logged. That way we can avoid any user spamming from different IPs. These are business rules I use but you can custom it the way you want.

Also you can see that we share the rate limit service before each check: if there is a rate limit issue then an exception will be thrown (thanks to the `ensureAccepted` method) and the second check won't happen so we will have the correct rate limit service shared.

### Headers

Finally, to use that shared rate limit service, we can generate some headers to indicate how the rate limit went and other metrics:

```php
final class RateLimitingResponseHeadersListener
{
  public function onKernelResponse(ResponseEvent $event): void
  {
    if (($rateLimit = $event->getRequest()->attributes->get('rate_limit')) instanceof RateLimit) {
      $event->getResponse()->headers->add([
        'RateLimit-Remaining' => $rateLimit->getRemainingTokens(),
        'RateLimit-Reset' => time() - $rateLimit->getRetryAfter()->getTimestamp(),
        'RateLimit-Limit' => $rateLimit->getLimit(),
      ]);
    }
  }
}
```

I took the headers names from the [RateLimit headers RFC](https://tools.ietf.org/id/draft-polli-ratelimit-headers-00.html), it's still a draft but theses headers are already widely used.

And here we are - with only a few lines of code, you can add a rate limit to any route by adding your new `RateLimiting` attribute!

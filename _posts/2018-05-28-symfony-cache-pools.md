---
layout: post
title:  "How to make your custom cache pool"
description: "Creating and releasing cache pool with Symfony"
date:   2018-05-28 21:00:00 +0200
categories: symfony cache open-source
---

# How to create clean Symfony cache pool

There is one thing I never found in [the official symfony/cache documentation](https://symfony.com/doc/current/components/cache/cache_pools.html): how to create my own custom cache pool for my bundle (or my application).

I even created my own cache implementation (based on `Symfony\Component\Cache\Adapter\FilesystemAdapter`) !

But thanks to [@stof](https://github.com/stof) who made me discover [the following code sample](https://github.com/Incenteev/hashed-asset-bundle/blob/master/src/Resources/config/cache.xml#L35).
I saw how to make theses cache pool and wanted to share it !

So with the dependencyInjection:
## YAML

```yaml
services:
  my_bundle.cache:
    parent: cache.system
    public: false
    tags:
      - { name: cache.pool }
```

## PHP

```php
use Symfony\Component\DependencyInjection\ChildDefinition;

$definition = new ChildDefinition('cache.system');
$definition->setPublic(false);
$definition->addTag('cache.pool');
$container->setDefinition('my_bundle.pool', $definition);
```

## Usage

After that you just have to use usual service injection.
It will returns you a [PSR-6 compatible](https://www.php-fig.org/psr/psr-6/) object.

## Clearing pool

By the way, you can also clear your pool with both:
```bash
$ php bin/console cache:clear
```
This one will clear all Symfony cache

```bash
$ php bin/console cache:pool:clear my_bundle.cache
```
And this one will clear only pool cache
+++
title = "✍️ Maîtrisez la planification des tâches avec Symfony Scheduler"
date = "2023-12-05"
tags = ["php", "symfony", "cron"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/maitrisez-la-planification-des-taches-avec-symfony-scheduler).

Aujourd’hui, utiliser une crontab pour nos tâches récurrentes est assez courant mais pas très pratique car complètement déconnecté de notre application. Le composant Scheduler se présente comme une excellente alternative. Il a été introduit en 6.3 par Fabien Potencier lors de sa keynote d’ouverture du [SymfonyLive Paris 2023](/blog/notre-retour-sur-le-symfonylive-paris-2023). Le composant est maintenant réputé comme stable depuis la sortie de Symfony 6.4.
Regardons comment l’utiliser !

## Installation

Installons le composant :

```shell
composer require symfony/messenger symfony/scheduler
```

Comme toutes les fonctionnalités du composant se basent sur Messenger, il est nécessaire de l’installer aussi.

## Une première tâche

Créons un premier message à planifier :

```php
// src/Message/Foo.php
readonly final class Foo {}

// src/Handler/FooHandler.php
#[AsMessageHandler]
readonly final class FooHandler
{
    public function __invoke(Foo $foo): void
    {
        sleep(5);
    }
}
```

De la même manière qu'un Message dispatché dans Messenger, nous dispatchons ici un Message, que Scheduler traitera de façon similaire à Messenger, excepté que le déclenchement du traitement se fera sur une base temporelle

En plus du couple Message/Handler, nous avons besoin de définir un "Schedule" :

```php
#[AsSchedule(name: 'default')]
class Scheduler implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return (new Schedule())->add(
            RecurringMessage::every('2 days', new Foo())
        );
    }
}
```

Il va permettre d’indiquer à notre application que nous avons un Schedule "default" qui contient un message lancé tous les deux jours. Ici, la fréquence est simple, mais il est tout à fait possible de configurer cela plus finement :

```php
RecurringMessage::every('1 second', $msg)
RecurringMessage::every('15 day', $msg)

# format relatif
RecurringMessage::every('next friday', $msg)
RecurringMessage::every('first sunday of next month', $msg)

# se lance à un horaire spécifique tous les jours
RecurringMessage::every('1 day', $msg, from: '14:42')
# vous pouvez donner un objet DateTime aussi
RecurringMessage::every('1 day', $msg,
    from: new \DateTimeImmutable('14:42', new \DateTimeZone('Europe/Paris'))
)

# définir la fin de la récurrence
RecurringMessage::every('1 day', $msg, until: '2023-09-21')

# vous pouvez aussi utiliser des expressions cron
RecurringMessage::cron('42 14 * * 2', $msg) // every Tuesday at 14:42
RecurringMessage::cron('#midnight', $msg)
RecurringMessage::cron('#weekly', $msg)
```

Ici, nous pouvons voir des formats relatifs; vous trouverez plus d’informations sur ce format en PHP sur la [page de documentation](https://www.php.net/manual/fr/datetime.formats.php#datetime.formats.relative).

Pour les syntaxes `cron`, il vous faudra installer une librairie tierce qui permet à Scheduler de les interpréter :

```shell
composer require dragonmantank/cron-expression
```

Une fois votre Schedule défini, comme pour un transport Messenger, il vous faudra un worker qui va écouter sur le Schedule de la façon suivante:

```shell
bin/console messenger:consume -v scheduler_default
```

Le préfix `scheduler_` est le nom générique du transport pour tous les Schedule, auquel nous ajoutons le nom du Schedule créé.

## Les collisions

Plus nous avons de tâches, plus nous avons de chances d’avoir des tâches qui vont arriver au même moment. Mais si une collision arrive, comment Scheduler va-t-il gérer ça ? Imaginons le cas suivant :

```php
(new Schedule())->add(
    RecurringMessage::every('2 days', new Foo()),
    RecurringMessage::every('3 days', new Foo())
);
```

Tous les 6 jours, les deux messages vont entrer en collision :

![Schedule collision](/media/original/2023/articles/scheduler/image_1_fr.png)

Si nous avons qu'un seul worker, alors il prendra la première tâche configurée dans le Schedule **puis**, une fois la première tâche finie, il exécutera la seconde tâche. Autrement dit, l'heure d'exécution de la 2ème tâche est dépendante de la durée d'exécution de la 1ère.

Souvent nous voulons que nos tâches soient exécutées à un moment précis, pour régler ce soucis il existe deux solutions:

- La bonne pratique serait de préciser la date et heure d'exécution de notre tâche grâce au paramètre `from`, par exemple: `RecurringMessage::every('1 day', $msg, from: '14:42')` pour un des messages et fixer à `15:42` pour l’autre tâche (aussi possible avec une syntaxe `cron`) ;
- Avoir plusieurs workers qui tournent: si vous avez 2 workers, alors il pourra gérer 2 tâches en même temps !

## Plusieurs workers ?

Mais aujourd'hui, si nous lançons 2 workers, notre tâche sera exécutée deux fois !

![Schedule workers](/media/original/2023/articles/scheduler/image_2_fr.png)

Scheduler fournit les outils pour éviter ça ! Mettons un peu à jour notre Schedule :

```php
#[AsSchedule(name: 'default')]
class Scheduler implements ScheduleProviderInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly LockFactory $lockFactory,
    ) {
    }

    public function getSchedule(): Schedule
    {
        return (new Schedule())
            ->add(RecurringMessage::every('2 days', new Foo(), from: '04:05'))
            ->add(RecurringMessage::cron('15 4 */3 * *', new Foo()))
            ->stateful($this->cache)
            ->lock($this->lockFactory->createLock('scheduler-default'))
        ;
    }
}
```

Nous récupèrons un service pour gérer son cache et pour créer des locks (penser à installer `symfony/lock` auparavant). Puis nous indiquons que notre schedule peut maintenant bénéficier d’un état et possède un lock grâce à ces nouveaux éléments.

Et voilà 🎉, maintenant nous pouvons avoir autant de workers que nous voulons, ils ne lanceront pas plusieurs fois le même message :)

![Schedule stateful workers](/media/original/2023/articles/scheduler/image_3_fr.png)

## Du tooling !

### Debug de nos Schedules

Une commande console a été ajoutée depuis [cette PR](https://github.com/symfony/symfony/pull/51795), elle permet de lister toutes les tâches des Schedules que vous avez créé !

```shell
$ bin/console debug:scheduler

Scheduler
=========

default
-------

 -------------- -------------------------------------------------- ---------------------------------
  Trigger    	Provider                                       	Next Run                    	
 -------------- -------------------------------------------------- ---------------------------------
  every 2 days   App\Messenger\Foo(O:17:"App\Messenger\Foo":0:{})   Sun, 03 Dec 2023 04:05:00 +0000
  15 4 */3 * *   App\Messenger\Foo(O:17:"App\Messenger\Foo":0:{})   Mon, 04 Dec 2023 04:15:00 +0000
 -------------- -------------------------------------------------- ---------------------------------
```

En plus de voir les tâches de vos Schedules, vous aurez aussi la prochaine date d'exécution.

### Changer le transport de vos tâches

Parfois un message peut prendre du temps à être traité. Il est donc possible de dire dans son Schedule que notre message doit être traité par un transport donné. Par exemple :

```php
(new Schedule())->add(
    RecurringMessage::cron('15 4 */3 * *', new RedispatchMessage(new Foo(), ‘async’)))
);
```

Ici, quand le message doit être distribué, le worker va le renvoyer vers le transport `async` qui s’occupera alors de le traiter. Très pratique pour les tâches lourdes car cela libérera le worker `scheduler_default` pour traiter les prochains messages.

### Gérer les erreurs

Scheduler permet d’écouter plusieurs événements via le composant `EventDispatcher`. Il existe 3 événements écoutables: `PreRunEvent`, `PostRunEvent` et `FailureEvent`. Les deux premiers seront déclenchés, respectivement, avant et après chaque tâche exécutée. Le dernier, quant à lui, sera lancé en cas d’exception dans une tâche. Cela peut être très pratique pour monitorer de façon efficace vos erreurs :

```php
#[AsEventListener(event: FailureEvent::class)]
final class ScheduleListener
{
    public function __invoke(FailureEvent $event): void
    {
        // triggers email to yourself when your schedules have issues
    }
}
```

Avec ce code, lorsqu’un événement `FailureEvent` arrive, vous pourrez vous envoyer un email ou rajouter des logs pour mieux comprendre le soucis.

### Console as Scheduler

Une des fonctionnalités les plus intéressantes de Scheduler selon moi : les attributs `AsCronTask` et `AsPeriodicTask` ! Ceux-ci permettent de transformer une commande console en une tâche périodique de façon très simple ! `AsPeriodicTask` permet de définir une tâche via une récurrence simple: `2 days` par exemple, et `AsCronTask` permet de faire la même chose via une expression cron.

```php
#[AsCommand(name: 'app:foo')]
#[AsPeriodicTask('2 days', schedule: 'default')]
final class FooCommand extends Command
{
    public function execute(InputInterface $input, OutputInterface $output): int
    {
        // run you command

        return Command::SUCCESS;
    }
}
```

Et voilà, la commande sera exécutée dans le Schedule `default` tous les 2 jours !

Nous retrouvons souvent des doublons entre les commandes console et vos tâches récurrentes, c’est la fonctionnalité parfaite pour faire le lien entre les deux !

## Conclusion

Le composant Scheduler s'impose comme un outil essentiel pour intégrer efficacement les tâches récurrentes dans Symfony. Sa simplicité d'utilisation, sa flexibilité, la gestion des expressions cron, ainsi que son intégration transparente avec les commandes console en font un choix incontournable.

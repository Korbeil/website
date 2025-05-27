+++
title = "✍️ Astuces pour traiter des gros volumes de données dans Symfony"
date = "2023-04-03"
tags = ["php", "symfony", "database", "doctrine"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/astuces-pour-traiter-des-gros-volumes-de-donnees-dans-symfony).

Dans la vie d'un développeur, il arrive forcément un moment où l'on doit traiter un volume important de données via une ligne de commande. Et les premières fois, ça fait BOOM 💥, on utilise du `memory_limit=-1`, [des optimisations](/blog/redis-et-la-memoire-de-php-sont-dans-un-bateau-il-coule)… Dans cet article, je dresse une liste des différents points d'attention qui m'ont déjà été utiles, ainsi que des astuces qui peuvent grandement aider à l'utilisation et à la maintenance de votre traitement.
Attention, cette liste n'est pas exhaustive et les différents points peuvent être applicables ou non selon votre cas.

## Astuces génériques

En premier, des conseils génériques qui peuvent s'appliquer dans la majorité des cas :

- **Avoir un repère d’avancement.** Cela peut paraître trivial, mais avoir une barre d’avancement ou un pourcentage permet de voir rapidement et facilement  la quantité d'éléments à traiter et l'état d'avancement. Par exemple, Symfony propose une classe ProgressBar dans son composant Console qui permet de "calculer" le temps de traitement passé ou restant - estimé en fonction de la vitesse à laquelle nos éléments ont été traités jusqu'à présent - et la consommation mémoire. Voici un exemple :

```php
ProgressBar::setFormatDefinition('custom', '%current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s% %memory:6s% %message%');
$bar = $io->createProgressBar($count);
$bar->setMessage('Starting');
$bar->setFormat('custom');
$bar->start();

foreach ($items as $item) {
	$bar->setMessage(sprintf('Idem ID: %s', $item->getId()));
	$bar->advance();
}

$bar->finish();
```

Utiliser cette technique pose aussi deux soucis. Il faut pouvoir compter les éléments en amont, ce qui n’est pas toujours possible. Et il faut faire attention au taux de refresh du repère affiché en console pour parfois le réduire s’il est trop gourmand (il existe [un paramètre sur la classe ProgressBar](https://github.com/symfony/symfony/blob/6.3/src/Symfony/Component/Console/Helper/ProgressBar.php#L70) qui permet de gérer ça).

- **Rendre son traitement tolérant aux erreurs**, toujours traiter les erreurs via un [channel de log](https://symfony.com/doc/current/logging.html) ou via un rapport. Bien sûr, ce n'est pas une raison pour ignorer ces erreurs, pensez à regarder ce rapport et à agir en conséquence.
- **Ne pas regrouper les traitements**. Une tâche qui traite beaucoup d'éléments va prendre soit du temps, soit de la charge CPU ou elle peut juste planter. Si vous avez plusieurs actions à faire, séparez les en plusieurs commandes qui pourront être lancées indépendamment. J’ai déjà pu voir une commande qui gérait des statuts de paiements et qui faisait aussi la détection de fraude. Vous pouvez totalement séparer les deux actions en deux commandes et ainsi réduire la charge de vos commandes.
- **Rendre votre commande relançable** sans impact sur son exécution. En effet, il peut arriver d’avoir de très long processus qui bugguent au milieu, ou même que la commande soit trop longue pour être exécutée d’un coup. Il faudra donc pouvoir la lancer plusieurs fois. Pour cela, il existe plusieurs méthodes. La première serait d’utiliser un flag quand un élément est déjà traité, ce qui permettra de l’exclure au prochain passage. Une seconde méthode serait d’avoir une borne dans votre commande, par exemple `--continue-after-id` ou encore `--continue-after-date` - ici l’idée est d’avoir une borne à partir de laquelle on reprend le traitement des éléments.
- **Attention à l'utilisation des fonction de manipulation de tableaux**. Par exemple, un `array_merge` au milieu d'une boucle. Il vaut mieux regarder les alternatives pour faire ça en dehors de cette boucle. Vous pouvez aussi préparer votre donnée de base pour éviter d'avoir à faire ce genre de manipulation. De façon plus générale, il vaut mieux éviter tout ce qui utilise de la mémoire dans une boucle, car plus la boucle est grande, plus l’impact est important pour votre serveur.
- **Un rapport à la fin** vous sera toujours utile. Par exemple, avoir le nombre d'éléments traités, la durée de ce traitement, la quantité de mémoire utilisée, le nombre de requêtes SQL engendrées etc. Il est également intéressant d'ajouter des indicateurs métiers : par exemple si votre commande vérifie des paiements pour trouver des fraudes potentielles, avoir le nombre de paiements détectés comme "fraude" ou "validé" peut être pertinent.
- **Paralléliser sur plusieurs workers** permet aussi de beaucoup accélérer le traitement des données. Attention cependant, cela rendra tout de suite votre tâche beaucoup plus complexe car il va falloir partager votre traitement de base en plusieurs tâches en parallèle de façon indépendante.

## Doctrine

Doctrine est souvent utilisé dans les applications Symfony et c'est l'une des sources de données les plus populaires. Je ne pouvais donc pas passer à côté d'une section concernant Doctrine 😀.

- **Nettoyer l’EntityManager** de Doctrine. De nombreuses requêtes vont faire que beaucoup d’objets seront instanciés et des fuites de mémoire vont apparaître à coup sûr. C’est pour cela qu’il est recommandé de nettoyer l’EntityManager à chaque tour de boucle ou dès que vous le pouvez grâce à la méthode `EntityManager::clear()`.
- **Préférer utiliser `toIterable()`** sur les retours de requêtes avec beaucoup de résultats. En effet, lorsqu’on appelle `getResult()`, Doctrine va préparer toute la collection d'un coup, et donc potentiellement avec beaucoup d'éléments. En utilisant `toIterable()`, on aura les éléments hydratés un par un grâce à [un générateur](https://www.php.net/manual/fr/language.generators.syntax.php).
- **[Borner les requêtes](https://stackoverflow.com/questions/8269916/what-is-sliding-window-algorithm-examples#answer-64111403)**. Il peut arriver que nos processus aient des éléments traités en erreur pour diverses raisons. C’est pour cela qu’il est recommandé de borner ses requêtes de façon temporelle (ne récupérer que les éléments des 3 derniers jours par exemple). Ainsi, lorsqu'un élément est traité en erreur, il ne sera automatiquement plus traité au bout de 3 jours (puisqu'il ne sera plus sélectionné). Cela permet de gagner du temps CPU sur vos serveurs une fois la borne dépassée. Une autre solution est de mettre un compteur de tentatives sur vos éléments et d’ignorer les éléments à partir d’un certain nombre de tentatives. Et encore une fois, n’oubliez pas de suivre et traiter ces erreurs !
- **Réduire les combinatoires**. Généralement, on préfère tout traiter dans une commande. Par exemple, si on a des paniers clients à supprimer, on fera une commande journalière pour supprimer les paniers expirés. Mais parfois, il vaut mieux traiter la tâche en plusieurs parties pour éviter de trop faire d’un coup. N’hésitez pas à diviser la tâche en plusieurs lots pour réduire leur taille ou leur temps de traitement. Par exemple, une première commande pour les paniers de la matinée et une seconde pour les paniers de l’après-midi.
- **Utiliser des références plutôt qu’un `find()`**. Quand on doit récupérer une référence à une entité,  sans pour autant lui appliquer des traitements directement, il vaut mieux privilégier l’utilisation de `getReference(Entity::class, $id)`. Cela renverra un proxy de l’objet demandé sans faire de requête vers la base de données, comme le ferait un `find()`. Vous pourrez trouver plus d’informations sur la méthode `getReference` dans [la documentation Doctrine](https://www.doctrine-project.org/projects/doctrine-orm/en/2.14/reference/advanced-configuration.html#reference-proxies).
- **Bien choisir le mode d’hydratation** lors de l’utilisation de vos requêtes SQL. Par défaut, Doctrine met l’hydratation de vos requêtes à `object`. Il va alors transformer le résultat de la requête en un objet et garder en mémoire tous les objets ainsi créés. Cette hydratation d’objet va forcément prendre du temps. Ici, je vous recommande de passer par de l’hydratation `array` voire `scalar` (selon les besoins) pour réduire au maximum le temps de process de vos données par Doctrine.

```
  object - 113Mib -  575ms
  simple - 106Mib -  457ms
   array - 066Mib -  333ms
  scalar - 066Mib -  291ms
```

- **Régler le cache Doctrine et sur votre base de données**. Lorsqu’on fait de nombreuses fois la même requête avec possiblement des doublons, il peut être très utile de mettre en place du cache côté Doctrine. Pour cela il existe trois niveaux de cache dans Doctrine: d’abord le cache sur le mapping entre vos entités et les tables en base de données. Puis le Query cache, qui permet de mettre en cache la construction de la requête SQL depuis le <abbr title="Doctrine Query Language">DQL</abbr>. Et le Result cache qui va permettre de conserver les résultats de votre requête pour les réutiliser si la requête venait à être exécutée une seconde fois. Les caches sur les metadata (donc le mapping) et la query sont activés par défaut dans Symfony.
  Aussi, il existe parfois, selon le moteur utilisé, un cache côté base de données qui permet de garder les résultats des requêtes pour ensuite répondre plus vite. Pour MySQL il est désactivé par défaut depuis 5.6 et ⚠️ il a été [retiré depuis MySQL 8.0](https://dev.mysql.com/blog-archive/mysql-8-0-retiring-support-for-the-query-cache/).

## Fichier

Il est parfois nécessaire de traiter de la donnée provenant d’un fichier, ou bien d’écrire cette donnée dans un fichier.

- **Générer le contenu du fichier via un fichier temporaire**. Quand on doit créer de gros fichiers, le fait de les générer puis de les écrire sur le disque peut utiliser beaucoup de mémoire. C’est pour ça qu’il est recommandé de passer par un générateur et d’écrire dans un fichier temporaire (via l'utilisation de `tmpfile()`) ligne par ligne (au plus petit élément possible) puis d’envoyer le fichier temporaire. Cela évite d’avoir beaucoup de RAM occupée par la variable qui va stocker ces données et permet aussi de gérer plus facilement les soucis de données. Vous pouvez aussi générer votre propre fichier et utiliser des `file_put_contents()` pour écrire dedans, mais pour ma part, je suis plus habitué à `fwrite` donc la fonction `tmpfile()` est plus adaptée.
- **Génération de fichiers en asynchrone**. Lorsqu'on génère des fichiers, la tâche nous semble toujours rapide au début. Puis au fur et à mesure de l'évolution de notre code, le volume de ces fichiers augmente, les rendant de plus en plus lents à générer. C'est pour cela que je recommande de générer vos fichiers via un worker asynchrone. De la même manière, si vous travaillez sur une requête HTTP pour une API, vous pouvez envoyer la génération du fichier en tâche de fond. Après, vous pouvez soit mettre en place une interface pour récupérer vos fichiers, soit passer par un envoi de lien dans un email ou autre moyen de notifier la disponibilité du fichier généré.

## Monitoring

Et pour finir, on aime traiter de la donnée, mais il faut aussi la surveiller !

- Lorsque vous faites du traitement de lots de données, pensez à **surveiller les informations système de vos machines** ! RAM, CPU, espace disque… on ne sait jamais ce qui peut poser souci, donc autant avoir un œil sur tout ce qui pourrait bloquer ou ralentir votre traitement. On peut imaginer de suivre la RAM utilisée ou disponible en combinaison d’une barre de progression. Aussi, je ne peux que vous recommander quelques outils comme [Netdata](https://www.netdata.cloud/) ou l’utilisation de sondes de monitoring comme [Datadog](https://www.datadoghq.com/), [Prometheus](https://github.com/prometheus/node_exporter) et bien d’autres.
- Faites attention au **garbage collector de PHP**. En effet, par défaut, le garbage collector de PHP passera automatiquement quand il détecte que 10,000 objets sont instanciés pour nettoyer tous les objets qui ne sont plus utilisés. Lorsque vous avez un processus sur beaucoup de données, il peut être intéressant de désactiver ce comportement du garbage collector pour le faire vous-même à la main. Plus d’informations sur [la documentation PHP](https://www.php.net/manual/en/features.gc.collecting-cycles.php).
- Et bien sûr **ajoutez des métriques métiers** ! Par exemple, si l'on doit passer des commandes d'un statut "en attente" à "validé", il peut être judicieux de mesurer le nombre de commandes traitées, le nombre de commandes validées ou encore le nombre d'erreurs. La moindre information vous aidera à comprendre votre tâche et suivre le traitement de ces données.

Vous pourrez aussi trouver des astuces pour monitorer votre code PHP simplement dans [un de nos précédents articles de blog](/blog/comment-monitorer-rapidement-du-code-php).

## Pour finir

J'espère que ce billet vous aura permis d'apprendre des choses, n'hésitez pas à y revenir lorsque vous aurez à traiter des gros lots de données. Un peu comme une checklist pour voir ce que nous vous recommandons de faire dans ce cas là. Mais n’oubliez pas qu’il existe des outils de profiling qui vous feront remonter les métriques CPU, RAM, requêtes SQL… et qui vous permettront d’ajuster vos process avec plus de précision ! Je pense faire évoluer cette liste au fur et à mesure et pourquoi pas ajouter vos recommandations si vous en avez. 😀
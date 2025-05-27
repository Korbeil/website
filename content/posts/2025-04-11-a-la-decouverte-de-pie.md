+++
title = "✍️  À la découverte de PIE, l’alternative moderne à PECL pour les extensions PHP"
date = "2025-04-11"
tags = ["php", "extension", "pie"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/a-la-decouverte-de-pie-lalternative-moderne-a-pecl-pour-les-extensions-php).

Récemment vous avez peut-être entendu parler de **PIE**, un nouveau binaire pour PHP. PIE c’est le diminutif de **“PHP Installer for Extensions”** et c’est donc le descendant de PECL.

## Pourquoi PIE ?

PHP, né en 1995, célèbre cette année ses 30 ans d'existence 🎉. Durant ces trois décennies, l'écosystème de ce langage s'est considérablement enrichi avec l'émergence d'outils devenus indispensables aux développeurs.

En 1999, **PEAR** (PHP Extension and Application Repository) a posé les bases de la réutilisation de code en servant de premier gestionnaire de bibliothèques standardisés.

Puis en 2003, **PECL** (PHP Extension Community Library) a étendu les capacités du langage en facilitant l'intégration d'extensions compilées pour traiter efficacement des données complexes comme le XML ou les chaînes de caractères multioctets (mbstring).

La véritable révolution est survenue en 2012 avec Composer, qui a transformé radicalement la gestion des dépendances dans les projets PHP.

Plus récemment, après plus de 25 ans d'existence de PECL, **PIE** (PHP Install Extensions) a fait son apparition comme alternative prometteuse. Encore en phase de développement, cette initiative, lancée par la **PHP Foundation**, apporte une approche moderne et simplifiée pour l'installation et la gestion des extensions PHP. Son utilisation actuelle, bien que limitée par sa maturité, s’illustre déjà par sa simplicité que nous allons voir ensemble.

D'ici quelque temps, vous installerez vos extensions PHP de cette façon.
## Mise en place

Avant de commencer l'installation de PIE, vous devez préparer votre environnement avec quelques outils de développement essentiels. Utilisez la commande suivante pour installer les pré-requis nécessaires (pour linux) :

```bash
sudo apt install git autoconf automake libtool m4 make gcc
```

Puis l'installation de l’outil est remarquablement simple, ne nécessitant qu'une seule ligne de commande (oui c'est un PHAR, comme Composer ou [Castor](https://castor.jolicode.com/)) :

```bash
sudo curl -L --output /usr/local/bin/pie https://github.com/php/pie/releases/latest/download/pie.phar && sudo chmod +x /usr/local/bin/pie
```

Cette méthode d’installation est adaptée pour les distributions Linux. Si vous utilisez autre chose, vous pouvez voir dans la documentation de PIE [pour les méthodes alternatives d’installation](https://github.com/php/pie/blob/main/docs/usage.md#installing-pie)

Et voilà, avec ces deux commandes vous avez l’outil installé dans sa dernière version et prêt à être utilisé ! 🎉

## Première utilisation

Après l'installation, vous pouvez rapidement enrichir votre environnement PHP avec de nouvelles extensions.
Commençons par un exemple pratique. Supposons que vous souhaitiez ajouter l'extension `uuid` pour gérer des identifiants uniques dans votre projet, alors il vous suffira de faire la commande suivante :

```bash
pie install pecl/uuid
```

Selon votre niveau de privilèges sur ce système Linux, le programme pourrait vous demander d'utiliser la commande sudo pour obtenir les droits nécessaires à l'installation de l'extension demandée.
Suite à cette installation, l’extension sera disponible globalement sur l'ensemble du système et non limitée au projet actuel.
Vous pouvez découvrir toutes les extensions disponibles sur [**packagist**](https://packagist.org/extensions).

## Diverses commandes

L'outil propose une panoplie de commandes pour gérer vos extensions PHP. Par exemple, pour supprimer une extension précédemment installée :

```bash
pie uninstall pecl/uuid
```

Ou encore pour lister les extensions installées :
```bash
pie show
```

Et enfin pour afficher les détails d'une extension en particulier :
```bash
pie info pecl/uuid
```

## Gérer plusieurs dépôts d'extensions

PIE ne se limite pas aux extensions disponibles sur Packagist. L'outil offre une grande flexibilité en permettant l'installation d'extensions provenant de diverses sources : dépôts GitHub, archives locales, ou même des extensions développées en interne.

```bash
pie repository:add path /path/to/your/local/extension
pie repository:add vcs https://github.com/youruser/yourextension
pie repository:add composer https://repo.packagist.com/your-private-packagist/
```

Et vous retrouverez aussi les commandes pour supprimer un dépôt :
```bash
pie repository:remove /path/to/your/local/extension
```

Ou juste lister les dépôts que vous avez ajoutés :
```bash
pie repository:list
```

Comme l’outil utilise symfony/console, vous pouvez avoir de l’autocompletion :

```bash
pie completion | sudo tee /etc/bash_completion.d/pie
```
## Conclusion

Lors de la relecture de cet article, une question m'a été posée : **pourquoi ne pas utiliser APT** ou d'autres gestionnaires de paquets (comme Brew, Yum, etc.) à la place ? À mon avis, PIE offre un avantage considérable : l'accès à une bibliothèque d'extensions PHP beaucoup plus vaste. Les gestionnaires de paquets traditionnels exigent des développeurs qu'ils créent et compilent leur package pour différentes architectures, ce qui représente un travail conséquent. En revanche, PIE simplifie radicalement ce processus. Il récupère automatiquement le code source et les dépendances de l'extension, puis la compile spécifiquement pour l'appareil où il est installé. Pour les développeurs, la procédure est allégée : il leur suffit d'enregistrer leur extension sur Packagist pour la rendre disponible à tous les utilisateurs de PIE.

Tout de même, PIE démontre déjà des fonctionnalités solides et intuitives pour la gestion des extensions PHP. Bien qu'encore en développement, l'outil offre une base prometteuse qui simplifie l'installation et la gestion des extensions.

Nous sommes impatients de voir comment la communauté PHP va adopter et faire évoluer PIE dans les mois et années à venir.

Pour plus d'informations n'hésitez pas à consulter le [repository github de l'outil](https://github.com/php/pie). Ou suivre le [blog de la PHP Foundation](https://thephp.foundation/blog/) !

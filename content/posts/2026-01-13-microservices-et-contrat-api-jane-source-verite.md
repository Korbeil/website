+++
title = "✍️  Microservices et contrats d'API : Jane comme source de vérité"
date = "2026-01-13"
tags = ["php", "symfony", "json", "api", "jane", "openapi"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/microservices-et-contrats-d-api-jane-comme-source-de-verite).

Dans le développement d'une API, nous sommes tous confrontés au même défi : maintenir la cohérence entre la documentation et le code.

Qui n'a jamais perdu des heures à débugger une erreur parce que le champ `user_id` était devenu `userId` dans le code, mais pas dans la documentation ? C'est ce qu'on appelle le "drift". À mesure que le projet évolue, le code change, mais la documentation (OpenAPI, Wiki, Postman) traîne souvent la patte, devenant une source d'erreurs plutôt qu'une aide.

Et si la solution n'était pas de mettre à jour manuellement notre code pour coller à la doc, mais de générer automatiquement notre code *à partir* de la doc ? C'est ici qu'intervient **Jane**.

## Découvrir Jane

En deux mots : Jane est une suite de librairies PHP dont la mission est de générer du code de qualité à partir de vos spécifications **[JSON Schema](https://json-schema.org/)** ou **[OpenAPI](https://www.openapis.org/)**.

Si l'on devait résumer son rôle dans une API moderne, Jane agit comme le "traducteur automatique" entre vos contrats d'interface (vos spécifications `.yaml` ou `.json`) et votre code PHP. Au lieu d'écrire manuellement vos classes, vos validateurs et vos clients HTTP — une tâche répétitive et sujette à l'erreur humaine — Jane les fabrique pour vous.

Concrètement, Jane analyse votre schéma et produit :

* **Des Modèles (DTO) :** Des classes PHP simples (POPO - *Plain Old PHP Objects*) strictement typées qui représentent vos données ;
* **Des Normalizers :** Toute la logique nécessaire pour transformer ces objets en JSON et inversement, en s'appuyant sur le composant `symfony/serializer` ;
* **Un Client HTTP complet :** (dans le cas du composant OpenAPI) Une implémentation prête à l'emploi (compatible PSR-18) pour consommer l'API, gérant les requêtes, les réponses et même les exceptions définies dans votre spec.

L'atout majeur de Jane n'est pas seulement le gain de temps, c'est la garantie de **conformité**. Puisque le code est généré directement depuis la source de vérité (le schéma), il est impossible d'avoir une divergence ("drift") entre ce que votre documentation prétend faire et ce que votre code fait réellement. Si le schéma change, vous régénérez le code, et le tour est joué.

## **Voyons un projet d’exemple**

Pour passer de la théorie à la pratique, nous allons construire un cas d'usage classique : un **tunnel d'achat e-commerce**.

Nous allons simuler la transformation d'un panier en une commande validée. Pour cela, nous découpons la logique en deux micro-services distincts :

1. **Le service Panier** : Il matérialise l'état d'attente avant la commande. C'est lui qui mémorise les articles choisis au fur et à mesure que le client parcourt le site, avant qu'il ne décide (ou non) de passer à l'achat ;
2. **Le service Commande** : Il gère la finalisation de la vente et la collecte des informations client.

Ce scénario nous permettra d'explorer des cas concrets :

* **Côté Panier**, nous utiliserons des UUID pour récupérer le contenu du panier ;
* **Côté Commande**, nous mettrons en place une validation stricte des données (regex pour le téléphone, énumération pour le pays) lors de la transformation du panier en commande, ainsi qu'un endpoint pour les codes promo.

Voyons maintenant comment formaliser tout cela dans nos contrats d'API.

## **Créer nos micro-services (1 / 2) : Panier**

Commençons par le service le plus simple : le **Panier**.

Ici, nous prenons le parti du *Design First* : avant d'écrire la moindre ligne de PHP, nous allons figer la structure de nos échanges dans un fichier `cart.yaml`.

Pour ce service, le besoin est basique : nous voulons pouvoir récupérer un panier via son identifiant unique. Voici notre définition en OpenAPI 3.0.3 :

```yaml
openapi: 3.0.3
info:
  title: 'Service Panier'
  version: 1.0.0
paths:
  /carts/{id}:
    get:
      summary: 'Récupérer un panier'
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: 'Le panier trouvé'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Cart'
components:
  schemas:
    Cart:
      type: object
      properties:
        id:
          type: string
          format: uuid
        items:
          type: array
          items:
            type: string # simplifié pour l'exemple
```

### **Ce qu'il faut retenir ici**

L'utilisation du format `uuid` sur l'identifiant (`id`) n'est pas anodine. Jane exploite cette information pour enrichir le code généré avec des règles de validation strictes, garantissant que la donnée n'est pas une simple chaîne de caractères mais bien un UUID valide.

Nous avons posé les bases du Panier. Attaquons-nous maintenant au second morceau du puzzle, qui va nous demander un peu plus de rigueur : le service Commande.

## **Créer nos micro-services (2 / 2) : Commande**

Passons maintenant aux choses sérieuses avec le service **Commande**. Si le Panier était une simple lecture, la Commande implique de recevoir et valider des données utilisateur critiques.

C'est l'occasion idéale pour définir des règles strictes directement dans notre fichier `order.yaml`. Nous allons déclarer deux routes :

1. Une route de création de commande, qui valide l'adresse et le téléphone, données indispensables pour assurer la livraison ;
2. Une route pour appliquer un code de réduction à une commande (qui impactera le prix).

Voici à quoi ressemble notre contrat :

```yaml
openapi: 3.0.3
info:
  title: 'Service Commande'
  version: 1.0.0
paths:
  /orders:
    post:
      summary: 'Créer une commande à partir d un panier'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderInput'
      responses:
        '201':
          description: 'Commande créée'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

/orders/{id}/discount:
    post:
      summary: 'Ajouter un code promo'
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                code:
                  type: string
      responses:
        '200':
          description: 'Code promo appliqué'

components:
  schemas:
    OrderInput:
      type: object
      required: ['cartId', 'customer']
      properties:
        cartId:
          type: string
          format: uuid
        customer:
          $ref: '#/components/schemas/CustomerAddress'

    CustomerAddress:
      type: object
      required: ['firstName', 'lastName', 'phoneNumber', 'countryCode']
      properties:
        firstName:
          type: string
        lastName:
          type: string
        phoneNumber:
          type: string
          pattern: '^\+33[1-9]\d{8}$' # Validation Regex pour numéros français
          description: 'Format: +33xxxxxxxxx'
        countryCode:
          type: string
          enum: ['FR', 'BE', 'LU'] # Limitation par énumération

    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: ['pending', 'paid', 'shipped']
        price:
          type: number
          description: 'Prix final après réduction éventuelle'
```

### **Pourquoi aller aussi loin dans la spec ?**

Regardez bien le schéma `CustomerAddress`. Nous n'avons pas seulement défini des chaînes de caractères, nous avons imposé des contraintes métiers :

* `pattern` : Le champ `phoneNumber` doit correspondre à une regex précise (+33...).
* `enum` : Le champ `countryCode` ne peut être qu'une valeur parmi une liste définie (France, Belgique, Luxembourg).

Au lieu de coder ces validations "à la main" dans chaque contrôleur PHP, nous les déclarons une seule fois dans le contrat. Jane se chargera de traduire ces contraintes dans le code généré, garantissant que si une donnée ne respecte pas la spec, elle ne passera pas.

Nos contrats sont prêts. Il est temps de passer au PHP.

**Configuration de Jane pour le projet exemple**

Nos spécifications OpenAPI sont prêtes (`cart.yaml` et `order.yaml`). Il faut maintenant expliquer à Jane comment les transformer en code PHP.

Cela se passe via un fichier de configuration, généralement nommé `.jane-openapi.php` à la racine du projet.

Pour gérer nos deux micro-services (Panier et Commande) sans mélanger leur code, nous utilisons l'option `mapping`. Elle permet de définir des règles de génération spécifiques pour chaque fichier OpenAPI au sein d'une seule et même configuration.

Voici à quoi ressemble notre fichier `.jane-openapi.php`:

```php
<?php

return [
    'mapping' => [
        __DIR__ . '/cart.yaml' => [
            'namespace' => 'App\Generated\Cart',
            'directory' => __DIR__ . '/generated/Cart',
        ],
        __DIR__ . '/order.yaml' => [
            'namespace' => 'App\Generated\Order',
            'directory' => __DIR__ . '/generated/Order',
        ],
    ],
    ‘validation’ => true,
];
```

### **Les options disponibles**

Déchiffrons ensemble cette configuration :

* `mapping` : C'est la clé maîtresse qui nous permet d'associer chaque fichier de spécification (la clé du sous-tableau, ici le chemin vers `cart.yaml` ou `order.yaml`) à sa propre configuration.
* `namespace` : C'est ici que l'organisation se joue. En donnant des namespaces différents (`App\Generated\Cart` vs `App\Generated\Order`), nous isolons complètement les domaines. Si un modèle s'appelle `Error` dans les deux services, il n'y aura aucun conflit de nom de classe dans notre projet.
* `directory` : Le dossier de destination où le code PHP généré pour chaque service va être écrit.
* `validation` : **C'est une option capitale.** En la passant à `true`, nous demandons à Jane de traduire les contraintes du schéma OpenAPI (comme `minimum: 1`, `required`, `email`) en métadonnées de **Symfony Validator**. C'est le socle qui nous permettra de garantir la robustesse des données sans avoir à réécrire manuellement les règles métiers en PHP. Nous verrons plus loin à quel point cela nous simplifie la vie.

Avec cette structure, Jane va itérer sur chaque entrée du mapping et produire automatiquement deux clients HTTP (compatibles PSR-18) distincts et autonomes.

## **Et les erreurs ? Standardisation et gestion avec Jane**

Gérer le "happy path" (code 200), c'est facile. Mais dans la vraie vie, une API échoue : validation invalide (400), accès refusé (403) ou crash serveur (500).

Si nous laissons Jane deviner, elle lèvera des exceptions génériques HTTP. Mais nous pouvons faire mieux : nous pouvons uniformiser le format de nos erreurs pour que notre SDK PHP renvoie des objets structurés, faciles à manipuler dans un `try/catch`.

### **Définir un schéma d'erreur standard**

Dans nos fichiers OpenAPI (`cart.yaml` et `order.yaml`), nous allons ajouter une définition réutilisable dans la section `components`. Cela permet d'avoir un format unique (par exemple : un message et un code) pour toutes nos erreurs.

Ajoutons ceci à la fin de nos fichiers YAML :

```yaml
components:
  schemas:
    # ... nos autres modèles (Cart, Product...)

    # Notre modèle d'erreur standardisé
    Error:
      type: object
      required:
        - message
        - code
      properties:
        message:
          type: string
          description: Le message d'erreur pour le développeur
        code:
          type: integer
          description: Un code d'erreur interne spécifique
```

### **Référencer ce schéma dans les endpoints**

Maintenant que le modèle `Error` existe, nous devons dire à chaque endpoint : "Si nous renvoyons une 400, 403 ou 500, le corps de la réponse respectera ce schéma".

Reprenons l'exemple de l'ajout d'un code de réduction sur une commande:

```yaml
paths:
  /orders/{id}/discount:
    post:
      summary: 'Ajouter un code promo'
      # ... (requestBody, etc)
      responses:
        '200':
          description: 'Code promo appliqué'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

        # Gestion des erreurs standardisée
        '400':
          description: 'Données invalides (ex: code non trouvé)'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '403':
          description: Action non autorisée
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '404':
          description: Commande introuvable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: Erreur serveur
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
```

### **Ce que Jane va générer pour nous**

C'est ici que la magie opère. Lors de la génération, Jane va détecter ces configurations et produire deux choses très utiles :

1. **Un <abbr title="Plain Old PHP Objects">POPO</abbr>** `Error` : Une classe PHP classique (`App\Generated\Order\Model\Error`) avec ses getters et setters.
2. **Des Exceptions spécifiques** : Pour l'endpoint ci-dessus, Jane va générer des exceptions comme `AddDiscountBadRequestException` ou `AddDiscountNotFoundException`.

Ces exceptions auront une méthode `getError()` qui retournera... notre instance du modèle `Error` remplie avec les données de l'API !

Cela permet d'écrire un code client très propre :

```php
try {
    $api->addDiscount('C17891', new OrdersIdDiscountPostBody('NOEL2025'));
} catch (AddDiscountBadRequestException|AddDiscountNotFoundException $e) {
    // On récupère notre objet Error proprement
    $errorModel = $e->getResponse();
    echo "Erreur fonctionnelle : " . $errorModel->getMessage();
} catch (ClientExceptionInterface $e) {
    // Erreur réseau ou autre
}
```
En standardisant vos retours d'erreurs dans l'OpenAPI, vous garantissez que le code PHP généré est prévisible et facile à utiliser pour les développeurs qui consommeront votre SDK.

## **Lancer la génération du code !**

C'est ici que la magie opère. Fini d'écrire des DTOs, des clients HTTP et des exceptions à la main. Jane va faire tout ce travail fastidieux pour nous.

Dans votre terminal, à la racine du projet, lancez simplement :

```bash
vendor/bin/jane-openapi generate --config-file .jane-openapi.php
```

Jane va lire le fichier `.jane-openapi.php`, détecter notre mapping et générer les fichiers correspondants. Si vous allez voir dans votre dossier `generated/`, nous avons deux nouveaux dossiers qui sont apparus : `Cart` et `Order`.

### **Une étape indispensable : l'autoloader**

Avoir les fichiers PHP, c'est bien, mais pouvoir les utiliser, c'est mieux ! Comme ce sont de nouvelles classes dans un nouveau dossier, Composer ne les connaît pas encore.

Pour que Composer puisse charger ces nouvelles classes, nous devons mettre à jour le `composer.json`. Plutôt que de déclarer chaque service un par un, allons au plus simple : nous allons faire pointer le namespace parent `App\Generated\` directement vers le dossier `generated/`.

Ouvrez votre `composer.json` et ajoutez ces lignes dans la section `autoload` :

```json
"autoload": {
    "psr-4": {
        "App\\": "src/",
        "App\\Generated\\": "generated/"
    }
},
```
Ensuite, pour que cette modification soit prise en compte, régénérez l'autoloader :

```bash
composer dump-autoload
```

C'est tout ! Votre projet est maintenant prêt à utiliser les librairies générées.

### **Que contient ce dossier generated ?**

Si vous êtes curieux (et vous devriez l'être), jetez un œil à l'intérieur de `generated/Cart` ou `generated/Order`. Vous y trouverez une structure très organisée :

```
generated/
├── Order
│   ├── Client.php
│   ├── Endpoint
│   ├── Exception
│   ├── Model
│   ├── Normalizer
│   ├── Runtime
│   │   ├── Client
│   │   └── Normalizer
│   └── Validator
└── Cart
    ├── Same file structure as Order
    └── ...
```

* `Client.php` : Votre **point d'entrée** principal. C'est la classe qui contient toutes les méthodes comme `createOrder()`, `addDiscount()`, etc.
* `Endpoint/` :  Chaque opération définie dans votre schéma OpenAPI (comme `GET /carts/{id}`) possède ici sa propre classe dédiée. C'est elle qui orchestre l'appel API : elle sait quelle méthode HTTP utiliser, quel corps de requête envoyer, comment transformer la réponse brute en un objet de votre `Model` et comment gérer les exceptions selon le code de retour.
* `Exception/` : C'est ici que se trouvent nos **erreurs typées** (`AddDiscountBadRequestException`, etc.) basées sur les codes HTTP et nos schémas d'erreurs.
* `Model/` : Vos **POPO** (Plain Old PHP Objects). Ce sont les classes `Cart`, `Order`, `OrderInput`, etc. Elles contiennent simplement des propriétés, des getters et des setters.
* `Normalizer/` : C'est le cœur de Jane (basé sur le composant **Serializer** de Symfony). Ces classes savent comment transformer vos objets en JSON et inversement.
* `Runtime/` : C'est la salle des machines. Ce dossier contient le code "support" nécessaire au fonctionnement global du SDK généré (configuration du client, normalizers de base). Ce sont des composants techniques qui font le lien entre votre code généré et les librairies tierces (comme le client HTTP PSR-18).
* `Validator/` : C'est la particularité liée à notre option `"validation": true`. Jane génère des classes de contraintes spécifiques (ex: `OrderInputConstraint`).
* *Détail technique :* Ces classes étendent `Compound` de Symfony Validator. Elles regroupent toutes les règles définies dans votre OpenAPI (`Required`, `Email`, `Regex` etc.) en une seule classe appelable. Cela garde vos modèles propres tout en garantissant une validation stricte.

##

## **Utilisation pratique : intégrer les modèles Jane dans votre code**

Maintenant que nos classes sont générées, comment les utiliser concrètement ?

Jane brille par sa rigueur : elle sécurise les échanges via ses Normalizers et force l'utilisation d'objets typés.

### La validation automatique

L'un des atouts de Jane est qu'elle intègre la validation directement dans le processus de **sérialisation et de désérialisation**.

Cela signifie que le contrôle des données est actif dans les deux sens :

1. **En entrée (Désérialisation)** : Vous ne pouvez pas créer un objet PHP invalide à partir d'un JSON reçu.
2. **En sortie (Sérialisation)** : Vous ne pouvez pas générer un JSON invalide à partir d'un objet PHP (par exemple, si vous avez oublié de remplir un champ obligatoire avant de renvoyer la réponse).

Vous n'avez donc plus besoin d'appeler le validateur manuellement. Regardez le code généré dans le `Normalizer` (ici `OrdersIdDiscountPostBodyNormalizer`) : il vérifie les contraintes **pendant** la transformation.

Si vous utilisez le Serializer de Symfony dans votre contrôleur, la validation est implicite :

```php
namespace App\Controller;

use App\Generated\Order\Model\OrdersIdDiscountPostBody; // Le DTO généré
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Serializer\SerializerInterface;
use Symfony\Component\Serializer\Exception\ValidationFailedException;

class OrderController extends AbstractController
{
    public function addDiscount(Request $request, SerializerInterface $serializer)
    {
        try {
            // C'est ici que la magie opère :
            // Jane désérialise ET valide en même temps.
            // Si le JSON est invalide (code manquant, trop court...), une exception est levée.
            /** @var OrdersIdDiscountPostBody $body */
            $body = $serializer->deserialize(
                $request->getContent(),
                OrdersIdDiscountPostBody::class,
                'json'
            );

        } catch (ValidationFailedException $e) {
            // On récupère les violations pour répondre une 400 propre via la constante Symfony
            return $this->json($e->getViolations(), Response::HTTP_BAD_REQUEST);
        }

        // Si on arrive ici, $body est un objet valide et typé !
        $code = $body->getCode();
        // ... traitement métier
    }
}
```
**Le gain ?** Votre code métier ne manipule jamais de données "sales". Et inversement, vous avez la certitude absolue que vos réponses API respectent toujours le contrat défini dans votre fichier YAML.

### Gérer les erreurs (Exceptions typées)

C'est ici que notre travail sur les schémas d'erreurs `400` / `404` / `500` porte ses fruits. Jane a généré des exceptions spécifiques pour chaque cas d'erreur documenté.

Fini les vérifications manuelles du status code, place aux `try/catch` explicites :

```php
use App\Generated\Order\Exception\AddDiscountBadRequestException;
use App\Generated\Order\Exception\AddDiscountNotFoundException;
use App\Generated\Order\Model\OrdersIdDiscountPostBody;

try {
    $payload = new OrdersIdDiscountPostBody();
    $payload->setCode('INVALID');

    $apiClient->addDiscount('order-12345', $payload);

} catch (AddDiscountBadRequestException $e) {
    // C'est une 400 spécifique à cette route (Response::HTTP_BAD_REQUEST)
    // On récupère notre objet Error structuré défini dans l'OpenAPI
    $errorModel = $e->getError();

    // Affiche : "Code promo invalide ou expiré"
    $logger->warning($errorModel->getMessage());

} catch (AddDiscountNotFoundException $e) {
    // C'est une 404 spécifique (Commande ou code de réduction introuvable)
    // ...
} catch (\Exception $e) {
    // Erreur réseau ou 500 générique
}
```

## Communication inter-services : Quand le service Commande appelle le service Panier

Dans une architecture microservices, la communication synchrone (HTTP) est souvent le talon d'Achille. On se retrouve vite avec des appels `curl` ou `Guzzle` éparpillés, des tableaux associatifs non typés et, surtout, une confiance aveugle envers le service appelé.

Maintenant, nous allons voir comment Jane sécurise l'échange entre notre **service Commande** (le consommateur) et le **service Panier** (le producteur).

### Le contrat comme dépendance : Réutiliser le schéma OpenAPI

Au lieu de redéfinir manuellement une classe `PanierDTO` dans le service Commande (qui finirait inévitablement par diverger de la réalité), nous utilisons directement la définition OpenAPI du service Panier.

Concrètement, cela signifie que le fichier `openapi.yaml` du service Panier devient une "dépendance" du service Commande.

* **Configuration Jane** : Dans le service Commande, nous configurons Jane pour pointer vers ce fichier (via une URL, un submodule git, …).
* **Single Source of Truth** : Si l'équipe Panier met à jour son modèle (par exemple, en ajoutant un champ `promoCode`), le service Commande le saura dès la prochaine régénération du code.

### Génération d'un Client HTTP typé (SDK interne)

C'est une des fonctionnalités clé de Jane pour la communication inter-services. En plus des modèles (DTOs), Jane a généré un Client HTTP complet prêt à l'emploi (compatible PSR-18).

Pour l'utiliser, on va instancier le client via sa méthode statique `create()` :

```php
use App\Generated\Panier\Client;

$panierClient = Client::create();
$panier = $panierClient->getPanier($uuid);

// $panier est une instance de la classe \App\Generated\Panier\Model\Panier
// L'IDE connaît toutes les propriétés :
$total = $panier->getTotal();
foreach ($panier->getItems() as $item) {
    // ...
}
```
Nous manipulons des objets PHP natifs, typés, générés spécifiquement pour cette interaction.

### Validation automatique de la réponse

Que se passe-t-il si le service Panier a un bug et renvoie un prix sous forme de chaîne de caractères `10,50€` au lieu d'un nombre `10.50`, ou s'il manque un champ obligatoire ?

Avec un client HTTP manuel, votre code planterait probablement plus loin, de manière obscure (`Call to member function on null` ou erreur de calcul), ou pire, corromprait vos données silencieusement.

Avec le client généré par Jane :

* Le client valide **automatiquement** la réponse HTTP reçue par rapport au schéma OpenAPI ;
* Si la réponse ne respecte pas le contrat (type incorrect, champ manquant), Jane lève immédiatement une `UnexpectedValueException` ou une `InvalidResponseException` ;
* **Fail Fast** : L'application s'arrête net à la frontière du service, empêchant des données invalides de polluer votre logique métier de commande.

En résumé, Jane transforme un appel HTTP incertain en un appel de méthode PHP sûr et typé.

## Validation stricte dans les tests

Au lieu de charger vos données de test et ne jamais vérifier leur intégrité, passez-les dans la moulinette de Jane.

Si votre donnée ne correspond plus à la réalité de l'API (changement de type, champ manquant), Jane le détecte immédiatement.

```php
// 1. On charge la donnée brute (ici on prends un JSON en exemple
// mais un endpoint API fonctionnera pareil
$data = json_decode(file_get_contents('panier_fixture.json'), true);

// 2. On demande à Jane de créer l'objet.
// C'est ici que la magie opère : Jane vérifie TOUT le contrat via la validation
// dans le Normalizer généré
try {
    $panier = $serializer->denormalize($data, Panier::class);
} catch (Exception $e) {
    // Le test échoue immédiatement si le JSON est périmé !
    $this->fail('Vos données de test ne respectent plus le contrat OpenAPI : ' . $e->getMessage());
}

// 3. On utilise un objet PHP valide pour configurer le mock
// ou autre test que vous voudriez faire
$mockService->method('getPanier')->willReturn($panier);
```
Pourquoi c'est puissant ?

Cela transforme vos tests unitaires en tests de contrat légers. Vous n'avez plus peur que vos tests "passent au vert" alors qu'ils utilisent des données obsolètes qui feront planter la production.

## Évolution et versioning : Faire évoluer votre API sans tout casser

Une API n'est jamais figée. Elle vit, elle change, elle s'étend. Le cauchemar de tout développeur est de déployer une mise à jour qui casse la communication entre les services. Avec Jane, l'impact d'une modification du contrat OpenAPI devient immédiatement visible dans votre code PHP.

### Ajout d'un champ optionnel (Non-breaking change)

C'est le scénario idéal. Vous ajoutez, par exemple, une `description` facultative à votre objet `Commande`.

* **Dans l'OpenAPI :** Vous ajoutez le champ sans le marquer comme `required`.
* **Après régénération Jane :** Votre classe `Commande` gagne une propriété `protected ?string $description = null;` ainsi que ses getters et setters.
* **Impact :** **Nul.** Votre code existant continue de fonctionner parfaitement car le constructeur reste compatible (le nouveau champ est initialisé à `null` par défaut). C'est une évolution en douceur.

### Ajout d'un champ obligatoire (Breaking change)

C'est ici que Jane vous sauve la mise. Imaginons que le service Panier exige désormais un `customerId` pour chaque création de panier.

* **Dans l'OpenAPI :** Le champ `customerId` est ajouté à la liste `required`.
* **Après régénération Jane :** La signature du constructeur de la classe `Panier` change radicalement.
  * *Avant :* `public function __construct(string $uuid)`
  * *Après :* `public function __construct(string $uuid, string $customerId)`
* **Impact :** **Immédiat et visible.** Partout dans votre code où vous instanciez un `Panier` sans ce nouvel argument, **votre code ne fonctionne plus**. Votre IDE (PhpStorm, VSCode) surligne les erreurs en rouge avant même que vous ne lanciez le moindre test.

**Note clé :** Jane transforme une erreur de "runtime" (qui arriverait en production) en une erreur de "compile-time" (qui arrive sur votre machine).

### Régénération et impact sur le code : La boucle de sécurité

Le workflow de mise à jour devient alors très sécurisant :

1. **Mise à jour du contrat :** Vous récupérez la nouvelle version de `openapi.yaml`.
2. **Régénération :** Vous lancez la génération du code Jane.
3. **Analyse d'impact :** Vous lancez l'analyse statique (PHPStan, Psalm) ou simplement vos tests.
4. **Correction :** Jane vous a forcé à voir tous les endroits du code impactés par le changement. Vous ne pouvez pas "oublier" de traiter le nouveau champ obligatoire.

En résumé, Jane agit comme un **révélateur de dette technique** immédiat lors des mises à jour d'API.

## Conclusion : Pourquoi passer à l'approche "Contract-First" avec Jane ?

Au fil de cet article, nous avons vu que Jane n'est pas simplement un générateur de code, c'est un changement de paradigme dans la façon de concevoir la communication entre vos services. En plaçant le schéma OpenAPI au centre du jeu, vous gagnez sur quatre tableaux majeurs :

### Type Safety : L'armure de votre code

Fini les array mystérieux qui circulent d'un service à l'autre. Avec Jane, chaque donnée entrante ou sortante est encapsulée dans un objet PHP fortement typé.

Vous savez exactement ce que vous manipulez : des entiers sont des int, des dates sont des \DateTime. PHPStan et votre IDE vous remercient, et votre code devient auto-documenté par sa structure même.

### Documentation Vivante

Souvent, la documentation est le parent pauvre du projet : écrite au début, jamais mise à jour, et finalement trompeuse.

Ici, la spécification OpenAPI est votre Source de Vérité. Puisque c'est elle qui génère le code qui tourne en production, elle est de facto toujours à jour.

Couplée à des outils de visualisation comme **SwaggerUI** ou **ReDoc**, cette documentation devient interactive et fiable pour toutes les équipes (frontend, mobile, partenaires). Ce que vous voyez dans la doc est exactement ce que le code attend.

### Moins de Bugs en Production

En transformant les erreurs de runtime (données mal formées, champs manquants) en erreurs de "compile-time" (le code généré change, l'IDE signale l'erreur), vous capturez les bugs le plus tôt possible.

La validation automatique des réponses agit comme un filet de sécurité : aucune donnée corrompue ne peut pénétrer silencieusement dans votre système pour causer des erreurs en cascade plus loin.

### Une DX retrouvée

C'est peut-être le point le plus important au quotidien. Plus besoin de faire des allers-retours incessants entre le code et une documentation PDF externe.

* L'autocomplétion de l'IDE fonctionne instantanément.
* Les refactorings sont sûrs.
* Les tests sont plus simples à écrire et plus robustes.

### Performance accrue

Au-delà de la sécurité et du confort, le gain de performance est significatif. Le code généré par Jane étant spécifique à vos modèles, il est beaucoup plus performant que l'utilisation générique de l'`ObjectNormalizer` du Serializer de Symfony. En évitant d'utiliser la Reflection lors de l'exécution de votre code, vos applications consomment moins de ressources et répondent plus vite.

<u>Le mot de la fin :</u>
Arrêtez d'écrire vos clients HTTP et vos DTOs à la main. Laissez Jane faire, de manière optimisée, le travail répétitif et concentrez-vous sur ce qui a de la valeur : votre logique métier.

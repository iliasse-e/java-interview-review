# 🚀 Module 1 : Java Moderne, Gestion des Erreurs & Architecture Spring Boot


## 📌 1. L'API Optional : En finir avec le `NullPointerException`
Le repository renvoie un `Optional<T>` pour indiquer qu'une ressource est peut-être absente. Le service se charge de l'ouvrir ou de lever une exception métier si le résultat est vide.

* **Anti-pattern à éviter :** Récupérer un `null` via un `orElse(null)` pour faire un `if (obj == null)` juste après :

 ```java
Customer customer = customerRepository.findById(customerId).orElse(null);

if (customer == null) {
    throw new CustomerNotFoundException("Customer not found");
}
```

* **Bonne pratique :** Utiliser `.orElseThrow(...)` pour lever directement une exception métier en une seule ligne fluide.

```java
Customer customer = customerRepository.findById(customerId)
    .orElseThrow(() -> new CustomerNotFoundException("Customer not found"));

```

---

## 📌 2. Voyage et Hiérarchie des Exceptions dans Spring Boot

### Pourquoi on lève des Exception ?

Lever une exception, c'est poser un verrou de sécurité absolu pour empêcher ton application d'écrire des bêtises en base de données ou de corrompre l'état du système. C'est ce qu'on appelle le principe du *fail-fast* (échouer vite et fort dès qu'un problème survient).

### La Hiérarchie

* **`Error` :** Catastrophes système / JVM (ex: `OutOfMemoryError`). On ne les attrape **jamais**.
* **`Exception` :**
* *Checked Exceptions :* Obligent à un `try/catch` (vieille école, à éviter).
* *Unchecked / RuntimeException :* Le standard absolu de Spring Boot pour la logique métier (`CustomerNotFoundException`, etc.). Le compilateur n'oblige pas à les catcher.



### Le Parcours d'une Exception (Le "Bubbling Up")

1. Le **Repository** renvoie un Optional vide.
2. Le **Service** détecte l'anomalie et jette l'exception (`throw new ...`).
3. L'exception remonte la pile d'appels (*call stack*) en traversant le Controller, pour arriver tout en haut entre les mains de Spring.
4. Si rien n'est fait, Spring renvoie un vilain `500 Internal Server Error`. S'il y a un `@ControllerAdvice`, l'erreur est interceptée et traduite proprement en code HTTP.

---

## 📌 3. Le `@ControllerAdvice` et la Gestion Globale (`GlobalExceptionHandler`)

Le contrôleur ou l'exception métier ne doivent pas connaître le protocole HTTP. C'est le gestionnaire global qui fait la passerelle.

* **Exemple de Handler pour une erreur métier (HTTP 403 / 404) :**

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BalanceNotYetAvailableException.class)
    public ResponseEntity<String> handleBalanceNotYetAvailable(BalanceNotYetAvailableException ex) {
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(ex.getMessage());
    }
}

```

### Qui a la responsabilité d'être conscient des exceptions ? Les services ou le handler ?

La réponse est tranchée, et c'est la clé de voûte d'une architecture propre :

* **Le Service** a la responsabilité exclusive de DÉTECTER et DÉCLARER l'erreur métier.

  - Ce qu'il sait : Le service est le "cerveau" métier. C'est lui qui sait que "le solde est insuffisant" ou que "le client n'existe pas". Il prend la décision de jeter l'exception.

  - Ce qu'il ignore totalement : Le service ne sait pas ce qu'est le Web, le HTTP, ou un statut 404/400. Il ne doit jamais importer de classes liées à HTTP. S'il est appelé depuis une application mobile, un script en ligne de commande ou une API Web, son code doit rester exactement le même.

* **Le Handler (``@ControllerAdvice``)** a la responsabilité de TRADUIRE l'erreur pour le monde extérieur.

---

## 📌 4. Validation des DTOs & Interception des Erreurs de Formulaire

Quand un front-end (Angular) envoie un DTO annoté (ex: `@NotNull`), Spring intercepte les données *avant* d'entrer dans le contrôleur grâce à l'annotation `@Valid`.

* **Si une règle échoue :** Spring jette automatiquement une `MethodArgumentNotValidException`.
* **Le Handler associé pour formater l'erreur pour le front (HTTP 400) :**

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, Object>> handleValidationErrors(MethodArgumentNotValidException ex) {
    List<Map<String, String>> errors = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(error -> Map.of("field", error.getField(), "message", error.getDefaultMessage()))
        .toList();

    return ResponseEntity.badRequest().body(Map.of("errors", errors));
}

```

* **Résultat JSON envoyé au Front :** Un tableau propre associant chaque champ en erreur à son message explicatif.

```json
{
  "errors": [
    {
      "field": "email",
      "message": "ne doit pas être vide"
    },
    {
      "field": "amount",
      "message": "doit être supérieur à 0"
    }
  ]
}
```

# La Streams API (Le convoyeur de données moderne)

Avant Java 8, pour filtrer une liste, transformer ses objets et en récupérer une sous-liste, on écrivait des boucles `for` pleines de `if` et de listes temporaires. C'était verbeux, lourd et source de bugs.

Un **Stream**, ce n'est ni une structure de données (ce n'est pas une liste), ni un espace de stockage. **C'est un convoyeur (une chaîne de montage) sur lequel circulent tes données.**

### 1. L'anatomie d'un Stream en 3 temps

Un pipeline de Stream respecte toujours la même structure :


$$\text{Source} \rightarrow [\text{Opérations Intermédiaires}] \rightarrow \text{Opération Terminale}$$

1. **La Source :** On transforme une Collection (comme une `List`) en flux avec `.stream()`.
2. **Les Opérations Intermédiaires (Paresseuses) :** Ce sont les filtres et les transformateurs (ex: `.filter()`, `.map()`). Elles ne font **rien** tant que l'opération terminale n'est pas appelée. Elles renvoient un nouveau Stream.
3. **L'Opération Terminale (Déclencheuse) :** C'est elle qui lance la machine et récolte le résultat (ex: `.toList()`, `.count()`, `.findFirst()`). Une fois exécutée, le Stream est consommé et ne peut plus être réutilisé.

### 2. Le tiercé gagnant des méthodes indispensables :

* **`.filter(predicate)`** : Ne garde que les éléments qui respectent une condition (ex: ne garder que les comptes dont le solde est $> 0$).
* **`.map(function)`** : Transforme chaque élément du flux en un autre (ex: transformer une liste d'entités `BankAccount` en une liste de `BankAccountDTO`).
* **`.toList()`** : (Java 16+) Ramasse tous les éléments filtrés/transformés pour en refaire une belle `List` immuable.


#### Cas pratique

```java
    public List<BankAccountDTO> getActivatedAccounts() {
        return this.bankAccountList()
                .stream()
                .filter(bankAccountDTO -> bankAccountDTO.getStatus() == AccountStatus.ACTIVATED)
                .toList(); // Java 16+, si non c'était .collect(Collectors.toList());
    }
```

---

### 🎁 Les deux petits bonus syntaxiques demandés :

1. **Les `Map.of(...)` :**
C'est un raccourci moderne (introduit en Java 9) pour créer rapidement des petites Maps (clés/valeurs) sans s'embêter à faire des `new HashMap()` suivis de 10 lignes de `.put()`.
* *Exemple :* `Map.of("status", "SUCCESS", "code", 200)`


2. **Les méthodes de référence (`::`) :**
C'est du sucre syntaxique pour alléger les lambdas. Quand une lambda se contente d'appeler une méthode existante, tu peux la remplacer par `::`.
* *Lambda classique :* `customer -> customer.getName()`
* *Méthode de référence :* `Customer::getName` (Ça se lit : *"appelle la méthode getName sur chaque objet Customer du flux"*).


*La nuance subtile à garder en tête :*

    `ObjetInstance::nomDeLaMethode` (comme `dtoMapper::fromBankAccount`) : On appelle une méthode sur un objet précis qui existe déjà en mémoire (ton `dtoMapper`).

    `NomDeLaClasse::nomDeLaMethode` (comme `Integer::parseInt` ou `String::valueOf`) : On appelle une méthode statique (qui n'a pas besoin d'instance pour fonctionner).


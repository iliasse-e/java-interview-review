# 📚 Mini-Cours : La Pagination & le Tri avec Spring Data JPA (`Pageable`)

### 1. Le Problème (Pourquoi on ne fait pas de `findAll()` ?)

Imagine ton application e-banking. Au début, tu as 5 clients et 10 comptes, donc un `repository.findAll()` renvoie tout le monde sans problème.
Mais un an plus tard, ton appli gère 500 000 opérations bancaires. Si ton front-end Angular demande *"donne-moi toutes les opérations"*, il se passe trois catastrophes :

1. **Côté Base de données :** La requête SQL charge un demi-million de lignes d'un coup. Le disque et le CPU s'affolent.
2. **Côté Réseau :** Tu envoies un payload JSON de 50 Mo sur le réseau. Le navigateur du client freeze pendant 10 secondes.
3. **Côté Mémoire (JVM) :** Ton serveur Spring Boot risque un bel `OutOfMemoryError` si 10 utilisateurs font la même requête en même temps.

**La solution :** Ne jamais tout charger. On découpe les résultats en "pages" (ex: 20 éléments par page).

---

### 2. Les outils magiques de Spring : `Pageable` et `Page<T>`

Spring Data JPA gère la pagination nativement grâce à deux interfaces principales :

* **`Pageable` (Le contrat de la requête) :** C'est l'objet qui dit : *"Je veux la page numéro 0, avec 20 éléments par page, triés par date décroissante"*.
* **`Page<T>` (Le conteneur de la réponse) :** Ce n'est pas juste une liste. C'est un objet intelligent qui contient **la liste des éléments de la page**, mais aussi toutes les **métadonnées** vitales pour le front-end (le nombre total de pages, le nombre total d'éléments en base, si on est sur la dernière page ou non).

---

### 3. À quoi ça ressemble dans le code ?

#### Étape 1 : Le Repository

Ton repository hérite déjà de `JpaRepository`, donc il sait déjà paginer ! Il suffit de lui passer un `Pageable` en paramètre :

```java
public interface BankAccountRepository extends JpaRepository<BankAccount, String> {
    // Regarde le type de retour : Page<BankAccount> au lieu de List<BankAccount>
    Page<BankAccount> findByCustomerId(String customerId, Pageable pageable);
}

```

#### Étape 2 : Le Service

Le service se contente de transmettre le `Pageable` reçu du contrôleur au repository :

```java
public Page<BankAccountDTO> getAccountsByCustomer(String customerId, Pageable pageable) {
    Page<BankAccount> accountPage = bankAccountRepository.findByCustomerId(customerId, pageable);
    
    // On transforme chaque entité de la page en DTO (grâce au .map() propre aux objets Page !)
    return accountPage.map(dtoMapper::fromBankAccount);
}

```

*(Note : L'objet `Page` possède lui aussi une méthode `.map()` native, c'est un régal pour convertir une page d'entités en page de DTOs en une ligne !)*

#### Étape 3 : Le Controller

C'est là que la magie opère. Le contrôleur récupère automatiquement les paramètres de la requête HTTP :

```java
@GetMapping("/customers/{customerId}/accounts")
public Page<BankAccountDTO> getAccounts(
        @PathVariable String customerId,
        Pageable pageable // Spring va lire ?page=0&size=10&sort=createdAt,desc tout seul !
) {
    return bankAccountService.getAccountsByCustomer(customerId, pageable);
}

```

---

### 4. Vue côté Front-end (Angular)

Puisque tu as une casquette front, tu vas adorer ce que reçoit ton application Angular.
Quand tu fais un appel HTTP avec des query params de ce style :
`GET /api/customers/123/accounts?page=0&size=5&sort=balance,desc`

Spring renvoie un JSON structuré comme ça :

```json
{
  "content": [
    { "id": "ACC-1", "balance": 5000.0 },
    { "id": "ACC-2", "balance": 1200.0 }
  ],
  "totalPages": 15,
  "totalElements": 73,
  "size": 5,
  "number": 0,
  "first": true,
  "last": false
}

```

* **`content` :** Le tableau des données à afficher dans ton tableau ou ta liste Angular.
* **`totalPages` / `totalElements` :** Parfait pour afficher la pagination en bas de page (`Page 1 sur 15`).

---

### Est ce que dans le cas du Page, on doit renvoyer une erreur si on a pas de données ? 

Non, mais Spring renvoie ceci :

```json
{
  "content": [],
  "totalPages": 0,
  "totalElements": 0,
  "size": 10,
  "number": 0,
  "first": true,
  "last": true,
  "empty": true // c'est via cette prop qu'on peut réagir sur le front
}
```

Pour une collection paginée, un résultat vide n'est jamais une erreur HTTP 404. On renvoie un 200 OK avec une page vide (`content: []`, `totalElements: 0`), car la ressource (la liste) existe, elle est simplement sans éléments à l'instant T. La 404 est réservée aux entités uniques introuvables par identifiant.

### 🧠 Question pour valider le cours :

Si un développeur front-end t'appelle en disant : *"Dis donc, ton API de comptes me renvoie une `List<BankAccountDTO>` alors que j'ai besoin de savoir combien il y a de pages en total pour afficher mes boutons de pagination"*, **qu'est-ce que tu dois modifier dans ton code backend (du Controller jusqu'au Repository) pour résoudre son problème ?**

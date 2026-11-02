### 👹 C'est quoi, le problème du N+1 ?

Le problème du N+1 survient lorsqu'Hibernate exécute :

1. **1 requête** pour récupérer la liste principale (ex: les clients).
2. **N requêtes supplémentaires** (où *N* est le nombre de clients récupérés) pour aller chercher les données associées (ex: les comptes de chaque client).

Au lieu d'avoir **1 seule requête optimisée**, tu te retrouves avec **1 + N requêtes** envoyées à la base de données.

---

### 🎬 Exemple concret : Le piège en action

Imaginons que tu souhaites afficher la liste de tous tes clients avec leurs comptes bancaires.

#### Le code Java anodin que tu écris :

```java
List<Customer> customers = customerRepository.findAll(); // Récupère tous les clients

for (Customer customer : customers) {
    // Tu accèdes à la liste des comptes (chargée en LAZY par défaut)
    System.out.println("Client : " + customer.getName() + " a " + customer.getBankAccounts().size() + " comptes.");
}

```

#### Ce qui se passe réellement sous le capot (Les requêtes SQL générées) :

Si tu as **10 clients** en base (`N = 10`), regarde la catastrophe :

1. **Requête 1 (Le "1") :** Hibernate récupère la liste des clients.
```sql
SELECT * FROM customer;

```


2. **Requêtes 2 à 11 (Les "N") :** Pour *chaque* client, parce que tu demandes ses comptes et qu'ils sont en `LAZY`, Hibernate relance une requête SQL individuelle !
```sql
SELECT * FROM bank_account WHERE customer_id = 1;
SELECT * FROM bank_account WHERE customer_id = 2;
SELECT * FROM bank_account WHERE customer_id = 3;
-- ... et ainsi de suite jusqu'au 10ème client !

```



* **Résultat :** **11 requêtes SQL** pour afficher une simple liste. Si tu as 1 000 clients, Hibernate va exécuter **1 001 requêtes** d'un coup. La base de données sature, le réseau explose, et l'application rame.

---

### 🛡️ Comment résoudre le Piège du N+1 ?

Pour tuer ce problème, il faut forcer Hibernate à rapatrier toutes les données **en une seule fois** dès la première requête, en utilisant une jointure anticipée.

#### Solution 1 : Le `JOIN FETCH` en JPQL (La méthode reine)

Dans ton `CustomerRepository`, tu écris une requête personnalisée avec un `JOIN FETCH` :

```java
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    // Le "JOIN FETCH" dit à Hibernate : "Ramène le client ET ses comptes en une seule requête SQL !"
    @Query("SELECT c FROM Customer c LEFT JOIN FETCH c.bankAccounts")
    List<Customer> findAllWithAccounts();
}

```

* **Le SQL généré :**
Hibernate ne fait plus 11 requêtes, il en fait **une seule** avec un `LEFT JOIN` propre :
```sql
SELECT c.*, b.* FROM customer c LEFT JOIN bank_account b ON c.id = b.customer_id;

```


Le problème est réglé en un coup de baguette magique.

#### Solution 2 : L'annotation `@EntityGraph` (L'alternative propre)

Si tu ne veux pas écrire de requête JPQL à la main, tu peux utiliser un `@EntityGraph` directement au-dessus de ta méthode standard :

```java
@EntityGraph(attributePaths = {"bankAccounts"})
List<Customer> findAll();

```

Hibernate comprend qu'il doit charger la liste des comptes en même temps que les clients.

---

### 📝 Ce qu'il faut retenir pour tes entretiens :

> *"Le piège du N+1 arrive avec le chargement `LAZY` lorsqu'on parcourt une collection d'enfants à partir d'une liste de parents. L'ORM génère une requête initiale pour les parents, puis N requêtes individuelles pour chaque enfant. On le résout en utilisant des `JOIN FETCH` en JPQL ou des Entity Graphs pour forcer une jointure SQL unique."*

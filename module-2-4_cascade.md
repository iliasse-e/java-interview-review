## 1. Le piège de la cascade (`CascadeType.ALL` vs Suppression)

C'est un classique des questions pièges en entretien.
Imaginons que tu supprimes un client (`customerRepository.delete(customer)`). Que se passe-t-che en base de données pour ses comptes bancaires ?

* **Sans Cascade :** Si tu tentes de supprimer le client, la base de données va refuser (ou lever une erreur de contrainte d'intégrité SQL) car des comptes pointent encore vers lui (`customer_id` est bloqué par une clé étrangère).
* **Avec `CascadeType.ALL` (ou `CascadeType.REMOVE`) :** Tu dis à JPA : *"Si je supprime le client, supprime aussi automatiquement tous ses comptes associés en cascade."*
* **Le danger :** Si tu fais un `customerService.deleteCustomer(id)` par erreur ou sans réfléchir, tu effaces d'un coup sec tous les comptes et potentiellement tout l'historique financier lié. En entreprise, sur des données sensibles (comme la banque ou l'e-commerce), on évite le `CascadeType.ALL` aveugle sur les suppressions, ou on gère ça proprement par des règles métiers.

---

## 2. 🛠️ Exemple concret de Cascade dans ton application

Dans ton `e-banking-backend`, tu as une relation entre `Customer` et `BankAccount`.
Si tu veux que la suppression ou la sauvegarde d'un client se propage automatiquement à ses comptes, tu configures l'annotation `@OneToMany` avec l'attribut `cascade`.

### Dans ton entité `Customer.java` :

```java
@Entity
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;

    // Le cascade et orphanRemoval en action :
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<BankAccount> bankAccounts = new ArrayList<>();
    
    // getters, setters...
}

```

### 🔍 Qu'est-ce que ça fait sous le capot ?

1. **`CascadeType.ALL`** :
* Si tu sauves un client (`customerRepository.save(customer)`) qui contient une liste de nouveaux comptes, Hibernate va **automatiquement sauvegarder le client ET insérer tous ses comptes** en base de données en une seule opération. Plus besoin de faire un `save()` manuel pour chaque compte !
* Si tu supprimes le client (`customerRepository.delete(customer)`), Hibernate va automatiquement supprimer tous les comptes qui lui appartiennent avant de supprimer le client, évitant ainsi l'erreur de clé étrangère bloquante.


2. **`orphanRemoval = true`** :
* Si tu retires un compte de la liste Java d'un client (`customer.getBankAccounts().remove(account)`) et que tu sauves le client, Hibernate comprend que ce compte est devenu un "orphelin" et va **le supprimer automatiquement de la base de données**.



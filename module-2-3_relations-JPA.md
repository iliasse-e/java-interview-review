# 📚 Les Relations JPA et leur réalité en SQL

En Java, on manipule des objets reliés entre eux. En SQL, on manipule des tables reliées par des clés étrangères. Le rôle de Hibernate (JPA) est de faire la passerelle entre les deux.

## 1. La relation Bidirectionnelle classique : `@OneToMany` et `@ManyToOne`

Prenons ton exemple : un **`Customer`** possède plusieurs **`BankAccount`**, et un **`BankAccount`** appartient à un seul **`Customer`**.

### 🛠️ Côté Base de Données (SQL) : Où se cache la clé étrangère ?

En SQL, une relation 1-N (One-to-Many) **ne peut se matérialiser que d'un seul côté** : du côté de la table qui "subit" la relation, c'est-à-dire la table "Many" (la table `bank_account`).

```text
Table [customer]                      Table [bank_account]
+----+--------+                       +---------+------------+-------------+
| id | name   |                       | id      | balance    | customer_id | ---> Clé étrangère !
+----+--------+                       +---------+------------+-------------+
|  1 | Hassan | <==================== | "ACC-1" |    5000    |      1      |
+----+--------+                       +---------+------------+-------------+

```

* **Règle absolue SQL :** La table `customer` **ne contient aucune colonne** faisant référence aux comptes. Le lien est uniquement porté par la colonne `customer_id` dans la table `bank_account`.

---

### ☕ Côté Java (JPA) : Le rôle mystère de `mappedBy`

Puisque le monde Java est bidirectionnel (le client a une liste de comptes, et le compte a un client), il faut dire à Hibernate comment relier ces deux entités sans créer de doublon en base. C'est là qu'intervient **`mappedBy`**.

Regarde tes entités :

**Dans l'entité `Customer` :**

```java
@Entity
public class Customer {
    @Id
    private Long id;
    
    // Le mappedBy="customer" dit à Hibernate : 
    // "Ne cherche pas de colonne 'bankAccounts' dans la table customer. 
    // La vraie clé étrangère est gérée de l'autre côté, dans l'attribut 'customer' de BankAccount."
    @OneToMany(mappedBy = "customer")
    private List<BankAccount> bankAccounts;
}

```

**Dans l'entité `BankAccount` :**

```java
@Entity
public class BankAccount {
    @Id
    private String id;

    // C'est ICI que le lien physique se fait en base via la clé étrangère
    @ManyToOne
    @JoinColumn(name = "customer_id") // Crée la colonne customer_id en SQL
    private Customer customer;
}

```

* **Ce qu'il faut retenir :**
* Le côté **propriétaire** de la relation (celui qui écrit la clé étrangère en base) est toujours annoté avec `@ManyToOne` et `@JoinColumn`.
* Le côté **esclave / inverse** est annoté avec `@OneToMany(mappedBy = "...")`. Si tu oublies le `mappedBy`, Hibernate va paniquer et essayer de créer une table de jointure inutile en base de données.






### Rappel : Le rapport `OneToMany` / `ManyToOne` existe-t-il en SQL ? Ou bien seulement en Java et UML ?

* En **UML / Java**, c'est une relation bidirectionnelle riche et texturée : *un client connaît ses comptes, et un compte connaît son client*.
* En **SQL (relationnel pur)**, cette notion d'objet n'existe pas. Le SGBD ne connaît que des lignes, des tables et des **clés étrangères (`FOREIGN KEY`)**.

#### Comment le SGBD la représente et la retrouve ?

Le SGBD se fiche totalement de savoir qu'il y a un `OneToMany` ou un `ManyToOne`. Tout ce qu'il voit, c'est une simple colonne `customer_id` dans la table `bank_account`.

Pour retrouver qui est le client d'un compte, ou quels sont les comptes d'un client, la base de données fait simplement une **jointure (`JOIN`)** en se basant sur cette valeur :

```sql
-- "Donne-moi tous les comptes du client dont l'ID est 1"
SELECT * FROM bank_account WHERE customer_id = 1;

```

C'est magique : Hibernate lit ton `@ManyToOne` en Java, traduit ça par cette requête SQL sous le capot, et recrée l'objet `Customer` en mémoire pour toi.

---




---

## 2. Le piège de la cascade (`CascadeType.ALL` vs Suppression)

C'est un classique des questions pièges en entretien.
Imaginons que tu supprimes un client (`customerRepository.delete(customer)`). Que se passe-t-che en base de données pour ses comptes bancaires ?

* **Sans Cascade :** Si tu tentes de supprimer le client, la base de données va refuser (ou lever une erreur de contrainte d'intégrité SQL) car des comptes pointent encore vers lui (`customer_id` est bloqué par une clé étrangère).
* **Avec `CascadeType.ALL` (ou `CascadeType.REMOVE`) :** Tu dis à JPA : *"Si je supprime le client, supprime aussi automatiquement tous ses comptes associés en cascade."*
* **Le danger :** Si tu fais un `customerService.deleteCustomer(id)` par erreur ou sans réfléchir, tu effaces d'un coup sec tous les comptes et potentiellement tout l'historique financier lié. En entreprise, sur des données sensibles (comme la banque ou l'e-commerce), on évite le `CascadeType.ALL` aveugle sur les suppressions, ou on gère ça proprement par des règles métiers.

---

### 🧠 Question pour vérifier ton ancrage :

Dans ton `e-banking-backend`, est-ce que tu as configuré des cascades ou des `mappedBy` sur tes entités `Customer` et `BankAccount` ? Si tu devais expliquer à un collègue pourquoi on met `mappedBy`, tu lui dirais quoi en une phrase ?

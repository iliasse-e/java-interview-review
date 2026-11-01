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


## 2. La relation One-to-One (1-1) : L'exclusivité

### Le cas d'usage :

Imaginons un **`Customer`** et son **`CustomerProfile`** (les informations détaillées de profil, comme une bio, une photo et des préférences). Un client possède un et un seul profil, et un profil appartient à un et un seul client.

### 🛠️ Comment ça se matérialise en SQL ?

En SQL, une relation 1-1 se gère en plaçant une **clé étrangère (`FOREIGN KEY`) unique** dans l'une des deux tables (généralement du côté de la table dépendante). Pour garantir qu'un même profil ne soit pas partagé par plusieurs clients, on ajoute une contrainte **`UNIQUE`** sur cette clé étrangère.

```text
Table [customer]                     Table [customer_profile]
+----+--------+                      +----+------------------+-------------+
| id | name   |                      | id | bio              | customer_id | ---> FK + UNIQUE
+----+--------+                      +----+------------------+-------------+
|  1 | Hassan | <=================== | 10 | "Développeur..." |      1      |
+----+--------+                      +----+------------------+-------------+

```

* **Le piège JPA :** C'est le côté qui porte la clé étrangère (`customer_profile`) qui est le "propriétaire" de la relation. L'autre côté utilisera `mappedBy = "customer"`, exactement comme pour le `OneToMany`.

---

## 3. La relation Many-to-Many (N-N) : Le club des multiples

### Le cas d'usage :

Imaginons des **`Student`** (Étudiants) et des **`Course`** (Cours). Un étudiant peut suivre plusieurs cours, et un cours contient plusieurs étudiants.

### 🛠️ Comment ça se matérialise en SQL ?

Une relation N-N est **impossible à stocker directement** dans deux tables simples, car une colonne ne peut pas contenir plusieurs valeurs à la fois (on ne peut pas mettre une liste d'IDs dans une seule case SQL).

La solution universelle en SGBD relationnel, c'est de créer une **Table de Jonction (Join Table)** intermédiaire. Cette table ne contient que des paires de clés étrangères :

```text
Table [student]                    Table [enrollment] (Table de jonction)      Table [course]
+----+---------+                   +------------+-----------+                   +----+---------------+
| id | name    |                   | student_id | course_id |                   | id | title         |
+----+---------+                   +------------+-----------+                   +----+---------------+
|  1 | Hassan  | <---------------> |     1      |    101    | <---------------- |101 | Spring Boot   |
|  2 | Yassine | <---------------> |     2      |    101    |                   |102 | Angular Avancé|
+----+---------+                   +------------+-----------+                   +----+---------------+

```

* **Ce qui se passe en SQL :**
Pour savoir quels cours suit Hassan, le SGBD fait une double jointure : il part de `student`, passe par la table de jonction `enrollment`, et arrive sur `course`.
* **Côté Java (JPA) :**
C'est ici qu'intervient l'annotation `@ManyToMany` accompagnée d'un `@JoinTable` pour configurer le nom de cette table de jonction intermédiaire.

---

### 📊 Tableau Récapitulatif pour tes Entretiens

| Type de Relation | Traduction SQL physique | Élément clé en base |
| --- | --- | --- |
| **`Many-to-One` / `One-to-Many**` | 2 tables | Une **clé étrangère** dans la table "Many". |
| **`One-to-One`** | 2 tables | Une **clé étrangère unique (`UNIQUE`)** dans l'une des tables. |
| **`Many-to-Many`** | **3 tables** | Une **table de jonction** dédiée qui porte deux clés étrangères. |

---



## ☕ Côté Java (JPA) : Le rôle mystère de `mappedBy`

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

### 1. Avec `mappedBy` (La relation propre et maîtrisée)

Tu indiques explicitement : *"C'est l'attribut `customer` dans `BankAccount` qui pilote la clé étrangère."*

**En Base de Données (SQL) :**

```text
Table [customer]                     Table [bank_account]
+----+--------+                      +---------+------------+-------------+
| id | name   |                      | id      | balance    | customer_id | ---> La clé étrangère
+----+--------+                      +---------+------------+-------------+
                                     | "ACC-1" |    5000    |      1      |
                                     +---------+------------+-------------+

```

* **Résultat :** Exactement 2 tables. Le lien est matérialisé proprement par une unique colonne `customer_id` dans la table `bank_account`.

---

### 2. Sans `mappedBy` (Le piège de la table de jointure fantôme)

Si tu écris `@OneToMany` dans `Customer` et `@ManyToOne` dans `BankAccount`, mais que tu **omets** le `mappedBy = "customer"` :
JPA se dit : *"Ah, les deux côtés gèrent la relation de manière indépendante, je ne sais pas qui est le chef."* Par défaut, il décide de créer une **table de jointure intermédiaire** (comme pour un ManyToMany).

**En Base de Données (SQL) :**

```text
Table [customer]       Table [customer_bank_accounts] (Table fantôme !)       Table [bank_account]
+----+--------+        +-------------+------------------+                   +---------+------------+
| id | name   |        | customer_id | bank_account_id  |                   | id      | balance    |
+----+--------+        +-------------+------------------+                   +---------+------------+
|  1 | Hassan | <----> |      1      | "ACC-1"          | <---------------- | "ACC-1" |    5000    |
+----+--------+        +-------------+------------------+                   +---------+------------+

```

* **Résultat :** Hibernate va générer **3 tables** au lieu de 2 ! Il crée une table de jointure supplémentaire (`customer_bank_accounts`) qui fait double emploi avec la colonne `customer_id` déjà présente dans `bank_account`. C'est lourd, inutile, et ça génère des requêtes SQL complexes et inefficaces pour rien.

---

### Qu'en est-il des requêtes SQL ?

Ton intuition était en revanche tout à fait vraie concernant le comportement d'Hibernate :

* **Avec `mappedBy**`, Hibernate utilise directement la clé étrangère `customer_id` pour faire ses `JOIN` de manière ultra-optimisée.
* **Sans `mappedBy**`, Hibernate va devoir faire des requêtes supplémentaires pour aller piocher les IDs dans cette fameuse table de jointure intermédiaire, ce qui multiplie les allers-retours inutiles avec la base de données.

### En résumé :

Le `mappedBy` n'est pas qu'un simple mot-clé cosmétique en Java : il dit à Hibernate **où regarder en base de données** pour ne pas s'inventer des tables de jointure qui n'ont pas lieu d'être.



---

### Requêtes SQL : Avec et sans `mappedBy`


Voici concrètement ce qu’Hibernate génère en SQL dans les deux cas, lorsque tu souhaites **récupérer un client et tous ses comptes associés** (par exemple lors d’un `findById` ou d’un chargement complet).

---

### 1. 🟢 AVEC `mappedBy` (Le SQL Propre et Direct)

Puisque la relation est clairement définie (la clé étrangère `customer_id` est directement dans la table `bank_account`), Hibernate va droit au but avec une simple jointure (`JOIN`).

**La requête SQL générée par Hibernate :**

```sql
SELECT 
    c.id AS customer_id, 
    c.name AS customer_name, 
    b.id AS account_id, 
    b.balance AS account_balance, 
    b.customer_id AS fk_customer 
FROM customer c 
LEFT JOIN bank_account b ON c.id = b.customer_id 
WHERE c.id = 1;

```

* **Ce qui se passe :**
* 1 seule requête SQL.
* Hibernate traverse directement la table `customer` vers la table `bank_account` en utilisant la colonne `customer_id`. C'est ultra rapide et performant.



---

### 2. 🔴 SANS `mappedBy` (Le SQL Lourd avec la Table Fantôme)

Puisque tu as oublié de mettre `mappedBy`, Hibernate crée une table de jointure intermédiaire (`customer_bank_accounts`) pour lier les deux tables, même si la colonne `customer_id` existe déjà dans `bank_account`.

Pour récupérer le client et ses comptes, Hibernate doit donc faire **deux jointures en cascade** : traverser la table intermédiaire pour arriver au compte.

**La requête SQL générée par Hibernate :**

```sql
SELECT 
    c.id AS customer_id, 
    c.name AS customer_name, 
    b.id AS account_id, 
    b.balance AS account_balance 
FROM customer c 
LEFT JOIN customer_bank_accounts cba ON c.id = cba.customer_id            -- 1ère jointure inutile
LEFT JOIN bank_account b ON cba.bank_account_id = b.id                    -- 2ème jointure inutile
WHERE c.id = 1;

```

---

### 3. Pire encore : L'impact sur les écritures (INSERT / UPDATE)

Le problème ne concerne pas seulement la lecture (`SELECT`), mais aussi l'écriture (`INSERT`) lorsque tu ajoutes un compte à un client.

* **Avec `mappedBy` :**
Hibernate fait un simple `INSERT` dans la table `bank_account` en remplissant la colonne `customer_id = 1`. C'est propre.
* **Sans `mappedBy` :**
Hibernate va faire ton `INSERT` dans `bank_account`, **ET** il va en plus exécuter un `INSERT` dans la table de jointure `customer_bank_accounts` (`INSERT INTO customer_bank_accounts (customer_id, bank_account_id) VALUES (1, 'ACC-1')`).

### Bilan en entretien :

> *"Oublier le `mappedBy` force JPA à créer une table de jointure superflue. Cela alourdit non seulement le schéma de la base de données, mais multiplie aussi les jointures inutiles sur les requêtes de lecture et génère des requêtes d'écriture redondantes à chaque modification d'association."*


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

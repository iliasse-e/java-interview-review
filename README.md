
### 🔥 Plan Commando (3 Semaines - Accélération & Hardcore)

#### **Semaine 1 (1 à 2 jours max) : Java Moderne & Le socle Spring Boot**

* **Ce qu'on survole rapidement (puisque tu connais) :** L'IoC, l'injection par constructeur, les controllers REST de base.
* **Ce qu'on bosse à fond (Le Java Moderne & Pièges) :**
* **Java 17/21 :** Les `records` (immuabilité propre), le *pattern matching* dans les `switch`, et l'utilisation rigoureuse des `Optional<T>` pour éradiquer les `NullPointerException`.
* **Streams API & Lambdas :** Maîtriser le triptyque `map`, `filter`, `reduce`, `collect` sans hésiter (indispensable pour manipuler les collections en backend).
* La gestion propre des erreurs métiers (`@ControllerAdvice` et exceptions custom).



#### **Semaine 2 (3 à 4 jours) : JPA Avancé, Relations & Le Piège du N+1**

* **Ton point noir identifié :** Les relations entre entités et le SQL sous-jacent.
* **Le programme de la semaine :**
* Rappel sur la pagination (`Page` & `Pageable`).
* @JSONIgnore de Jackson et le role d'une DTO.
* Le role de FETCH.Lazy vs FETCH.Eager.
* Démystifier une bonne fois pour toutes les relations : `@OneToMany`, `@ManyToOne`, `@ManyToMany` (comment ça se traduit en clés étrangères en base, le rôle du `mappedBy`, et surtout le piège de la cascade `CascadeType.ALL` vs suppression).
* **Le problème du N+1 :** Comprendre pourquoi Hibernate génère 50 requêtes SQL là où tu en voulais une seule, et comment le résoudre (`JOIN FETCH`, entités graph).
* Introduction au **Pattern Strategy** (que tu as mentionné) appliqué à la logique métier pour éviter les services monstres remplis de `if/else`.



#### **Semaine 3 (2 semaines) : L'Affrontement Final (Sécurité, Keycloak, Tests & Docker)**

*C'est le gros morceau, celui qui demande le plus de temps.*

* **Spring Security 6 & Keycloak (5-6 jours) :**
* Déploiement de Keycloak via Docker.
* Configuration du flux OIDC, des rôles, et passage de ton API Spring Boot en *Resource Server* pour valider les JWT.


* **Tests Unitaires & d'Intégration (3-4 jours) :**
* JUnit 5 et Mockito pour les tests unitaires.
* **Testcontainers** : Lancer un vrai PostgreSQL dans Docker pour tester tes repositories et controllers en conditions réelles (l'argument massue en entretien).


* **Le bouclage Docker-Compose :** Assembler le tout (Front Angular + Spring Boot + Keycloak + PostgreSQL).

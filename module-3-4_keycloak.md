
### 1. Qu'est-ce que Keycloak et pourquoi on l'utilise ?

Keycloak est un serveur de **gestion des identités et des accès (IAM)** qui implémente les standards **OAuth2** (autorisation / accès délégué) et **OpenID Connect (OIDC)** (authentification / identité par-dessus OAuth2).

Sans Keycloak, chaque application ou microservice doit gérer lui-même :

* Le stockage des utilisateurs, le hachage des mots de passe, les règles de complexité et la réinitialisation
* Les écrans de connexion, la déconnexion et le cycle de vie des sessions
* La connexion à des annuaires ou fournisseurs externes (Google, LDAP, Active Directory)
* L'émission, le renouvellement et la révocation des jetons

Keycloak centralise tout cela dans un service dédié. Ton API Spring Boot n'a plus à fabriquer de jetons (elle cesse d'agir comme serveur d'autorisation maison) : elle devient un **Resource Server** qui valide les jetons émis par Keycloak.

---

### 2. Les concepts clés que tu vas manipuler dans Keycloak

| Concept | Rôle | Analogie concrète |
| --- | --- | --- |
| **Realm** | Un espace étanche qui regroupe ses propres utilisateurs, rôles, clients et clés de signature. On n'utilise pas le realm `master` pour les utilisateurs métier. | Un bâtiment ou un périmètre isolé avec son propre système de badges. |
| **Client** | Une application ou un service qui dialogue avec Keycloak (un front Angular, un back-end, ou un outil de test type Postman). | Un lecteur de badge ou une borne autorisée à demander un accès. |
| **User** | Un compte rattaché à un realm, avec ses attributs, son mot de passe (ou double facteur) et ses rôles. | La personne qui possède le badge. |
| **Role** | Un profil ou une permission nommée (`ADMIN`, `CLIENT`, `AUDITOR`). Peut être défini au niveau du **Realm** (partagé) ou d'un **Client** (spécifique). | Les niveaux d'accès inscrits sur le badge. |
| **Token (JWT)** | Un jeton JSON signé par Keycloak : **Access Token** (courte durée, envoyé à l'API) et **Refresh Token** (durée plus longue, pour obtenir un nouvel access token). | Un laissez-passer horodaté vérifié par l'API à chaque appel. |

---

### 3. Comment l'authentification et l'appel à l'API fonctionnent ensemble

1. **Connexion et obtention des jetons :** Le client (par exemple une appli Angular ou une requête de test) s'authentifie auprès de Keycloak sur le bon **Realm** et le bon **Client**. Keycloak vérifie l'identité et renvoie des jetons, dont un **Access Token** au format JWT.
2. **Appel à l'API Spring Boot :** Le client envoie l'en-tête HTTP `Authorization: Bearer <access-token>`.
3. **Vérification du jeton (Resource Server) :** Spring Boot vérifie la signature cryptographique grâce aux clés publiques exposées par Keycloak (**JWKS**), contrôle l'émetteur (`iss`) et l'expiration (`exp`). Si le jeton est invalide ou expiré, la requête est rejetée en `401 Unauthorized`.
4. **Mapping des rôles et autorisation :** Spring Security transforme le JWT en objet `Authentication` dans le `SecurityContext`, puis évalue les règles d'accès sur les routes (`authorizeHttpRequests`) ou sur les méthodes (`@PreAuthorize`). Si l'utilisateur est authentifié mais n'a pas le bon rôle, l'API répond `403 Forbidden`.

---

### 4. Les pièges classiques à anticiper

* **L'emplacement des rôles dans le JWT :** Par défaut, Keycloak range les rôles de realm dans `realm_access.roles` (et non à la racine sous `"roles"`). Il faudra configurer un convertisseur côté Spring Boot pour lire ce chemin et produire les autorités attendues (`ROLE_...` ou `hasAuthority(...)`).
* **La cohérence de l'émetteur (`iss`) :** La propriété `spring.security.oauth2.resourceserver.jwt.issuer-uri` côté Spring Boot doit correspondre exactement à l'URL d'émetteur présente dans le jeton Keycloak (par exemple `http://localhost:8080/realms/ebanking-realm`).
* **Type de client (public vs confidentiel) :** Une application web exécutée dans le navigateur (comme Angular) est un client public (sans secret embarqué côté client), tandis qu'un échange serveur-à-serveur utilise un client confidentiel avec secret.

Une fois ces repères bien en tête, l'enchaînement dans la console Keycloak (`Realm` -> `Client` -> `Roles` -> `User`) puis dans Spring Boot devient très lisible.

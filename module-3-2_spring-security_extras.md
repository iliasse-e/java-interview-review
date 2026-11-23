Voici trois mécanismes concrets qui reviennent très souvent en production et en entretien :

### 1. Le piège du préfixe `ROLE_` (`hasRole` vs `hasAuthority`)

Dans Spring Security, la façon dont les rôles sont nommés et vérifiés prête régulièrement à confusion :

* **`hasRole("ADMIN")`** attend automatiquement une autorité nommée **`ROLE_ADMIN`**. Spring ajoute le préfixe `ROLE_` sous le capot.
* **`hasAuthority("ROLE_ADMIN")`** vérifie la chaîne exacte telle quelle, sans préfixe ajouté.
* **En pratique :** Si ta base ou ton token contient `"ADMIN"` sans `ROLE_`, `hasRole("ADMIN")` renverra un `403 Forbidden`. C'est l'un des premiers points à surveiller quand on mappe des rôles issus d'une base ou d'un fournisseur externe (comme Keycloak).

### 2. CORS (Cross-Origin Resource Sharing) et Spring Security

Quand un front-end (par exemple sur `http://localhost:4200`) appelle une API Spring Boot sur un autre port ou domaine (`http://localhost:8080`), le navigateur envoie d'abord une requête préparatoire **`OPTIONS` (preflight)**.

* Si CORS est configuré uniquement sur un contrôleur (`@CrossOrigin`) mais pas dans le `SecurityFilterChain`, Spring Security peut bloquer la requête `OPTIONS` avant même d'atteindre le contrôleur.
* **La bonne pratique :** Activer `.cors(Customizer.withDefaults())` dans le `SecurityFilterChain` et exposer un bean `CorsConfigurationSource` centralisé, pour que la couche sécurité laisse passer les requêtes `OPTIONS` légitimes et applique les bons en-têtes.

### 3. `SecurityContextHolder` et threads asynchrones

Le `SecurityContext` est généralement stocké dans un `ThreadLocal` : il est attaché au thread qui traite la requête HTTP.

* Tant que tout s'exécute de manière synchrone sur le même thread (filtre -> contrôleur -> service -> repository), l'authentification suit naturellement.
* **La subtilité :** Si tu lances un traitement asynchrone (`@Async`, `CompletableFuture`, nouveau thread), ce nouveau thread ne récupère pas automatiquement le contexte de sécurité sauf configuration adaptée (par exemple un `DelegatingSecurityContextExecutor` ou une stratégie de mode adaptée).


### À quoi ressemble un objet `FilterChain` ?

Dans la configuration Spring Boot, tu manipules un `SecurityFilterChain` (souvent construit via `HttpSecurity.build()`), et à l'exécution Servlet/Spring, chaque filtre reçoit un objet `FilterChain` (par exemple `filterChain.doFilter(request, response)`).

Concrètement, un `FilterChain` n'est pas un gros bloc monolithique : **c'est une liste ordonnée de filtres** (des objets implémentant `Filter` / `OncePerRequestFilter`) reliés entre eux comme des maillons.

À quoi ça ressemble conceptuellement ?

```text
Requête HTTP (request, response)
   │
   ▼
[ Filter 1 : CORS / Headers ] ──▶ chain.doFilter(req, res)
   │
   ▼
[ Filter 2 : BearerTokenAuthenticationFilter / JwtAuthorizationFilter ] ──▶ chain.doFilter(req, res)
   │
   ▼
[ Filter 3 : AuthorizationFilter ] ──▶ chain.doFilter(req, res)
   │
   ▼
[ DispatcherServlet -> @RestController ]

```

* Chaque filtre a une **position/priorité** définie dans la chaîne.
* Chaque filtre décide s'il laisse passer la requête au maillon suivant en appelant `filterChain.doFilter(request, response)`, ou s'il **court-circuite** la chaîne (par exemple en renvoyant une `401` ou `403` et en s'arrêtant là).
* Si un filtre **oublie** d'appeler `filterChain.doFilter(request, response)` sans avoir envoyé de réponse d'erreur, la requête reste "bloquée" en chemin et n'atteint jamais le contrôleur. (C'est d'ailleurs un bug classique quand on écrit un `if` d'exclusion sans `return` ou sans `else`.)

---

### Ce qu'il faut savoir de plus sur la `FilterChain`

#### A. Il existe deux chaînes de filtres distinctes dans une appli Spring Boot

On les confond souvent au début :

1. **La chaîne de filtres Servlet / Web globale :** Gérée par le conteneur web (Tomcat) et Spring. Elle contient des filtres techniques applicatifs (encodage, logs, CORS, etc.).
2. **La Spring Security `FilterChainProxy` :** C'est un filtre spécial qui s'insère dans la chaîne globale et qui délègue la requête à **une ou plusieurs `SecurityFilterChain**` définies dans ta configuration Spring.

#### B. Tu peux avoir plusieurs `SecurityFilterChain` dans une même application

C'est très pratique quand une application expose plusieurs types d'endpoints avec des règles différentes. Par exemple :

* Une `SecurityFilterChain` avec `@Order(1)` et `.securityMatcher("/api/**")` en mode **stateless** (JWT / OAuth2 Resource Server) pour ton API REST.
* Une seconde `SecurityFilterChain` avec `@Order(2)` pour d'autres routes (par exemple une interface d'admin ou une page de statut) avec d'autres règles d'accès.

Spring évalue ces chaînes dans l'ordre (`@Order`) et applique la **première** dont le `securityMatcher` correspond à l'URL de la requête entrante.

#### C. L'ordre des filtres est critique

Dans Spring Security, l'ordre d'exécution compte énormément :

* Les filtres qui **authentifient** (qui lisent un token ou des identifiants et remplissent le `SecurityContext`) doivent s'exécuter **avant** les filtres qui **autorisent** (qui vérifient si l'utilisateur a le droit d'accéder à la route).
* C'est pour cette raison qu'avec des filtres custom on utilise des méthodes comme `addFilterBefore(..., UsernamePasswordAuthenticationFilter.class)` ou `addFilterAfter(...)` : pour placer le filtre au bon endroit dans la séquence.


### C'est quoi `@Order(1)` et `@Order(2)` ?

Quand tu déclares **plusieurs beans `SecurityFilterChain**` dans une application Spring Boot, Spring doit savoir **dans quel ordre les évaluer** pour chaque requête HTTP entrante.

* `@Order(...)` donne une priorité numérique : **plus le chiffre est petit, plus la priorité est haute** (`@Order(1)` passe avant `@Order(2)`).
* Spring Security prend la **première** `SecurityFilterChain` dont la condition (`securityMatcher(...)`) correspond à l'URL de la requête, puis ignore les suivantes pour cette requête.
* Si tu oublies `@Order(...)` sur plusieurs chaînes, l'ordre de chargement des beans n'est pas garanti de façon explicite : une chaîne trop large (ex: `anyRequest()`) pourrait intercepter des routes avant une chaîne plus spécifique.

---

### Exemple de deux `SecurityFilterChain` distinctes

Imagine une application qui expose :

1. Une **API REST** (`/api/**`) consommée par un client Angular en mode **stateless** (sans session HTTP, protégée par JWT / Bearer token).
2. Des **routes d'administration ou de monitoring** (`/admin/**`) avec une autre politique (par exemple un accès restreint ou une configuration séparée).

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class MultiSecurityConfig {

    // 1. Chaîne prioritaire pour l'API REST stateless
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }

    // 2. Chaîne de repli pour le reste de l'application
    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}

```

* Si une requête arrive sur `/api/accounts`, `@Order(1)` matche `/api/**` et applique la chaîne **stateless**.
* Si une requête arrive sur `/admin/users`, `@Order(1)` ne matche pas, Spring passe donc à `@Order(2)` et applique la seconde chaîne.

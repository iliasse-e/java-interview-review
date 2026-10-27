# 🚀 Module 1 : Java Moderne, Gestion des Erreurs & Architecture Spring Boot


## 📌 1. L'API Optional : En finir avec le `NullPointerException`
Le repository renvoie un `Optional<T>` pour indiquer qu'une ressource est peut-être absente. Le service se charge de l'ouvrir ou de lever une exception métier si le résultat est vide.

* **Anti-pattern à éviter :** Récupérer un `null` via un `orElse(null)` pour faire un `if (obj == null)` juste après.

* ```java
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


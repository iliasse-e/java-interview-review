Plongeons dans le cambouis. **Spring Core**, c'est le moteur thermique de toute la formule. Sans lui, aucune autre brique (ni Data, ni Web, ni Security) ne peut tourner.

Pour bien le maîtriser, il faut décomposer Spring Core en 3 grands piliers que tu vas manipuler tous les jours en entreprise : **les Beans**, **l'Injection de Dépendances**, et **le Component Scanning**.

---

### 1. Qu'est-ce qu'un "Bean" exactement ?

Dans le jargon Spring, un **Bean** n'est rien d'autre qu'un objet Java dont la création, le cycle de vie et la destruction sont pilotés par le conteneur Spring (l' `ApplicationContext`), et non par toi avec un `new MonObjet()`.

* **Le scope (la portée) par défaut : Le Singleton**
Par défaut, un Bean dans Spring est un **Singleton**. Cela signifie que Spring n'instancie ta classe **qu'une seule fois** au démarrage de l'application. Ensuite, chaque fois qu'un autre composant a besoin de ce Bean, Spring lui redonne exactement la même référence en mémoire. Ça évite de gaspiller de la mémoire à recréer des objets en boucle.
* **L'autre scope important : Le Prototype**
Si tu demandes explicitement un scope `Prototype` (via `@Scope("prototype")`), Spring va recréer une nouvelle instance de l'objet *à chaque fois* qu'on le lui demande.

---

### 2. Les 3 façons d'injecter des dépendances (et le grand gagnant)

Puisque c'est Spring qui gère les objets (l'IoC), il faut bien dire à ton code comment récupérer les dépendances. Il y a 3 méthodes historiques.

#### A. L'injection par champ (`@Autowired` sur l'attribut) — ❌ *À bannir*

```java
@Service
public class CustomerService {
    @Autowired // 🚫 ANTI-PATTERN
    private CustomerRepository customerRepository;
}

```

* **Pourquoi c'est nul :** Ça a l'air magique et court, mais c'est un cauchemar pour les tests unitaires purs (sans Spring), et ça masque le couplage fort.

#### B. L'injection par Setter — ⚠️ *Rarement utilisée*

```java
@Service
public class CustomerService {
    private CustomerRepository customerRepository;

    @Autowired
    public void setCustomerRepository(CustomerRepository repo) {
        this.customerRepository = repo;
    }
}

```

* **Utilité :** Pratique uniquement si tu as des dépendances optionnelles qui peuvent changer après la création de l'objet.

#### C. L'injection par Constructeur — ⭐ *La règle d'or en entreprise*

```java
@Service
public class CustomerService {

    private final CustomerRepository customerRepository;

    // Pas besoin de mettre @Autowired si le constructeur est unique (depuis Spring 4.3)
    public CustomerService(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }
}

```

* **Pourquoi c'est le roi :**
1. Tes dépendances peuvent être déclarées en `final` (immuabilité, un énorme plus pour la sécurité des threads).
2. Ton objet ne peut pas être instruit de force sans ses dépendances obligatoires.
3. C'est ultra-facile à tester unitairement en Java pur (tu peux faire `new CustomerService(mockRepo)` sans démarrer Spring).



---

### 3. Comment Spring Core trouve-t-il ton code ? (Le Component Scanning)

Au démarrage, Spring Core a besoin de savoir quelles classes il doit transformer en Beans et stocker dans son contexte. Pour ça, il utilise **le balayage de composants (Component Scanning)**.

Il fouille dans tes packages à la recherche d'annotations spécifiques :

* **`@Component`** : Le marqueur générique. "Hé Spring, fais de cette classe un Bean !"
* **`@Service`** : Spécialisé pour la couche métier (pure sémantique, mais utile pour les architectures).
* **`@Repository`** : Spécialisé pour la couche d'accès aux données (en plus de déclarer un Bean, il traduit automatiquement les erreurs SQL brutes en exceptions Spring compréhensibles).
* **`@Controller` ou `@RestController**` : Pour la couche web/API.
* **`@Configuration`** : Pour indiquer que cette classe sert à fabriquer manuellement des Beans (via des méthodes annotées `@Bean`).

---

### 📝 Ce qu'il faut retenir pour tes entretiens :

> *"Spring Core gère l'Inversion de Contrôle via un conteneur (`ApplicationContext`) qui instancie et administre des objets appelés Beans (Singleton par défaut). L'injection de dépendances doit se faire de préférence par le **constructeur** pour garantir l'immuabilité et faciliter les tests, et le framework repère ces composants grâce au Component Scanning et aux stéréotypes (`@Service`, `@Repository`, etc.)."*


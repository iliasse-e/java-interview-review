## 🗺️ Schéma : Le Voyage d'une Exception (Du fond de la base jusqu'au Front)

Visualisons le chemin que prend une erreur lorsqu'elle se produit dans ton application Spring Boot :

```text
[Base de données / Repository]
       │
       ▼ (1. La donnée n'est pas là, l'Optional est vide)
[Le Service (Le Cerveau Métier)]
       │
       ▼ (2. throw new BankAccountNotFoundException("..."))  <-- C'est ICI que l'exception naît !
[Le Contrôleur (La Porte d'Entrée Web)]
       │
       ▼ (3. L'exception traverse le contrôleur sans s'arrêter...)
[DispatcherServlet (Le chef d'orchestre de Spring)]
       │
       ▼ (4. Il cherche un bouclier pour sauver la situation...)
┌────────────────────────────────────────────────────────┐
│  LE @CONTROLLER_ADVICE (GlobalExceptionHandler)        │
│  - Il intercepte l'exception au vol                    │
│  - Il la traduit en HTTP 404 + JSON propre             │
└────────────────────────────────────────────────────────┘
       │
       ▼ (5. Réponse propre transmise au navigateur)
[Le Front-End Angular (Reçoit un bel objet d'erreur)]

```

Si **l'étape 4 n'existe pas** (pas de `@ControllerAdvice`), l'exception continue de remonter jusqu'au serveur web embarqué (Tomcat), qui panique et renvoie un vilain **HTTP 500** avec la stack trace complète dans la figure du client.

---

## 🔬 Comment lire une Stack Trace de 200 lignes sans saigner des yeux ?

Quand tu ouvres une stack trace, **oublie 90% des lignes**. La majorité des lignes du haut ne sont que du code interne de Spring, de Tomcat ou de Java qui gèrent le trafic.

Pour autopsier une erreur rapidement, suis cette méthode en 3 étapes :

### 1. Lis la toute première ligne (L'exception finale)

C'est le résumé de ce qui vient de crasher.

> *Exemple :* `com.ebanking.exceptions.BankAccountNotFoundException: Compte introuvable pour l'id 123`

* **Ce que ça te dit :** C'est *ton* exception métier. Ça prouve que ton service a bien fait son travail en détectant l'anomalie et en l'émettant.

### 2. Cherche la première ligne avec TON nom de package (Ton code source)

Ignore les lignes qui contiennent `org.springframework...`, `java.base...`, ou `sun.reflect...`. Descends les yeux jusqu'à voir ton propre code.

> *Exemple :* `at com.ebanking.services.BankAccountServiceImpl.getAccount(BankAccountServiceImpl.java:45)`

* **Ce que ça te dit :** C'est **exactement la ligne 45 de ton fichier Java** qui a déclenché le `throw`. C'est là que se trouve le bug ou la règle métier non respectée.

### 3. Regarde s'il y a un `Caused by:` (La cause racine)

Parfois, une exception en cache une autre (par exemple, une erreur SQL qui déclenche une erreur Hibernate, qui elle-même déclenche ton erreur).

* Le `Caused by:` tout en bas de la stack trace est souvent **l'origine ultime du problème** (ex: une connexion à la base de données coupée, une violation de contrainte SQL unique, etc.).

---

## 💡 En résumé pour tes entretiens

Si un tech lead te demande : *"Comment tu gères le debugging d'une erreur en production sur une API Spring Boot ?"*, ta réponse idéale en mode senior :

> *"Je commence par identifier le type d'exception tout en haut de la stack trace, puis je descends directement chercher la première ligne correspondant à mon package métier pour cibler la ligne exacte du service ou du repository qui a levé l'alerte. Ensuite, je m'assure que le `GlobalExceptionHandler` intercepte bien ce type d'exception spécifique pour la transformer en un statut HTTP clair, évitant ainsi de fuiter des stack traces techniques 500 vers le front-end."*

Est-ce que cette vision du "voyage" et de la lecture des erreurs démythifie un peu les grandes lignes rouges dans ta console ?

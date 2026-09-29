# TP 1 — Java : « BankKata », une mini-application bancaire

> **Niveau** : tu as déjà vu la POO et les collections à l'école, et suivi le tuto de Grafikart.
> **Durée estimée** : 8 à 10 h, en 3 ou 4 séances.
> **Correspond à la roadmap** : S01 (Setup + POO), S02 (Collections, Generics, Exceptions), S03 (Lambdas, Streams, Optional, Maven + JUnit 5).
> **Livrable** : un dépôt GitHub public `java-bank-kata` avec un README, plus de 15 tests verts, et une application console qui fonctionne.
> **Suite** : le TP 2 (SQL) reprend le même univers (clients, comptes, virements). Ce que tu codes ici en mémoire, tu le retrouveras en base de données.

---

## Comment travailler avec ce TP

Je te parle comme si nous étions en séance. Trois règles simples :

1. **Cherche d'abord.** Chaque partie contient des *rappels*, puis un *énoncé*. Lis le rappel, puis code. Ne regarde le corrigé (`java-bank-kata-corrige.zip`) qu'à la fin de la partie, pour comparer, pas pour copier.
2. **Un test, un commit.** Dès qu'un morceau fonctionne et que ses tests passent, tu fais un commit. C'est l'habitude n°1 en entreprise : un historique propre montre comment tu travailles.
3. **Compile souvent.** Lance `mvn test` toutes les 15 minutes. Une erreur détectée tôt est facile à corriger.

Les encadrés **🏦 En vrai, à la banque** te donnent le point de vue « production » : ce qu'on fait vraiment quand un bug peut coûter de l'argent.

---

## 0. Ce que tu vas construire

### Le domaine

Une banque gère des **clients**. Chaque client possède un ou plusieurs **comptes** : courants ou d'épargne. On peut y faire des **opérations** (dépôt, retrait, virement) qui sont enregistrées dans un historique.

### Schéma des classes

```
                    ┌────────────────────┐
                    │      Client        │
                    │ id, nom, prénom,   │
                    │ email              │
                    └─────────▲──────────┘
                              │ titulaire
                    ┌─────────┴──────────┐        ┌───────────────────┐
                    │  «abstract»        │ 1    * │  Operation        │
                    │  Compte            ├────────▶│  (record)         │
                    │ iban, solde,       │         │ date, type,       │
                    │ historique         │         │ montant, libellé  │
                    └─────────▲──────────┘         └───────────────────┘
               ┌──────────────┴──────────────┐
    ┌──────────┴─────────┐        ┌──────────┴─────────┐
    │  CompteCourant     │        │  CompteEpargne     │
    │ découvert autorisé │        │ plafond            │
    └────────────────────┘        └────────────────────┘

    Banque (service)      : gère clients + comptes, dépôt / retrait / virement
    StatistiquesService   : calculs avec les Streams
    CompteCsvRepository   : sauvegarde / chargement dans un fichier CSV
    App                   : menu console
```

### Les règles métier (à respecter partout)

| Règle | Détail |
|---|---|
| Argent | Toujours en `BigDecimal`, **2 décimales exactement**. Jamais de `double`. |
| Montant d'une opération | Strictement positif. Le sens (crédit/débit) vient du type d'opération. |
| Compte courant | Le solde peut descendre jusqu'à `- découvert autorisé`. |
| Compte d'épargne | Jamais à découvert. Le solde ne peut pas dépasser le plafond. |
| Virement | **Tout ou rien** : si une des deux opérations est impossible, aucun compte ne bouge. |
| Historique | Consultable, mais jamais modifiable de l'extérieur. |
| IBAN | Fictif : `MA64` suivi de 24 chiffres, soit `MA` + 26 chiffres (même format que la colonne `compte.iban` du TP SQL). |

🏦 **En vrai, à la banque** : on ne « corrige » jamais un historique d'opérations. On ajoute une écriture inverse. C'est pour ça que `Operation` sera un `record` (immuable) et que l'historique sera exposé en lecture seule.

---

## 1. Préparation de l'environnement (Ubuntu)

### 1.1 Installer les outils

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk maven git
java -version        # doit afficher 21 (ou 17)
mvn -version         # doit afficher Maven 3.8+ et la version de Java
git --version
```

Si `openjdk-21-jdk` n'existe pas sur ta version d'Ubuntu, prends `openjdk-17-jdk` : le projet vise Java 17.

**IDE au choix** : IntelliJ IDEA Community (`sudo snap install intellij-idea-community --classic`) ou VS Code avec l'extension *Extension Pack for Java*.

### 1.2 Configurer Git (une seule fois)

```bash
git config --global user.name  "Ton Nom"
git config --global user.email "ton.email@example.com"
git config --global init.defaultBranch main
```

### 1.3 Créer le projet

```bash
mkdir java-bank-kata && cd java-bank-kata
git init
mkdir -p src/main/java/com/bankkata/{model,exception,service,persistence}
mkdir -p src/test/java/com/bankkata/{model,service,persistence}
```

Crée ensuite deux fichiers à la racine.

**`pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.bankkata</groupId>
    <artifactId>java-bank-kata</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.10.2</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <mainClass>com.bankkata.App</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

**`.gitignore`**

```
target/
.idea/
*.iml
.vscode/
*.log
data/*.csv
```

### 1.4 Rappel : Maven en 5 lignes

| Notion | À retenir |
|---|---|
| Structure standard | `src/main/java` (code), `src/test/java` (tests). Maven les trouve tout seul. |
| Dépendance | Déclarée dans `pom.xml`, téléchargée dans `~/.m2`. |
| Cycle de vie | `compile` → `test` → `package`. Chaque commande exécute les étapes précédentes. |
| Commandes utiles | `mvn test` · `mvn -q test` (silencieux) · `mvn -Dtest=CompteCourantTest test` · `mvn clean` |
| Rapports de tests | Dans `target/surefire-reports/` |

### 1.5 Vérifier que tout fonctionne

```bash
mvn -q test
```

Tu dois voir « No tests to run » ou un `BUILD SUCCESS`, sans erreur rouge. Premier commit :

```bash
git add . && git commit -m "chore: initialise le projet Maven (Java 17, JUnit 5)"
```

> 💡 **Convention de messages de commit** : `feat:` (fonctionnalité), `test:` (tests), `fix:` (correction), `docs:` (documentation), `refactor:` (réorganisation), `chore:` (outillage). Un verbe à l'infinitif, moins de 70 caractères.

---

## Partie 1 — Les fondations : `Montants`, `Client`, `Operation`

### 📚 Rappels

**1) Encapsulation et immuabilité.** Les attributs sont `private`, souvent `final`. On les initialise dans le constructeur et on **valide** à cet endroit : un objet ne doit jamais exister dans un état invalide.

**2) Pourquoi pas `double` pour l'argent ?**

```java
System.out.println(0.1 + 0.2);            // 0.30000000000000004  ← inacceptable pour de l'argent
BigDecimal a = new BigDecimal("0.10");
BigDecimal b = new BigDecimal("0.20");
System.out.println(a.add(b));             // 0.30
```

À retenir sur `BigDecimal` :

| Point | Explication |
|---|---|
| Construction | `new BigDecimal("0.1")` (avec une **String**) ou `BigDecimal.valueOf(0.1)`. Jamais `new BigDecimal(0.1)` : le `double` est déjà faux. |
| Immuable | `a.add(b)` **renvoie** un nouvel objet. `a` ne change pas. |
| Scale | Nombre de décimales. `new BigDecimal("2.0")` a un scale de 1, `new BigDecimal("2.00")` de 2. |
| `equals` vs `compareTo` | `2.0.equals(2.00)` est **faux** (scale différent). `2.0.compareTo(2.00) == 0` est **vrai**. |
| Arrondi | `setScale(2, RoundingMode.HALF_EVEN)` (arrondi bancaire). `RoundingMode.UNNECESSARY` lève une exception si un arrondi serait nécessaire. |
| Division | `a.divide(b)` sans précision lève une exception si le résultat est infini. Toujours `a.divide(b, 2, RoundingMode.HALF_EVEN)`. |
| Signe | `x.signum()` renvoie -1, 0 ou 1. Plus lisible que `compareTo(BigDecimal.ZERO)`. |

**Choix du TP :** on impose `scale = 2` partout. Ainsi `equals` fonctionne aussi dans les tests.

**3) `enum` avec un champ.** Un `enum` peut avoir un constructeur, des attributs et des méthodes.

```java
public enum Couleur {
    ROUGE(255, 0, 0), VERT(0, 255, 0);
    private final int r, g, b;
    Couleur(int r, int g, int b) { this.r = r; this.g = g; this.b = b; }
}
```

**4) `equals` et `hashCode`.** Le contrat : si `a.equals(b)` alors `a.hashCode() == b.hashCode()`. Sans ça, un `HashMap` ou un `HashSet` se comporte de façon incohérente. On base l'égalité sur l'**identifiant métier**, pas sur tous les champs.

**5) `record` (Java 16+).** Une classe de données immuable, avec constructeur, accesseurs, `equals`, `hashCode` et `toString` générés.

```java
public record Point(int x, int y) {
    public Point {                       // « constructeur compact » : idéal pour valider
        if (x < 0) throw new IllegalArgumentException("x négatif");
    }
}
// accesseur : p.x()   (et non p.getX())
```

### 🎯 Énoncé

**Exercice 1.1 — `TypeOperation` (package `model`).**
Crée un `enum` avec 4 valeurs : `DEPOT`, `RETRAIT`, `VIREMENT_EMIS`, `VIREMENT_RECU`. Chaque valeur porte un booléen indiquant si c'est un **crédit**. Ajoute la méthode `boolean estCredit()`.

**Exercice 1.2 — `Montants` (package `model`).**
Une classe utilitaire (constructeur privé, méthodes `static`) avec :

- une constante `ZERO` valant `0.00` (scale 2) ;
- `BigDecimal normaliser(BigDecimal m)` : renvoie `m` avec exactement 2 décimales. Si `m` a plus de 2 décimales *significatives* (ex. `10.123`), lève `IllegalArgumentException` (message : « Un montant a au plus 2 décimales »). Si `m` est `null`, lève `NullPointerException` ;
- `BigDecimal strictementPositif(BigDecimal m)` : normalise, puis lève `IllegalArgumentException` si le montant est ≤ 0.

> Piste : `setScale(2, RoundingMode.UNNECESSARY)` lève une `ArithmeticException` quand un arrondi est nécessaire. À toi de la transformer en `IllegalArgumentException`.

**Exercice 1.3 — `Client` (package `model`).**
Attributs `id`, `nom`, `prenom`, `email`, tous `final`. Le constructeur refuse les valeurs `null` ou vides, et un email sans `@`. Ajoute `getNomComplet()` (« Prénom Nom »), `equals`/`hashCode` **basés sur `id` seulement**, et un `toString` lisible.

**Exercice 1.4 — `Operation` (package `model`).**
Un `record` avec les composants `LocalDateTime date`, `TypeOperation type`, `BigDecimal montant`, `String iban`, `String libelle`.
Dans le constructeur compact : refuse `date`, `type`, `iban` nuls ; impose `montant = Montants.strictementPositif(montant)` ; remplace un libellé nul par une chaîne vide.
Ajoute `BigDecimal montantSigne()` : positif pour un crédit, négatif pour un débit.

### ✅ Vérifie ton travail

```java
Client c = new Client("C001", "Benali", "Yassine", "yassine@example.com");
System.out.println(c);                                           // C001 - Yassine Benali <yassine@example.com>
System.out.println(Montants.normaliser(new BigDecimal("10")));   // 10.00
Montants.normaliser(new BigDecimal("10.123"));                   // IllegalArgumentException
new Operation(LocalDateTime.now(), TypeOperation.RETRAIT,
              new BigDecimal("50"), "MA64...", "test").montantSigne();   // -50.00
```

### 🧪 Tests à écrire (voir aussi la partie 7)

- `Montants` : `10` devient `10.00` ; `10.123` est refusé ; `0` et `-5` refusés par `strictementPositif`.
- `Client` : deux clients de même `id` mais de noms différents sont égaux ; email sans `@` refusé.
- `Operation` : `montantSigne` négatif pour `RETRAIT` et `VIREMENT_EMIS`, positif pour `DEPOT` et `VIREMENT_RECU`.

```bash
git add . && git commit -m "feat: ajoute Montants, Client, TypeOperation et Operation"
```

---

## Partie 2 — Héritage et polymorphisme : `Compte`, `CompteCourant`, `CompteEpargne`

### 📚 Rappels

**1) Classe abstraite ou interface ?**

| | Classe abstraite | Interface |
|---|---|---|
| État (attributs) | Oui | Non (constantes seulement) |
| Code partagé | Oui (méthodes concrètes) | Oui, via `default` (à utiliser avec parcimonie) |
| Héritage | Une seule classe parente | Plusieurs interfaces |
| On l'utilise pour | Une **famille** de classes qui partagent un état et un comportement (« un compte *est un* compte ») | Un **contrat** ou une capacité (« peut être trié », « peut être sauvegardé ») |

Ici, un compte a un IBAN, un solde et un historique : c'est un état commun. On choisit donc une **classe abstraite**.

**2) Polymorphisme.** Une variable de type `Compte` peut contenir un `CompteCourant` ou un `CompteEpargne`. À l'exécution, c'est la méthode de la **classe réelle** qui est appelée.

```java
Compte c = new CompteEpargne(...);
c.verifierDebit(montant);      // appelle la version de CompteEpargne, pas celle d'une autre classe
```

**3) Redéfinition (`@Override`).** Toujours mettre l'annotation : le compilateur vérifie que la méthode existe bien dans la classe parente.

**4) Constructeurs.**
- `super(...)` appelle le constructeur parent (première instruction).
- `this(...)` appelle un autre constructeur de la même classe (chaînage : évite de dupliquer du code).

**5) Visibilité.** `private` (la classe seule) · *(rien)* (le package) · `protected` (package + sous-classes) · `public` (tout le monde). Règle : on commence par le plus restrictif.

**6) Le patron « vérifier, puis modifier ».** Pour qu'une opération soit sûre, on vérifie **toutes** les règles avant de toucher à l'état. Si une vérification échoue, rien n'a changé.

```java
public void debiter(BigDecimal montant, ...) throws SoldeInsuffisantException {
    BigDecimal m = Montants.strictementPositif(montant);
    verifierDebit(m);                 // peut lever une exception : l'état n'est PAS encore modifié
    solde = solde.subtract(m);        // modification seulement après les vérifications
    historique.add(new Operation(...));
}
```

**7) Pattern matching pour `instanceof` (Java 16+).**

```java
if (compte instanceof CompteCourant courant) {     // déclare et affecte « courant » en une ligne
    System.out.println(courant.getDecouvertAutorise());
}
```

### 🎯 Énoncé

**Exercice 2.1 — `Compte` (abstraite).**

Attributs : `iban` (String), `titulaire` (Client), `solde` (BigDecimal, modifiable), `historique` (`List<Operation>`).

| Élément | Consigne |
|---|---|
| Constructeur `protected` | `(String iban, Client titulaire, BigDecimal soldeInitial)`. Refuse un IBAN vide, un titulaire nul. Normalise le solde. |
| `abstract void verifierDebit(BigDecimal montant)` | Lève `SoldeInsuffisantException` si le débit est interdit. **Ne modifie rien.** |
| `void verifierCredit(BigDecimal montant)` | Version par défaut : vérifie seulement que le montant est valide. Les sous-classes peuvent la redéfinir. |
| `abstract String getType()` | Renvoie `"COURANT"` ou `"EPARGNE"`. |
| `deposer(montant)` / `retirer(montant)` | Raccourcis qui appellent `crediter` / `debiter` avec `DEPOT` / `RETRAIT`. |
| `crediter(montant, type, libelle)` | Refuse un `type` qui n'est pas un crédit. Vérifie, puis ajoute au solde, puis enregistre l'`Operation`. |
| `debiter(montant, type, libelle)` | Idem pour un débit. |
| `getHistorique()` | Renvoie une **vue non modifiable** (`Collections.unmodifiableList`). |

> Dans un premier temps, tu peux ignorer les exceptions déclarées (`throws`) : tu les créeras dans la partie 3. Écris d'abord la logique, puis reviens.

**Exercice 2.2 — `CompteCourant`.**
- Attribut `decouvertAutorise` (≥ 0).
- Deux constructeurs : `(iban, titulaire, decouvert)` qui appelle `this(..., Montants.ZERO)`, et `(iban, titulaire, decouvert, soldeInitial)` qui appelle `super(...)`.
- `getSoldeDisponible()` = solde + découvert.
- `verifierDebit` : refuse si `montant > soldeDisponible`.

**Exercice 2.3 — `CompteEpargne`.**
- Attribut `plafond` (> 0).
- `verifierDebit` : refuse si `montant > solde` (jamais de découvert).
- `verifierCredit` : refuse si `solde + montant > plafond`.

### ❓ Questions de réflexion (réponds par écrit dans ton README)

1. Pourquoi `Compte` est-elle abstraite ? Que se passerait-il avec `new Compte(...)` ?
2. Pourquoi sépare-t-on `verifierDebit` (qui ne modifie rien) de `debiter` (qui modifie) ? Pense au virement de la partie 4.
3. Pourquoi `getHistorique()` ne renvoie-t-il pas directement la liste interne ?
4. Ajouter un `CompteJoint` demanderait de modifier quelles classes ? (C'est le principe ouvert/fermé.)

### ✅ Vérifie ton travail

```java
CompteCourant cc = new CompteCourant("MA64000000000000000000000001", client, new BigDecimal("1000"));
cc.deposer(new BigDecimal("500"));
cc.retirer(new BigDecimal("1400"));           // autorisé : solde -900.00 (découvert 1000)
cc.retirer(new BigDecimal("200"));            // refusé : disponible 100.00, demandé 200.00
```

```bash
git add . && git commit -m "feat: ajoute Compte, CompteCourant et CompteEpargne"
```

---

## Partie 3 — Les exceptions

### 📚 Rappels

**1) Checked ou unchecked ?**

| | Checked | Unchecked |
|---|---|---|
| Hérite de | `Exception` (mais pas de `RuntimeException`) | `RuntimeException` |
| Le compilateur impose | `try/catch` **ou** `throws` | Rien |
| Quand l'utiliser | Situation **normale du métier** que l'appelant peut traiter : solde insuffisant, compte introuvable | **Bug de programmation** ou erreur technique : argument invalide, fichier illisible |
| Exemples du JDK | `IOException`, `SQLException` | `IllegalArgumentException`, `NullPointerException`, `IllegalStateException` |

**2) Syntaxe.**

```java
public void retirer(BigDecimal m) throws SoldeInsuffisantException {   // déclare
    if (m.compareTo(disponible) > 0) {
        throw new SoldeInsuffisantException(iban, disponible, m);      // lève
    }
}

try {
    compte.retirer(m);
} catch (SoldeInsuffisantException e) {                                // traite le cas précis
    System.out.println("Refusé : " + e.getMessage());
} finally {
    // exécuté dans tous les cas (rarement utile depuis try-with-resources)
}
```

**3) `try-with-resources`.** Toute ressource `AutoCloseable` (fichier, connexion SQL…) déclarée entre parenthèses est fermée automatiquement, même en cas d'exception.

```java
try (BufferedReader r = Files.newBufferedReader(chemin)) {
    return r.readLine();
}    // r.close() est appelé ici, toujours
```

**4) Bonnes pratiques.**
- Attraper l'exception **la plus précise** possible, jamais un `catch (Exception e)` fourre-tout au milieu du code.
- **Ne jamais avaler** une exception (`catch { }` vide).
- Quand on transforme une exception, on **garde la cause** : `throw new PersistanceException("...", e)`.
- Un message d'erreur doit contenir les valeurs utiles (IBAN, montant, solde).
- Une exception métier peut porter des **données** (l'IBAN, le disponible) pour que l'appelant les exploite.

🏦 **En vrai, à la banque** : un message d'erreur qui s'affiche au client ne doit **jamais** contenir de donnée sensible. Le détail va dans les logs internes. Ici, c'est un exercice : on affiche l'IBAN pour te faciliter le débogage.

### 🎯 Énoncé

**Exercice 3.1 — La hiérarchie (package `exception`).**

```
Exception
 └── BanqueException                    (checked, base des erreurs métier)
      ├── SoldeInsuffisantException     (iban, disponible, demandé)
      ├── PlafondDepasseException       (iban, plafond, montant)
      └── CompteIntrouvableException    (iban)

RuntimeException
 └── PersistanceException               (unchecked, erreurs de fichier)
```

Chaque classe doit fournir des accesseurs pour ses données et un message clair.

**Exercice 3.2 — Choisir le bon type d'erreur.** Pour chaque situation, décide : *exception checked métier*, *IllegalArgumentException*, ou *Optional vide*. Note ta décision, puis compare avec le corrigé.

| Situation | Ta décision |
|---|---|
| Retrait de 500 sur un compte qui ne peut plus rien débiter | ? |
| Dépôt d'un montant de `-20` | ? |
| Recherche d'un compte par IBAN dans un écran de consultation (« peut-être inexistant ») | ? |
| Virement vers un IBAN qui doit exister | ? |
| Montant `10.123` (3 décimales) | ? |
| Fichier CSV introuvable au chargement | ? |
| Dépôt qui ferait dépasser le plafond d'un livret | ? |

**Exercice 3.3 — Mettre à jour `Compte`.** Ajoute les clauses `throws` aux méthodes de la partie 2 et fais lever les bonnes exceptions.

```bash
git add . && git commit -m "feat: ajoute les exceptions métier et technique"
```

---

## Partie 4 — Collections et service `Banque`

### 📚 Rappels

**1) Les interfaces de base.**

| Interface | Idée | Implémentations courantes |
|---|---|---|
| `List<E>` | Séquence ordonnée, doublons permis, accès par index | `ArrayList` (défaut), `LinkedList` (rarement utile) |
| `Set<E>` | Pas de doublons | `HashSet` (rapide, sans ordre), `LinkedHashSet` (ordre d'insertion), `TreeSet` (trié) |
| `Map<K,V>` | Association clé → valeur | `HashMap`, `LinkedHashMap`, `TreeMap` |

**2) Les trois `Map`.**

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| Recherche par clé | O(1) en moyenne | O(1) | O(log n) |
| Ordre d'itération | Aucun | Ordre d'insertion | Ordre trié des clés |
| Clé nulle | Oui | Oui | Non |
| Quand | Défaut | On veut un affichage stable | On veut un tri ou des recherches par intervalle |

**3) Le contrat `equals`/`hashCode` dans les collections.** `HashMap` et `HashSet` utilisent `hashCode` pour trouver le « casier » puis `equals` pour comparer. Un `equals` sans `hashCode` cohérent = éléments « introuvables ».

**4) `Optional<T>` : un retour qui peut être vide.**

```java
Optional<Compte> oc = banque.trouverCompte(iban);
oc.isPresent();                                   // vrai ou faux
oc.map(Compte::getSolde).orElse(BigDecimal.ZERO); // transformation avec valeur par défaut
oc.orElseThrow(() -> new CompteIntrouvableException(iban));   // ou exception si vide
```

À éviter : `optional.get()` sans vérification (équivalent d'un `NullPointerException`), `Optional` en attribut ou en paramètre.

**5) Copies défensives.** Une méthode qui renvoie sa liste interne permet à l'appelant de la casser. Renvoie `List.copyOf(...)` ou `Collections.unmodifiableList(...)`. Différence : `copyOf` est un **instantané** ; `unmodifiableList` est une **vue** qui suit la liste d'origine.

**6) Comparer avec `Comparator`.**

```java
comptes.sort(Comparator.comparing(Compte::getSolde).reversed());
comptes.sort(Comparator.comparing(Compte::getType).thenComparing(Compte::getIban));
```

**7) Génériques : le minimum.** `List<Compte>` garantit le type à la compilation. `List<? extends Compte>` accepte une liste de `Compte` **ou** de sous-classes, en lecture. Une méthode générique : `static <T> T premier(List<T> l) { return l.get(0); }`.

### 🎯 Énoncé

**Exercice 4.1 — La classe `Banque` (package `service`).**

Attributs : `Map<String, Client> clients` et `Map<String, Compte> comptes`. Choisis l'implémentation de `Map` et **justifie ton choix en une phrase** dans un commentaire.

| Méthode | Comportement |
|---|---|
| `Client creerClient(nom, prenom, email)` | Génère un identifiant `C001`, `C002`… et enregistre le client. |
| `CompteCourant ouvrirCompteCourant(Client, BigDecimal decouvert)` | Refuse un client inconnu. Génère un IBAN unique. |
| `CompteEpargne ouvrirCompteEpargne(Client, BigDecimal plafond)` | Idem. |
| `Optional<Compte> trouverCompte(String iban)` | Vide si inconnu. |
| `Compte getCompte(String iban)` | Lève `CompteIntrouvableException` si inconnu. |
| `List<Compte> getComptes()` / `getClients()` | Copies non modifiables. |
| `List<Compte> comptesDe(Client)` | Les comptes d'un client. |
| `deposer(iban, montant)` / `retirer(iban, montant)` | Délèguent au compte. |
| `virement(source, destination, montant)` | Voir 4.2. |
| `enregistrer(Compte)` | Ajoute un compte déjà construit (utile pour le chargement CSV). |

> Génération d'IBAN : `"MA64" + String.format("%024d", numero)`. Assure-toi qu'un IBAN déjà pris n'est jamais réutilisé.

**Exercice 4.2 — Le virement « tout ou rien ».**

Le piège : on débite la source, puis le crédit de la destination échoue (plafond dépassé). La source a perdu de l'argent, et personne ne l'a reçu.

```
   MAUVAIS                                       BON
   ───────                                       ───
   1. source.debiter()   ✔  (état modifié)       1. source.verifierDebit()          (rien ne bouge)
   2. dest.crediter()    ✘  exception !          2. destination.verifierCredit()    (rien ne bouge)
   → source débitée pour rien                    3. source.debiter()    ─┐ ne peuvent plus échouer
                                                 4. destination.crediter()─┘ (déjà vérifiés)
```

Consignes :
- refuse un virement d'un compte vers lui-même (`IllegalArgumentException`) ;
- vérifie les deux côtés **avant** toute modification ;
- libellés : « Virement vers <iban> » et « Virement de <iban> » ;
- types d'opération : `VIREMENT_EMIS` côté source, `VIREMENT_RECU` côté destination.

🔗 **Pont vers le TP SQL** : c'est exactement ce que fait une transaction SQL (`BEGIN … COMMIT`, ou `ROLLBACK` en cas d'erreur). Ici tu obtiens l'atomicité « à la main » ; en base, PostgreSQL te la garantit. Tu le manipuleras dans la partie D du TP 2.

**Exercice 4.3 (bonus) — La version « compensation ».** Réécris le virement sans les vérifications préalables : on débite, on tente de créditer, et si ça échoue on **recrédite** la source. Quels sont les inconvénients par rapport à la vérification préalable ? (Indice : que se passe-t-il si la compensation échoue elle aussi ? Et l'historique ?)

### 🧪 Tests essentiels

- virement nominal : soldes des deux comptes + une opération dans chaque historique ;
- virement avec solde insuffisant : **aucun** compte modifié, **aucune** opération ajoutée ;
- virement vers une épargne pleine : la source n'est pas débitée ;
- virement vers un IBAN inconnu ; virement vers soi-même ;
- `trouverCompte` inconnu → `Optional.empty()`.

```bash
git add . && git commit -m "feat: ajoute le service Banque avec virement atomique"
```

---

## Partie 5 — Lambdas et Streams

### 📚 Rappels

**1) Interfaces fonctionnelles.** Une interface avec **une seule** méthode abstraite : on peut la remplacer par une lambda.

| Interface | Signature | Exemple |
|---|---|---|
| `Predicate<T>` | `T → boolean` | `c -> c.getSolde().signum() < 0` |
| `Function<T,R>` | `T → R` | `Compte::getSolde` |
| `Consumer<T>` | `T → void` | `System.out::println` |
| `Supplier<T>` | `() → T` | `() -> new ArrayList<>()` |
| `Comparator<T>` | `(T,T) → int` | `Comparator.comparing(Compte::getSolde)` |

Deux écritures équivalentes : lambda `c -> c.getSolde()` et référence de méthode `Compte::getSolde`.

**2) Un pipeline de Stream.** Source → opérations intermédiaires (paresseuses) → **une** opération terminale.

```java
List<String> ibans = comptes.stream()                   // source
        .filter(c -> c.getSolde().signum() < 0)         // intermédiaire : garde
        .sorted(Comparator.comparing(Compte::getSolde)) // intermédiaire : trie
        .map(Compte::getIban)                           // intermédiaire : transforme
        .toList();                                      // terminale : produit le résultat
```

| Catégorie | Opérations |
|---|---|
| Intermédiaires | `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek` |
| Terminales | `collect`, `toList`, `reduce`, `count`, `min`, `max`, `findFirst`, `anyMatch`, `allMatch`, `noneMatch`, `forEach` |

**3) Les `Collectors` les plus utiles.**

```java
Collectors.toList()                                     // ou stream.toList() (Java 16+, non modifiable)
Collectors.groupingBy(Compte::getTitulaire)             // Map<Client, List<Compte>>
Collectors.groupingBy(Operation::type, Collectors.counting())          // Map<TypeOperation, Long>
Collectors.reducing(BigDecimal.ZERO, Compte::getSolde, BigDecimal::add) // somme d'un BigDecimal
Collectors.partitioningBy(c -> c.getSolde().signum() >= 0)             // Map<Boolean, List<...>>
Collectors.joining(", ")                                               // String
```

**4) `flatMap`.** Transforme « un élément → plusieurs éléments » et aplatit. Ici : `comptes.stream().flatMap(c -> c.getHistorique().stream())` donne toutes les opérations de tous les comptes.

**5) Sommer des `BigDecimal`.** Il n'y a pas de `sum()` pour `BigDecimal` : utilise `reduce(BigDecimal.ZERO, BigDecimal::add)`.

**6) Pièges.**
- Un stream ne se réutilise pas : une fois l'opération terminale appelée, il est consommé.
- Ne **modifie pas** de variable externe dans une lambda (état partagé). Utilise `collect` ou `reduce`.
- Ne transforme pas tout en stream par principe : une boucle `for` simple est parfois plus lisible.

**7) 🔗 Le pont avec SQL** (tu vas t'en servir au TP 2) :

| Stream Java | SQL |
|---|---|
| `filter(...)` | `WHERE` |
| `map(...)` | `SELECT colonnes` |
| `sorted(...)` | `ORDER BY` |
| `limit(n)` | `LIMIT n` |
| `groupingBy(...)` + `counting()` | `GROUP BY` + `COUNT(*)` |
| `reduce(..., add)` | `SUM(...)` |
| `distinct()` | `SELECT DISTINCT` |
| `anyMatch(...)` | `EXISTS (...)` |

### 🎯 Énoncé

**Exercice 5.1 — `StatistiquesService` (package `service`).** Chaque méthode reçoit une `Collection<Compte>` et **ne modifie rien**.

| Méthode | Résultat attendu |
|---|---|
| `Map<Client, BigDecimal> soldeTotalParClient(...)` | Somme des soldes de chaque client. |
| `List<Operation> topOperations(..., int n)` | Les *n* plus grosses opérations (par montant décroissant), tous comptes confondus. |
| `Optional<BigDecimal> soldeMoyen(...)` | Moyenne arrondie à 2 décimales (`HALF_EVEN`). **Vide** s'il n'y a aucun compte (pas de division par zéro). |
| `List<Compte> comptesADecouvert(...)` | Comptes de solde < 0, du plus négatif au moins négatif. |
| `Optional<Compte> compteLePlusRiche(...)` | Le compte au plus gros solde. |
| `Map<TypeOperation, Long> nombreOperationsParType(...)` | Nombre d'opérations par type. Utilise une `EnumMap` pour l'ordre. |

**Exemple de vérification** avec 3 comptes (800, 500 et -150) : `soldeMoyen` doit valoir `383.33` ; `(800 + 500 - 150) / 3 = 383,333…`.

**Exercice 5.2 (bonus).** Ajoute `Map<Boolean, List<Compte>> partitionnerParSigne(...)` avec `partitioningBy`, et `String rapport(...)` qui produit, avec `Collectors.joining`, une ligne par client (« Yassine Benali : 1300.00 »).

```bash
git add . && git commit -m "feat: ajoute StatistiquesService (streams)"
```

---

## Partie 6 — Persistance dans un fichier CSV

### 📚 Rappels

**1) L'API `java.nio.file` (moderne).**

```java
Path p = Path.of("data", "comptes.csv");                    // chemin RELATIF au dossier d'où on lance le programme
Files.createDirectories(p.toAbsolutePath().getParent());    // crée data/ s'il manque
try (BufferedWriter w = Files.newBufferedWriter(p, StandardCharsets.UTF_8)) {
    w.write("ligne");
    w.newLine();
}
try (BufferedReader r = Files.newBufferedReader(p, StandardCharsets.UTF_8)) {
    String ligne;
    while ((ligne = r.readLine()) != null) { /* traiter */ }
}
```

**2) Toujours préciser l'encodage** (`UTF-8`) : sinon les accents (« Zoé ») dépendent de la machine.

**3) Découper une ligne.** `ligne.split(";", -1)` : le `-1` conserve les champs vides en fin de ligne. Sans lui, `"a;b;;"` donne 2 champs au lieu de 4.

**4) Les pièges du CSV.**
- Une donnée qui contient le séparateur (« Ben;ali ») casse le format. Solution simple ici : **refuser** à l'écriture. Solution professionnelle : entourer de guillemets, ou utiliser une bibliothèque (Apache Commons CSV).
- Le format doit être **documenté** et **validé** à la lecture : nombre de champs, types, valeurs autorisées.
- Un message d'erreur utile indique le **numéro de ligne**.

**5) `switch` en expression (Java 14+).**

```java
Compte c = switch (type) {
    case "COURANT" -> new CompteCourant(...);
    case "EPARGNE" -> new CompteEpargne(...);
    default        -> throw new PersistanceException("type inconnu : " + type);
};
```

**6) Enveloppe des exceptions.** `IOException` est *checked*. À la frontière de ta classe de persistance, tu la captures et tu relances une `PersistanceException` (*unchecked*) en gardant la cause. Le reste de l'application n'a plus à connaître `IOException`.

### 🎯 Énoncé

**Format du fichier** (séparateur `;`, UTF-8, première ligne = entête) :

```
type;iban;idClient;nom;prenom;email;solde;parametre
COURANT;MA64000000000000000000000001;C001;Benali;Yassine;yassine@example.com;1234.56;500.00
EPARGNE;MA64000000000000000000000002;C001;Benali;Yassine;yassine@example.com;300.00;20000.00
```

`parametre` = découvert autorisé pour un compte courant, plafond pour un compte d'épargne.

**Exercice 6.1 — `CompteCsvRepository` (package `persistence`).**

- `void sauvegarder(Path fichier, Collection<Compte> comptes)` : crée le dossier parent s'il manque, écrit l'entête puis une ligne par compte. Refuse (avec `PersistanceException`) un champ contenant `;`.
- `List<Compte> charger(Path fichier)` :
  - saute l'entête ;
  - crée **un seul objet `Client` par identifiant** (deux comptes du même client partagent le même objet — pense à `Map.computeIfAbsent`) ;
  - vérifie qu'il y a 8 champs ; sinon `PersistanceException("Ligne N invalide …")` ;
  - convertit les montants ; un montant illisible donne une `PersistanceException` avec le numéro de ligne ;
  - un type inconnu donne aussi une `PersistanceException`.
- L'historique **n'est pas** sauvegardé : après un chargement, il est vide, mais les soldes sont corrects. (C'est pour cela que les constructeurs acceptent un `soldeInitial`.)

**Exercice 6.2 (bonus).** Sauvegarde aussi l'historique dans `operations.csv`, et recharge-le. Quelle difficulté rencontres-tu pour recréer les `Operation` dans un `Compte` sans modifier son solde ?

```bash
git add . && git commit -m "feat: ajoute la persistance CSV des comptes"
```

---

## Partie 7 — Tests JUnit 5 (à faire tout au long du TP)

### 📚 Rappels

**1) Anatomie d'un test.**

```java
class CompteCourantTest {

    private CompteCourant compte;

    @BeforeEach                                   // exécuté avant CHAQUE test : état neuf
    void preparer() {
        Client client = new Client("C001", "Benali", "Yassine", "y@example.com");
        compte = new CompteCourant("MA64000000000000000000000001", client, new BigDecimal("1000.00"));
    }

    @Test
    @DisplayName("Un retrait au-delà du découvert est refusé et ne modifie rien")
    void retraitAuDelaDuDecouvert() {
        // Arrange : fait dans @BeforeEach
        // Act + Assert
        assertThrows(SoldeInsuffisantException.class,
                () -> compte.retirer(new BigDecimal("1000.01")));
        assertEquals(new BigDecimal("0.00"), compte.getSolde());   // l'état n'a pas bougé
    }
}
```

**2) Le patron AAA.** *Arrange* (préparer) → *Act* (agir) → *Assert* (vérifier). Un test = **un comportement**.

**3) Les assertions.**

| Assertion | Usage |
|---|---|
| `assertEquals(attendu, reel)` | **L'attendu en premier.** Sinon les messages d'erreur sont inversés. |
| `assertTrue` / `assertFalse` | Conditions booléennes. |
| `assertThrows(Classe.class, () -> ...)` | Vérifie qu'une exception est levée ; renvoie l'exception pour inspecter son message. |
| `assertAll(() -> ..., () -> ...)` | Regroupe plusieurs assertions : elles sont **toutes** évaluées. |
| `assertSame` / `assertNotEquals` / `assertNull` | Identité, différence, nullité. |
| `assertInstanceOf(Type.class, obj)` | Vérifie le type réel. |

**4) Tests paramétrés** : un même test avec plusieurs jeux de données.

```java
@ParameterizedTest
@ValueSource(strings = {"0", "-10", "-0.01"})
void montantNulOuNegatifEstRefuse(String valeur) {
    assertThrows(IllegalArgumentException.class, () -> compte.deposer(new BigDecimal(valeur)));
}
```

**5) `@TempDir`** : JUnit crée un dossier temporaire, supprimé après le test. Indispensable pour tester des fichiers sans polluer ton disque.

**6) Ce qu'un bon test est** (F.I.R.S.T.) : **F**ast (rapide), **I**ndependent (indépendant des autres), **R**epeatable (même résultat à chaque fois), **S**elf-validating (vert ou rouge, sans lecture manuelle), **T**imely (écrit avec le code).

**7) Que tester ?** Le cas nominal, **et** les erreurs, **et** les valeurs limites (pile au découvert, pile au plafond, `0.01` de trop). La plupart des bugs bancaires se cachent aux limites.

### 🎯 Le cahier de tests (objectif : au moins 25 tests verts)

Coche au fur et à mesure.

| # | Classe testée | Comportement à vérifier | ☐ |
|---|---|---|---|
| 1 | `Montants` | `10` devient `10.00` | ☐ |
| 2 | `Montants` | `10.123` refusé | ☐ |
| 3 | `Montants` | `0` et `-5` refusés par `strictementPositif` (paramétré) | ☐ |
| 4 | `Client` | Égalité par `id` seulement + `hashCode` cohérent (test dans un `HashSet`) | ☐ |
| 5 | `Client` | Email sans `@` refusé ; champs vides refusés | ☐ |
| 6 | `Operation` | `montantSigne` négatif pour un débit, positif pour un crédit | ☐ |
| 7 | `CompteCourant` | Compte neuf : solde 0, historique vide | ☐ |
| 8 | `CompteCourant` | Un dépôt augmente le solde **et** ajoute une opération | ☐ |
| 9 | `CompteCourant` | Un retrait dans le solde fonctionne | ☐ |
| 10 | `CompteCourant` | Retrait **pile** au découvert accepté | ☐ |
| 11 | `CompteCourant` | Retrait `0.01` au-dessus du découvert : `SoldeInsuffisantException`, état inchangé | ☐ |
| 12 | `CompteCourant` | Montants `0`, `-10`, `-0.01` refusés (paramétré) | ☐ |
| 13 | `CompteCourant` | Montant à 3 décimales refusé | ☐ |
| 14 | `CompteCourant` | Historique non modifiable (`UnsupportedOperationException`) | ☐ |
| 15 | `CompteEpargne` | Dépôt pile au plafond accepté | ☐ |
| 16 | `CompteEpargne` | Dépôt qui dépasse le plafond : `PlafondDepasseException`, solde inchangé | ☐ |
| 17 | `CompteEpargne` | Jamais de découvert | ☐ |
| 18 | `Banque` | Identifiants clients/IBAN uniques et bien formés (`MA` + 26 chiffres) | ☐ |
| 19 | `Banque` | `trouverCompte` : `Optional` présent / vide | ☐ |
| 20 | `Banque` | `getCompte` inconnu : `CompteIntrouvableException` | ☐ |
| 21 | `Banque` | Virement nominal (2 soldes + 2 historiques) | ☐ |
| 22 | `Banque` | Virement avec solde insuffisant : rien ne bouge | ☐ |
| 23 | `Banque` | Virement vers une épargne pleine : la source n'est pas débitée | ☐ |
| 24 | `Banque` | Virement vers soi-même refusé | ☐ |
| 25 | `StatistiquesService` | Total par client, top *n*, moyenne (`383.33`), moyenne vide, découverts | ☐ |
| 26 | `CompteCsvRepository` | Sauvegarde puis chargement : mêmes comptes, mêmes soldes, même client partagé | ☐ |
| 27 | `CompteCsvRepository` | Fichier absent : `PersistanceException` avec la cause `IOException` | ☐ |
| 28 | `CompteCsvRepository` | Ligne invalide : le message contient « Ligne 2 » | ☐ |
| 29 | `CompteCsvRepository` | Nom contenant `;` refusé à l'écriture ; accents conservés | ☐ |

### Commandes

```bash
mvn -q test                                          # tous les tests
mvn -q -Dtest=BanqueTest test                        # une seule classe
mvn -q -Dtest=BanqueTest#virementNominal* test       # une méthode (motif)
```

Ouvre `target/surefire-reports/` si tu veux le détail d'un échec.

---

## Partie 8 — L'application console et la mise en forme du dépôt

### 📚 Rappels

- **Lecture clavier** : `Scanner(System.in)`. Utilise `nextLine()` partout (jamais `nextInt()` seul : il laisse le retour à la ligne dans le tampon et le `nextLine()` suivant lit une ligne vide).
- **Saisie d'un montant** : accepte la virgule (`replace(',', '.')`) et convertis les erreurs de format en `IllegalArgumentException` lisible.
- **Séparer les responsabilités** : `App` ne contient **aucune règle métier**. Elle lit, appelle `Banque`, affiche. Toute la logique est testable sans clavier.
- **`switch` moderne** avec `->` et bloc `{ ... }` pour les cas à plusieurs instructions.
- **Text block** (`"""`) pour afficher un menu multi-lignes proprement.
- **Une boucle, trois niveaux d'erreurs** : `catch (BanqueException)` = refus métier ; `catch (IllegalArgumentException)` = saisie invalide ; `catch (RuntimeException)` = incident technique. Le programme ne doit **jamais** planter sur une mauvaise saisie.

### 🎯 Énoncé

**Exercice 8.1 — `App` (package racine `com.bankkata`).**

Menu à boucle avec ces options :

```
 1. Créer un client          6. Virement
 2. Ouvrir un compte         7. Historique d'un compte
 3. Lister les comptes       8. Statistiques
 4. Dépôt                    9. Sauvegarder (CSV)
 5. Retrait                 10. Charger (CSV)
 0. Quitter
```

Lance-la avec :

```bash
mvn -q compile exec:java
```

**Scénario de recette** (fais-le à la main, il doit fonctionner du premier coup) :

1. Créer le client « Benali Yassine ».
2. Lui ouvrir un compte courant (découvert 1000) et un compte d'épargne (plafond 50000).
3. Déposer 2500 sur le courant.
4. Virer 3000,50 du courant vers l'épargne → doit réussir (utilise le découvert).
5. Virer 1000 du courant vers l'épargne → doit être **refusé** (solde insuffisant).
6. Afficher l'historique du courant : 2 lignes (dépôt et virement émis).
7. Sauvegarder, quitter, relancer, charger : les soldes sont retrouvés.

**Exercice 8.2 — Le README.** Il doit contenir : le but du projet, les prérequis (Java 17+, Maven), les commandes (`mvn test`, `mvn -q compile exec:java`), l'architecture (la liste des packages en une phrase chacun), les règles métier, et **ce que tu as appris / ce qui t'a bloqué**. Un recruteur lit d'abord le README.

**Exercice 8.3 — La check-list de livraison.**

- [ ] `mvn clean test` : tous les tests sont verts
- [ ] Aucun `System.out` dans les classes `model`, `service`, `persistence`
- [ ] Aucun `catch` vide, aucun `e.printStackTrace()` laissé
- [ ] Pas de `double` pour de l'argent
- [ ] Le `.gitignore` exclut `target/` et `data/*.csv`
- [ ] Historique Git lisible (au moins 8 commits avec les bons préfixes)
- [ ] Dépôt public sur GitHub, README à jour

```bash
git remote add origin git@github.com:<ton-compte>/java-bank-kata.git
git push -u origin main
git tag v1.0.0 && git push --tags
```

---

## Partie 9 — Auto-évaluation

### Barème (sur 100)

| Critère | Points |
|---|---|
| POO : abstraction, polymorphisme, encapsulation, immuabilité | 20 |
| `BigDecimal` correct partout, aucune fuite de `double` | 10 |
| Exceptions : bon choix checked/unchecked, messages utiles, pas d'exception avalée | 15 |
| Virement atomique (vérifier avant modifier) | 10 |
| Collections et `Optional` bien choisis | 10 |
| Streams lisibles et corrects | 10 |
| Persistance CSV robuste (erreurs claires, UTF-8, ressources fermées) | 10 |
| Tests : au moins 25, cas limites, erreurs testées | 10 |
| Qualité du dépôt : README, commits, `.gitignore` | 5 |

**Moins de 60** : refais les parties faibles avant de passer au SQL. **60 à 80** : tu peux passer au TP 2. **Plus de 80** : passe aux extensions.

### Questions d'entretien (réponds à voix haute, en 1 minute chacune)

1. Pourquoi ne pas utiliser `double` pour représenter de l'argent ?
2. Différence entre `equals` et `==` ? Pourquoi redéfinir `hashCode` avec `equals` ?
3. Classe abstraite ou interface : comment choisis-tu ?
4. Exception *checked* ou *unchecked* : donne un exemple de chaque dans ton projet.
5. Qu'est-ce que le polymorphisme ? Montre-le avec `Compte`.
6. Comment as-tu garanti qu'un virement échoué ne modifie aucun compte ?
7. `HashMap`, `LinkedHashMap` ou `TreeMap` : lequel as-tu choisi et pourquoi ?
8. Pourquoi renvoyer `Optional` plutôt que `null` ?
9. Que fait `try-with-resources` ? Quelle interface faut-il implémenter ?
10. Différence entre `map` et `flatMap` ?
11. Une opération intermédiaire de Stream s'exécute-t-elle sans opération terminale ?
12. Pourquoi `Operation` est-il un `record` ?
13. À quoi sert `@BeforeEach` ? Pourquoi les tests doivent-ils être indépendants ?
14. Que se passe-t-il si deux threads exécutent un virement sur les mêmes comptes en même temps ? (Piste : ton code n'est pas thread-safe. Comment SQL règle-t-il ce problème ? → verrous, partie D du TP 2.)
15. Comment ferais-tu évoluer le projet pour utiliser une base de données au lieu du CSV ? (Piste : une interface `CompteRepository` avec deux implémentations.)

### Extensions (dans l'ordre de difficulté)

1. **Frais de tenue de compte** mensuels pour les comptes courants (opération `FRAIS`).
2. **Intérêts** annuels sur un compte d'épargne : `solde × taux`, arrondi `HALF_EVEN`.
3. **Plafond de retrait journalier** (nécessite de filtrer l'historique par date : `LocalDate`).
4. **Historique persisté** dans un second fichier CSV.
5. **Interface `CompteRepository`** (méthodes `sauvegarder`/`charger`) avec l'implémentation CSV, préparant l'implémentation JDBC.
6. **Journalisation** avec `java.util.logging` : chaque virement accepté ou refusé est loggé (sans donnée sensible).
7. **Contrôle anti-fractionnement** : refuser, ou signaler, plusieurs dépôts en espèces juste sous un seuil sur 7 jours (tu retrouveras ce cas au TP 2, requête C9).

---

## Annexes

### A. Erreurs fréquentes

| Symptôme | Cause probable | Remède |
|---|---|---|
| `error: release version 17 not supported` | JDK trop ancien | `sudo apt install openjdk-17-jdk` ou plus récent, puis `java -version` |
| `mvn` ne trouve aucun test | Fichier ou classe mal nommé | Le nom doit finir par `Test`, dans `src/test/java`, avec `@Test` de `org.junit.jupiter.api` (pas `org.junit`) |
| `assertEquals` échoue : « expected 100.00 but was 100 » | Scale différent | Passe toujours par `Montants.normaliser` |
| `NoSuchFileException: data/comptes.csv` | Dossier absent ou mauvais dossier de lancement | Lance depuis la racine du projet ; crée le dossier parent dans `sauvegarder` |
| Accents illisibles (`Zo?`) | Encodage | `StandardCharsets.UTF_8` partout ; `project.build.sourceEncoding` dans le `pom.xml` |
| `ArithmeticException: Non-terminating decimal expansion` | `divide` sans précision | `divide(x, 2, RoundingMode.HALF_EVEN)` |
| `UnsupportedOperationException` sur une liste | `List.of`, `toList()` ou `unmodifiableList` sont non modifiables | Voulu : travaille sur une copie modifiable `new ArrayList<>(liste)` si besoin |
| `unreported exception … must be caught or declared` | Exception *checked* non traitée | `try/catch` ou `throws` |
| `ConcurrentModificationException` | Modification d'une liste pendant une boucle | Stream + `collect`, ou `removeIf` |

### B. Aide-mémoire Git

```bash
git status                       # état
git add -p                       # ajoute par morceaux (relis ce que tu commits)
git commit -m "feat: ..."        # commit
git log --oneline --graph        # historique
git restore fichier              # annule les modifications non commitées d'un fichier
git commit --amend               # corrige le DERNIER commit (avant push seulement)
```

### C. Ce que tu retrouveras dans le TP 2 (SQL)

| Java (ce TP) | SQL (TP 2) |
|---|---|
| `Client`, `CompteCourant`, `Operation` | Tables `client`, `compte`, `transaction` |
| `BigDecimal` à 2 décimales | `NUMERIC(15,2)` |
| Validation dans les constructeurs | Contraintes `CHECK`, `NOT NULL`, `UNIQUE`, clés étrangères |
| Virement « vérifier puis modifier » | `BEGIN … COMMIT` / `ROLLBACK` |
| `StatistiquesService` (Streams) | `GROUP BY`, fonctions d'agrégat, fonctions de fenêtrage |
| Recherche par IBAN dans une `Map` | Index sur `compte.iban` |
| `CompteCsvRepository` | JDBC (bonus du TP 2) |

Bon courage. Si un test te résiste plus de 30 minutes, écris précisément ce que tu attendais et ce que tu obtiens : dans 8 cas sur 10, la réponse apparaît en rédigeant la question.

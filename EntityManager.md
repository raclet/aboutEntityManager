## Persistance de données avec un ORM : JPA et l'EntityManage illustré avec Hibernate

### Rappel du contexte et motivation

Lorsqu'on utilise JDBC afin de lire des données en base, le code écrit implique l'exécution d'une requête SQL écrite en dur ainsi que la conversion en un objet Java d'une ligne 
de la base de données obtenue en réponse dans un objet _ResultSet_ :

``` java 
ResultSet fetchedArticles = statement.executeQuery("SELECT * FROM articles"); 
while(fetchedArticles.next()) {
    Article art = new Article();
    art.setArticleId(fetchedArticles.getInt("id"));
    art.setTitre(fetchedArticles.getString("titre"));
    art.setCategorie(fetchedArticles.getString("categorie"));
    articles.add(art);
 }
```   

Un ORM (_Object-Relational Mapping_) vise à éliminer ce genre de code qui se répète pour chaque type d'objet persisté
en définissant des correspondances entre les données dans une base et leur représentation sous forme d'objet Java.
De même, sans ORM, ajouter une colonne à une table d'une base de données demande de modifier toutes les requêtes SQL de type 
INSERT / UPDATE.

JPA (_Java Persistance API_) est une spécification de Java EE décrivant un ORM. Hibernate est une implantation de JPA.
Lors des TP, vous pourrez voir que votre projet Spring Boot a été configuré pour utiliser Hibernate en regardant les librairies importées
par Maven.

### Contexte de persistance

Un contexte de persistance (_persistance context_) est un ensemble d'entités chargées depuis une base de données ou sauvegardées dans une base de données.

En Hibernate, il est représenté par une instance de [org.hibernate.Session](https://docs.jboss.org/hibernate/orm/current/javadocs/org/hibernate/Session.html). Dans JPA, il s'agit d'une instance 
de [jakarta.persistence.EntityManager](https://jakarta.ee/specifications/platform/9/apidocs/?jakarta/persistence/EntityManager.html)
Quand Hibernate est utilisé comme implantation de JPA et qu'on utilise un _EntityManager_ dans le code, c'est implicitement un objet _Session_ qui
est manipulé.

L'_EntityManager_ associé à un contexte de persistance peut être obtenu en annotant un objet de type _EntityManager_ avec
@PersistenceContext.
    
```java
@PersistenceContext
private EntityManager entityManager;
```

Notez que, par défaut, un contexte de persistance vit (existe) le temps d'une transaction. Sa portée peut être modifiée
via un attribut de l'annotation @PersistenceContext.
        
### États possibles d'une entité

A chaque entité manipulée dans l'application, on associe un des 4 états suivants :

* _transient_ : l'entité n'est pas et n'a jamais été attaché à un contexte de persistance. Cette entité n'a pas de 
correspondance dans la base de données. Cet état correspond à celui d'un objet Java qui vient d'être créé.

* _persistent_ : l'entité est attachée à un contexte de persistance; après un _flush_ sur l'_EntityManager_, l'entité 
correspondante aura une entrée à jour dans la base de données.

* _detached_ : l'entité a été attachée à un contexte de persistance mais ne l'est plus; c'est le cas lorsque l'instance
a été persistée au sein d'une transaction qui s'est achevée;

* _removed_ : l'entité a été supprimée et disparaîtra effectivement de la base de données au prochain _flush_.

Quand l'entité est dans l'état _persistent_, tous les changements effectués sur ses attributs seront répercutés sur
l'enregistrement correspondant en base de données au prochain _flush_ qui sera automatiquement déclenché en fin de vie
du contexte de persistance ou manuellement par un appel à _flush_ sur  l'_EntityManager_.

### Quelques méthodes de JPA et leur implantation chez Hibernate

On considère la classe ci-dessous définissant des entités :

```java
@Entity
public class Person {
 
    @Id
    @GeneratedValue
    private Long id;
 
    private String name;
 
    // ... getters and setters
 
}
```

Dans cette section, on discute du comportement des différentes méthodes offertes pour manipuler les entités précédentes 
dans un contexte de persistance. Ces méthodes requièrent un contexte transactionnel pour s'exécuter lorsque 
la portée du contexte de persistance est réglé sur "transactionnel".

#### Persist

Cette méthode vise à changer l'état d'une entité de _transient_ à _persistent_ et donc à ajouter cette entité au
contexte de persistance.

```java
Person person = new Person();
person.setName("John");
entityManager.persist(person);
```

Au moment où la transaction associée au contexte de persistance s'achèvera ou si un _flush_ est déclenché, une requête
SQL de type INSERT est exécutée et provoque l'ajout en base de l'instance de Person. 

Notez que le retour de _persist_ est _void_. La variable _person_ référence l'objet persisté. Quel est l'effet de la méthode sur l'état de l'entité associée au paramètre ?

* une entité _transient_ devient _persistent_;

* si l'entité est déjà _persistent_ alors _persist_ n'a aucun effet;

* si l'entité est _detached_  alors un appel à _persist_ va lever une exception.

#### Merge

L'intention de cette méthode est de mettre à jour une entité _persistent_ en fixant la valeur de ses attributs 
en fonction d'une entité _detached_.

```java
Person person = new Person();
person.setName("John");
entityManager.persist(person);

entityManager.evict(person); // on force l'état "detached"
person.setName("Mary");
 
Person mergedPerson = (Person) entityManager.merge(person);
```

L'exécution du _merge_ provoque d'abord la recherche d'une entité dont l'id est celui de l'objet en paramètre 
(cette entité est soit récupérée du contexte de persistance, soit directement de la base de données); 
la valeur des attributs de l'objet en paramètre du _merge_ est alors copiée dans l'objet retrouvé à l'étape initiale; l'entité mise à jour est retournée.

Dans l'exemple, sont donc manipulés deux objets différents de type Person : celui reçu en paramètre du _merge_
et celui retourné qui appartient au contexte de persistance et qui a été mise à jour. Quel est l'effet de la méthode sur l'état de l'entité associée au paramètre ?

* si l'entité est _detached_  alors elle est copiée dans une entité _persistent_.

* si l'entité est _transient_ alors elle est copiée dans une nouvelle entité _persistent_;

* si l'instance est déjà _persistent_ alors _merge_ n'a aucun effet sur son état.

#### Refresh

La méthode _refresh_ permet de refraîchir l'état de l'entité reçue en paramètre à partir des informations
de la base de données, écrasant ainsi les modifications faites depuis. 

#### Cycle de vie JPA

Le diagramme suivant représente les différents états JPA possibles pour les entités et l'effet des différentes
méthodes sur celui-ci.

![cycle](https://i.imgur.com/CQklpNZ.png)

#### Méthodes spécifiques à Hibernate

Hibernate implante JPA mais fournit aussi des méthodes additionnelles qui ne font pas partie de la spécification : 
_SaveOrUpdate_, _Update_, _Save_, ... Pour se renseigner sur ces méthodes, on pourra lire le lien suivant :

<http://www.baeldung.com/hibernate-save-persist-update-merge-saveorupdate>

Notez qu'il est préférable de se contenter de _persist_, _merge_ et des différentes méthodes décrites dans JPA car cela
garantit qu'on pourrait quitter Hibernate et changer d'implantation de JPA sans avoir à revoir profondément son application. 
On respectera ce principe durant les TPs.

### Références

Exemples de code utilisant JPA et son _EntityManager_ : <https://www.concretepage.com/java/jpa/jpa-entitymanager-and-entitymanagerfactory-example-using-hibernate-with-persist-find-contains-detach-merge-and-remove>

Documentation Hibernate : <http://docs.jboss.org/hibernate/entitymanager/3.6/reference/en/html_single/>

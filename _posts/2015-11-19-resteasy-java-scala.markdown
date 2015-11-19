---
layout: post
title:  "Traduire une API REST de Java à Scala"
date:   2015-11-19 08:00:00
author: Olivier Lafleur
comments: true
published: false
---

À la suite de [mon dernier article](/2015/11/11/scala-java.html), [un lecteur](https://twitter.com/JoelTHebert) m'a
posé la (très intéressante) question suivante :

> J'aimerais bien voir comment tu t'y prendrais pour faire la conversion d'un back end Java vers Scala. Je me demande à quel point les librairies comme RESTEasy ou Spring pourrait encore être utilisées.

En effet, un des avantages forts de Scala est l'intéropérabilité transparente
avec Java, ce qui fait que l'on peut facilement inclure du Scala dans un projet
Java.

J'ai donc décidé de partir d'une [API REST Java de base](https://github.com/olafleur/convert-java-restapi-to-scala/tree/java_initial),
bâtie avec RESTEasy (et récupérée dans [les exemples de base](https://github.com/resteasy/Resteasy/tree/3.0.13.Final/jaxrs/examples/oreilly-workbook/ex03_1) de la librairie), et
de convertir les fichiers qui font l'application à proprement parler en Scala. Par ailleurs,
je me suis aussi mis comme contrainte de garder mes objets de domaine en Java et
de garder le test qui était déjà dans le projet intact.

Je me suis forcé à garder ces éléments-là puisque dans mon contexte professionnel, nous
utilisons Java au niveau du client et au niveau du serveur, avec des objets du
domaine d'affaires qui sont partagés entre les deux et qui sont bâtis en Java.
Le framework que nous utilisons au niveau client ([GWT](http://www.gwtproject.org/))
n'est pas en mesure (à ce que je sache) de compiler du Scala au niveau du client[^1].
Ainsi, en gardant des objets de domaine (et accessoirement, un test de REST API)
en Java, on garde la compabilité avec ce framework tout en changeant notre backend
de façon transparente.

### Traduction

#### Comment procéder

La première étape sera de simplement renommer les fichiers à convertir, soit `CustomerResource` et
`ShoppingApplication` en changeant le suffixe de `.java`à `.scala`. Vous vous doutez
bien qu'à ce point-ci dans le processus, notre IDE n'est pas très heureux de ça et
indiquera plein de rouge (des erreurs) un peu partout.

Voici donc un exemple de fonction :

<pre><code class="java">@POST
@Consumes("application/xml")
public Response createCustomer(InputStream is) {
   Customer customer = readCustomer(is);
   customer.setId(idCounter.incrementAndGet());
   customerDB.put(customer.getId(), customer);
   System.out.println("Created customer " + customer.getId());

   return Response.created(URI.create("/customers/" + customer.getId())).build();
}
</code></pre>

#### Annotations
La première chose à remarquer est que les annotations fonctionneront (essentiellement)
de façon identique. La seule différence est que nous devrons dire explicitement
que le type du contenu est dans un tableau, en mettant `Array("application/xml")`.
C'est probablement le seul endroit où cela prendra *plus* de code pour exprimer la
même chose.

#### Fonctions et variables
Pour ce qui est du type de retour de la fonction, il n'est plus nécessaire puisque
automatiquement inféré. On mettra simplement `def` à la place de `public Response`.

Même chose pour les déclarations de variables. On remplacera `Customer customer`
par `val customer`[^2].

Lors de l'utilisation de paramètres, il faut définir le type, ce que l'on fait
en mettant le type à la fin (`inputStream: InputStream`) plutôt qu'au début
(`InputStream inputStream`).

Le mot-clé `return`, quant à lui, n'est plus nécessaire puisque Scala retourne automatiquement
la dernière valeur calculée.

Lorsqu'une fonction
n'a pas de paramètre, on peut enlever les parenthèses de l'appel à la fonction, créant
volontairement une ambiguïté entre ce qu'est un appel à une fonction et une variable
membre.

Finalement, pour imprimer à la console, le `System.out` n'est plus nécessaire.
Ce qui nous amène donc à la fonction suivante.

<pre><code class="scala">@POST
@Consumes(Array("application/xml"))
def createCustomer(inputStream: InputStream) = {
  val customer = readCustomer(inputStream)
  customer.setId(idCounter.incrementAndGet)
  customerDB.put(customer.getId, customer)
  println("Created customer " + customer.getId)

  Response.created(URI.create("/customers/" + customer.getId)).build()
}
</code></pre>

#### Boucle
Plus loin dans la même classe, nous avons une boucle. La syntaxe en Scala est
légérement différente de Java. L'en-tête de la boucle `for` passe de

`for (int i = 0; i < nodes.getLength(); i++)`

à

`for (i <- 0 until nodes.getLength)`,

ce qui n'est pas très dépaysant mais à mon sens pas mal plus lisible.

#### _Pattern matching_
Une des forces de Scala (et un des plus par rapport à Java) est ce qu'on appelle le
_pattern matching_. C'est une sorte de `switch`, mais que l'on peut étendre à
n'importe quel type (donc pas seulement les types primitifs ou permis par le
  compilateur) et qui permet d'extraire les variables contenu dans l'objet.

Par exemple, dans l'exemple ci-dessous, où nous avons une vérification de `null`
à faire en Java, nous pouvons directement traiter la variable comme un `Option`
(l'équivalent de `Optional` en Java 8) et extraire le `Customer` de celui-ci. Ainsi,

<pre><code class="java">if (customer == null) {
    throw new WebApplicationException(Response.Status.NOT_FOUND);
}

return new StreamingOutput() {
    public void write(OutputStream outputStream) throws IOException, WebApplicationException {
      outputCustomer(outputStream, customer);
    }
};
</code></pre>

devient

<pre><code class="scala">customer match {
  case Some(cust) => new StreamingOutput() {
    def write(outputStream: OutputStream) {
      outputCustomer(outputStream, cust)
    }
  }

  case None => throw new WebApplicationException(Response.Status.NOT_FOUND)
}
</code></pre>

Cela fait essentiellement le tour des styles de modifications à faire dans `CustomerResource`.

La conversion du fichier `ShoppingApplication` est elle aussi simple. La chose
particulièrement intéressante que celle-ci montre est que, non seulement on peut
appeler des objets et fonctions Java en Scala de façon transparente (et vice-versa),
on peut aussi faire de l'héritage de l'un à l'autre de façon transparente. Dans
ce cas-ci, nous héritons de la même classe Java que dans l'application initiale.

<pre><code class="scala">class ShoppingApplication extends Application {
   var singletons = new util.HashSet[AnyRef]()
   val empty = new util.HashSet[Class[_]]()

   singletons.add(new CustomerResource)

   override def getSingletons = singletons

   override def getClasses = empty
}
</code></pre>

Autre petite remarque, il n'est pas nécessaire de définir un constructeur par
défaut et il est permis de mettre du code directement dans la classe. Ainsi, la
ligne de code `singletons.add(new CustomerResource)` serait en Java une ligne
incluse dans un constructeur sans paramètre.

### Conclusion
Je vous invite à faire la comparaison et regarder la [version finale](https://github.com/olafleur/convert-java-restapi-to-scala) du code converti
ou même à voir [le _diff_ des deux projets](https://github.com/olafleur/convert-java-restapi-to-scala/compare/java_initial...master).
Bien entendu, le code final est loin d'être du Scala idiomatique, mais cela démontre bien que ce
n'est pas sorcier de faire la conversion de l'un à l'autre. On peut ainsi aisément imaginer une
équipe qui ferait une transition graduelle de Java à Scala de façon transparente et évolutive.
Dans le projet qui nous concerne, le processus de _build_ était exactement le même dans les
deux cas : `mvn clean install`[^3]. Et puis après tout, comme le dit David Pollack[^4],

> Scala is really “just another Java library”.  Just stick scala-library.jar on your classpath and all of your Scala classes should be readily available within your Java application.

---
---

[^1]: Il faudrait, j'imagine, en faire la démonstration, puisque je ne l'ai jamais essayé. Pour faire du développement frontend avec Scala, je vous recommande la librairie [Scala.js](http://www.scala-js.org/), qui pourrait éventuellement faire l'objet d'un autre article.

[^2]: Pour les lecteurs attentifs, vous remarquerez que l'on utilise `val`, qui est censé indiqué l'immutabilité d'un objet, mais nous effectuons un `setId` à la ligne d'après. En fait, c'est que ce qui n'est pas permis comme modification c'est la réassignation d'une variable (ce que `var` permet). Ainsi, si vous voulez vraiment qu'un objet soit immutable, il ne faut pas lui donner de _setter_.

[^3]: En Scala idiomatique, on utilise plutôt le *Scala Build Tool* (sbt) que Maven, par contre.

[^4]: Cité par Daniel Spiewak dans [son excellent article très complet](http://www.codecommit.com/blog/java/interop-between-java-and-scala) sur l'intéropérabilité entre Java et Scala, allant même jusqu'à comparer le résultat de la compilation sur la JVM.

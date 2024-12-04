---
layout: post
title:  "Scala pour le web : Scala.js"
date:   2015-12-04 08:00:00
author: Olivier Lafleur
---

Lorsque l'on développe des applications web, JavaScript est essentiellement un passage obligé[^1], et on doit soit utiliser ce langage en soi ou utiliser un autre langage qui va compiler en JavaScript ([il en existe une panoplie](https://github.com/jashkenas/coffeescript/wiki/list-of-languages-that-compile-to-js)).

Il y a par contre plusieurs propriétés que certains développeurs n'apprécient pas tellement de JavaScript : le typage dynamique, [l'héritage par prototypes](https://fr.wikipedia.org/wiki/Programmation_orient%C3%A9e_prototype#Exemple_:_l.27h.C3.A9ritage_en_JavaScript) (et non pas l'héritage hiérarchique comme dans la majorité des langages les plus populaires) et le fait que le langage agisse parfois de façon inattendue[^2], entre autres.

Scala.js est une tentative de régler une partie des problèmes de JavaScript tout en permettant d'utiliser Scala du côté client et du côté serveur. Ainsi, pour les développeurs travaillant sur un projet découpé de cette façon, il n'est pas nécessaire de changer de langage pour réaliser l'application au complet.

### Mise en place de Scala.js

L'outil de *build* pour Scala le plus utilisé est [SBT](http://www.scala-sbt.org/). Il est donc recommandé [de l'installer](http://www.scala-sbt.org/download.html) et de créer un nouveau projet SBT, soit [de façon manuelle](http://www.scala-sbt.org/0.13/tutorial/Hello.html) ou avec votre éditeur (par exemple, sur IntelliJ, il faut installer le plugin Scala qui permet de créer un projet SBT avec un *wizard*).

Par la suite, il faut ajouter cette ligne dans le fichier `project/plugins.sbt` (en remplaçant `0.6.5` par la dernière version) pour installer le plugin de Scala.js dans votre application :

<pre><code class="scala">addSbtPlugin("org.scala-js" % "sbt-scalajs" % "0.6.5")
</code></pre>

Finalement, il faut activer le plugin en ajoutant la ligne suivante dans le fichier `build.sbt` :

<pre><code>enablePlugins(ScalaJSPlugin)</code></pre>

### Bonjour, monde !

Une fois que le projet est créé, nous pouvons maintenant créer notre première page web avec Scala.js.

Tout d'abord, nous allons créer notre classe Scala qui sera le point d'entrée de notre application web.

<pre><code class="scala">import scala.scalajs.js.JSApp

object MonApp extends JSApp {
  def main(): Unit = {
    println("Bonjour, monde")
  }
}
</code></pre>

Vous vous doutez bien que ceci fera afficher dans la console du navigateur le simple message `Bonjour, monde`.

Pour que ce code Scala devienne du JavaScript, vous devrez taper `sbt` en ligne de
commande (on l'a installé plus tôt, vous vous souvenez?) puis taper la commande
qui générera le JavaScript de la façon la plus rapide possible (et non pas nécessairement le fichier le plus petit) : `fastOptJS`.[^3]

Pour que le fichier JavaScript soit regénéré automatiquement à chaque fois que le code Scala est sauvegardé, il suffit d'ajouter un tilde devant la commande lors de l'exécution : `~fastOptJS`.

Maintenant, il manque une partie essentielle d'une application web : le fichier HTML.

Voici un exemple de fichier HTML qui ferait l'affaire :
<pre><code class="html">&lt;!DOCTYPE html>
&lt;html>
  &lt;head>
    &lt;meta charset="UTF-8">
    &lt;title>Test de Scala.js</title>
  &lt;/head>
  &lt;body>
    &lt;div id="test"></div>
    &lt;!-- On inclut le code Scala.js compilé -->
    &lt;script type="text/javascript" src="./target/scala-2.11/scala-js-tutorial-fastopt.js"></script>
    &lt;!-- On appelle la fonction générée en JavaScript -->
    &lt;script type="text/javascript">
      MonApp().main();
    &lt;/script>
  &lt;/body>
&lt;/html>
</code></pre>

### Intéropérabilité avec JavaScript

On sait que Scala est intéropérable de façon transparente avec Java. De la même façon, une fonctionnalité très intéressante de Scala.js est le fait que celui-ci soit intéropérable de façon transparente avec JavaScript.
En effet, il contient un type (`js.Dynamic`) qui permet de mettre du JavaScript tel quel dans le code. Par exemple, à l'intérieur d'un fichier Scala, on pourrait mettre ce code-ci:
<pre><code class="scala">import scala.scalajs.js
//...
val document = js.Dynamic.global.document
val testDiv = document.getElementById("test")

val nouveauP = document.createElement("p")
nouveauP.innerHTML = "Ce message est ajouté par JavaScript"
testDiv.appendChild(nouveauP)
</code></pre>

Ce qui est intéressant de noter, c'est que toutes les fonctions qui sont appelées dans ce code sont du JavaScript pur, c'est à dire typées dynamiquement et sans *hint* de l'éditeur. Ainsi, si on y glisse une erreur, ce n'est qu'à l'exécution que l'erreur sera détectée.

### Manipulation du DOM

Ainsi, il serait probablement plus souhaitable de pouvoir faire le même genre de traitement, mais en ayant une validation à la compilation.

C'est tout à fait possible, en ajoutant une librairie pour Scala.js dans notre fichier `build.sbt` :

<pre><code class="scala">libraryDependencies += "org.scala-js" %%% "scalajs-dom" % "0.8.0"
</code></pre>

À quoi ressemblerait le code ci-haut, mais avec cette librairie?

<pre><code class="scala">import org.scalajs.dom.document
//...
val testDiv = document.getElementById("test")

val nouveauP = document.createElement("p")
nouveauP.innerHTML = "Ce message est ajouté par JavaScript"
testDiv.appendChild(nouveauP)
</code></pre>

Comme vous le voyez, le code est identique, à l'exception du `import` et de la ligne
<pre><code class="scala"> val document = js.Dynamic.global.document
</code></pre>
qui n'est plus nécessaire.
Par contre, cette fois-ci, lorsque nous sommes dans l'éditeur et qu'une erreur s'y glisse, le compilateur nous avertit immédiatement.

### ScalaTags

Typiquement (en particulier lors de la réalisation d'une [*Single Page Application*](https://fr.wikipedia.org/wiki/Application_web_monopage)), on ne veut pas mettre tout notre HTML initial dans la page de départ. On voudrait pouvoir générer du HTML de façon dynamique à partir de Scala.
Il existe une librairie qui permet non seulement de le faire, mais aussi de le faire de façon fortement typée. Elle s'appelle [ScalaTags](http://lihaoyi.github.io/scalatags/).

Par exemple :

<pre><code class="scala">document.body.appendChild(
      div(
        h1(id:="titre", "Ceci est un titre"),
        p("Ceci est un paragraphe de texte")
      ).render
    )
</code></pre>

### Conclusion

Comme vous voyez, on peut avoir bien du plaisir avec cette librairie. Bien entendu, ceci n'est qu'un rapide tour d'horizon de quelques fonctionnalités de base, n'hésitez pas à explorer [la documentation et les exemples](http://www.scala-js.org/tutorial/) afin d'en apprendre plus. Par ailleurs, il existe aussi d'excellentes conférences ([comme celle-ci](https://www.youtube.com/watch?v=9SalPdAEI28)) qui permettent de voir de façon plus interactive de quoi il en retourne. Amusez-vous bien!

---
---

[^1]: Du moins, jusqu'à l'utilisation de [WebAssembly](https://medium.com/javascript-scene/what-is-webassembly-the-dawn-of-a-new-era-61256ec5a8f6) par les navigateurs.

[^2]: [Ce petit vidéo classique](https://www.destroyallsoftware.com/talks/wat) de [Gary Bernhardt](http://destroyallsoftware.com) en donne un aperçu, pour JavaScript, et aussi pour Ruby.

[^3]: La commande en production pour avoir le fichier JavaScript le plus petit (mais qui prendra plus de temps à compiler) est `fullOptJS`

**Merci** à JF Bourget et JC Larivière pour leurs commentaires, qui m'ont permis d'améliorer cet article.

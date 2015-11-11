---
layout: post
title:  "Scala pour les développeurs Java"
date:   2015-11-11 08:00:00
author: Olivier Lafleur
comments: true
---

![Logo de Scala](/images/header-scala.png)

Ces jours-ci, j'explore plusieurs langages, et un que j'apprécie particulièrement
est Scala. Fait intéressant : dépendant comment on l'utilise, il peut être très
similaire ou très différent de Java. Dans cet article, je vous présenterai les
similitudes afin de montrer comment un développeur Java pourrait vouloir
transitionner (c'est une bonne première étape) vers ce langage.

### Fonctionnel ou pas?
Tout d'abord, il est intéressant de noter que Scala a la réputation d'être un
langage fonctionnel (comme [Haskell](https://www.haskell.org), par exemple).
Ce n'est pas faux de dire que le langage contient les outils qu'il faut pour faire de la programmation
fonctionnelle, mais la "pureté" fonctionnelle n'est pas requise. En fait, il
est tout à fait possible de rédiger du code qui serait relativement familier dans son intention
à un développeur Java (mais de façon plus concise).

Scala est reconnu comme un langage multi-paradigme puisque l'on peut notamment faire de la programmation
orientée-objet, de la programmation impérative ainsi que de la programmation
fonctionnelle.

### Qui l'utilise?
Lorsqu'on étudie un langage, il est toujours intéressant, puisque la programmation
est foncièrement un acte social, de voir s'il y a une communauté et des
entreprises qui l'utilisent. Parmi la liste des entreprises qui, à un moment
ou un autre l'ont utilisé : Twitter, LinkedIn, The Guardian, FourSquare, etc.

Il est utilisé comme langage en production et parfois même en remplacement
de d'autres langages (comme Ruby)
[lorsque ceux-ci ne sont plus "suffisants"](https://www.quora.com/Why-did-twitter-move-away-from-Ruby-on-Rails/answer/Benjamin-Darfler).
Il s'agit d'un langage qui existe
maintenant depuis 12 ans et qui a acquis une certaine reconnaissance dans le
milieu des développeurs.

### Intéropérabilité avec Java
Un des éléments les plus intéressants, à mon avis, pour les gens qui réfléchissent à utiliser
Scala, est que c'est un langage qui compile sur la JVM, comme Java, et que celui-ci
est parfaitement intéropérable avec Java. Ainsi, on peut utiliser des classes
Java en Scala et vice-versa de façon tout à fait naturelle. En fait, si on
veut simplifier à l'extrême, pour ajouter Scala à un projet Java déjà existant,
il suffit d'ajouter un JAR (ou une dépendance) à votre projet et vous
êtes bons pour commencer.


### Syntaxe
Scala est un langage fortement typé, tout comme Java, avec une inférence de type
qui permet de souvent omettre les noms des types sans perdre le filet de sécurité qu'un
typage fort permet.

#### Variables
Ainsi, lorsque l'on déclare une variable, là où en Java nous mettrions le type
du côté gauche de l'égalité, le type est automatiquement inféré. Ce que l'on doit
mettre est soit `val` ou `var`, dépendant si la variable est mutable ou non.
Cela nous permet donc d'enforcer l'immutabilité, si c'est ce que nous cherchons.

Par exemple, cette déclaration en Java :
<pre><code class="java">Dragon dragon = new Dragon();
</code></pre>
est remplacée par celle-ci en Scala :
<pre><code class="scala">var dragon:Dragon = new Dragon
//ou même ceci grâce à l'inférence de type
var dragon = new Dragon
</code></pre>

Lorsque l'on veut déclarer un objet comme étant immutable (ce qui est en
général une propriété souhaitable en programmation fonctionnelle), en Java
nous devons utiliser l'opérateur `final` alors qu'en Scala il suffit simplement
d'écrire ceci :
<pre><code class="scala">val dragon = new Dragon
</code></pre>

Comme vous le remarquez sans doute, les points virgules ne sont pas nécessaires
à la fin des instructions. Il suffit simplement de mettre chaque ligne de code
sur sa propre ligne. De plus, on voit aussi que les parenthèses ne sont pas
nécessaires à l'appel d'une fonction (ou d'un constructeur dans ce cas) s'il
n'y a pas de paramètre.

#### Fonctions
Pour déclarer une fonction, il suffit d'utiliser le mot clé `def` pour ce faire :
<pre><code class="scala">//:String n'est pas nécessaire puisque inféré
def mange(repas: String):String = "Votre dragon a mangé un " + repas
</code></pre>

là où nous aurions écrit ceci en Java :
<pre><code class="java">public String mange(String repas) {
  return "Votre dragon a mangé un " + repas;
}
</code></pre>

Cet exemple nous fait voir qu'en Scala, le `return` n'est jamais obligatoire
puisque le résultat de la dernière ligne est retourné (et donc le type de retour
  peut aussi être inféré).

#### Classes
Un des endroits ou Scala brille le plus, à mon avis, c'est dans la définition des
classes. En Java, nous avons à notre disposition des classes (`class`) et des
interfaces (`interface`).

En Scala, il y a trois types principaux : `class`, `object` et `trait`.

De façon simpliste, `trait` est comparable aux interfaces en Java et `class`
est comparable à une classe,
mais sans les éléments qui seraient `static` en Java. L'élément `object`, quant
à lui, contient ceux-ci.

Tout comme les interfaces en Java 8, les traits peuvent avoir des fonctions qui
ont une implémentation. Par exemple :
<pre><code class="scala">trait PeutManger {
  def mange():Unit

  def digerer() = "Miam, c'était délicieux"
}
</code></pre>

En Java, cela aurait l'air de ceci :
<pre><code class="java">public interface PeutManger {
    void mange();

    static void digerer() {
        System.out.println("Miam, c'était délicieux");
    }
}
</code></pre>

Maintenant, créons-nous, dans les deux cas, un objet ordinaire (POJO) avec des
*getter/setter*, l'implémentation d'une "interface" et une fonction statique
qui agit comme un *Builder*.

En Java :
<pre><code class="java">public class Dragon implements PeutManger {
    private String nom;
    private int age;

    public Dragon(String nom, int age) {
        this.nom = nom;
        this.age = age;
    }

    public String getNom() {
        return nom;
    }

    public void setNom(String nom) {
        this.nom = nom;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public void mange() {
        System.out.println("Votre dragon " + nom + " a mangé");
    }

    public static Dragon buildDummyDragon() {
        return new Dragon("Ami", 42);
    }
}
</code></pre>

devient en Scala :

<pre><code class="scala">class Dragon(_nom: String, _age: Int) extends PeutManger {
  def nom = _nom
  def age = _age

  def age_= (value:Int) = age = value
  def nom_= (value:String) = nom = value

  override def mange() = println("Votre dragon " + _nom + " a mangé")
}

object Dragon {
  def construireDragon() = new Dragon("Ami", 42)
}
</code></pre>

En terme de bénéfice additionnel, en plus d'être (à mon goût) plus élégant comme
code, le "setter" en Scala permet de faire directement une assignation grâce à
la syntaxe `dragon.age = 15`, par exemple, mais en passant par le biais d'une
fonction.

### Conclusion
Dans cet article, nous avons abordé la base de Scala d'un point de vue des
similitudes du langage à Java. Cela nous permet de faire une première approche
du langage qui permette de nous familiariser avec la structure de ce langage.
Dans le prochain article, nous commencerons à explorer les éléments particuliers
de Scala qui nous permettent d'utiliser de nouveaux outils dans la résolution
de nos problèmes informatiques.

---
layout: post
title:  "Héritage prototypal vs hiérarchique"
date:   2015-12-11 08:00:00
author: Olivier Lafleur
comments: true
published: true
---

[Dans un précédent article](/2015/12/04/scala-web-scalajs.html), j'ai parlé de certaines différences que JavaScript a par rapport à d'autres langages.
Une de ces différences est l'héritage, qui est non pas hiérarchique, mais bien prototypal.

### Rappel : héritage hiérarchique ("classique")
Tout d'abord, rappelons-nous ce qu'est l'héritage classique.

Dans ce cas, on part d'une classe parente `Animal`, par exemple, de laquelle on hérite une classe `Chat` et une classe `Chien`. La classe `Animal` peut (ou pas) être abstraite, mais dans tous les cas, n'est pas obligé d'être instantiée. Par la suite, on peut avoir des objets concrets, instances de `Chat` et `Chien`.

<pre><code class="java">public abstract class Animal {
    protected int age = 0;

    public void naitre() {
        System.out.println("L'animal naît");
    }

    public abstract void grandir();
}

public class Chat extends Animal {
    @Override
    public void grandir() {
        age += 7;
    }

    public void miauler() {
        System.out.println("Miaou!");
    }
}

public class Chien extends Animal {
    @Override
    public void grandir() {
        age += 8;
    }

    public void japper() {
        System.out.println("Wouaf!");
    }
}
</code></pre>

On a donc une simple définition de classe avec un champ `age`, une fonction concrète `naitre()` et finalement une fonction `grandir()` qui doit être implantée dans chaque cas. Finalement, chaque classe fille a ses particularités (`japper()` ou `miauler()`, respectivement).

Rien de trop surprenant là-dedans, il s'agit plus ou moins d'un exemple académique d'héritage simple en Java. La classe est définie dans un "patron", puis elle est instantiée concrètement, au bon vouloir du développeur.

### Héritage prototypal

Lorsqu'on entre dans l'héritage par prototypes en JavaScript[^1], la différence principale est que la structure d'une classe donnée est beaucoup plus flexible, les types étant dynamiques. Plutôt que de définir de façon rigide la structure d'une classe, on peut la modifier (ajouter/supprimer des membres ou des fonctions).
Par exemple, si on reprend le même exemple que tantôt, mais en JavaScript, cela nous donnerait :

<pre><code class="javascript">var Animal = function() {
    this.age = 0;
    this.naitre = function() {
        console.log("L'animal naît");
    };
};

var Chien = function() {
    this.grandir = function() {
        this.age += 8;
    };
    this.japper = function() {
        console.log("Wouaf!");
    };
};

var Chat = function() {
    this.grandir = function() {
        this.age += 7;
    };
    this.miauler = function() {
        console.log("Miaou!");
    };
};

Chien.prototype = new Animal();
Chat.prototype = new Animal();
</code></pre>

On retrouve ici essentiellement les mêmes idées que plus haut.
Tout d'abord, on définit une classe `Animal`, mais celle-ci n'est plus abstraite. On pourrait y définir une fonction `grandir()` comme plus haut, mais celle-ci devrait avoir déjà une implémentation concrète qui serait écrasée par les instances qui héritent de ce prototype qui implémentent cette fonction.
Par ailleurs, on voit à la fin que l'on fait "hériter" les classes `Chien` et `Chat` du prototype `Animal`.
Ainsi, celles-ci récupèrent les champs et méthodes de la classe `Animal`, sans toutefois écraser ceux qui ont le même nom dans les classes `Chat` et `Chien`.
On peut donc exécuter le code suivant, qui agira de façon prévisible :

<pre><code class="javascript">var kitty = new Chat();
var boris = new Chien();

kitty.naitre(); //"L'animal naît"
boris.naitre(); //"L'animal naît"
kitty.grandir();
boris.grandir();
kitty.miauler(); //"Miaou!"
boris.japper(); //"Wouaf!
console.log(kitty.age); //7
console.log(boris.age); //8
</code></pre>

Là où ça devient différent, c'est lorsque l'on se met à jouer avec les champs d'une instance concrète ou d'un prototype.
Il est par exemple possible de supprimer ou ajouter un champ d'un ou l'autre. Dans le cas suivant, on supprime la fonction du prototype. Ce prototype étant la référence de tous les objets créés avec celui-ci, cette fonction sera supprimée de toutes les instances de ce prototype (sans affecter les instances de `Chien`).
<pre><code class="javascript">delete Chat.prototype.naitre;

kitty.naitre(); //Ici, on obtient une erreur
</code></pre>

On peut aussi ajouter un champ à une seule instance d'un objet, comme dans le cas suivant. Un autre objet créé avec le même prototype (`Chat`), n'aurait pas ce champ.
<pre><code class="javascript">kitty.poil = "soyeux";

console.log(kitty.poil); //"soyeux"
</code></pre>

Finalement, on peut aussi ajouter une fonction à un prototype, ce qui affectera ainsi toutes les instances de cette classe.

<pre><code class="javascript">Chat.prototype.salive = function() {
  console.log("baveuse");
};

kitty.salive(); //"baveuse"
</code></pre>

### Problème
Un des problèmes majeurs que l'héritage par prototypes apporte est qu'il est difficile d'avoir une garantie qu'un champ sera bel et bien accessible ou que celui-ci contiendra une valeur. La structure malléable d'un prototype rend la tâche plus ardue à ce niveau.

### Avantage
Dans certains contextes où l'on veut essayer des choses et développer rapidement, il peut être très utile de simplement créer un objet en s'inspirant d'un objet existant auquel on veut juste ajouter ou supprimer quelques éléments. Ainsi, il est facile de le faire, sans pour autant modifier la définition initiale de la classe comme on serait obligé de le faire en héritage classique.

### Conclusion
L'héritage par prototypes n'est pas quelque chose auquel les développeurs qui ne font pas du JavaScript sont habitués. Cependant, lorsqu'on s'y met, on s'aperçoit que ce n'est pas plus compliqué que l'héritage hiérarchique. Il s'agit simplement de penser d'une façon différente lors de la conception du code. Finalement, ce n'est peut-être pas une mauvaise chose puisqu'après tout, des paradigmes différents, c'est ce qui nous enrichit en tant que programmeur, pas vrai?

---
---

[^1]: À prendre en note : je ne suis pas un expert en JavaScript, alors tout ce que je raconte et explique à propos de l'héritage par prototypes tel que pratiqué en JavaScript est basé sur mon expérience (ma foi, limitée) en JavaScript. N'hésitez pas à me corriger si je dis des âneries.

**Merci** à JC Larivière pour ses commentaires, qui m'ont permis d'améliorer cet article.

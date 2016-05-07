---
layout: post
title:  "Bâtir son propre framework de test"
date:   2016-05-10 08:00:00
author: Olivier Lafleur
comments: true
published: false
---

Une des librairies les plus utilisées (si ce n'est la plus utilisée) dans le monde des programmeurs Java est JUnit. Il s'agit d'un _framework_ de tests (pas nécessairement unitaires) qui nous permet de rédiger des tests de ce genre :

<pre><code class="Java">@Test
public void unTest() {
    Pile pile = new Pile();

    assertTrue(pile.estVide());
}
</code></pre>

C'est un outil très pratique, mais pour beaucoup de développeurs, cette librairie (tout comme beaucoup d'autres outils des programmeurs) ont parfois un aspect magique, ou on se dit qu'on ne pourrait certainement pas réaliser quelque chose du genre.

Un des éléments qui est, je crois, important en informatique, est que l'on devrait comprendre (au moins à haut niveau) comment fonctionnent les outils que l'on utilise. On devrait utiliser les outils non pas parce que qu'on ne sait pas comment les faire, mais plutôt, parce que qu'on n'a pas le goût de les réécrire. Dans la majorité des cas, il est beaucoup plus avantageux de réutiliser un outil déjà existant, qui a déjà été utilisé par une quantité importante de personne, qui a été pensé, structuré et _débuggé_ depuis déjà un certain temps.

Je me suis donc mis en tête de comprendre un peu mieux comment un framework de test pourrait être bâti.

Tout d'abord, quels sont les éléments qui composent un tel _framework_ (bien que petit est incomplet)? On devrait :

1. Avoir accès à des fonctions qui valident le résultat du test comme `assertTrue` ou `assertEquals`
1. Pouvoir rouler tous les tests d'une classe, peu importe leur nom ou leur nombre.
1. Notre framework ne devrait pas planter si un test est en erreur (s'il lève une exception, par exemple).

La première chose que l'on pourrait faire est écrire une classe de tests telle qu'un développeur qui utilise notre framework l'écrirait.

<pre><code class="Java">import static com.olivierlafleur.testtest.MonFrameworkDeTest.verifieEgal;
import static com.olivierlafleur.testtest.MonFrameworkDeTest.verifieVrai;

public class ClasseDeTests {
    //Ce test devrait être en erreur puisqu'il lève une exception
    public void divisionParZeroTest() {
        int var1 = 1;
        int var2 = 2;

        int res = 2/0;

        verifieVrai(var2 > var1);
    }

    //Ce test devrait passer
    public void unAutreTest() {
        int var1 = 1;
        int var2 = 2;

        verifieVrai(var2 > var1);
    }

    //Ce test devrait échouer
    public void unAutreTest2() {
        int var1 = 1;
        int var2 = 2;

        verifieVrai(var1 > var2);
    }

    //Ce test devrait passer
    public void unTestEgalite() {
        verifieEgal(2, 2);
    }

    //Ce test devrait échouer
    public void unTestInEgalite() {
        verifieEgal(1, 2);
    }
}
</code></pre>

Alors voici donc les éléments à remarquer dans ce code :

1. Chaque test correspond à une fonction dans la classe
1. On a accès à (au moins) deux fonctions, `verifieVrai(...)` et `verifieEgal(..., ...)` qui nous permettent de dire si le test passe ou échoue.
1. Nous n'utilisons pas (pour le moment?) d'annotation `@Test` comme en JUnit. Nous exécuterons simplement la classe comme un paramètre à envoyer au framework plutôt que de faire le tout via un _Annotation Processor_.

Créer les fonctions de validation
---------------------------------

Pour créer nos fonctions de validation génériques, nous devons donc en faire des fonctions statiques, qui seront importables dans la classe qui contient les tests.

Pour la vérification d'une condition booléenne :

<pre><code class="Java">public static void verifieVrai(boolean condition) {
    if(condition) {
        reussite();
    } else {
        echec();
    }
}
</code></pre>

Pour la vérification d'une égalité :

<pre><code class="Java">public static void verifieEgal(int attendu, int resultat) {
    if(attendu == resultat) {
        reussite();
    } else {
        fail(nomMethode + " : Attendu " + attendu + " / Obtenu " + resultat);
    }
}
</code></pre>

Pour le moment, nous ferons simplement écrire `.` lorsque le test passera et `F` lorsqu'il ne passera pas.

Une fois que cela est fait, on a donc la fonctionnalité de vérification à savoir si un test passe ou pas.

Exécution de tous les tests
---------------------------
Pour le moment, si l'on souhaite avoir le résultat d'un test, il faut appeler directement la fonction qui contient le test par son nom. Ce que l'on vise est d'exécuter tous les tests d'une classe, peu importe leur nombre et leur nom.

Pour ce faire, il faudra utiliser la réflexivité. Il s'agit d'aller inspecter la structure d'une autre classe à l'intérieur d'un programme.

Dans ce cas-ci, ce qui nous intéresse est d'aller chercher toutes les méthodes qui sont membres de la classe. On peut donc le faire de la façon suivante :

<pre><code class="Java">Class c = Class.forName("ClasseDeTests.java");
Method[] m = c.getDeclaredMethods();
</code></pre>

Nous avons donc maintenant accès à un tableau contenant toutes les méthodes qui sont définies dans cette classe.

Puisque l'on veut exécuter toutes les fonctions, il suffit de boucler sur le tableau, après avoir défini une instance de la classe, et exécuter chacune des méthodes de l'instance `testsClass` en appelant la méthode `invoke(...)` :

<pre><code class="Java">Constructor constructor = c.getConstructor();
Object testsClass = constructor.newInstance();

for (Method test : m) {
    test.invoke(testsClass);
}
</code></pre>

Ne pas planter si un test plante
--------------------------------
Actuellement, avec ce que nous avons fait, si jamais un test lève une exception, c'est toute l'application qui plante. Ce que l'on souhaiterait, c'est que l'erreur soit attrapée, et qu'elle soit probablement aussi affichée. Par contre, il faudrait que les tests continuent à s'exécuter.

Nous allons donc entourer l'appel à `invoke` d'un `try ... catch` de cette façon :
<pre><code class="Java">for (Method test : m) {
    try {
        test.invoke(testsClass);
    } catch (Exception e) {
        fail(test.getName() + " : \n" + e.getCause().getMessage() + "\n" + printArray(e.getCause().getStackTrace()));
    }
}</code></pre>

Ainsi, on conserve le test qui échoue, la cause et la _stack trace_ de l'échec.

Messages d'erreurs et nombre de tests
-------------------------------------
Lorsqu'une exception est levée, il est facile d'écrire le nom du test qui échoue, puisqu'on est justement en train de boucler dessus. Par contre, lorsque ce n'est que la condition finale qui échoue, il peut être un peu plus ardu de détecter exactement quel test est en échec (parce qu'ultimement, ce n'est qu'une fois l'appel à la méthode de vérification fait que l'on sait si on a besoin du nom du test ou pas.)

Ainsi, on modifiera donc la fonction `verifieVrai` de cette façon, pour qu'elle aie chercher dans la pile d'appels de quel test on arrive :

<pre><code class="Java">public static void verifieVrai(boolean condition) {
    if(condition) {
        reussite();
    } else {
        String nomMethode = Thread.currentThread().getStackTrace()[2].getMethodName();

        echec(nomMethode + " : Condition non vérifiée");
    }
}
</code></pre>

Conclusion et exécution
-----------------------
Si on reprend donc notre exemple initial, on obtient donc le résultat suivant à l'exécution :
<pre><code>F.F.F
-----
3 test(s) en erreur sur 5 tests exécutés

divisionParZeroTest :
/ by zero
com.olivierlafleur.testtest.ClasseDeTests.divisionParZeroTest(ClasseDeTests.java:12)
sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:497)
com.olivierlafleur.testtest.MonFrameworkDeTest.main(MonFrameworkDeTest.java:21)
sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:497)
com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)

unAutreTest2 : Condition non vérifiée
unTestInEgalite : Attendu 1 / Obtenu 2

Process finished with exit code 0
</code></pre>

Bien entendu, on est loin d'un framework de test complet et utilisable en production, mais j'espère que cela aura pu vous donner une meilleure idée de comment les _framework_ de tests peuvent être bâtis, sans que cela ne soit de la magie noire. :)

Une amélioration potentielle de cette librairie serait de faire l'exécution des tests non pas avec une classe `Main` mais avec un processeur d'annotations, comme en JUnit (qui utilise l'annotation `@Test` pour dire qu'il s'agit d'un test à exécuter).

Par ailleurs, vous pourrez [trouver sur GitHub l'exemple complet](https://github.com/olafleur/my-own-test-framework)  dans lequel la classe qui contient les tests est reçue en argument à l'exécution (rendant ainsi le logiciel beaucoup plus générique).

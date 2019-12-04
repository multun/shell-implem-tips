Lexeur
======

Le gros challenge de cette partie est de délimiter les bornes des mots. Encore une fois, la
version simple mais incomplète est beaucoup plus accessible et rentable en temps.

C'est probablement la partie la plus dure à faire parfaitement.

 - Faites des tests
 - Beaucoup de tests
 - Testez ce que vous ne savez pas

Méthode simple
--------------

Voici un algorithme de lexing minimal :

1) On saute les espaces
2) Si le caractère actuel commence un opérateur, on lexe et retourne un nouveau token
3) Sinon, on fait partie d'un mot. On continue à lexer tant que le caractère n'est pas une opérateur / un espace.

Cet algorithme ne gère pas les quotes. Il est fortement conseillé d'utiliser une approche récursive pour gérer celles-ci.

Méthode propre
--------------

Le lexeur devrait être réutilisable pour la partie expansion. Sinon, vous pourriez interprêter différement deux
chaines, et échouer avec perte et fracas.

Au premier abord, la tache est difficile: la tache du lexeur est de délimiter les strings, et celle de l'expansion
de les interprêter. Il s'agit donc de leur construire une base commune

Vous avez envie de le construire récursivement, de telle sorte que les différents contextes (``', ", $(), \```)
soient gérés sans cas particulier. Idéalement, le code doit être réutilisable pour la partie expansion (quand
vous allez réutiliser les strings. De plus, (si vous voulez les gérer, c'est à dire certainement pas), les
HEREDOC ne devraient pas être un type de token différent du point de vue du parseur, mais plutôt une sorte de
promesse future de mot.

Une solution intéressante peut être de construire deux lexeurs, le premier servant de base commune au second et
à l'expansion.


Les alias
---------

Les alias sont gérés lors du lexing. L'environnement d'exécution contient un tableau associant le nom de l'alias à la liste de tokens qui devront le remplacer.

Considérez l'exemple suivant:

.. code-block:: shell

   alias oops='mafonction ('
   oops) {
     echo hm
   }

Le shell fait comme suit:

1) il parse la première commande comme le mot alias suivit d'un unique argument: ``oops=mafonction (``.
2) il exécute la première commande, qui rajoute enregistre ``oops`` dans la table des alias. La valeur de l'alias est lexée pour pouvoir être plus facilement substituée par la suite.
3) il commence à parser la seconde ligne. Comme le token oops commence une commande, le shell regarde si il existe un alias à ce nom. comme c'est le cas, il remplace le token par la liste de tokens contenue dans l'alias.
4) ``mafonction`` commence une commande. Il n'y a pas d'alias à ce nom, le parsing continue. Une fonction est reconnue.

Quelques exemples de comportements importants:

.. code-block:: shell

   # les alias travaillent au niveau des tokens
   alias atroce='func('
   # la ligne suivante donne une erreur de syntaxe,
   # car func(\n n'est pas du shell valide.

   atroce  # ERROR: syntax error near unexpected token `newline'

   # ceci est une déclaration de fonction valide.
   # les tokens sont substitués car en début de ligne
   atroce ) {
       echo vraiment atroce
   }

   if true; then
       alias machin='echo hm ok'

       # la commande suivante n'est pas trouvée car
       # l'alias n'était pas encore là quand le mot
       # a été lexé.
       machin  # ERROR: machin: command not found
   fi

   # l'alias déclaré au dessus est maintenant actif
   machin  # "hm ok"

   # ATTENTION: cet alias s'appelle lui-même.
   # Cela doit être détecté lors de l'exécution.
   alias echo='echo bidule'
   alias machin='echo truc'

   machin  # "bidule truc"

   # les alias ne sont substitués qu'en début de ligne
   printf '%s\n' echo  # "echo"

   # la recherche d'alias se fait forcément avant l'expansion
   # (on est au lexing). "\echo" n'est pas littéralement un nom
   # d'alias, et ne le deviendra qu'après expansion.
   \echo vrai echo  # "vrai echo"

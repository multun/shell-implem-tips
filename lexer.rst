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

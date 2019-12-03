Exécution
=========

Maintenant que vous avez un AST, il faut en faire quelque chose.
Il vous faut parcourir récursivement votre AST, en effectuant les actions associées pour chaque nœud.

 - L'exécution d'un nœeud retourne un entier, qui représente le succès de l'opération s'il vaut zéro,
   un échec autrement. Vu que vous n'exécuterez toujours qu'un nœuds à la fois, vous pouvez en faire
   une valeur globale.

 - Certaines de ces instructions, comme ``mavariable=valeur`` modifient des variables shell. Il vous faut
   alors modifier la table de hash associant le nom de la variable à sa valeur en conséquence.

 - Certaines chaines de caractère, au contraire, vont accéder à ces variables durant le processus
   d'expansion, détaillé plus tard.

L'expansion
-----------

L'expansion, c'est le processus qui permet de passer d'une chaine de caractère non traitée, comme
``"$mavariable"`` à sa version finale, cad ``valeurdemavariable``.

Ce processus implique entre autres :

  - le traitement particulier des différents types d'échapement (``'"``)
  - le parsing des subshells (``$()``)
  - le parsing de l'expansion arithmétique (``$(())``)
  - l'expansion de variables (``$mavariable``)
  - la découpe du résultat avec l'IFS, dans les portions du mot où c'est applicable.
    par exemple: ``a='un test'; printf 'mot: %s\n' 'ceci est'$a``
    mais ``a='un test'; printf 'mot: %s\n' 'ceci est'"$a"``

Il s'agit en fait de parcourir les mots comme lors du lexing, mais en les traduisant cette fois-ci
en opérations concrètes.

Deux solutions possibles:

 - Vous devez faire la même chose que le lexer, mais celui-ci n'est pas assez modulaire ? Copié collé, modifié, et hop !
 - Si vous avez suivi la méthode propre du lexer, vous devriez avoir une base commune permettant d'implémenter l'expansion sans trop de diffiicultés.

Les command substitutions
~~~~~~~~~~~~~~~~~~~~~~~~~

Les substitutions de commande se présentent sous deux formes:

 - :literal:`echo \`subshell\``
 - :literal:`echo $(subshell)`

Analyse
#######

L'exécution de la première forme ne requiert pas de précaution particulière. Lorsque le premier backtick est rencontré, on cherche simplement celui de fin.
La deuxième forme est plus complexe à délimiter. Il faut relancer un lexeur à partir du ``$(`` ouvrant, lancer le subshell avec comme donnée le résultat du lexing, et recommencer l'expansion là où il s'est terminé.

Exécution
#########

On crée un pipe, on fork, on attache la sortie standard du fils dans l'entrée du pipe et on lit dans la sortie avec le parent.
Le fils parse, lexe et exécute ensuite le contenu du subshell, dans un boucle. Il est important d'exit et de ne pas continuer d'exécuter l'ast dans le fils une fois terminé.
Le parent lit les données du fils et les rajoute dans le résultat d'expansion. Une fois la lecture terminée, il wait le process et continue l'expansion.

L'expansion arithmétique
~~~~~~~~~~~~~~~~~~~~~~~~

L'expansion arithmétique du shell est définie comme suivant la spécification C. Il faut construire un mini lexeur et un mini parseur pour évaluer ces sections.
Attention, il n'est pas nécessaire de faire un AST! tout peut être évalué au fur et à mesure si on utilise le bon type de parseur.

`Les parseurs de pratt s'y prêtent particulièrement bien <https://github.com/multun/python-pratt-parser/>`_.

Les commandes
-------------

Les commandes sont composées de trois éléments:

- des assignations
- des arguments
- des redirections

Lors du parsing, ces trois types d'éléments sont ajoutés dans trois tableaux.
Lors de l'exécution, ces éléments sont traités dans l'ordre suivant:

- les redirections sont effectuées de gauche à droite
- les arguments de la commande sont expand (``echo $var`` est transformé en ``echo varcontent``)
- les assignations sont effectuées (après le fork si il existe, tant pis sinon)
- la commande est exécutée
- les redirections sont annulées

Les redirections
----------------

`Conférences Redirection et Pipe 42SH - 2019 <https://www.youtube.com/watch?v=ceNaZzEoUhk>`_

.. code-block:: none

                      terminal

                      +   ^   ^
  +--------+---+      |   |   |
  |        | 0 +<-----+   |   |
  |        +---+          |   |
  |        | 1 +----------+   |
  |   sh   +---+              |
  |        | 2 +--------------+
  |        +---+
  |        | … |
  +--------+---+


Le schéma ci-dessus représente un scénario typique d'exécution d'un shell. Celui-ci lit ses commandes depuis stdin (file descriptor 0), écrit normalement sur stdout (file descriptor 1), et sur stderr (file descriptor 2) en cas d'erreur.

Chaque process dispose d'un tableau de pointeurs vers des ressources. Ce tableau n'étant pas directement accessible, on utilise à la place l'index dans ce tableau. Cet index est appelé file descriptor.

Après un ``open("monfichier")``, ce tableau aura été transformé comme suit:

.. code-block:: none

                      terminal

                      +   ^   ^
  +--------+---+      |   |   |
  |        | 0 +<-----+   |   |
  |        +---+          |   |
  |        | 1 +----------+   |
  |   sh   +---+              |
  |        | 2 +--------------+
  |        +---+                    +------------+
  |        | 3 +------------------->+ monfichier |
  +--------+---+                    +------------+

On peut effectuer différentes opérations sur ce tableau:

 - ``close(fd)`` supprime un lien
 - ``dup(fd)`` fait une copie du lien
 - ``dup2(oldfd, newfd)`` fait une copie de oldfd, et la met à l'index de newfd. si la case est déjà prise, l'ancien fd est close.

Après un ``dup2(3, 2)``, le nouvel état sera:

.. code-block:: none

                      terminal

                      +   ^
  +--------+---+      |   |
  |        | 0 +<-----+   |
  |        +---+          |
  |        | 1 +----------+
  |   sh   +---+
  |        | 2 +------------+
  |        +---+        +---v--------+
  |        | 3 +------->+ monfichier |
  +--------+---+        +------------+

Après un ``close(3)`` le nouvel état sera:

.. code-block:: none

                      terminal

                      +   ^
  +--------+---+      |   |
  |        | 0 +<-----+   |
  |        +---+          |
  |        | 1 +----------+
  |   sh   +---+
  |        | 2 +------------+
  |        +---+        +---v--------+
  |        | 3 |        | monfichier |
  +--------+---+        +------------+

Ce qui est plus ou moins équivent à l'état nécessaire à un ``sh 2>monfichier``.

Attention toutefois ! les redirections doivent pouvoir être annulées. Il faut dupliquer le file descriptor qui va être écrasé par le dup2 pour pouvoir ensuite le restaurer. Sinon, les redirections persisteront pendant le reste de l'exécution du shell. On peut être tenté d'exécuter les redirections après avoir fork pour exécuter une commande, mais cela ne fonctionnera pas lorsqu'on ne fork pas (lors de l'exécution des fonctions et des builtins).

Les variables locales à une commande
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Lorsque vous faites ``a=b macommande``, la variable a ne vaudra b que pendant l'exécution de ``macommande``. Attention toutefois, si macommande est une fonction, cette règle ne s'applique pas, et il n'est pas nécessaire de restaurer la valeur de a.

Les fonctions
-------------

Le support des fonction se décompose en deux partie. Leur définition, et leur appel.

Définition
~~~~~~~~~~
Lors de l'exécution, il faut soit recopier le bout d'ast du corps de la fonction dans un tableau associant le nom de la fonction à son code.

Il y a plusieurs moyen de faire en sorte que le bout d'AST de la fonction ne soit pas free avec le reste:

- le retirer de l'AST
- le marquer comme ne devant pas être free d'une manière ou d'une autre
- faire du reference counting
- garder les AST de toutes les commandes en mémoire (plutôt bof)

La première méthode est sûrement la plus simple.

Appel
~~~~~
Les fonctions sont gérées légèrement différemment du reste des commandes simples, sans pour autant diverger énorment :
la fonction en charge de lancer une commande vérifie d'abord si une fonction du nom demandé existe, et l'appelle si c'est le cas.
dans le cas contraire, le processus continue comme à son habitude.

Les redirections sont effectuées comme d'habitude, et si les assignations doivent être visibles à l'intérieur de la fonction,
elles peuvent aussi l'être une fois l'appel terminé (au choix).

Les pipes
---------

`Conférences Redirection et Pipe 42SH - 2019 <https://www.youtube.com/watch?v=ceNaZzEoUhk>`_

**Errata**:

- j'ai par erreur interverti les deux extrémités du pipe lors de la conférence (``fd[0]`` et ``fd[1]``)
- il est préférable de marquer les sauvegardes de file descriptor avec ``CLOEXEC`` pour éviter que les process enfants en héritent

.. admonition:: Allez plus loin

  La conférence présente une version volontairement simplifiée de l'algorithme, qui considère que le pipe est un opérateur binaire.
  C'est tout à fait acceptable et fonctionnel, mais devient problématique si vous souhaitez implémenter du job control.

  **Attention, le job control est une fonctionnalité très avancée**

  https://www.linusakesson.net/programming/tty/index.php

  https://blog.nelhage.com/2010/01/a-brief-introduction-to-termios-signaling-and-job-control/


Pipelines
~~~~~~~~~

La manière la plus simple d'exécuter une suite de pipes se fait de la même manière qu'on exécute un opérateur pipe binaire:

.. code-block:: none

   a | b | c

       |
      / \
     |   c
    / \
   a   b

on exécute d'abord le nœud pipe racine, puis les autres, récursivement. Comme n'importe quel autre nœud d'ast.


La boucle read / eval
---------------------

Vous avez besoin de faire une boucle qui va soit lire une string et appeler votre parseur dessus,
pour la méthode sale, soit appeller votre parseur avec un lexeur configuré avec un backend readline,
pour la méthode propre.

C'est aussi pas mal d'avoir un état, histoire de pouvoir se trimballer des fonctions / variables entre
deux lignes / AST.

Faites attention de ne pas free les bouts d'AST utilisés pour les fonctions.


FAQ
---

Mon programme fait N fois la même chose
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Quand l'exécution d'une commande rate (``execvpe`` n'a pas fonctionné) ou qu'un sous-shell se termine,
le processus créé par ``fork`` est toujours là. Il faut afficher un message d'erreur
et ``exit`` avec le bon code de retour.


Les dernières lignes de mon fichier sont lues plusieurs fois
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Si le fichier depuis lequel vous lisez les commandes n'est pas
close avant d'``exit``, la bibliothèque standard va lseek vers le début du fichier
pour prendre en compte le fait que des données on été lues en avance par le buffering.

Étant donné que le process parent partage le descripteur de fichier, le ``lseek``
agira également sur son instance de fichier ouvert. Quand le parent aura vidé son buffer,
il lira à nouveau dans le fichier à partir de l'index corrigé par le fils.

À vous d'utiliser l'exemple suivant pour régler votre problème:

  .. code-block:: c

    #define _GNU_SOURCE

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>

    #include <sys/types.h>
    #include <sys/wait.h>
    #include <fcntl.h>

    static ssize_t next_line(FILE *f)
    {
        char *line = NULL;
        size_t size = 0;
        ssize_t line_size = 0;
        if ((line_size = getline(&line, &size, f)) <= 0)
            printf("no more lines\n");
        else
            printf("> %.*s\n", (int)line_size, line);
        free(line);
        return line_size;
    }

    int main(void)
    {
        FILE *f = fopen("test.txt", "r+");

        next_line(f);
        pid_t pid = fork();
        if (pid == 0) {
            // UNCOMMENT ME: fclose(f);
            return 2;
        }

        int status;
        waitpid(pid, &status, 0);
        printf("child exited with status %d\n", WEXITSTATUS(status));

        while (next_line(f) > 0)
            continue;

        fclose(f);
    }

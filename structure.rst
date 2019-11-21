Structure générale
==================

.. code-block:: none

                                                                     +---> file input
                   +----------+     +---------+     +------------+   |
                   |          |     |         |     |            |   |
       +---------->+  parser  +---->+  lexer  +---->+ IO backend +------> readline input
       |           |          |     |         |     |            |   |
   +---+---+       +----------+     +---------+     +------------+   |
   |       |                                                         +---> string input
   | Read  |
   | Eval  |
   | Loop  |
   |       |
   +---+---+       +-----------+       +-----------+
       |           |           |       |           |
       +---------->+ execution +------>+ expansion |
                   |           |       |           |
                   +-----------+       +-----------+

- Le lexeur découpe votre suite de charactère en mots. Il donne du sens basique à ces mots:

  * points-virgule / retours à la ligne (``; \n``)
  * IO number (pour les redirections) (``1>``)
  * mot ``toto if then``
  * mot d'assignation ``A=B``
  * nom générique ``0abc``
  * opérateurs ``|``

- Le parseur prend ces mots, et analyse leur sens pour les organiser en arbre
- Le composant d'exécution prend cet arbre et un état initial (variables), effectue les
  actions spécifiées par l'arbre et modifie l'état en conséquence.

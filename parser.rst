Parseur
=======

Le plus gros challenge de cette partie est de ne pas :

 - leak
 - violer la coding style

Vous devez prendre des tokens, et les convertir en nœuds d'AST.

Lire une grammaire
------------------

Une grammaire est un arbre de choix qui décrit comment un produire un programme syntaxiquement valide.
Ainsi, le passage de la grammaire à un parseur récursif n'est pas forcément évident.

Quelques pistes:

 - une partie du travail consiste à chercher les cas d'arrêt. par exemple, une simple command shell peut être arrêtée par différents tokens: `; `, `\n` ou `)`. pour trouver les différents cas d'arrêts, on cherche les tokens qui peuvent suivre une règle parmis tous les chemins qui y mènent dans la grammaire.
 - parfois, le token suivant ne suffit pas à décider quelle règle s'applique. par exemple:

.. code-block:: shell

  macommande
  macommande() {
   echo oops
  }

dans l'exemple précédent, lire le premier token de la ligne ne suffit pas à savoir s'il s'agit d'une commande ou d'une définition de fonction.


Exemple de fonction de parsing
------------------------------

Une manière raisonnable d'écrire des fonctions de parsing est la suivante :

.. code-block:: c

  enum token_type
  {
      TOKEN_WHILE,
      TOKEN_DO,
      TOKEN_DONE,
  };

  enum parser_status
  {
      PARSER_DONE = 0,
      PARSER_NO_MATCH,
      PARSER_ERROR,
  };

  struct ast_node_while *ast_node_while_attach(struct ast_node **res)
  {
      // depending on the ast implementation
      struct ast_node_while *node = ast_node_while_alloc();
      *res = &node->base;
      return node;

      // OR

      *res = ast_node_alloc();
      (*res)->type = AST_WHILE;
      return &(*res)->data.while_data;
  }

  enum parser_status parse_rule_while(struct ast_node **result, struct lexer *lexer)
  {
      int rc;
      // is the next token isn't a while token, the rule doesn't match
      if (!parser_consume(lexer, TOKEN_WHILE))
          return PARSER_NO_MATCH;

      // create a new AST node, and attach it to the result pointer
      struct ast_node_while *while_node = ast_node_while_attach(result);

      // parse the condition, and return the error is a failure occurs
      if ((parse_compound_list(&while_node->condition, lexer)))
          return rc;

      // return the error if there's no "do"
      if ((rc = parser_consume(lexer, TOKEN_DO)))
          return rc;

      // parse the while body
      if ((rc = parse_compound_list(&while_node->body, lexer)))
          return rc;

      // return the error if there's no "done"
      if ((rc = parser_consume(lexer, TOKEN_DONE)))
          return rc;
      return 0;
  }


Type de parseur
---------------

Un [parseur à descente récursive](https://en.wikipedia.org/wiki/Recursive_descent_parser) suffit. Vous pouvez faire autre chose si vous voulez, mais ça sera difficilement plus rentable en temps que ce type là.

L'AST
-----

Un AST shell **n'est pas un arbre binaire**. Chaque type de nœud (commande, if, …)
dispose de champs particulier. Il doit y avoir un moyen de déterminer le type d'un nœud
lors d'un parcours de l'arbre.

Méthode 1
~~~~~~~~~
Un nœud d'AST pourrait ressembler à la chose suivante :

.. code-block:: c

  struct ast_node
  {
      enum node_type
      {
          NODE_IF,
          NODE_COMMAND,
          ...
      } type;

      union
      {
          struct node_if node_if;
          struct node_command node_command;
          /* ... */
      } data;
  };

N'hésitez-pas à créer une fonction dédiée à l'allocation de nœuds: vous l'utiliserez tellement que chaque 
ligne compte.

Par exemple:

.. code-block:: c

  struct ast_node *node_if = alloc_node(NODE_IF);
  if (parse_if(lexer, &node_if->data.node_if) == 0)
      return node_if;

  // handle error
  return NULL;

Méthode 2
~~~~~~~~~

On peut implémenter de l'héritage simple avec fonctions virtuelles:

.. code-block:: c

  typedef void (*node_free)(struct ast_node *node);
  typedef void (*node_print)(struct ast_node *node);
  typedef void (*node_exec)(struct ast_node *node, struct sh_context *context);

  struct node_type {
      const char *name;
      node_free free;
      node_print print;
      node_exec exec;
  };

  extern struct node_type node_if;
  extern struct node_type node_command;

  struct ast_node
  {
      struct node_type *type;
      // champs communs à tous les nœuds
  };

  /* ... */

  struct ast_node_if
  {
      struct ast_node base;
      // les champs nécessaires au if
  };

Beaucoup plus propre, il faut une fonction d'allocation par nœud.

Méthode 3
~~~~~~~~~


.. code-block:: c

  enum ast_node_type
  {
      AST_NODE_IF,
  };

  struct ast_node
  {
      struct ast_node_type type;
      // champs communs à tous les nœuds
  };

  /* ... */

  struct ast_node_if
  {
      struct ast_node base;
      // les champs nécessaires au if
  };

Cette dernière méthode est probablement la plus adaptée au problème.

Voici un exemple d'usage de ce type d'ast :

.. code-block:: c

  void ast_free_if(struct ast_node *node)
  {
      struct ast_node_if *if_node = (struct ast_node_if *)node;
      /* ... */
      free(if_node);
  }

  typedef void (*ast_free_f)(struct node *);

  ast_free_f ast_free_func[] =
  {
      [NODE_IF] = ast_free_if,
  }

  void ast_free(struct node *ast_node)
  {
      return ast_free_func[ast_node->type](node);
  }

Simplifier sa gestion d'erreur
------------------------------

Un des éléments les plus problématiques du parseur shell est sa gestion d'erreur:
pratiquement chaque opération peut mener à une erreur qu'il faudrait prendre en charge.

Il faut donc trouver des moyens de rendre plus simple sa gestion d'erreur.

Le free par le haut
~~~~~~~~~~~~~~~~~~~

La méthode la plus souvent utilisée pour allouer des AST implique de retourner le nœud créé par la fonction de parsing.
Ainsi, en cas d'erreur, cette fonction devra se charger de libérer toute la mémoire déjà allouée.

Une méthode moins lourde est de passer en argument l'endroit où rattacher le nœud d'ast à créer. Ainsi, en cas d'erreur,
on peut directement sortir de la fonction sans free, et appeller une seule fois une fonction de free après être sorti du parseur.

Vous pouvez voir un exemple de cette technique dans la fonction de parsing plus haut.

La méthode des free list
~~~~~~~~~~~~~~~~~~~~~~~~

Vos pouvez rajouter une liste d'addresses à free à votre AST. La méthode est peu élégante mais efficace.
Mettre cette liste dans une variable globale / indépendante de l'AST est par contre absurde.

La méthode des exceptions
~~~~~~~~~~~~~~~~~~~~~~~~~

Implémentez des exceptions (setjmp / longjmp) permet de ne pas avoir à vérifier à la main le code de retour des fonctions appelées.
Si une erreur fatale se produit, le parseur s'arrêtera. Il est nécessaire de combiner cette méthode avec celle du free par le haut
pour éviter les leaks.

**/!\\ Attention, cette méthode est très pratique mais difficile à implémenter /!\\**

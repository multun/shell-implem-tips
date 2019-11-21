Intéractions entre les composants
=================================

Interactions lexeur-entrée (IO)
-------------------------------

C'est peut-être la partie la plus déterminante du projet. Vous devez choisir entre
avoir un lexeur qui appelle un backend qui lit le programme, et lire le programme d'abord,
et le donner au lexeur.

Dites-vous que faire cette partie à moitié, c'est diviser sa note par deux.

La méthode sale
~~~~~~~~~~~~~~~

Une solution possible, mais incomplète, consiste à utiliser un lexeur qui ne gère qu'un
type d'entrée: une string, terminée par un 0. Il faudra alors batailler pour convertir
le reste vers ce format :

 - avec la command line, tout vas bien
 - avec un fichier en argument, tout vas bien, vous le lisez en entier dans un gros buffer
 - en lisant sur l'entrée standard, ça se complique.
 - si l'entrée standard n'est pas interactive, on peut lire jusqu'à EOF
 - pour le mode interactif, vous pouvez lire l'entrée ligne par ligne
 - n'espérez pas passer les tests de commande sur plusieurs lignes en mode interactif 
   avec cette méthode

La méthode propre
~~~~~~~~~~~~~~~~~

Le lexer interagit avec un backend d'entrée sortie, similaire aux [FILE streams génériques](
https://www.gnu.org/software/libc/manual/html_node/Streams-and-Cookies.html).
L'idée étant d'avoir un ensemble d'opérations (next, look) identique pour toutes les
entrées possibles: si ces bases marchent, le reste fonctionnera uniformément.

Cela permet de gérer les commandes interactives sur plusieurs lignes, et tout ce que vous voudriez faire d'autre.

il s'agit d'avoir une structure du genre:

.. code-block:: c

  struct cstream
  {
      /* ... */
  };

  /* creates a new stream associated with the given string */
  struct cstream *stream cstream_open_string(char *string);

  /**
  ** \brief looks at the next char in the stream
  **
  ** The function does not modify the character stream.
  **
  ** \code
  ** struct cstream *cs = cstream_open_string("test");
  ** assert(cstream_look(cs) == 't');
  ** assert(cstream_look(cs) == 't');
  ** \endcode
  ** \return the next character in the stream
  */
  char cstream_look(const struct cstream *stream);

  /**
  ** \brief gets the next char in the stream, and moves forward
  **
  ** \code
  ** struct cstream *cs = cstream_open_string("test");
  ** assert(cstream_next(cs) == 't');
  ** assert(cstream_look(cs) == 'e');
  ** assert(cstream_next(cs) == 'e');
  ** assert(cstream_next(cs) == 's');
  ** \endcode
  ** \return the next character in the stream
  */
  char cstream_next(const struct cstream *stream);

Interactions parseur-lexeur
---------------------------

La méthode sale
~~~~~~~~~~~~~~~

Tout lexer d'abord, puis tout donner au parseur. Vous ne pourrez par gérer les alias, et consommerez plus de mémoire.

La méthode propre
~~~~~~~~~~~~~~~~~

Faire en sorte que le parseur demande des tokens au lexeur, qui lexe au fur et à mesure.

# Creando nuevos posts

{video: make-new-post}

Teniendo en cuenta que ya tenemos los untos finales en su lugar, el siguiente paso para dejar a los usuarios crear nuevos posts es una interfaz. Lograremos esto añadiendo un botón en la esquina inferior derecha de la pantalla. Cuando hacemos click en el botón, se ensaña un modal preguntando al usuario escribir en el post.

Por el momento queremos hacer visible este botón en la pagina indice, así que abre `static/templates/layout/index.html` y añade el siguiente código al final del fichero:

    <a class="btn btn-primary btn-fab btn-raised mdi-content-add btn-add-new-post"
        href="javascript:void(0)"
        ng-show="vm.isAuthenticated"
        ng-dialog="/static/templates/posts/new-post.html"
        ng-dialog-controller="NewPostController as vm"></a>

El tag a, en este código, utiliza la directiva `ngDialog` que hemos incluido como dependencia anteriormente, para mostrar el modal cuando el usuario quiere enviar el nuevo post.

Como queremos que el botón este fijo en la esquina inferior derecha de la pantalla, también necesitaremos añadir una nueva regla CSS.

Abre `static/stylesheets/styles.css` y añade la siguiente regla al final del fichero:

    .btn-add-new-post {
      position: fixed;
      bottom: 20px;
      right: 20px;
    }

{x: new_post_modal_button}
Añade el botón para mostrar la ventana de nuevo post

{x: style_new_post_button}
Define el estilo del nuevo boton

## Una interfaz para enviar un nuevo posts
Ahora necesitamos crear el formulario que debe rellenar para crear un nuevo post. Abre `static/templates/posts/new-post.html` y añade lo siguiente al final del fichero:

    <form role="form" ng-submit="vm.submit()">
      <div class="form-group">
        <label for="post__content">New Post</label>
        <textarea class="form-control" 
                  id="post__content" 
                  rows="3" 
                  placeholder="ex. This is my first time posting on Not Google Plus!" 
                  ng-model="vm.content">
        </textarea>
      </div>

      <div class="form-group">
        <button type="submit" class="btn btn-primary">
          Submit
        </button>
      </div>
    </form>

{x: new_post_template}
Crea una plantilla para añadir un nuevo post en `static/templates/posts/new-post.html`

## Controlando la nueva interfaz de post con NewPostController
Crea `static/javascripts/posts/controller/new-post.controller.js` con el siguiente contenido:

    /**
    * NewPostController
    * @namespace thinkster.posts.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.posts.controllers')
        .controller('NewPostController', NewPostController);

      NewPostController.$inject = ['$rootScope', '$scope', 'Authentication', 'Snackbar', 'Posts'];

      /**
      * @namespace NewPostController
      */
      function NewPostController($rootScope, $scope, Authentication, Snackbar, Posts) {
        var vm = this;

        vm.submit = submit;

        /**
        * @name submit
        * @desc Create a new Post
        * @memberOf thinkster.posts.controllers.NewPostController
        */
        function submit() {
          $rootScope.$broadcast('post.created', {
            content: vm.content,
            author: {
              username: Authentication.getAuthenticatedAccount().username
            }
          });

          $scope.closeThisDialog();

          Posts.create(vm.content).then(createPostSuccessFn, createPostErrorFn);


          /**
          * @name createPostSuccessFn
          * @desc Show snackbar with success message
          */
          function createPostSuccessFn(data, status, headers, config) {
            Snackbar.show('Success! Post created.');
          }

          
          /**
          * @name createPostErrorFn
          * @desc Propogate error event and show snackbar with error message
          */
          function createPostErrorFn(data, status, headers, config) {
            $rootScope.$broadcast('post.created.error');
            Snackbar.error(data.error);
          }
        }
      }
    })();

{x: new_post_controller}
Crea `NewPostController` en `static/javascripts/posts/controllers/new-post.controller.js`

Hay algunas cosas de las que deberíamos hablar un poco.

    $rootScope.$broadcast('post.created', {
        content: vm.content,
        author: {
            username: Authentication.getAuthenticatedAccount().username
        }
    });

Antes hemos configurado un listaren en `IndexController` que escucha al evento `post.created` y entonces envía el nuevo post al principio de `vm.posts`. Veamos esto con mas detalle, ya que esto resulta ser una característica importante de las aplicaciones web.

Lo que estamos haciendo aquí es siendo *optimistas* y asumiendo que la respuesta de la API de `Posts.reate()` contendrá un código de estado 200 diciéndonos que todo esta conforme a lo previsto. Esto, puede parecer una mala idea al principio. Algo puede fallar durante la petición y nuestros datos son erróneos. ‚Por que no esperar a una respuesta entonces?

Cuando digimos que estábamos incrementando la *percepción* de potencia de nuestra aplicación, era de esto a lo que nos referiamos. Queremos que el usuario perciba la respuesta como inmediata.

El hecho es que esta llamada raravez falla. Solo hay dos casos en los que esta fallara de forma razonable: o el usuario no esta autenticado o el servidor esta apagado.

En el caso de que el usuario no esta autenticado, no debería poder enviar un post nuevo de ninguna forma. Considera el error como un pequeño castigo para el usuario que hace cosas que no debería.

Si el servidor esta caído, no hay nada que podamos hacer. Salvo en el caso en el que el usuario tuviera esta pagina cargada antes que el servidor falle, no deberían ser capaces de ver esta pagina de todos modos.

Otras cosas que podrían salir mal conformarán un porcentaje tan pequeño que estamos dispuestos a permitir una experiencia un poco peor para hacer la experiencia mejor para el 99,9% de los casos en el que todo funciona correctamente.

Por otra parte, el objeto que pasamos como segundo argumento se entiende que debe emular la respuesta del servidor. Esto no es el mejor patrón de diseño ya que asume que conocemos como será la respuesta. Si la respuesta cambia, debemos actualizar el código. De todas formas, dado lo que tenemos, es un coste aceptable.

Así que, ¿que ocurre cuando la llamada a la API devuelve un error?

    $rootScope.$broadcast('post.created.error');

Si se dispara la llamada al error, entonces transmitiremos un nuevo evento: `post.created.error`. El oyente del evento  que hemos configurado antes, será lanzado por este evento y eliminar el post al principio de `vm.posts`. También enseñaremos el mensaje de error al usuario para que sepa lo que ha ocurrido.

    $scope.closeThisDialog();

Este es un metodo proporcionado por `ngDialog`. Todo lo que hace es cerrar el modelo que hemos abierto. También vale la pena señalar que `closeThisDialog()` no esta almacenado en el ViewModel, así que debemos llamar a `$scope.closeThisDialog()` en lugar de llamar a `vm.closeThisDialog()`.

Asegurate de incluir `new-post.controller.js` en `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/posts/controllers/new-post.controller.js' %}"></script>

{x: js_include_new_post}
Incluye `new-post.controller.js` en `javascripts.html`

## Punto de control
Visita `http://localhost:8000/` y haz click en el boton + en la esquina inferior derecha. Rellena el formulario para crear un nuevo post. Sabrás que todo ha funcionado por que se mostrara un nuevo post en la parte superior de la pagina.

{x: checkpoint_new_post}
Crea un nuevo objeto `Post` con la interfaz que acabas de crear.


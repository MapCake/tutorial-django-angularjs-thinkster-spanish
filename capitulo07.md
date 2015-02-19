# Renderizando objetos Post
Hasta ahora la pagina indice estaba vacía. Ahora que ya manejamos la autenticación y los detalles del backend para el modelo `Post`, es hora de dar a nuestros usuarios algo con lo que interactuar. Haremos esto creando un servicio que gestiona la recuperación y la creación de Posts y algunos controladores y directivas para gestionar la forma de visualizar los datos.

{video: render-post-object}

## Un modulo para posts
Vamos a definir los modulos posts.

Crea un fichero en `static/javascripts/posts` llamado `posts.module.js` y añade lo siguiente:

    (function () {
      'use strict';

      angular
        .module('thinkster.posts', [
          'thinkster.posts.controllers',
          'thinkster.posts.directives',
          'thinkster.posts.services'
        ]);

      angular
        .module('thinkster.posts.controllers', []);

      angular
        .module('thinkster.posts.directives', ['ngDialog']);

      angular
        .module('thinkster.posts.services', []);
    })();

{x: posts_module}
Define el modulo `thinkster.posts`

Recuerda añadir `thinkster.posts` como dependencia de `thinkster` en `thinkster.js`:

    angular
      .module('thinkster', [
          'thinkster.config',
          'thinkster.routes',
          'thinkster.authentication',
          'thinkster.layout',
          'thinkster.posts'
      ]);

{x: posts_module_dep_thinkster}
Añade `thinkster.posts` como dependencia del modulo `thinkster`

Hay dos cosas que vale la pena destacar de este módulo.

Primera, hemos creado un modulo llamado `thinkster.posts.directives`. Como podrás imaginar, esto significa que vamos a introducir el concepto de directivas en nuestra aplicacion en este capitulo.

Segunda, el modulo `thinkster.posts.directives` requiere el modulo `ngDialog`. `ngDialog` esta incluido en la plantilla del proyecto y gestiona las pantallas de modales(?). Utilizaremos un modal en el siguiente capitulo cuando escribamos el código para crear nuevos posts.

Incluye este archivo en `javascript.html`:

  <script type="text/javascript" src="{% static 'javascripts/posts/posts.module.js' %}"></script>

{x: posts_include_module}
Incluye `posts.module.js` en `javascript.html`

## Creando el servicio Posts
Antes de poder renderizar nada, necesitamos transportar los datos desde el servidor al cliente.

Crea un fichero en `static/javascripts/posts/services/` llamado `posts.service.js` y añade lo siguiente:

    /**
    * Posts
    * @namespace thinkster.posts.services
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.posts.services')
        .factory('Posts', Posts);

      Posts.$inject = ['$http'];

      /**
      * @namespace Posts
      * @returns {Factory}
      */
      function Posts($http) {
        var Posts = {
          all: all,
          create: create,
          get: get
        };

        return Posts;

        ////////////////////
        
        /**
        * @name all
        * @desc Get all Posts
        * @returns {Promise}
        * @memberOf thinkster.posts.services.Posts
        */
        function all() {
          return $http.get('/api/v1/posts/');
        }


        /**
        * @name create
        * @desc Create a new Post
        * @param {string} content The content of the new Post
        * @returns {Promise}
        * @memberOf thinkster.posts.services.Posts
        */
        function create(content) {
          return $http.post('/api/v1/posts/', {
            content: content
          });
        }

        /**
         * @name get
         * @desc Get the Posts of a given user
         * @param {string} username The username to get Posts for
         * @returns {Promise}
         * @memberOf thinkster.posts.services.Posts
         */
        function get(username) {
          return $http.get('/api/v1/accounts/' + username + '/posts/');
        }
      }
    })();

{x: posts_service}
Crea una nueva factoría llamada `Posts` en `static/javascripts/posts/services/posts.service.js`

Añade este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/posts/services/posts.service.js' %}"></script>

{x: posts_service_include_javascripts}
Añade `posts.service.js` fichero en `javascript.html`

Este código te parecera bastante familiar. Es muy parecido al de los servicios que hemos creado antes.

El servicio `Posts` solo tiene dos métodos: `all` y `create`.

En la pagina indice, utilizaremos `Posts.all()` para obtener la lista de objetos que queremos mostrar. Utilizaremos `Posts.create()` para añadir sus propios posts.

## Creando la interfaz para la pagina indice
Crea `static/templates/layout/index.html` con el siguiente contenido:

    <posts posts="vm.posts" ng-show="vm.posts && vm.posts.length"></posts>

{x: index_template}
Crea la plantilla indice

Añadiremos un poco mas después, pero no mucho. La mayoría de lo que necesitamos esta en la plantilla que crearemos para la directiva posts que crearemos después.

## Creando un servicio Snackbar
En el proyecto plantilla de este tutorial, hemos incluido SnackbarJS. SnackbarJS es una pequeña librería Javascript que facilita mostrar una snackbar (concepto creado por el Material de Diseño de Google). Aquí vamos a crear una servicio para incluir esta funcionalidad en nuestra aplicación AngularJS.

Abre `static/javascripts/utils/services/snackbar.service.js` y añade lo siguiente:

    /**
    * Snackbar
    * @namespace thinkster.utils.services
    */
    (function ($, _) {
      'use strict';

      angular
        .module('thinkster.utils.services')
        .factory('Snackbar', Snackbar);

      /**
      * @namespace Snackbar
      */
      function Snackbar() {
        /**
        * @name Snackbar
        * @desc The factory to be returned
        */
        var Snackbar = {
          error: error,
          show: show
        };

        return Snackbar;

        ////////////////////
        
        /**
        * @name _snackbar
        * @desc Display a snackbar
        * @param {string} content The content of the snackbar
        * @param {Object} options Options for displaying the snackbar
        */
        function _snackbar(content, options) {
          options = _.extend({ timeout: 3000 }, options);
          options.content = content;

          $.snackbar(options);
        }


        /**
        * @name error
        * @desc Display an error snackbar
        * @param {string} content The content of the snackbar
        * @param {Object} options Options for displaying the snackbar
        * @memberOf thinkster.utils.services.Snackbar
        */
        function error(content, options) {
          _snackbar('Error: ' + content, options);
        }


        /**
        * @name show
        * @desc Display a standard snackbar
        * @param {string} content The content of the snackbar
        * @param {Object} options Options for displaying the snackbar
        * @memberOf thinkster.utils.services.Snackbar
        */
        function show(content, options) {
          _snackbar(content, options);
        }
      }
    })($, _);

{x: angularjs_snackbar_service}
Crea el servicio `Snackbar`

No olvides activar tus modulos. Abre `static/javascripts/utils/utils.module.js` y añade lo siguiente:

    (function () {
      'use strict';

      angular
        .module('thinkster.utils', [
          'thinkster.utils.services'
        ]);

      angular
        .module('thinkster.utils.services', []);
    })();

Y haz de `thinkster.utils` una dependencia de `thinkster` en `static/javascripts/thinkster.js`:

    angular
        .module('thinkster', [
            // ...
            'thinkster.utils',
            // ...
        ]);

{x: angularjs_utils_module}
Añade los módulos al módulo `thinkster.utils`

{x: angularjs_include_utils}
Haz de `thinkster.utils` una dependencia de `thinkster`

El ultimo paso para este servicio es incluir los nuevos ficheros JavaScript en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/utils/utils.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/utils/services/snackbar.service.js' %}"></script>

{x: include_utils_js}
Añade `utils.modules.js` y `snackbar.service.js` a `javascripts.html`

## Controlando la interfaz del indice con IndexController
Crea un fichero en `static/javascripts/layout/controllers/` llamado `index.controller.js` y añade lo siguiente:

    /**
    * IndexController
    * @namespace thinkster.layout.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.layout.controllers')
        .controller('IndexController', IndexController);

      IndexController.$inject = ['$scope', 'Authentication', 'Posts', 'Snackbar'];

      /**
      * @namespace IndexController
      */
      function IndexController($scope, Authentication, Posts, Snackbar) {
        var vm = this;

        vm.isAuthenticated = Authentication.isAuthenticated();
        vm.posts = [];

        activate();

        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.layout.controllers.IndexController
        */
        function activate() {
          Posts.all().then(postsSuccessFn, postsErrorFn);

          $scope.$on('post.created', function (event, post) {
            vm.posts.unshift(post);
          });

          $scope.$on('post.created.error', function () {
            vm.posts.shift();
          });


          /**
          * @name postsSuccessFn
          * @desc Update posts array on view
          */
          function postsSuccessFn(data, status, headers, config) {
            vm.posts = data.data;
          }


          /**
          * @name postsErrorFn
          * @desc Show snackbar with error
          */
          function postsErrorFn(data, status, headers, config) {
            Snackbar.error(data.error);
          }
        }
      }
    })();

{x: index_controller}
Crea un nuevo controlador llamado `IndexController` en `static/javascripts/layout/controllers/index.controller.js`

Incluye el fichero en `javascript.html`:

    <script type="text/javascript" src="{% static ‘javascripts/layout/controllers/index.controller.js' %}"></script>

{x: index_controller_include_javascripts}
Incluye `index.controller.js` en `javascripts.html`

Veamos un par de cosas aquí.

    $scope.$on('post.created', function (event, post) {
        vm.posts.unshift(post);
    });

Mas tarde, cuando estemos trabajando en la creación de un nuevo post, dispararemos un nuevo evento llamado `post.created` cuando el usuario cree un nuevo post. Al detectar este evento aquí, podemos añadir este nuevo post al frente del array `vm.posts`. Esto nos evita tener que hacer una petición extra a la API del servidor para actualizar los datos. Hablaremos de esto en breve, pero por el momento debes saber que hacemos esto para aumentar la *percepción* de rendimiento de nuestra aplicación.

    $scope.$on('post.created.error', function () {
        vm.posts.shift();
    });

De forma analoga al listener de eventos anterior, este eliminara el post del principio de `vm.posts` si la petición a la API devuelve un código de estado de error.

## Creando una ruta para la pagina indice
Con el controlodar y la plantilla creados, necesitamos configurar una ruta para la pagina indice.

Abre `static/javascripts/thinkster.route.js` y añade la siguiente ruta:

    .when('/', {
        controller: 'IndexController',
        controllerAs: 'vm',
        templateUrl: '/static/templates/layout/index.html'
    })

{x: index_route}
Añade la ruta a `thinkster.routes.js` para el camino /

## Creando una directiva para mostrar Posts
Crea `static/javascripts/posts/directives/posts.directives.js` con el siguiente contenido:

    /**
    * Posts
    * @namespace thinkster.posts.directives
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.posts.directives')
        .directive('posts', posts);

      /**
      * @namespace Posts
      */
      function posts() {
        /**
        * @name directive
        * @desc The directive to be returned
        * @memberOf thinkster.posts.directives.Posts
        */
        var directive = {
          controller: 'PostsController',
          controllerAs: 'vm',
          restrict: 'E',
          scope: {
            posts: '='
          },
          templateUrl: '/static/templates/posts/posts.html'
        };

        return directive;
      }
    })();

{x: posts_directive}
Crea una nueva directiva llamada `posts` en `static/javascripts/posts/directives/posts.directive.js`

Incluye este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/posts/directives/post.directive.js' %}"></script>

{x: posts_directive_include_js}
Incluye `post.directive.js` en `javascripts.html`

Hay dos partes de las directivas de la API que vamos a comentar: `scope` y `restrict`.

    scope: {
        posts: '='
    },

`scope` define el limite de esta directiva, trabaja de forma similar a como lo hace `$scope` en los controladores.La diferencia es que, en un controlador, se crea de forma implícita un nuevo limite. Para una directiva, tenemos la opción de definirlo explicitamente definiendo nuestros limites, y esto es lo que hemos hecho aquí.

La segunda linea, `posts: '='` simplemente significa que queremos darle el valor `$scope.posts` al valor pasado a través del atributo `posts` en la plantilla que hemos hecho antes.

    restrict: 'E',

`restrict` le dice a Angular como se nos permite utilizar esta directiva. En nuestro caso, seleccionamos el valor `restrict` como `E` (para elemento) lo que significa que Angular solamente debería emparejar el nombre de nuestra directiva con el nombre de un elemento:`<posts></posts>`.

Otra opción común es `A` (para atributo), lo que sugiere a Angular que solo emparejara el nombre de una directiva con el nombre de un atributo. `ngDialog` utiliza esta opción, como veremos en breve.

## Controlar la directiva posts con PostsController
La directiva que acabamos de crear necesita un controlador llamado `PostsController`.

Crea `static/javascripts/posts/controllers/posts.controller.js` con el siguiente contenido:

    /**
    * PostsController
    * @namespace thinkster.posts.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.posts.controllers')
        .controller('PostsController', PostsController);

      PostsController.$inject = ['$scope'];

      /**
      * @namespace PostsController
      */
      function PostsController($scope) {
        var vm = this;

        vm.columns = [];

        activate();


        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.posts.controllers.PostsController
        */
        function activate() {
          $scope.$watchCollection(function () { return $scope.posts; }, render);
          $scope.$watch(function () { return $(window).width(); }, render);
        }
        

        /**
        * @name calculateNumberOfColumns
        * @desc Calculate number of columns based on screen width
        * @returns {Number} The number of columns containing Posts
        * @memberOf thinkster.posts.controllers.PostsControllers
        */
        function calculateNumberOfColumns() {
          var width = $(window).width();

          if (width >= 1200) {
            return 4;
          } else if (width >= 992) {
            return 3;
          } else if (width >= 768) {
            return 2;
          } else {
            return 1;
          }
        }


        /**
        * @name approximateShortestColumn
        * @desc An algorithm for approximating which column is shortest
        * @returns The index of the shortest column
        * @memberOf thinkster.posts.controllers.PostsController
        */
        function approximateShortestColumn() {
          var scores = vm.columns.map(columnMapFn);

          return scores.indexOf(Math.min.apply(this, scores));

          
          /**
          * @name columnMapFn
          * @desc A map function for scoring column heights
          * @returns The approximately normalized height of a given column
          */
          function columnMapFn(column) {
            var lengths = column.map(function (element) {
              return element.content.length;
            });

            return lengths.reduce(sum, 0) * column.length;
          }


          /**
          * @name sum
          * @desc Sums two numbers
          * @params {Number} m The first number to be summed
          * @params {Number} n The second number to be summed
          * @returns The sum of two numbers
          */
          function sum(m, n) {
            return m + n;
          }
        }


        /**
        * @name render
        * @desc Renders Posts into columns of approximately equal height
        * @param {Array} current The current value of `vm.posts`
        * @param {Array} original The value of `vm.posts` before it was updated
        * @memberOf thinkster.posts.controllers.PostsController
        */
        function render(current, original) {
          if (current !== original) {
            vm.columns = [];

            for (var i = 0; i < calculateNumberOfColumns(); ++i) {
              vm.columns.push([]);
            }

            for (var i = 0; i < current.length; ++i) {
              var column = approximateShortestColumn();

              vm.columns[column].push(current[i]);
            }
          }
        }
      }
    })();

{x: posts_controller}
Haz un nuevo controlador llamado `PostsController` en `static/javascripts/posts/controllers/posts.controller.js`

Incluye este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/posts/controllers/posts.controller.js' %}"></script>

{x: posts_controller_include_js}
Incluye `posts.controller.js` en `javascript.html`

No vale la pena pasar tiempo explicando este controlador linea por linea. Es suficiente con decir que este controlador presenta un algoritmo para asegurar que las columnas de los posts tienen aproximadamente la misma anchura.

La unica cosa que mencionaremos aquí es la linea:

    $scope.$watchCollection(function () { return $scope.posts; }, render);

Ya que no tenemos un acceso directo a ViewModel donde se almacenan los `post`, buscaremos en `$scope.posts` en vez de `vm.posts`. Ademas, utilizamos `$watchCollection` aquí por que `$scope.posts` es un array. `$watch` busca las referencias a objetos, no su valor actual. `$watchCollection` busca el valor de un array desde sus cambios. Si utilizamos, `$watch` aquí en vez de `$watchCollection`, los cambios provocados por `$scope.posts.shift()` y `$scope.posts.unshift()` no darían lugar a ninguna busqueda.

##Creando una plantilla para la directiva posts
En nuestra directiva hemos definido una `templateURL` que no señala a ninguna plantilla existente. Sigamos adelante y creemos una nueva plantilla.

Crea `static/templates/posts/posts.html` con el siguiente contenido:

    <div class="row" ng-cloak>
      <div ng-repeat="column in vm.columns">
        <div class="col-xs-12 col-sm-6 col-md-4 col-lg-3">
          <div ng-repeat="post in column">
            <post post="post"></post>
          </div>
        </div>
      </div>

      <div ng-hide="vm.columns && vm.columns.length">
        <div class="col-sm-12 no-posts-here">
          <em>The are no posts here.</em>
        </div>
      </div>
    </div>

{x: posts_template}
Crea una plantilla para la directiva `posts`

Un par de cosas a tener en cuenta:

1. Utilizamos la directiva `ng-cloak` para evitar parpadeo ya que esta directiva se utilizará en la primera página cargada.
2. Necesitaremos crear una directiva `post` para renderizar cada post individualmente.
3. Si no existen posts, rendiremos un mensaje informando al usuario.

## Creando una directiva para mostrar un único Post
En la plantilla de la directiva posts, hemos utilizado otra directiva llamada `post`. Vamos a crearla.

Crea `static/javascripts/posts/directives/post.directive.js` con el siguiente contenido:

/**
    * Post
    * @namespace thinkster.posts.directives
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.posts.directives')
        .directive('post', post);

      /**
      * @namespace Post
      */
      function post() {
        /**
        * @name directive
        * @desc The directive to be returned
        * @memberOf thinkster.posts.directives.Post
        */
        var directive = {
          restrict: 'E',
          scope: {
            post: '='
          },
          templateUrl: '/static/templates/posts/post.html'
        };

        return directive;
      }
    })();

{x: post_directive}
Crea la nueva directiva llamada `post` en `static/javascripts/posts/directives/post.directive.js`

Incluye este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/posts/directives/post.directive.js' %}"></script>

{x: post_directive_include_js}
Añade `post.directive.js` en `javascripts.html`

No hay nada nuevo que discutir aquí. Esta directiva es idéntica a la anterior. La única diferencia es que hemos utilizado otra plantilla.

##Creando una plantilla para la directiva post
Como hicimos para la directiva `posts`, ahora necesitaremos una plantilla para la directiva `post`.

Crea `static/templates/posts/post.html` con el siguiente contenido:

  <div class="row">
      <div class="col-sm-12">
        <div class="well">
          <div class="post">
            <div class="post__meta">
              <a href="/+{{ post.author.username }}">
                +{{ post.author.username }}
              </a>
            </div>

            <div class="post__content">
              {{ post.content }}
            </div>
          </div>
        </div>
      </div>
    </div>

{x: post_template}
Crea la plantilla para la directiva `post`

## Algo de CSS rapido
Queremos añadir estilos simples para que nuestros posts tengan un mejor aspecto. Abre `static/stylesheets/styles.css` y añade lo siguiente:

    .no-posts-here {
      text-align: center;
    }

    .post {}

    .post .post__meta {
      font-weight: bold;
      text-align: right;
      padding-bottom: 19px;
    }

    .post .post__meta a:hover {
      text-decoration: none;
    }

{x: post_css}
Añade algo de CSS `static/stylesheets/style.css` para que los posts tengan un mejor aspecto

## Punto de control
Asumiendo que todo esta bien, puedes verificar que estas en la buena vía cargando `http://localhost:8000/` en tu navegador. ¡Deberías ver los objetos `Post` que has creado en la ultima sección!

Esto también confirma que `PostViewSet` de la ultima sección esta funcionando

{x: checkpoint_render_posts}
Visita `http://localhost:8000/` y confirma que los objetos `Post` que has creado antes se muestran.

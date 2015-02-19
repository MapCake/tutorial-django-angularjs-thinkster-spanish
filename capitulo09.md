# Mostrando el perfil de usuario

{video: display-user-profile}

Ya hemos hecho las vistas Django y rutas necesarias para mostrar un perfil para cada usuario. Así que ahora podemos saltar a la creación de un servicio AngularJS y continuar con los controladores y plantillas.

{info}
En esta sección y la siguiente nos referiremos a cuentas y perfiles. Para el propósito de nuestro cliente, esto es en lo que se traduce el modelo `Accounts`: un perfil de usuario.


## Haciendo el modulo perfil
Ya hemos creado el servicio y un par de controladores de nuestro perfil de usuario, así que sigamos adelante y definamos los modulos que necesitaremos.

Crea `static/javascripts/profiles/profiles.module.js` con el siguiente contenido:

    (function () {
      'use strict';

      angular
        .module('thinkster.profiles', [
          'thinkster.profiles.controllers',
          'thinkster.profiles.services'
        ]);

      angular
        .module('thinkster.profiles.controllers', []);

      angular
        .module('thinkster.profiles.services', []);
    })();

{x: profile_modules}
Define los modulos necesarios para los perfiles en `profiles.module.js`

Como siempre, no te olvides de registrar `thinkster.profiles` como una dependencia de `thinkster` en `thinkster.js`:

    angular
      .module('thinkster', [
        'thinkster.config',
        'thinkster.routes',
        'thinkster.authentication',
        'thinkster.layout',
        'thinkster.posts',
        'thinkster.profiles'
      ]);

{x: thinkster_profile_module}
Registra `thinkster.profiles` como dependencia del modulo `thinkster`

Incluye este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/profiles.module.js' %}"></script>

## Creando la factoria profile
Con la definición del modulo en su sitio, estamos listos para crear el servicio `Profile` que comunicara con nuestra API.

Crea `static/javascript/profiles/services/profile.service.js` con el siguiente contenido:

    /**
    * Profile
    * @namespace thinkster.profiles.services
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.profiles.services')
        .factory('Profile', Profile);

      Profile.$inject = ['$http'];

      /**
      * @namespace Profile
      */
      function Profile($http) {
        /**
        * @name Profile
        * @desc The factory to be returned
        * @memberOf thinkster.profiles.services.Profile
        */
        var Profile = {
          destroy: destroy,
          get: get,
          update: update
        };

        return Profile;

        /////////////////////

        /**
        * @name destroy
        * @desc Destroys the given profile
        * @param {Object} profile The profile to be destroyed
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function destroy(profile) {
          return $http.delete('/api/v1/accounts/' + profile.id + '/');
        }


        /**
        * @name get
        * @desc Gets the profile for user with username `username`
        * @param {string} username The username of the user to fetch
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function get(username) {
          return $http.get('/api/v1/accounts/' + username + '/');
        }


        /**
        * @name update
        * @desc Update the given profile
        * @param {Object} profile The profile to be updated
        * @returns {Promise}
        * @memberOf thinkster.profiles.services.Profile
        */
        function update(profile) {
          return $http.put('/api/v1/accounts/' + profile.username + '/', profile);
        }
      }
    })();

{x: profiles_factory}
Crea la nueva factoría llamada `Profiles` en `static/javascripts/profiles/services/profiles.service.js`

No estamos haciendo nada en particular aquí. Cada llamada a la API es una operación básica de CRUD (Créate, , Retrieve, Update, Delete), así que estamos lejos de tener mucho código aquí.

Añade este fichero a `javascripts.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/services/profile.service.js' %}"></script>

{x: profiles_service_include_js}
Incluye `profiles.service.js` en `javascript.html`

## Creando una interfaz para los perfiles de usuario
Crea `static/templates/profiles/profile.html` con el siguiente contenido:

    <div class="profile" ng-show="vm.profile">
      <div class="jumbotron profile__header">
        <h1 class="profile__username">+{{ vm.profile.username }}</h1>
        <p class="profile__tagline">{{ vm.profile.tagline }}</p>
      </div>

      <posts posts="vm.posts"></posts>
    </div>

{x: profile_template}
Crea una plantilla para mostrar los perfiles en `static/templates/profiles/profile.html`

Esto renderizara una cabecera con el nombre de usuario y un tagline para el propietario del perfil, seguido de una lista de sus posts. Los posts estarán renderizados utilizando la directiva que hemos creado anteriormente para la pagina indice.

## Controlando la interfaz de perfiles con ProfileController
El siguiente paso es crear un controlador que utilizara el servicio que acabamos de crear, junto con el servicio `Post`, para recupera los datos que queremos mostrar.

Crea `static/javascripts/profiles/controllers/profile.controller.js` con el contenido siguiente:

    /**
    * ProfileController
    * @namespace thinkster.profiles.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.profiles.controllers')
        .controller('ProfileController', ProfileController);

      ProfileController.$inject = ['$location', '$routeParams', 'Posts', 'Profile', 'Snackbar'];

      /**
      * @namespace ProfileController
      */
      function ProfileController($location, $routeParams, Posts, Profile, Snackbar) {
        var vm = this;

        vm.profile = undefined;
        vm.posts = [];

        activate();

        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.profiles.controllers.ProfileController
        */
        function activate() {
          var username = $routeParams.username.substr(1);

          Profile.get(username).then(profileSuccessFn, profileErrorFn);
          Posts.get(username).then(postsSuccessFn, postsErrorFn);

          /**
          * @name profileSuccessProfile
          * @desc Update `profile` on viewmodel
          */
          function profileSuccessFn(data, status, headers, config) {
            vm.profile = data.data;
          }


          /**
          * @name profileErrorFn
          * @desc Redirect to index and show error Snackbar
          */
          function profileErrorFn(data, status, headers, config) {
            $location.url('/');
            Snackbar.error('That user does not exist.');
          }


          /**
            * @name postsSucessFn
            * @desc Update `posts` on viewmodel
            */
          function postsSuccessFn(data, status, headers, config) {
            vm.posts = data.data;
          }


          /**
            * @name postsErrorFn
            * @desc Show error snackbar
            */
          function postsErrorFn(data, status, headers, config) {
            Snackbar.error(data.data.error);
          }
        }
      }
    })();

{x: profile_controller}
Crea un nuevo controlador llamado `ProfileController` en `static/javascripts/profiles/controllers/profile.controller.js`

Incluye este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/controllers/profile.controller.js' %}"></script>

{x: profile_controller_include_js}
Incluye `profile.controller.js` en `javascript.html`

## Creando una ruta para ver el perfil de usuario
Abre `static/javascript/thinkster.routes.js` y añade la siguiente ruta:

    .when('/+:username', {
        controller: 'ProfileController',
        controllerAs: 'vm',
        templateUrl: '/static/templates/profiles/profile.html'
    })

{x: profile_route}
Crea una ruta para ver los perfiles de usuario

## Punto de control
Para ver tu perfil de usuario, redirige tu navegador a `http://localhost:8000/+<username>`. Si la pagina se renderiza, ¡todo esta bien!

{x: checkpoint_display_user_profile}
Visita tu pagina de perfil en `http://localhost:8000/+<username>`

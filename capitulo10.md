# Actualizando los Perfiles de Usuario

La ultima característica que vamos a implementar en este tutorial es la habilidad de que el usuario pueda actualizar su perfil. Las actualizaciones que ofreceremos serán minimas, incluyendo actualizar el nombre, apellidos email y tagline, pero con esto aprenderás lo esencial, y podrás añadir opciones a voluntad.

{video: update-user-profile}

## ProfileSettingsController
Para empezar, abre `static/javascripts/profiles/controllers/profile-settings.controller.js` y añade el siguiente contenido:

    /**
    * ProfileSettingsController
    * @namespace thinkster.profiles.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.profiles.controllers')
        .controller('ProfileSettingsController', ProfileSettingsController);

      ProfileSettingsController.$inject = [
        '$location', '$routeParams', 'Authentication', 'Profile', 'Snackbar'
      ];

      /**
      * @namespace ProfileSettingsController
      */
      function ProfileSettingsController($location, $routeParams, Authentication, Profile, Snackbar) {
        var vm = this;

        vm.destroy = destroy;
        vm.update = update;

        activate();


        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated.
        * @memberOf thinkster.profiles.controllers.ProfileSettingsController
        */
        function activate() {
          var authenticatedAccount = Authentication.getAuthenticatedAccount();
          var username = $routeParams.username.substr(1);

          // Redirect if not logged in
          if (!authenticatedAccount) {
            $location.url('/');
            Snackbar.error('You are not authorized to view this page.');
          } else {
            // Redirect if logged in, but not the owner of this profile.
            if (authenticatedAccount.username !== username) {
              $location.url('/');
              Snackbar.error('You are not authorized to view this page.');
            }
          }

          Profile.get(username).then(profileSuccessFn, profileErrorFn);

          /**
          * @name profileSuccessFn
          * @desc Update `profile` for view
          */
          function profileSuccessFn(data, status, headers, config) {
            vm.profile = data.data;
          }

          /**
          * @name profileErrorFn
          * @desc Redirect to index
          */
          function profileErrorFn(data, status, headers, config) {
            $location.url('/');
            Snackbar.error('That user does not exist.');
          }
        }


        /**
        * @name destroy
        * @desc Destroy this user's profile
        * @memberOf thinkster.profiles.controllers.ProfileSettingsController
        */
        function destroy() {
          Profile.destroy(vm.profile.username).then(profileSuccessFn, profileErrorFn);

          /**
          * @name profileSuccessFn
          * @desc Redirect to index and display success snackbar
          */
          function profileSuccessFn(data, status, headers, config) {
            Authentication.unauthenticate();
            window.location = '/';

            Snackbar.show('Your account has been deleted.');
          }


          /**
          * @name profileErrorFn
          * @desc Display error snackbar
          */
          function profileErrorFn(data, status, headers, config) {
            Snackbar.error(data.error);
          }
        }


        /**
        * @name update
        * @desc Update this user's profile
        * @memberOf thinkster.profiles.controllers.ProfileSettingsController
        */
        function update() {
          Profile.update(vm.profile).then(profileSuccessFn, profileErrorFn);

          /**
          * @name profileSuccessFn
          * @desc Show success snackbar
          */
          function profileSuccessFn(data, status, headers, config) {
            Snackbar.show('Your profile has been updated.');
          }


          /**
          * @name profileErrorFn
          * @desc Show error snackbar
          */
          function profileErrorFn(data, status, headers, config) {
            Snackbar.error(data.error);
          }
        }
      }
    })();

{x: profile_settings_controller}
Crea el controlador `ProfileSettingsController`

Asegurate de incluir este fichero en `javascript.html`:

    <script type="text/javascript" src="{% static 'javascripts/profiles/controllers/profile-settings.controller.js' %}"></script>

{x: profile_settings_controller_include_js}
Incluye `profile-settings.controller.js` en `javascritps.html`

Aqui hemos creado dos métodos que estaban disponibles en la vista: `update` y `destroy`. Como sugieren sus nombre, `update` permitirá al usuario actualizar su perfil y `destroy` eliminará la cuenta de usuario.

La mayoría de este controlador te sera familiar, pero vamos a repasar algunos métodos que hemos creado para aclarar.

    /**
    * @name activate
    * @desc Actions to be performed when this controller is instantiated.
    * @memberOf thinkster.profiles.controllers.ProfileSettingsController
    */
    function activate() {
      var authenticatedAccount = Authentication.getAuthenticatedAccount();
      var username = $routeParams.username.substr(1);

      // Redirect if not logged in
      if (!authenticatedAccount) {
        $location.url('/');
        Snackbar.error('You are not authorized to view this page.');
      } else {
        // Redirect if logged in, but not the owner of this profile.
        if (authenticatedAccount.username !== username) {
          $location.url('/');
          Snackbar.error('You are not authorized to view this page.');
        }
      }

      Profile.get(username).then(profileSuccessFn, profileErrorFn);

      /**
      * @name profileSuccessFn
      * @desc Update `profile` for view
      */
      function profileSuccessFn(data, status, headers, config) {
        vm.profile = data.data;
      }

      /**
      * @name profileErrorFn
      * @desc Redirect to index
      */
      function profileErrorFn(data, status, headers, config) {
        $location.url('/');
        Snackbar.error('That user does not exist.');
      }
    }


En `activate`, seguimos un patron familiar. Ya que esta pagina permite realizar operaciones peligrosas, debemos estar seguros que el usuario actual esta autorizado a ver esta pagina. Hacemos esto primero, verificando que el usuario esta autenticado y y después verificando si el usuario autenticado es el propietario del perfil. Si en cualquiera de estos casos hay algo falso, redirigimos a la pagina indice con un error en la snackbar diciendo que el usuario no esta autorizado a ver esta pagina.

Si el proceso de autorización tiene éxito, simplemente grabamos el perfil de usuario desde el servidor y permitimos al usuario hacer lo que desee.

    /**
    * @name destroy
    * @desc Destroy this user's profile
    * @memberOf thinkster.profiles.controllers.ProfileSettingsController
    */
    function destroy() {
      Profile.destroy(vm.profile.username).then(profileSuccessFn, profileErrorFn);

      /**
      * @name profileSuccessFn
      * @desc Redirect to index and display success snackbar
      */
      function profileSuccessFn(data, status, headers, config) {
        Authentication.unauthenticate();
        window.location = '/';

        Snackbar.show('Your account has been deleted.');
      }


      /**
      * @name profileErrorFn
      * @desc Display error snackbar
      */
      function profileErrorFn(data, status, headers, config) {
        Snackbar.error(data.error);
      }
    }

Cuando un usuario esa borrar su perfil, debemos desautenticarlo y redirigirlo a la pagina indice, realizando un refrescado de la pagina en el proceso. Esto hara que la barra de navegación vuelva a renderizarse con la vista sin conexión de cuenta.

Si por alguna razón, el borrado devuelve un código de estado de error, simplemente mostraremos la snackbar con el mensaje de error devuelto por el servidor. No haremos ninguna otra acción ya que no vemos ninguna razón para que falle esta llamada, al menos que el usuario no este autorizado a eliminar el perfil, pero ya hemos contemplado este escenario en el método activate.

    /**
    * @name update
    * @desc Update this user's profile
    * @memberOf thinkster.profiles.controllers.ProfileSettingsController
    */
    function update() {
      Profile.update(vm.profile).then(profileSuccessFn, profileErrorFn);

      /**
      * @name profileSuccessFn
      * @desc Show success snackbar
      */
      function profileSuccessFn(data, status, headers, config) {
        Snackbar.show('Your profile has been updated.');
      }


      /**
      * @name profileErrorFn
      * @desc Show error snackbar
      */
      function profileErrorFn(data, status, headers, config) {
        Snackbar.error(data.error);
      }
    }

`update()` es bastante simple. Si la llamada tiene éxito o no mostraremos una snack-bar con el mensaje apropiado.

## Una plantilla para la pagina settings
Como de costumbre, ahora que tenemos el controlador, necesitamos crear la plantilla correspondiente.

Crea `static/templates/profiles/settings.html` con el siguiente contenido:

    <div class="col-md-4 col-md-offset-4">
      <div class="well" ng-show="vm.profile">
        <form role="form" class="settings" ng-submit="vm.update()">
          <div class="form-group">
            <label for="settings__email">Email</label>
            <input type="text" class="form-control" id="settings__email" ng-model="vm.profile.email" placeholder="ex. john@example.com" />
          </div>

          <div class="form-group">
            <label for="settings__password">New Password</label>
            <input type="password" class="form-control" id="settings__password" ng-model="vm.profile.password" placeholder="ex. notgoogleplus" />
          </div>

          <div class="form-group">
            <label for="settings__confirm-password">Confirm Password</label>
            <input type="password" class="form-control" id="settings__confirm-password" ng-model="vm.profile.confirm_password" placeholder="ex. notgoogleplus" />
          </div>

          <div class="form-group">
            <label for="settings__username">Username</label>
            <input type="text" class="form-control" id="settings__username" ng-model="vm.profile.username" placeholder="ex. notgoogleplus" />
          </div>

          <div class="form-group">
            <label for="settings__tagline">Tagline</label>
            <textarea class="form-control" id="settings__tagline" ng-model="vm.profile.tagline" placeholder="ex. This is Not Google Plus." />
          </div>

          <div class="form-group">
            <button type="submit" class="btn btn-primary">Submit</button>
            <button type="button" class="btn btn-danger pull-right" ng-click="vm.destroy()">Delete Account</button>
          </div>
        </form>
      </div>
    </div>

{x: profile_settings_html}
Crea la plantilla para `ProfileSettingsController`

Esta plantilla es similar a los formularios que hemos creado para registrar y conectar. No hay nada aquí que valga la pena discutir.

## Ruta para las propiedades del perfil
Abre `static/javascript/thinkster.route.js` y añade la siguiente ruta:

    // ...
    .when('/+:username/settings', {
        controller: 'ProfileSettingsController',
        controllerAs: 'vm',
        templateUrl: '/static/templates/profiles/settings.html'
    })
    // ...

## Punto de control
¡Esta es nuestra ultima funcionalidad! Ahora deberías ser capaz de cargar la pagina de propiedades en `http://localhost:8000/+:username/settings` y actualizar tus datos a tu gusto.

Intenta actualizar tu tagline. Si funciona, deberias ver como se muestra tu nuevo tagline en tu pagina de perfil.

{x: checkpoint_update_user_profile}
Actualiza tu tagline y visualiza tu nuevo tagline en tu pagina de perfil

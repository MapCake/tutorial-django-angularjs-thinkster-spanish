# Conectando a los usuarios
Ahora que los usuarios pueden registrarse, necesitan una forma de conectarse. Resulta que, esto es parte de lo que nos falta en nuestro sistema de registro. Una vez que un usuario se registra, debería estar conectado de forma automática.

{video: log-user-in}

Para empezar, crearemos las vistas para conectarse y desconectarse. Una vez hecho esto, progresaremos de la misma manera que para el sistema de registro: servicios, controladores, etc.

## Construyendo la vista login de la API
Abre `authentication/views.py` y añade lo siguiente:

    import json

    from django.contrib.auth import authenticate, login

    from rest_framework improt status, views
    from rest_framework.response import Response

    class LoginView(views.APIView):
        def post(self, request, format=None):
            data = json.loads(request.body)

            email = data.get('email', None)
            password = data.get('password', None)

            account = authenticate(email=email, password=password)

            if account is not None:
                if account.is_active:
                    login(request, account)

                    serialized = AccountSerializer(account)

                    return Response(serialized.data)
                else:
                    return Response({
                        'status': 'Unauthorized',
                        'message': 'This account has been disabled.'
                    }, status=status.HTTP_401_UNAUTHORIZED)
            else:
                return Response({
                    'status': 'Unauthorized',
                    'message': 'Username/password combination invalid.'
                }, status=status.HTTP_401_UNAUTHORIZED)

{x: django_login_view}
Crea una vista llamada `LoginView` en `authentication/views.py`

Este es un trozo largo que hemos visto anteriormente, pero vamos a revisarlo de la misma manera: hablando de lo que es nuevo e ignorando lo que ya hemos visto.

    class LoginView(views.APIView):

Notaras que no estamos utilizando vistas genéricas esta vez. Ya que esta vista no tiene una actividad perfecta genérica como crear o actualizar un objeto, debemos empezar con algo mas básico. Utilizaremos `views.APIView` de Django REST Framework. Mientras que `APIView` no hace todo por nosotros nos ofrece mas funcionalidades que un vista standard de Django. En particular, `views.APIView` esta hecha especialmente para gestionar peticiones AJAX. Esto nos va a permitir ahorrar tiempo.

    def post(self, request, format=None):

Al contrario que las vistas genéricas, debemos gestionar cada verbo HTTP por nosotros mismos. Conectar debería ser una petición `POST` típica, así que sobreescribimos el metodo `self.post()`.

    account = authenticate(email=email, password=password)

Django nos proporciona un buen conjunto de utilidades para autentificar usuarios. El método `authenticate()` es la primera herramienta que vamos a cubrir. `autenticase()` toma un email y una contraseña. Django comprueba en la base de datos que existe una cuenta `Account` con el email `email`. Si se encuentra uno, Django intentará verificar la contraseña dada. Si el nombre de usuario y la contraseña son correctos, se devuelve la cuenta `Account` encontrada por `authenticate()`. Si alguno de estos pasos falla, `authenticate()` devuelve `None`.

    if account is not None:
        # ...
    else:
        return Response({
            'status': 'Unauthorized',
            'message': 'This account has been disabled.'
        }, status=status.HTTP_401_UNAUTHORIZED)

En el caso en que `authenticate()` devuelva `None`, responderemos con un código de estado `401` y le diremos al usuario que la combinación email/password que ha proporcionado no es valida.

    if account.is_active:
        # ...
    else:
        return Response({
            'status': 'Unauthorized',
            'message': 'This account has been disabled.'
        }, status=status.HTTP_401_UNAUTHORIZED)

Si por cualquier razón, la cuenta del usuario esta inactiva, responderemos con un código de estado `401`. Simplemente diremos que la cuenta esta desactivada.

    login(request, account)

Si `authenticate()` tiene éxito y el usuario esta activo, entonces utilizamos la herramienta `login()` de Django para crear una nueva sesión para este usuario.

    serialized = AccountSerializer(account)

    return Response(serialized.data)

Queremos almacenar alguna informacion de este usuario en el navegador si la petición de conexión tiene éxito, asi que serializamos el objeto `Account` encontrado por `authenticate()` y devolvemos como respuesta el JSON resultante.

## Añadiendo un punto final a la API de conexion
Como ya hicimos para `AccountViewSet`, necesitamos añadir una ruta para `LoginView`.

Abre `thinkster_django_angular_boilerplate/urls.py` y añade la siguente URL entre `^/api/v1/` y `^`:

    from authentication.views import LoginView

    urlpatterns = patterns(
        # ...
        url(r'^api/v1/auth/login/$', LoginView.as_view(), name='login'),
        # ...
    )

{x: url_login}
Añade el punto final para `LoginView`

## Servicio de autentificación
Vamos a añadir algunos métodos a nuestro servicio `Authentication`. Lo vamos a hacer en dos pasos. Primero añadiremos el método `login()` y después añadiremos otros métodos útiles para almacenar datos de la sesión en el navegador.

Abre el fichero `static/javascript/authentication/services/authentication.service.js` y añade el siguiente método al objeto `Authentication` que hemos creado antes:

    /**
    * @name login
    * @desc Try to log in with email `email` and password `password`
    * @param {string} email The email entered by the user
    * @param {string} password The password entered by the user
    * @ returns {Promise}
    * @memberOf thinkster.authentication.services.Authentication
    */
    function login(email, password) {
         return $http.post('/api/v1/auth/login/', {
            email: email, password: password
        });
    }

Asegúrate de de exponerlo como parte del servicio.

    var Authentication = {
        login:login,
        register: register,
    };

{x: angularjs_authentication_service_login}
Añade el metodo `login` al sevicio `Authentication`

Así como el anterior método `register()`, la respuesta del método `login()` es una petición AJAX a nuestra API y devuelve una promesa(?).

Ahora vamos a hablar sobre algunos métodos que necesitaremos para gestionar la información sobre la sesión en el cliente.

Queremos mostrar información sobre el usuario que esta conectado actualmente en la barra de navegación y en la cabecera de la pagina. Esto significa que necesitaremos una manera de almacenar la respuesta devuelta por `login()`. También necesitaremos una forma de recuperar el usuario conectado. También necesitaremos una forma de desconectar al usuario en el navegador. Y por ultimo, estaría bien tener una forma sencilla de comprobar si el usuario actual esta autentificado.

*NOTA: desautentificar es diferente a desconectar. cuando un usuario se desconecta, necesitamos una forma de suprimir todos los datos de la sesión que quedan en el cliente.

Con esto requerimientos, propongo 4 métodos: `getAuthenticatedAccount`, `isAuthenticated`, `setAuthenticatedAccount`, y `unauthenticate`.

Vamos a implementarlos ahora. Añade cada una de las funciones al servicio `Authenticate`:

    /**
     * @name getAuthenticatedAccount
     * @desc Return the currently authenticated account
     * @returns {object|undefined} Account if authenticated, else `undefined`
     * @memberOf thinkster.authentication.services.Authentication
     */
    function getAuthenticatedAccount() {
      if (!$cookies.authenticatedAccount) {
        return;
      }

      return JSON.parse($cookies.authenticatedAccount);
    }

Si no hay una cookie `authenticatedAccount` (creada en `setAuthenticatedAccount()`), entonces sale; si no devuelve el objeto parseado `usuario de la cookie.

    /**
     * @name isAuthenticated
     * @desc Check if the current user is authenticated
     * @returns {boolean} True is user is authenticated, else false.
     * @memberOf thinkster.authentication.services.Authentication
     */
    function isAuthenticated() {
      return !!$cookies.authenticatedAccount;
    }

Devuelve el valor de la cookie `authenticatedAccount`.

    /**
    * @name setAuthenticatedUser
    * @desc Stringify the account object and store it in a cookie
    * @param {Object} account The acount object to be stored
    * @returns {undefined}
    * @memberOf mapcake.authentication.services.Authentication
    */
    function setAuthenticatedAccount(account) {
      $cookies.authenticatedAccount = JSON.stringify(account);
    }

Selecciona la cookie `authenticatedAccount` en forma de cadena del objeto `account.

    /**
    * @name unauthenticate
    * @desc Delete the cookie where the account object is stored
    * @returns {undefined}
    * @memberOf mapcake.authentication.services.Authentication
    */
    function unauthenticate() {
      delete $cookies.authenticatedAccount;
    }

Suprime la cookie `authenticatedAccount.

Una vez mas, no olvides exponer los métodos como parte del servicio:

    var Authentication = {
        getAuthenticatedAccount: getAuthenticatedAccount,
        isAuthenticated: isAuthenticated,
        login: login,
        register: register,
        setAuthenticatedAccount: setAuthenticatedAccount,
        unauthenticate: unauthenticate
    };


{x: angularjs_authentication_service_utilities}
Añade los metodos `getAuthenticatedAccount`, `isAuthenticated`, `setAuthenticatedAccount` y `unauthenticate` al servicio `Authentication`.

Antes de continuar con la interfaz, vamos a actualizar rapidamente el metodo `login` del servicio `Authentication` para que utilice uno de estos nuevos métodos. Reemplaza `Authentication.login` por lo siguiente:

    /**
     * @name login
     * @desc Try to log in with email `email` and password `password`
     * @param {string} email The email entered by the user
     * @param {string} password The password entered by the user
     * @returns {Promise}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function login(email, password) {
      return $http.post('/api/v1/auth/login/', {
        email: email, password: password
      }).then(loginSuccessFn, loginErrorFn);

      /**
       * @name loginSuccessFn
       * @desc Set the authenticated account and redirect to index
       */
      function loginSuccessFn(data, status, headers, config) {
        Authentication.setAuthenticatedAccount(data.data);

        window.location = '/';
      }

      /**
       * @name loginErrorFn
       * @desc Log "Epic failure!" to the console
       */
      function loginErrorFn(data, status, headers, config) {
        console.error('Epic failure!');
      }
    }


{x: update_authentication_login}
Actualiza el metodo `Authnetication.login` para utilizar los nuevos métodos.

## Creando una interfaz de conexión
Ahora ya tenemos `Authentication.login()` para conectar a los usuarios, así que vamos a crear el formulario de conexión. Abre `static/templates/authentication/login.html` y añade el siguiente código HTML:

    <div class="row">
      <div class="col-md-4 col-md-offset-4">
        <h1>Login</h1>

        <div class="well">
          <form role="form" ng-submit="vm.login()">
            <div class="alert alert-danger" ng-show="error" ng-bind="error"></div>

            <div class="form-group">
              <label for="login__email">Email</label>
              <input type="text" class="form-control" id="login__email" ng-model="vm.email" placeholder="ex. john@example.com" />
            </div>

            <div class="form-group">
              <label for="login__password">Password</label>
              <input type="password" class="form-control" id="login__password" ng-model="vm.password" placeholder="ex. thisisnotgoogleplus" />
            </div>

            <div class="form-group">
              <button type="submit" class="btn btn-primary">Submit</button>
            </div>
          </form>
        </div>
      </div>
    </div>

{x: angularjs_login_template}
Crea la plantilla `login.html`

## Controlando la interfaz de conexión con LoginController
Crea un nuevo fichero en `static/javascript/authentication/controllers/` llamado `login.controller.js` y añade el siguiente contenido:

    /**
    * LoginController
    * @namespace thinkster.authentication.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.authentication.controllers')
        .controller('LoginController', LoginController);

      LoginController.$inject = ['$location', '$scope', 'Authentication'];

      /**
      * @namespace LoginController
      */
      function LoginController($location, $scope, Authentication) {
        var vm = this;

        vm.login = login;

        activate();

        /**
        * @name activate
        * @desc Actions to be performed when this controller is instantiated
        * @memberOf thinkster.authentication.controllers.LoginController
        */
        function activate() {
          // If the user is authenticated, they should not be here.
          if (Authentication.isAuthenticated()) {
            $location.url('/');
          }
        }

        /**
        * @name login
        * @desc Log the user in
        * @memberOf thinkster.authentication.controllers.LoginController
        */
        function login() {
          Authentication.login(vm.email, vm.password);
        }
      }
    })();

{x: angularjs_login_controller}
Crea el controlador llamado `LoginController` en `static/javascripts/authentication/controllers/login.controller.js`

Vamos a ver la función `activate`.

    function activate() {
        // If the user is authenticated, they should not be here.
        if (Authentication.isAuthenticated()) {
            $location.url('/');
        }
    }

Habras notado que utilizamos mucho una función llamada `activate` en este tutorial. No hay nada especialmente importante en este nombre; elegimos un nombre estándar para la función que se ejecutará cuando se instancia a un controlador en particular.

Como sugiere el comentario, si un usuario esta ya autentificado, no tiene nada que hacer en la pagina login. Resolvemos esto, redireccionandolo a la pagina indice.

Deberíamos hacer esto con la pagina de registro también. Cuando hemos escrito el controlador de registro, no teníamos `Authentication.isAuthenticated()`. Actualizaremos `RegisterController` en breve.

## De vuelta a RegisterController
Volviendo un paso atras, vamos a añadir una prueba a `RegisterController` y redirigir al usuario si ya esta autentificado.

Abre `static/javascripts/authentication/controllers/register.controller.js` y añade lo siguiente dentro de la definición del controlador:

    activate();
    
    /**
     * @name activate
     * @desc Actions to be performed when this controller is instantiated
     * @memberOf thinkster.authentication.controllers.RegisterController
     */
    function activate() {
      // If the user is authenticated, they should not be here.
      if (Authentication.isAuthenticated()) {
        $location.url('/');
      }
    }

{x: angularjs_register_controller_auth}
Redirecciona a la pagina de indice a los usuarios que ya están autentificados en `RegisterController`

Si recuerdas, también hemos hablado sobre conectar a un usuario de forma automática cuando se registra. Como ya hemos actualizado el contenido de registro, vamos a actualizar el método `register` del servicio `Authentication`.

Reemplaza `Authentication.register` por lo siguiente:

    /**
    * @name register
    * @desc Try to register a new user
    * @param {string} email The email entered by the user
    * @param {string} password The password entered by the user
    * @param {string} username The username entered by the user
    * @returns {Promise}
    * @memberOf thinkster.authentication.services.Authentication
    */
    function register(email, password, username) {
      return $http.post('/api/v1/accounts/', {
        username: username,
        password: password,
        email: email
      }).then(registerSuccessFn, registerErrorFn);

      /**
      * @name registerSuccessFn
      * @desc Log the new user in
      */
      function registerSuccessFn(data, status, headers, config) {
        Authentication.login(email, password);
      }

      /**
      * @name registerErrorFn
      * @desc Log "Epic failure!" to the console
      */
      function registerErrorFn(data, status, headers, config) {
        console.error('Epic failure!');
      }
    }

{x: angularjs_register_controller_login}
Actualiza `Authentication.register`

## Creando una ruta para la interfaz de login
El siguen paso es crear una ruta en el cliente para el formulario de conexión.

Abre `static/javascripts/thinkster.routes.js` y añade la ruta para el formulario de conexión:

    $routeProvider.when('/register', {
        controller: 'RegisterController',
        controllerAs: 'vm',
        templateUrl: '/static/templates/authentication/register.html'
    }).when('/login', {
        controller: 'LoginController',
        controllerAs: 'vm',
        templateUrl: '/static/templates/authentication/login.html'
    }).otherwise('/');

{x: angularjs_login_route}
Añade la ruta de `LoginContorller`

{info}
¿Has visto como puedes encadenar peticiones a `$routeProvider.when()`? Mas adelante ignoraremos las rutas ya añadidas para abreviar. Solamente ten en mente que esta llamadas deben encadenarse y la primera ruta valida tomara el control.

## Incluye los nuevos ficheros .js
Como podras imaginarte, solamente hemos creado un único fichero JavaScript desde la ultima vez: `login.controller.js`. Vamos a añadirlo a `javascritp.html` con los otros ficheros JavaScript:

    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/login.controller.js' %}"></script>

{x: angularjs_javascripts_login}
Añade `login.controller.js` a `javascript.html`

## Punto de control
Abre `http://localhost:8000/login` en el navegador y conéctate con el usuario que has creado anteriormente. Si esto funciona, la pagina debería redirigirte a `http://localhost:8000/` y la barra de navegación debería cambiar.

{x: checkpoint_login}
Conecta con uno de los usuarios creados anteriormente visitando `http://localhost:8000/login`
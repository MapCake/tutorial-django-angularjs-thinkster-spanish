# Registrando nuevos usuarios

{video: register-user-1}

En este punto tenemos modelos y serializadores necesarios para representar usuarios. Ahora necesitamos construir un sistema de autentificacion. Esto implica la creación de varias vistas e interfaces para registrarse, conectarse y desconectarse. También tocaremos un servicio de AngularJS de `Authentication`y algunos controles.

Como no podemos conectarnos con usuarios que no existen, tiene sentido empezar con el registro de usuarios.

Para registrar a un usuario necesitamos un punto final de la API que creará un objeto `Account`, un servicio AngularJS para hacer peticiones AJAX a la API y un formulario de registro. Vamos a crear el punto final de la API primero.

## Construyendo el set de vistas de la API de account
Abre `authentication/views.py` y reemplaza su contenido por el siguiente código:

    from rest_framework import permissions, viewsets

    from authentication.models import Account
    from authentication.permissions import IsAccountOwner
    from authentication.serializers import AccountSerializer


    class AccountViewSet(viewsets.ModelViewSet):
        lookup_field = 'username'
        queryset = Account.objects.all()
        serializer_class = AccountSerializer

        def get_permissions(self):
            if self.request.method in permissions.SAFE_METHODS:
                return (permissions.AllowAny(),)

            if self.request.method == 'POST':
                return (permissions.AllowAny(),)

            return (permissions.IsAuthenticated(), IsAccountOwner(),)

        def create(self, request):
            serializer = self.serializer_class(data=request.data)

            if serializer.is_valid():
                Account.objects.create_user(**serializer.validated_data)

                return Response(serializer.validated_data, status=status.HTTP_201_CREATED)

            return Response({
                'status': 'Bad request',
                'message': 'Account could not be created with received data.'
            }, status=status.HTTP_400_BAD_REQUEST)


{x: create_account_viewset}
Crea el set de vistas llamado `AccountViewSet` en `authentication/views.py`

Veamos que es cada cosa linea por linea:

    class AccountViewSet(viewsets.ModelViewSet):

El Framework REST de Django tienen una característica llamada viewsets. Un viewset, como el nombre en ingles implica, es un conjunto de vistas. Especificamente, `ModelViewSet` ofrece una interface para listar, crear, recuperar , actualizar y suprimir objetos de un modelo determinado.

    lookup_field = 'username'
    queryset = Account.objects.all()
    serializer_class = AccountSerializer

Aqui hemos definido el queryset y el serializador con el que va a trabajar el viewset. Django REST Framework utiliza el queryset especificado y el serializador para realizar las acciones comentadas anteriormente. Observa que también especificamos un atributo `lookup_field`. Como hemos dicho antes, utilizaremos el atributo `username` del modelo `Account` para buscar en las cuentas en vez de el atributo `id`. Al sobreescribir el atributo `lookup_filed` estamos hacen esto.

    def get_permissions(self):
        if self.request.method in permissions.SAFE_METHODS:
            return (permissions.AllowAny(),)

        if self.request.method == 'POST':
            return (permissions.AllowAny(),)

        return (permissions.IsAuthenticated(), IsAccountOwner(),)

El unico usuario que puede llamar a métodos "peligrosos" (como `update()` y `delete()`) es el propietario de la cuenta. Primero comprobamos si el usuario esta autentificado y entonces llamamos a un permiso personalizado que escribiremos en unos instantes. Este caso no se cumple cuando el método HTTP es `POST`. queremos que cualquier usuario pueda crear una cuenta.

Si el metodo HTTP de la petición ('GET', 'POST', etc) es "seguro", entonces cualquiera puede utilizar el punto final de la API.

    def create(self, request):
        serializer = self.serializer_class(data=request.data)

        if serializer.is_valid():
            Account.objects.create_user(**serializer.validated_data)

            return Response(serializer.validated_data, status=status.HTTP_201_CREATED)
        return Response({
            'status': 'Bad request',
            'message': 'Account could not be created with received data.'
        }, status=status.HTTP_400_BAD_REQUEST)


Cuando creamos un objeto utilizando el método del serializador `.save()`, los atributos del objeto se guardan de forma literal. Esto significa que que un usuario registrándose con la contraseña `'password'` tendrá su contraseña guardada como `'password'`. Esto no es buena por idea por dos razones: 1) Almacenar contraseñas en texto plano es un problema de seguridad masivo. 2) Django enmascara las contraseñas antes de compararlas, así que el usuario no sera capaz de conectarse a su cuenta utilizando la contraseña `'password'`.

Resolvemos este problema sobreescribiendo el metodo .create() de este conjunto de vistas utilizando `Account.objects.create_user()` para crear un objeto `Account`.

##Construyendo los permisos de IsAccountOwner.
Vamos a crear los permisos de `IsAccountOwner()` para la vista que acabamos de crear.

Crea un fichero llamado `authentication/permissions.py` con el siguiente contenido:

    from rest_framework import permissions


    class IsAccountOwner(permissions.BasePermission):
        def has_object_permission(self, request, view, account):
            if request.user:
               return account == request.user
            return False

{x: is_account_owner_permission}
Crea los permisos llamados `IsAccountOwner` en `authentication/permissions.py`

Estos permisos son muy básicos. Si existe un usuario asociado con la petición actual, comprobamos que ese usuario es el mismo que el del objeto `account`. Si no hay un usuario asociado con esta petición, simplemente devolvemos `False`.

## Añadiendo un punto final a la API
Ahora que hemos creado la vista, necesitamos añadirla al fichero de URLs. Abre `thinkster_django_angular_boilerplate/urls.py` y actualizalo para que parezca:

    # .. Imports
    from rest_framework_nested import routers

    from authentication.views import AccountViewSet

    router = routers.SimpleRouter()
    router.register(r'accounts', AccountViewSet)

    urlpatterns = patterns(
         '',
        # ... URLs
        url(r'^api/v1/', include(router.urls)),

        url('^.*$', IndexView.as_view(), name='index'),
    )

{x: url_account_view_set}
Añade el punto final de la API para `AccountViewSet`

{info}
Es importante que la ultima URL de la lista de URLs anterior sea siempre la ultima URL. Esto se conoce como passthrough or catch-all route. Acepta todas las peticiones que no corresponden con ninguna de las reglas anteriores  y pasa la petición al router de AngularJS para procesarla. El orden de las URLs normalmente no tiene importancia.

## Un servicio Angular para registrar nuevos usuarios
Con el punto final de la API configurado, podemos crear el servicio AngulareJS que gestionará la comunicación entre el cliente y el servidor.

Crea un fichero en `static/javascript/authentication/services` llamado `authentication.service.js` y añade el siguiente código:

{info}
Siente libre de quitar los comentarios de tu código. Toma mucho tiempo escribirlos todos! pero recomiendo comentar el código!

    /**
    * Authentication
    * @namespace thinkster.authentication.services
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.authentication.services')
        .factory('Authentication', Authentication);

      Authentication.$inject = ['$cookies', '$http'];

      /**
      * @namespace Authentication
      * @returns {Factory}
      */
      function Authentication($cookies, $http) {
        /**
        * @name Authentication
        * @desc The Factory to be returned
        */
        var Authentication = {
          register: register
        };

        return Authentication;

        ////////////////////

        /**
        * @name register
        * @desc Try to register a new user
        * @param {string} username The username entered by the user
        * @param {string} password The password entered by the user
        * @param {string} email The email entered by the user
        * @returns {Promise}
        * @memberOf thinkster.authentication.services.Authentication
        */
        function register(email, password, username) {
          return $http.post('/api/v1/accounts/', {
            username: username,
            password: password,
            email: email
          });
        }
      }
    })();

{x: angularjs_authentication_service}
Construye una factoría llamada Authentication en `static/javascript/authentication/services/authentivation.service.js`

Vamos a ver este código paso por paso:

    angular
        .module('thinkster.authentication.services')

AngularJS soporta el uso de módulos. Modularidad es una gran característica ya que promueve la encapsulación y desata el enganche. Haremos un uso exhaustivo del sistema de modelos de AngularJS en el tutorial. Por el momento, todo lo que debes saber es que este servicio esta en el modulo `thinkster.authentication.services`.

    .factory('Authentication', Authentication);

Esta linea registra una factoría llamada `Authentication` en el modulo de la linea anterior.

    function Authentication($cookies, $http) {

Aqui definimos la factoría que acabamos de registrar. Inyectamos los servicios `$cookies` y `$http` como dependencias. Utilizaremos `$cookies` después.

    var Authentication = {
        register: register
    };

Esto es una preferencia personal, pero encuentro que es mas legible definir tu servicio como el objeto definido anteriormente y devolverlo, dejando los detalles mas abajo en el fichero.

    function register(email, password, username) {

En este punto, el servicio Authentication solo tiene un método: `register`, que toma como parámetros un `username`, `password` y un `email`. Añadiremos mas métodos al servicio mas adelante.

    return $http.post('/api/v1/accounts/', {
        username: username,
        password: password,
        email: email
    });

Como hemos mencionado antes, necesitamos hacer una petición AJAX al punto final de la API que hemos hecho antes. Como datos, incluiremos `username`, `password` y `email` que son los parámetros que recibe este método. No tenemos ninguna razón de hacer nada particular con la respuesta, así que que dejaremos la llamada de `Authentication.register` devolver la respuesta.

## Creando una interfaz para registrar nuevos usuarios

{video: register-user-2}

Empecemos creando la interfaz que los usuarios utilizaran para registrarse. Crearemos un fichero en `static/templates/authnetication` llamado `register.html` con el siguiente contenido:

    <div class="row">
        <div class="col-md-4 col-md-offset-4">
            <h1>Register</h1>

            <div class="well">
                <form role="form" ng-submit="vm.register()">
                    <div class="form-group">
                        <label for="register__email">Email</label>
                        <input type="email" class="form-control" id="register__email" ng-model="vm.email" placeholder="ex. john@notgoogle.com" />
                    </div>

                    <div class="form-group">
                        <label for="register__username">Username</label>
                        <input type="text" class="form-control" id="register__username" ng-model="vm.username" placeholder="ex. john" />
                    </div>

                    <div class="form-group">
                        <label for="register__password">Password</label>
                        <input type="password" class="form-control" id="register__password" ng-model="vm.password" placeholder="ex. thisisnotgoogleplus" />
                    </div>

                    <div class="form-group">
                        <button type="submit" class="btn btn-primary">Submit</button>
                    </div>
                </form>
            </div>
        </div>
    </div>

{x: angularjs_register_template}
Crea la plantilla register.html

No vamos a dar demasiados detalles esta vez por que es HTML básico. muchas de las clases tienen de Bootstrap, que esta incluido en el proyecto de muestra. Solo hay dos lineas en las que vamos a fijarnos:

    <form role="form" ng-submit="vm.register()">

Esta es la linea responsable de llamar a `$scope.register`, que hemos configurado en nuestro controlador. `ng-submit` llamará a `vm.register` cuando el formulario se envíe. Si has utilizado Angular con anterioridad, probablemente hayas utilizado `$scope`. En este tutoría, optamos por evitar el uso de `$scope` cuando sea posible en favor de `vm` de ViewModel. Observa en la sección [Controllers](https://github.com/johnpapa/angularjs-styleguide#controllers) de la guía de estilo de AngularJS de John Papa’s para mas información sobre esto.

    <input type="email" class="form-control" id="register__email" ng-model="vm.email" placeholder="ex. john@notgoogle.com" />

En cada `<input />`, veras otra directiva, `ng-model`. `ng-model` es responsable para almacenar el valor de la entrada en ViewModel. Esto es como obtenemos username, password y email cuando se llama a `vm.register`.

## Controlando la interfaz con RegisterController
Con el servicio y la interfaz preparados, necesitaremos enlazar ambos. El controlador que vamos a crear, `RegisterController` nos permitirá llamar al método register del servicio `Authentication` cuando un usuario envía el formulario que acabamos de construir.

Crea un fichero en `static/javascripts/authentication/controllers/` que se llame `register.controller.js` y añade lo siguiente:

    /**
    * Register controller
    * @namespace thinkster.authentication.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.authentication.controllers')
        .controller('RegisterController', RegisterController);

      RegisterController.$inject = ['$location', '$scope', 'Authentication'];

      /**
      * @namespace RegisterController
      */
      function RegisterController($location, $scope, Authentication) {
        var vm = this;

        vm.register = register;

        /**
        * @name register
        * @desc Register a new user
        * @memberOf thinkster.authentication.controllers.RegisterController
        */
        function register() {
          Authentication.register(vm.email, vm.password, vm.username);
        }
      }
    })();


{x: angularjs_register_controller}
Crea un controlador llamado `RegisterController` en `static/javascripts/authentication/controllers/register.controller.js`

Como siempre, vamos a saltar las partes que ya conocemos y hablar de los nuevos conceptos

    .controller('RegisterController', RegisterController);

Esto es similar a la forma en la que hemos registrado nuestro servicio. La diferencia es que, esta vez, estamos registrando un controlador.

    vm.register = register;

`vm` permite al formulario que hemos creado, acceder al método `register` que hemos definido después en el controlador.

    Authentication.register(vm.email, vm.password, vm.username);

Aqui llamamos al servicio que hemos creado antes. Pasamos los valores de username, password y email desde `vm`.

## Rutas y Modulos para registrarse
Vamos a configurar algunas rutas del lado del cliente, así, los usuarios de la aplicación podran navegar hasta el formulario de registro.

Crea un fichero en `static/javascripts` llamado `thinkster.routes.js` y añade lo siguiente:

    (function () {
      'use strict';

      angular
        .module('thinkster.routes')
        .config(config);

      config.$inject = ['$routeProvider'];

      /**
      * @name config
      * @desc Define valid application routes
      */
      function config($routeProvider) {
        $routeProvider.when('/register', {
          controller: 'RegisterController', 
          controllerAs: 'vm',
          templateUrl: '/static/templates/authentication/register.html'
        }).otherwise('/');
      }
    })();


{x: angularjs_register_route}
Define una ruta para el formulario de registro

Hay algunos puntos que vamos a tratar.

    .config(config);

Angular, como cualquier framework que puedas imaginar, te permite editar su configuración. Esto se hace con un bloque `.config`.

    function config($routeProvider) {

Aquí estamos injectando `$routerProvider` como dependencia, la cual nos permitirá añadir la capacidad de erutar en el cliente.

    $routeProvider.when('/register', {

`$routerProvider.when` toma dos argumentos: un camino y un objeto options. Aquí utilizamos /register como el camino, ya que es aquí donde queremos que se muestre el formulario de registro.

    controller: 'RegisterController',
    controllerAs: 'vm',

En el objeto options podemos incluir varias claves, entre ellas `controller`. Esta opción mapeará un control para esta ruta. Aquí utilizamos el control `RegisterController` que hicimos antes. `controllerAs` es otra opción. Esta es necesaria para utilizar la variable `vm`. En resumen, estamos diciendo que nos queremos referir al controlador como `vm` en el template.

    templateUrl: '/static/templates/authentication/register.html'

La otra clave que utilizamos es `templateUrl`. Esta opción toma una cadena de la URL donde esta el template que queremos utilizar para esta ruta.

    }).otherwise('/');

Vamos a añadir mas rutas según vayamos avanzando, pero es posible que un usuario entre una URL que no soportamos. Cuando esto pase, `$routerProvider.otherwise` redirigirá al usuario al camino especificado. En este caso '/'.

## Configurando los modulos de AngularJS
Vamos a hablar un poco de los modelos de AngularJS.

En Angular, debes definir los modulos anetes de utilizarlos. Por el momento tendremos que definir `thinkster.authentication.services`, `thinkster.authentication.controllers` y `thinkster.routes`. Como `thinkster.authentication.services` y `thinkster.authentication.controllers` son submódulos de `thinkster.authentication`, necesitaremos crear el modulo `thinkster.authentication` tambien.

Crea un fichero en `static/javascripts/authentication` llamado `authentication.module.js` y añade lo siguiente:

    (function () {
      'use strict';

      angular
        .module('thinkster.authentication', [
          'thinkster.authentication.controllers',
          'thinkster.authentication.services'
        ]);

      angular
        .module('thinkster.authentication.controllers', []);

      angular
        .module('thinkster.authentication.services', ['ngCookies']);
    })();

{x: angularjs_authentication_module}
Define el modulo `thinkster.authentication` y sus dependencias

Hay algunas cosas interesantes que comentar aquí.

    angula
        .module('thinkster.authentication', [
            'thinkster.authentication.controllers',
            'thinkster,authentication.services'
        ]);

Esta sintaxis define el modulo `thinkster.authentication` con  `thinkster.authentication.controllers` y `thinkster.authentication.services` como dependencias.

    angular
        .module('thinkster.authentication.controllers', []);

Esta sintaxis define el modulo `thinkster.authentication.controllers` sin dependencias.

Ahora necesitaremos definir `thinkster.authentication` y `thinkster.routes` como dependencias de `thinkster`.

Abre el fichero `static/javascript/thinkster.js`, define los módulos requeridos e incluyelos como dependencias en el modulo `thinkster`. Observa que `thinkster.routes` enlaza con `ngRoute`, que esta incluido en el proyecto de referencia.

    (function () {
      'use strict';

      angular
        .module('thinkster', [
          'thinkster.routes',
          'thinkster.authentication'
        ]);

      angular
        .module('thinkster.routes', ['ngRoute']);
    })();

{x: angularjs_thinkster_module}
Actualiza el modulo thinkster para incluir las nuevas dependencias

## Enrutamiento Hash
Por defecto, Angular utiliza una característica llamada enrutamiento hash. Si alguna vez has visto alguna URL como esta `www.google.com/#/search` entonces sabes de lo que estamos hablando. Una vez mas, es una opinión personal, pero creo que esto es bastante feo. Para evitar este tipo de enrutamiento podemos activar `$locationProvider.html5Mode`. En navegadores que no soportan enrutamiento HTML5, Angular enrutará de forma inteligente con enrutamiento hash.

Crea un fichero en `static/javascripts/` llamado `thinkster.config.js` y añade el siguiente contenido:

    (function () {
      'use strict';

      angular
        .module('thinkster.config')
        .config(config);

      config.$inject = ['$locationProvider'];

      /**
      * @name config
      * @desc Enable HTML5 routing
      */
      function config($locationProvider) {
        $locationProvider.html5Mode(true);
        $locationProvider.hashPrefix('!');
      }
    })();


{x: angularjs_html5mode_config}
Activa el enrutamiento HTML para AngularJS.

Como hemos mencionado, activando `$locationProvider.html5Mode` eliminamos el enrutamiento hash de URLs. La otra configuración que vemos aquí, `$locationProvider.hashPrefix`, convierte `#` en `#!`. Esto es para beneficiar los motores de búsqueda principalmente.

Como estamos utilizando un nuevo modulo, necesitamos abrir `static/javascripts/thinkster.js`, define el modulo e incluye como dependencia en el modulo `thinkster`.

    angular
        .module(’thinkster', [
            ’thinkster.config',
            // ...
    ]);

    angular
        .module('thinkster.config', []);

{x: angularjs_config_module}
Define el modulo `thinkster.config`

## Incluye los nuevos ficheros .js
En este capitulo, hemos creado varios ficheros JavaScript. Necesitamos incluirlos en el cliente añadiéndolos en `template/javascripts.html` dentro del bloque `{% compress js %}`.

Abre el fichero `templates/javascripts.html` y añade las siguientes lineas antes de la sentencia '{% endcompress %}`

    <script type="text/javascript" src="{% static 'javascripts/thinkster.config.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/thinkster.routes.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/authentication.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/services/authentication.service.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/authentication/controllers/register.controller.js' %}"></script>

{x: django_javascripts}
Añade los nuevos ficheros javascript en `template/javascripts.html`.

## Manipulando la protección CSRF
Como estamos utilizando autentificacion basada en sesiones, necesitamos ocuparnos de la protección CSRF. No entraremos en detalles en CSRF por que esta fuera del objetivo de este tutoría, solo basta con saber que CSRF es bastante malo.

Django, por defecto, almacena un token CSRF en una cookie llamado `csrftoken` y espera una cabecera con el nombre `X-CSRFToken` para cualquier petición HTTP peligrosa (`POST`, `PUT`, `PATCH`, `DELETE`). Podemos configurar fácilmente Angular para gestionar esto.

Abre `static/javascripts/thinkster.js` y añade lo siguiente, después de la definición de los módulos:

    angular
        .module('thinkster')
            .run(run)

    run.$inject = ['$http'];

    /**
    * @name run
    * @desc Update xsrf $http headers to align with Django's defaults
    */
    function run($http) {
        $http.defaults.xsrfHeaderName = 'X-CSRFToken';
        $http.defaults.xsrfCookieName = 'csrftoken';
    }

{x: angularjs_run_csrf}
Configura CSRF en Angular

## Punto de control
Intenta registrar un nuevo usuario ejecutando el servidor (`python manage.py runserver`), visita la url `http://localhost:8000/register` en tu navegador y rellena y envía el formulario.

Si el proceso de registro funciona, podrás ver el nuevo objeto `Account` creado abriendo el terminal (`python manage.py shell`) y ejecutando los comando siguientes:

    >>> from authentication.models import Account
    >>> Account.objects.latest('created_at')

El objeto `Account` devuelto debería ser el que acabas de crear.

{x: checkpoint_register_account}
Registra un nuevo usuario en `http://localhost:8000/register` y confirma que el objeto `Account` se  ha creado.

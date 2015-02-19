# Desconcetando a los usuarios
Dado que los usuarios pueden registrarse y acceder, debemos imaginar que querran de alguna forma salir. La gente se vuelve loca si no puede desconectarse.

{video: log-user-out}

## Creando una vista de desconexion en la API
Vamos a implementar la ultima vista de la API relacionada con autenticación.

Abre `authentication/views.py` y añade los siguientes imports y clases:

    from django.contrib.auth import logout

    from rest_framework import permissions

    class LogoutView(views.APIView):
        permissions_classes = (permissions.IsAuthenticated,)

        def post(self, request, format=None):
            logout(request)

            return Response({}, status=status.HTTP_204_NO_CONTENT)

{x: django_logout_view}
Haz una vista llamada `LogoutView` en `authentication/views.py`

Solo hay unas pocas cosas nuevas de las que hablar esta vez.

    permissions_classes = (permissions.IsAuthenticated,)

Solo usuarios autenticados son capaces de acceder a este punto final. `permissions.IsAuthenticated` de DjangoREST Framework gestiona esto por nosotros. Si el usuario no esta autenticado, tendrá un error `403`.

    logout(request)

Si el usuario esta autenticado, lo único que debemos hacer es llamar al método `logout()` de Django.

    return Response({}, status=status.HTTP_204_NO_CONTENT)

No hay nada razonable para devolver cuando nos desconectamos, así que devolvemos un respuesta vacía y código de estado `200`.

Si vamos a las URL.

Abre `thinkster_django_angular_boilerplate/urls.py` otra vez y añade el siguiente import y la siguiente URL:

    from authentication.views import LogoutView

    urlpatterns = patterns(
        # ...
        url(r'^api/v1/auth/logout/$', LogoutView.as_view(), name='logout'),
        #...
    )

{x: django_url_logout}
Crea un punto final de la API para `LogoutView`

## Logout: Servicio AngularJS
El último metodo que necesitaremos añadir a nuestro servicio `Authentication` es el método `logout()`.

Añade el siguiente metodo al servicio `Authentication` en `authentication.service.js`:

    /**
     * @name logout
     * @desc Try to log the user out
     * @returns {Promise}
     * @memberOf thinkster.authentication.services.Authentication
     */
    function logout() {
      return $http.post('/api/v1/auth/logout/')
        .then(logoutSuccessFn, logoutErrorFn);

      /**
       * @name logoutSuccessFn
       * @desc Unauthenticate and redirect to index with page reload
       */
      function logoutSuccessFn(data, status, headers, config) {
        Authentication.unauthenticate();

        window.location = '/';
      }

      /**
       * @name logoutErrorFn
       * @desc Log "Epic failure!" to the console
       */
      function logoutErrorFn(data, status, headers, config) {
        console.error('Epic failure!');
      }
    }

Y como siempre, acuérdate de exponer `logout` como parte del servicio `Authentication`:

    var Authentication = { 
        getAuthenticatedUser: getAuthenticatedUser,
        isAuthenticated: isAuthenticated,
        login: login,
        logout: logout,
        register: register,
        setAuthenticatedUser: setAuthenticatedUser,
        unauthenticate: unauthenticate
    };

{x: angularjs_authentication_service_logout}
Añade el metodo `logout()` a tu servicio `Authentication`

## Controlando la barra de navegación con NavbarController
Por el moemnto no tenemos ni `LogoutController` ni `logout.html`. En lugar de esto, la barra de navegación ya contiene un enlace logout para usuarios registrados. Crearemos un `NavbarController` para gestionar la funcionalidad del botón logout cuando se selecciona y actualizaremos el enlace con un atributo `ng-click`.

Crea un fichero en `static/javascripts/layout/controllers/` llamado `navbar.controller.js` y añade lo siguiente en el:

    /**
    * NavbarController
    * @namespace thinkster.layout.controllers
    */
    (function () {
      'use strict';

      angular
        .module('thinkster.layout.controllers')
        .controller('NavbarController', NavbarController);

      NavbarController.$inject = ['$scope', 'Authentication'];

      /**
      * @namespace NavbarController
      */
      function NavbarController($scope, Authentication) {
        var vm = this;

        vm.logout = logout;

        /**
        * @name logout
        * @desc Log the user out
        * @memberOf thinkster.layout.controllers.NavbarController
        */
        function logout() {
          Authentication.logout();
        }
      }
    })();

{x: angularjs_navbar_controller}
Crea `NavbarController` en `static/javascripts/layout/controllers/navbar.controller.js`

Abre `templates/navbar.html` y añade una directiva `ng-controller` con el valor` NavbarController as vm ' al tag '<nav />` como sigue:

    <nav class="navbar navbar-default" role="navigation" ng-controller="NavbarController as vm">

Mientras `templates/navbar.html` esta abierto, busca el enlace logout y añade `ng-click=“vm.logout()”` para que parezca a lo siguiente:

    <li><a href="javascript:void(0)" ng-click="vm.logout()">Logout</a></li>

{x: angularjs_navbar_template_update}
Actualiza `navbar.html` para incluir las directivas `ng-controller` y `ng-click` donde corresponda

## Modulos Layout
Necesitaremos añadir algunos modulos en esta ocasión.

Crea un fichero en `static/javascripts/layout/` llamado `layout.module.js` y introduce el siguiente contenido:

    (function () {
        'use strict';

        angular
            .module('thinkster.layout', [
                'thinkster.layout.controllers'
            ]);

        angular
            .module('thinkster.layout.controllers', []);
        })();


Y no olvides actualizar `static/javascripts/thinkster.js`:

    angular
        .module('thinkster.', [
            'thinkster.config',
            'thinkster.routes',
            'thinkster.authentication',
            'thinkster.layout',
        ]);

{x: angularjs_static_module}
Define los nuevos modulos `thinkster.layout` y `thinkster.layout.controllers`

## Incluyendo los nuevos ficheros .js
Esta vez hay un par de ficheros nuevos a añadir. Abre `javascript.html` y añade lo siguiente:

    <script type="text/javascript" src="{% static 'javascripts/layout/layout.module.js' %}"></script>
    <script type="text/javascript" src="{% static 'javascripts/layout/controllers/navbar.controller.js' %}"></script>

{x: include_javascript_static}
Añade los nuevos ficheros JavaScript a `javascript.html`

## Punto de control
Si vas a `http://localhost:8000/` en el navegador, deberias estar aun conectado. Si no, vuelve a conectare.

Puedes verificar que la funcionalidad logout esta funcionando seleccionando el botón logout en la barra de navegación. Esto debería actualizar la pagina y actualizar la barra de navegación  a una vista de desconectado.

{x: checkpoint_logout}
Desconectate de tu cuenta utilizando el botón de la barra de navegación.
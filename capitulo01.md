# Extendiendo el modelo User por defecto de Django
Django posee un modelo `User` que ofrece muchas funcionalidades. El problema con este modelo `User` es que no se puede ampliar para incluir mas información. Por ejemplo, estaremos dando a nuestros usuarios un lema que se mostrará en su perfil. El modelo `User` de Django no tiene este atributo y no podemos añadir.

El modelo `User` hereda de `AbstractBaseUser`. De aqui es de donde `User` obtiene la mayoría de sus funcionalidades. Creando un nuevo modelo llamado ``Account`` y heredando de `AbstractBaseUser`, tendremos las funcionalidades necesarias de `User` (ocultado de password, gestión de sesiones, etc) y seremos capaces de extender ``Account`` para incluir información extra, como el lema (tagline). 

En Django, el concepto de “app” se usa para organizar el código de forma significativa. Una app es un module que alberga el código de modelos, vistas, serializadores, etc que están relacionados de alguna manera. En forma de ejemplo, nuestro primer paso en construir nuestra aplicación web en Django y AngularJS sera crear una app llamada `authentication`. La aplicación `authentication` contendrá el código relacionado con el modelo `Account` que acabamos de hablar, así como vistas para conectarse, desconectarse y registrarse.

Hacer una nueva aplicación llamada authentication ejecutando el comando siguiente:

    $ python manage.py startapp authentication

{x: startapp_authentication}
Crea una aplicación Django llamada authentication

## Creando el modelo Account

{video: account-manager}

Para empezar, vamos a crear el modelo `Account` del que acabamos de hablar.

Abre `authentication/models.py` en tu editor de texto favorito y y editalo con el siguiente contenido:

    from django.contrib.auth.models import AbstractBaseUser
    from django.db import models

    class Account(AbstractBaseUser):
        email = models.EmailField(unique=True)
        username = models.CharField(max_length=40, unique=True)

        first_name = models.CharField(max_length=40, blank=True)
        last_name = models.CharField(max_length=40, blank=True)
        tagline = models.CharField(max_length=140, blank=True)

        is_admin = models.BooleanField(default=False)

        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

        objects = AccountManager()

        USERNAME_FIELD = 'email'
        REQUIRED_FIELDS = ['username']

        def __unicode__(self):
            return self.email

        def get_full_name(self):
            return ' '.join([self.first_name, self.last_name])

        def get_short_name(self):
            return self.first_name


x: create_user_profile_model}
Crea el modelo `Account` en `authentication/models.py`

Vamos a examinar con detalle cada atributo y método por turnos.

    email = models.EmailField(unique=True)

    # ...

    USERNAME_FIELD = 'email'

El modelo `User` de Django por defecto, requiere un username o nombre de usuario. Este nombre de usuario se utiliza para conectar al usuario a la aplicación. Por el contrario, nuestra aplicación utilizará la dirección de email del usuario para este proposito.

Para decirle a Django que queremos utilizar el campo de email como el `username` para este modelo, seleccionamos el atributo `USERNAME_FIELD` a `email`. El campo especificado en `USERNAME_FIELD` debe ser único, así que pasamos el argumento `unique=True` en el campo email.

    username = models.CharField(max_length=40, unique=True)

Aunque el usuario utilice el email para conectarse, seguiremos queriendo que el usuario tenga un nombre de usuario. Necesitaremos uno para mostrarlo en sus post y pagina de perfil. También lo utilizaremos en nuestras URLs, así que deberá ser único. Así que pasaremos el argumento `unique=True` en el campo username con este fin.

    first_name = models.CharField(max_length=40, blank=True)
    last_name = models.CharField(max_length=40, blank=True)

Idealmente deberíamos tener una forma mas personal para referirnos a nuestros usuarios. También entendemos que no todos los usuarios están comodos dando detalles personales, así que hacemos opcionales los campos `first_name` y `last_name` pasando el argumento `blank=True`.

    tagline = models.CharField(max_length=140, blank=True)

como hemos mencionado antes, el atributo `tagline` se mostrará en el perfil de usuario. Esto da al perfil de usuario un toque personal, así que merece la pena incluirlo.

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

El campo `created_at` guarda la fecha y hora en la que el objeto `Account` fue creado. Pasando como argumento `auto_now_add=True` a `models.DateTimeField`, le estamos diciendo a Django que este campo debe rellenarse de forma automática una vez que se crea el objeto y que no es editable después de esto.

De forma similar a created_at, updated_at se rellena de forma automática por Django. La diferencia entre `auto_now_add=True` y `auto_now=True` es que `auto_now=True` se actualiza cada vez que el objeto se guarda.

    objects = AccountManager()

Cuando queremos obtener una instancia a un modelo en Django, usamos una expresión de tipo `Model.objects.get(**kwargs)`. El objeto, aquí es una clase `Manager` cuyo nombre sigue la convención `<model name>Manager`. En nuestro caso , crearemos un clase `AccountManager`. Haremos esto en unos instantes.

    REQUIRED_FIELDS = ['username']

Vamos a mostrar el nombre de usuario en varios sitios. Por eso, tener un nombre de usuario no es opcional, así que lo incluimos en la lista de campos requerido `REQUIRED_FIELDS`. Normalmente el argumento `required=True` cumple con este propósito, pero como este modelo esta reemplazando al modelo `User`, Django requiere que especifiquemos los campos obligatorios de esta forma.

    def __unicode__(self):
        return self.email

Cuando trabajamos en la linea de comandos, como veremos en breve, la representacion de la cadena de un objeto `Account` se parece a algo asi `<Account: Account>`. Esto no sera de mucha ayuda ya que vamos a tener diferentes tipos de cuentas. Sobreescribiendo `__unicode__()` cambiara el comportamiento por defecto. Aqui hemos decidido mostrar el email del usuario en su lugar. Asi que la representacion de la cadena de una cuenta de usario  con el siguiente email `james@notgoogleplus.com` sera ahora `<Account: james@notgoogleplus.com>`.

    def get_full_name(self):
        return ' '.join([self.first_name, self.last_name])

    def get_short_name(self):
        return self.first_name

`get_full_name()` y `get_short_name()` son convenciones de Django. No vamos a utilizar estos methods, pero sigue siendo una buena idea incluirlos para cumplir con las convenciones de Django.

## Haciendo una clase Manager para Account
Cuando sustituimos el modelo user por uno personalizado, se requiere definir una clase `Manager` que sobrescrito los métodos `create_user()` y `create_superuser()`.

Como el fichero `authentication/models.py` sigue abierto, añadiremos la siguiente clase antes dela clase `Account`:

    from django.contrib.auth.models import BaseUserManager


    class AccountManager(BaseUserManager):
        def create_user(self, email, password=None, **kwargs):
            if not email:
                raise ValueError('Users must have a valid email address.')

            if not kwargs.get('username'):
                raise ValueError('Users must have a valid username.')

            account = self.model(
                email=self.normalize_email(email), username=kwargs.get('username')
            )

            account.set_password(password)
            account.save()

            return account

        def create_superuser(self, email, password, **kwargs):
            account = self.create_user(email, password, **kwargs)

            account.is_admin = True
            account.save()

            return account

Como hicimos con `Account`, vamos a explicar esto linea por linea. Pero esta vez solo cubriremos la nueva información.

        if not email:
            raise ValueError('`User`s must have a valid email address.')

        if not kwargs.get('username'):
            raise ValueError('`User`s must have a valid username.')

Como se requiere que los usuarios tengan email y username, lanzaremos un error si alguno de estos dos atributos no se incluyen.

        account = self.model(
            email=self.normalize_email(email),username=kwargs.get('username')
        )

Desde el momento en que no definimos un atributo `model` en la clase `AccountManager`, `self.model` se refiere al atributo `model` de Base`UserManager`. Este es por defecto `settings.AUTH_USER_MODEL`, que cambiaremos en un momento para que señale a la clase `Account`.

        account = self.create_user(email, password, **kwargs)

        account.is_admin = True
        account.save()

Ya que escribir varias veces lo mismo es pesado. En vez de copiar todo el código del método `create_account` y pegarlo en `create_superuser`, simplemente dejamos que el método `create_user` haga la creación. Asi, que de lo único que debemos preocuparnos es de que el método `create_superuser` convierte en superusuario el objeto `Account`.

## Cambiar la propiedad AUTH_USER_MODEL de Django
A pesar de haber creado el modelo `Account`, el comando `python manage.py createsuperuser` (del cual hablaremos en breve) sigue creando objetos `User`. Esto es por que en este punto, Django sigue pensando que el modelo a utilizar para autentificar usuarios sigue siendo `User`.

Para poner las cosas claras y empezar a utilizar `Account` como modelo de autentificación, tendremos que actualizar `settings.AUTH_USER_MODEL`.

Abrimos `thinkster_django_angular_tutorial/settings.py` y añadimos al final del archivo la siguiente linea:

    AUTH_USER_MODEL = 'authentication.Account'

{x: auth_user_model_setting}
Cambia settings.AUTH_USER_MODEL para utilizar `Account` en vez de `User`

Esta linea le dice a Django que mire en la aplicación `authentication` y busque el modelo llamado `Account`.

## Instalando la primera app
En Django, debes declara explicitamente que aplicaciones serán utilizadas. Como no hemos añadido nuestra aplicación `authentication` a la lista de aplicaciones instaladas aun, vamos a hacerlo ahora.

Abre `thinkster_django_angular_boilerplate/settings.py` y añadiremos `'authnetication'`, a `INSTALLED_APPS` como sigue:

    INSTALLED_APPS = (
        …,
        'authentication',
    )

{x: install_authentication_app}
Instala la aplicación `'authentication'`

## Migrando nuestra primera aplicación
Cuando se lanzo la version Dajngo1.7, fue como Navidad en Septiembre. ¡Habían llegado las Migraciones!

Cualquiera que tenga conocimiento en Rails estará familiarizado con el concepto de migraciones. En breve, la migraciones toman el código SQL necesario para actualizar el esquema de la base de datos, así que no tendremos que hacerlo nosotros. Como ejemplo, imagina nuestro modelo `Account` que acabamos de crear. Este modelo debe ser guardado en la base de datos, pero nuestra base de datos, no tiene tabla para guardar objetos `Account` aun. ¿Como lo hacemos? !Creamos nuestra primera migración! La migración añadir las tablas a la base de datos y nos da la posibilidad de volver atrás en los cambios si cometemos alguna error.

Cuando estes preparado, genera la migración para la aplicación `authentication` y aplicada:
    
    $ python manage.py makemigrations
    Migrations for 'authentication':
        0001_initial.py:
            - Create model Account
    $ python manage.py migrate
    Operations to perform:
        Synchronize unmigrated apps: rest_framework
        Apply all migrations: admin, authentication, contenttypes, auth, sessions
    Synchronizing apps without migrations:
        Creating tables...
        Installing custom SQL...
        Installing indexes...
    Running migrations:
        Applying authentication.0001_initial... OK


{info}
A partir de ahora, no se incluirá la salida de los comandos de migración para abreviar.

{x: initial_migration}
Genera la migración para la aplicación authentication y aplicala.

## Hazte superusuario
Hablemos algo mas sobre el comando `python manage.py createsuperuser`.

Usuarios diferentes tienen niveles diferentes de acceso en cualquier aplicación. Algunos usuarios son administradores y pueden ir donde quieran, mientras que otros son usuarios normales y sus acciones deben estar limitadas. En Django, un super usuario es el nivel mas alto de acceso que puedes tener. Como queremos tener la posibilidad de trabajar sin restricciones en la aplicación, vamos a crear un super usuario. Esto es lo que hace el comando python manage.py createsuperuser.

Después de ejecutar el comando, Django te preguntará algunas informaciones y creará una cuanta `Account` con privilegios de superusuario. Pruébalo.

    $ python manage.py createsuperuser

{x: create_superuser}
Haz una cuenta de superusuario

## Punto de control
Para estar seguros de que todo esta bien configurado, toma una pequeña pausa y abre una consola de Django:

    $ python manage.py shell

Deberías ver una nueva linea de comandos con: `>>>`. Dentro de esta linea de comandos podemos obtener el objeto `Account` que acabamos de crear:

    >>> from authentication.models import Account
    >>> a = Account.objets.latest('create_at')

Si todo a ido bien, deberías ser capaz de acceder a los atributos del objeto `Account`:

    >>> a
    >>> a.email
    >>> a.username

{x: get_new_account}
Accede al objeto `Account` que acabamos de crear

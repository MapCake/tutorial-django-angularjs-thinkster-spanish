# Haciendo un modelos Post
En esta sección vamos ha hacer una nueva aplicación y crear un modelo `Post` similar a el estado en Facebook o un tweet en twitter. Una vez creado nuestro modelo iremos a la creación de un serializador para los `Post`s y después crearemos los puntos finales de nuestra API.

{video: making-post-model}

## Haciendo una aplicación posts
Lo primero es lo primero: crea la nueva aplicación llamada `posts`.

    $ python manage.py startapp posts

{x: django_app_posts}
Crea una nueva aplicaron llamada `posts`

Recuerda: cada vez que creas una nueva aplicación debes añadirla a la configuración `INSTALLED_APPS`. Abre `thinkster_django_angular_boilerplate/settings.py` y modificado de la siguiente manera:

    INSTALLED_APPS = (
        # ...
        'posts',
    )

## Haciendo el modelo Post
Después de crear la aplicación `posts` en Django crea un nuevo fichero llamado `posts/models.py`. Abre este fichero y añade lo siguiente:

    from django.db import models

    from authentication.models import Account


    class Post(models.Model):
        author = models.ForeignKey(Account)
        content = models.TextField()

        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

        def __unicode__(self):
            return '{0}'.format(self.content)

{x: django_model_post}
Crea un nuevo modelo llamado `Post` en `posts/models.py

Nuestro metodo de ir linea por linea en el código esta funcionando por el momento. ¿Por qué cambiarlo si funciona? Continuemos.

    author = models.ForeignKey(Account)

Ya que `Account` puede tener muchos objetos `Post`, necesitamos definir una relación muchos a uno.

Para hacer esto en Django utilizamos el campo `ForeignKey` para asociar cada `Post` con un `Account`.

Django es lo suficientemente inteligente para saber que la clave foránea que definamos debe ser reversible. Es decir, dado un `Account`, deberías ser capaz de acceder a los `Post` del usuario. En Django estos objetos `Post` son accesibles a través de `Account.post_set` (no a través de `Account.posts`).

Ahora que existe el modelo, no te olvides de migrar.

    $ python manage.py makemigrations
    $ python manage.py migrate

{x: django_model_post_migrate}
Haz las migraciones para `Post` y aplícalas

## Serializando el modelo Post
Crea un nuevo fichero en `posts/` llamado `serializers.py` y añade lo siguiente:

    from rest_framework import serializers

    from authentication.serializers import Account
    from posts.models import Post


    class PostSerializer(serializers.ModelSerializer):
        author = AccountSerializer(read_only=True, required=False)

        class Meta:
            model = Post

        fields = ('id', 'author', 'content', 'created_at', 'updated_at')
        read_only_fields = ('id', 'created_at', 'updated_at')

        def get_validation_exclusions(self, *args, **kwargs):
            exclusions = super(PostSerializer, self).get_validation_exclusions()

        return exclusions + ['author']

{x: django_serializer_postserializer}
Crea un nuevo serializador llamado `PostSerializer` en `posts/serilizers.py`

No hay muchas cosas nuevas en este código, pero hay una linea en particular que me gustaria revisar.

    author = AccountSerializer(read_only=True, required=False)

Habíamos definido explicitamente un numero de campos en nuestro `AccountSerializer` de esta manera, pero esta definición es un poco diferente.

Estamos serializando un objeto `Post`, queremos incluir toda la información del autor. Con Django REST Framework, esto se conoce como una relación anidada. Básicamente estamos serializando el `Account` relacionado con el `Post` y lo incluimos en nuestro JSON.

Pasamos el argumento `read_only=True` porque no deberiamos poder actualizar el objeto `Account` con `PostSerializer`. También seleccionamos `required=False` ya que seleccionaremos el autor de forma automática.

    def get_validation_exclusions(self, *args, **kwargs):
        exclusions = super(PostSerializer, self).get_validation_exclusions()

        return exclusions + ['author']

Por la misma razón que utilizamos `required=False`, también debemos añadir `author` a la lista de validaciones que queremos evitar.

## Creando vistas de la API de los objetos Post
El siguiente paso en la creación de objetos `Post`, es añadir un punto final en el API que gestionará las acciones a realizar en el modelo `Post` como por ejemplo, crear o actualizar.

Reemplaza el contenido de `posts/views.py` por el siguiente:

    from rest_framework import permissions, viewsets
    from rest_framework.response import Response

    from posts.models import Post
    from posts.permissions import IsAuthorOfPost
    from posts.serializers import PostSerializer


    class PostViewSet(viewsets.ModelViewSet):
        queryset = Post.objects.order_by('-created_at')
        serializer_class = PostSerializer

        def get_permissions(self):
            if self.request.method in permissions.SAFE_METHODS:
                return (permissions.AllowAny(),)
            return (permissions.IsAuthenticated(), IsAuthorOfPost(),)

    def perform_create(self, serializer):
        instance = serializer.save(author=self.request.user)

        return super(PostViewSet, self).perform_create(serializer)



    class AccountPostsViewSet(viewsets.ViewSet):
        queryset = Post.objects.select_related('author').all()
        serializer_class = PostSerializer

        def list(self, request, account_username=None):
            queryset = self.queryset.filter(author__username=account_username)
            serializer = self.serializer_class(queryset, many=True)

            return Response(serializer.data)

{x: django_viewset_post}
Crea un set de vistas `PostViewSet`

{x: django_viewset_account_post}
Crea un conjunto de vistas `AccountPostsViewSet`

¿No te parecen similares las vistas? No son tan diferentes como las que hablamos creado para los objetos `User`.

    def perform_create(self, serializer):
        instance = serializer.save(author=self.request.user)

        return super(PostViewSet, self).perform_create(serializer)


`perform_create` se ejecuta antes de que el modelo de esta vista sea salvado.

Cuando se crea un objeto `Post`, debe tener asociado un autor. Haciendo el tipo de autor en su propio nombre de usuario o id cuando creamos un post al sitio será un mal ejercicio, así que haremos esta asociación con el gancho `perform_create`. Simplemente grabamos el usuario asociado con esta petición y lo hacemos el autor del `Post`.

    def get_permissions(self):
        if self.request.method in permissions.SAFE_METHODS:
            return (permissions.AllowAny(),)
        return (permissions.IsAuthenticated(), IsAuthorOfPost(),)


De forma similar a los permisos que hemos utilizado en el conjunto de vistas de `Account`, métodos HTTP peligrosos requieren que el usuario este autenticado y autorizado a hacer cambios en ese `Post`. Crearemos los permisos de `IsAuthorOfPost` en breve. Si el metodo HTTP es seguro, dejamos a todos acceder a la vista.

    class AccountPostsViewSet(viewsets.ViewSet):

Este conjunto de vistas se utilizara para listar los post asociados un `Account` especifico

    queryset = Post.objects.select_related('author').all()

Aqui filtramos nuestro queryset basándonos en el nombre de usuario. El argumento `account_username` será dado por el router que crearemos en unos minutos.

## Creando los permisos de IsAuthorOfPost
Crea el fichero `permissions.py` en el directorio `posts/` con el siguiente contenido:

    from rest_framework import permissions


    class IsAuthorOfPost(permissions.BasePermission):
        def has_object_permission(self, request, view, post):
            if request.user:
                return post.author == request.user
            return False

{x: django_permission_isauthenticatedandownsobject}
Define los nuevos permisos llamados `IsAuthorOfPost` in `posts/permissions.py`

Saltaremos la explicación de esto. Estos permisos son idénticos a los que hablamos hecho antes.

## Creando un punto final para posts en la API
Con las vistas creadas, es hora de añadir el punto de acceso a nuestra API.

Abre `thinkster_django_angular_boilerplate/urls.py` y añade el siguiente import:

    from posts.views import AccountPostsViewSet, PostViewSet

Ahora añade estas lineas, justo antes de `urlpatterns = patterns(`:

    router.register(r'posts', PostViewSet)

    accounts_router = routers.NestedSimpleRouter(
        router, r'accounts', lookup='account'
    )
    accounts_router.register(r'posts', AccountPostsViewSet)

`accounts_router` proporciona la necesidad de anidar las rutas para acceder a los posts de un `Account` especifico. También debes añadir `accounts_router` a `urlpatterns` de la forma siguiente:

    urlpatterns = patterns(
        # ...

        url(r'^api/v1/', include(router.urls)),
        url(r'^api/v1/', include(accounts_router.urls)),

        # ...
    )

{x: django_url_postviewset}
Crea el punto de acceso a la API para el conjunto de vistas `PostViewSet`

{x: django_url_accountpostsviewset}
Crea el punto de acceso a la API para el conjunto de vistas `AccountPostsViewSet`

## Punto de control
En este momento, siente libre de abrir un terminal con `python manage.py shell` y juega creando y serializando objetos `Post`.

    >>> from authentication.models import Account
    >>> from posts.models import Post
    >>> from posts.serializers import PostSerializer
    >>> account = Account.objects.latest('created_at')
    >>> post = Post.objects.create(author=account, content='I promise this is not Google Plus!')
    >>> serialized_post = PostSerializer(post)
    >>> serialized_post.data

Juega con el modelo `Post` y el serializador `PostSerializer` en el terminal de Django

Validaremos que las vistas funcionan al final de la siguiente sección.
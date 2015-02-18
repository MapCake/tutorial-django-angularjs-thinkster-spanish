## Serializando los modelos User y UserProfile
La aplicación AngularJS que estamos construyendo, hará peticiones AJAX al servidor para obtener los datos que debería mostrar. Antes de poder enviar estos datos al cliente, necesitamos darles un formato de forma que el cliente puede interpretarlos; en este caso hemos elegido JSON. El proceso de transformar los modelos Django en JSON se llama serialización y es de lo que vamos a hablar a continuación.

{video: serialize-account-model}

Como el modelo que queremos serializar se llama `Account`, el serializador que vamos a crear se llamará `AccountSerializer`.

## Django REST Framework
Como parte del proyecto modelo que hemos clonados antes, hemos incluido un proyecto llamado Django REST Framework. Django REST Framework es un conjunto de herramientas que propone un numero de características comunes a la gran mayoría de plantaciones web, incluyendo los serializadores. Vamos a utilizar estas características durante el tutorial para ahorrarnos tiempo y frustraciones. Nuestro primer vistazo a Django REST Framework empieza aquí.

## AccountSerializer
Antes de escribir nuestro serializador, creemos un fichero `serializers.py` dentro de nuestra aplicación `authentication`:

    $ touch authentication/serializers.py

{x: create_serializers_module}
Crea el fichero `serializers.py` dentro de la aplación `authentication`

Abre `authentication/serializers.py` y añade el siguiente código e `imports`:

    from django.contrib.auth import update_session_auth_hash

    from rest_framework import serializers

    from authentication.models import Account

    
    class AccountSerializer(serializers.ModelSerializer):
        password = serializers.CharField(write_only=True, required=False)
        confirm_password = serializers.CharField(write_only=True, required=False)

        class Meta:
            model = Account
            fields = ('id', 'email', 'username', 'created_at', 'updated_at',
                      'first_name', 'last_name', 'tagline', 'password',
                      'confirm_password',)
            read_only_fields = ('created_at', 'updated_at',)

            def create(self, validated_data):
                return Account.objects.create(**validated_data)

            def update(self, instance, validated_data):
                instance.username = validated_data.get('username', instance.username)
                instance.tagline = validated_data.get('tagline', instance.tagline)

                instance.save()

                password = validated_data.get('password', None)
                confirm_password = validated_data.get('confirm_password', None)

                if password and confirm_password and password == confirm_password:
                    instance.set_password(password)
                    instance.save()

                update_session_auth_hash(self.context.get('request'), instance)

                return instance

{x: create_account_serializer}
Crea un serializador llamado Account Serializer en authentication/serializers.py

{info}
A partir de ahora, vamos a declarar los imports que utilizaremos en cada retazo de código. Esto también debe estar presente en el fichero. Así que, no debes añadirlo por segunda vez.

Vamos a ver el código en detalle

    password = serializers.CharField(write_only=True, required=False)
    confirm_password = serializers.CharField(write_only=True, required=False)

En vez de incluir `password` en la tupla `fields`, de la cual hablaremos después, definimos explícitamente el campo en la parte superior de la clase `AccountSerializer`. La razón de hacer esto es poder pasar el argumento `required=False`. Cada campo en `fields` es obligatorio, pero no queremos actualizar el campo password hasta que no se especifique uno nuevo.

`confirm_pssword` es similar a `password` y se utiliza solo para verificar que el usuario no comete errores de escritura.

Observa también el uso del argumento `write_only=True`. El password de usuario, también esta oculto, no debe ser visible por el cliente en la respuesta AJAX.

    class Meta:

La subclase `Meta` define metadatos del serializador para operar. Hemos definido algunos atributos comunes de la clase Meta.

    model = Account

Ya que el serializador hereda de `serilizers.ModelSerilaizer`, debería tiener sentido que le digamos al serializador que modelo utilizar. Al especificar el modelo garantizamos que solo los atributos de ese modelo o los campos creados explícitamente pueden serializarse. Cubriremos la serializacion de atributos de un modelo ahora y la de campos creados explícitamente en breve.

    fields = ('id', 'email', 'username', 'created_at', 'updated_at',
              'first_name', 'last_name', 'tagline', 'password',
              'confirm_password',)

El atributo `fields` de la clase `Meta` es donde especificaremos que atributos del modelo `Account` deben serializarse. Debemos tener cuidado al especificar los campos a serializar, ya que algunos campos, como `is_superuser`, no debería estar disponible para el cliente por razones de seguridad.

    read_only_fields = ('created_at', 'updated_at',)

Si lo recuerdas, cuando creamos el modelo `Account`, hicimos que los campos `created_at` y `updated_at` se actualizaran de forma automática. Debido a esta propiedad, cuando los añadimos a la lista deben ser campos de solo lectura.

    def create(self, validated_data):
        # ...

    def update(self, instance, validated_data):
        # ...

Anteriormente habíamos mencionado que a veces querríamos transformar objetos JSON en objetos Python. Esto se llama deserialización y lo gestionan los métodos `.create()` y `.update()`. Cuando creemos un objeto nuevo, como lo es el objeto `Account`, se utilizará `.create()`. Cuando mas tarde actualicemos ese objeto `Account`, se utilizara el método `.update()`.

    instance.username = validated_data.get('username', instance.username)
    instance.tagline = validated_data.get('tagline', instance.tagline)

Dejaremos que el usuario pueda actualizar los atributos de nombre de usuario y su lema por el momento. Si estos elementos están presentes en el diccionario del array, utilizaremos el nuevo valor. Si no, se utilizara el valor actual del objeto `instance`. Aquí, `instance` es de tipo `Account`.

    password = validated_data.get('password', None)
    confirm_password = validated_data.get('confirm_password', None)

    if password and confirm_password and password == confirm_password:
        instance.set_password(password)
        instance.save()

Antes de actualizar el password de usuario necesitamos confirmar que el usuario a proporcionado valores para ambos campos `password` y `password_confirmation`. Así que verificamos que el contenido de ambos campos idéntico.

Después de haber verificado esto, debemos actualizar el password y utilizamos `Account.set_password()` para actualizar. `Account.ser_password()` se encarga de almacenar el password de forma segura. Es importante también saber que debemos guardar de forma explicita el modelo después de actualizar el password.

{info}
Esta es una impletencion simple de como validar un password. No te recomendamos utilizar este forma en un sistema de producción, pero para nuestro propósito esta bien.

    update_session_auth_hash(self.context.get('request'), instance)

Cuando se actualiza el password de usuario, la autenticación de usuario debe estar enmascarada de forma explicita. Si no lo hacemos, el usuario no sera autentificado en su siguiente petición al servidor y tendrá que volver a conectarse otra vez.

## Punto de control
Por el momento no deberíamos tener problemas al visualizar la serializacion en JSON de un objeto `Account`. Abre el terminal de Django ejecutando `pyhton manage.py shell` e intenta escribir los comandos siguientes:

    >>> from authentication.models import Account
    >>> from authentication.serializers import AccountSerializer
    >>> account = Account.objects.latest('created_at')
    >>> serialized_account = AccountSerializer(account)
    >>> serialized_account.data.get('email')
    >>> serialized_account.data.get('username')

{x: checkpoint_auth_serializers}
Asegurate que tu serializador AccountSerializer funciona
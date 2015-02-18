# Aprendiendo Django y AngularJS

{intro-video: django-intro}

En este tutorial vamos a crear un clon simplificado de Google+ con Django y AngularJS. Lo Llamaremos “Not Google Plus”.

Antes de empezar a construir aplicaciones web modernas y ricas con Django y AngularJS, vamos a tomar un momento en explorar la motivación detrás de este tutorial y como puedes obtener el máximo partido de este.

## ¿Cual es el objetivo de este tutorial?
En thinkster, nos esforzamos por crear gran valor, en contenido de profundidad, manteniendo una barrera de entrada baja. Publicamos este contenido de forma gratuita, con la esperanza de que puedas encontrarlo excitante e informativo.

Cada tutorial que publicamos tiene un objetivo especifico. En este, el objetivo es dar una breve introducción de Django y AngularJS comunicándose juntos y como estas tecnologías pueden combinarse para construir aplicaciones web increíbles. Ademas, ponemos énfasis en construir buenos hábitos de ingeniería. Esto incluye la consideración de ventajas y desventajas derivadas de las decisiones de arquitectura, para mantener la alta calidad del código a través de tus proyectos. Aunque estas cosas no parezcan muy divertidas, son la clave para convertirse en un buen desarrollados de software.

## ¿Para quien es este tutorial?
Todos los autores deben responder a esta complicada pregunta. Nuestro objetivo es hacer este tutorial util para desarrolladores novatos y expertos.

Para aquellos que empezáis con vuestra carrera de desarrolladores, hemos intentado  ser exhaustivos y lógicos en nuestras explicaciones en la medida de lo posible, intentando que el texto sea fluido; también tratamos de evitar saltos intuitivos aunque tengan sentido.

Para aquellos que ya tienen algo de experiencia, y quizás esta simplemente interesados en aprender mas sobre Django o AngularJS, sabemos que no necesitas que te expliquemos las bases. Uno de nuestros objetivos al escribir este tutorial era hacerlo lo mas sencillo posible. Esto hace que con tus conocimientos actuales puedas leerlo mas rápido e identificar los conceptos con los que no estas familiarizado para que puedas asimilarlos rápidamente y continuar.

Queremos hacer este tutorial accesible a cualquiera con el interés suficiente para tomar el tiempo necesario para aprender y entender los conceptos presentados.

## Un breve interludio acerca del formato
A lo largo de este tutorial, nos esforzamos por mantener un formato coherente. En esta sección detallamos como debe ser el formato y que significa.

* Cuando mostremos un nuevo retazo de código, lo presentaremos por completo y después lo recorreremos linea por linea si es necesario para cubrir los nuevos conceptos.
* Los nombre de variables y ficheros aparecen con el formato especial siguiente:`thinkster_django_angular/settings.py`.
* Los retazos de código largos aparecen en sus propias lineas:

        def is_this_google_plus():
           return False

* Los comandos del terminal también aparecen en sus propias lineas, con el prefijo `$`:

        $ python manage.py runserver

* Si no se especifica de otra manera, debes asumir que todos los comandos del terminal deben ejecutarse desde el directorio principal de tu proyecto.

## Unas palabras sobre el estilo del código
Cuando es posible, optamos por seguir la guía de estilo creado por las comunidades de Django y Angular.

Para Django, seguimos el estilo [PEP8](http://legacy.python.org/dev/peps/pep-0008/) estrictamente e intentamos seguir el [estilo de Codigo de Django] (https://docs.djangoproject.com/en/1.7/internals/contributing/writing-code/coding-style/).

Para Angular JS, hemos adoptado la [guía de estilo de John Papa para AngularJS](https://github.com/johnpapa/angularjs-styleguide). También utilizamos la [guía de estilo de Google para JavaScript] (https://google-styleguide.googlecode.com/svn/trunk/javascriptguide.xml) cuando es lógico utilizarlo.

## Una humilde petición de retroalimentación
A riesgo de sonar a cliché, no tendríamos una razón para hacer este tutorial si no fuera por ti. Porque creemos que tu éxito es nuestro éxito, te invitamos a ponerte en contacto con nosotros con cualquier idea que tengas sobre el tutorial. Puedes comunicarte con nosotros (en inglés) a través de la caja en la esquina inferior derecha de la pantalla, a través de Twitter en [@jamesbrwr](http://twitter.com/jamesbrwr) o en @GoThinkster](http://twitter.com/gothinkster), o enviando un correo electrónico [support@thinkster.io](mailto:support@thinkster.io).

Damos la bienvenida a la crítica abierta y aceptamos alagos si crees que es merecido. Estamos interesados en saber lo que te gusta, lo que no te gusta, sobre los temas que quiere saber más, y cualquier cosa que consideres relevante.

Si estás demasiado ocupado para llegar a nosotros, está bien. Sabemos que el aprendizaje requiere un montón de trabajo. Si, por el contrario, quieres ayudarnos a construir algo increíble, estamos a la espera de tu correo.

Si tus comentarios son sobre esta traduccion al español, no dudes en escribir en español a [damarmo@gmail.com](mailto:damarmo@gmail.com).

## Unas últimas palabras antes de empezar
Es nuestra experiencia que los desarrolladores que obtienen el mayor beneficio de nuestros tutoriales son los que toman un enfoque activo de su aprendizaje.

Te recomendamos **energicamente** que escribas el código tu mismo. Al copiar y pegar el código, no interactúas con él y esta interacción es a su vez, lo que te hace un mejor desarrollador.

Además de escribir el código por ti mismo, no tengas miedo de ensuciarte las manos; saltar y jugar, romper cosas y construir características que faltan. Si encuentras un error, explora y averigua lo que lo está causando. Estos son los obstáculos que nosotros como ingenieros deben abordar varias veces al día, por lo que hemos aprendido a abrazar estas exploraciones como la mejor fuente de aprendizaje.

Vamos a construir algo de software.

# Configurando el entorno de trabajo

{video: setup-environment}

La aplicación que vamos a construir requiere una gran cantidad de trabajo repetitivo y no trivial. En vez de pasar tiempo configurando el entorno, que no es el propósito de este tutorial, hemos creado una plantilla del proyecto para empezar.

Puedes encontrar la plantilla del proyecto en Github en [brwr/thinkster-django-angular-boilerplate](https://github.com/brwr/thinkster-django-angular-boilerplate). El repositorio incluye una lista de comandos que necesitas ejecutar para tener todo en marcha.

Nota
Si estas interesado en un apéndice detallado en configurar tu entorno de trabajo háznoslo saber en el Twitter http://twitter.com/jamesbrwr.

Ve al repositorio y sigue las instrucciones ahora.

{x: set_up_envrionment}
Sigue las instrucciones para configurar tu entorno de trabajo.

## Punto de Control

Si todo ha ido bien, ejecutando el servidor con el comando `python manage.py runserver`, deberías poder visitar el enlace `http://localhost:8000/` en un navegador . La pagina debería estar en blanco excepto una barra de navegación en la parte superior. El enlace en la barra de navegación no hace nada por el momento.

{x: checkpoint_environment}
Asegurate que tu entorno de trabajo esta funcionando, visitando el enlace `http://localhost:8000/`

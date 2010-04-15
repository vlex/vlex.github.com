---
layout: post
title: Cucumber, Selenium, Webrat, Integrity\: herramientas para testar páginas web
author: claudiob
---

Para garantizar la calidad de cualquier página web, es útil tener una **suite de test** que simulen el comportamiento de los usuarios. U suite de test averigua de forma automática que la página web cumpla con sus requisitos. 

## Escribir historias

El primer paso para escribir una suite de test es **escribir historias**.
Una historia es una sentencia que detalla un requerimiento de la página web. Una historia común a muchas páginas web, por ejemplo, es la siguiente:

- Dado que me registré como usuario

- Cuando voy a la página de login y relleno el formulario con mi nombre de usuario y contraseña

- Entonces tengo que ver un mensaje de bienvenida

En general, las historias pueden ser más o menos complejas, pero todas se pueden escribir siguiendo la estructura anterior, es decir:

{% highlight cucumber %}
Dado que se cumplen unas condiciones
Cuando hago unas acciones
Entonces tengo que ver unos resultados
{% endhighlight %}

Historias escritas con esta estructura son simples de leer también para los no informáticos. De hecho, los mismos clientes pueden escribir estas historias, así como los responsables de otros departamentos (comercial, producto, marketing, atención al cliente).

Más importante aún es que una herramienta llamada [Cucumber](http://wiki.github.com/aslakhellesoy/cucumber/) es capaz de *interpretar* historias con esta estructura y ejecutarlas de manera automática.

## De las historias a los pasos

Cuando Cucumber interpreta una historia, lee todas las acciones que la componen y intenta ejecutarlas en el mismo orden. Si Cucumber *no sabe* como ejecutar una acción, pide al programador una explicación. Por ejemplo, después de leer la historia anterior, Cucumber devuelve un resultado de este estilo:

- Explícame como ejecutar la acción "me registré como usuario"

- Explícame como ejecutar la acción "voy a la página de login"

- Explícame como ejecutar la acción "relleno el formulario con mi nombre de usuario y contraseña"

- Explícame como averiguar la condición "tengo que ver un mensaje de bienvenida"

En sí, el trabajo de Cucumber es muy sencillo: leer las historias, averiguar si los *pasos* están definidos, ejecutarlos si lo están o pedir una *definición* si no lo están.

El trabajo del programador de tests es *definir los pasos* que componen las historias.

### Definir pasos de manera interna

Una manera para definir pasos es trabajar de forma interna a la página web.
Por ejemplo, un paso como "Cuando creo un usuario 'guest'" se puede definir como un comando SQL:

{% highlight sql %}
INSERT INTO Users (name) VALUES "guest"
{% endhighlight %}

o también como un comando Ruby si la web es una aplicación Rails:

{% highlight ruby %}
`Usuario.create :name => 'guest'`
{% endhighlight %}

Esta manera de trabajar es muy rápida ya que permite agrupar acciones en pocas lineas (por ejemplo, crear 100 usuarios con un solo comando). 

### Definir pasos de manera externa

Otra manera muy diferente para definir pasos es trabajar de forma externa a la página web. Cada paso se define como la secuencia de acciones que un usuario tendría que ejecutar en un browser web para completar ese paso.
Por ejemplo, el paso "Cuando creo un usuario 'guest'" se traduce en:

> Visito la página web
>
> Sigo el enlace "Crea un nuevo usuario"
>
> Relleno el campo "Nombre" con "guest"
>
> Aprieto el botón "Crear"

Los pasos definidos de forma externa son *agnósticos* con el código fuente. Aunque la base de datos cambie de MySQL a Oracle, o la aplicación cambie de Ruby a PHP, los pasos siguen válidos.

Trabajar de forma externa equivale a ponerse en la piel de un usuario de la página web que ejecuta una serie de acciones en el browser (visitar, rellenar, apretar) y esperan unos resultados, sin conocer nada de la implementación ni del código fuente.

### Grabar secuencias de pasos de navegación

Una herramienta muy útil para definir pasos de forma externa es [Selenium IDE](http://seleniumhq.org/projects/ide/), un plugin para Mozilla Firefox que graba secuencias de acciones de navegación.
Por ejemplo, activando el plugin en Firefox y ejecutando a mano las acciones necesarias para crear un usuario 'guest', Selenium IDE devuelve una secuencia de comandos como:

{% highlight ruby %}
page.open "/"
page.click "link=Crea un nuevo usuario"
page.wait_for_page_to_load "30000"
page.type "username", "guest"
page.click "//input[@value='Crear']"
page.wait_for_page_to_load "30000"
{% endhighlight %}

La herramienta complementaria a Selenium IDE es [Selenium Web Driver](http://code.google.com/p/selenium/wiki/GettingStarted). Mientras la primera sirve para *grabar* secuencia de acciones, la segunda permite escribir programas que *ejecutan* estas acciones de forma automática. Selenium Web Driver se integra con muchos lenguajes de programación, entre ellos Ruby gracias a la  gem [selenium client](http://selenium-client.rubyforge.org/). 

## Escribir tests con Cucumber + Selenium

Cucumber y Selenium son los dos componentes de base para escribir una suite de test externos:

- para cada historia, Cucumber interpreta la estructura "Dado que.. Cuando.. Entonces.." y devuelve al programador de tests la lista de pasos que todavía no están definidos;

- el programador ejecuta cada uno de estos pasos en Firefox y obtiene de Selenium IDE la lista de acciones correspondientes;

- el programador copia/pega estas acciones en las definiciones de los pasos y volver a ejecutar las historia contra Cucumber;

- Cucumber encuentra que la historia está completamente definida e invoca Selenium Web Driver para que vuelva a ejecutar las acciones; si el resultado final es el esperado, el test *pasa*, de forma contraria *falla*.

### Definir pasos con una sintaxis más natural

Selenium IDE es muy útil para empezar, ya que permite ejecutar los tests a mano en Firefox y luego simplemente copiar y pegar las secuencias de acciones del browser a la suite de test.

Cuando el número de tests crece, todavía, este procedimiento manual puede resultar lento, y es más inmediato definir pasos directamente en la suite de test. La sintáxis que utiliza Selenium para esto no es totalmente intuitiva. Como en el ejemplo anterior, las acciones correspondientes a crear un usuario "guest" se definen como:

{% highlight ruby %}
page.open "/"
page.click "link=Crea un nuevo usuario"
page.wait_for_page_to_load "30000"
page.type "username", "guest"
page.click "//input[@value='Crear']"
page.wait_for_page_to_load "30000"
{% endhighlight %}

En cambio, una sintaxis como la siguiente es más clara y natural:

{% highlight ruby %}
visit "/"
click_link "Crea un nuevo usuario"
fill_in "username", :with => "guest"
click_button "Crear"
{% endhighlight %}

Esta es la sintaxis que utiliza otra herramienta para escribir tests externos, llamada [Webrat](http://wiki.github.com/brynary/webrat/). 
Webrat tiene un [conjunto reducido de acciones](http://cheat.errtheblog.com/s/webrat/) que se pueden ejecutar en un browser, como `visit`, `click_link`, `fill_in`, `click_button`. 
Webrat se preocupa de encontrar los elementos en la página web (sea por texto, expresión regular, título o ID del objeto) y de explicar a Selenium Web Driver como ejecutar cada una de las acciones.

## Escribir tests con Cucumber + Webrat + Selenium

Las tres herramientas para escribir y ejecutar una suite de test se reparten el trabajo de esta forma:

- Cucumber interpreta las historias, controla que todos los pasos tengan una definición, ejecuta los pasos y enseña el resultado del test;

- Webrat traduce cada paso en acciones comprensibles por Selenium y reporta los resultados a Cucumber;

- Selenium abre un browser web y ejecuta las acciones de navegación especificadas por Webrat.

La ventaja de utilizar Webrat como capa entre Cucumber y Selenium es que, si necesario, Webrat también puede funcionar *sin Selenium*, es decir, ejecutar las acciones de navegación directamente, a través de llamadas HTTP en lugar de esperar que Selenium abra un browser y reproduzca todas las acciones.

La modalidad Webrat sin Selenium es más limitada que la modalidad Webrat::Selenium (por ejemplo, no puede ejecutar Javascript o confirmar mensajes de alerta) y menos intuitiva (los errores no se pueden capturar como pantallas de un browser) pero es mucho más rápida, lo que puede ser útil cuando el numero de tests es muy grande.

### Averiguar condiciones esperadas

El paso más importante de cada historia es el "Entonces..", donde se describe el resultado esperado después de una secuencia de acciones. 

La sintaxis que utiliza Webrat para describir este paso es la misma que utiliza RSpec. Por ejemplo, "Entonces tengo que ver 'Bienvenido guest'" se traduce en Webrat/RSpec como:

{% highlight ruby %}
response.should contain "Bienvenido Usuario"
{% endhighlight %}

### Cuando una historia falla

Cuando un test no pasa, Selenium permite guardar la imagen del browser en el momento del fallo, para poder comprobar la origen del fallo. En general, se pueden clasificar los fallos en dos tipos:

- @bug: cuando la página web está diseñada para ofrecer una función, pero no lo hace

- @wish: cuando el test espera que la página ofrezca una función que no se ha planteado como requerimiento fundamental de la misma

En el caso de un bug, el paso siguiente es señalarlo a los programadores para que lo resuelvan. En el caso de un wish, el paso siguiente es decidir si es útil implementarlo o se puede dejar como anotación.

En ambos casos, es útil *etiquetar* la historia como @bug o @wish, para que conste nota de que esa historia ha fallado y se ha reconocido el porque.

## Integración continua

Una vez escrita la suite de test, el último pasa es ejecutarla, arreglar los fallos, y ejecutarla otra vez, hasta que todas las historias *pasen*. Esto permite saber que **en este momento** la página web cumple todas las expectativas.

Claramente, ejecutar la suite de test sólo una vez no es suficiente, ya que una página web cambia continuamente. Idealmente hay que ejecutar los tests cada vez que algo cambia en el código de la página web.

Este es el objetivo de una herramienta de *integración continua* llamado [Integrity](http://integrityapp.com). Integrity es una aplicación web que se preocupa de ejecutar toda la suite de test cada vez que recibe un mensaje HTTP POST. 
La manera más natural de trabajar es instalar Integrity en un servidor web y añadir un post-receive hook al repositorio que contiene el código de la página web (por ejemplo [GitHub](http://help.github.com/post-receive-hooks/)).
De esta forma, cualquier cambio en el código hace que Integrity ejecute todos los tests otra vez.

Integrity enseña los resultados de cada test mediante una interfaz web, pero también admite notificación por e-mail, por lo que es posible recibir un mensaje cada vez que los tests fallan, y reaccionar inmediatamente.



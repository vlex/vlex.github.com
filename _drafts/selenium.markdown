---
layout: post
title: Testing, or how I fit a Cucumber in 200k lines of Ruby code
author: claudiob
---

Desde hace dos meses estoy escribiendo una suite de test para vLex.com que intenta simular el comportamiento de los usuarios de vLex para averiguar que todas las funciones disponibles en la página web hagan lo que tienen que hacer.

La web de vLex es muy grande y tiene muchos usuarios y muchos clientes de pago. El código de la web es muy grande, 200,000 lineas de código.

Escribir historias (Cucumber)
=============================

Lo primero que hice fue entrevistar los comerciales de vLex que hacen las presentaciones a los clientes, para ver cuales son las funciones que se enseñan a los usuarios, las más delicadas, porque si no funcionan, pierdes la venta.
También Angel me ha dicho unas cuantas, y he navegado por la web para ver más.

El resultado de este trabajo inicial ha sido de escribir una serie de **historias**. Cada historia es una frase con una estructura muy sencilla:

- *Dado* que se cumplen unas condiciones

- *Cuando* hago una acción

- *Entonces* tengo que ver un resultado

Por ejemplo, una historia clave para los comerciales es la siguiente:

- Dado que estoy registrado como cliente de pago de vLex España

- Cuando busco "código civil" en vLex

- Entonces tengo que ver un enlace al Código Civil Español

La gracia de escribir historias con esta estructura es que no sólo las entienden los humanos, sino que también los ordenadores, gracias a una herramienta llamada [Cucumber](http://wiki.github.com/aslakhellesoy/cucumber/), capaz de leer historias escrita en lenguaje natural y ejecutarlas como tests automatizados. 

Tests internos o externos
=========================

Para cada historia, Cucumber intenta completar todos los pasos, es decir, ejecutar los "Dado que" y los "Cuando" y al final averiguar que se cumplan las condiciones "Entonces".
Cucumber sabe leer la *estructura* de cada historia, pero al principio no sabe como ejecutar los pasos. Por ejemplo, si una historia empieza con:

- Dado que estoy registrado como cliente de pago de vLex España

Cucumber entiende que tiene que ejecutar una acción, pero no sabe como.
Hay dos manera muy diferentes de explicar a Cucumber como ejecutar acciones.

La primera manera es **de forma interna**, es decir, hacer que Cucumber tenga *acceso directo* al código fuente de vLex.com y pueda relacionarse directamente con la base de datos o con las estructuras de datos. Por ejemplo, se podría decir a Cucumber que el paso:
- Dado que estoy registrado como cliente de pago de vLex España
se traduzca en una query SQL como:

> `SELECT * FROM usuarios WHERE country = 'Spain' LIMIT 1`

o en un comando Ruby como:

> `Usuario.find :first, :conditions => {:country => 'Spain'}`

La gracia de trabajar de forma interna es que es muy rápido. Por ejemplo si quiero hacer un test para crear 100 usuarios, lo puedo hacer con una linea de código, saltando las validaciones que ven los usuarios cuando se registran a través de la página web.

La segunda manera es **de forma externa**, es decir, que Cucumber no sepa nada del código fuente de vLex.com y ejecute cada paso exactamente como si fuera un usuario a través de un browser web. En este caso, el paso:

> Dado que estoy registrado como cliente de pago de vLex España

se traduce en una serie de acciones que efectuar en el browser:
> Visita vLex.com
>
> Sigue el enlace "Acceso clientes"
>
> Rellena "Usuario" y "Password" con los datos de un cliente de pago de vLex España
>
> Aprieta el botón "Acceder"

La gracia de trabajar de forma externa es que simula **todo** lo que haría un usuario de verdad, sin atajos, sin *fingir* acciones, pudiendo así averiguar un mayor número de errores: errores de lógica pero también de interfaz, de validación y de navegación. Otra ventaja es poder separar totalmente el código de vLex.com de la suite de tests: los tests no necesitan saber nada de como se ha *implementado* un paso como el login, simplemente tienen que poder hacerlo.

Por estas razones, la suite de test para vLex.com trabaja de forma externa.

Escribir pasos externos (Selenium)
==================================

Trabajar de forma externa quiere decir describir cada paso como una serie de acciones que hay que ejecutar en un browser web. Una herramienta muy útil para este objetivo es [Selenium IDE](http://seleniumhq.org/projects/ide/).

Selenium IDE es un plugin para Mozilla Firefox que graba secuencias de acciones de navegación. Por ejemplo si activo Selenium IDE en Firefox y luego abro la home page de vLex, hago click en enlace "Acceso clientes", relleno con mi usuario y contraseña y aprieto "acceder" para entrar en vLex, Selenium IDE graba esta secuencia como:
> `page.open "/"`
>
> `page.click "link=Acceso clientes"`
>
> `page.wait_for_page_to_load "30000"`
>
> `page.type "username", "mi_usuario"`
>
> `page.type "password", "mi_contraseña"`
>
> `page.click "//input[@value='acceder']"`
>
> `page.wait_for_page_to_load "30000"`

Esto es muy útil para escribir tests, porque ya presenta en secuencia todas las acciones que corresponden con un paso, en este caso el login. Mientras Selenium IDE sirve para *grabar* esta secuencia de acciones, [Selenium web driver](http://code.google.com/p/selenium/wiki/GettingStarted) sirve para volver a *ejecutarlas*. Selenium web driver es una herramienta que tiene interfaces para muchos lenguajes de programación, entre ellos una [gem para Ruby](http://selenium-client.rubyforge.org/). 

Cucumber + Selenium son los dos elementos fundamentales para escribir una suite de test. 
Para cada historia del tipo "Dado que.. Cuando.. Entonces", Cucumber entiende la estructura y pide que se explique come realizar cada paso (ej.: el paso "Dado que estoy registrado como cliente de pago de vLex España"). Selenium permite grabar la ejecución de cada paso en Firefox y pasar la secuencia de acciones a Cucumber como descripción de cada paso. 

Con estos dos instrumentos se puede empezar a escribir tests. Tenemos las historias escritas con estructura "Dado que.. Cuando.. Entonces", las ejecutamos con Cucumber que nos dice que pasos no entiende y hay que especificar. Con Selenium podemos escribir todos estos pasos y volver a ejecutar las historias. Esta vez Cucumber entiende los pasos y llama Selenium Web Driver que efectivamente va abriendo un browser y ejecutando todos los pasos como están escrito, simulando al 100% el comportamiento de un usuario web.
Al final de un paso, si el resultado no es el esperado, se puede ver en el browser el error, que es lo que se ve y que no se tendría que ver o al revés.
Esta es otra ventaja de testar de forma externa, que permite ver en un browser (y no en una terminal o un fichero de log) lo que está pasando, los resultados esperados y los fallos.

Una sintaxis más universal (Webrat)
===================================

Utilizar Selenium para escribir pasos para Cucumber puede resultar sencillo al principio, ya que los pasos vienen escritos por el plugin de Firefox.
Pero, cuando el número de tests crece y se toma experiencia con esta herramienta, ya no sale tan a cuenta abrir Firefox, ejecutar el test a mano, grabar los pasos con Selenium IDE, copiarlos y pegarlos.

A parte, la sintaxis que utiliza Selenium para describir las acciones no es totalmente natural. Por ejemplo, los pasos vistos antes para hacer un login son:
> `page.open "/"`
>
> `page.click "link=Acceso clientes"`
>
> `page.wait_for_page_to_load "30000"`
>
> `page.type "username", "mi_usuario"`
>
> `page.type "password", "mi_contraseña"`
>
> `page.click "//input[@value='acceder']"`
>
> `page.wait_for_page_to_load "30000"`

En cambio, una sintaxis como la siguiente es más clara y natural:

> `visit "/"`
>
> `click_link "Acceso clientes"`
>
> `fill_in "username", :with => "mi_usuario"`
>
> `fill_in "password", :with => "mi_contraseña"`
>
> `click_button "acceder"`

Para utilizar esta sintaxis, se introduce una nueva herramienta llamada [Webrat](http://wiki.github.com/brynary/webrat/). Por un lado, Webrat tiene un [conjunto reducido de acciones](http://cheat.errtheblog.com/s/webrat/) que se entienden muy fácilmente. Por ejemplo click_link sirve para hacer click en un enlace, que se puede indicar con el texto contenido, con una expresión regular, el TITLE o el ID del objeto. Por el otro, Webrat puede *hablar* con Selenium y traducir su sintaxis en pasos de Selenium. 

Webrat se sitúa come una capa entre Cucumber y Selenium, haciendo Selenium invisible para hacer test. Simplemente se escriben las acciones y Webrat se ocupa de invocar el Selenium Web Driver y de pasarle las acciones en un formato comprensible.

Poner una capa más entre Cucumber y Selenium tiene una ventaja. Normalmente Webrat ejecuta las acciones invocando el Selenium Web Driver, pero Webrat tiene también la opción de ejecutar las acciones directamente, por su cuenta, es decir, puede funcionar *sin Selenium*.

La modalidad Webrat::Selenium ejecuta todas las acciones abriendo un browser web; esta modalidad es intuitiva para detectar errores.
En cambio la modalidad Webrat a solas ejecuta las acciones con llamadas HTTP directas, sin browser; esta modalidad es menos intuitiva pero mucho más rápida, y es muy útil cuando el numero de acciones que testar es muy grande.

Una última ventaja de Webrat es la sintaxis de los pasos "Entonces" (según la estructura "Dado que.. Cuando.. Entonces.."). Estos pasos describen condiciones que tiene que verificarse al final de una serie de acciones. La ventaja de Webrat es que permite utilizar la sintaxis RSpec, que es muy conocida en la comunidad Ruby. Por ejemplo, un paso como

Entonces tengo que ver "Bienvenido Usuario"

se escribe en la sintaxis webrat/RSpec como:

response.should contain "Bienvenido Usuario"

Para resumir, el ritmo de trabajo con Cucumber + Webrat + Selenium es el siguiente:

- Escribir historias con estructura "Dado que.. Cuando.. Entonces.."

- Ejecutar las historias con Cucumber, que pide una explicación sobre como ejecutar cada paso

- Escribir los pasos según la sintaxis de Webrat y ejecutarlos otra vez

Cuando una historia falla
=========================

Cuando por fin se han escrito todos los pasos para describir una historia, se puede ejecutar el test. Si el resultado es positivo (la historia se cumple), no hay más trabajo. 

En cambio, si el resultado es negativo, algo falla. Un fallo se clasifica de dos maneras:

- @bug: el código está diseñado para cumplir una historia pero no lo hace. 

- @wish: estaría bien que la historia pasara, pero nunca se había planteado que esa historia fuera una componente fundamental del código.

En el primer caso, hay que ponerse a la obra y solucionar el fallo en el código o señalarlo donde necesario (ejemplo: Zendesk). En el segundo caso, se puede apuntar como punto para discutir en futuro, sobre si es necesaria o no esa historia.

En ambos casos, es útil **etiquetar** la historia, poniendo el tag @bug o @wish, para acordarse que esa historia ya ha fallado y no hay que sorprenderse si seguirá fallando.

Integración continua (Integrity)
================================

Utilizando Cucumber + Webrat + Selenium se puede escribir una suite de test muy avanzado para averiguar el funcionamiento de una página web. Todo lo que sale (o no sale) en el browser se puede testar, incluyendo mensaje de confirmación, enlaces asíncronos (AJAX), estilos, recorridos de navegación.

Una suite de test completa garantiza que todas las historias descritas *pasan*, es decir, que *en este momento* la página web cumple con las expectativas. 
Una página web, pero, es algo que cambia mucho con el tiempo, y ejecutar la suite de test una sola vez no es suficiente. 

Es importante que se ejecuten los tests en continuación, en particular cuando el código fuente cambia. Para esto, es importante que la suite de test se no ejecute *a mano*, si no que haya una máquina dedicada a controlar constantemente cuando el código cambia, a ejecutar los tests y, si necesario, avisar los desarrolladores de algún fallo inesperado.


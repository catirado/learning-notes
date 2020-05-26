# [Advanced Distributed System Design](https://learn.particular.net/courses/adsd-online)

https://github.com/Particular/Workshop

https://gist.github.com/craigtp/05a82b51557adc278acd71b5a2b88905

## Falacias de la computación distribuida

TBD

## Acoplamiento

Es muy subjetivo. ¿Que es el acoplamiento? Una medida de dependencias.
2 tipos de acoplamiento, efferent and afferent.

https://stackoverflow.com/questions/15272195/what-is-the-difference-between-afferent-couplings-and-efferent-couplings-of-a-cl

## Vendiendo messaging a tu organización

Primeros pasos parar introducir:

* Considera utilizar tablas de BD bajo una API message-driven (p.ej. SQL Server transport)
* Desarrollar código message-driven

## Messaging patterns

### ¿Como lidiar con mensajes desordenados?

Hacer colar con orden mensajes en es muy nuevo y complejo (DynamoDB???).
¿Como lidiar con el orden? En un web service puede pasar lo mismo (sobretodo en asincronos), aunque es menos probable.
No es realmente un problema de queue, tiene que ver más con asincronía.
Incluir un sequence number, y con eso podemos saber que nuestra version de bd en la ultima.
No es necesario lidiar con ello en todas las acciones del sistema.
Puede haber escenarios específicos que digas si no has procesado el mensaje 5 no proceses el 6. 
Es algo relacionado con el negocio, por eso no se solventa en las colas.

Ejemplo:
OrderAccepted y OrderBilled, puede llegar en casos raros primero el OrderBilled antes del OrderAccepted.

```
+---------+  OrderAccepted   +---------+
|  Sales  +----------------->+ Billing |
+-----+---+                  +------+--+
      |                             |
      |                             |
      |                             |
      | OrderAccepted               | OrderBilled
      |                             |
      |                             |
      |       +---------+           |
      +------->Shipping +<----------+
              +---------+  Can't find order in DB?
```

Si los retries fallan y fallan nos metemos en un loop infinito. En algun momento hay que meterlo al error queue. Si algo consideramos que ha sido un fallo de negocio (o ha pasado el número máximo de retries), no lanzamos excepción, lo mandamos a la cola de errores. La excepciones puede ser algo temporal, las excepciones no tienen porque ser errores.

En *NServiceBus*, las excepciones primero las tratan  en los retires inmmediatos como Info, si entra en el delay retries como warning, si pasa todos los retries configurados entonces lo manda al error queue y si loguea como error.

```csharp
public Task Handle(OrderBilled message) {
    var shippingOrder = orm.Get<ShippingOrder>(message.OrderId);
    if (shippingOrder != null)
      shippingOder.Ship();
    else
      throw new OrderNotFoundException(message.OrderId);
}
```

Tratarlas no dentro de cada Handle, sino a nivel de infraestructura, un único punto. Dejar que lance el ```NullReferenceException```, sin capturar la excepción especifica, porque ya tendríamos que tener el mensaje, ya llega donde peta y con eso sabemos. No queremos que la gente mire si es nulo o no aquí, asi el developer no tiene que acordarse de ello. **Don't check for null**.

```csharp
public Task Handle(OrderBilled message) {
    var shippingOrder = orm.Get<ShippingOrder>(message.OrderId);
    shippingOrder.Ship();
}
```

### Request-response

#### Return address patterns

Cuando mandes un mensaje si esperas una respuesta, mandan la dirección donde se espera esa respuesta. 
Físicamente no hay diferencia entre un request message y un response message.
Además, que añadir correlación (correlation ID), para saber que mensaje responde a una request.
Multi-respuesta: podemos tener multiples respuestas para una única request. Por ejemplo, para reportar estado de long-running procesos.
Esto no te lo dan by default los sistemas, son patrones a implementar.

*CausationId para relacionar las cosas que pasan relacionadas.??*

*¿Cómo hacer que sean el mismo contexto para que sean transacción? porque eso creo que lo hace con NServiceBus...*

## Pub-Sub

Mejor nombre sería subscribe-publish, porque es el orden en que tiene que hacerse, primero subscribirse y luego publicar.

```
+------------+      +-----------+
| Subscriber |      | Publisher |
+---+--------+      +---+-------+
    |       Subscribe     |
    +--------------------->
    |       Publish       |
    <---------------------+
    |       Publish       |
    <---------------------+
    |       Publish       |
    <---------------------+
    +                     +
```

No hay correlación entre la subscripción y las publicaciones que hay después.
Puedes que los suscriptores esten down cuando se publica el mensaje, pero cuando se levanten el mensaje llegará.

Pub-sub con scale out. Si tenemos dos maquina bajo un LB susbcrito, tiene que ir a las dos? o sólo a una? Depende. Para facturación está claro que sólo queremos ir a una. Pero si lo estamos utilizando para cachear datos, podemos querer que vaya a todas y esté actualizado en todos. Para esto, haz cache distribuida tipo Memcache, no utilices esto! Para Azure si que la gente lo utiliza y por eso en NServiceBus lo han implementado, porqeu cuesta pasta, y haces cache distribuida con esto e in-memory process.

### Topics

Se pueden publicar y subscribir a topics específicos. Hay un acuerdo implícito sobre el contento y la estructura del mensaje. En NServiceBus, usan el nombre de clase de un evento como nombre del topic. Así tienen un strongly typed acuerdo del topic, asi cuando hay un refactor no hay problema (no con magic strings!).

Jerarquia de topics. Products, Products.InStock, Products.InStock.PricedToClear...

Herencia multiple es más interesante aún. Modelar los eventos como interfaces en vez de clases. Un evento A hereda de B, C y D. Puedo susbcriptions a alguno o todos.

No deberíamos utilizarlo, evitar herencia. Hay casos raros en los que tiene sentido. P.ej. un producto y un producto digital, uno con con Shippable y otro no. To hook en prexistente. Reduce el content based routing.

### Events: In process vs distributed

in-memory, sync. publisher puede saber que subscribers están actualizados.
distributed, aync: publisher (y otrohers) subscriber no saben del estado global. Evitar a common database (copuling store) con el estado global de todo.

No global state, cada subscriptor tiene que tener su propia persitencia.

Unit testing no es lo más efectivo para detectar bugs en el un sistema, es el 3ro o 4to. Leer el Code Complete. Code reading (explanining) (aka pair programming) es la mejor ténica para encontrar bugs, explicas el código a un compañero y con eso es más fácil verlo. Es bueno para encontrar regresiones.

### Visualización

Tener diagramas in-sync de como el actual sistema funciona. Basado en audit log.

## Arquitectural syles: bus and broker

¿Que es estilo de arquitectura? lo que está permitido y lo que no en una arquitectura.
Puedes tener más de uno en un proyecto.
Comunes: layering, MVC, pipes & filters... 

### Broker

Conocido como "Hub and Spoke" o "Mediator". Contexto: Fue diseñado para evitar hacer cambios en apps (EAI). Tenemos aplicaciones preexistentes y queremos que hablen entre ellas, entonces montamos un punto único para la integración, así no tenemos que tocar las aplicaciones (ej. FORTRAN, COBOL, C...).

```
               +------+
               | App1 |
               +---^--+
+------+           |           +------+
| App6 |           |           | App2 |
+------+           |           +------+
      ^------>+----v----+<------^
              | Broker  |
      v------>+----^----+<-----v
+------+           |          +------+
| App5 |           |          + App3 |
+------+           |          +------+
               +---v--+
               | App4 |
               +------+
```

Está físicamnete separado. Todas las comunicaciones pasar por el broker. Handle failover, routing. Se convierte en punto unico de fallo, por lo que tenemos que ser robustos y performant.

**Broker technology**: BizTalk, WebSphere, Sonic ESB, MS Sql Service Broker, CORBA, UDDI.

|Ventajas |Desventajas|
|---|---|
|Concentra todas las comunicaciones en un único sitio lógico, lo que habilita gestión central.|11th fallacy: "bussines logic can be centralized"|
|Habilita "routing" inteligente, transformación de datos, orquestación.|Programación procedural a larga escala, sin buenos sistemas de test unitarios o control de versiones.|
|No requiere cambios en las surronding apps.|Previenen que la apps ganen autonomía.|

### Bus

Event source and sinks use bus for pub/sub.
Diseñado para permitir la evolución independiente de sources and sinks.

```
+----------+    +----------+   +----------+
|  Source  |    |  Sink    |   |  Source  |
+----+-----+    +----+-----+   +-----+----+
     |               |               |
+----v---------------v---------------v----+
|                B   U   S                |
+----^---------------^---------------^----+
     |               |               |
+----+-----+    +----+-----+   +-----+----+
|   Sink   |    |  Source  |   |   Sink   |
+----------+    +----------+   +----------+
```

No tiene necesariamente porque esta fisiscamente separado. La comunicación es distribuida (no hay punto único de fallo). El bus es más simple (no content based routing o data transformation). Tu tienes que cambiar el código para adaptarlo y solo se hace el routing. Su proposito es ortogonal al del broker.

Alguno de los "nuevos" ESB son realmente brokers (WebSphere, Mule, Sonic...).

**Tecnologia**: NServiceBus, MassTRansit, Rhino Service Bus, Tibco Rendezvous, RabbitMQ, Qpid...

|Ventajas |Desventajas|
|---|---|
|No hay punto unico de fallo. No rompe la autonomía de los servicios.|Más complicado diseñar soluciones distribuidas que las centralizadas.|

### Bus vs Broker

Lo común entre bus y broker es que los dos son un intento to hanlde spatial coupling.

Usa lo mejor de cada mundo. No trates de forzar un stile arquitecure to rule them all.

Busca alternativa para los 3 elementos de integracion:

* Data transformation -> MapForce, XSLT
* Protocol bridgning -> IP*Works
* Bussines logic

## Introducción a SOA

SOA fue una reacción a los fallos de EAI, de centralizar en una una pieza, gartner en los 90s.

### ¿Que es un servicio?

Principios de **SOA (Orientación a Servicios)**:

1. Los servicios son autónomos.
2. Los servicios tienen fronteras explícitas.
3. Los servicios comparte contrato y esquema, no clases o tipo.
4. La interacción entre servicios es controlada por políticas.

1 y 2, significa que encapsulación es una buena idea.
3 o cualquiera otra cosa, especificamente, no base de datos! No solo lo que comparten, lo que no comparten.
4 json, xml, over AMQP...

*Parte del problema con SOA es que coincidio con lo web services y se intento asociar a lo mismo. Web Services != Service Oriented Architecture*

```
+-----------+                        +------------+
| Esquema A <-----+    +----------->    Esquema B |
+---^-------+     |    |             +-----^------+
    |             |    |                   |
    |             |    |                   |
    |             +---------+              |
    |                  |    |              |
+---+-------+          |    |       +------+------+
| Servicio A+----------+    +-------+  Servicio B |
+---+-------+                       +------+------+
    |                                      |
    |                                      |
    |                                      |
    |                                      |
    |                                      |
    |        +---------------------+       |
    +------->+   Comunicaciones    <-------+
             +---------------------+
```

Un servicio es la **autoridad técnica para una capacidad específica del negocio**. Todos los datos y reglas de negocio residen en el servicio.

Todo debe estar un algún servicio. Nada "sobra" después de identificar los servicios.

#### Que NO es un servicio

* Un servicio que tiene solo una funcionalidad (como calculo, validación, etc.) es una función no un servicio.
* Un servicio que tiene solo datos (like CRUD entities) es una BD no un servicio.
* WSDL/REST, que lo llamemos remotamente, no cambia su responsabilidad lógica. Por mucho que pongas REST sobre una BD para un CRUD, no es un servicio.

Utilizando los [4+1 views](https://www.cs.ubc.ca/~gregor/teaching/papers/4+1view-architecture.pdf), los servicios están en la vista lógica, si lo vemos con la vista de desarrollo, *un servicio podría ser un repositorio en nuestro sistema de control de versiones*.

¿era aquí lo de mirar en el control de versiones para ver los cambios que se hacian????

Para compartir esquemas, publicarlos como NuGet por ejemplo.

lifecycle service events
Los procesos de negocio remain within services. Cascadin events give rise to enterprise processes.
como una cascada, no hay orquestación. cada servicio lanza un evento y diche hecho esto, y otros servicios reaccionana a eso.

Price changed -> Order Received -> Customer Billed -> Order shipped -> Inventory Replenished -> ...

Lo dificil encontrar los boundaries de que es un servicio. boundaries es algo que es obvio una vez lo has hecho, lo ves y dices, claro.  

#### Despliegue de servicios

* Varios servicios pueden ser desplegados en la misma máquina.
* Varios servicios pueden ser desplegados en la misma aplicación.
* Varios servicios pueden cooperar en un workfolow.
* Varios servicios puede ser mashed up in the same page

Un error comun es pensar que porque el usuario necesita ver un pieza de datos, esta tiene que estar en el servicio principal del sistema que usa. una vez ves que se puede componer la UI, ya no necesitas tener tantos datos compartidos entre diferentes servicios. ¿a quine le pertence el dato?unico punto de verdad. Eventual consistency es un atributo necesario para saber que hay que.

Ej. Amazon product page. Para el usuario puede ser una única página, pero para nosotros son varios servicios con sus repositorieos indepndientes, etc.
UI composition. Widgets que acceden a los diferentes servicios

* product catalog
* pricing
* inventory
* cross sell

en MVC multiple controller for the same URL, one page puede ser más de un controller. Cada servicio registrar sus propios controladores.

#### Performance optimization

Si es un problema tener que hacer muchas request (sobretodo en móvil), podemos tener en la parte cliente una libreria quer retenga las request en memory y cuando este todas las mande en una única request. En servidor desempaquete las request y las mandamos a los servicios correspondientes, collecta las respuesta y las devuvle al cliente. El cliente hace del dispatch a los correspondientes callbacks. Netflix lo hace? buscar

El mismo servicio se tiene que utilizar en los diferentes servicios (aplicación movil, una aplicacion de backwend, un portal, etc.). Bussies boundaries vs techincal boundaries. Son crosffucionales"?

#### Checkout workflow

En vez de tener un gran objeto padre que contenga la info de los demás, lo hacemos al revés, podria haber pedidos sin dirección de envio (libros kindle) o sin detalles de tarjeta de credito (bitcoins).

Antes de llegar al checkout ya he pasado por aquí:
ShippigAddressSelected -> OrderId + shipping info
BillingSelected -> OrderId + billing info

Cuando hago el checkout -> OrderReceived - solo el OrderId. Es decir, tenemos ya la info necesaria asociada a ese OrderId anets de que se haga.
El servicio de Billing se entera -> Customer Billed
El servicio de Shipping se entera -> OrderShipped

¿cuando se crea el orderid? en amazon por ejemplo en el primer paso, cuando pulsamos en Proceed to checkout. Entonces ya lo podemos utilizar en los demás servicios.

### UI Composition and branding service

Al servicio de precios no le importan que salgan en verde o en azul. No es su responsabilidad.

* Esquemas de color, layout, fuentes, CSS, imagenes, etc.
* All coumnicate the "corporate brand"
* La responsabilidad del servicio de branding

Client side composition.
Los ViewModel creados con Whatehver.js (angular, react, vue).
Cada servicio binds su model a la parte del view model.

SOA-VM -> en vez de tener una lista con todos los elementos ```List<ProductInfo>```, tenemos un VM con varios diccionarios.

``` csharp
Dictionary<ProductId, Name>
Dictionary<ProductId, Price>
Dictionary<ProductId, Inventory>
```

El servicio aun tiene cliente code, para popular el vm pero no de como se muestra.

Or leverrage CSS classes, p.ej. class price make it bold, red, class inventory make it green. O algo mejor si utilizamos less, saas...

Currency? branding es acoplado a la data que tiene que mostrar... neutral as possible y dejar la mayor posible responsabilidad al servicio.

Branding manda en el page flow, cual es la siguiente URL p. ej. hay cierto nivel de undserstangind que se hsae que los datos vienen de distintos servicios.

https://github.com/Particular/Workshop/tree/master/exercises/01-composite-ui

Para mails server side composition.

### IT/Ops service

*~ Infrastructure ¿de hexagonal?*

Responsable for keerping formation flowing (and secure) in the enterpise

Focused on the deployment & physical views: Responsible for hosting (web servsers, DBs, etc).
Owns connections string, queue names...

Authentication, authorzaction (pero sin movernos a las bussines rules, que pueda acceder al sistema A o B es su responsabilidad, que pueda hacer una transferencia (accion) de tal NO es su responsabilidad), LDAP/AD acces..

User login screen, sign in, sign out, user permissions, impersonation...

Mientras todos los servicios sigan unas convenciones, mágicamente haremos todo el pegamento físico con este servicio. Logging, containers, ORMs...

Bussines oriented services not concern about the deployment view.

Normalmente not hace pub/sub con otros services. no introducirlo a no ser que sea necesario.

Un caso (extraño).
HR - empleado contratado o despedido -> provisionar/ deprovisionas machines, cuentas, etc...

También las librerias que se encargan de agrupar las llamadas y disparchar en front y luego redigir en back, es de IT/Ops.

#### Integration endpoints (+ email)

https://www.enterpriseintegrationpatterns.com/ramblings/03_hubandspoke.html
hub & spoke for integration (data exporter message handler) es su responsabilidad también. Desde los bussines services se utiliza in process, no hay que hacer llamadas remotas entre it/ops y los otros servicios.
//Explicar mejor esto

Se puede hacer un paquete Nuget para las interfaces de integración.

request-response in a service, not between bussines services boundaries.

Cada servicio decide que data model es mejor para el (no todo tiene que ser relacional). Pero meter un ORM (en vez de 3), y por eso pertcene a IT service, por normalización. Lo mismo de branding y meter solo un fmw por estandarizar. Si tienes una necesidad especifica para tu servicio, vamos a hablarlo.

La interacción con branding o it/ops normalmente no se hace a través de messaging. Son más como un conjunto de librerías y "algo más".

## Ejercicio: services modeling

No poner nombre primero a los servicios, hazlo luego, delay. Utiliza colores al principio, y ve añadiendo responsabilidades y como se comunican, los eventos... muchas veces tener puesto el nombre te lleva a tomar decisiones basado en eso.

¿tiene sentido generar eventos aunque no haya nadie que los vaya a escuchar? Poner enfoque outside-in, ver que eventos necesitamos para nuestro bussines case, porque cada evento que generamos en un contrato. Default approach, prefer no exponer cosas a no ser que las necesites.

¿xq si podemos compartir algunos datos como el BookingId pero no los prices? Pensar, ¿como los datos/requisitos van a cambiar a lo largo del tiempo? a booking, a romm es constante, pero los datos que tiene una reserva van a cambiar a lo largo del tiempo. Precio puede empezar como un concepto simple, un number, pero luego le metemos currency, bitcoin, luego bitcoin es un comoddity, etc... si lo compartimos, cuando elconcept evolves tenemos que reflejarlo en mas de un sitio, violando SRPm y eso es lo que queremos evitar. Los identificadores son conceptos muy estables.

Encontra los boundaries es complicado en un dominio nuevo, tienes la sensación de que nos sabes cuanto queda hasta que empiezas a ver la luz... cuando lo enesñes y te digan, pero es obvio! entonces vas por buen camino.

Tener cuidado con department boundaries! Marketing service, sales services... suena tentador. Pero la realidad es que modelar lo que hacen actualmente puede ser caótico, no real boundaries, power fighting...

Processing: billing, shipping, etc. tienden a ser procesos mas que servicios. La mayoria de procesos usan más de un servicio together.

Lo que normalmente pasa es que lo que en OOD definiamos como Entidad, acaba pertenciendo a varios servicios, parte de ella en cada uno.

P.ej. piensas que prices y description de la habitación pueden pertecenecer a diferente servicios.
"empty requirements" -> buscar requirement que refuten. p.ej. si la longitud del nombre de la hbitacón es mayor que 20, el precio sube. la reacción tiene que ser, no tienes ni idea, nadie en todo el mundo va a querer eso!! avisar te voy a hacer preguntar estupidas, pero la idea es buscar que es estable sobre este dominio y que es volatil.

Si podemos seperarlo en un servicio, lo separamos, si vemos que llevamos el 60-70% de los casos de uso analizados y sigue siendo solo un CRUD, igual es momento de juntarlo en otro servicio.

¿pueden otro servicios invalidar el evento? preguntarnos eso para saber si tiene sentido.

Recomendación. si puede ser separado, empieza en un servicio separado, y mira más casos de uso a ver si tiene sentido unirlos.

https://github.com/crmorgan/hotel-management-system

Los ejemplos que hemos visto eran con dominios cononocidos (Amazon, reservas de hotel...) por eso, en general, necesitaremos expertos de dominio para modelar los servicios. No tiene porque ser un VP, a veces el viejo ingenerio de mainframes que ya ha se ha pegado con todos los dptos...

## Advanced SOA

En negocios grandes podemos tener la necesidad de partir la funcionalidad de un servicio.

### Bussines componentes
Elementos técnicos que implementan the capability's constituent parts. next level of bussines decomposition. similar to bounded contexts?

En la mayoria un servicio solo un BC. Caso común.

#### QoS requirements
p.ej los usuarios hacen pedidos de 10 unidades, pero hay unos nuevos que hacen pedidos de 10 millones de unidades. Desde la perspectiva de deserelización los últimos by default le va a costar más... esto puede indicar que algo de decomposition es necesita a nivel de bussines level.

```
+-------------------------------------------+
|                S  A  L  E  S              |
|   +--------------+     +--------------+   |
|   |  Regular     |     |  Strategic   |   |
|   |  Customer    |     |  Customer    |   |
|   +--------------+     +--------------+   |
+-------------------------------------------+
```

Puede que acaben con APIs distintas, etc..

```csharp
if (customer.IsStrategic()) {
    // don't show cross shell
}

if (customer.IsStrategic()) {
    // show history of sales in home page
}

//...
```

Si vemos esto repetido (rule of three?) es un indicador de que es un componente separado del negocio para ese capability.

Vamos a tener requisitos que nos llevan en una dirección y requisito que nos llevan a otra. Vemos que continuan evolviendo en diferentes.

Forking the codebase, a copy of the code, only forking el servicio especifico donde tiene impacto (en este caso Sales).

Así puede que necesites utilizar diferente modelo de bd o lo que sea...

> Quick note: cuando divides un servicio en BC, 2 es el caso más común, 3 seguramente sea incorrecto...

Y no debe cambiar durante el ciclo, si es strategic lo es durante todo el ciclo de vida. Lo de silver, gold, no es un buen ejemplo.

Más ejemplos:

* Airlines: different counters for bussines class
* Shiping: necesitas track temperaturas for frozen/meat. Los truck para llevar papel son distintos
* Billing: monthly invoices for customers with agreement. Otro billed directamente a Credit Card.

#### Bussines components & PUB/SUB
Los otros servicios no tiene porque enterarse de la división interna del servicio en BC.
Entre BC no se hablan, ni directamente, con PUB/SUB... stronger subdivision. No esta pensando para hablar entre ellos... como mucho pueden compartir un set of tables en la BD.

El contrato de los eventos (nuget package) es lo unico que se mantiene al nivel de servicio, cuando separamos en varios BC. Y compartimos mismo namespace.

Puede ser que bifurquemos y solo uno sea capaz de aceptar y generar el evento. Pero tambien que nos haga cambiar nuestros eventos, antes OrderShipped para mandar email, ahora por ejemplo, ProductIdsShipped, ya que unos se envian con el camión normal y otro con el frio y eso puede hacer que mande dos mails en vez de uno.

¿diseñar BC from start? si hacemos un refactor de un monlito y ya tenemos identificados esos problemas.

### Autonomous components

*similar to microservices?*
Puede que tengamos mensaje que no cambien estado, otros que puedan perderse en caso de fallo y no pase nada (datos volatiles como stock quotes)...

Podemos dividir un BC along transactional lines for improved performance in "Autonomous components". Most bussines transactions stay within the boundary of an AC.

Un BC está compuesto de uno o más AC.
Un AC es responsable de uno o más tipos de mensajes -> Idealmente uno -> foco SRP, puede ser todo el CRUD, no volverse loco.

Están compuestos de uno o más message handlers (controlador MVC, command handler, etc) y el resto de las capas en el servicio.

>Podemos verlos como **paquetes independientes** (pensar en paquetes de  NuGet).

Multiples ACs pueden estar hosteados juntos (p. ej. en el browser, o en integration), esa es la principal diferencia con el concepto de microservicios. Ocasiolamente se puede hostear separado.

¿Cuantos AC en un servicio/BC? Al menos uno por sistema (pensar en portal en java, app en swift, backend en donet). Pero en el backend seguramente lo partas en más. Diferentes tipos de mensajes norlamente nos indican que son diferentes AC, pensar en SRP.

are AC vertical slices? not usually. most of the time dont have tha ability to own data.

#### Acoplamiento entre AC

* No intentes reutilizar código (o compartir dependencias) entre AC's
* Esforzarse por conseguir un código "desechable".
* Solve for today problem, not tomorrows ([JFHCI](https://ayende.com/blog/3547/jfhci-the-evil-that-is-configuration)).

Como partimos en partes tan pequeñas es muy fácil reescribir una de ellas si hace falta. Mejor mantenibilidad.

database shared entre AC's? 

Services is the logica view. AC's is the packaging view.
```
                                                            Service
             +------------------------------------------------------+
             |   +--------+                            +-------+    |
+--------------->+   A C  +--------------------------->+       |    |
             |   +--------+                            |  D B  |    |
             |        +--------+                       |       |    |
+-------------------->+   A C  +---------------------->+       |    |
             |        +--------+                       +-------+    |
             |                                                B  C  |
             +------------------------------------------------------+
             |  +----------+                           +-------+    |
+-------------->+   A C    +-------------------------->+       |    |
             |  +----------+                           |  D B  |    |
             |           +----------+                  |       |    |
             |           |   A C    +----------------->+       |    |
+----------------------->+----------+                  +-------+    |
             |                                                B  C  |
             +------------------------------------------------------+
```

Podemos tener common service layer in multiple ACs. Librerias comunes, entidades comunes, etc.

AC no implica o requiere physical autonomy. Empezar con todo shared, pero se puede llegar a tener todo owned.

Lo malo de los Stores Procedures es el reuso (las dependencias realmente)... llamar de uno a otro, etc... las dependencias. Y también la testabilidad.

Cada AC enfocada en un caso de uso diferente.

https://www.infoq.com/presentations/SOA-Business-Autonomous-Components/

AC son the unit of packaging (nuget, NPM, etc) en SOA, deployed into systems. AC toman responsabilidad de un tipo especifico de mensajes (idealmente solo uno ->SRP) . se esfuerzan por usar el estilo de bus entre la ACs (client-side event in browsers, server-side events o message queues events), para tener autonomía entre los ACS -> escability.
Fisical deployment -> en 2 sistemas -> fron y back.

services been logical -> bc logical view -> ac/ci packages -> packages put together en multiples system creates a system. octopus deploy.

https://octopusadsd.particular.net/app#/
https://gist.github.com/dvdstelt/2528b08ce2476e73e009433f419d632e

Los AC de message handlers hay veces que se despliegan como procesos separados, pero muchas veces no.

Relation DDD y SOA
Strategic design bounder context similar to BC. Similar la busqueda pero SOA añade que no puedas compartir datos entre boundarios, DDD no dice nada de eso.
ITOps service similar to anticorruption layer.

### Services boundaries

**Insertar los dibujos del iPad**

En la mayoria de organizaciones cuando datos son coleccionados en un parte de la organización and flow to otra parte de la organización when process it , casi siempre, un servicio catch acrross all of this parts.

Buscar las cosas que naturalmente cambian juntas en (acrross) diferentes partes de la organización. después de eso tracing how this data flows ¿quién más la está usando? eso te da más casos de uso.

Hay veces que es necesario que alguien externo le diga a managmente lo mismo que estas diciendo tú...

Encontrar las boundaries correctas nos ayuda con la complejidad de los problemas de negocio pero nos lleva a problemas adicionales.

### Reporting

Es complejo, pero no es muy diferente de Composite UI grids.
Primero intentar ver que problema esta intentando resolver con el report, recordemos que no vienen con requerimientos sino con workarounds. Realemnte quieren buscar algo.

Pero hay mejores maneras:

**Regular reports**
Informe que la gente ejecuta periodicamente (diariamente, semanal...). En esos normalmente hay un patrón. P.ej. ¿cual es la cancelación más grande en las últimas 24h?

* Buscar el patrón que los usuarios están buscando.
* Modelar las constantes como conceptos del dominio. (puede ser que metamos nuevos conceptos de dominio como pedido grande que metamos al publicar el evento, paises de alto riesgo, etc.)
* Implementarlo como event-correlation y mandar un mail.

La gente acaba pidiendo de este tipo en vez de informes, porque ya no tienen que preocuparse... incluso tienen cierto tipo de "real time" reports.

No queremos tener un reporting service. Porque meteriamos coupling, si cambiamos un dato para hacer un tipo de informe puede que fastidiemos otro.

**Research / Data Science / Ad-hoc reporting**
Data ware houses o data marts nunca han encajado bien aquí, porque igual solo lo ejecutan una vez y luego quieren otra cosa y tienes que preparar los datos para eso. ¿Como lo haces en SOA? no lo haces, le das los datos que quieran. Usar data exporter para exportar los datos que necesiten (un mes, un año...). Hay tooling especifico para data science, pero eso no es service oriented.

### Integridad referencial

No hay BD central, no hay FK entre boundaries.

Puede haber inserciones de "hijos" sin que se haya insertado el "padre". Esto se soluciona con la consistencia eventual. *ej. creamos el guest con un bookingId, pero aún no hemos creado el booking.* ¿es un problema? con los retries al final se solucionará aunque al comprobar algo sea nulo en un primer momento. capeces de recuperarse de transient inconsistency.

Borrado (pero no "en cascada"). Borrado en cascada es evil. Borrar pedido, borrar el inventario...

* Datos privados. Todo en el servicio. OK.
* Datos publicos. Datos compartidos entre servicios. No borrar.

ej. borrar un producto? que hacemos sobre los pedidos, el inventario...?

No pensarlo como soft delete, no es un problema técnico, es de negocio.

http://udidahan.com/2009/09/01/dont-delete-just-dont/

### Team structure

Especialistas técnicos en IT/Ops favoreciendo elecciones de frameworks, lenguages, infraestructura... sirviiendo como consultores a los equipos de servicios. Los requisitos tambien pasan por IT/Ops, no se puede saltar a los team services. Puede crear algo de silos. En empresas pequeñas no es necesario hacerlo así y se puede seguir haciendo SOA.

Modelo alternativo -> task forces. No recomienda ir directamente aquí, primero hacer la anterior. En vez de tener equipos alineados con services boundaries.
https://twitter.com/UdiDahan/status/944944757444435968
https://threadreaderapp.com/thread/944944757444435968.html

### ¿Cuando no hacer SOA?

Startups -> SOA es para modelar abstraciones estables del negocio. En las startups no sabemos que es el negocio aún! Es un proceso de descubrimiento.

Generic/extensible platform project. Tienes un proyecto, viene un cliente que quiere algo parecido y acabas haciendo el plataforma genérica/extensible en la que quepa todo. SOA no va a ayudar. No va a ser éxitoso :( En SAP o Salesforce, total tienes que tirarte un año para implantarlo con nuevo código.

El problema que tiene con microservices es que cogen un concepto de arquitectura lógica y lo acoplan uno a uno con un elemento de la arquitectura de despliegue. logical unit = deployment unit.

## Service Structure - CQRS (4h17m)

CQRS como el enfoque, el proceso de segregar las responsabilidades en comandos y queries.

> No es algo que aplique para todo el sistema. Coger un servicio o un BC y ver como aplicarlo en ese fragmento. Cada servicio/BC podemos aplicar un enfoque distinto.

Lo que intenta solucionar: multi-user collaboration. multiples usuarios/actores (3rd party, batch jobs..) influencing some common data. Es común que varios usuarios consuman los datos, pero menos frecuente que varios lo modifiquen concurrentemente. Single writer - multiple reader content. Soluciones tradicionales, optismistc concurrency. Asumir que los usuarios estan siempre mirando stale data, porque puede que alguien lo haya modificado despues de haya consultado tú.

¿cuantos datos son colaborativos?
La mayoría de los datos no son colaborativos. Ej. en Amazon, muchos lectores para nombre,imaagen, etc del libro, muy pocos writers. El stock y las valoraciones podrían ser modificados por múltiples escritores.

First question of CQRS approach, este dato es colaborativo o no? many readers few writers contexts o multiple writers context?

> Greg Young dice puedes utilizar CQRS para non colaborative domain, pero es cuando lo haces como un patrón de implementación. Para Udi CQRS es más un patrón de análisis que un patrón de implementación.

### Non colaborative domains
Pocos writers, muchos reders.

Traditional layer architecture, ¿es la solución más simple? ¿es apropiado para non colaborative domains?
```UI <---> Facade (Cache) <---> BL <---> DAL <---> DB```

¿Porqué necesitamos transformación entre todas las capas?
* DB: ORM para mapear tablas a objetos de dominios
* WS: map from DTOs and WS to domain objects
* UI: map from DTOs & WS to View Model objects.

Díficil de explicar a un Junior porque necesitamos todas estas transformaciones...o porque es más escalable o mantenible...

P.ej. product catalog, es unas cuantas strings guardadas en database. invetory, prices es en otro servicio. Aqui no tenemos relaciones, por lo que podemos pensar en soluciones más simples.

Soluciones más simples:
* Datasets UI through DB
* RoR
* **Browser consuming a DocumentDB (RavenDB, Mongo, Cosmos..) through JSON over HTTP**. IT/OPs hace the authorization/authentication through intercepction. Las validaciones en el browser con JS y podriamos tener ese mismo codigo JS en la DocumentDB.

Para escalar esta última opción:
* A través de read-write replication
* A través de sharding

Muchas veces nos olvidamos de las capabilities ouf-of-the-box in databases e intentamos hacer CQRS cuando ya tenemos esto...

Securtity perspective movemos responabilidad a IT/Ops. Si es un problema igual podemos meter un servidor de nodejs (por ser tmb JS) en medio, pero no hacemos tanto mapping como antes.

### Collaborative domains

Ejemplo black friday... los dias normales funcionan bien... un item muy popular nos puede bloquear todo.

`ProductId | Qty`

Multiples transacciones intentando actualizar el mismo record nos da un lock. Cuando estan las transacciones en una cola, block. Cuando hay demasiadas conexiones abiertas, connection refuse y ya no se puede comprar ningún otro libro.

los que son high collaborative puede tener high [contention](https://en.wikipedia.org/wiki/Block_contention), y esos los que nos pueden tirar todo el sistema si no los diseñamos bien. Por eso nos hacemos esa pregunta.

Si ponemos una cola delante, la única forma que ayuda es si limita la concurrencia, pero también hace que las compras de otros libros que no son sobre el que tenemos problema no se ejecuten esas transacciones en paralelo. BD ya hace una especie de queue.

Con esto quitamos respn del command, y lo vemos al query. Ya que tenemos que hacer un SUM del delta column.

```
UUID | ProductId | Delta Inventory
----------------------------------
     | 2         | -1
     | 2         | -2
     | 2         | +250
```

Esta solución no es 100% equivalente a lo anterior, ¿como garantizamos que hay suficente inventario cuando alguien hace una compra?

Con el anterior podemos checkear directamente, ahora tenemos que hacer un requery como parte del comando. El problema con esto es que si solo nos queda uno y dos transacciones lo hacen a la vez, nos quedamos con inventario negativo, porque la BD no bloquea el record. Por lo que la única forma de evitarlo en cuando hacemos el select bloquearlo, lo que hace peor el problema original, ahora no solo bloqueamos un registro sino varios! Incluso podria escalar a un table lock!

Para solventar este problema de escalabilidad, tenemos que meter async e introducir un new bussines state, temporary negative inventory.

En highly collaborative domain with high contention u need to make changes in your domain to solve the problems. Special attention to data model. CQRS is not only a techincal thing, estando cambiando la manera en la que el negocio funciona.

Para la solución de `ProductId|Qty` hay algunas bd que tienen **atomic increment/decrement** para que esas operaciones se hacen en pararelo y no bloquea. Pero si no podemos soportar inventario negativo esto esta out of the table. Es mejor solución así no tenemos que tener varias rows por product y las queries son más simples.

https://docs.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/in-memory-oltp-in-memory-optimization?view=sql-server-ver15
https://redis.io/commands/incr

Para la otra alternativa tenemos que aplicar algún algoritmo de compaction para que no crezca tanto. Cada x minutos/horas hacer el sum, borrar los registros y poner el actual. Careful cuando implementemos esto, compaction a la vez que se hace inserción... para que esto vaya bien añadir una columna Timestamp, y borramos por Id y un timestamp window, asi no tnemoes que preocuparnos.

CQRS como técnica de análisis.

Customers who bough this also bought -> customer who bought product id X also bought product id Y. Model populate by a lot of users. Probablemente el orden es importantante.

```
ProductId | ProductId | Qty
----------------------------------
2        | 3          | 5
2        | 4          | 6
```

Qty (seguramente siempre que veamos un qty) problmatic in a content. Esto es lo típico que funciona bien cuando no hay muchas compras, etc.

¿es colaborativo? si es colaborativo, ver la naturaleza del contention. Y el siguiente paso, data modeling. Ver como podemos hacer el modeling para que no haya contention.

Si vemos que la simple navie solucion no va funcionar, pensar algo más, desconectarnos del pensamiento del modelo relacional.

Una tecnica que le suele ayudar en el modeling es ver sobre un papel como organizarias los datos, para ver como lo organizarias de forma natural. Y en este caso ves que parece un grafo... entonces igual deberiamos considerar una BD de grafos para esto.

> Si alguien en la org. te pone pegas y te dice si no lo puedes hacer con una relacional? si, con un mucho tiempo puedo hacer algo similar, que tenga las misma caracterisiticas, con una tecnologia que no controlo, etc et.c

Mucho de CQRS es data modeling y data storage selection. CQRS and NoSql van de la mano. Y así es como pueden acabar varios servicios con distinta tecnología de BD.

### Teoría CQRS

En dominio colaborativos, ¿cual es el nivel de refresco de datos que necesitan los usuarios? Analizarlo. Por ejemplo, un balance bancario de hace 10 minutos puede ser suficiente, pero no de hace 2 semanas. Ser sincero sobre ello, incluir un timestamp.

Mantener las consultas simples.

**2 layers == 2 tiers** (no introducir capas adicionales)

``` UI ---> (query only) Persistent View Model```

Para cada vista en la UI, tener una vista/tabla en la BD (en non collaborative domain haciamos lo mismo). Basada en los tiempos que hemos analizado antes.

``` sql
SELECT * FROM MyTable (WHERE ID=@ID)
```

Esto funciona bien en entornos SOA (small nice boundaries services), en no SOA esto de tener desnormalizado no funciona tan bien. Sino tendriamos que tener una vista con un montón de columnas de todo lo que el usuario quiere ver en la pantalla.

Be aware if you're creating read models that appear to differ in a minor way (i.e. a report for standard users containing CustomerId, Name, PhoneNumber and a report for supervisors that contains the same fields and adds LifetimeValue). Lifetime value is derived from orders and prices, and will exist in a different service than customer names and phone numbers, so always re-evaluate the service boundaries to ensure they're correct.

Desplegar the persistent view model DB to the web tier (only ```sql SELECT``` is permitted). No tiene porque pasar a través del Firewall (más rápido).

Role based security: Diferentes pantallas para roles distintos va a tablas distintas - SELECT permissions por rol. Al tenerlas particionadas la seguridad se puede simplificar, este grupo de usuarios puede o no puede acceder a este grupo de tablas, etc.

Es tán seguro como in-memory caches, sino más.

Usar para la premilinary validation in bussines rules before submiting a command.
Enforce uniquiness, user name unique in a signup por ejemplo. 

http://udidahan.com/2009/12/09/clarified-cqrs/
http://udidahan.com/2011/04/22/when-to-avoid-cqrs/
http://udidahan.com/2011/10/02/why-you-should-be-using-cqrs-almost-everywhere/
http://udidahan.com/2012/02/10/udi-greg-reach-cqrs-agreement/

## SOA: Operational aspects (1h23min)

## Sagas/Long running bussines processes modeelings (1h14min)

## Exercise: saga desing (1h13min)

## SOA: modeeling (1h14min)

## Organizatinoal transition to SOA (1h53m)

## Web Services and User interfaces (58m)

## Referencias

* [Own the future](https://www.youtube.com/watch?v=2iYdKQXGY2E)
* [Udi Dahan - If (domain logic) then CQRS, or Saga?](https://www.youtube.com/watch?v=fWU8ZK0Dmxs)
* [Finding Service Boundaries – illustrated in healthcare](https://vimeo.com/showcase/3715841/video/113515335)
* [Microservices and Rules Engines – a blast from the past](https://www.youtube.com/watch?v=Fuac__g928E)
* [Avoid a Failed SOA: Business & Autonomous Components to the Rescue](https://www.infoq.com/presentations/SOA-Business-Autonomous-Components)
* [Business Logic, a different perspective](https://vimeo.com/131757759)
* [CQRS – but different](https://vimeo.com/131199089)
* [Udi Dahan - Commands, Queries, and Consistency](https://vimeo.com/43612850)
* [So You Think You Know Pub/Sub?](https://www.infoq.com/presentations/pub-sub-domains)
* [Multi-dimensional architecture](https://skillsmatter.com/skillscasts/7784-multi-dimensional-architecture)
* [Loosely-coupled orchestration with messaging](https://skillsmatter.com/skillscasts/5090-loosely-coupled-orchestration-with-messaging)
* [Integration lessons for the green-field developer](https://www.youtube.com/watch?v=AXU4-VlAAcg)
* [An Integrated Services Approach](https://skillsmatter.com/skillscasts/5235-keynote-an-integrated-services-approach)



http://udidahan.com/2009/06/07/the-fallacy-of-reuse/
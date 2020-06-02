# [Advanced Distributed System Design](https://learn.particular.net/courses/adsd-online)

https://github.com/Particular/Workshop

https://gist.github.com/craigtp/05a82b51557adc278acd71b5a2b88905

## Falacias de la computación distribuida

TBD

## Acoplamiento

Es muy subjetivo. ¿Que es el acoplamiento? Una medida de dependencias.
2 tipos de acoplamiento, efferent and afferent.

https://stackoverflow.com/questions/15272195/what-is-the-difference-between-afferent-couplings-and-efferent-couplings-of-a-cl

## Introducción to messaging


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

## Service Structure - CQRS

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

- Antes de submit data, comprobar si ya existe en el PVM.
- Enforce uniquiness, user name unique in a signup por ejemplo.
- Related entity existence. p.ej. address validation, validación de un nombre de calle

Esto hace que tengamos menos comando rechazados, ya que evitamos mandar comando que sabemos que van a fallar.

Ejemplo. Al intentar hacer sign-up, me fijo en el PVM pen front ara asegurarme que no exista. En el improbable caso de que esto haga que cuando llegue el comando realmente ese usuario exista, por signal-r notificamos al usuario de que su nombre ya esta cogido y que le hemos asignado otro. Esto escala mejor. Facilitar la UX a los usuarios que siguen el flujo normal.

**Validation vs bussines rules**
Validation:
- is the input potentially good?
- la estructura es correcta?
- rangos, lengths, etc

Bussines rules:
- debemos hacer esto?
- bassado en el estado actual del sistema

> PEGAR DIAGRAMA

Normalmente solo se hace las validations en front, esto es intentar mover alguna (no todo) de las reglas de negocio al front consiguiendo feedback mas temprano y evitamos tener que esperar la respuesta del commando (asincrono) porque igual ni la necesitamos.

Otro ejemplo. En un carrito al hacer un add item, normalmente sacariamos el precio como respuesta de un comando pensamos que así sacamos los datos de verdad. ¿pero es así? puede que nos hayan cambiado por detrás el estado del producto y ya no esté a la venta, o incluso que se haya después de que yo haya entrado a la página (condition race...), lo tengo en múltiples tabs (esto se puede solucionar son SignalR).

Casi cualquier if statement en un command encontramos detras un modelo colaborativo.

Un comando casi nunca debe fallar, tiene éxito, y algunos tienen éxito pero de forma diferente.

**¿Deberiamos hacer lo que el usuario pide?**

En el pasado se hace mucho in-place edit, and then save when is ready. Esta bien para pequeño numero de usuarios, cada uno en su sandbox..

```
Id | Total | Account
----------------------------------
2        | 3,45$         | A100
4        | 4,50$          | A234

[SAVE]
```

Transactions boundaries, tu mandas un monton y un usuario ha modificado un record cuando tu estabas modificando, entonces cuando lo envias te rechaza toda la transacción... puedes pensar en hacer una transaccion por cada fila, pero puede que haya movimientos relacionados (tipo transaccion bancaria, etc.)

Muchas veces lo complicado para los modelos de colaboración está en la UI... ui nacio pensando en un usuario único.

P.ej. sistema de reservas (eventbrite) para reservar un asiento. necesitamos datos reales, no podemos usar un PVM... aqui lo anterior no funciona muy bien xk la mayoria de comandos van a fallar... aqui CQRS lo hace peor... aqui no podemos capturar la intención de usuario correctamente con esta UI (checkbox, user can select multiple seats? entonces cocurenccia ocupera el primero coge los asientos. ...) -> rethink user intentions to rethink UI. diseñar quien compra cada asiento es el problema, no podemos escalarlo.

big problem -> subdivice in small problems.

Capturing user inten
group reservations (people want to si togheter), number of people, seat type (indicate cost), email back when reservation can be filled, include waiting list. O incluso particionar en VIP cliente (1ra venta de tiempo, quieren lo mejores asientos y pagaran extra), grupos (2do ventana, se quieren sentar juntos) y el resto (la ult, cogeran cualquier aasiento libre). Nos costará menos dinero y podremos cobrar menos dinero a lagente...

CQRS is not an approach u can apply behind the scenes and supply the bussines as it was...

¿Que es un buen comando? Los que puedes contestar con:
- Gracias, tu email de confirmación te llegará pronto 
- OR just fake it the UI (y di que se ha complentado aunque este en una cola)

Inherently asyncronoes
Not realmente relacionado con entidades

### CQRS en acción

> Meter un grafico de tipico de CQRS

Cuando creamos un model optimizado para reads, normalmente no esta optimizado para actualizaciones. Lo que podemos hacer es batchear las actualizaciones del View Model, porque sino lo vamos a bloquear y vamos a tener el mismo problema.

A veces utilizar la misma BD o replica como ViewModel, y no tener que mantener otra BD o vista distinta. CQRS es un enfoque.

> pregunta, separate BC para libros que tienen mucha dmeanda y para los normales para poner distintas soluciones al tema del inventario. pero a veces puede cambiar de uno a otrao, eso no cuadra con la definición de BC, pero puede valer para nosotros.

discounting rules, move this to the client? preguntarse si es colaborativo. entonces pueder ser sincrono, hablando directamente a la bd y cogiendo la respuesta...

Resumen
- Pensar en lecturas y escrituras de forma diferente. how many write in parallel? 
- Reflejar las diferencias en los esquemas.
- Diseñar comandos que casi nunca fallen (en un dominio colaborativo). Cambiar modelo de dominio, explicar que el modelo antiguo no escala. Es física, sino la única solución es pagar un montón.

### Event sourcing

Event Sourcing es un teérmino reciente, pero append only datamode is nothing new, de hecho transaction in databse (transaction logs). Se construy encima del concepto de transactin log, la distinción, es que son realmente eventos lógicos. La manera en la que escribimos la lógica debe ser tenida en cuenta y como un replay. transaction log tiene que ver solo con infra.

> Pone el ejemplo del inventario, pero no es event source. Es más transaction log.

event store definir a proyection para que puedas calcular SUM y tal. Pero no ve que la BD relacional tenga un weakness.

No empezeis a utilizar como silver bullet. Ver para cada parte que funciona mejor.

**Search, filter and group by any column (like Excel)**: Workaround, not true requirements... normalmente lo que quieren son reports... algo que podemos hacer que cubre el 70% -> la lista of top x files u opened it... porque normalmente, muchas veces vuelven a consultar lo mismo. Sino hacer un data-dump... y ahí igual no hace falta tenerlo joineado.

Usuarios se han spoiled by google... igual usas full text search o lucene, eso funciona bien para un cierta cantidad de data... luego deja de funcionar. typeas smith y devuelves 10000 smiths... puede ser dificil dar un google like experiencia, lo que el usuaario quiere releventa result. cada vezque buscan y pinchan en el resultado, tendriamos que incrementar el relevant score... a menos que hagan click back... si buscar bar en google en casa, igual te dice para montar un bar o el bar de física, si buscas en el movil igual los bares cercanos.

### Engine pattern

disscount, risk, fraud... calculation involve a large number of factors. sobre varios servicios sin romper services boundaries.

IT/ops esta muy involucrado. Alguien este en el equipo del engine y en el del dominio. Como rules engine, pero las reglas no tiene por que estar en el rule engine.

the engine provide hooks, en el cual los servicios meten sus plugin. No llama el a los demás servicios.

> GRAFICO de mi cuadero

plugin model.

No importa el orden de cual se aplica, son independientes.

Combination of data, personal disscount... puede ser que te parezca que necesitas combinar datos, pero si haces el análisis necesario puedes descubrir que no. Trabajar con test cases para acordarlo con negocio.

take the output of the engine y pasarla como entrada no rompe los services boundaries. o dentro del egine, que partes de distintos servicios tomen parte en la siguiente calculo, no es problema. igual podemos crear otro modelo estadistico que es igual de bueno sin romper boundaries.

¿Como se le llama? a veces con un evento (order accepted -> trigger the engine pattern to calculate price) pero normalmente dentro se llama desde un servicio dentro del proceso de checkout por ejemplo.

fraud como servicio no le cuadra mucho, no es un bussines capability... hariamos que los otros esrvicios publique cosas que normalmente no tendrian que publicar...

se puede ver como un pipeline (price), o lanza eventos distintos dpend del resultdo (fraud)... depende del dominio. En price tiene sentido devolver un resultado para usarlo, en fraud no tanto.
normalmente los retries y tal, queue, no aporta mucho aqui.

a veces tiene logica el engine... separar lo que es puramente algoritmico y ques data bussines oriented.

the code is part of it/ops, but maybe in a separate area. provee interfaces y diferentes plugin proveen implementaciones, tipo a meter varias patron strategies.

* [Microservices and Rules Engines – a blast from the past](https://www.youtube.com/watch?v=Fuac__g928E)
http://udidahan.com/2009/12/09/clarified-cqrs/
http://udidahan.com/2011/04/22/when-to-avoid-cqrs/
http://udidahan.com/2011/10/02/why-you-should-be-using-cqrs-almost-everywhere/
http://udidahan.com/2012/02/10/udi-greg-reach-cqrs-agreement/

## SOA: Operational aspects

### Despliegue

https://octopus.com/
https://octopusadsd.particular.net/app

Sistemas compuestos de multiples paquetes nuget, y paquetes nuget usados en multiples sistemas.

La diferencia con microservios y usar paquetes nuget, tengo que actualizar en todos . La wakness si cambio un microservicio rompo todo. Con nuget puede ir actualizando distintas partes, por ejemplo solo el backend primero o solo en un servicio.

Cuando creamos una versión del paquete NuGet sabemos que sistemas hay que tocar. Con este [script](https://gist.github.com/dvdstelt/2528b08ce2476e73e009433f419d632e) podemos hacerlo en Octopus.

puede tener una cola o varias para cada AC, no es requisito tener varias.

Tener varias partes es más complejo que tener un monolito.

**Naming queues y processes**
NameService.NameBC.NameAC
El nombre de AC muy relacionado el tipo de mensaje o a la acción que responder. Billing.MonthyInvoice.OrderAccepted. Para los de front relacionado con el use case Customer.OnlineSystem.Search. Si es un front end component el nombre del sistema es usualmente parte. Para windows service/queues meter antes el nombre del proyecto.

### Monitorización

Ventajas de queue system. visibilidad del numbero que mensajes que las colas ingestan. Error queue notifies admin of problems. No preocuparse de las expceciones, mirar la cola de errores.

* Nos ayuda a identificar cuellos de botella: Performance counters. \# of messages y el throughput (msgs/s) podemos saber cuanto va a costar procesar un mensaje en cada cola e incluse predecir cuando vamos a romper nuestro SLA.

Si tenemos una cola compartida es más complicado, usar la cola compartida para los mensajes que sean críticos.

### Escabalabilidad

Competing Consumers pattern: Podemos tener varioss servidores cada uno corriendo una instancia del mismo componente autónomo.

Usar un distributor, similar a un load balancer, cuando un new worker se pone online. sender side disbitrution aun más escalable (dsix??)

> Dibujo

**Virtulización**
Ventajas de contendores, puedes conectar la monitorización con el sistema de escaalado. Podemos provisionar o desprovisionar más contendores cuando necesitemos más o menos recursos. Sistemas autoescalables, que funcionan bien en cloud servers.

en queue bases env puede escalar mejor separdos, que con microservicios poruqe están más relacionados...

### Fault-tolerance, backups, recuperación de destrastres

**Fault tolerance**
En un entorno vritualizado el disco duro C esta relamente almacenado en un fichero on the SAN. Asi que en un docker los mensajes estaría en ese fichero y podriamos recuperalo. Por lo que ya no necesitamos un broker centralizado para ser reliable. Aunque perdamos un ACI no pasa nada.

**Backups**
Si usamos SAN, si tienes las colas en SQL Server, con tener backups de BD sobra.
Si usamos SAN, tanto los datos como la BD estara ahí. Hacer snapshots del SAN para tener un fully consistent backup. Usar el snapshot to disaster recovery a un point of time, tmb para data incosistency. Frequency of snapshot depends de cuanto tiempo estemos dispuestos a perder y el precio que cuesta mover esos datos cada X minutos.

### Versionado
Colas proveen temporal separation.

Estrategia back to front: DB -> Server -> Message (contract) -> Cliente

Si lo hacemos de esta manera evitamos garbage data, porque vamos más seguros, lo hacemos con más cuidado.

Zero-downtime upgrades. install v2 mientras sigue v1. los dos alimentamos el mismo mensaje. Y cuando vemamos que no hay errores, quitamos la v1, todo con scripts automáticos en la CI. consideer hooking into CI.

Componentes autónomos nos permite:

* Flexibilidad
* Escalabilidad
* Monitorización
* Versionado (nuget pakcages version)

## Sagas/Long running bussines processes modelings

**¿Que en un proceso?**
Un proceso puede ser descrito como un conjunto de actividades que son performed en una cierta secuencia como resultado de triggers externo y externos.

**¿Que es un long-running process?**
A long-running proceso es un un proceso cuya ejecución lifetime excede el tiempo para procesar un single external evento o mensaje. Intermediano long, no sabemos cuando el siguiente trigger va a llegar.

Long-running significa que multiples event/trigger externos son manejados por la misma instancia de un proceso - es *Stateful*.

ejemplo de transferencia, el dinero sale de uno, pero aun no está en el otro... hay varios intermediarios... y el estado de compensacion si no se puede hacer, no vuelve todo el dinero... por fesss.

Tiene un state management facility that enables a system to encpatusalte the logic and data for handlign an external stream of events.

Think as Object Oriented progrmaming, ecapsular behaviour y se invoca con triggers (métodos) que modifican estado.

Integration example. en vez de llamar a los demas distintos, lo invertimos y lo hacemos el punto central.

> Meter dibujo

### Sagas

Los triggers son mensajes.
Similar a message handerls: pueden manejar un número diferente de tipos de mensajes
Difierencia con message hanlders: tienen estado, message handerls not.

Find existing Saga por id y la procesando hasta que todos los mensajes se complenten. Normalmente, después de long-runing process se publicara un evento o mensaje. En general no queremos que la saga modifique directamente un master data. Está enfocada en procesar estado (SRP). En general podemos borrar los estado intermedios cuando finalice.

**Integration example. Request-response**
Mandamos shipordermessage, itops habla con fedex o lo que esa y devuelve respuesta. podria llamar a varios servicios para ver cual es el más barato para este order por ejemplo.

```csharp

public class OrderProcessingSaga : Saga<OrderProcessingSaga.MyData>, IAmStarteByMessage<OrderMessage>,
IHandleMessage<ShipOrderResponse>
{
    // este lo pone como internal class, ya que nadie más que la saga deberia usar esos datos
    public class MyData : ContainSagaData { }

    protected override ConfigureHowToFindSaga(SataPropertyMapper<MyData> mapper) { }  

    public Task Handle(OrderMessage message, IMessgageHandlerContext context)
    {
        // aqui el bus coge el saga id y tipo hace un enrich del mensaje
        return context.Send(new ShipOrderMessage());
    }

    public Task Handle(ShipOrderResponse message, IMessgageHandlerContext context)
    {
        MarkAsComplete();
        return Task.CompletedTask;
    }
}

public class ITOpsFedExIntegrationThing : IHandleMessage<ShipOrderMessage>{
    public Task Handle(ShipOrderMessage message, IMessgageHandlerContext context){
        // call FeDex
        // el correlation id es relleando para el ShipOrderMessage, ¿coom se conecta con el saga id? aqui como copia los saga header de la request a la response, para que llega a la misma instancia de la saga
        return context.Reply(new ShipOrderResponse());
    }
}
```

Depende de la tecnologia igual puedes hacer un lock de la saga.

**Event driven saga**

Puede ser empezada por aceptada o billed, pero si esta  billed ya busca no por aceptada. Este tipo de saga son tipicos como replacement de los reports.

```csharp
public class ShipOrderSaga : Saga<ShipOrderSaga.MyData>,
IAmStarteByMessages<OrderAccepted>,
IAmStarteByMessages<OrderBilled>
{
    public class MyData : ContainSagaData
    {
        public Guid OrderId {get;set;} //unique
        public bool Accepted {get;set;}
        public bool Billed {get;set;}
    }

    protected override ConfigureHowToFindSaga(SataPropertyMapper<MyData> mapper)
    {
        //Debemos garantizar uniqueness para el orderid property (esto NServiceBus lo hace by dfeault), para que no cree dos sagas con el mismo OrderId si llegan los mensajes a la vez

        //lookup si encuentra la saga que tenga orderid como el de billed
        mapper.ConfigureMapping<OrderBilled>(m => m.OrderId).ToSaga(s => s.OrderId);
        //si no encuentra busca por aceptada
        mapper.ConfigureMapping<OrderAccepted>(m => m.OrderId).ToSaga(s => s.OrderId);
        //sino, creará una nueva
    }  

    public Task Handle(OrderAccepted message, IMessgageHandlerContext context)
    {
        Data.OrderId = message.OrderId;
        Data.Accepted = true;

        return ShipIfPossible(context);
    }

    public Task Handle(OrderBilled message, IMessgageHandlerContext context)
    {
        Data.OrderId = message.OrderId;
        Data.Billed = true;

        return ShipIfPossible(context);
    }

    Task ShipIfPossible(IMessgageHandlerContext context){
        if (Data.Accepted && Data.Billed)
        {
            MarkAsComplete();
            return context.Send(new ShipOrderMessage());
        }
        return Task.CompletedTask;
    }
}
```

**Time component**

Necesitamos una manera de gestionar tiempos, y necesitamos que sea durable y transactions.

```csharp
public class ShipOrderSaga : Saga<ShipOrderSaga.MyData>,
IAmStarteByMessages<OrderAccepted>,
IAmStarteByMessages<OrderBilled>,
IHandleTimeouts<OrderAccepted>
{
    //...

    public Task Handle(OrderAccepted message, IMessgageHandlerContext context)
    {
        Data.OrderId = message.OrderId;
        Data.Accepted = true;

        RequestTimeout(context, TimeSpan.FromHours(24), message);

        return ShipIfPossible(context);
    }

    public Task Timeout(OrderAccepted message, IMessgageHandlerContext context)
    {
        //if the behind some queue have defer sending of messages, le decimos que envie el mensaje después de x tiempo
    }

    //...
}
```

Si algo es crucial que se ejecuta a las 12, necesitas algo de infra que te asegure que se esta ejeuntado a esa hora, porque los tiemout no te solucionan esto. Puede que tengas el servicio tirado 3 días.

¿Que pasa si es 24 horas no tenemos respuesta de FedEx p.ej.? ¿Y no podemos enviar los productos? Igual podemos llamar a otro, hay que veces que después del tiemout hacemos otra cosa. Luego puede que fedex te de ok, e intentes cancelar post us, pero ya la haya dado el ok tambien... Al final hay que llevarlos a bussines, SLA con FedeEx que nos garantize respuesta, y entonces si ha pasado 24h no lo enviés...

Scatter gather pattern: https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html

http://udidahan.com/2009/04/20/saga-persistence-and-event-driven-architectures/

### Exercise: saga design

No pasa nada por guardar datos en la bd como si fuese entidades en las sagas. Los datos de las sagas no son accesibles desde otros boundaries.

General rule of thumb/dumb: don't try to model the real world, it does not exists :) phsycally realitty is almost indicendatally las responsabilidad. en amazon cuando entras en incongito eres un visitor, no un customer... a customer en reality un customer class... te pondrá en dirección incorrecta.

El concepto de status es muy peligroso! casado, fallecido, favorito... en la mayoría de dominios nos puede dar problemas. Sospecha si una entidad tiene un campo llamado estado.

Es fácil usar los distintos bloques (la implementación) de las sagas, lo complicado es analizar los procesos de negocio para identificar cuales deberían ser los pasos, los services boundaries...

Cuando usamos un sistema legacy o 3rd party usamos una estructura similar a arquitectura hexágonal/port & adapters.
> pintar imagen

**Orquestación no es un servicio en sí mismo.**

Divide los workflows/orquestadores a través de services boundaries.

* Eventos son publicados al final of the sub-flow en un servicio
* Eventos triggers a sub-flow in other services

Sagas pueden ser usadas para CEP(complex event processing)/ESP(event stream processing).

high-encapsulated, loosley coupled. las sagas son una buena unidad para unit testing. especialmente las de time-bound. es facil hacer un diagrama de sucuencia y luego hacer TDD de la saga.

Use message building block to support long running processes.
Si en batch processes usas las mismas datos que usan en front, eso es coupling! igual cambian un modelo y accidentalmente fasitidias un batch job. SRP. Con sagas. batch job great places to use sagas.

Manten servicies boundaries explicit.

## SOA: modeeling

### Domain models

No siempre ha existido el concepto. El EAA fue el librio que cambio todo.
https://martinfowler.com/eaaCatalog/domainModel.html

Esta compuesto de:

* Propiedades
* Métodos
* Eventos

**¿Cuando usarlo y cuando no?**
“If you have complicated and everchanging business rules...”
“If you have simple not-null checks and a couple of sums to calculate, a [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html) is a better bet”
-- Martin Fowler EAA

Es decir, no hay que usarlo en todo el bussines layer. Pero tiene un nombre tan bueno... no todo tiene que ser un domain model.

Deben ser independientes. No depdender de UI, de BD (tampoco inyectando repositorios), ni de comunicaciones. Ya que encapsulan rules. Separation of concerns.

Domain models hecho de [POJO's](https://martinfowler.com/bliki/POJO.html) no cuadran nuestra definición de domain model de EAA (puede que la Evans si). Por eso, si no tienes un reglas de negocio complicadas y que cambien, esta bien tener un conjunto de POJOs que nos ayuden a encapsular datos para persitir y recuperar los datos. Salio el patrón [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html), pero es que no tiene porque serlo, esto son solo POJOs que quiero persistir, pero realmente nunca debiese haber sido un domain model.

Son highly testable:
- Register for events
- Call a method
- Check properties & events args

Testearlo como black component en nuestro tests. people try to TDD their domain models...

Si los escribimos así, hay probabilidades que al cambiar nuestro domain model, rompamos el test y tengamos que modificarlo.  

```csharp
[TestMethod]
public void CreateOrderShouldConnectToCustomer() {
    Customer c = new Customer();
    Order o = c.CreateOrder();
    Assert.AreEqual(c, o.Customer);
}
```

Domain models pueden ser desplegados multi-tier. Nadie dice que sólo pueda ser uno. diferentes tipo de domain models para diferentes capas. Incluso no tiene porque ser escritos en diferentes lengaujes Js en front, c# en back, incluso t-sql en la bd.

para algún nightly bacth de integation dentro de nuestros sistema igual tiene más sentido hacer difectamente con procedimientos en SQL, porque el contexto es distinto:

* Otras partes de la organización ya esta usando estos datos, por lo que si los rechazamos ya hay gente que lo usa...
* Tiene mejor rendimiento, line by line REST API...

Massive file import, use database features to import, bulk. insert. ¿y para la lógica de esos datos que entran? entonces tiene más sentido tener el domain model en la BD... clean data, flag data as error...  es mover un script de 1mg vs mover 1tb de data... las pones en una tabla separadas, o quitas el flags y las integras con el resto de tu producción data... la lógica para ese contexto es distintas de transaction processing...

### Concurrency models

* Optimista (first one wins, last one wins, either way someone loses data)
* Realistas
* Pesimistas (alguien hace el lock y se va de vacaciones...) everyone has to stand in line and each data update is dealt with one at a time

Hemos asumido que las transacciones. Beware of the false sense of security of wrapping things in a database transaction. By default, transactions operate in READ COMMITTED mode, so if the transaction does two reads, some computation, then an update, the things being read can be updated by other transactions before your transaction ends. CUando los ORMs se han vuelto populares, cada vez pensamos menos en como se comporta la bd... ni pensamos si necesitamos distintos modelo de isolación...

#### Concurrencia realista
Get current set of data before changing anything.

> Get domain object (just ONE)
> Ask it to update itself

```csharp
public void Handle(MakeCustomerPreferredMsg m) {
    Customer c = s.Get<Customer>(m.CustomerId);
    c.MakePreferred();
}

// MAL
public void MakePreferred() {
    foreach(Order o in this.UnshippedOrders){
        foreach(Orderline ol in o.OrderLines){
            ol.Discount(10.Percent);
        }
    }
}
```

**Las relaciones entre entidades nos exponen a posibles inconsistencias!!**
Otras transacciones pueden venir y hacer cambios sobre esas entidades.

Si hemos seguido services boundaries, casi no tendremos relaciones. Entities BAD, entities relationships WORSE.
Para single non-colaborative system esta bien usar entities, pero casi nadie ya construye algo así...

Algunos cambios pueden ser concurrentes:
- Tu cambia la dirección del cliente.
- Yo actualizo el credit history del cliente.

Otros no:
- Tu cancelas el pedido.
- Yo intento enviar el pedido.

En estos, o ganas tú o yo, pero no ambos. Estan en distintos servicios, no podemos lanzar una transaccion sobre ellos... puede que pensemo que están mal las boundaries... intentar aguantar la tentación de volver a meterlo todo junto y romper las boundaries (aunque esa fácil). Si llegamos a una problema de estos de race condition -> CQRS

In CQRS commands no fallan (pero aquí no podían ganar ambos...). Realmente race conditions no existen en bussines, un microsegundo no debería cambiar los objetivos de negocio.

Buscar los underlying bussines objectives. Usar los [5 whys](https://en.wikipedia.org/wiki/Five_whys)
Reglas:

1. No se pueden cancelar pedidos enviados
 Why? xk shipping cuesta dinero
 So? That money would be lost if the customer cancelled  
 Why? Because we refund the customers money
2. No enviar pedidos cancelados

Ahora vemos que el problema esta en Refund policies... otro servicio... el problema es dinero.

Cuando un pedido es cancelado, el refund tiene que ser inmediato? No...
Podemos hacer refund parciales? Si...
Entonces podemos devolver el dinero del pedido - el shipping más tarde... así no nos preocupamos de race condition.

Realmente cuando lo expresan igual están pensando en un día... y nosotros como ingenieros los convertimos en invariantes.. y hacemos que verdad al microsegundo...

¿Que tiene que hacer el customer para conseguir el refund? Devolver los productos. 
Simpre intentar entender la probablity distribution over time of a given option.

> grafico cuaderno https://en.wikipedia.org/wiki/Buyer%27s_remorse

¿puede que tenga sentido esperar pra publicar el evento ORderAccepted? ya que igual a los 30 minutos se producen la mayoría de cancelaciones... incluso se puede mejorar si un cliente nunca cancela le damos menos, si suele cancelar esperamos más, si es un producto que compro en el pasado...

Al final vemos que tenemos a time-bound process, que podriamos modelar como una Saga. Sagas son muy testables. Saga single object, no teniendo refencias... deberian ser pojoness. son buenos candidatos para seguir el domain model pattern, más que las entities. una de las mejores implementacion de agreggate root. presevan consitency boundaries. 

Les llama policy, refund policy o remorse policy. En soa nos enfocamos en modelar bussines capabilities y bussines policy. realmente en el mundo real veras que hay muchas policies... en vez de diseñar bussines procesos o bussines entities, ya que realmente true bussines world son más policies centry. capbabilities by services, y policies by sagas.

La implementación de las sagas es similar a los actores. Pero no hemos intentado construir un actor... hemos venido de collaboriativde domains, commands can't fail, dig deeper in domain... y hemos llegado a este punto.

## Organizational transition to SOA

No hacemos big bang rewrite, no funcionan porque no tienes suficiente tiempo. Un importante valor en rewrites, pensar en que tenemos que hacer algo diferente y hay que pagar por ello... no intentes hacer SOA from scratch. rewrite tax, vas a pagar este tax in feature by fauturew rewrite batches... en vez de big rewrite.

Pensar si la organización esta preparada para pagar la siguiente fase de tax. Cada feature costará el porcentaje de tasa más... para pasar a la siguiente fase, debemos estar preparados para pagar la tasa. el team se enfoca en que necesita para la siguiente fase.

| Phase|Ops |Dev|Team|Duration| Tax |
|---|---|---|---|---|---|
|I|CI/CD-  *Med*|Unit testing - *Low*|Internal training (lunches, user groups) in good practices y conceptos de mensajeria y composite UI|3-9 months|5%
|II|Source control- *Med*|*Med*|SOA|9-18 months|10%|
|III|*Med*|*High*|SOA Domain Expertise. Team structure discussion|12-24 months|30% *(lo que nos cuesta 3 meses, nos costará un mes más!)*|
|IV|*Med*|*Even Higher*|Reorganization teams|12 months-never|30%|

**Phase I:** Around of each use case you generate an event. Aqui no metemos muchas cosas de queues, como mucho mandar un email, o usar la bd como queue...

**Phase II:** Creamos subscriber dondoe aporten valor, en algunas features... no hacer big rewrites. samll refactoring, samlla composite UI... en Ops hay que empezar a crear colar y tal...

**Phase III**: vas a necesitar tiempo para restabailizar... no sabes donde romper... sacar partes enteras (SOA). Empezando a enfocarnos en análisis de services boundaries. rewrite son más atractivos, pero casi nunca tienen éxito... report replacement. si el BBOM es the un 3rd party ?? experiencia para saber como de terrible es ese 3rd party, que limitaciones y constraints tiene... hasta que sea visible es díficil, es parte política...

**Phase IV**: Ahora tenemos que atacar la bd monolítica. data migration no es nunca tan fácil como parece. migrate code is hard, migrate data is even harder. prepara solid group of people para la data migration. pero el desarrollo de nuevas features será más sencillo. Aqui ya tenedremos varios repos, con varios CI, nuget packages...

## Web Services and User interfaces (58m)

### Caching

#### In process caching

Inprocess caching usually the first silver bullet cuando los usuarios dicen que va lento.
Keeping the cache up to date acrross a server farm is complicado.
A menudo requiere "sticky sesions" -> el problema con esto es que undermain the load balancer. no sabemos cuales van a ser muy activos y cuales son.

#### Distributed caching

No es un silver bullet. No todos los datos están guardados en cada nodo. Beware the tempation of loop!!! intermanete una cache hara un network para buscarlo en otro, puede que haga muchos roundtrips para deovlverte los datos. A veces añadir más máquinas hace más lento el sistema :(. si tienes que hacer join con la cache... sabes que tecnologia es buena haciendo joins!? SQL!

Es bueno para buscar un valor a través de una key.  

tip: wrapper arounda cache, y log warning si hace más de x llamadas...

nunca vamo a tener suficiente memoria para cachear todo. 

cache invalidation.  reads can interfere with writes and that hurts perfomance, más que si no hacemos ningún tipo de cache.

Si empiezas a cacher por usuario, es peligroso... la probabilidad para encontrar algo en cache, decrece. solo cachear static data y no por usuario o por request. !!!hit rate??? mirarlo https://scalegrid.io/blog/6-crucial-redis-monitoring-metrics/
y el response time de la cache... a veces con distributed, y round trips... es más lento que ir a la BD!!!!

### CDN (Content Delivery Networks)

serie de servidores espred internet avaiable to u pay as service to server static content, js, css, images... they propagate this files close and close u your users.

Si tienes usuario por todo el mundo, en vez de hacer request to tu servidor las devuelves el CDN. cuanto más spread sea tu publico, mejor funciona. pero si todos tus usuarios están en todos londres y el CDN no tiene servidores en londres pues igual es peor.

http://acme.com/imgs.jpg

creas un subdomain
http://content.acme.com

y rediriges eso a un CDN, the netwowkr rerouting is behind the scenes. you pay for bandwith. normalmente son very cost effective para mejorar la experiencia de usuarios.

page output caching, no pensar en toda la página, pensar en los más volatiles y. No ir al servidor todas las veces cuando hay cosas que casi no cambian. por ejemplo en Amazon el listado de departamentos. Diferentes granularidades de staticness behind the scenes. si preguntas a bussines cuando lo puedes cachear no van a saber... xk ellos les gusta que cuando lo cambies en backedn aparezca directamente... si te doy 100€ para mejor rendimiento cual querrias que se referesque más y cual menos, y hazme una lista... the static version of a dynamic web, pensar como si fuesen ficheros... es cacheado por el browser, incluso por el ISP, y por el CDN... si no está en uno buscaría en el otro intermediario hasta llegar a nuestra web sino lo encuentra, cuando más usuarios más se speradea por internet.

Another problema, request in parallel (dependes on the brower), 16 parallel request per domain. Si alguno de ellos es servido por intermediarios, tenemos respuesta antes y nos llega menos request al servidor.

### Personalization

Ex. the weather widget:
- Look up location by IP
- Look up weather by location

2 back-to-back remote calls to costly 3rd party services.

Si son return visitor xk tengo que buscar siempre la IP, lo normal es que si vives en Londres cuando vuelvas estén en Londres aunque la IP sea distinta... cache user location así no tenemos que pedir cada vez... -> cookies or browser local-storage.

El weather el london no cambia por user request, podemos cachearlo. Una cosa que podriamos hacer es evita query strings, usar slash. con http://weather.acme.com/london internet lo cachea, sin embargo http://weather.acme.com?q=London con query stirng no. cuando tiempo deberiamos decirle ainternet que cachee este recurso? seguro que no menos de una hora, lo cacheamos 60 minutos. Un millon de usuarios algunos llegaran a nuestro servidor, pero otros muchos usaran caches...

breaking things down to the smallest components.

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
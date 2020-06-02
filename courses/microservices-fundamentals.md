# Microservice fundamentals

## ¿Qué son los microservicios?
Son una idea poderosas, pero traer mucha complejidad. Arquitectura monolítica debe ser by default.

Balance entre collaborate e independiente.
Independently deployable services that work together, modelled around a business domain

organizar entre equipos
stable boundaries is quite important.

¿tamaño? el concepto de tamaño varia con el tiempo, que es grande o pequeño para ti... más servicios no es necesariamente más bueno. No copies lo de los demás, no tiene sentido.

Los microservicios son un tipo de SOA.

Pueder usar REST, RPC, eventos...

downsides***
cognitive overload
testing becomes difficult -> cross service testing
https://learning.oreilly.com/library/view/building-microservices/9781491950340/ch07.html#testing-chapter

Es más complicado ver donde están los problemas (en un monolito solo tengo un sitio donde mirar)
Monitoring es más complejo. Ponemos mucha complejidad en las operaciones.
Resilencia no es gratis. Hacer la aplicación robusta es más complicado. Si un servicio no responde tengo que hacer retries, pero puedo back pressure...
Y los problemas fundamentales:

* Transacciones distribuidas
* Consistencia eventual
* CAP theory

Igual no el primer día, pero al final vas a tener problemas con esto.

https://learning.oreilly.com/library/view/monolith-to-microservices/9781492047834/ch05.html#growing-pains-chapter

## El problema con el acoplamiento



## Domain-driven design

Vertical slicing systems with bussines functionalities logic.

bounded context. un dominio puede tener varios bunded contexts.

![alt text](https://martinfowler.com/bliki/images/boundedContext/sketch.png "Bounded Context")

https://martinfowler.com/bliki/BoundedContext.html


Intenta evitar cache, porque los hace más rápido, pero más dificil de pensar sobre ello.

Puede haber mS sin bd, que solo cojan datos de otros servicios p.ej.

Mapear bc to microservices es un buen starting point.
Getting fine-grainend y haciendo mS por cada uno de los agregados.
acaba haciendo mS por agregado? keep agregates intact accross boundaries.

## Referencias
https://martinfowler.com/articles/microservices.html
https://learning.oreilly.com/library/view/microservice-architecture/9781491956328/
https://samnewman.io/talks/rip-it-up/

https://samnewman.io/talks/confusion-land-serverless/

https://samnewman.io/blog/2015/04/07/microservices-for-greenfield/

https://philcalcado.com/2017/03/22/pattern_using_seudo-uris_with_microservices.html

https://samnewman.io/offerings/videos/securing-microservices/
https://learning.oreilly.com/library/view/the-devops-handbook/9781457191381/
https://teamtopologies.com/
https://martinfowler.com/articles/micro-frontends.html

https://www.slideshare.net/adriancockcroft/evolution-of-microservices-craft-conference
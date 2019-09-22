# [Deliveritis aguda](https://www.youtube.com/watch?v=vGCowJY5QCQ)

El objetivo de un CTO es ganar pasta. Y la mejor manera de hacerlo es con el delivery.

## Delivery first

Es una actitud. Foco en el delivery y no en la tecnología.

¿Cómo consigo que esta tarea/user story/proyecto suba antes a producción?

### 1. Definition of Done

Una funcionalidad está acabada (done) si está en producción, la puede ver/utilizar algún usuario y el sistema sigue estable.

### 2. Estrategia de control de versiones

De gitflow/github flow/custom flow a Trunk Development con Toogle Feature.

No complicarse la vida con gestión de ramas (sobre todo en equipos pequeños). Hacer la gestión más lean posible.

### 3. Elegidos para deployar

>Si eres valiente para escribir código que va a producción también los eres para subirlo. Conviértelo en un acto no heroico a través de automatización y testing.

Todo el mundo despliega, hasta el más junior.

### 4. Optimizar las user stories

Como developer identificar el coste de las partes y hablarlo con el PO. Muchas veces el 20% se una feature se lleva el 80% del coste de implementación.

- **Caso A y Caso B.** P.ej. no unificar oidc de FB y Google en la misma historia. Son dos distintas.
- **Segmentación por usuarios.** No es lo mismo a nivel de escalado probar para un % pequeño.
- **Segmentación por plataformas.** No pasa nada por sacar una feature sólo en una plataforma y medir su uso.

### 5. One click deploy y rollback

Proceso de deploy automatizado pero no fliparse con el hype (k8s, microservicios), ya que no siempre aplica. No es lo mismo el estilo de juego para un equipo de tercera que para uno de champions.

### 6. Cuellos de botella

Equipos multidisciplinares, todas las features necesarias para subir una feature a producción viven en el equipo, pero no en personas físicas.

- **QA sin QAS.** Muchas veces es cuello de botella, quitarlos. El equipo hace el testing, cross-testing, como se quiera.
- **Scrum sin SM.** La figura de Scrum Master rotando dentro del equipo.
- **Arquitectura sin arquitectos.** El skill de arquitecto está en todos los devs (más o menos desarrollado)
- **Todos atacan /todos defienden**

### 7. Atacar el Sprint

En muchos equipos, si hay 10 historias, 10 personas, una persona está trabajando en cada tarea... **hay que trabajar en la historia con mayor proiridad la mayor gente posible sin que se moleste.**

El objetivo del equipo entregar la tarea con mayor prioridad aunque trabajes en otra historia Los stand-up deberían ser de decir si necesito ayuda para la tarea con mayor prioridad, y sino trabajo en ella, como puedo ayudar en la tarea con mayor prioridad.

### 8. RETROs

¿Como podemos subir a producción más rápido, más frecuentemente y con mayor calidad?

Referencias:

- [Accelerate: Building and Scaling High-Performing Technology Organizations](https://www.goodreads.com/book/show/35747076-accelerate) por Nicole Forsgren, Jez Humble, Gene Kim.
- [Feature Branches And Toggles In A Post-GitHub World](https://samnewman.io/talks/branching-and-feature-toggles/) por Sam Newman
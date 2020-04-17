# Accelerate: The Science of Lean Software and Devops: Building and Scaling High Performing Technology Organizations

- [Accelerate: The Science of Lean Software and Devops: Building and Scaling High Performing Technology Organizations](#accelerate-the-science-of-lean-software-and-devops-building-and-scaling-high-performing-technology-organizations)
  - [Acelera](#acelera)
  - [Midiendo el rendimiento](#midiendo-el-rendimiento)
    - [Rendimiento a través de la entrega de software](#rendimiento-a-trav%c3%a9s-de-la-entrega-de-software)

## Acelera

El software es un diferenciador clave para que las organizaciones puedan entregar valor a sus clientes y stakeholders.

Los ejecutivos suelen sobrestimar cuanto DevOps se hace en la organización.

Pon el **foco en las capacidades** más que en un modelo de madurez, ya que es un un proceso de mejora continúa. Las mejores organizaciones nunca se consideran lo suficientemente "maduras".

Cada equipo tiene su manera de hacer las cosas, en que capacidades enfocarse dependerá de eso. No funciona lo mismo para todos.

Las industrias cambian constantemente, lo que hoy funciona y es bueno, puede que no lo sea dentro un año.

Las investigaciones muestran que nada de lo siguiente puede predecir el rendimiento:

- Antigüedad y tecnología usada (p.ej. mainframe vs greenfield)
- Si operaciones o desarrollo realiza los despliegues
- Si hay implementado un Change Approval Board (CAB)

## Midiendo el rendimiento

Las líneas de código (LOC), la velocidad o la ocupación no son una buena manera de medir el rendimiento del equipo.

- **Líneas de código**: Más lineas de código no equivale a mejor software. Lo ideal sería utilizar el menor número de líneas posible para resolver el problema (mejor si no hay que escribir ninguna). Pero pocas líneas tampoco es una medida ideal, demasiado pocas puede influir negativamente en la legibilidad (p.ej *one-liners*).
- **Velocidad**: La velocidad es algo relativo y dependiente de cada equipo, no es una medida absoluta. Cuando sea usa para medir la productividad puede hacer que los equipos inflen los puntos de las historias, se enfoquen en completarlas y no en colaborar con otros equipos... La velocidad fue diseñada para ser usada para planificar la capacidad, no para medir la productividad del equipo.
- **Ocupación**: Puede ser útil hasta cierto punto. Pero una vez la ocupación llega al 100% no hay capacidad de reserva para trabajo no planificado (bugs, cambio de prioridades, etc).

### Rendimiento a través de la entrega de software

Una manera exitosa de medir el rendimiento debe tener dos características claves:

- Poner el foco en **resultados globales** y asegurarse que los equipos no se enfrenten unos a otros
- Poner el foco en **resultados no en salidas**. PUede haber gente trabajando muchas horas pero que realmente no alcance los objetivos.

Medir el **rendimiento a través de la entrega de software** cumple estos dos criterios. Para ello se han establecido cuatro métricas:

- Tiempo de espera (*Lead time*)
- Frecuencia de despliegue
- Tiempo medio de reparación (*MTTR - Mean Time To Restore*)
- Porcentaje de cambios con fallos

**Lead time**: En términos Lean el lead time es el tiempo que pasa desde que un cliente hace una petición hasta que esa petición es satisfecha. Tiene 2 partes: el tiempo de diseñar y validar la feature a desarrollar y el tiempo de entrega esa feature al cliente. Esta última parte (el tiempo que cuesta desarrollar, testear y entregar) es más fácil de medir y tiene menos variabilidad. Son mejores ciclos de entrega más cortos, ya que nos proporcionan feedback temprano sobre lo que estamos construyendo y nos permiten reconducir más rápido. Se mide como *"el tiempo que cuesta desde el que el código es commiteado hasta que el código está corriendo correctamente en producción"*.

**Frecuencia de despliegue**: Reducir el tamaño de los lotes (batchs) en otro elemento central del paradigma Lean. Se utiliza la frecuencia de despliegue como proxy para medir el tamaño de los batchs, ya que es más sencillo de medir y con menos variabilidad. Entendiendo despliegue a entorno de producción o a un app store.

**Tiempo medio de reparación**: Tradicionalmente, la fiabilidad se media como el tiempo entre fallos. Hoy en día, con productos complejos que cambian rápidamente, el fallo es inevitable. Así que lo que se mide aquí es cuanto tiempo se tarda en restaurar el servicio después de un incidente (interrupción no planificada, deterioro del servicio...).

**Porcentaje de cambios con fallos**: Una métrica clave cuando haciendo cambios en sistemas es que porcentaje de cambios en producción (p.ej. release, cambios de configuración de infraestructura, etc.) falla. Entendiendo por fallar una degradación de servicio o que requieren posteriormente una reparación (p.ej. hotfix, rollback, parche, etc.).

En el estudio se observa que los equipos de alto rendimiento mejoran cada año y que impactan directamente en el rendimiento de la organización.

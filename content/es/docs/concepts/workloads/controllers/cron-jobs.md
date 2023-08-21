---
title: CronJob
content_type: concept
weight: 80
---

<!-- overview -->

Un _Cron Job_ ejecuta tareas, [Jobs](/docs/concepts/workloads/controllers/jobs-run-to-completion/), a intervalos regulares.

Un objeto CronJob es similar a una línea de un archivo _crontab_ (tabla cron). Ejecuta un trabajo de forma periódica
según un horario programado escrito en formato [Cron](https://en.wikipedia.org/wiki/Cron).

{{< note >}}
Todos los `horarios` de **CronJob** se basan en la zona horaria del máster donde se inicia el trabajo.
{{< /note >}}

Para instrucciones sobre cómo crear y trabajar con trabajos programados, 
incluyendo definiciones de ejemplo, 
puedes consultar [Ejecutar tareas automatizadas con trabajos programados](/docs/tasks/job/automated-tasks-with-cron-jobs).




<!-- body -->

## Limitaciones de las tareas programadas

Un trabajo programado crea un objeto job _como mínimo_ una vez por cada ejecución de su programación. Decimos "como mínimo" porque
hay circunstancias bajo las cuales dos trabajos pueden crearse, o ninguno de ellos se crea. Se intenta que estos casos sean residuales,
pero no pueden evitarse por completo. Por lo tanto, los trabajos deberían ser _idempotentes_, es decir, que se puedan ejecutar más de una vez con el mismo resultado.

Si el valor de `startingDeadlineSeconds` se establece con un valor grande o se deja sin especificar (por defecto)
y si el valor de `concurrencyPolicy` se establece en `Allow`, los trabajos siempre se ejecutarán al menos una vez.

Para cada CronJob, el controlador de CronJob verifica cuántas programaciones se han perdido desde la última programación hasta el momento actual. 
Si hay más de 100 programaciones perdidas, entonces no volverá a ejecutar el trabajo y registrará el error:

````
Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
````

Es importante señalar que si el campo `startingDeadlineSeconds` está configurado, es decir, no es nulo (`nil`), el controlador contabiliza cuántos trabajos perdidos se han producido desde el valor de `startingDeadlineSeconds` 
hasta el momento actual, en lugar de contar desde la última programación. Por ejemplo, si `startingDeadlineSeconds` es `200`, el controlador registra cuántos trabajos perdidos se han producido en los últimos 200 segundos.

Un CronJob se considera como perdido si no se ha podido crear en el momento programado. Por ejemplo, si establecemos el valor de `concurrencyPolicy` en `Forbid` e intentamos programar 
un CronJob cuando otro previamente programado todavía está en ejecución, entonces se considera como perdido.

Por ejemplo, imaginemos que un CronJob se configura para programar un nuevo Job cada minuto a partir de las `08:30:00`, y su campo 
`startingDeadlineSeconds` no está configurado. Si el controlador del CronJob no está disponible desde las `08:29:00` hasta las `10:21:00`, 
el trabajo no comenzaría porque el número de trabajos perdidos que se habrían acumulado en su programación sería superior a 100.

Para ilustrar este concepto de manera más clara, supongamos que programamos un CronJob para ejecutar un nuevo Job cada minuto, comenzando a las `08:30:00`, y establecemos el valor del campo
`startingDeadlineSeconds` en 200 segundos. Si el controlador del CronJob no está disponible
durante el mismo período que en el ejemplo anterior (`08:29:00` a `10:21:00`,) aún así el Job comenzará a las 10:22:00. 
Esto ocurre porque el controlador en este caso verifica cuántas programaciones perdidas ha habido en los últimos 200 segundos (es decir, 3 programaciones que no se han ejecutado), en lugar de hacerlo desde la última programación hasta el momento actual.

El CronJob es únicamente responsable de crear los Jobs que coinciden con su programación. Por otro lado,
el Job es responsable de gestionar los Pods que representa.



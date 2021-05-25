# Guía de Compatibilidad con Ivy

El equipo de Angular ha trabajado arduamente para asegurar, en lo posible, su compatibilidad hacia atrás con el motor de renderización anterior ("View Engine"). Sin embargo, en algunos casos, ciertos cambios pequeños fueron necesarios para asegurar que el comportamiento de Angular fuese predecible y consistente, corrigiendo problemas en la implementación de View Engine. En pro de que la transición fuese suave, hemos proveído [migraciones automáticas](guide/updating-to-version-10#migrations) para que tu aplicación sea migrada automáticamente por el CLI en donde sea posible. Habiendo dicho esto, algunas aplicaciones pueden necesitar aplicar algunas actualizaciones manuales.

{@a debugging}
## Cómo depurar errores con Ivy

En la versión 10, [se removieron algunas APIs deprecadas](guide/updating-to-version-10#removals) y hay algunos cambios importantes no relacionados con Ivy. Si estás viendo errores después de actualizar a la versión 9, lo primero que querrás hacer será descartar esos cambios.

Para hacerlo, [apaga Ivy temporalmente ](guide/ivy#opting-out-of-angular-ivy) en tu `tsconfig.base.json` y reinicia tu app.

Si aún ves los errores, no están relacionados a Ivy. En este caso, quizás quieras consultar la [guía general de la versión 10](guide/updating-to-version-10). Si has optado por alguna de las nuevas y más estrictas configuraciones de verificación de tipos (type-checking), es posible que también quieras consultar [la guía plantilla de verificación de tipos](guide/template-typecheck).

Si los errores desaparecieron, prende de nuevo Ivy removiendo los cambios en tu `tsconfig.base.json` y revisando la lista de cambios esperados que se describen a continuación.

{@a payload-size-debugging}
### Depuración del tamaño de la carga útil (payload)

Si notas que el tamaño del bundle principal de tu aplicación ha aumentado con Ivy, quizás quieras chequear lo siguiente:

1. Verifica que los componentes y los `NgModules` que quieres que carguen mediante carga diferida (lazy loading), solo sean importados en módulos lazy. Todo lo que importes fuera de un módulo lazy, puede irse directamente al bundle principal. Puedes ver más detalles del issue original [acá](https://github.com/angular/angular-cli/issues/16146#issuecomment-557559287).

2. Verifica que las librerías importadas hayan sido marcadas como libres de efectos secundarios (side-effect-free). Si tu aplicación importa librerías compartidas que están hechas para ser libres de efectos secundarios, agrega `"sideEffects": false` a tu `package.json`. Esto asegurará que las librerías serán propiamente llamadas si están importadas pero no directamente referenciadas. Puedes ver más detalles del problema original [acá](https://github.com/angular/angular-cli/issues/16799#issuecomment-580912090).

3. Los proyectos que no usen el CLI de Angular, verán una reducción de tamaño significativa, a menos que actualicen las configuraciones de su minificador y establezcan las constantes de tiempo de compilación `ngDevMode`, `ngI18nClosureMode` y `ngJitMode` en `false` (para Terser, por favor establece estas variables en `false` a través de las [opciones de configuración `global_defs`](https://terser.org/docs/api-reference.html#conditional-compilation)). Por favor ten en cuenta que estas constantes no están hechas para ser usadas por librerías de terceros o por código de aplicaciones que no sean parte de nuestra API, y que podría cambiar en un futuro.

{@a common-changes}
### Cambios que podrás notar

* Las consultas `@ContentChildren` por defecto sólo podrán buscar nodos que sean hijos directos en la jerarquía del DOM (anteriormente, estas consultas podían buscar en cualquier nivel de anidación del DOM mientras que no hubiese otra directiva por encima de él que coincidiese). Ver más [detalles](guide/ivy-compatibility-examples#content-children-descendants).

* Todas las clases que usen Angular ID deben tener un decorador Angular como `@Directive()` o `@Injectable` (anteriormente, las clases no decoradas solo se permitían en el modo AOT, o si se usaban indicadores de inyección). Ver más [detalles](guide/ivy-compatibility-examples#undecorated-classes).

* Las entradas no vinculadas a las directivas (por ejemplo, nombres en `<my-comp name="">`), ahora se configuran al crear la vista, antes de que se ejecute la detección de cambios (anteriormente, todas las entradas se configuraban durante la detección de cambios).

* Los atributos estáticos establecidos en el HTML de una plantilla, sobreescribirán cualquier atributo host que tenga conflictos y haya sido establecido por directivas o componentes (anteriormente, los atributos host estáticos establecidos por directivas o componentes, anularían los atributos estáticos de plantilla, de haber un conflicto).

{@a less-common-changes}
### Cambios menos comunes

* Propiedades como `host` dentro de los decoradores `@Component` y `@Directive`, pueden ser heredados (anteriormente, solo serían heredadas las propiedades con decoradores de campos específicos como `@HostBinding`).

* El soporte de HammerJS es aceptado mediante la importación del módulo `HammerModule` (anteriormente, éste siempre era incluído en bundles de producción, independientemente si la aplicación usaba HammerJS).

* Las consultas `@ContentChild` y `@ContentChildren` ya no podrán coincidir con el nodo host de su propia directiva (anteriormente, estas consultas coincidirían con el nodo host además de sus elementos secundarios).

* Si un token es inyectado en la bandera `@Host` o `@Self`, el modulo inyector no será buscado para ese token (anteriormente, los tokens marcados con estas banderas igualmente buscarían en el nivel del módulo).

* Al acceder a varias referencias locales con el mismo nombre en los enlaces de platilla, coincidirá la primera referencia (anteriormente, coincidía la última instancia).

* Las directivas que son usadas en un módulo exportado (pero que no se exportan a sí mismas), son importadas públicamente (anteriormente, el compilador escribía automáticamente un exportación privada, con un alias que podría usar su base de conocimiento global para resolver hacia abajo).

* Las funciones externas o las constantes externas en los metadatos de los decoradores, no se pueden resolver estáticamente (anteriormente, podías importar una constante o función de otra unidad de compilación, como una librería, y usar esa constante/función en tu definición de `@NgModule`).

* Las referencias directas a entradas de directivas que son accesadas mediante referencias locales, ya no son soportadas de forma predeterminada. [Más detalles](guide/ivy-compatibility-examples#forward-refs-directive-inputs).

* Si hay un atributo de clase independiente y un enlace `[class]`, las clases en el atributo independiente también serán agregadas (anteriormente, el enlace de las clase sobreescribía las clases del atributo independiente).

* Ahora es un error asignar valores a variables de tipo plantilla como `item` en `ngFor="let item of items"` (anteriormente, el compilador ignoraba estas asignaciones).

* Ya no es posible sobreescribir los hooks del ciclo de vida con simulaciones (mocks) en las instancias de directivas para realizar pruebas (en lugar de eso, modifica el hook del ciclo de vida del tipo de directiva en sí).

* Los tokens de inyección especial (como `TemplateRef` o `ViewContainerRef`) retornan una nueva instancia cuando sea que sean requeridas (anteriormente, las instancias de tokens especiales eran compartidas solo si eran requeridas en el mismo nodo). Esto afecta principalmente las pruebas que comparan la identidad de estos objetos.

* El parsing ICU ocurre en tiempo de ejecución, por lo que solo se permite texto, etiquetas HTML y enlaces de texto dentro de los casos de ICU (anteriormente, las directivas también eran permitidas dentro de las ICU).

* Agregar enlaces de texto en las traducciones i18n que no están presente en la plantilla de origen, generará un error en tiempo de ejecución (anteriormente, era permitido incluír enlaces adicionales en las traducciones).

* Las etiquetas HTML adicionales en las traducciones i18n que no estén presente en la plantilla de origen, serán renderizadas como texto plano (anteriormente, estas etiquetas se renderizaban como HTML).

* Los proveedores formateados como `{provide: x}` sin las propiedades `useValue`, `useFactory`, `useExisting`, o `useClass`, son tratados como `{provide: X, useClass: X}` (anteriormente, estaba como predeterminado `{provide: X, useValue: undefined}`).

* `DebugElement.attributes`  retorna `undefined` para los atributos que fueron añadidos y luego se eliminaron (anteriormente, los atributos agregados y luego eliminados tenían un valor de `null`).

* `DebugElement.classes` retorna `undefined` para las clases que fueron agregadas y luego eliminadas (anteriormente, las clases agregadas y luego eliminadas tenían un valor de `false`).

* Si seleccionas el elemento nativo `<option>` en un `<select>` donde las etiquetas `<option>` fueron creadas a través de un `*ngFor`, usa la propiedad `[selected]` de una etiqueta `<option>` en lugar de enlazarlo a la propiedad `[value]` del elemento `<select>` (anteriormente, podías enlazarlo a cualquiera de los dos elementos). [Más detalles](guide/ivy-compatibility-examples#select-value-binding).

* Las vistas embebidas (como aquellas creadas con `*ngFor`) ahora se insertan delante del nodo de comentario ancla del DOM (como por ejemplo `<!--ng-for-of-->`), en lugar de hacerlo detrás como era el caso anteriormente. En la mayoría de los casos esto no tendrá ningún impacto en el DOM renderizado. En algunos casos (como animaciones que retrasan la eliminación de una vista embebida) cualquier vista embebida nueva, se insertará después que se termine la animación que elimina la vista embebida anterior. Esta diferencia solo dura mientras la animación está activa, y puede alterar la apariencia visual de la animación. Una vez finalizada la animación, el DOM renderizado resultante es idéntico al que hubiese sido renderizado con el View Engine.

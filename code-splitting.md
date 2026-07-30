# Code Splitting

## Contexto
Por defecto, cuando hacemos una aplicación web moderna con JavaScript, bundlers (como Vite.js,
Webpack o Rollup) empaquetan todo el código fuente, dependencias y librerías en un único
archivo JavaScript masivo (a menudo llamado `main.js` o `bundle.js`).

Cuando un usuario visita el sitio web, su navegador tiene que descargar, parsear y ejecutar
todo el archivo antes de que tan siquiera puedan interactuar con la página, incluso si, dentro
de la aplicación, solo están viendo la landing page y nunca visitan su dashboard o la pestaña
de configuraciones.

Aquí es donde entra el **code splitting**, el cual, es la práctica de separar dicho archivo
monolito en pequeñas fracciones (o porciones) que pueden ser cargados en caliente (lazy-loaded)
o incluso en paralelo.

Imaginemos el caso de un e-commerce. En lugar de forzar al usuario a descargar la vista del
"Checkout" (que puede ser pesada por toda la lógica e interacción detrás) en la página "Home",
**code splitting** asegura que solo se descargue la vista "Home". La vista "Checkout"
se carga únicamente cuando el usuario le dé click a "Proceder al Checkout", por ejemplo.

## Uso
La forma más común de lograr **code splitting** en aplicaciones web modernas de JavaScript es
mediante la sintaxis de import dinámico (`import()`). A diferencia de los import estáticos,
import dinámicos devuelven una promesa y son evaluados en tiempo de ejecución (runtime).

### Vanilla
Puedes cargar de forma dinámica una librería de utilidad pesada o un módulo específico
cuando ocurra una acción, como hacer click en un botón.

```JS
// En lugar de importar de forma estática al principio
// import { renderChart } from './charts.js'

const button = document.querySelector('#view-charts-btn')

button.addEventListener('click', async () => {
  try {
    // El navegador solo cargar el modulo cuando el boton es clickeado
    const { renderChart } = await import('./charts.js')
    renderChart()
  } catch (error) {
    console.error('Ha ocurrido un error al cargar el modulo de graficas', error)
  }
})
```

### React 19
En React, se acompaña el uso de import dinámicos con dos APIs integradas de la librería:
la función `lazy` y el componente `<Suspense>`. Con `lazy` le decimos a React que vamos
a diferir la carga del componente (que, en este momento, es la promesa retornada por
`import()`), y con `<Suspense>` envolvemos el componente dentro para que pueda ser
resuelto por React al momento que es llamado por el usuario.

Dos cosas a tener en cuenta: Primero, resolver el componente toma tiempo, al menos solo
la primera vez que no se encuentra cargado en la aplicación. `<Suspense>` provee un
prop `fallback` para renderizar un componente (usualmente indicadores de carga)
que le muestren al usuario que la aplicación está cargando la vista o UI requerida.

Segundo, qué creen que pasa si ocurre un error al intentar resolver el módulo? Simple,
la aplicación crashea y te quedas con una pantalla en blanco (o en el caso de Webpack,
un log enorme de errores más pesado que la propia aplicación). `<Suspense>` no provee
un prop integrado para manejar dichos errores, por lo que deberás crear un componente
"Error Boundary" y envolver el componente `<Suspense>` con este, para mostrar una UI
de error si falla al cargar.

```JSX
import { Activity, Suspense, lazy, useState } from 'react'
import { ErrorBoundary } from './ErrorBoundary'
import { Spinner } from '@components/ui/Spinner'

const CartModal = lazy(() => import('./CartModal'))

export const Home = () => {
  const [isOpen, setIsOpen] = useState(false)
  const toggle = () => setIsOpen(prev => !prev)

  return (
    <div>
      <header className="w-full flex items-center justify-between">
        <img src="/imgs/logo.svg" alt="Company Logo">
        <section className="relative">
          <button type="button" onClick={toggle}>
            {isOpen ? 'Cerrar' : 'Abrir'} carrito
          </button>
          {/* Activity es casi el equivalente a {isOpen && (...)} pero sin perder el estado */}
          <Activity mode={isOpen}>
            <ErrorBoundary>
              <Suspense fallback={<Spinner />}>
                <CartModel className="absolute top-0 right-0" />
              </Suspense>
            </ErrorBoundary>
          </Activity>
        </section>
      <header>
    <div>
  )
}
```

### Svelte 5
En desarrollo manejado por componentes, a menudo estarás haciendo code splitting a nivel de ruta o
para componentes pesados o condicionados. Svelte maneja componentes asincronos de forma natural
aprovechando promesas directamente en el markup.

```SVELTE
<script>
  import ErrorFallback from './ErrorFallback.svelte'
  import Spinner from '$lib/components/ui/Spinner.svelte'

  const loadCartModel = () => import('./CartModel.svelte').then(module => module.default)

  let isOpen = $state(false)
  const toggle = () => { isOpen = !isOpen }
</script>

<div>
  <header className="w-full flex items-center justify-between">
    <img src="/imgs/logo.svg" alt="Company Logo">
    <section className="relative">
      <button type="button" onclick={toggle}>
        {isOpen ? 'Cerrar' : 'Abrir'} carrito
      </button>
      {#if isOpen}
        {#await loadCartModel()}
          <Spinner />
        {:then CartModel}
          <CartModel class="absolute top-0 right-0" />
        {:catch error}
          <ErrorFallback message={error.message} />
      {/if}
    </section>
  </header>
</div>
```

## A que stack beneficia mas?
**Code Splitting** provee el mejor valor de inversion en Single Page Applications (SPAs) y
Frontends altamente interactivos.

- Si tu frontend maneja estado complejo, UIs interactivas, visualización de datos o formularios complejos enteramente desde el navegador, **code splitting** previene la carga inicial de inflarse.
- Stacks que dependen mucho en paquetes de terceros de NPM (Moment.js, editores de texto o librerias de graficas como D3 o Three.js), eso beneficia inmensamente al aislar dichas librerias a las vistas especificas que las uses.
- Usuarios en dispositivos moviles o con conexiones inestables son los que mas se benefician, ya que, al reducir el tamano de la carga inicial mejora drasticamente metricas como **Time to Interactive** (TTI) o **Largest Contentful Paint** (LCP).

## Recomendaciones

### Se puede...
- **Splits por ruta**: Asegura que navegar a una nueva URL haga que descargue nuevas porciones de codigo.
- **Aisla dependencias pesadas**: Si un componente necesita una dependencia pesada, importalo dinamicamente para que no sea empaquetado en el punto de entrada inicial de tu aplicacion.
- **Provee UI util**: Dado que import dinamicos son asincronos y dependen del internet, siempre muestra un indicador de carga, un skeleton placeholder o mensaje de estado para que el usuario sepa que la aplicacion esta cargando la vista requerida.
- **Monitorea el tamaño de tus porciones de codigo**: Usa herramientas visuales como `rollup-plugin-visualizer` o `webpack-bundle-analyzer` para ver exactamente que hay dentro de tus porciones y asegurate de no filtrar accidentalmente utilidades compartidas en tu empaquetado principal.

### No se puede...
- **Over-split**: Hacer splitting de cada componente crea un antipatron llamado **chunk fragmentation**. El navegador desperdiciara mas tiempo creando decenas de peticiones HTTP por pequeños archivos de 2KB que lo que hubiera desperdiciado descargando un solo archivo de 50KB.
- **No hacer code-splitting de los layouts principales**: Nunca cargar el layout principal de forma asincrona, navegacion de cabecera o el contenido principal visible inmediatamente en la carga de la pagina. Esto causa cambios de layout notables y daña metricas de experiencia de usuario (UX).
- **No olvidar manejo de errores**: Las peticiones de componentes pueden fallar (al usuario se le va el internet mientras clickea un boton). Implementa siempre vistas para errores en tus import dinamicos para permitir a tus usuarios reintentar cargarlos.

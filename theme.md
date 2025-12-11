# Cambio de tema
Para implementar el cambio de tema, usaremos la API de React y TailwindCSS para manejar los estilos.

> [!NOTE]
> Primero debes tener instalado TailwindCSS (versión 4 en adelante) para realizar esta implementación.

## Integra TailwindCSS en tu archivo principal de CSS
Para que se pueda realizar el cambio de tema a nivel Tailwind, debemos tener nuestro archivo principal de CSS
configurado de la siguiente forma:

```CSS
@import "tailwindcss";

/* Aquí tu root para las variables de cada tema */
:root {}

/*
  Aquí extiendes la paleta de colores de tailwind con tus propia paleta (recomendado
  extender con la paleta de colores por defecto de la app).
*/
@theme {}

/*
  Esto es esencial para el cambio a modo oscuro. Si manejas multitemas, puedes
  cambiar dark por otro nombre o agregar más `@custom-variant`
*/
@custom-variant dark (&:is(.dark *));

/* Este selector debe coincidir con el nombre del `@custom-variant` que agregaste arriba */
.dark {}
```

## Crea el contexto
Crea un archivo de nombre `ThemeContext.js` (o `.ts` si usas TypeScript).

```JS
import { createContext } from 'react'

/**
 * @typedef {'light' | 'dark'} Theme
 */

/**
 * @typedef {object} ThemeContextType
 * @property {Theme} theme
 * @property {() => void} changeTheme
 */

/** @type {ThemeContextType | null} */
export const ThemeContext = createContext(null)
```

## Crea el provider
El provider va a contener la lógica del cambio de tema. Antes de realizar el snippet de código, ten en cuenta estas condiciones:
1. El cambio debe hacerse en el estado (React)
2. El cambio debe verse en la UI (DOM)
3. El cambio debe persistir al recargar la página (`localStorage`)

Crea un archivo `ThemeProvider.jsx` (o `.tsx` si usas TypeScript).
```JSX
import { useEffect, useState } from 'react'
import { ThemeContext } from '@contexts/ThemeContext'

/**
 * Retorna un provider con funcionalidad para cambio de temas
 * @param {object} props - las propiedades del provider.
 * @param {import('react').ReactNode} props.children - Un nodo de react.
 * @returns {import('react').JSX.Element} El provider con las funcionalidades
export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState(() => localStorage.getItem('theme') ?? 'light')

  /**
   * Cambia de tema a claro u oscuro.
   * @returns {void} No retorna nada.
   */
  const changeTheme = () => setTheme((prev) => (prev === 'light' ? 'dark' : 'light')) // Cambio en React.

  useEffect(() => {
    document.documentElement.classList.remove('light', 'dark')
    document.documentElement.classList.add(theme) // Cambio en UI.
    localStorage.setItem('theme', theme) // Persistencia.
  }, [theme])

  return <ThemeContext value={{ theme, change }}>{children}</ThemeContext>
}
```

### Envuelve la parte de tu aplicación que necesite el provider
Por ejemplo, `main.jsx` si quieres globalidad.

```JSX
import './index.css'
import { App } from './App.jsx'
import { StrictMode } from 'react'
import { ThemeProvider } from '@providers/ThemeProvider.jsx'
import { createRoot } from 'react-dom/client'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <ThemeProvider>
      <App />
    </ThemeProvider>
  </StrictMode>,
)
```

## Crea un Custom Hook para validar existencia del contexto
Esto es importante porque te evita fallar si el contexto o provider no existen o no están envueltos dentro del provider
y puedes separar la responsabilidad del contexto al Custom Hook (testeabilidad mucho menos compleja). Llamemos al archivo
`useThemeContext.js` (o `.ts si usas TypeScript).

```JS
import { use } from 'react'
import { ThemeContext } from '@contexts/ThemeContext'

/**
 * Custom hook que accede al contexto de temas.
 * @returns {ReturnType<typeof ThemeContext>} Si el contexto está envuelto dentro de un provider.
 * @throws {Error} Si no hay provider del contexto de temas.
 */
export const useThemeContext = () => {
  const context = use(ThemeContext)
  if (!context) throw new Error('El contexto de temas debe ser usado dentro de su respectivo provider')

  return context
}
```


## Agrega la función para cambiar el tema
Finalmente, crea un componente para cambiar el tema de la app.

```JS
import { Button } from '@components/ui/Button'
import { useThemeContext } from '@hooks/useThemeContext'

/**
 * Renderiza un header con un boton para cambiar el tema de la aplicacion.
 * @returns {import('react').JSX.Element} El componente Header.
 */
export const Header = () => {
  const { changeTheme, theme } = useThemeContext()

  return (
    <header>
      <Button onClick={changeTheme}>
        {theme === 'dark' ? 'Tema claro' : 'Tema oscuro'}
      </Button>
    </header>
  )
}
```

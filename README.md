This project was generated with [Angular CLI](https://github.com/angular/angular-cli) version 12.2.18.

# Creación y configuración de una aplicación remota (Microfrontend)

Esta documentación detalla los pasos para crear y configurar una aplicación **remota** (o **Microfrontend**), llamada para este ejemplo `movieForm3`, utilizando **Angular 12** y los paquetes de **Module Federation**.

El **objetivo** de este proyecto es habilitar la carga de esta aplicación remota por una aplicación **Host/Shell de Angular 19**, demostrando la capacidad de ejecutar componentes con **diferentes versiones de Angular** en la misma página (arquitectura de Microfrontends Políglotas).

🛠️ Prerrequisitos
*   **Node.js:** Se requiere la versión 14.x.
    
*   **Angular CLI:** Se usará la versión 12.x.
    
*   **nvm** (Node Version Manager): Opcional, pero recomendado para gestionar versiones de Node.js.

### 1. Preparación del Entorno

Si no tienes las versiones requeridas de Node.js y Angular CLI instaladas:
*   Instalar la versión 14.21 de Node.js y activarla (si usas `nvm`):
    Bash
    
        nvm install 14.21
        nvm use 14.21
    
*   Instalar la versión 12 de Angular CLI globalmente:
    Bash
    
        npm install -g @angular/cli@12
    

### 2. Creación del Proyecto Base

Crea una nueva aplicación de Angular. Este será tu proyecto remoto:
Bash

    ng new movieForm3
    cd movieForm3

### 3. Configuración de Module Federation
Instala y configura el _plugin_ de Module Federation para tu proyecto de Angular 12.
*   Añade el _plugin_ al proyecto, especificando el puerto de desarrollo (por ejemplo, `4206`):
    Bash
    
        ng add @angular-architects/module-federation@12.5.3 --project movieForm3 --port 4206
    
*   Instala las dependencias adicionales necesarias para la construcción y _runtime_:
    Bash
    
        # Dependencia de construcción (requerida por Module Federation en esta versión)
        npm i ngx-build-plus@^12.0.0 --save-dev
        
        # Dependencia de runtime
        npm i @angular-architects/module-federation-runtime@^12.5.3 --save
        
        # Dependencia de herramientas de desarrollo (opcional, pero recomendada)
        npm i @angular-architects/module-federation-tools@^12 -D
        
        # Instala Angular Elements para la exposición de componentes como Web Components
        npm i @angular/elements@^12
    

### 4. Modificación de `webpack.config.js`

El paso `ng add` crea un archivo `webpack.config.js`. Edita este archivo para configurar qué componente o módulo expondrá la aplicación remota y cómo se compartirán las dependencias.
*   **Verificar `exposes`:** Asegúrate de que la sección `exposes` esté configurada correctamente para exponer el `Bootstrap` de tu Microfrontend. Esto permitirá que la aplicación _host_ cargue el módulo remoto.
    *   _Ejemplo de estructura (a revisar según tu implementación específica):_
        JavaScript
        
            // ... dentro de ModuleFederationPlugin
            exposes: {
              './Component': './src/bootstrap.ts', // O la ruta de tu componente/módulo a exponer
            },
            // ...
        
*   **Configuración de `shared` dependencies:** Debido a que el objetivo es mezclar versiones (Angular 12 y 19), se **debe** evitar el uso de `singleton: true` y `strictVersion: true` para que ambas aplicaciones puedan cargar sus propias versiones de las librerías compartidas si es necesario. Si se omiten, los valores predeterminados son `false`.
    JavaScript
    
        // ... dentro de ModuleFederationPlugin
        shared: {
            "@angular/core": { requiredVersion: 'auto' }, // singleton: false y strictVersion: false por defecto
            "@angular/common": { requiredVersion: 'auto' },
            "@angular/router": { requiredVersion: 'auto' },
            // ... otras librerías compartidas
        },
        // ...
    
### 5. Configuración del Módulo Raíz (`app.module.ts`)

Para que el componente pueda ser cargado como un elemento autónomo, es necesario modificar el _lifecycle_ de _bootstrap_ de Angular.
*   **Implementar `ngDoBootstrap`:** En `src/app/app.module.ts`, agrega la interfaz `DoBootstrap` a tu `AppModule` e implementa el método `ngDoBootstrap`. Este método se usa para convertir tu componente remoto en un **Custom Element** (Web Component) que el _host_ puede referenciar.
 **Nota importante:** el nombre con el que se define el custom element debera llevar al menos un guion **-** y todo no debe incluir mayusculas.
    
*   **Vaciar `bootstrap`:** Modifica el _metadata_ de `@NgModule` para dejar el _array_ de `bootstrap` **vacío**:

**Ejemplo de `src/app/app.module.ts`:**

    import { NgModule, Injector, DoBootstrap } from '@angular/core';
    import { BrowserModule } from '@angular/platform-browser';
    import { createCustomElement } from '@angular/elements'; // Importar
    import { AppComponent } from './app.component';
    
    @NgModule({
      declarations: [
        AppComponent
      ],
      imports: [
        BrowserModule
      ],
      // Importante: Deja el array vacío para controlarlo manualmente
      bootstrap: [],
    })
    export class AppModule implements DoBootstrap {
      constructor(private injector: Injector) {}
    
      ngDoBootstrap() {
        // 1. Convierte el componente a un Custom Element
        const ce = createCustomElement(AppComponent, { injector: this.injector });
        
        // 2. Define el tag que usará la aplicación Host
        customElements.define('movie-form3-element', ce);
      }
    }

### 6. Configuración de `index.html` (Prueba Local)

Modifica el archivo `src/index.html` para incluir el tag de tu _Custom Element_. Esto permite probar que la aplicación remota, al ser cargada, registra y renderiza su componente correctamente.
**Contenido de `src/index.html`:**

HTML

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>MovieForm3</title>
      <base href="/">
      <meta name="viewport" content="width=device-width, initial-scale=1">
      <link rel="icon" type="image/x-icon" href="favicon.ico">
    </head>
    <body>
      <movie-form3-element></movie-form3-element>
    </body>
    </html>
        

🔑 Valores Clave Expuestos para la Integración
----------------------------------------------

Tras la configuración de la aplicación remota (`movieForm3`), se han definido cuatro valores esenciales que la aplicación **Host (Angular 19)** utilizará para descubrir, cargar y renderizar el componente de Angular 12.
Estos valores son la base para el patrón **Module Federation** y la carga dinámica de Microfrontends.

* * *

### 1. Definiciones y Ubicación

| **Valor** | **Descripción** | **Ubicación en Remota (movieForm3)** | **Host (Shell) lo Usará como:** |
| --- | --- | --- | --- |
| **`remoteName`** | El **nombre único** asignado al _Microfrontend_ dentro de la federación. | `webpack.config.js` (`name`) | La clave para referenciar el remoto. |
| **`filename`** | El nombre del archivo que contiene el _manifest_ (mapa) del remoto y su código de carga. | `webpack.config.js` (`filename`) | Se usa para construir la URL de `remoteEntry`. |
| **`exposedModule`** | La referencia interna al módulo o componente que el remoto _expone_. | `webpack.config.js` (`exposes`) | La sub-ruta que el Host pedirá cargar. |
| **`elementName`** | El nombre del _Custom Element_ (Web Component) creado a partir del componente. | `app.module.ts` (`ngDoBootstrap`) | La etiqueta HTML real para inyectar el componente. |

* * *

### 2. Extracción de `webpack.config.js`

El archivo `webpack.config.js` define cómo el módulo remoto se identifica y qué expone:
JavaScript

    // ... dentro de ModuleFederationPlugin
    name: "movieForm3", // <-- remoteName
    filename: "remoteEntry.js", // <-- filename
    exposes: {
       './Component': './src/bootstrap.ts', // <-- exposedModule y su ruta
    },
    // ...
*   **`remoteName`:** **`movieForm3`**
    
*   **`filename`:** **`remoteEntry.js`**
    
Estos dos valores se combinan para formar la URL de acceso:
*   **`remoteEntry` (URL de Carga):** `http://localhost:4206/remoteEntry.js`
    *   **Nota:** La URL completa (`http://localhost:4206/`) depende del entorno de ejecución (`ng serve`) o de la URL de publicación final del servidor (ej. NGINX).
        

* * *

### 3. Extracción de `app.module.ts` (Angular Elements)

El _Custom Element_ se define en el ciclo de vida `ngDoBootstrap` del módulo raíz. Este es el nombre del _tag_ HTML que el Host insertará en el DOM:
TypeScript

    ngDoBootstrap() {
      const ce = createCustomElement(AppComponent, { injector: this.injector });
      // Custom element names must be lowercase and include a hyphen.
      customElements.define('movie-form3-element', ce); // <-- elementName
    }
*   **`elementName`:** **`movie-form3-element`**

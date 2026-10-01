# Angular Dojo: Prerendering (SSG – Static Site Generation)
**Datum:** 2026-10-01
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular mit dem `prerender`-Builder statische HTML-Seiten zur Build-Zeit erzeugt (SSG), und verstehst den Unterschied zu klassischem SSR sowie die typischen Fallstricke bei der Kombination mit dynamischen Routen.

## Hintergrund & Theorie

Seit Angular 17 ist das serverseitige Rendering direkt im Angular-Ökosystem integriert. Neben dem klassischen SSR (Laufzeit-Rendering auf dem Server) unterstützt Angular auch **Prerendering**: Dabei werden HTML-Seiten bereits während des Build-Prozesses statisch generiert – ähnlich wie bei Next.js' Static Site Generation oder Astro.

### Vorteile gegenüber SSR
- Keine laufende Server-Infrastruktur nötig – statische Dateien können auf einem CDN ausgeliefert werden.
- Sehr schnelle Time-to-First-Byte (TTFB), da kein Request auf dem Server verarbeitet wird.
- Ideal für Inhalte, die sich selten ändern (Landing Pages, Dokumentationen, Blog-Einträge).

### Funktionsweise
Angular verwendet den `@angular/ssr`-Builder mit der Option `prerender: true`. Zur Build-Zeit wird die App mit Node.js gerendert; für jede konfigurierte Route entsteht eine `index.html`-Datei im Output-Verzeichnis.

Routen können explizit angegeben (`routes`-Array), aus der App-Konfiguration ermittelt (automatisch via Route-Discovery) oder aus einer externen Datei geladen werden. Für parametrisierte Routen (z. B. `/products/:id`) muss die Liste möglicher Parameter explizit bereitgestellt werden – entweder statisch oder über eine `PrerenderFallback`-Strategie.

```
dist/
  browser/
    index.html          ← Startseite
    products/
      index.html        ← /products
      42/
        index.html      ← /products/42 (vorgerendert)
```

## Aufgabe

Erweitere eine bestehende Angular-App (oder erstelle eine kleine Beispiel-App) um eine vorgerenderte Produktdetailseite.

### Schritte

1. **Projekt anlegen / SSR aktivieren**
   Erstelle ein neues Angular-Projekt mit SSR-Unterstützung:
   ```bash
   ng new dojo-ssg --ssr
   ```
   oder aktiviere SSR in einem bestehenden Projekt:
   ```bash
   ng add @angular/ssr
   ```

2. **Routen definieren**
   Erstelle zwei Routen: `/products` (Übersicht) und `/products/:id` (Detailseite).
   Implementiere zwei einfache Standalone-Komponenten für diese Routen.
   Die Detailseite soll die `id` aus den Route-Parametern lesen und anzeigen.

3. **`server.routes.ts` konfigurieren**
   Öffne (oder erstelle) `src/app/server.routes.ts` und definiere die Prerender-Konfiguration:
   ```typescript
   import { RenderMode, ServerRoute } from '@angular/ssr';

   export const serverRoutes: ServerRoute[] = [
     { path: '',          renderMode: RenderMode.Prerender },
     { path: 'products',  renderMode: RenderMode.Prerender },
     {
       path: 'products/:id',
       renderMode: RenderMode.Prerender,
       async getPrerenderParams() {
         // Simuliere einen API-Aufruf, der bekannte IDs zurückgibt
         return [{ id: '1' }, { id: '2' }, { id: '3' }];
       },
     },
   ];
   ```

4. **`app.config.server.ts` anpassen**
   Stelle sicher, dass `provideServerRoutesConfig(serverRoutes)` in der Server-App-Konfiguration eingetragen ist:
   ```typescript
   import { provideServerRoutesConfig } from '@angular/ssr';
   import { serverRoutes } from './server.routes';

   export const serverConfig: ApplicationConfig = {
     providers: [provideServerRoutesConfig(serverRoutes)],
   };
   ```

5. **Build ausführen und Ergebnis prüfen**
   ```bash
   ng build
   ```
   Inspiziere das Verzeichnis `dist/<projektname>/browser/` – dort sollten
   `products/1/index.html`, `products/2/index.html` und `products/3/index.html` liegen.

6. **Lokalen Static-File-Server starten**
   ```bash
   npx serve dist/<projektname>/browser
   ```
   Öffne `http://localhost:3000/products/1` und prüfe im View-Source, ob der HTML-Inhalt
   bereits serverseitig gerendert ist (kein leeres `<app-root>`).

## Hints

<details>
<summary>Hint 1 – Detailseite mit `input()` statt `ActivatedRoute`</summary>

Mit Angular 17+ und aktivierten Router Input Bindings kannst du Route-Parameter direkt als Signal-Input empfangen:

```typescript
// In app.config.ts
provideRouter(routes, withComponentInputBinding())

// In der Detailkomponente
@Component({ ... })
export class ProductDetailComponent {
  readonly id = input.required<string>();
}
```

Das ist kürzer und testbarer als das Injizieren von `ActivatedRoute`.
</details>

<details>
<summary>Hint 2 – Fallback-Strategie für unbekannte IDs</summary>

Wenn ein Nutzer `/products/999` aufruft und diese Route nicht vorgerendert wurde,
kannst du einen Fallback definieren:

```typescript
{
  path: 'products/:id',
  renderMode: RenderMode.Prerender,
  fallback: PrerenderFallback.Server, // SSR-Rendering als Fallback
  async getPrerenderParams() { ... },
}
```

Mögliche `PrerenderFallback`-Werte: `Server`, `Client`, `None`.
</details>

## Beispiellösung

```typescript
// src/app/server.routes.ts
import { RenderMode, ServerRoute, PrerenderFallback } from '@angular/ssr';

const KNOWN_PRODUCT_IDS = ['1', '2', '3', '4', '5'];

export const serverRoutes: ServerRoute[] = [
  {
    path: '',
    renderMode: RenderMode.Prerender,
  },
  {
    path: 'products',
    renderMode: RenderMode.Prerender,
  },
  {
    path: 'products/:id',
    renderMode: RenderMode.Prerender,
    fallback: PrerenderFallback.Client,
    async getPrerenderParams() {
      return KNOWN_PRODUCT_IDS.map(id => ({ id }));
    },
  },
];
```

```typescript
// src/app/product-detail/product-detail.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <h1>Produkt {{ id() }}</h1>
    <p>Diese Seite wurde zur Build-Zeit statisch generiert.</p>
  `,
})
export class ProductDetailComponent {
  readonly id = input.required<string>();
}
```

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'products/:id',
    loadComponent: () =>
      import('./product-detail/product-detail.component').then(
        m => m.ProductDetailComponent
      ),
  },
  {
    path: 'products',
    loadComponent: () =>
      import('./product-list/product-list.component').then(
        m => m.ProductListComponent
      ),
  },
];
```

## Weiterführendes
- **Angular Docs – Server-side rendering**: https://angular.dev/guide/ssr – offizielle Doku zu SSR und Prerendering inkl. `RenderMode`-Überblick.
- **Tipp**: Kombiniere Prerendering mit `TransferState` (Dojo vom 2026-09-11), um API-Daten aus dem vorgerenderten HTML direkt in die Client-App zu übertragen und einen doppelten HTTP-Request zu vermeiden.
- **Tipp**: Mit `RenderMode.Server` für häufig wechselnde Daten und `RenderMode.Prerender` für stabile Inhalte kannst du beide Strategien innerhalb derselben App mischen (sog. Hybrid Rendering).

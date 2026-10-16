# Angular Dojo: Hybrid Rendering Modes (RenderMode API)
**Datum:** 2026-10-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit Angulars `RenderMode`-API (ab Angular 17) einzelnen Routen gezielt Server-Side Rendering, Static Prerendering oder reines Client-Side Rendering zuweist – und warum die richtige Wahl entscheidend für Performance und SEO ist.

## Hintergrund & Theorie

Angular 17 führte das Konzept des **Hybrid Renderings** ein: Statt eine App pauschal als SSR oder CSR zu definieren, kann jede Route eine eigene Rendering-Strategie erhalten.

Die drei `RenderMode`-Werte sind:

| Modus | Beschreibung |
|---|---|
| `RenderMode.Server` | Seite wird pro Request serverseitig gerendert (klassisches SSR) |
| `RenderMode.Prerender` | Seite wird einmalig zur Build-Zeit gerendert und als statisches HTML ausgeliefert (SSG) |
| `RenderMode.Client` | Kein Prerendering – normales SPA-Rendering im Browser |

Die Konfiguration erfolgt über `app.routes.server.ts`, eine separate Routing-Datei, die `serverRoutes` definiert und mit `provideServerRendering({ routes: serverRoutes })` registriert wird.

**Wann welcher Modus?**
- **`Prerender`**: Marketing-Seiten, Blog-Posts, statische Inhalte → maximale Performance, kein Server nötig
- **`Server`**: Dashboards mit nutzerspezifischen Daten, Seiten mit häufig änderndem Inhalt
- **`Client`**: Admin-Bereiche hinter Auth, stark interaktive Tools – SSR bringt hier keinen Vorteil

**Wichtig**: Für `Prerender` mit dynamischen Parametern (z. B. `/product/:id`) müssen die Parameter via `getPrerenderParams()` angegeben werden.

## Aufgabe

Du baust eine kleine Angular-Applikation mit drei Routen und konfigurierst für jede Route einen anderen Rendering-Modus.

**Ausgangslage**: Eine App mit drei Routen:
- `/` – Startseite (statisch, nie verändert)
- `/dashboard` – Nutzerspezifisches Dashboard
- `/product/:id` – Produktdetailseite mit bekannten IDs

### Schritte

1. Erstelle `app.routes.server.ts` mit `serverRoutes` und weise den Routen passende `RenderMode`-Werte zu. Die Startseite bekommt `Prerender`, das Dashboard `Server`, und die Produktseiten ebenfalls `Prerender` mit `getPrerenderParams()`.

2. Konfiguriere `getPrerenderParams()` für die Produktroute, sodass die IDs `['1', '2', '3']` vorgerendert werden. Die Funktion muss ein `Promise<Array<Record<string, string>>>` zurückgeben.

3. Registriere die `serverRoutes` in `app.config.server.ts` über `provideServerRendering({ routes: serverRoutes })`.

4. Überprüfe: Wo müsste `TransferState` eingesetzt werden, damit `RenderMode.Server` keine doppelten HTTP-Requests auslöst?

5. Füge eine Konsolen-Log-Ausgabe hinzu, die im Serverkontext (via `isPlatformServer`) eine Nachricht ausgibt, um zu verifizieren, in welchem Kontext gerendert wird.

## Hints

<details>
<summary>Hint 1 – Grundstruktur von app.routes.server.ts</summary>

```typescript
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'dashboard', renderMode: RenderMode.Server },
  { path: 'product/:id', renderMode: RenderMode.Prerender, getPrerenderParams: async () => [...] },
];
```

</details>

<details>
<summary>Hint 2 – getPrerenderParams für dynamische Routen</summary>

`getPrerenderParams` ist eine asynchrone Funktion, die ein Array von Parameterobjekten zurückgibt. Jedes Objekt entspricht einem vorgerenderten URL-Segment:

```typescript
getPrerenderParams: async () => [
  { id: '1' },
  { id: '2' },
  { id: '3' },
]
```

Damit werden `/product/1`, `/product/2` und `/product/3` statisch gerendert.

</details>

<details>
<summary>Hint 3 – Registrierung in app.config.server.ts</summary>

```typescript
import { mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig = mergeApplicationConfig(appConfig, {
  providers: [
    provideServerRendering({ routes: serverRoutes }),
  ],
});

export default serverConfig;
```

</details>

## Beispiellösung

```typescript
// app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  {
    path: '',
    renderMode: RenderMode.Prerender,
  },
  {
    path: 'dashboard',
    renderMode: RenderMode.Server,
  },
  {
    path: 'product/:id',
    renderMode: RenderMode.Prerender,
    getPrerenderParams: async () => [
      { id: '1' },
      { id: '2' },
      { id: '3' },
    ],
  },
];
```

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering({ routes: serverRoutes }),
  ],
};

export default mergeApplicationConfig(appConfig, serverConfig);
```

```typescript
// dashboard.component.ts – TransferState für RenderMode.Server
import { Component, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { TransferState, makeStateKey } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { of } from 'rxjs';

const DASHBOARD_KEY = makeStateKey<DashboardData>('dashboard');

@Component({
  selector: 'app-dashboard',
  standalone: true,
  template: `
    @if (data()) {
      <h1>Willkommen, {{ data()!.username }}</h1>
    }
  `,
})
export class DashboardComponent {
  private platformId = inject(PLATFORM_ID);
  private transferState = inject(TransferState);
  private http = inject(HttpClient);

  data = toSignal(
    this.transferState.hasKey(DASHBOARD_KEY)
      ? of(this.transferState.get(DASHBOARD_KEY, null))
      : this.http.get<DashboardData>('/api/dashboard').pipe(
          tap(data => {
            if (isPlatformServer(this.platformId)) {
              this.transferState.set(DASHBOARD_KEY, data);
            }
          })
        )
  );
}

interface DashboardData {
  username: string;
}
```

## Weiterführendes

- **Angular Docs – Hybrid Rendering**: Die offizielle Dokumentation unter `angular.dev/guide/ssr` beschreibt alle `RenderMode`-Optionen und deren Build-Ausgaben.
- **Combine with `@defer`**: Für `RenderMode.Server`-Seiten lohnt sich `@defer (on viewport)` für Komponenten unterhalb des Folds – das reduziert die Server-Render-Zeit und verbessert LCP.
- **`getPrerenderParams` mit API-Daten**: In Production-Apps ruft man in `getPrerenderParams` echte Daten ab (z. B. alle Produkt-IDs aus einer DB), damit der Build alle Seiten statisch vorrendert.

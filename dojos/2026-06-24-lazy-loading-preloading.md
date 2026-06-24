# Angular Dojo: Lazy Loading & Preloading Strategies

**Datum:** 2026-06-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie Angular-Module und Standalone-Routes lazy geladen werden, und wie du mit eigenen Preloading-Strategien die Balance zwischen initialem Bundle-Size und Nutzererfahrung optimierst.

## Hintergrund & Theorie

Lazy Loading ist eine der wirkungsvollsten Techniken zur Performance-Optimierung in Angular. Anstatt die gesamte Applikation beim Start zu laden, werden Feature-Module erst dann heruntergeladen, wenn der Nutzer die entsprechende Route aufruft. Dies reduziert den initialen Bundle und verkürzt die Time-to-Interactive (TTI) erheblich.

**Wie funktioniert es?** Im Router wird statt `component` oder `children` eine dynamische `loadComponent`- bzw. `loadChildren`-Funktion angegeben. Angular erzeugt daraus automatisch separate JavaScript-Chunks über Code Splitting.

**Das Problem mit reinem Lazy Loading:** Beim ersten Aufruf einer lazy Route gibt es eine kurze Verzögerung, da der Chunk erst geladen werden muss. **Preloading-Strategien** lösen dieses Problem: Sie laden Chunks im Hintergrund, nachdem die Applikation gestartet ist.

Angular bietet drei eingebaute Strategien:
- `NoPreloading` – kein Preloading (Standard)
- `PreloadAllModules` – lädt alle lazy Chunks sofort im Hintergrund
- **Eigene Strategie** – lädt nur ausgewählte Routen, z. B. die als `preload: true` markiert sind

Mit Angular 17+ und Standalone Components wird `loadComponent` bevorzugt gegenüber `loadChildren` mit einem NgModule.

## Aufgabe

Erstelle eine kleine Angular-App mit drei Feature-Routen (Dashboard, Settings, Reports) als Standalone Components. Implementiere eine **Custom Preloading Strategy**, die nur Routen mit dem Route-Data-Flag `preload: true` im Hintergrund vorlädt.

### Schritte

1. **Routen anlegen:** Definiere drei lazy Routen mit `loadComponent`. Markiere nur die `dashboard`-Route mit `data: { preload: true }`.

2. **Custom Strategy implementieren:** Erstelle einen Injectable `SelectivePreloadingStrategy`, der `PreloadingStrategy` implementiert. Die `preload()`-Methode soll nur dann laden, wenn `route.data?.['preload'] === true`.

3. **Strategy registrieren:** Übergib die Custom Strategy bei `provideRouter(routes, withPreloading(SelectivePreloadingStrategy))`.

4. **Verifikation:** Öffne die Browser DevTools → Network-Tab. Beim App-Start soll nur der `dashboard`-Chunk geladen werden, nicht `settings` oder `reports`.

5. **Bonus:** Füge einen `PreloadingService` hinzu, der zur Laufzeit steuert, welche Routen preloaded werden sollen (z. B. nach einem Login-Event).

## Hints

<details>
<summary>Hint 1 – Grundstruktur der Routen</summary>

```typescript
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component')
        .then(m => m.DashboardComponent),
    data: { preload: true },
  },
  {
    path: 'settings',
    loadComponent: () =>
      import('./features/settings/settings.component')
        .then(m => m.SettingsComponent),
    // kein preload flag
  },
  {
    path: 'reports',
    loadComponent: () =>
      import('./features/reports/reports.component')
        .then(m => m.ReportsComponent),
    // kein preload flag
  },
];
```

</details>

<details>
<summary>Hint 2 – PreloadingStrategy implementieren</summary>

```typescript
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    return route.data?.['preload'] === true ? load() : of(null);
  }
}
```

Die `load`-Funktion, die Angular übergibt, triggert das dynamische `import()`. Wenn du `of(null)` zurückgibst, wird der Chunk nicht vorgeladen.

</details>

<details>
<summary>Hint 3 – Bonus: Dynamisches Preloading via Service</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class PreloadingService {
  private preloadRoutes = new Set<string>();

  enablePreload(routePath: string): void {
    this.preloadRoutes.add(routePath);
  }

  shouldPreload(route: Route): boolean {
    return this.preloadRoutes.has(route.path ?? '');
  }
}

@Injectable({ providedIn: 'root' })
export class DynamicPreloadingStrategy implements PreloadingStrategy {
  constructor(private service: PreloadingService) {}

  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    return this.service.shouldPreload(route) ? load() : of(null);
  }
}
```

Nach dem Login kannst du dann `preloadingService.enablePreload('reports')` aufrufen.

</details>

## Beispiellösung

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component')
        .then(m => m.DashboardComponent),
    data: { preload: true },
  },
  {
    path: 'settings',
    loadComponent: () =>
      import('./features/settings/settings.component')
        .then(m => m.SettingsComponent),
  },
  {
    path: 'reports',
    loadComponent: () =>
      import('./features/reports/reports.component')
        .then(m => m.ReportsComponent),
  },
];

// selective-preloading.strategy.ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  readonly preloadedRoutes: string[] = [];

  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    if (route.data?.['preload'] === true) {
      this.preloadedRoutes.push(route.path ?? '');
      return load();
    }
    return of(null);
  }
}

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withPreloading } from '@angular/router';
import { routes } from './app.routes';
import { SelectivePreloadingStrategy } from './selective-preloading.strategy';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(SelectivePreloadingStrategy),
    ),
  ],
};

// app.component.ts
import { Component, inject } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';
import { SelectivePreloadingStrategy } from './selective-preloading.strategy';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav>
      <a routerLink="/dashboard">Dashboard</a> |
      <a routerLink="/settings">Settings</a> |
      <a routerLink="/reports">Reports</a>
    </nav>
    <router-outlet />
    <footer>
      Vorgeladene Routen: {{ strategy.preloadedRoutes.join(', ') }}
    </footer>
  `,
})
export class AppComponent {
  strategy = inject(SelectivePreloadingStrategy);
}
```

**Erwartetes Verhalten im Network-Tab:**
- App startet → `dashboard-chunk.js` wird im Hintergrund geladen
- `settings-chunk.js` und `reports-chunk.js` werden erst beim Navigieren geladen

## Weiterführendes

- **`withRouterConfig({ onSameUrlNavigation: 'reload' })`** kombinieren: Verstehe, wie weitere `withFeatures`-Funktionen die Router-Konfiguration ergänzen.
- **Zoneless + Preloading:** Teste, ob deine Custom Strategy auch ohne `zone.js` (mit `provideExperimentalZonelessChangeDetection()`) korrekt funktioniert.
- **Angular-Dokumentation:** [Lazy-loading feature modules](https://angular.dev/guide/ngmodules/lazy-loading) und [Preloading](https://angular.dev/guide/routing/preloading) auf angular.dev.
- **Bundle-Analyse:** Nutze `ng build --stats-json` und den [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer), um die erzeugten Chunks zu visualisieren.

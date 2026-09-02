# Angular Dojo: Custom Route Reuse Strategy
**Datum:** 2026-09-02
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit einer eigenen `RouteReuseStrategy` den Komponentenzustand beim Navigieren beibehältst — etwa den Scroll-Offset oder Formularinhalte — ohne dass Angular die Komponente neu erstellt.

## Hintergrund & Theorie

Standardmäßig zerstört Angular eine Komponente, sobald du von ihrer Route wegnavigierst, und erstellt sie bei Rückkehr neu. Das `RouteReuseStrategy`-Interface ermöglicht es, dieses Verhalten vollständig zu kontrollieren.

Die Strategie besteht aus fünf Methoden:

| Methode | Bedeutung |
|---|---|
| `shouldDetach(route)` | Soll die Komponente beim Verlassen gespeichert werden? |
| `store(route, handle)` | Speichert den detached ComponentRef. |
| `shouldAttach(route)` | Soll ein gespeicherter Handle wiederverwendet werden? |
| `retrieve(route)` | Gibt den gespeicherten Handle zurück. |
| `shouldReuseRoute(future, curr)` | Soll bei gleichem Pfad die Instanz behalten werden? |

**Typischer Use-Case:** Eine Liste mit Scroll-Position soll ihren Zustand behalten, wenn der User ein Detail aufruft und zurückkehrt. Ohne Reuse Strategy beginnt die Liste wieder ganz oben.

Angular aktiviert `shouldReuseRoute` bereits standardmäßig für gleiche Route-Konfigurationsobjekte (gleiche Child-Route bleibt bestehen). Für tiefergehende Kontrolle implementierst du eine eigene Klasse und registrierst sie via `provide`.

## Aufgabe

Baue eine `SimpleRouteReuseStrategy`, die ausgewählte Routen (anhand eines `data.reuse`-Flags in der Route-Config) ihren Zustand behalten lässt. Demonstriere das Verhalten anhand einer Liste mit einem Zähler und einer Detail-Seite.

### Schritte

1. **Erstelle zwei Standalone-Komponenten:** `ListComponent` (mit einem lokalen Zähler und einem Link zur Detail-Seite) und `DetailComponent` (einfache Ansicht mit Zurück-Link).

2. **Konfiguriere die Routen** so, dass die List-Route `data: { reuse: true }` trägt, die Detail-Route nicht.

3. **Implementiere `SimpleRouteReuseStrategy`** in einer eigenen Datei:
   - Speichere `DetachedRouteHandle`-Objekte in einer `Map<string, DetachedRouteHandle>`.
   - Der Map-Key ist der vollständige URL-Pfad der Route.
   - `shouldDetach`: gibt `true` zurück, wenn `route.data['reuse'] === true`.
   - `store`: legt den Handle in der Map ab.
   - `shouldAttach`: gibt `true` zurück, wenn ein Handle für diesen Pfad vorhanden ist.
   - `retrieve`: gibt den gespeicherten Handle zurück.
   - `shouldReuseRoute`: Standardverhalten — `future.routeConfig === curr.routeConfig`.

4. **Registriere die Strategy** in `app.config.ts`:
   ```typescript
   { provide: RouteReuseStrategy, useClass: SimpleRouteReuseStrategy }
   ```

5. **Teste das Verhalten:** Zähler erhöhen → Detail öffnen → zurück → Zähler ist noch vorhanden. Ohne Strategy wäre er zurückgesetzt.

## Hints

<details>
<summary>Hint 1 – Route-Key ableiten</summary>

Ein stabiler Key lässt sich aus dem aktivierten `ActivatedRouteSnapshot` gewinnen:

```typescript
private getKey(route: ActivatedRouteSnapshot): string {
  return route.pathFromRoot
    .map(r => r.routeConfig?.path ?? '')
    .filter(Boolean)
    .join('/');
}
```

</details>

<details>
<summary>Hint 2 – Provider in app.config.ts</summary>

```typescript
import { RouteReuseStrategy } from '@angular/router';
import { SimpleRouteReuseStrategy } from './simple-route-reuse-strategy';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    { provide: RouteReuseStrategy, useClass: SimpleRouteReuseStrategy },
  ],
};
```

</details>

<details>
<summary>Hint 3 – shouldAttach-Falle</summary>

`shouldAttach` wird auch für die allererste Navigation aufgerufen, bevor `store` je aufgerufen wurde. Stelle sicher, dass deine Map-Abfrage sicher ist:

```typescript
shouldAttach(route: ActivatedRouteSnapshot): boolean {
  return !!this.handles.get(this.getKey(route));
}
```

</details>

## Beispiellösung

```typescript
// simple-route-reuse-strategy.ts
import { Injectable } from '@angular/core';
import {
  ActivatedRouteSnapshot,
  DetachedRouteHandle,
  RouteReuseStrategy,
} from '@angular/router';

@Injectable()
export class SimpleRouteReuseStrategy implements RouteReuseStrategy {
  private handles = new Map<string, DetachedRouteHandle>();

  private getKey(route: ActivatedRouteSnapshot): string {
    return route.pathFromRoot
      .map(r => r.routeConfig?.path ?? '')
      .filter(Boolean)
      .join('/');
  }

  shouldDetach(route: ActivatedRouteSnapshot): boolean {
    return route.data['reuse'] === true;
  }

  store(route: ActivatedRouteSnapshot, handle: DetachedRouteHandle): void {
    this.handles.set(this.getKey(route), handle);
  }

  shouldAttach(route: ActivatedRouteSnapshot): boolean {
    return !!this.handles.get(this.getKey(route));
  }

  retrieve(route: ActivatedRouteSnapshot): DetachedRouteHandle | null {
    return this.handles.get(this.getKey(route)) ?? null;
  }

  shouldReuseRoute(
    future: ActivatedRouteSnapshot,
    curr: ActivatedRouteSnapshot,
  ): boolean {
    return future.routeConfig === curr.routeConfig;
  }
}

// list.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  standalone: true,
  imports: [RouterLink],
  template: `
    <h2>Liste</h2>
    <p>Zähler: {{ count }}</p>
    <button (click)="count++">Erhöhen</button>
    <br /><br />
    <a routerLink="/detail">Zum Detail</a>
  `,
})
export class ListComponent {
  count = 0;
}

// app.routes.ts
import { Routes } from '@angular/router';
import { ListComponent } from './list.component';
import { DetailComponent } from './detail.component';

export const routes: Routes = [
  { path: '', component: ListComponent, data: { reuse: true } },
  { path: 'detail', component: DetailComponent },
];

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, RouteReuseStrategy } from '@angular/router';
import { routes } from './app.routes';
import { SimpleRouteReuseStrategy } from './simple-route-reuse-strategy';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    { provide: RouteReuseStrategy, useClass: SimpleRouteReuseStrategy },
  ],
};
```

## Weiterführendes

- **Cache invalidieren:** Füge der Strategy eine `clearCache(path: string)`-Methode hinzu und rufe sie aus einem Service auf, wenn sich Daten ändern — z. B. nach einem erfolgreichen HTTP-POST in der Detail-Ansicht.
- **Memory-Leaks vermeiden:** Bei sehr vielen Routen kann die Map unbegrenzt wachsen. Implementiere eine LRU-Strategie oder lösche den Handle, wenn die Komponente mit `ngOnDestroy` signalisiert, dass ein Reset nötig ist.
- **Offizielle Docs:** [angular.dev/api/router/RouteReuseStrategy](https://angular.dev/api/router/RouteReuseStrategy)

# Angular Dojo: Route Animations – Seitenübergänge animieren
**Datum:** 2026-10-07
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular Animations-System flüssige Übergänge zwischen verschiedenen Routen erzeugst – indem du `RouterOutlet` mit einem `@routeAnimation`-Trigger verknüpfst und Zustandsdaten über `ActivatedRoute` weitergibst.

## Hintergrund & Theorie

Angular ermöglicht Route-Animationen, indem das `<router-outlet>`-Element mit einem `@trigger`-Binding versehen wird. Der Trick: `RouterOutlet` stellt eine `activatedRoute`-Property bereit. Diese wird als Animationszustand gebunden – wenn sich die Route ändert, erkennt Angular den neuen Zustand und startet die passende Transition.

**Kernkonzepte:**

- **`RouterOutlet.activatedRoute`** – Liefert die aktuell aktivierte Route; dient als Animation-State-Wert
- **`routerOutlet.isActivated`** – Gibt `true` zurück, wenn eine Route aktiv ist (wichtig für `:enter`/`:leave`)
- **`transition(':enter', [...])` und `transition(':leave', [...])`** – Animieren neue und verlassene Views
- **`query(':leave', [...])`** – Ermöglicht, die alte View parallel zur neuen zu animieren (Slide-Effekt)
- **`group([...])`** – Führt mehrere Animations-Schritte gleichzeitig aus
- **`style({ position: 'absolute', ... })`** – Notwendig für überlagernde Slide-Effekte, da beide Views gleichzeitig im DOM existieren müssen

**Wichtiger Hinweis:** Die `RouterOutlet`-Komponente muss mit `provideAnimations()` (Standalone) oder `BrowserAnimationsModule` aktiviert sein. Die Animations-Funktion wird im `@Component`-Decorator definiert, nicht im Modul.

```typescript
// Muster für den Template-Zustand
<div [@routeAnimation]="outlet.activatedRoute" style="position: relative;">
  <router-outlet #outlet="outlet" />
</div>
```

## Aufgabe

Erstelle eine Mini-App mit drei Routen (`/home`, `/about`, `/contact`), zwischen denen mit einem **Slide-In/Slide-Out-Effekt** navigiert wird: Die neue Seite schiebt sich von rechts herein, während die alte Seite nach links herausgleitet.

### Schritte

1. **Projekt vorbereiten:** Erstelle drei Standalone-Komponenten (`HomeComponent`, `AboutComponent`, `ContactComponent`) und konfiguriere `provideRouter` mit den drei Routen. Aktiviere Animationen mit `provideAnimations()`.

2. **Animation definieren:** Erstelle eine `routeAnimation`-Funktion in einer separaten `animations.ts`-Datei. Der Trigger soll bei `* => *` folgendes tun:
   - Beide Views (`query(':enter')` und `query(':leave')`) werden auf `position: absolute`, volle Breite gesetzt
   - Die `:enter`-View startet bei `translateX(100%)` und fährt auf `translateX(0%)`
   - Die `:leave`-View startet bei `translateX(0%)` und fährt auf `translateX(-100%)`
   - Nutze `group([...])` damit beide Animationen gleichzeitig laufen

3. **Trigger im App-Template verdrahten:** Im `AppComponent`-Template referenzierst du den `router-outlet` via Template-Variable (`#outlet="outlet"`) und bindest `[(@routeAnimation)]="outlet.activatedRoute"` an ein umschließendes `<div>`.

4. **Navigation hinzufügen:** Füge Links (`routerLink`) zur Navigation hinzu und teste alle drei Übergänge.

5. **Optional – Richtungsabhängige Animation:** Übergib der Route eine `data: { animIndex: number }`-Property und berechne im `AppComponent`, ob die Navigation vorwärts oder rückwärts geht. Animiere entsprechend von links oder rechts.

## Hints

<details>
<summary>Hint 1 – Warum erscheint keine Animation?</summary>

Häufigste Ursache: Das umschließende `<div>` hat kein `position: relative` / `overflow: hidden` und die beiden Views liegen nicht tatsächlich übereinander. Stelle sicher:

```html
<div [@routeAnimation]="outlet.activatedRoute"
     style="position: relative; overflow: hidden; min-height: 100vh;">
  <router-outlet #outlet="outlet" />
</div>
```

Und in der Animation selbst: `query(':enter, :leave', style({ position: 'absolute', width: '100%', top: 0 }), { optional: true })`

Das `optional: true` verhindert Fehler, wenn beim ersten Laden noch kein `:leave`-Element vorhanden ist.

</details>

<details>
<summary>Hint 2 – Wie baue ich den `trigger` korrekt auf?</summary>

```typescript
import { trigger, transition, style, query, group, animate } from '@angular/animations';

export const routeAnimation = trigger('routeAnimation', [
  transition('* <=> *', [
    query(':enter, :leave', [
      style({ position: 'absolute', width: '100%', top: 0 })
    ], { optional: true }),
    group([
      query(':leave', [
        animate('300ms ease-in', style({ transform: 'translateX(-100%)' }))
      ], { optional: true }),
      query(':enter', [
        style({ transform: 'translateX(100%)' }),
        animate('300ms ease-out', style({ transform: 'translateX(0%)' }))
      ], { optional: true })
    ])
  ])
]);
```

Im `@Component`-Decorator: `animations: [routeAnimation]`

</details>

## Beispiellösung

```typescript
// animations.ts
import { trigger, transition, style, query, group, animate } from '@angular/animations';

export const routeAnimation = trigger('routeAnimation', [
  transition('* <=> *', [
    query(':enter, :leave', [
      style({ position: 'absolute', top: 0, width: '100%' })
    ], { optional: true }),
    group([
      query(':leave', [
        animate('300ms ease-in', style({ transform: 'translateX(-100%)' }))
      ], { optional: true }),
      query(':enter', [
        style({ transform: 'translateX(100%)' }),
        animate('300ms ease-out', style({ transform: 'translateX(0%)' }))
      ], { optional: true })
    ])
  ])
]);
```

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';
import { routeAnimation } from './animations';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  animations: [routeAnimation],
  template: `
    <nav>
      <a routerLink="/home">Home</a>
      <a routerLink="/about">About</a>
      <a routerLink="/contact">Contact</a>
    </nav>

    <div [@routeAnimation]="outlet.activatedRoute"
         style="position: relative; overflow: hidden; min-height: 200px;">
      <router-outlet #outlet="outlet" />
    </div>
  `
})
export class AppComponent {}
```

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: 'home',    loadComponent: () => import('./home.component').then(m => m.HomeComponent) },
  { path: 'about',   loadComponent: () => import('./about.component').then(m => m.AboutComponent) },
  { path: 'contact', loadComponent: () => import('./contact.component').then(m => m.ContactComponent) },
  { path: '',        redirectTo: 'home', pathMatch: 'full' }
];
```

```typescript
// main.ts (Standalone Bootstrap)
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimations } from '@angular/platform-browser/animations';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideAnimations(),
    provideRouter(routes)
  ]
});
```

## Weiterführendes
- **Richtungsabhängige Animationen:** Speichere in `data: { animIndex: 1 }` pro Route einen Index. Im `AppComponent` vergleiche den alten und neuen Index in `router.events` (NavigationEnd) und übergib statt `outlet.activatedRoute` einen eigenen State-String (`'forward'` / `'backward'`). Definiere dann zwei separate `transition`-Blöcke.
- **`routerOutlet.activatedRouteData`** als Alternative zum rohen `activatedRoute`-Objekt, wenn du Route-`data` direkt im Trigger verwenden möchtest.
- Offizielle Angular-Docs: [Route transition animations](https://angular.dev/guide/animations/route-animations)

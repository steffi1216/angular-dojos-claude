# Angular Dojo: View Transitions API
**Datum:** 2026-07-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du die native Browser-View-Transitions-API zusammen mit dem Angular Router nutzt, um flüssige, performante Seitenübergänge ohne externe Animationsbibliotheken zu realisieren.

## Hintergrund & Theorie

Die [View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) ist eine native Browser-API (Chrome 111+, Firefox 130+), die sanfte visuelle Übergänge zwischen DOM-Zuständen ermöglicht – ähnlich wie Animationen in nativen Apps.

Ab Angular 17 kann die API direkt über `withViewTransitions()` in die Router-Konfiguration eingebunden werden:

```typescript
provideRouter(routes, withViewTransitions())
```

Der Router ruft automatisch `document.startViewTransition()` bei jeder Navigation auf. Standardmäßig erzeugt das einen einfachen Cross-Fade. Durch CSS lässt sich das Verhalten weitreichend anpassen:

```css
/* Standard-Fade anpassen */
::view-transition-old(root) { animation-duration: 300ms; }
::view-transition-new(root) { animation-duration: 300ms; }
```

Mit benannten View-Transition-Targets (`view-transition-name`) kann man einzelne Elemente animieren (sog. **shared element transitions**): Das Browser-Snapshot des alten Elements wird fließend ins neue übergeblendet – ohne JavaScript.

`withViewTransitions()` akzeptiert eine Konfiguration mit einem `onViewTransitionCreated`-Callback, über den du z. B. Übergänge bei `popstate`-Navigation (Zurück/Vor) skippen kannst oder eigene Logik hook-en kannst.

## Aufgabe

Erstelle eine kleine Angular-App mit zwei Routen (`/list` und `/detail/:id`). Beim Navigieren soll ein Kartenelement aus der Liste animiert in die Detailansicht übergehen (Shared Element Transition), und der Übergang bei Vor-/Zurück-Navigation soll mit einer anderen Richtungsanimation laufen.

### Schritte

1. **Router konfigurieren** – Binde `withViewTransitions()` in `provideRouter()` ein. Füge den Callback `onViewTransitionCreated` hinzu, der prüft, ob die Navigation über `popstate` (Zurück/Vor) ausgelöst wurde, und in diesem Fall den Übergang via `transition.skipTransition()` abbricht oder eine eigene CSS-Klasse am `<html>`-Element setzt.

2. **Listenkomponente** – Rendere 5 Karten. Jede Karte bekommt ein individuelles `view-transition-name` (z. B. `view-transition-name: card-1`), das per Style-Binding gesetzt wird: `[style.view-transition-name]="'card-' + item.id"`.

3. **Detailkomponente** – Lese die `id` aus den Router-Params (nutze `inject(ActivatedRoute)` oder `input()`-Binding via `withComponentInputBinding()`). Setze dasselbe `view-transition-name` wie die entsprechende Karte in der Liste.

4. **CSS** – Definiere die View-Transition-Animationen:
   - Globaler Fade für die Gesamtseite
   - Slide-In/Out für die Vorwärts-Navigation (`::view-transition-old(root)` / `::view-transition-new(root)`)
   - Anderes Slide-Verhalten bei Rückwärts-Navigation (via CSS-Klasse, die du im Callback setzt)

5. **Testen** – Navigiere zwischen Liste und Detail und beobachte den Shared-Element-Übergang. Prüfe auch den Zurück-Button.

## Hints

<details>
<summary>Hint 1 – onViewTransitionCreated und NavigationStart</summary>

```typescript
import { NavigationStart } from '@angular/router';

withViewTransitions({
  onViewTransitionCreated: ({ transition, currentNavigation }) => {
    const trigger = currentNavigation.trigger; // 'popstate' | 'imperative' | 'hashchange'
    if (trigger === 'popstate') {
      document.documentElement.classList.add('back-navigation');
      transition.finished.finally(() => {
        document.documentElement.classList.remove('back-navigation');
      });
    }
  }
})
```

`transition` ist ein `ViewTransition`-Objekt mit den Promises `ready`, `updateCallbackDone` und `finished`.

</details>

<details>
<summary>Hint 2 – view-transition-name per Style-Binding</summary>

```typescript
@Component({
  template: `
    @for (item of items; track item.id) {
      <div class="card"
           [style.view-transition-name]="'card-' + item.id"
           (click)="goToDetail(item.id)">
        {{ item.title }}
      </div>
    }
  `
})
export class ListComponent {
  items = [
    { id: 1, title: 'Item A' },
    { id: 2, title: 'Item B' },
    // ...
  ];
  private router = inject(Router);
  goToDetail(id: number) { this.router.navigate(['/detail', id]); }
}
```

In der Detailkomponente:
```typescript
@Component({
  template: `
    <div class="detail-card" [style.view-transition-name]="'card-' + id()">
      Detail für {{ id() }}
    </div>
  `
})
export class DetailComponent {
  id = input.required<string>(); // via withComponentInputBinding()
}
```

</details>

<details>
<summary>Hint 3 – CSS für Richtungsanimation</summary>

```css
/* Vorwärts-Navigation */
::view-transition-old(root) {
  animation: slide-out-left 250ms ease-in forwards;
}
::view-transition-new(root) {
  animation: slide-in-right 250ms ease-out forwards;
}

/* Rückwärts-Navigation */
.back-navigation::view-transition-old(root) {
  animation: slide-out-right 250ms ease-in forwards;
}
.back-navigation::view-transition-new(root) {
  animation: slide-in-left 250ms ease-out forwards;
}

@keyframes slide-out-left  { to   { transform: translateX(-100%); } }
@keyframes slide-in-right  { from { transform: translateX( 100%); } }
@keyframes slide-out-right { to   { transform: translateX( 100%); } }
@keyframes slide-in-left   { from { transform: translateX(-100%); } }
```

</details>

## Beispiellösung

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withViewTransitions, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),
      withViewTransitions({
        onViewTransitionCreated: ({ transition, currentNavigation }) => {
          if (currentNavigation.trigger === 'popstate') {
            document.documentElement.classList.add('back-navigation');
            transition.finished.finally(() =>
              document.documentElement.classList.remove('back-navigation')
            );
          }
        }
      })
    )
  ]
};

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: 'list',       loadComponent: () => import('./list/list.component').then(m => m.ListComponent) },
  { path: 'detail/:id', loadComponent: () => import('./detail/detail.component').then(m => m.DetailComponent) },
  { path: '**', redirectTo: 'list' }
];

// list/list.component.ts
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  standalone: true,
  template: `
    <h1>Bücher</h1>
    @for (book of books; track book.id) {
      <div class="card"
           [style.view-transition-name]="'book-' + book.id"
           (click)="router.navigate(['/detail', book.id])">
        <strong>{{ book.title }}</strong>
        <span>{{ book.author }}</span>
      </div>
    }
  `,
  styles: [`
    .card {
      padding: 1rem;
      margin: 0.5rem 0;
      border: 1px solid #ccc;
      border-radius: 8px;
      cursor: pointer;
      contain: layout; /* wichtig für view-transition-name */
    }
  `]
})
export class ListComponent {
  router = inject(Router);
  books = [
    { id: 1, title: 'Clean Code', author: 'Robert C. Martin' },
    { id: 2, title: 'The Pragmatic Programmer', author: 'Hunt & Thomas' },
    { id: 3, title: 'Designing Data-Intensive Applications', author: 'Martin Kleppmann' },
    { id: 4, title: 'Domain-Driven Design', author: 'Eric Evans' },
    { id: 5, title: 'Refactoring', author: 'Martin Fowler' },
  ];
}

// detail/detail.component.ts
import { Component, input } from '@angular/core';
import { RouterLink } from '@angular/router';

const BOOKS: Record<string, { title: string; author: string; description: string }> = {
  '1': { title: 'Clean Code', author: 'Robert C. Martin', description: 'Über lesbare, wartbare Software.' },
  '2': { title: 'The Pragmatic Programmer', author: 'Hunt & Thomas', description: 'Pragmatische Softwareentwicklung.' },
  '3': { title: 'Designing Data-Intensive Applications', author: 'Martin Kleppmann', description: 'Skalierbare Datensysteme.' },
  '4': { title: 'Domain-Driven Design', author: 'Eric Evans', description: 'Komplexe Domänenmodellierung.' },
  '5': { title: 'Refactoring', author: 'Martin Fowler', description: 'Code sicher verbessern.' },
};

@Component({
  standalone: true,
  imports: [RouterLink],
  template: `
    @if (book(); as b) {
      <div class="detail-card" [style.view-transition-name]="'book-' + id()">
        <h2>{{ b.title }}</h2>
        <p><em>{{ b.author }}</em></p>
        <p>{{ b.description }}</p>
      </div>
    }
    <a routerLink="/list">← Zurück zur Liste</a>
  `,
  styles: [`
    .detail-card {
      padding: 2rem;
      border: 2px solid #333;
      border-radius: 12px;
      margin-bottom: 1rem;
      contain: layout;
    }
  `]
})
export class DetailComponent {
  id = input.required<string>();
  book = () => BOOKS[this.id()];
}
```

```css
/* styles.css – globale Animationen */
::view-transition-old(root) {
  animation: vt-slide-out-left 280ms cubic-bezier(0.4, 0, 0.2, 1) forwards;
}
::view-transition-new(root) {
  animation: vt-slide-in-right 280ms cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

.back-navigation::view-transition-old(root) {
  animation: vt-slide-out-right 280ms cubic-bezier(0.4, 0, 0.2, 1) forwards;
}
.back-navigation::view-transition-new(root) {
  animation: vt-slide-in-left 280ms cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

@keyframes vt-slide-out-left  { to   { transform: translateX(-30%); opacity: 0; } }
@keyframes vt-slide-in-right  { from { transform: translateX( 30%); opacity: 0; } }
@keyframes vt-slide-out-right { to   { transform: translateX( 30%); opacity: 0; } }
@keyframes vt-slide-in-left   { from { transform: translateX(-30%); opacity: 0; } }

/* Shared Element Transition für Karten */
::view-transition-old(book-1),
::view-transition-new(book-1),
::view-transition-old(book-2),
::view-transition-new(book-2) {
  animation-duration: 400ms;
}
```

## Weiterführendes

- **Browser-Support prüfen:** Nutze `if (document.startViewTransition)` als Progressive Enhancement – Angular's `withViewTransitions()` macht das intern bereits, liefert aber keinen Fallback-Indikator.
- **`skipTransition()`** kann nützlich sein, wenn der Nutzer `prefers-reduced-motion` gesetzt hat: Prüfe `window.matchMedia('(prefers-reduced-motion: reduce)').matches` im Callback.
- Für komplexere Sequenzanimationen (mehrere Elemente nacheinander) lies die Spec zu **Multiple View Transitions** und dem `types`-Parameter (Chrome 125+).
- Offizielle Doku: [Angular Router – withViewTransitions](https://angular.dev/api/router/withViewTransitions) und [MDN View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API).

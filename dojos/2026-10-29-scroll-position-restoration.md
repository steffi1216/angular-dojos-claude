# Angular Dojo: Scroll Position Restoration & ViewportScroller
**Datum:** 2026-10-29
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular die Scroll-Position beim Navigieren verwaltet und wie du mit `ViewportScroller` eigene Scroll-Strategien implementierst – einschließlich Anchor-Scrolling und benutzerdefinierter Scroll-Offsets.

## Hintergrund & Theorie

Standardmäßig setzt eine Angular SPA die Scroll-Position beim Navigieren **nicht** zurück – der Browser bleibt an der alten Position, was sich für Nutzer seltsam anfühlt. Angular bietet dafür zwei eingebaute Mechanismen:

**`scrollPositionRestoration`** (in `RouterModule.forRoot` bzw. `provideRouter`):
- `'disabled'` (Standard): Kein automatisches Scroll-Handling
- `'top'`: Scrollt bei jeder Navigation nach oben
- `'enabled'`: Speichert & stellt die Position wieder her (ideal für Zurück-Navigation)

**`anchorScrolling`** ermöglicht das Springen zu `#fragment`-Links innerhalb der Seite.

Der `ViewportScroller` (aus `@angular/common`) ist der zugrundeliegende Service. Er abstrahiert `window.scroll*`-Aufrufe und kann durch `@angular/common/browser` oder eine eigene Implementierung ersetzt werden.

Wichtig: `scrollPositionRestoration: 'enabled'` funktioniert nicht gut mit Lazy Loading + langen Listen, da die Seite beim Wiederherstellen noch lädt. Hier braucht man Custom-Logik mit `NavigationEnd`-Events und `ViewportScroller`.

## Aufgabe

Baue eine kleine Angular-App mit zwei Routen:
1. Eine **Liste** mit 50 Einträgen (z.B. Artikel-Liste), sodass man scrollen muss
2. Eine **Detail-Seite** für jeden Eintrag

Implementiere dann:
- Beim Navigieren zur Detail-Seite: Scroll nach oben
- Beim Zurücknavigieren zur Liste: Ursprüngliche Scroll-Position wiederherstellen
- Ein `#section2`-Anchor in der Liste, zu dem über einen Link gesprungen werden kann
- Einen `scrollOffset` von `[0, 64]` (für eine fixe Navbar mit 64px Höhe)

### Schritte

1. **App konfigurieren** – `scrollPositionRestoration` und `anchorScrolling` in `provideRouter` setzen:

```typescript
// app.config.ts
import { provideRouter, withInMemoryScrolling } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withInMemoryScrolling({
        scrollPositionRestoration: 'enabled',
        anchorScrolling: 'enabled',
      })
    ),
  ],
};
```

2. **Scroll-Offset für fixe Navbar** – `ViewportScroller` im App-Komponent oder Root konfigurieren:

```typescript
// app.component.ts
import { ViewportScroller } from '@angular/common';

@Component({ ... })
export class AppComponent {
  constructor(private scroller: ViewportScroller) {
    this.scroller.setOffset([0, 64]);
  }
}
```

3. **Artikel-Liste** aufbauen – 50 Items rendern, in der Mitte eine Section mit `id="section2"`, und einen `<a routerLink="." fragment="section2">Zu Section 2</a>` Link.

4. **Manuelles Scroll-Handling** für Lazy-Load-Szenarien – Nutze den Router und `NavigationEnd`, um nach vollständigem Laden manuell zu scrollen:

```typescript
// list.component.ts
import { Router, NavigationEnd } from '@angular/router';
import { ViewportScroller } from '@angular/common';
import { filter } from 'rxjs/operators';

@Component({ ... })
export class ListComponent implements OnInit {
  constructor(
    private router: Router,
    private scroller: ViewportScroller
  ) {}

  ngOnInit() {
    this.router.events.pipe(
      filter(e => e instanceof NavigationEnd)
    ).subscribe(() => {
      // Position wird automatisch durch 'enabled' wiederhergestellt
      // Hier könntest du nach dem Laden von Daten manuell scrollen:
      // this.scroller.scrollToPosition([0, savedY]);
    });
  }
}
```

5. **Programmatisches Scrolling testen** – Füge Buttons hinzu, die `ViewportScroller`-Methoden aufrufen:
   - `scroller.scrollToAnchor('section2')`
   - `scroller.scrollToPosition([0, 0])`
   - `scroller.getScrollPosition()` (in der Konsole ausgeben)

## Hints

<details>
<summary>Hint 1 – withInMemoryScrolling vs. withScroll</summary>

In Angular 15+ wird `withInMemoryScrolling(...)` als Feature-Funktion an `provideRouter` übergeben. In älteren Apps mit `RouterModule.forRoot` verwendet man das zweite Argument:
```typescript
RouterModule.forRoot(routes, {
  scrollPositionRestoration: 'enabled',
  anchorScrolling: 'enabled',
})
```
</details>

<details>
<summary>Hint 2 – Scroll-Restoration mit Lazy Loading</summary>

Wenn Inhalte asynchron laden (z.B. eine HTTP-Liste), ist die Seite beim Scroll-Restore noch nicht in der vollen Höhe gerendert. Speichere die Position manuell vor der Navigation:

```typescript
const pos = this.scroller.getScrollPosition();
this.router.navigate(['/detail', id]);
// Beim Zurücknavigieren:
this.scroller.scrollToPosition(pos);
```
Oder nutze einen Service, der die Position pro Route speichert.
</details>

<details>
<summary>Hint 3 – Fragment-Scrolling mit Offset</summary>

Der `scrollOffset` wird automatisch auf Anchor-Scrolling angewendet. Der Router springt bei `[routerLink]="['.']" [fragment]="'section2'"` zum Element mit `id="section2"` und zieht den Offset ab, damit der Inhalt nicht hinter der fixen Navbar versteckt ist.
</details>

## Beispiellösung

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withInMemoryScrolling } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withInMemoryScrolling({
        scrollPositionRestoration: 'enabled',
        anchorScrolling: 'enabled',
      })
    ),
  ],
};

// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';
import { ViewportScroller } from '@angular/common';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav style="position:fixed;top:0;height:64px;width:100%;background:#333;color:#fff;display:flex;align-items:center;padding:0 16px;z-index:10">
      Angular Scroll Demo
      <a routerLink="/list" style="color:#fff;margin-left:16px">Liste</a>
    </nav>
    <div style="margin-top:64px">
      <router-outlet />
    </div>
  `,
})
export class AppComponent {
  constructor(private scroller: ViewportScroller) {
    this.scroller.setOffset([0, 64]);
  }
}

// list.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-list',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div style="padding:16px">
      <h1>Artikel-Liste</h1>
      <a routerLink="." fragment="section2">→ Zu Section 2 springen</a>

      @for (item of items.slice(0, 25); track item.id) {
        <div style="padding:12px;border-bottom:1px solid #eee">
          <a [routerLink]="['/detail', item.id]">{{ item.title }}</a>
        </div>
      }

      <h2 id="section2" style="margin-top:32px">Section 2</h2>
      <p>Du bist zu Section 2 gescrollt!</p>

      @for (item of items.slice(25); track item.id) {
        <div style="padding:12px;border-bottom:1px solid #eee">
          <a [routerLink]="['/detail', item.id]">{{ item.title }}</a>
        </div>
      }
    </div>
  `,
})
export class ListComponent {
  items = Array.from({ length: 50 }, (_, i) => ({
    id: i + 1,
    title: `Artikel ${i + 1}`,
  }));
}

// detail.component.ts
import { Component } from '@angular/core';
import { ActivatedRoute, RouterLink } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-detail',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div style="padding:16px">
      <a routerLink="/list">← Zurück zur Liste</a>
      <h1>Detail für Artikel {{ id() }}</h1>
      <p>Hier stehen die Details. Navigiere zurück – die Scroll-Position der Liste wird wiederhergestellt.</p>
    </div>
  `,
})
export class DetailComponent {
  id = toSignal(
    this.route.paramMap.pipe(map(p => p.get('id'))),
    { initialValue: '' }
  );

  constructor(private route: ActivatedRoute) {}
}

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'list', pathMatch: 'full' },
  { path: 'list', loadComponent: () => import('./list.component').then(m => m.ListComponent) },
  { path: 'detail/:id', loadComponent: () => import('./detail.component').then(m => m.DetailComponent) },
];
```

## Weiterführendes

- **Custom `ScrollStrategy`**: Implementiere das `ViewportScroller`-Interface selbst (z.B. für Smooth-Scrolling oder Route-basierte Animationen mit Position-Memory in einem Signal-Store).
- **Scroll-Events beobachten**: `router.events.pipe(filter(e => e instanceof Scroll))` – das `Scroll`-Event enthält `position`, `anchor` und `routerEvent`, was gezieltes Custom-Handling ermöglicht.
- [Angular Docs – ViewportScroller](https://angular.dev/api/common/ViewportScroller)
- [Angular Docs – withInMemoryScrolling](https://angular.dev/api/router/withInMemoryScrolling)

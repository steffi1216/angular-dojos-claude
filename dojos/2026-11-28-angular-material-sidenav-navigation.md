# Angular Dojo: Angular Material Sidenav & Responsive Navigation
**Datum:** 2026-11-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `MatSidenav` ein responsives Navigation-Layout baust, das sich automatisch an verschiedene Bildschirmgrößen anpasst – auf dem Desktop als dauerhaft sichtbare Seitenleiste, auf Mobilgeräten als überlagertes Drawer-Menü.

## Hintergrund & Theorie
`MatSidenav` ist Teil von `@angular/material/sidenav` und besteht aus drei Haupt-Elementen:

- **`<mat-sidenav-container>`** – der äußere Wrapper, der den Inhaltsbereich und die Seitenleisten koordiniert
- **`<mat-sidenav>`** – die eigentliche Seitenleiste (es kann bis zu zwei geben: `position="start"` und `position="end"`)
- **`<mat-sidenav-content>`** – der Hauptinhaltsbereich

Das wichtigste Steuerungselement ist das `mode`-Property:
- **`side`** – die Sidenav ist neben dem Content (kein Overlay, schiebt den Content zur Seite)
- **`over`** – legt sich über den Content (ideal für mobile Geräte)
- **`push`** – schiebt den Content beiseite ohne ihn zu verkleinern

Für responsives Verhalten kombiniert man `MatSidenav` mit dem `BreakpointObserver` aus `@angular/cdk/layout`. Der `BreakpointObserver` liefert ein Observable, das bei Breakpoint-Änderungen emittiert – ideal um `mode` und `opened`-State reaktiv zu steuern.

Mit Signals lässt sich der Navigation-State noch sauberer kapseln: Ein `signal<boolean>` für `isOpen` und ein `computed()` für `mode` sorgen für eine komplett reaktive, zoneless-kompatible Lösung.

## Aufgabe
Baue ein responsives App-Shell-Layout mit einer `MatSidenav`:

- Auf **Desktop** (> 768px): Sidenav ist dauerhaft geöffnet im `side`-Mode (kein Hamburger-Button nötig)
- Auf **Mobilgeräten** (≤ 768px): Sidenav ist versteckt, öffnet/schließt über einen Hamburger-Button im `MatToolbar`, Mode ist `over`

Die Navigation soll drei Links enthalten (Home, Dashboard, Settings), die mit dem Angular Router verknüpft sind. Beim Klick auf einen Link auf mobilen Geräten soll sich die Sidenav automatisch schließen.

### Schritte
1. Erstelle eine `AppShellComponent` als Standalone Component und importiere `MatSidenavModule`, `MatToolbarModule`, `MatListModule`, `MatIconModule`, `RouterModule` und `AsyncPipe`.
2. Injiziere `BreakpointObserver` und erstelle ein `isMobile$` Observable (Breakpoint: `'(max-width: 768px)'`).
3. Erstelle im Template das `<mat-sidenav-container>` Layout mit `<mat-sidenav>` und `<mat-sidenav-content>`. Binde `mode` und `opened` reaktiv mit `async pipe` oder Signals.
4. Füge im `<mat-sidenav>` eine `<mat-nav-list>` mit drei `<a mat-list-item routerLink>` Einträgen ein.
5. Implementiere die Auto-Close-Logik: Beim Klick auf einen Nav-Link auf mobilen Geräten wird die Sidenav geschlossen (via `sidenav.close()` oder Signal-Update).
6. Stelle sicher, dass ein `MatToolbar` im Content-Bereich einen Hamburger-Button (`<button mat-icon-button>`) zeigt – aber nur auf mobilen Geräten (`@if (isMobile$ | async)`).

## Hints
<details>
<summary>Hint 1 – BreakpointObserver & Signals verbinden</summary>

Nutze `toSignal()` aus `@angular/core/rxjs-interop`, um das Observable in ein Signal zu konvertieren:

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { BreakpointObserver } from '@angular/cdk/layout';

readonly isMobile = toSignal(
  inject(BreakpointObserver)
    .observe('(max-width: 768px)')
    .pipe(map(result => result.matches)),
  { initialValue: false }
);

readonly sidenavMode = computed(() => this.isMobile() ? 'over' : 'side');
readonly sidenavOpened = signal(true);
```

Beim Breakpoint-Wechsel zu Mobile willst du die Sidenav schließen und bei Desktop öffnen – das geht am saubersten mit einem `effect()`.
</details>

<details>
<summary>Hint 2 – Auto-Close beim Navigation-Link-Klick</summary>

Greife auf die `MatSidenav`-Instanz über `viewChild` zu und rufe `close()` auf:

```typescript
readonly sidenav = viewChild.required(MatSidenav);

onNavLinkClick(): void {
  if (this.isMobile()) {
    this.sidenav().close();
  }
}
```

Im Template:
```html
<a mat-list-item routerLink="/home" (click)="onNavLinkClick()">
  <mat-icon matListItemIcon>home</mat-icon>
  <span matListItemTitle>Home</span>
</a>
```
</details>

## Beispiellösung

```typescript
import { Component, computed, effect, inject, signal, viewChild } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { BreakpointObserver } from '@angular/cdk/layout';
import { RouterModule } from '@angular/router';
import { AsyncPipe } from '@angular/common';
import { map } from 'rxjs/operators';
import { MatSidenav, MatSidenavModule } from '@angular/material/sidenav';
import { MatToolbarModule } from '@angular/material/toolbar';
import { MatListModule } from '@angular/material/list';
import { MatIconModule } from '@angular/material/icon';
import { MatButtonModule } from '@angular/material/button';

@Component({
  selector: 'app-shell',
  standalone: true,
  imports: [
    RouterModule,
    AsyncPipe,
    MatSidenavModule,
    MatToolbarModule,
    MatListModule,
    MatIconModule,
    MatButtonModule,
  ],
  template: `
    <mat-sidenav-container class="shell-container">
      <mat-sidenav
        #sidenav
        [mode]="sidenavMode()"
        [opened]="sidenavOpened()"
        (openedChange)="sidenavOpened.set($event)"
      >
        <mat-nav-list>
          <a mat-list-item routerLink="/home" routerLinkActive="active" (click)="onNavLinkClick()">
            <mat-icon matListItemIcon>home</mat-icon>
            <span matListItemTitle>Home</span>
          </a>
          <a mat-list-item routerLink="/dashboard" routerLinkActive="active" (click)="onNavLinkClick()">
            <mat-icon matListItemIcon>dashboard</mat-icon>
            <span matListItemTitle>Dashboard</span>
          </a>
          <a mat-list-item routerLink="/settings" routerLinkActive="active" (click)="onNavLinkClick()">
            <mat-icon matListItemIcon>settings</mat-icon>
            <span matListItemTitle>Settings</span>
          </a>
        </mat-nav-list>
      </mat-sidenav>

      <mat-sidenav-content>
        <mat-toolbar color="primary">
          @if (isMobile()) {
            <button mat-icon-button (click)="sidenavOpened.set(!sidenavOpened())">
              <mat-icon>menu</mat-icon>
            </button>
          }
          <span>My App</span>
        </mat-toolbar>

        <div class="content">
          <router-outlet />
        </div>
      </mat-sidenav-content>
    </mat-sidenav-container>
  `,
  styles: [`
    .shell-container {
      height: 100vh;
    }
    mat-sidenav {
      width: 240px;
    }
    .content {
      padding: 16px;
    }
    a.active {
      background: rgba(0, 0, 0, 0.08);
    }
  `],
})
export class AppShellComponent {
  private readonly breakpointObserver = inject(BreakpointObserver);
  readonly sidenavRef = viewChild.required(MatSidenav);

  readonly isMobile = toSignal(
    this.breakpointObserver
      .observe('(max-width: 768px)')
      .pipe(map(result => result.matches)),
    { initialValue: false }
  );

  readonly sidenavMode = computed(() => (this.isMobile() ? 'over' : 'side') as 'over' | 'side');
  readonly sidenavOpened = signal(true);

  constructor() {
    // Sidenav state beim Breakpoint-Wechsel anpassen
    effect(() => {
      this.sidenavOpened.set(!this.isMobile());
    });
  }

  onNavLinkClick(): void {
    if (this.isMobile()) {
      this.sidenavOpened.set(false);
    }
  }
}
```

## Weiterführendes
- Schau dir `MatDrawer` an – die Low-Level-Variante ohne den Container-Wrapper, die mehr manuelle Kontrolle bietet.
- Für komplexe Navigation-Layouts (z.B. zwei Sidnavs gleichzeitig oder eine Sidenav mit `position="end"` als Detail-Panel) empfiehlt sich die [offizielle MatSidenav-Dokumentation](https://material.angular.io/components/sidenav/overview).
- Kombiniere die Sidenav mit `RouterLinkActive` und einem `[activated]`-Input für visuelles Feedback des aktuellen Routen-Zustands.

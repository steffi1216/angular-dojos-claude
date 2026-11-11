# Angular Dojo: CDK BreakpointObserver
**Datum:** 2026-11-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `BreakpointObserver` aus `@angular/cdk/layout` responsives Verhalten in Angular-Komponenten reaktiv und signal-basiert umsetzt – ohne CSS-only-Ansätze oder manuelle `window.resize`-Events.

## Hintergrund & Theorie

Das Angular CDK bietet mit `BreakpointObserver` einen Service, der Media Queries als RxJS-Observable zurückgibt. Er löst zwei häufige Probleme:

**Problem 1:** `window.matchMedia` ist nicht server-seitig verfügbar und erfordert manuelles Cleanup.

**Problem 2:** Responsives Verhalten in der Komponentenlogik (z. B. Layout-Varianten, Lazy Loading, unterschiedliche Daten) lässt sich mit reinem CSS nicht abbilden.

`BreakpointObserver.observe(query)` gibt ein Observable aus, das bei jeder Änderung des Breakpoint-Status feuert:

```typescript
interface BreakpointState {
  matches: boolean;
  breakpoints: { [key: string]: boolean };
}
```

Das CDK liefert vordefinierte Breakpoints über die `Breakpoints`-Konstante (`Breakpoints.Handset`, `Breakpoints.Tablet`, `Breakpoints.Web`, `Breakpoints.HandsetPortrait` usw.), die auf Material Design-Werten basieren. Du kannst aber auch eigene Media-Query-Strings übergeben.

Seit Angular 16+ lässt sich das Observable elegant mit `toSignal()` in ein Signal umwandeln, was die Template-Integration vereinfacht und Change Detection mit OnPush effizient macht.

## Aufgabe

Erstelle eine `ResponsiveLayoutComponent`, die je nach Breakpoint unterschiedliche Layouts rendert:
- **Mobile (Handset):** Eine einspaltige Karte mit kompakter Darstellung
- **Tablet:** Zweispaltige Grid-Ansicht
- **Desktop:** Dreispaltige Grid-Ansicht mit Seitenleiste

Zusätzlich soll ein Service `LayoutService` den aktuellen Breakpoint als Signal bereitstellen, damit andere Komponenten darauf reagieren können.

### Schritte

1. **LayoutService erstellen** (`layout.service.ts`): Injiziere `BreakpointObserver` und erstelle mit `toSignal()` Signals für `isMobile`, `isTablet` und `isDesktop`. Verwende `Breakpoints.Handset`, `Breakpoints.Tablet` und `Breakpoints.Web`.

2. **ResponsiveLayoutComponent erstellen**: Injiziere den `LayoutService` via `inject()`. Leite ein `computed()` Signal `columns` ab, das 1, 2 oder 3 zurückgibt. Setze `changeDetection: ChangeDetectionStrategy.OnPush`.

3. **Template bauen**: Nutze das `@if`/`@else`-Control-Flow (oder `[ngClass]`) um unterschiedliche Layout-Klassen anzuwenden. Zeige die aktuelle Spaltenanzahl und den Gerätetyp an.

4. **Custom Breakpoint hinzufügen**: Ergänze im Service einen eigenen Breakpoint `isWidescreen` für `'(min-width: 1600px)'`.

5. **Cleanup prüfen**: Stelle sicher, dass kein manuelles `unsubscribe()` nötig ist – `toSignal()` übernimmt das Cleanup automatisch im Injection Context.

## Hints

<details>
<summary>Hint 1 – BreakpointObserver mit toSignal verbinden</summary>

```typescript
import { inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';
import { map } from 'rxjs';

// Im Service (muss in einem Injection Context ausgeführt werden):
const breakpointObserver = inject(BreakpointObserver);

const handsetState = toSignal(
  breakpointObserver.observe(Breakpoints.Handset).pipe(
    map(state => state.matches)
  ),
  { initialValue: false }
);
```

</details>

<details>
<summary>Hint 2 – Mehrere Breakpoints gleichzeitig beobachten</summary>

```typescript
// Mehrere Queries in einem observe()-Aufruf:
const layoutSignal = toSignal(
  breakpointObserver
    .observe([Breakpoints.Handset, Breakpoints.Tablet, Breakpoints.Web])
    .pipe(
      map(state => ({
        isMobile: state.breakpoints[Breakpoints.HandsetPortrait] ||
                  state.breakpoints[Breakpoints.HandsetLandscape],
        isTablet: state.breakpoints[Breakpoints.TabletPortrait] ||
                  state.breakpoints[Breakpoints.TabletLandscape],
        isDesktop: state.breakpoints[Breakpoints.WebPortrait] ||
                   state.breakpoints[Breakpoints.WebLandscape],
      }))
    ),
  { initialValue: { isMobile: false, isTablet: false, isDesktop: true } }
);
```

</details>

## Beispiellösung

```typescript
// layout.service.ts
import { Injectable, computed, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';
import { map } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class LayoutService {
  private breakpointObserver = inject(BreakpointObserver);

  private state = toSignal(
    this.breakpointObserver
      .observe([
        Breakpoints.HandsetPortrait,
        Breakpoints.HandsetLandscape,
        Breakpoints.TabletPortrait,
        Breakpoints.TabletLandscape,
        Breakpoints.WebPortrait,
        Breakpoints.WebLandscape,
        '(min-width: 1600px)',
      ])
      .pipe(map(result => result.breakpoints)),
    { initialValue: {} as Record<string, boolean> }
  );

  isMobile = computed(
    () =>
      this.state()[Breakpoints.HandsetPortrait] ||
      this.state()[Breakpoints.HandsetLandscape]
  );

  isTablet = computed(
    () =>
      this.state()[Breakpoints.TabletPortrait] ||
      this.state()[Breakpoints.TabletLandscape]
  );

  isDesktop = computed(
    () =>
      this.state()[Breakpoints.WebPortrait] ||
      this.state()[Breakpoints.WebLandscape]
  );

  isWidescreen = computed(() => this.state()['(min-width: 1600px)']);
}
```

```typescript
// responsive-layout.component.ts
import {
  ChangeDetectionStrategy,
  Component,
  computed,
  inject,
} from '@angular/core';
import { LayoutService } from './layout.service';

@Component({
  selector: 'app-responsive-layout',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div [class]="containerClass()">
      <p>Aktives Layout: <strong>{{ layoutLabel() }}</strong></p>
      <p>Spalten: {{ columns() }}</p>

      @if (isWidescreen()) {
        <p>Widescreen-Modus aktiv!</p>
      }

      @for (item of items; track item) {
        <div class="card">{{ item }}</div>
      }
    </div>
  `,
  styles: [`
    .grid-1 { display: grid; grid-template-columns: 1fr; gap: 8px; }
    .grid-2 { display: grid; grid-template-columns: repeat(2, 1fr); gap: 16px; }
    .grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 24px; }
    .card { padding: 16px; border: 1px solid #ccc; border-radius: 4px; }
  `],
})
export class ResponsiveLayoutComponent {
  private layout = inject(LayoutService);

  protected isMobile = this.layout.isMobile;
  protected isTablet = this.layout.isTablet;
  protected isWidescreen = this.layout.isWidescreen;

  protected columns = computed(() => {
    if (this.layout.isMobile()) return 1;
    if (this.layout.isTablet()) return 2;
    return 3;
  });

  protected containerClass = computed(() => `grid-${this.columns()}`);

  protected layoutLabel = computed(() => {
    if (this.layout.isMobile()) return 'Mobile';
    if (this.layout.isTablet()) return 'Tablet';
    if (this.layout.isWidescreen()) return 'Widescreen';
    return 'Desktop';
  });

  protected items = Array.from({ length: 6 }, (_, i) => `Item ${i + 1}`);
}
```

## Weiterführendes

- Die `BreakpointObserver`-Implementation nutzt intern `MediaMatcher`, der in Server-Umgebungen (SSR/Angular Universal) automatisch `no-op`-Implementierungen verwendet – der Code ist also SSR-sicher ohne weitere Anpassungen.
- Mit `LayoutModule` aus `@angular/cdk/layout` kannst du auch `MediaMatcher` direkt injizieren, wenn du Zugriff auf das native `MediaQueryList`-Objekt benötigst (z. B. für `addEventListener('change', ...)`).
- Kombiniere `BreakpointObserver` mit Angular Material's `MatSidenavContainer` für ein vollständiges responsives Shell-Layout, das die Sidenav automatisch öffnet/schließt.
- Doku: [Angular CDK Layout](https://material.angular.io/cdk/layout/overview)

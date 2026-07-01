# Angular Dojo: Deferrable Views mit @defer
**Datum:** 2026-07-01
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem `@defer`-Block in Angular 17+ Teile deines Templates lazy loaden und deren Rendering gezielt steuern kannst – für bessere Initial-Performance ohne Route-Level Lazy Loading.

## Hintergrund & Theorie

`@defer` ist eine template-native Lösung für Deferrable Views, eingeführt in Angular 17. Damit kannst du beliebige Teile eines Templates (inklusive der zugehörigen Components, Directives und Pipes) in einen eigenen Lazy-Bundle auslagern, der erst dann geladen wird, wenn ein bestimmter Trigger eintrifft.

**Trigger-Optionen:**
- `on idle` – wenn der Browser idle ist (Standard)
- `on viewport` – wenn das Element in den sichtbaren Bereich scrollt
- `on interaction` – bei Klick oder Fokus
- `on hover` – beim Hovern
- `on timer(2s)` – nach Zeitablauf
- `when condition` – wenn ein boolescher Ausdruck wahr wird

**Begleitende Blöcke:**
- `@placeholder` – wird angezeigt, bevor der Trigger ausgelöst wird
- `@loading` – wird während des Downloads angezeigt
- `@error` – wird bei einem Ladefehler angezeigt

Das Besondere: Angular analysiert beim Build automatisch, welche Components/Directives/Pipes ausschließlich in einem `@defer`-Block verwendet werden, und packt diese in ein separates Chunk – vollautomatisch, ohne manuelle `import()` Aufrufe.

## Aufgabe

Du baust eine Dashboard-Seite mit einem schweren Analytics-Widget, das erst geladen werden soll, wenn der Nutzer dorthin scrollt – plus einem Debug-Panel, das nur auf Interaktion hin erscheint.

### Schritte

1. Erstelle eine `HeavyAnalyticsComponent` (`heavy-analytics.component.ts`), die im Constructor eine Verzögerung simuliert und eine Meldung in `ngOnInit` ausgibt. Sie soll ein simples `<div>` mit Chart-Platzhalterdaten rendern.

2. Erstelle eine `DebugPanelComponent` (`debug-panel.component.ts`), die interne State-Infos anzeigt (simulierte Daten reichen).

3. Erstelle eine `DashboardComponent`, die beide Components per `@defer` einbindet:
   - `HeavyAnalyticsComponent`: Trigger `on viewport`, mit `@placeholder` (Skeleton), `@loading` (Spinner-Text) und `@error`.
   - `DebugPanelComponent`: Trigger `on interaction`, mit `@placeholder` (Button-Text „Debug anzeigen").

4. Stelle sicher, dass beide Components **nicht** im `imports`-Array der `DashboardComponent` stehen – Angular erkennt sie automatisch als deferrable.

5. Füge einen manuellen `when`-Trigger hinzu: Ein Signal `showDebug` soll alternativ das Debug-Panel sofort anzeigen können.

### Code-Gerüst

```typescript
// dashboard.component.ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [], // HeavyAnalyticsComponent und DebugPanelComponent kommen NICHT hier rein
  template: `
    <h1>Dashboard</h1>

    <!-- Aufgabe: @defer on viewport für HeavyAnalyticsComponent -->

    <button (click)="showDebug.set(true)">Debug aktivieren</button>

    <!-- Aufgabe: @defer on interaction ODER when showDebug() für DebugPanelComponent -->
  `
})
export class DashboardComponent {
  showDebug = signal(false);
}
```

## Hints

<details>
<summary>Hint 1 – @defer Grundstruktur</summary>

```html
@defer (on viewport) {
  <app-heavy-analytics />
} @placeholder {
  <div class="skeleton">Lade Analytics...</div>
} @loading (minimum 500ms) {
  <span>Wird geladen...</span>
} @error {
  <p>Fehler beim Laden der Analytics.</p>
}
```

`minimum 500ms` beim `@loading`-Block verhindert ein Flackern bei schnellen Verbindungen.
</details>

<details>
<summary>Hint 2 – Kombination von on interaction und when</summary>

Mehrere Trigger werden mit Semikolon getrennt:

```html
@defer (on interaction; when showDebug()) {
  <app-debug-panel />
} @placeholder {
  <button>Debug anzeigen</button>
}
```

Der Placeholder muss ein fokussierbares oder klickbares Element enthalten, damit `on interaction` greift.
</details>

<details>
<summary>Hint 3 – Warum imports leer lassen?</summary>

Angular's Compiler erkennt Components, die **ausschließlich** innerhalb von `@defer`-Blöcken vorkommen, und behandelt sie automatisch als lazy. Trägst du sie zusätzlich in `imports` ein, werden sie im Hauptbundle mitgeliefert – der Defer-Effekt ist damit aufgehoben. Prüfe im Build-Output (`ng build --stats-json`), ob ein separater Chunk erzeugt wurde.
</details>

## Beispiellösung

```typescript
// heavy-analytics.component.ts
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-heavy-analytics',
  standalone: true,
  template: `
    <div class="analytics-widget">
      <h2>Analytics Widget</h2>
      <p>Sessions: {{ data.sessions }}</p>
      <p>Conversions: {{ data.conversions }}</p>
      <div class="chart-placeholder" style="height:200px; background:#e0e0e0; border-radius:8px;"></div>
    </div>
  `
})
export class HeavyAnalyticsComponent implements OnInit {
  data = { sessions: 4821, conversions: 312 };

  ngOnInit() {
    console.log('HeavyAnalyticsComponent geladen');
  }
}
```

```typescript
// debug-panel.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';

@Component({
  selector: 'app-debug-panel',
  standalone: true,
  imports: [JsonPipe],
  template: `
    <div style="background:#fff3cd; padding:1rem; border-radius:8px;">
      <h3>Debug Panel</h3>
      <pre>{{ debugInfo | json }}</pre>
    </div>
  `
})
export class DebugPanelComponent {
  debugInfo = {
    changeDetectionRuns: 42,
    signalReads: 7,
    injectorDepth: 3
  };
}
```

```typescript
// dashboard.component.ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [],
  template: `
    <h1>Dashboard</h1>

    <section style="height: 120vh; padding-top: 2rem;">
      <p>Scroll nach unten, um die Analytics zu laden...</p>
    </section>

    @defer (on viewport) {
      <app-heavy-analytics />
    } @placeholder {
      <div style="height:300px; background:#f5f5f5; display:flex; align-items:center; justify-content:center;">
        Analytics-Bereich (noch nicht geladen)
      </div>
    } @loading (minimum 500ms) {
      <p>Analytics werden geladen...</p>
    } @error {
      <p>Fehler beim Laden der Analytics.</p>
    }

    <hr />

    <button (click)="showDebug.set(true)">Debug per Signal aktivieren</button>

    @defer (on interaction; when showDebug()) {
      <app-debug-panel />
    } @placeholder {
      <button style="padding:0.5rem 1rem; cursor:pointer;">
        Debug-Panel anzeigen (Klick oder Hover)
      </button>
    }
  `
})
export class DashboardComponent {
  showDebug = signal(false);
}
```

## Weiterführendes

- **Prefetching:** `@defer (on viewport; prefetch on idle)` – lade den Bundle vorab im Idle, rendere ihn aber erst beim Viewport-Eintritt. Ideal für kritische Below-the-fold-Inhalte.
- **Testing:** Mit `DeferBlockBehavior.Manual` im TestBed kannst du Defer-Blöcke in Unit-Tests manuell triggern (`fixture.getDeferBlocks()`).
- [Angular Docs: Deferrable Views](https://angular.dev/guide/defer)
- [Angular Blog: Introducing @defer in Angular 17](https://blog.angular.dev)

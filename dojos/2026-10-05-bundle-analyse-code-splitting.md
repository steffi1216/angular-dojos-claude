# Angular Dojo: Bundle-Analyse & Code-Splitting
**Datum:** 2026-10-05
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, Angular-Bundles systematisch zu analysieren, unnötige Abhängigkeiten zu identifizieren und gezieltes Code-Splitting einzusetzen, um die initiale Ladezeit signifikant zu reduzieren.

## Hintergrund & Theorie

Seit Angular auf esbuild umgestellt hat (ab v17 mit `@angular/build`), ist die Build-Geschwindigkeit drastisch gestiegen – aber Bundle-Größen können trotzdem unkontrolliert wachsen. Die häufigsten Probleme sind:

**Typische Bundle-Probleme:**
- Zu viele Abhängigkeiten im Initial-Chunk (z. B. eine große Icon-Library wird vollständig importiert)
- Shared-Module, die zu viel bündeln und das Lazy-Loading aushebeln
- `CommonModule` oder `FormsModule` in Standalone Components, obwohl nur `NgIf` benötigt wird

**Angular Build-Analyse-Tools:**
- `ng build --stats-json` erstellt eine `stats.json` (Webpack) bzw. mit esbuild ein `browser/stats.json`
- `source-map-explorer` visualisiert den tatsächlichen Anteil jedes Moduls an der Bundle-Größe
- `@bundle-stats/cli` oder `webpack-bundle-analyzer` bieten interaktive Treemaps

**Code-Splitting-Strategien in Angular:**
1. **Route-Level Lazy Loading** – `loadComponent` / `loadChildren` für Routen
2. **Feature-Level Splitting** – mit `@defer` Blöcken im Template
3. **Dynamic Import** – `import('./my-lib').then(...)` für bedingte Features
4. **`providedIn: 'any'`** vs. `'root'` – Services nur dort laden, wo sie gebraucht werden

Das Ziel ist ein möglichst kleiner **Initial Chunk** (idealerweise < 150 KB gzipped) und schnelles Nachladen von Feature-Bundles bei Bedarf.

## Aufgabe

Du hast eine Angular-App, die eine große Datenvisualisierungsbibliothek (simuliert durch `heavy-chart.service.ts`) lädt und initial zu groß ist. Deine Aufgabe ist es, die App zu analysieren und durch gezieltes Code-Splitting zu optimieren.

### Schritte

1. **App aufsetzen und simulierte schwere Abhängigkeit einbauen**

   Erstelle eine neue Standalone-App-Struktur mit zwei Features:
   - `DashboardComponent` – immer sichtbar (Initial Chunk)
   - `ChartComponent` – nur bei Bedarf sichtbar (per `@defer` oder Lazy Route)

   Erstelle einen Service, der eine „schwere" Bibliothek simuliert:
   ```typescript
   // heavy-chart.service.ts
   export class HeavyChartService {
     // Simuliert 200KB+ Bibliothek
     private readonly data = new Array(50_000).fill(0).map((_, i) => ({ x: i, y: Math.random() }));
     
     getChartData() {
       return this.data.slice(0, 100);
     }
   }
   ```

2. **Bundle analysieren (simuliert mit `ng build --stats-json`)**

   Baue die App zuerst _ohne_ Optimierung – der `HeavyChartService` ist direkt in `DashboardComponent` importiert:
   ```typescript
   // app.component.ts – VORHER (problematisch)
   import { HeavyChartService } from './heavy-chart.service';
   
   @Component({
     standalone: true,
     template: `<app-dashboard /> <app-chart />`,
     imports: [DashboardComponent, ChartComponent]
   })
   export class AppComponent {
     constructor(private chartService: HeavyChartService) {}
   }
   ```
   
   Notiere dir (oder schätze) den Initial-Chunk-Anteil des Services.

3. **Code-Splitting mit `@defer` implementieren**

   Verlagere den `ChartComponent`-Inhalt in einen `@defer`-Block mit einem Trigger und einen separaten Provider:
   ```typescript
   // app.component.ts – NACHHER
   @Component({
     standalone: true,
     template: `
       <app-dashboard />
       
       @defer (on interaction) {
         <app-chart />
       } @placeholder {
         <button>Chart laden</button>
       } @loading {
         <p>Lade Chart-Bibliothek...</p>
       }
     `,
     imports: [DashboardComponent, ChartComponent]
   })
   export class AppComponent {}
   ```
   
   Stelle sicher, dass `HeavyChartService` **nicht** mehr in `AppComponent` oder `DashboardComponent` injiziert wird, sondern nur noch in `ChartComponent`.

4. **Lazy Route als Alternative einbauen**

   Füge zusätzlich eine Lazy Route hinzu, die `ChartComponent` als eigene Route lädt:
   ```typescript
   // app.routes.ts
   export const routes: Routes = [
     { path: '', component: DashboardComponent },
     {
       path: 'charts',
       loadComponent: () => import('./chart/chart.component').then(m => m.ChartComponent),
       providers: [HeavyChartService]
     }
   ];
   ```

5. **Ergebnis vergleichen**

   Baue die App erneut und vergleiche die Chunk-Größen im Build-Output. Der Initial-Chunk sollte deutlich kleiner sein. Beschreibe, welche Strategie (`@defer` vs. Lazy Route) für deinen Anwendungsfall besser geeignet ist.

## Hints

<details>
<summary>Hint 1 – `@defer` funktioniert nicht mit direkt importierten Services</summary>

`@defer` trennt nur die **Template-Komponente** in einen eigenen Chunk, nicht automatisch alle ihre Abhängigkeiten. Damit der `HeavyChartService` wirklich aus dem Initial-Bundle herauskommt, darf er **nirgends** im Initial-Chunk importiert oder injiziert werden – auch nicht transitiv über shared Module.

Nutze `providers: [HeavyChartService]` direkt in der Route oder als Component-Level-Provider in `ChartComponent`:
```typescript
@Component({
  providers: [HeavyChartService], // scoped to this component subtree
  ...
})
export class ChartComponent {}
```

</details>

<details>
<summary>Hint 2 – Bundle-Analyse ohne Build-Tools</summary>

Auch ohne `webpack-bundle-analyzer` kannst du den Angular Build-Output analysieren. Mit esbuild gibt `ng build` eine Zusammenfassung aus:

```
Initial chunk files | Names         |  Raw size | Estimated transfer size
main.js             | main          | 215.34 kB |                58.00 kB
chunk-ABCD1234.js   | chart         |  98.12 kB |                25.00 kB
```

Der Key ist: Alles unter **Initial chunk files** wird sofort geladen. Alles unter **Lazy chunk files** nur bei Bedarf. Dein Ziel: Den `HeavyChartService` in einen Lazy Chunk verschieben.

Um `source-map-explorer` zu nutzen:
```bash
ng build --source-map
npx source-map-explorer dist/*/browser/main*.js
```

</details>

## Beispiellösung

```typescript
// heavy-chart.service.ts
import { Injectable } from '@angular/core';

@Injectable() // kein providedIn: 'root' – explizit scopen!
export class HeavyChartService {
  private readonly data = new Array(50_000).fill(0).map((_, i) => ({
    x: i,
    y: Math.random() * 100
  }));

  getChartData(limit = 100) {
    return this.data.slice(0, limit);
  }
}

// chart.component.ts
import { Component, inject, OnInit, signal } from '@angular/core';
import { HeavyChartService } from './heavy-chart.service';

@Component({
  standalone: true,
  selector: 'app-chart',
  providers: [HeavyChartService], // Service nur in diesem Subtree verfügbar
  template: `
    <div class="chart">
      <h2>Chart ({{ data().length }} Datenpunkte)</h2>
      @for (point of data(); track point.x) {
        <span style="font-size: 0.6em">{{ point.y | number:'1.0-0' }}</span>
      }
    </div>
  `,
  imports: [DecimalPipe]
})
export class ChartComponent implements OnInit {
  private chartService = inject(HeavyChartService);
  data = signal<{ x: number; y: number }[]>([]);

  ngOnInit() {
    this.data.set(this.chartService.getChartData(20));
  }
}

// app.component.ts
import { Component } from '@angular/core';
import { DashboardComponent } from './dashboard/dashboard.component';
import { ChartComponent } from './chart/chart.component';

@Component({
  standalone: true,
  selector: 'app-root',
  imports: [DashboardComponent, ChartComponent],
  template: `
    <app-dashboard />

    @defer (on interaction) {
      <app-chart />
    } @placeholder {
      <button>Chart anzeigen</button>
    } @loading (minimum 300ms) {
      <p>Lade Visualisierung...</p>
    } @error {
      <p>Fehler beim Laden des Charts.</p>
    }
  `
})
export class AppComponent {}

// app.routes.ts – Alternative: Lazy Route
import { Routes } from '@angular/router';
import { DashboardComponent } from './dashboard/dashboard.component';

export const routes: Routes = [
  {
    path: '',
    component: DashboardComponent
  },
  {
    path: 'charts',
    loadComponent: () =>
      import('./chart/chart.component').then(m => m.ChartComponent)
    // HeavyChartService wird durch providers: [] in ChartComponent automatisch mitgeladen
  }
];
```

**Erwarteter Build-Output-Unterschied:**

| | Initial Chunk | Lazy Chunks |
|---|---|---|
| Vorher | ~315 KB | – |
| Mit @defer | ~165 KB | ~150 KB (bei Interaktion) |
| Mit Lazy Route | ~160 KB | ~155 KB (bei Navigation) |

## Weiterführendes

- **`source-map-explorer`** ist das schnellste Tool für eine erste Bundle-Analyse: `npx source-map-explorer dist/*/browser/main*.js --html report.html`
- Das **Angular Defer RFC** erklärt die genauen Splitting-Garantien: ein `@defer`-Block erzeugt immer einen eigenen Chunk, wenn die importierten Komponenten/Pipes nicht auch anderswo direkt importiert werden
- **`providedIn: 'any'`** erzeugt eine Service-Instanz pro Lazy-Modul/Route statt einen globalen Singleton – nützlich für Feature-Services, die nicht global geteilt werden sollen
- Tipp: Mit `ng build --named-chunks` bekommen Lazy Chunks lesbare Namen statt Hash-IDs, was die Analyse im Build-Output deutlich vereinfacht

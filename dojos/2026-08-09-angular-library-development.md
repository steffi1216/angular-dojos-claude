# Angular Dojo: Angular Library Development mit ng-packagr

**Datum:** 2026-08-09
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du eine wiederverwendbare Angular-Bibliothek erstellst, strukturierst und veröffentlichst – inklusive sekundärer Entry Points, korrekter Peer Dependencies und dem Angular Package Format (APF).

## Hintergrund & Theorie

Angular-Bibliotheken werden mit **ng-packagr** gebaut, das aus deinem TypeScript/Angular-Code das **Angular Package Format (APF)** erzeugt – ein Standardformat mit ESM-Bundles, Type Definitions und einem korrekten `package.json`. Seit Angular 16+ wird standardmäßig nur noch ESM ohne Ivy-spezifisches Partial Compilation Format ausgeliefert.

**Wichtige Konzepte:**

- **Primary Entry Point**: Das Hauptpaket (`@myorg/ui`), definiert durch `ng-package.json` im Bibliotheksordner.
- **Secondary Entry Points**: Eigenständige Import-Pfade innerhalb derselben Bibliothek (`@myorg/ui/button`, `@myorg/ui/dialog`). Jedes benötigt ein eigenes Unterverzeichnis mit `ng-package.json` und `public-api.ts`.
- **Public API (`public-api.ts`)**: Steuert, was nach außen sichtbar ist. Nur was hier re-exportiert wird, ist für Konsumenten nutzbar.
- **Peer Dependencies**: Angular selbst darf nicht in `dependencies`, sondern muss in `peerDependencies` eingetragen sein, damit keine doppelten Angular-Instanzen entstehen.
- **`entryFile`**: Einstiegspunkt der Bibliothek, per Konvention `src/public-api.ts`.

Der Build läuft via `ng build my-lib` und legt die fertigen Artefakte in `dist/my-lib` ab. Mit `npm publish dist/my-lib` oder einem privaten Registry wird die Bibliothek veröffentlicht.

## Aufgabe

Erstelle eine minimale Angular-Bibliothek `@dojo/ui` mit zwei sekundären Entry Points: `@dojo/ui/badge` und `@dojo/ui/spinner`. Beide exportieren je eine Standalone Component. Stelle sicher, dass die Public API korrekt ist und der Bibliotheks-Build fehlerfrei durchläuft.

### Schritte

1. **Workspace vorbereiten** – Erstelle einen neuen Angular-Workspace (oder nutze einen bestehenden) und füge eine Bibliothek hinzu:
   ```bash
   ng new dojo-workspace --no-create-application
   cd dojo-workspace
   ng generate library ui --prefix dojo
   ```

2. **Sekundäre Entry Points anlegen** – Erstelle die Verzeichnisstruktur für `badge` und `spinner` innerhalb von `projects/ui/src/`:
   ```
   projects/ui/src/badge/
     ng-package.json
     public-api.ts
     badge.component.ts
   projects/ui/src/spinner/
     ng-package.json
     public-api.ts
     spinner.component.ts
   ```
   Jede `ng-package.json` für sekundäre Entry Points hat folgenden Inhalt:
   ```json
   {
     "lib": {
       "entryFile": "public-api.ts"
     }
   }
   ```

3. **Komponenten implementieren** – Erstelle `BadgeComponent` und `SpinnerComponent` als Standalone Components mit einem einfachen Template und einem `@Input()`. Exportiere sie jeweils in der lokalen `public-api.ts`.

4. **Primary Public API anpassen** – Entscheide bewusst, was der Primary Entry Point (`projects/ui/src/public-api.ts`) exportiert. Für sekundäre Entry Points ist das typischerweise *nichts* – der Konsument importiert direkt `@dojo/ui/badge`.

5. **`peerDependencies` prüfen** – Öffne `projects/ui/package.json` und stelle sicher, dass `@angular/core` und `@angular/common` als `peerDependencies` eingetragen sind, nicht als `dependencies`.

6. **Bibliothek bauen und prüfen** – Führe den Build aus und inspiziere das Ergebnis:
   ```bash
   ng build ui
   ls dist/ui/
   cat dist/ui/package.json
   ```

## Hints

<details>
<summary>Hint 1 – Struktur einer sekundären Entry Point `ng-package.json`</summary>

Jeder sekundäre Entry Point braucht eine `ng-package.json` **im eigenen Unterverzeichnis** – nicht im `src/`-Root. Das Verzeichnis selbst definiert den Import-Pfad. Beispiel für `projects/ui/src/badge/ng-package.json`:

```json
{
  "lib": {
    "entryFile": "public-api.ts"
  }
}
```

ng-packagr erkennt sekundäre Entry Points automatisch durch diese Datei.
</details>

<details>
<summary>Hint 2 – Standalone Component als Bibliotheks-Export</summary>

Eine minimale `BadgeComponent`:

```typescript
// projects/ui/src/badge/badge.component.ts
import { Component, Input } from '@angular/core';
import { NgClass } from '@angular/common';

@Component({
  selector: 'dojo-badge',
  standalone: true,
  imports: [NgClass],
  template: `
    <span class="badge" [ngClass]="variant">{{ label }}</span>
  `,
  styles: [`
    .badge { padding: 2px 8px; border-radius: 4px; font-size: 0.75rem; }
    .primary { background: #3b82f6; color: white; }
    .danger  { background: #ef4444; color: white; }
  `]
})
export class BadgeComponent {
  @Input() label = '';
  @Input() variant: 'primary' | 'danger' = 'primary';
}
```

`public-api.ts` im selben Verzeichnis:
```typescript
export * from './badge.component';
```
</details>

<details>
<summary>Hint 3 – Häufiger Fehler: zirkuläre Abhängigkeiten zwischen Entry Points</summary>

Sekundäre Entry Points dürfen **nicht** voneinander importieren, wenn dadurch Zyklen entstehen. ng-packagr baut Entry Points in der Reihenfolge ihrer Abhängigkeiten. Wenn `spinner` aus `badge` importiert, muss `badge` zuerst gebaut werden – das klappt, aber `badge` darf dann nicht zurück auf `spinner` zeigen.

Teile gemeinsame Logik am besten in einen eigenen Entry Point `@dojo/ui/common` aus, auf den beide zeigen dürfen.
</details>

## Beispiellösung

```typescript
// projects/ui/src/spinner/spinner.component.ts
import { Component, Input } from '@angular/core';
import { NgStyle } from '@angular/common';

@Component({
  selector: 'dojo-spinner',
  standalone: true,
  imports: [NgStyle],
  template: `
    <div
      class="spinner"
      [ngStyle]="{ width: size + 'px', height: size + 'px' }"
      role="status"
      aria-label="Loading">
    </div>
  `,
  styles: [`
    .spinner {
      border: 3px solid #e5e7eb;
      border-top-color: #3b82f6;
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }
  `]
})
export class SpinnerComponent {
  @Input() size = 32;
}
```

```typescript
// projects/ui/src/spinner/public-api.ts
export * from './spinner.component';
```

```json
// projects/ui/package.json (Auszug)
{
  "name": "@dojo/ui",
  "version": "1.0.0",
  "peerDependencies": {
    "@angular/common": ">=17.0.0",
    "@angular/core": ">=17.0.0"
  }
}
```

Konsumenten nutzen die Bibliothek so:
```typescript
// In einer App-Komponente
import { BadgeComponent } from '@dojo/ui/badge';
import { SpinnerComponent } from '@dojo/ui/spinner';

@Component({
  standalone: true,
  imports: [BadgeComponent, SpinnerComponent],
  template: `
    <dojo-badge label="Neu" variant="primary" />
    <dojo-spinner [size]="24" />
  `
})
export class AppComponent {}
```

## Weiterführendes

- **Angular Package Format (APF)**: Die offizielle Spezifikation unter [angular.io/guide/angular-package-format](https://angular.io/guide/angular-package-format) erklärt die Bundle-Struktur und warum ESM+CJS-Dualität abgelöst wurde.
- **Tipp**: Mit `ng-add` und einem Schematic kannst du deinen Nutzern eine komfortable Installation (`ng add @dojo/ui`) anbieten – kombiniere dieses Dojo mit dem Schematics-Dojo vom 2026-07-16!
- **Treeshaking-Test**: Importiere nur `@dojo/ui/badge` in eine App und prüfe mit `source-map-explorer` oder den Build-Stats, ob `spinner` tatsächlich aus dem Bundle ausgeschlossen wird. Sekundäre Entry Points garantieren das – ein einzelner Primary Export hingegen nicht immer.

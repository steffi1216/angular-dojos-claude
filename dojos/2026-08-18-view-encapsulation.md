# Angular Dojo: ViewEncapsulation – Styles gezielt steuern
**Datum:** 2026-08-18
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst die drei ViewEncapsulation-Modi (`Emulated`, `ShadowDom`, `None`) und kannst bewusst entscheiden, wann welcher Modus sinnvoll ist – inklusive der Auswirkungen auf Style-Scoping, Shadow DOM und das globale Stylesheet.

## Hintergrund & Theorie

Angular kapselt Komponentenstyles standardmäßig mit `ViewEncapsulation.Emulated`. Dabei fügt der Compiler jedem Element ein einzigartiges Attribut (z. B. `_ngcontent-abc-c123`) hinzu und schränkt CSS-Selektoren auf dieses Attribut ein – ohne echtes Shadow DOM.

**Drei Modi:**

| Modus | Shadow DOM | Styles isoliert | Globales CSS wirkt |
|---|---|---|---|
| `Emulated` (Standard) | Nein | Ja (emuliert) | Ja |
| `ShadowDom` | Ja (nativ) | Vollständig | Nein |
| `None` | Nein | Nein | Ja |

**Wichtige Konzepte:**
- `:host` – selektiert das Hostelementt der Komponente
- `:host-context()` – selektiert basierend auf einem Vorfahren-Element
- `::ng-deep` – durchbricht die Kapselung (deprecated, aber noch verbreitet)
- Mit `None` werden Styles global – hilfreich für Theme-Komponenten, gefährlich bei Bibliotheken

Der `ShadowDom`-Modus erzeugt ein echtes Shadow Root im Browser. Globales CSS (z. B. Bootstrap, Material Themes) erreicht die Komponente nicht mehr – das kann erwünscht oder problematisch sein.

## Aufgabe

Baue eine `ThemeCardComponent`, die verschiedene Encapsulation-Modi demonstriert und einen Style-Isolation-Test ermöglicht.

### Schritte

1. **Erstelle drei Komponenten** mit unterschiedlichem `encapsulation`-Modus:
   - `EmulatedCardComponent` mit `ViewEncapsulation.Emulated`
   - `ShadowCardComponent` mit `ViewEncapsulation.ShadowDom`
   - `NoneCardComponent` mit `ViewEncapsulation.None`

2. **Definiere in jeder Komponente** denselben CSS-Selektor:
   ```css
   .card-title {
     color: red; /* wird bei None global! */
   }
   ```
   Und füge im globalen `styles.css` hinzu:
   ```css
   .global-style {
     border: 3px solid blue;
   }
   ```

3. **Füge in jeder Komponente ein Element** mit der Klasse `global-style` ein und beobachte, welcher Modus die globale Klasse aufnimmt und welcher sie ignoriert.

4. **Implementiere `:host` und `:host-context()`** in der `EmulatedCardComponent`:
   ```css
   :host {
     display: block;
     padding: 16px;
   }
   :host-context(.dark-theme) .card-title {
     color: white;
   }
   ```
   Schalte in der App-Komponente eine `.dark-theme`-Klasse am `<body>` um und beobachte den Effekt.

5. **Bonus:** Erstelle eine `SharedStylesComponent` mit `ViewEncapsulation.None`, die nur globale CSS Custom Properties (CSS Variables) setzt – nutze sie als "Theme Provider" ohne eigenes Template-Markup.

## Hints

<details>
<summary>Hint 1 – ViewEncapsulation importieren und setzen</summary>

```typescript
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-shadow-card',
  template: `<div class="card-title">Shadow DOM Card</div>
             <div class="global-style">Globaler Stil?</div>`,
  styles: [`.card-title { color: red; }`],
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class ShadowCardComponent {}
```

Mit `ShadowDom` wird `.global-style` aus `styles.css` **nicht** angewendet, da Shadow DOM das verhindert.

</details>

<details>
<summary>Hint 2 – :host-context() und Dark Theme Toggle</summary>

```typescript
// app.component.ts
export class AppComponent {
  darkMode = false;

  toggleTheme() {
    document.body.classList.toggle('dark-theme', this.darkMode);
  }
}
```

```css
/* emulated-card.component.css */
:host-context(.dark-theme) .card-title {
  color: #e0e0e0;
  background: #1a1a1a;
}
```

`:host-context()` prüft Vorfahren des Host-Elements – ideal für Theme-Switching ohne Input-Binding durch mehrere Ebenen.

</details>

<details>
<summary>Hint 3 – ViewEncapsulation.None als Theme Provider</summary>

```typescript
@Component({
  selector: 'app-theme-provider',
  template: '', // kein sichtbares Markup nötig
  styles: [`
    :root {
      --primary-color: #3f51b5;
      --accent-color: #ff4081;
      --background: #ffffff;
    }
    .dark-theme {
      --primary-color: #7986cb;
      --accent-color: #ff80ab;
      --background: #121212;
    }
  `],
  encapsulation: ViewEncapsulation.None,
})
export class ThemeProviderComponent {}
```

Da `None` die Styles global macht, werden diese CSS Custom Properties überall im Dokument verfügbar.

</details>

## Beispiellösung

```typescript
// emulated-card.component.ts
import { Component, ViewEncapsulation } from '@angular/core';

@Component({
  selector: 'app-emulated-card',
  standalone: true,
  template: `
    <div class="card">
      <h3 class="card-title">Emulated Card</h3>
      <p class="global-style">Globaler Stil aktiv?</p>
    </div>
  `,
  styles: [`
    :host {
      display: block;
      margin: 8px;
    }
    .card {
      padding: 16px;
      border-radius: 8px;
      background: #f5f5f5;
    }
    .card-title {
      color: #e53935;
    }
    :host-context(.dark-theme) .card {
      background: #2c2c2c;
    }
    :host-context(.dark-theme) .card-title {
      color: #ef9a9a;
    }
  `],
  encapsulation: ViewEncapsulation.Emulated, // Standard
})
export class EmulatedCardComponent {}

// shadow-card.component.ts
@Component({
  selector: 'app-shadow-card',
  standalone: true,
  template: `
    <div class="card">
      <h3 class="card-title">Shadow DOM Card</h3>
      <p class="global-style">Globaler Stil aktiv? (Nein!)</p>
    </div>
  `,
  styles: [`
    :host {
      display: block;
      margin: 8px;
    }
    .card { padding: 16px; border-radius: 8px; background: #e8eaf6; }
    .card-title { color: #3949ab; }
  `],
  encapsulation: ViewEncapsulation.ShadowDom,
})
export class ShadowCardComponent {}

// none-card.component.ts – Vorsicht: Styles werden global!
@Component({
  selector: 'app-none-card',
  standalone: true,
  template: `
    <div class="none-card">
      <h3 class="none-card-title">None Encapsulation Card</h3>
      <p class="global-style">Globaler Stil aktiv?</p>
    </div>
  `,
  styles: [`
    /* Präfix nötig, sonst beeinflusst .card-title alle Komponenten! */
    .none-card { padding: 16px; border-radius: 8px; background: #fce4ec; }
    .none-card-title { color: #c62828; }
  `],
  encapsulation: ViewEncapsulation.None,
})
export class NoneCardComponent {}

// app.component.ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [EmulatedCardComponent, ShadowCardComponent, NoneCardComponent],
  template: `
    <div [class.dark-theme]="darkMode">
      <button (click)="darkMode = !darkMode">Dark Mode: {{ darkMode }}</button>
      <app-emulated-card />
      <app-shadow-card />
      <app-none-card />
    </div>
  `,
})
export class AppComponent {
  darkMode = false;
}
```

```css
/* styles.css – global */
.global-style {
  border: 3px solid blue;
  padding: 4px;
}
```

**Beobachtungen:**
- `Emulated`: `.global-style` greift, `:host-context(.dark-theme)` funktioniert
- `ShadowDom`: `.global-style` greift **nicht** (Shadow DOM blockt globales CSS)
- `None`: `.global-style` greift, aber eigene Styles sind ebenfalls global – Namenspräfixe zwingend!

## Weiterführendes

- [Angular Docs: Component Styles](https://angular.dev/guide/components/styling) – offizielle Referenz zu `:host`, `:host-context`, `::ng-deep`
- **Tipp:** In Angular Material werden Overlay-Panels (Dialog, Tooltip) außerhalb der Komponenten-Shadow-Root gerendert – daher verwenden Material-Komponenten `Emulated` und kein `ShadowDom`
- **Tipp:** `::ng-deep` gilt als deprecated; die empfohlene Alternative ist, Styles in `styles.css` zu globalisieren oder CSS Custom Properties als "API" der Komponente zu nutzen
- Für Web Components / Angular Elements ist `ShadowDom` der richtige Modus, da er echte Isolation gegenüber dem Host-Dokument bietet

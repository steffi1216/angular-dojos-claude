# Angular Dojo: Angular Material M3 Theming
**Datum:** 2026-11-09
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem neuen Material Design 3 (M3) Theming-System in Angular Material ein vollständig angepasstes Theme erstellst – inklusive eigener Farbpaletten, Typografie und Dark Mode.

## Hintergrund & Theorie
Angular Material unterstützt seit Version 17 vollständig das Material Design 3 (Codename "Material You"). Das M3-System unterscheidet sich grundlegend von M2: Statt vordefinierter Primär-/Akzentfarben gibt es ein ausgefeiltes **Tonal Color System** mit über 20 semantischen Farbrollen (`primary`, `on-primary`, `surface`, `on-surface-variant` etc.).

Das Theme wird in SCSS mit der `@angular/material` SCSS-API konfiguriert:

```scss
@use '@angular/material' as mat;
```

Drei Kernfunktionen sind zentral:
- `mat.define-theme()` – erstellt ein M3-Theme-Objekt mit Farb- und Typografiekonfiguration
- `mat.all-component-themes()` – appliziert das Theme auf alle Material-Komponenten
- CSS Custom Properties (Tokens) – alle Farbwerte werden als `--mat-*`-Variablen bereitgestellt, was effizientes Theme-Switching (z. B. Dark Mode) ermöglicht

Wichtig: M3-Themes basieren auf `mat.define-theme()` statt dem alten `mat.define-light-theme()`. Die `palettes`-Option nimmt HCT-kompatible Angular-Paletten (via `mat.m3-theme-palette()` oder direkte HEX-Angabe).

## Aufgabe
Erstelle eine Angular-App mit Custom M3 Theme, das sowohl einen Light- als auch einen Dark-Modus unterstützt, und baue eine einfache ThemeToggle-Komponente mit Signals.

### Schritte
1. **SCSS Theme definieren** – Erstelle `src/styles/theme.scss`. Definiere zwei Themes (light & dark) mit einer eigenen Markenfarbe (z. B. Teal `#006874`) als Primary und einem Sekundärton.

2. **Global-Styles verdrahten** – Importiere das Theme in `styles.scss` und appliziere es auf das `:root`-Element. Dark Mode soll über die CSS-Klasse `.dark-theme` am `<body>` aktiviert werden (kein `prefers-color-scheme` – das kommt in Schritt 5).

3. **ThemeService erstellen** – Implementiere einen `ThemeService` mit einem `isDark = signal(false)`. Eine Methode `toggle()` flippt den Zustand und fügt/entfernt die CSS-Klasse `dark-theme` am `document.body`.

4. **ThemeToggle-Komponente** – Baue eine Standalone-Komponente mit einem `MatSlideToggle` (oder `MatIconButton` + `MatIcon`), das den `ThemeService` nutzt. Zeige das aktuelle Icon (`light_mode` / `dark_mode`) als Signal-computed.

5. **Bonus: System-Präferenz** – Ergänze den `ThemeService` um einen `effect()`, der beim Start `window.matchMedia('(prefers-color-scheme: dark)').matches` prüft und das initiale Theme setzt.

## Hints
<details>
<summary>Hint 1 – SCSS Theme-Struktur</summary>

```scss
// src/styles/theme.scss
@use '@angular/material' as mat;

// Light Theme
$light-theme: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$azure-palette,  // oder custom HEX via mat.m3-theme-palette
    tertiary: mat.$blue-palette,
  ),
  typography: (
    brand-family: 'Roboto, sans-serif',
  ),
  density: (scale: 0),
));

// Dark Theme
$dark-theme: mat.define-theme((
  color: (
    theme-type: dark,
    primary: mat.$azure-palette,
    tertiary: mat.$blue-palette,
  ),
));

// In styles.scss:
// :root { @include mat.all-component-themes($light-theme); }
// .dark-theme { @include mat.all-component-color-themes($dark-theme); }
```

Für eine eigene HEX-Farbe als Palette kannst du `mat.m3-theme-palette` nutzen oder einfach die vordefinierten Angular-Material-Paletten (`$azure-palette`, `$spring-green-palette` etc.) verwenden.
</details>

<details>
<summary>Hint 2 – ThemeService mit Signal</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly document = inject(DOCUMENT);
  
  isDark = signal(false);
  
  constructor() {
    // Bonus: System-Präferenz
    effect(() => {
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      this.isDark.set(prefersDark);
    }, { allowSignalWrites: true }); // nur einmalig beim Init
  }

  toggle(): void {
    this.isDark.update(v => !v);
    this.document.body.classList.toggle('dark-theme', this.isDark());
  }
}
```

Alternativ kannst du mit `afterRenderEffect()` oder einem `input`-Signal arbeiten, wenn der Service in einer Komponente liegt.
</details>

<details>
<summary>Hint 3 – ThemeToggle-Komponente</summary>

```typescript
@Component({
  selector: 'app-theme-toggle',
  standalone: true,
  imports: [MatIconButton, MatIcon],
  template: `
    <button mat-icon-button (click)="themeService.toggle()" 
            [attr.aria-label]="label()">
      <mat-icon>{{ icon() }}</mat-icon>
    </button>
  `
})
export class ThemeToggleComponent {
  protected themeService = inject(ThemeService);
  
  protected icon = computed(() => 
    this.themeService.isDark() ? 'light_mode' : 'dark_mode'
  );
  
  protected label = computed(() =>
    this.themeService.isDark() ? 'Hell-Modus aktivieren' : 'Dunkel-Modus aktivieren'
  );
}
```
</details>

## Beispiellösung

```typescript
// theme.service.ts
import { Injectable, signal, effect, inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly document = inject(DOCUMENT);

  isDark = signal(false);

  constructor() {
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    this.isDark.set(prefersDark);
    this.applyTheme(prefersDark);

    effect(() => {
      this.applyTheme(this.isDark());
    });
  }

  toggle(): void {
    this.isDark.update(v => !v);
  }

  private applyTheme(dark: boolean): void {
    this.document.body.classList.toggle('dark-theme', dark);
  }
}
```

```scss
// src/styles/theme.scss
@use '@angular/material' as mat;

$brand-theme: mat.define-theme((
  color: (
    theme-type: light,
    primary: mat.$azure-palette,
    tertiary: mat.$spring-green-palette,
  ),
  typography: (brand-family: 'Roboto, sans-serif'),
  density: (scale: 0),
));

$brand-dark-theme: mat.define-theme((
  color: (
    theme-type: dark,
    primary: mat.$azure-palette,
    tertiary: mat.$spring-green-palette,
  ),
));
```

```scss
// styles.scss
@use './styles/theme' as t;
@use '@angular/material' as mat;

@include mat.core();

:root {
  @include mat.all-component-themes(t.$brand-theme);
}

.dark-theme {
  @include mat.all-component-color-themes(t.$brand-dark-theme);
}
```

```typescript
// theme-toggle.component.ts
import { Component, computed, inject } from '@angular/core';
import { MatIconButton } from '@angular/material/button';
import { MatIcon } from '@angular/material/icon';
import { MatTooltip } from '@angular/material/tooltip';
import { ThemeService } from './theme.service';

@Component({
  selector: 'app-theme-toggle',
  standalone: true,
  imports: [MatIconButton, MatIcon, MatTooltip],
  template: `
    <button mat-icon-button
            [matTooltip]="label()"
            (click)="themeService.toggle()">
      <mat-icon>{{ icon() }}</mat-icon>
    </button>
  `,
})
export class ThemeToggleComponent {
  protected themeService = inject(ThemeService);

  protected icon = computed(() =>
    this.themeService.isDark() ? 'light_mode' : 'dark_mode'
  );

  protected label = computed(() =>
    this.themeService.isDark() ? 'Hell-Modus aktivieren' : 'Dunkel-Modus aktivieren'
  );
}
```

## Weiterführendes
- **Angular Material M3 Migration Guide** – Wenn du von M2 auf M3 migrierst, gibt es ein offizielles Schematic: `ng generate @angular/material:m3-migration`. Es konvertiert `define-light-theme` → `define-theme` und ersetzt veraltete Token-Namen.
- **CSS Custom Properties direkt überschreiben** – M3-Themes exponieren alle Tokens als `--mat-*`-Variablen. Du kannst einzelne Farben komponentenspezifisch überschreiben: `.my-card { --mat-card-container-color: #f0fdf4; }` – ohne SCSS-Mixin.
- Offizielle Doku: [Angular Material Theming Guide](https://material.angular.io/guide/theming)

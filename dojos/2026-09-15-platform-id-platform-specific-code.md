# Angular Dojo: PLATFORM_ID und plattformspezifischer Code
**Datum:** 2026-09-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du in Angular-Applikationen zuverlässig zwischen Browser- und Server-Ausführungskontext unterscheidest – eine essentielle Fähigkeit für SSR-Projekte mit Angular Universal oder dem neuen App Engine.

## Hintergrund & Theorie

Angular kann auf zwei Plattformen laufen: im Browser (CSR) und auf dem Node.js-Server (SSR). Code, der `window`, `document`, `localStorage` oder `navigator` direkt aufruft, wirft auf dem Server einen `ReferenceError`, da diese Browser-APIs dort nicht existieren.

Angular löst das Problem mit zwei Mechanismen:

**1. `PLATFORM_ID` Injection Token**
Ein opaker String, der die aktuelle Plattform identifiziert. Die Hilfsfunktionen `isPlatformBrowser(platformId)` und `isPlatformServer(platformId)` aus `@angular/common` liefern boolsche Werte.

```typescript
import { PLATFORM_ID, inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

private platformId = inject(PLATFORM_ID);
if (isPlatformBrowser(this.platformId)) {
  // sicherer Browser-Code
}
```

**2. `DOCUMENT` Injection Token**
Statt `document` direkt zu referenzieren, injiziert man das Token aus `@angular/common`. Auf dem Server liefert Angular eine DOM-Simulation (z. B. via `domino`), sodass viele DOM-Operationen auch serverseitig funktionieren.

```typescript
import { DOCUMENT } from '@angular/common';
const document = inject(DOCUMENT);
```

Beide Patterns ermöglichen es, Code testbar und plattformunabhängig zu halten, ohne `if (typeof window !== 'undefined')` zu streuen.

## Aufgabe

Erstelle einen `ThemeService`, der das Farbthema der Applikation (`'light'` | `'dark'`) verwaltet. Der Service soll:

- Das Thema in `localStorage` persistieren (nur im Browser)
- Beim Start das gespeicherte Thema laden **oder** die Systempräferenz via `prefers-color-scheme` auslesen (nur im Browser)
- Auf dem Server immer `'light'` als Fallback zurückgeben
- Das aktuelle Thema als Signal exponieren
- Eine Methode `toggle()` anbieten, die das Thema wechselt und `<body>` mit einer CSS-Klasse markiert

### Schritte

1. **Service erstellen** – Lege `src/app/core/theme.service.ts` an. Injiziere `PLATFORM_ID` und `DOCUMENT`.

2. **Initialen Wert ermitteln** – Schreibe eine private Methode `getInitialTheme(): 'light' | 'dark'`, die:
   - Auf dem Server sofort `'light'` zurückgibt
   - Im Browser zuerst `localStorage.getItem('theme')` prüft
   - Als Fallback `window.matchMedia('(prefers-color-scheme: dark)').matches` auswertet

3. **Signal anlegen** – Halte das Thema in einem `WritableSignal<'light' | 'dark'>`.

4. **`toggle()`-Methode** – Wechselt das Signal, speichert den neuen Wert in `localStorage` und setzt/entfernt die Klasse `dark` auf `document.body`. Alle Browser-Zugriffe sind hinter `isPlatformBrowser` geschützt.

5. **Komponente verdrahten** – Binde `themeService.theme()` in einer Demo-Komponente ein und rufe `toggle()` per Button auf. Prüfe im Browser, ob die `dark`-Klasse korrekt gesetzt wird.

6. **(Bonus)** Schreibe einen Unit-Test, der den Service mit einem simulierten Server-`PLATFORM_ID` instanziiert und sicherstellt, dass `theme()` `'light'` zurückgibt, ohne `localStorage` anzufassen.

## Hints

<details>
<summary>Hint 1 – Initialen Wert sicher lesen</summary>

```typescript
private getInitialTheme(): 'light' | 'dark' {
  if (!isPlatformBrowser(this.platformId)) {
    return 'light';
  }
  const stored = localStorage.getItem('theme');
  if (stored === 'light' || stored === 'dark') return stored;
  return window.matchMedia('(prefers-color-scheme: dark)').matches
    ? 'dark'
    : 'light';
}
```

</details>

<details>
<summary>Hint 2 – Body-Klasse über DOCUMENT setzen</summary>

```typescript
toggle(): void {
  const next = this.theme() === 'light' ? 'dark' : 'light';
  this._theme.set(next);

  if (isPlatformBrowser(this.platformId)) {
    localStorage.setItem('theme', next);
    this.document.body.classList.toggle('dark', next === 'dark');
  }
}
```

Beachte: `this.document` ist das via `inject(DOCUMENT)` injizierte Token – kein direkter Zugriff auf die globale Variable.

</details>

<details>
<summary>Hint 3 – Server-Plattform in Tests simulieren</summary>

```typescript
TestBed.configureTestingModule({
  providers: [
    ThemeService,
    { provide: PLATFORM_ID, useValue: 'server' },
  ],
});
const service = TestBed.inject(ThemeService);
expect(service.theme()).toBe('light');
```

`'server'` ist der String-Wert, den Angular intern für die Server-Plattform verwendet. Mit `'browser'` simulierst du den Browser.

</details>

## Beispiellösung

```typescript
// src/app/core/theme.service.ts
import { inject, Injectable, signal } from '@angular/core';
import { DOCUMENT, isPlatformBrowser } from '@angular/common';
import { PLATFORM_ID } from '@angular/core';

export type Theme = 'light' | 'dark';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private platformId = inject(PLATFORM_ID);
  private document = inject(DOCUMENT);

  private _theme = signal<Theme>(this.getInitialTheme());
  readonly theme = this._theme.asReadonly();

  toggle(): void {
    const next: Theme = this._theme() === 'light' ? 'dark' : 'light';
    this._theme.set(next);

    if (isPlatformBrowser(this.platformId)) {
      localStorage.setItem('theme', next);
      this.document.body.classList.toggle('dark', next === 'dark');
    }
  }

  private getInitialTheme(): Theme {
    if (!isPlatformBrowser(this.platformId)) {
      return 'light';
    }

    const stored = localStorage.getItem('theme');
    if (stored === 'light' || stored === 'dark') return stored;

    return window.matchMedia('(prefers-color-scheme: dark)').matches
      ? 'dark'
      : 'light';
  }
}
```

```typescript
// src/app/app.component.ts (Ausschnitt)
import { Component, inject } from '@angular/core';
import { ThemeService } from './core/theme.service';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <p>Aktuelles Thema: {{ themeService.theme() }}</p>
    <button (click)="themeService.toggle()">Thema wechseln</button>
  `,
})
export class AppComponent {
  protected themeService = inject(ThemeService);
}
```

```typescript
// src/app/core/theme.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { PLATFORM_ID } from '@angular/core';
import { ThemeService } from './theme.service';

describe('ThemeService – Server', () => {
  it('gibt "light" zurück ohne localStorage zu berühren', () => {
    const spy = spyOn(Storage.prototype, 'getItem');

    TestBed.configureTestingModule({
      providers: [
        ThemeService,
        { provide: PLATFORM_ID, useValue: 'server' },
      ],
    });

    const service = TestBed.inject(ThemeService);
    expect(service.theme()).toBe('light');
    expect(spy).not.toHaveBeenCalled();
  });
});
```

## Weiterführendes

- **`afterNextRender` statt `isPlatformBrowser`**: Ab Angular 17+ empfiehlt das Team, browser-only DOM-Initialisierungen in `afterNextRender`/`afterRender` zu verschieben – diese Hooks laufen auf dem Server automatisch nicht. Kombiniere beide Ansätze je nach Use Case.
- Angular Docs – [Server-side rendering](https://angular.dev/guide/ssr)
- **`APP_BASE_HREF`**: Weiteres nützliches Plattform-Token für SSR-Routing-Konfigurationen.

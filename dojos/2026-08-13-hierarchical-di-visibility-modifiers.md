# Angular Dojo: Hierarchical DI – @Self, @SkipSelf, @Host, @Optional
**Datum:** 2026-08-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst, wie Angular den Injector-Baum (Element Injector → Environment Injector) durchsucht, und lernst, diese Suche mit `@Self`, `@SkipSelf`, `@Host` und `@Optional` gezielt zu steuern – inklusive der modernen `inject()`-Äquivalente.

## Hintergrund & Theorie
Angular unterhält zwei parallele Injector-Hierarchien:

1. **Element Injector**: Jede Komponente/Direktive besitzt einen eigenen Injector. Er reicht von der Komponente über ihre Vorfahren bis zur Root-Komponente.
2. **Environment Injector**: Modul-/Platform-Ebene (Root, Lazy-Module, Platform).

Wenn ein Token angefordert wird, wandert Angular den Element-Injector nach oben und fällt dann in den Environment-Injector. Mit vier Dekoratoren (und ihren `inject()`-Optionen) kannst du dieses Verhalten übersteuern:

| Dekorator | `inject()` Option | Wirkung |
|---|---|---|
| `@Self` | `{ self: true }` | Nur im **eigenen** Injector suchen – wirf Fehler (oder `null`) wenn nicht gefunden |
| `@SkipSelf` | `{ skipSelf: true }` | **Überspringt** den eigenen Injector, startet bei Eltern-Injector |
| `@Host` | `{ host: true }` | Sucht nur bis zum nächsten **Host-Element** (Komponenten-Grenze) |
| `@Optional` | `{ optional: true }` | Gibt `null` zurück statt eine Exception zu werfen |

Typische Anwendungsfälle:
- **`@SkipSelf`**: Ein Service überschreibt sich selbst und braucht die Eltern-Instanz (z. B. verschachtelter `Logger`).
- **`@Self` + `@Optional`**: Prüfen, ob eine Direktive auf einer Komponente mit einem bestimmten Interface vorhanden ist.
- **`@Host`**: Direktiven, die den Service ihrer Host-Komponente nutzen, aber nicht tiefer suchen sollen.

## Aufgabe
Baue eine verschachtelte Komponentenstruktur mit einem `ThemeService`, der auf verschiedenen Ebenen bereitgestellt werden kann. Demonstriere, wie `@Self`, `@SkipSelf` und `@Optional` unterschiedliche Instanzen liefern.

### Schritte
1. Erstelle einen `ThemeService` mit einer `name`-Property (z. B. `'root'`, `'parent'`, `'child'`). Stelle ihn als `providedIn: 'root'` bereit – das ist die Root-Instanz.
2. Erstelle eine `ParentComponent` (Standalone), die `ThemeService` in ihren `providers` mit `{ provide: ThemeService, useValue: new ThemeService('parent') }` erneut bereitstellt.
3. Erstelle eine `ChildComponent` (Standalone) innerhalb von `ParentComponent`, die `ThemeService` ebenfalls in ihren `providers` als `'child'`-Instanz bereitstellt.
4. Injiziere in `ChildComponent` drei Varianten des Service:
   - `inject(ThemeService, { self: true })` → eigene Instanz (`'child'`)
   - `inject(ThemeService, { skipSelf: true })` → Eltern-Instanz (`'parent'`)
   - `inject(ThemeService)` → eigene Instanz (`'child'`, weil zuerst gefunden)
5. Füge eine `ThemeDirective` hinzu, die `inject(ThemeService, { host: true })` nutzt, und platziere sie auf `ChildComponent`. Beobachte, welche Instanz sie erhält.

## Hints
<details>
<summary>Hint 1 – ThemeService und Mehrfach-Bereitstellung</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  constructor(public name = 'root') {}
}

// In ParentComponent:
providers: [{ provide: ThemeService, useValue: new ThemeService('parent') }]

// In ChildComponent:
providers: [{ provide: ThemeService, useValue: new ThemeService('child') }]
```

Achtung: `useValue` mit `new` funktioniert hier als schnelle Demo. Im echten Projekt lieber `useFactory` mit `inject()`-Aufruf.
</details>

<details>
<summary>Hint 2 – inject() mit Sichtbarkeits-Optionen</summary>

```typescript
export class ChildComponent {
  own    = inject(ThemeService, { self: true });     // 'child'
  parent = inject(ThemeService, { skipSelf: true }); // 'parent'
  any    = inject(ThemeService);                     // 'child' (nächste Instanz)
}
```

`{ self: true }` und `{ optional: true }` lassen sich kombinieren:

```typescript
const maybe = inject(SomeToken, { self: true, optional: true }); // null wenn nicht vorhanden
```
</details>

<details>
<summary>Hint 3 – @Host in einer Direktive</summary>

```typescript
@Directive({ selector: '[appTheme]', standalone: true })
export class ThemeDirective {
  // Sucht ThemeService nur bis zur Host-Komponente (ChildComponent):
  theme = inject(ThemeService, { host: true });

  constructor() {
    console.log('ThemeDirective sieht:', this.theme.name); // 'child'
  }
}
```

Wird die Direktive auf einer Komponente ohne eigenen `ThemeService`-Provider platziert, wirft `{ host: true }` einen Fehler – es wird nicht weiter nach oben gesucht. Verwende `{ host: true, optional: true }` als Absicherung.
</details>

## Beispiellösung

```typescript
// theme.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  constructor(public name = 'root') {}
}
```

```typescript
// theme.directive.ts
import { Directive, inject } from '@angular/core';
import { ThemeService } from './theme.service';

@Directive({ selector: '[appTheme]', standalone: true })
export class ThemeDirective {
  hostTheme = inject(ThemeService, { host: true });
}
```

```typescript
// child.component.ts
import { Component, inject } from '@angular/core';
import { ThemeService } from './theme.service';
import { ThemeDirective } from './theme.directive';

@Component({
  selector: 'app-child',
  standalone: true,
  imports: [ThemeDirective],
  providers: [{ provide: ThemeService, useValue: new ThemeService('child') }],
  template: `
    <div appTheme>
      <p>Self (own):   {{ ownTheme.name }}</p>
      <p>SkipSelf:     {{ parentTheme.name }}</p>
      <p>Default:      {{ defaultTheme.name }}</p>
    </div>
  `,
})
export class ChildComponent {
  ownTheme     = inject(ThemeService, { self: true });
  parentTheme  = inject(ThemeService, { skipSelf: true });
  defaultTheme = inject(ThemeService);
}
```

```typescript
// parent.component.ts
import { Component } from '@angular/core';
import { ThemeService } from './theme.service';
import { ChildComponent } from './child.component';

@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [ChildComponent],
  providers: [{ provide: ThemeService, useValue: new ThemeService('parent') }],
  template: `<app-child />`,
})
export class ParentComponent {}
```

```typescript
// app.component.ts (Root – ThemeService = 'root' via providedIn: 'root')
import { Component } from '@angular/core';
import { ParentComponent } from './parent.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ParentComponent],
  template: `<app-parent />`,
})
export class AppComponent {}

// Erwartete Ausgabe im Browser:
// Self (own):   child
// SkipSelf:     parent
// Default:      child
```

## Weiterführendes
- **Kombination `@Optional` + `@Self`** eignet sich perfekt, um optionale Host-Interfaces zu prüfen – z. B. ob eine Direktive auf einer Komponente sitzt, die `ControlValueAccessor` implementiert.
- **`EnvironmentInjector.runInContext()`**: Wenn du `inject()` außerhalb des Konstruktors aufrufen musst (z. B. in einem Callback), ist `runInInjectionContext(injector, fn)` das richtige Werkzeug – ein Thema für den nächsten Dojo.
- **Offizielle Doku**: [Hierarchical injectors – angular.dev](https://angular.dev/guide/di/hierarchical-dependency-injection) und [DI in action](https://angular.dev/guide/di/di-in-action).

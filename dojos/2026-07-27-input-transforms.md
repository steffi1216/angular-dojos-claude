# Angular Dojo: Input Transforms – Automatische Wertkonvertierung für Signal Inputs
**Datum:** 2026-07-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit der `transform`-Option von `input()` Eingabewerte automatisch konvertierst (z. B. Strings zu Booleans oder Numbers), eigene Transform-Funktionen schreibst und damit robust nutzbare Komponenten-APIs baust.

## Hintergrund & Theorie

Seit Angular 17 unterstützen Signal-basierte Inputs die `transform`-Option. Sie erlaubt es, einen Eingabewert automatisch zu transformieren, bevor er im Signal landet – ähnlich einem Getter/Setter-Paar, aber deklarativ und typsicher.

### Warum Input Transforms?

Wenn eine Komponente als HTML genutzt wird, kommen Attributwerte immer als **Strings** an:

```html
<my-button disabled="true" size="42"></my-button>
```

Ohne Transform muss man manuell konvertieren. Mit Transform übernimmt Angular das automatisch.

### Built-in Transforms

Angular 17 liefert zwei fertige Transform-Funktionen aus `@angular/core`:

| Funktion           | Konvertiert                                  |
|--------------------|----------------------------------------------|
| `booleanAttribute` | `""`, `"true"`, `true` → `true`; sonst false |
| `numberAttribute`  | Strings zu `number`, mit optionalem Fallback  |

```typescript
import { input, booleanAttribute, numberAttribute } from '@angular/core';

@Component({ ... })
export class ButtonComponent {
  disabled = input(false, { transform: booleanAttribute });
  tabIndex = input(0, { transform: numberAttribute });
}
```

### Typinferenz

Der Rückgabetyp des Signals ergibt sich aus dem Rückgabetyp der Transform-Funktion:

```typescript
// Transform: string → number  →  Signal<number>
size = input('16', { transform: (v: string) => parseInt(v, 10) });
```

### Einschränkung

`transform` ist **nur** mit der `input()`-Funktion verfügbar, nicht mit dem `@Input()`-Decorator.

## Aufgabe

Erstelle eine `<app-badge>`-Komponente, die eine Anzahl als Abzeichen anzeigt. Die Komponente soll folgende Inputs haben:

| Input       | Typ (intern) | Beschreibung                                    |
|-------------|-------------|--------------------------------------------------|
| `count`     | `number`    | Anzahl (kommt oft als String aus dem Template)   |
| `maxCount`  | `number`    | Maximaler Anzeigewert, Standard: 99              |
| `visible`   | `boolean`   | Sichtbarkeit, Standard: true                     |
| `label`     | `string`    | Screenreader-Label (trim + uppercase transform)  |

Die Anzeige soll `count` zeigen, aber wenn `count > maxCount`, soll `maxCount + "+"` angezeigt werden (z. B. `"99+"`).

### Schritte

1. Erstelle eine neue Standalone-Komponente `BadgeComponent` mit `ng generate component badge --standalone`.

2. Definiere die vier Signal-Inputs mit passenden `transform`-Optionen:
   - `count` und `maxCount` → `numberAttribute`
   - `visible` → `booleanAttribute`
   - `label` → eigene Transform-Funktion: trimmt den String und wandelt ihn in Großbuchstaben um

3. Erstelle ein `computed()`-Signal `displayValue`, das den anzuzeigenden Text ermittelt:
   - Wenn `count() > maxCount()` → `${maxCount()}+`
   - Sonst → `String(count())`

4. Baue das Template: Zeige das Badge nur, wenn `visible()` true ist (nutze `@if`). Binde `[attr.aria-label]` an `label()`.

5. Teste die Komponente in `AppComponent`, indem du sie mit String-Werten aus dem Template aufrufst:

   ```html
   <app-badge count="150" maxCount="99" visible="true" label="  neue nachrichten  " />
   <app-badge count="5" visible="" label="Aufgaben" />
   <app-badge count="0" [visible]="false" label="Leer" />
   ```

6. Prüfe in der Konsole/DevTools, dass alle Typen korrekt sind und `displayValue()` den richtigen Wert liefert.

## Hints

<details>
<summary>Hint 1 – Transform-Funktion für label</summary>

```typescript
const upperTrim = (value: string): string => value.trim().toUpperCase();

label = input('', { transform: upperTrim });
```

Beachte: Der Eingabetyp der Transform-Funktion muss zum Typ passen, den Angular vom Template übergeben bekommt (meist `string | undefined`). Nutze einen Union-Type falls nötig:

```typescript
const upperTrim = (value: string | undefined): string =>
  (value ?? '').trim().toUpperCase();
```

</details>

<details>
<summary>Hint 2 – computed() für den Anzeigewert</summary>

```typescript
displayValue = computed(() => {
  const c = this.count();
  const max = this.maxCount();
  return c > max ? `${max}+` : String(c);
});
```

`computed()` reagiert automatisch, wenn sich `count` oder `maxCount` ändert.

</details>

<details>
<summary>Hint 3 – numberAttribute mit Fallback</summary>

`numberAttribute` akzeptiert optional einen zweiten Parameter als Fallback-Wert, der genutzt wird, wenn die Konvertierung fehlschlägt (z. B. bei `NaN`):

```typescript
import { numberAttribute } from '@angular/core';

// Fallback: 0
count = input(0, { transform: (v: unknown) => numberAttribute(v, 0) });
```

Alternativ reicht oft `numberAttribute` direkt als Transform:

```typescript
count = input(0, { transform: numberAttribute });
```

</details>

## Beispiellösung

```typescript
// badge.component.ts
import { Component, computed, input } from '@angular/core';
import { booleanAttribute, numberAttribute } from '@angular/core';

const upperTrim = (value: string | undefined): string =>
  (value ?? '').trim().toUpperCase();

@Component({
  selector: 'app-badge',
  standalone: true,
  template: `
    @if (visible()) {
      <span
        class="badge"
        [attr.aria-label]="label()"
      >
        {{ displayValue() }}
      </span>
    }
  `,
  styles: [`
    .badge {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      min-width: 1.5rem;
      padding: 0.15rem 0.4rem;
      border-radius: 9999px;
      background: #e53e3e;
      color: white;
      font-size: 0.75rem;
      font-weight: 700;
    }
  `],
})
export class BadgeComponent {
  count   = input(0,    { transform: numberAttribute });
  maxCount = input(99,  { transform: numberAttribute });
  visible = input(true, { transform: booleanAttribute });
  label   = input('',   { transform: upperTrim });

  displayValue = computed(() => {
    const c   = this.count();
    const max = this.maxCount();
    return c > max ? `${max}+` : String(c);
  });
}
```

```typescript
// app.component.ts (Verwendung)
import { Component } from '@angular/core';
import { BadgeComponent } from './badge/badge.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [BadgeComponent],
  template: `
    <!-- String-Attribute aus dem Template werden automatisch konvertiert -->
    <app-badge count="150" maxCount="99" visible="true" label="  neue nachrichten  " />
    <!-- leeres Attribut wird von booleanAttribute als true interpretiert -->
    <app-badge count="5" visible="" label="Aufgaben" />
    <!-- Property Binding mit boolean false -->
    <app-badge count="0" [visible]="false" label="Leer" />
  `,
})
export class AppComponent {}
```

**Erwartete Ausgabe:**
- Badge 1: `"99+"` (150 > 99), label: `"NEUE NACHRICHTEN"`, sichtbar
- Badge 2: `"5"`, label: `"AUFGABEN"`, sichtbar (leeres Attribut → true)
- Badge 3: nicht sichtbar (`[visible]="false"`)

## Weiterführendes

- **Typsicherheit prüfen**: Mit `input<number>(0, { transform: numberAttribute })` kannst du den erwarteten Typ explizit angeben – TypeScript warnt, wenn Transform-Rückgabe und Typ-Parameter nicht übereinstimmen.
- **`@Input()` + transform**: Der klassische `@Input`-Decorator kennt kein `transform`. Für Migrationen lohnt sich das Umschreiben auf `input()` allein wegen dieser Funktion.
- **Offizielle Doku**: [angular.dev/guide/signals/inputs#input-transforms](https://angular.dev/guide/signals/inputs#input-transforms)
- **Tipp**: Verwende `input.required<number>({ transform: numberAttribute })` für Pflicht-Inputs – dann entfällt der Default-Wert und Angular erzwingt die Angabe im Template.

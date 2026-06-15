# Angular Dojo: Custom Structural Directives
**Datum:** 2026-06-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, eigene Structural Directives zu schreiben, die das DOM dynamisch manipulieren – ähnlich wie `*ngIf` und `*ngFor`, aber vollständig nach deinen eigenen Regeln.

## Hintergrund & Theorie
Structural Directives sind Direktiven, die das DOM-Layout verändern – sie fügen Elemente hinzu, entfernen sie oder ersetzen sie. Das Sternchen-Präfix (`*`) ist syntaktischer Zucker: Angular desugert `*myDir="value"` intern zu `<ng-template [myDir]="value">`.

Die drei wichtigsten Bausteine:
- **`TemplateRef`** – Referenz auf das `<ng-template>`, das du rendern möchtest
- **`ViewContainerRef`** – der Container, in den du Views einfügst oder aus dem du sie entfernst
- **`@Input()`** – um den Ausdruck aus dem Template als Property zu empfangen

Eine minimale Structural Directive sieht so aus:

```typescript
@Directive({ selector: '[appUnless]' })
export class UnlessDirective {
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}
```

Der Setter wird jedes Mal aufgerufen, wenn sich der gebundene Ausdruck ändert. `createEmbeddedView` rendert das Template, `clear()` entfernt alle eingebetteten Views.

Structural Directives können auch einen **typsicheren Kontext** (`$implicit`, benannte Variablen) bereitstellen und so eine `let`-Syntax im Template unterstützen.

## Aufgabe
Implementiere eine Directive `*appRepeat`, die einen Template-Block exakt `n`-mal rendert und dabei den aktuellen Index als Template-Variable (`let i`) bereitstellt.

**Gewünschtes Template-Verhalten:**
```html
<li *appRepeat="3; let i">Item {{ i + 1 }}</li>
```
Ergebnis im DOM:
```
Item 1
Item 2
Item 3
```

### Schritte
1. Erstelle eine neue Datei `repeat.directive.ts` (oder nutze `ng generate directive repeat`).
2. Injiziere `TemplateRef` und `ViewContainerRef` im Constructor.
3. Definiere einen `@Input() set appRepeat(count: number)` Setter.
4. Im Setter: rufe `viewContainer.clear()` auf (wichtig bei Änderungen), dann iteriere `count`-mal und rufe jeweils `viewContainer.createEmbeddedView(this.templateRef, { $implicit: index })` auf.
5. Registriere die Directive in deinem Modul oder als Standalone Directive.
6. Verwende sie im Template mit `*appRepeat="3; let i"`.
7. **Bonus:** Füge einen zweiten Input `appRepeatOf` hinzu, der ein Array akzeptiert, sodass `*appRepeat="items; let item"` über ein Array iteriert (ähnlich `*ngFor`, aber simplifiziert).

## Hints

<details>
<summary>Hint 1 – Kontext-Objekt für `let`-Variablen</summary>

Das zweite Argument von `createEmbeddedView` ist das Kontext-Objekt. `$implicit` ist der Wert, der an `let x` ohne Bindung gebunden wird:

```typescript
this.viewContainer.createEmbeddedView(this.templateRef, {
  $implicit: index,
  count: this.count
});
```

Im Template erreichst du ihn dann mit:
```html
<li *appRepeat="3; let i; let total = count">
  {{ i + 1 }} / {{ total }}
</li>
```

</details>

<details>
<summary>Hint 2 – Typdeklaration mit `ngTemplateContextGuard`</summary>

Damit TypeScript den Typ der Template-Variable kennt, kannst du einen statischen Type Guard hinzufügen:

```typescript
export interface RepeatContext {
  $implicit: number;
  count: number;
}

@Directive({ selector: '[appRepeat]' })
export class RepeatDirective {
  static ngTemplateContextGuard(
    _dir: RepeatDirective,
    ctx: unknown
  ): ctx is RepeatContext {
    return true;
  }
  // ...
}
```

Das sorgt für korrekte IDE-Autovervollständigung und Compile-Zeit-Checks im Template.

</details>

<details>
<summary>Hint 3 – Bonus: `appRepeatOf` als zweiter Input</summary>

Angular unterstützt Mikrosyntax der Form `*appRepeat="items; of items"`. Für einen zweiten Input gilt die Konvention `[Selector][InputSuffix]`:

```typescript
@Input() set appRepeatOf(items: unknown[]) {
  this._items = items;
  this.render();
}
```

Im Template: `*appRepeat="let item of items"` – Angular bindet `items` an `appRepeatOf`.

</details>

## Beispiellösung

```typescript
// repeat.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
} from '@angular/core';

export interface RepeatContext {
  $implicit: number;
  count: number;
}

@Directive({
  selector: '[appRepeat]',
  standalone: true,
})
export class RepeatDirective {
  private _count = 0;

  constructor(
    private templateRef: TemplateRef<RepeatContext>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appRepeat(count: number) {
    this._count = count;
    this.render();
  }

  private render(): void {
    this.viewContainer.clear();
    for (let i = 0; i < this._count; i++) {
      this.viewContainer.createEmbeddedView(this.templateRef, {
        $implicit: i,
        count: this._count,
      });
    }
  }

  // Sorgt für TypeScript-Typinferenz im Template
  static ngTemplateContextGuard(
    _dir: RepeatDirective,
    ctx: unknown
  ): ctx is RepeatContext {
    return true;
  }
}
```

```html
<!-- Verwendung im Template -->
<ul>
  <li *appRepeat="5; let i; let total = count">
    Item {{ i + 1 }} von {{ total }}
  </li>
</ul>
```

```typescript
// app.component.ts (Standalone)
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RepeatDirective],
  template: `
    <ul>
      <li *appRepeat="count; let i">
        Zeile {{ i + 1 }}
      </li>
    </ul>
    <button (click)="count = count + 1">+1</button>
  `,
})
export class AppComponent {
  count = 3;
}
```

### Bonus-Lösung: Array-Iteration mit `appRepeatOf`

```typescript
@Directive({
  selector: '[appRepeat]',
  standalone: true,
})
export class RepeatDirective {
  private _items: unknown[] = [];

  constructor(
    private templateRef: TemplateRef<{ $implicit: unknown; index: number }>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appRepeatOf(items: unknown[]) {
    this._items = items ?? [];
    this.render();
  }

  private render(): void {
    this.viewContainer.clear();
    this._items.forEach((item, index) => {
      this.viewContainer.createEmbeddedView(this.templateRef, {
        $implicit: item,
        index,
      });
    });
  }
}
```

```html
<!-- Array-Verwendung -->
<div *appRepeat="let fruit of fruits; let idx = index">
  {{ idx + 1 }}. {{ fruit }}
</div>
```

## Weiterführendes
- Lies den Angular-Quellcode von [`NgIf`](https://github.com/angular/angular/blob/main/packages/common/src/directives/ng_if.ts) und [`NgFor`](https://github.com/angular/angular/blob/main/packages/common/src/directives/ng_for_of.ts) – sie sind die besten Vorbilder für eigene Structural Directives.
- Mikrosyntax-Referenz in der [Angular-Dokumentation](https://angular.dev/guide/directives/structural-directives#microsyntax) erklärt, wie Angular `*dir="a; let x = b"` intern parst.
- Für Performance-kritische Listen: Schau dir an, wie `NgFor` mit `trackBy` und `EmbeddedViewRef`-Wiederverwendung arbeitet, anstatt jedes Mal `clear()` aufzurufen.

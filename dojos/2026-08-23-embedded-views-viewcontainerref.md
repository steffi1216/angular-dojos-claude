# Angular Dojo: Embedded Views & ViewContainerRef – Dynamische Templates
**Datum:** 2026-08-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `ViewContainerRef` und `TemplateRef` zur Laufzeit dynamische Views aus `<ng-template>`-Blöcken erzeugst, Kontext-Daten übergibst und Views gezielt verwaltest.

## Hintergrund & Theorie

Angular rendert Templates nicht immer direkt ins DOM – manchmal entscheidest du als Entwickler/in zur Laufzeit, *wann* und *wie* ein Template instanziiert wird. Dafür gibt es zwei Schlüsselkonzepte:

**`TemplateRef<C>`** ist ein Verweis auf einen `<ng-template>`-Block im Template. Er enthält keinen gerenderten DOM-Inhalt, sondern nur die Bauanleitung. Das generische `C` beschreibt den Typ des Template-Kontexts (die Daten, die via `let-variable` im Template verfügbar sind).

**`ViewContainerRef`** ist ein Einhängepunkt im DOM, an dem Views eingefügt werden können. Über `createEmbeddedView(templateRef, context?)` wird ein `EmbeddedViewRef` erzeugt und ins DOM eingefügt. Mit `clear()`, `remove()` und `detach()` kannst du Views wieder entfernen oder auslagern.

Dieses Pattern liegt vielen Angular-Built-ins zugrunde: `*ngIf`, `*ngFor`, `@defer`, CDK-Overlays und Structural Directives nutzen alle intern `ViewContainerRef`. Wer eigene wiederverwendbare Direktiven oder Rendering-Strategien baut, kommt an diesem Mechanismus nicht vorbei.

Wichtig: Im Gegensatz zu `ComponentRef` (dynamische Komponenten) erzeugt `createEmbeddedView` keine eigenständige Komponenten-Instanz, sondern einen leichtgewichtigen View aus einem Template.

## Aufgabe

Baue eine wiederverwendbare Direktive `appRepeat`, die ein `<ng-template>` genau N-mal rendert und dabei den Index als Kontext übergibt – ähnlich wie ein minimales `*ngFor`, aber für einfache Wiederholungen.

### Schritte

1. Erstelle eine Standalone-Direktive `RepeatDirective` mit dem Selektor `[appRepeat]`.
2. Injiziere `TemplateRef<{ $implicit: number; index: number }>` und `ViewContainerRef` via `inject()`.
3. Nimm einen Input `appRepeat` (Typ `number`) entgegen – die Anzahl der Wiederholungen.
4. Reagiere auf Änderungen des Inputs (nutze `input()` Signal oder `ngOnChanges`) und rufe eine `render()`-Methode auf.
5. In `render()`: Lösche zuerst alle bestehenden Views (`vcr.clear()`), dann erzeuge in einer Schleife je einen `EmbeddedViewRef` mit `vcr.createEmbeddedView(templateRef, { $implicit: i, index: i })`.
6. Verwende die Direktive in einer Komponente:
   ```html
   <ng-template appRepeat [appRepeat]="5" let-i>
     <div>Item #{{ i }}</div>
   </ng-template>
   ```
7. **Bonus:** Füge einen weiteren Input `appRepeatOf` hinzu, der ein Array entgegennimmt und dessen Elemente als `$implicit` übergibt (wie `*ngFor`).

## Hints

<details>
<summary>Hint 1 – Direktiven-Skelett</summary>

```typescript
@Directive({
  selector: '[appRepeat]',
  standalone: true,
})
export class RepeatDirective {
  private vcr = inject(ViewContainerRef);
  private templateRef = inject(TemplateRef<{ $implicit: number; index: number }>);

  // Input-Signal für die Anzahl
  appRepeat = input<number>(0);

  constructor() {
    effect(() => {
      this.render(this.appRepeat());
    });
  }

  private render(count: number): void {
    this.vcr.clear();
    for (let i = 0; i < count; i++) {
      this.vcr.createEmbeddedView(this.templateRef, { $implicit: i, index: i });
    }
  }
}
```

</details>

<details>
<summary>Hint 2 – Verwendung im Template & Kontext-Variablen</summary>

Im Template greifst du auf den Kontext mit `let-` zu. `let-i` bindet `$implicit`, `let-idx="index"` bindet die benannte Property:

```html
<ng-template appRepeat [appRepeat]="count" let-i let-idx="index">
  <p>Iteration {{ idx + 1 }}: Wert={{ i }}</p>
</ng-template>
```

Für den Bonus mit `appRepeatOf` (Array) kannst du eine zweite Direktive erstellen oder `$implicit` auf das Array-Element setzen:

```typescript
interface RepeatOfContext<T> {
  $implicit: T;
  index: number;
  first: boolean;
  last: boolean;
}
```

</details>

## Beispiellösung

```typescript
// repeat.directive.ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from '@angular/core';

interface RepeatContext {
  $implicit: number;
  index: number;
}

@Directive({
  selector: '[appRepeat]',
  standalone: true,
})
export class RepeatDirective {
  private vcr = inject(ViewContainerRef);
  private templateRef = inject(TemplateRef<RepeatContext>);

  appRepeat = input<number>(0);

  constructor() {
    effect(() => {
      const count = this.appRepeat();
      this.vcr.clear();
      for (let i = 0; i < count; i++) {
        this.vcr.createEmbeddedView(this.templateRef, { $implicit: i, index: i });
      }
    });
  }
}

// app.component.ts
import { Component, signal } from '@angular/core';
import { RepeatDirective } from './repeat.directive';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RepeatDirective],
  template: `
    <label>
      Anzahl:
      <input type="number" [value]="count()" (input)="count.set(+$any($event.target).value)" />
    </label>

    <ul>
      <ng-template appRepeat [appRepeat]="count()" let-i let-idx="index">
        <li>Eintrag {{ idx + 1 }} (i={{ i }})</li>
      </ng-template>
    </ul>
  `,
})
export class AppComponent {
  count = signal(5);
}

// Bonus: Generische repeat-of-Direktive
interface RepeatOfContext<T> {
  $implicit: T;
  index: number;
  first: boolean;
  last: boolean;
  count: number;
}

@Directive({
  selector: '[appRepeatOf]',
  standalone: true,
})
export class RepeatOfDirective<T> {
  private vcr = inject(ViewContainerRef);
  private templateRef = inject(TemplateRef<RepeatOfContext<T>>);

  appRepeatOf = input<T[]>([]);

  constructor() {
    effect(() => {
      const items = this.appRepeatOf();
      this.vcr.clear();
      items.forEach((item, index) => {
        this.vcr.createEmbeddedView(this.templateRef, {
          $implicit: item,
          index,
          first: index === 0,
          last: index === items.length - 1,
          count: items.length,
        });
      });
    });
  }

  // ngTemplateContextGuard für Type Safety
  static ngTemplateContextGuard<T>(
    _dir: RepeatOfDirective<T>,
    ctx: unknown
  ): ctx is RepeatOfContext<T> {
    return true;
  }
}
```

```html
<!-- Verwendung der Bonus-Direktive -->
<ng-template
  [appRepeatOf]="users"
  let-user
  let-idx="index"
  let-isFirst="first"
  let-isLast="last"
>
  <div [class.highlight]="isFirst || isLast">
    {{ idx + 1 }}. {{ user.name }}
  </div>
</ng-template>
```

## Weiterführendes

- **`ngTemplateContextGuard`**: Statische Methode auf der Direktive, die Angular mitteilt, welchen Typ der Template-Kontext hat – aktiviert Template-Type-Checking für `let-` Variablen. Unbedingt bei generischen Direktiven verwenden.
- **`EmbeddedViewRef`**: Der Rückgabewert von `createEmbeddedView()` bietet `detectChanges()`, `markForCheck()`, `detach()` und `destroy()` – nützlich für manuelle Change-Detection-Steuerung in Performance-kritischen Szenarien.
- **`ViewRef` vs. `EmbeddedViewRef` vs. `ComponentRef`**: Angular unterscheidet zwischen Host-Views (aus Komponenten) und Embedded Views (aus Templates). Beide können in einen `ViewContainerRef` eingefügt und zwischen Containern verschoben werden (`move()`).
- Offizielle Docs: [Angular – Dynamische Komponenten & Views](https://angular.dev/guide/components/programmatic-rendering)

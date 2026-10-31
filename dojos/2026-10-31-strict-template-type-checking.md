# Angular Dojo: Strict Template Type Checking & Extended Diagnostics
**Datum:** 2026-10-31
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehe, wie Angular's `strictTemplates`-Modus und die Extended Template Diagnostics Typfehler bereits zur Compilezeit aufdecken und dein Template vollständig typsicher machen.

## Hintergrund & Theorie
Angular bietet seit Version 9 einen strengen Template-Typprüfungsmodus, der durch die `strictTemplates`-Option in der `tsconfig.json` aktiviert wird. Damit prüft der Compiler nicht nur TypeScript-Code, sondern auch Ausdrücke und Bindungen direkt im Template auf Typkorrektheit – ähnlich wie TypeScript's `strict`-Modus für reguläre `.ts`-Dateien.

**`strictTemplates` aktiviert u. a.:**
- Typisierte `@Input()`-Bindungen: Falsche Typen in `[prop]="value"` werden als Fehler gemeldet
- Korrekte Inferenz von Template-Variablen (z. B. `*ngFor`, `@for`, `@let`)
- Strikte Prüfung von Event-Typen in `(event)="handler($event)"`
- Typisierte `ng-template`-Kontexte (inkl. `$implicit`)

**Extended Template Diagnostics** (Angular ≥ 14) gehen noch weiter und liefern Warnungen für häufige Fehler wie:
- `nullishCoalescingNotNullable`: `??` auf einem Wert der nie `null`/`undefined` ist
- `optionalChainNotNullable`: `?.` unnötigerweise auf einem non-nullable Typ
- `textAttributeNotBinding`: vergessenes `[attr]`-Binding (z. B. `min="someVar"` statt `[min]="someVar"`)

Diese Diagnostics werden in `tsconfig.json` unter `angularCompilerOptions.extendedDiagnostics` konfiguriert.

## Aufgabe
Erstelle eine kleine Angular-Applikation mit einer `ProductListComponent`, die `strictTemplates` und Extended Diagnostics nutzt. Behebe dann bewusst eingebaute Typfehler im Template.

### Schritte
1. **Projekt-Setup:** Stelle sicher, dass in `tsconfig.json` `strictTemplates: true` gesetzt ist und konfiguriere Extended Diagnostics so, dass alle Checks auf `"error"` gesetzt sind.
2. **Typen und Component erstellen:** Erstelle ein `Product`-Interface mit `id: number`, `name: string`, `price: number` und `discount?: number`. Erstelle eine `ProductListComponent` mit einem Signal-Input `products = input<Product[]>([])`.
3. **Fehlerhaftes Template bauen:** Schreibe zunächst ein Template mit eingebauten Typfehlern:
   - Binde ein nicht existierendes Property `[foo]="product.nonExistent"`
   - Nutze `product.discount ?? 0` und deklariere `discount` danach als `discount: number` (nicht optional) – löse damit die `nullishCoalescingNotNullable`-Warnung aus
   - Verwende `min="someMinVar"` statt `[min]="someMinVar"`
4. **Fehler beheben:** Korrigiere jeden Fehler und beobachte, wie der Compiler nach jeder Korrektur grün wird.
5. **`$implicit` typisieren:** Erstelle ein `ng-template` mit `ngTemplateOutletContext` und weise dem Kontext einen typisierten `$implicit`-Wert zu. Verwende `let-product` im Template und beobachte, dass `product` korrekt als `Product` inferiert wird.

## Hints
<details>
<summary>Hint 1 – strictTemplates in tsconfig.json</summary>

```json
// tsconfig.json
{
  "compilerOptions": { "strict": true },
  "angularCompilerOptions": {
    "strictTemplates": true,
    "extendedDiagnostics": {
      "defaultCategory": "error",
      "checks": {
        "textAttributeNotBinding": "error",
        "nullishCoalescingNotNullable": "error",
        "optionalChainNotNullable": "error"
      }
    }
  }
}
```
</details>

<details>
<summary>Hint 2 – Typisierter ng-template-Kontext</summary>

```typescript
// Im Component:
interface TemplateContext {
  $implicit: Product;
  index: number;
}

// Im Template:
// <ng-template #productTpl let-p let-i="index">
//   {{ p.name }} – Index: {{ i }}
// </ng-template>
//
// <ng-container *ngTemplateOutlet="productTpl; context: { $implicit: product, index: i }">
// </ng-container>
```

Damit weiß Angular, dass `p` vom Typ `Product` ist. Gibst du einen falschen Typ als Kontext, meldet der Compiler einen Fehler.
</details>

<details>
<summary>Hint 3 – ngTemplateContextGuard für eigene Direktiven</summary>

Wenn du eine eigene Direktive schreibst, die `TemplateRef` einbettet, kannst du den Template-Typ über eine statische Methode sichern:

```typescript
@Directive({ selector: '[appFor]' })
export class AppForDirective<T> {
  @Input() appForOf!: T[];

  static ngTemplateContextGuard<T>(
    dir: AppForDirective<T>,
    ctx: unknown
  ): ctx is { $implicit: T; index: number } {
    return true;
  }
}
```

Angular nutzt diese Methode zur Typinferenz für `let-item` im Template.
</details>

## Beispiellösung

```typescript
// product.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  discount?: number; // optional – ??-Operator ist hier korrekt
}
```

```json
// tsconfig.json (Auszug)
{
  "angularCompilerOptions": {
    "strictTemplates": true,
    "extendedDiagnostics": {
      "defaultCategory": "error",
      "checks": {
        "textAttributeNotBinding": "error",
        "nullishCoalescingNotNullable": "error",
        "optionalChainNotNullable": "error"
      }
    }
  }
}
```

```typescript
// product-list.component.ts
import { Component, input } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';
import { Product } from './product';

interface ProductTemplateContext {
  $implicit: Product;
  index: number;
}

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (product of products(); track product.id; let i = $index) {
        <ng-container
          [ngTemplateOutlet]="productTpl"
          [ngTemplateOutletContext]="{ $implicit: product, index: i }"
        />
      }
    </ul>

    <ng-template #productTpl let-p let-i="index">
      <li>
        {{ i + 1 }}. {{ p.name }} –
        {{ p.price | currency }}
        @if (p.discount != null) {
          <span>({{ p.discount ?? 0 }}% Rabatt)</span>
        }
      </li>
    </ng-template>
  `,
})
export class ProductListComponent {
  products = input<Product[]>([]);
}
```

```typescript
// app.component.ts
import { Component, signal } from '@angular/core';
import { ProductListComponent } from './product-list.component';
import { Product } from './product';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductListComponent],
  template: `
    <app-product-list [products]="products()" />
  `,
})
export class AppComponent {
  products = signal<Product[]>([
    { id: 1, name: 'Laptop', price: 1299, discount: 10 },
    { id: 2, name: 'Maus', price: 29 },
    { id: 3, name: 'Monitor', price: 399, discount: 5 },
  ]);
}
```

**Beispiel für einen Extended-Diagnostic-Fehler, der `nullishCoalescingNotNullable` auslöst:**
```typescript
// Wenn discount als `discount: number` (non-optional) deklariert wäre:
// p.discount ?? 0  →  NG8109: Left side of ?? is never null or undefined.
// Fix: discount?: number  oder  den ??-Ausdruck entfernen
```

## Weiterführendes
- [Angular Docs – Template type checking](https://angular.dev/tools/cli/template-typecheck)
- [Angular Docs – Extended template diagnostics](https://angular.dev/extended-diagnostics)
- Tipp: Aktiviere `"noPropertyAccessFromIndexSignature": true` in `compilerOptions` für noch strengere Typprüfung bei Index-Signaturen in Templates.
- Für Custom Directives: Lerne `ngTemplateContextGuard` und `ngTemplateGuard_*` kennen – sie erlauben Angular, Input-Typen deiner Direktive korrekt in den Template-Kontext zu propagieren.

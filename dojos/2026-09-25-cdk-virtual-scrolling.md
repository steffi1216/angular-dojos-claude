# Angular Dojo: CDK Virtual Scrolling
**Datum:** 2026-09-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK `ScrollingModule` sehr große Listen performant renderst, indem nur die sichtbaren Elemente im DOM existieren. Zusätzlich übst du den Umgang mit variablen Elementhöhen und einem eigenen `VirtualScrollStrategy`.

## Hintergrund & Theorie
Rendert eine App alle Elemente einer langen Liste ins DOM, steigt der Speicherverbrauch und die Renderzeit linear mit der Listenlänge – ab ein paar tausend Einträgen wird die UI spürbar träge. **Virtual Scrolling** löst dieses Problem durch ein *Viewport-Fenster*: Es werden nur die Items gerendert, die aktuell sichtbar sind, plus ein kleiner Puffer darüber und darunter. Alle anderen existieren nur als Datenpunkte.

Das Angular CDK bietet dafür zwei fertige Strategien:

| Strategie | Wann nutzen |
|---|---|
| `FixedSizeVirtualScrollStrategy` | Alle Items gleich hoch – maximale Performance |
| `AutoSizeVirtualScrollStrategy` (experimental) | Variable Höhen – CDK schätzt und korrigiert |

Das Herzstück ist die Direktive `<cdk-virtual-scroll-viewport>` in Kombination mit `*cdkVirtualFor` (statt `*ngFor`). `*cdkVirtualFor` unterstützt dieselben Optionen wie `*ngFor` (`trackBy`, `index`, `even`/`odd` etc.) und hat zusätzlich eine `templateCacheSize`-Option, um DOM-Recycling zu steuern.

Für spezialisierte Anforderungen (z. B. gruppenweise unterschiedliche Höhen) kann man eine eigene `VirtualScrollStrategy` implementieren.

## Aufgabe
Erstelle eine Standalone-Komponente `ProductListComponent`, die eine Liste von 10.000 Produkten mit CDK Virtual Scrolling rendert. Implementiere anschließend eine einfache Filterung per Signal, sodass die angezeigte Liste reaktiv aktualisiert wird, ohne die Performance zu beeinträchtigen.

### Schritte
1. Importiere `ScrollingModule` aus `@angular/cdk/scrolling` in deine Standalone-Komponente.
2. Erstelle ein Signal `products` mit 10.000 generierten Einträgen (`Array.from`). Jedes Produkt hat `id`, `name` und `price`.
3. Füge ein `filter`-Signal (string) und ein computed-Signal `filteredProducts` hinzu, das die Liste reaktiv filtert.
4. Baue das Template mit `<cdk-virtual-scroll-viewport itemSize="56" style="height: 600px">` und `*cdkVirtualFor="let product of filteredProducts()"`.
5. Füge ein `<input>` hinzu, das das `filter`-Signal aktualisiert.
6. Öffne die Browser-DevTools → Performance und vergleiche Rendering mit und ohne Virtual Scrolling.

## Hints
<details>
<summary>Hint 1 – Imports & Setup</summary>

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule, FormsModule],
  ...
})
```
`itemSize` in Pixeln muss der tatsächlichen Höhe eines Items entsprechen, damit die Scrollbalken-Position stimmt.
</details>

<details>
<summary>Hint 2 – Signals & Computed</summary>

```typescript
readonly filter = signal('');

readonly products = signal(
  Array.from({ length: 10_000 }, (_, i) => ({
    id: i + 1,
    name: `Produkt ${i + 1}`,
    price: +(Math.random() * 500).toFixed(2),
  }))
);

readonly filteredProducts = computed(() => {
  const q = this.filter().toLowerCase();
  return q
    ? this.products().filter(p => p.name.toLowerCase().includes(q))
    : this.products();
});
```

`computed` wird nur neu berechnet, wenn sich `filter` oder `products` ändern – kein unnötiges Re-Rendering.
</details>

<details>
<summary>Hint 3 – Template-Grundstruktur</summary>

```html
<input [value]="filter()" (input)="filter.set($any($event.target).value)"
       placeholder="Filtern…" />

<cdk-virtual-scroll-viewport itemSize="56" style="height: 600px; overflow-y: auto;">
  <div *cdkVirtualFor="let product of filteredProducts(); trackBy: trackById"
       style="height: 56px; display: flex; align-items: center; padding: 0 16px;">
    {{ product.id }} – {{ product.name }} ({{ product.price | currency }})
  </div>
</cdk-virtual-scroll-viewport>
```

`trackBy` ist auch bei Virtual Scrolling wichtig, um DOM-Recycling korrekt zu steuern.
</details>

## Beispiellösung

```typescript
import { Component, computed, signal } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { ScrollingModule } from '@angular/cdk/scrolling';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [ScrollingModule, CurrencyPipe],
  template: `
    <input
      [value]="filter()"
      (input)="filter.set($any($event.target).value)"
      placeholder="Produkt filtern…"
      style="margin-bottom: 8px; width: 300px; padding: 6px;"
    />
    <p>{{ filteredProducts().length }} / {{ products().length }} Produkte</p>

    <cdk-virtual-scroll-viewport itemSize="56" style="height: 600px; border: 1px solid #ccc;">
      <div
        *cdkVirtualFor="let product of filteredProducts(); trackBy: trackById"
        style="height: 56px; display: flex; align-items: center;
               padding: 0 16px; border-bottom: 1px solid #eee;"
      >
        <strong style="min-width: 60px;">#{{ product.id }}</strong>
        {{ product.name }}
        <span style="margin-left: auto;">{{ product.price | currency:'EUR' }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
})
export class ProductListComponent {
  readonly filter = signal('');

  readonly products = signal<Product[]>(
    Array.from({ length: 10_000 }, (_, i) => ({
      id: i + 1,
      name: `Produkt ${i + 1}`,
      price: +(Math.random() * 500).toFixed(2),
    }))
  );

  readonly filteredProducts = computed(() => {
    const q = this.filter().toLowerCase().trim();
    return q
      ? this.products().filter(p => p.name.toLowerCase().includes(q))
      : this.products();
  });

  trackById(_: number, product: Product): number {
    return product.id;
  }
}
```

**Bonus:** Ersetze `*cdkVirtualFor` durch `*cdkVirtualFor="...; bufferSize: 5"` und beobachte im DOM-Inspector, wie viele Elemente tatsächlich gerendert werden.

## Weiterführendes
- **Variable Höhen** mit `AutoSizeVirtualScrollStrategy` aus `@angular/cdk-experimental/scrolling` (experimentell, aber nützlich für Cards mit dynamischem Inhalt).
- **Eigene `VirtualScrollStrategy`**: Interface `VirtualScrollStrategy` implementieren und per `provide` als `VIRTUAL_SCROLL_STRATEGY`-Token bereitstellen – ideal für gruppierte Listen mit Sektions-Headern.
- Offizielle Docs: [Angular CDK – Scrolling](https://material.angular.io/cdk/scrolling/overview)
- Performance-Tipp: Kombiniere Virtual Scrolling mit `OnPush`-Change-Detection und Signals, um Re-Renders auf ein absolutes Minimum zu reduzieren.

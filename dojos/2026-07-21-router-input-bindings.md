# Angular Dojo: Router Input Bindings (withComponentInputBinding)
**Datum:** 2026-07-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `withComponentInputBinding` Route-Parameter, Query-Parameter und statische Route-Daten direkt als `@Input()` oder Signal-Input (`input()`) in Komponenten bindest – ganz ohne manuelle `ActivatedRoute`-Injection.

## Hintergrund & Theorie

Seit Angular 16 bietet der Router die Funktion `withComponentInputBinding()` als Teil der `provideRouter()`-Konfiguration. Damit werden Route-Parameter (`params`), Query-Parameter (`queryParams`) und statische `data`-Objekte automatisch als `@Input()`-Properties an die routable Komponente gebunden.

**Vorher (klassisch):**
```typescript
constructor(private route: ActivatedRoute) {
  this.route.params.pipe(map(p => p['id'])).subscribe(...)
}
```

**Nachher (modern):**
```typescript
@Input() id!: string; // wird automatisch aus :id gebunden
```

Das funktioniert auch mit Signal-Inputs (`input()` aus `@angular/core`):
```typescript
readonly id = input<string>('');
```

Wenn derselbe Key in `params`, `queryParams` und `data` vorkommt, gilt folgende Priorität: `params` > `queryParams` > `data`.

Zusätzlich können Route-Resolver-Daten ebenfalls direkt gebunden werden: Der Resolver-Name wird zum Input-Namen.

**Wichtig:** Die Feature-Funktion muss einmal in `provideRouter()` registriert werden:
```typescript
provideRouter(routes, withComponentInputBinding())
```

Dieser Ansatz reduziert Boilerplate erheblich, verbessert die Testbarkeit (Inputs lassen sich direkt setzen) und harmoniert ideal mit dem Signal-Reaktivitätsmodell.

## Aufgabe

Baue eine einfache Produktdetail-Ansicht mit drei routable Komponenten:
- Eine **Produktliste** mit Links zu einzelnen Produkten
- Eine **Produktdetail-Komponente**, die `id` (Route-Param), `ref` (Query-Param) und einen Resolver-Wert als Inputs empfängt
- Einen **Resolver**, der zu einer `id` die Produktdaten lädt

### Schritte

1. **`app.config.ts` / `main.ts` anpassen**: Aktiviere `withComponentInputBinding()` in `provideRouter()`.

2. **Routen definieren**: Lege eine Route `/products/:id` an. Füge statische `data: { section: 'shop' }` hinzu und registriere einen Resolver `productResolver`.

3. **Resolver erstellen**: Schreibe einen funktionalen Resolver (`ResolveFn<Product>`), der anhand der `id` ein simuliertes Produkt zurückgibt (z. B. per `of({...}).pipe(delay(0))`).

4. **`ProductDetailComponent` bauen**: Die Komponente soll folgende Inputs deklarieren:
   - `id` – aus dem Route-Param `:id`
   - `ref` – aus dem Query-Param `?ref=`
   - `section` – aus dem statischen Route-`data`-Objekt
   - `productResolver` – aus dem Resolver-Ergebnis
   
   Zeige alle Werte im Template an.

5. **Signal-Input-Variante**: Ersetze mindestens ein `@Input()` durch `input<string>()` und zeige, dass der Wert reaktiv verfügbar ist (z. B. mit `computed()`).

6. **Testen**: Navigiere zu `/products/42?ref=newsletter` und prüfe, ob alle Werte korrekt im Template erscheinen.

## Hints

<details>
<summary>Hint 1 – withComponentInputBinding aktivieren</summary>

```typescript
// app.config.ts
import { provideRouter, withComponentInputBinding } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

</details>

<details>
<summary>Hint 2 – Funktionaler Resolver</summary>

```typescript
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';
import { of } from 'rxjs';

export interface Product {
  id: string;
  name: string;
}

export const productResolver: ResolveFn<Product> = (route) => {
  const id = route.paramMap.get('id') ?? '';
  return of({ id, name: `Produkt ${id}` });
};
```

</details>

<details>
<summary>Hint 3 – Inputs in der Detailkomponente</summary>

```typescript
@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <p>ID: {{ id }}</p>
    <p>Ref: {{ ref }}</p>
    <p>Section: {{ section }}</p>
    <p>Name: {{ productResolver?.name }}</p>
    <p>Computed: {{ upperName() }}</p>
  `,
})
export class ProductDetailComponent {
  // aus Route-Param :id
  @Input() id!: string;

  // aus Query-Param ?ref=
  @Input() ref: string = '';

  // aus data: { section: '...' }
  @Input() section: string = '';

  // aus Resolver (Input-Name = Resolver-Key in Route)
  @Input() productResolver?: Product;

  // Signal-Input Variante (alternativ zu @Input)
  readonly upperId = input<string>('');
  readonly upperName = computed(() => this.upperId().toUpperCase());
}
```

**Route-Definition:**
```typescript
{
  path: 'products/:id',
  component: ProductDetailComponent,
  data: { section: 'shop' },
  resolve: { productResolver: productResolver },
}
```

</details>

## Beispiellösung

```typescript
// routes.ts
import { Routes } from '@angular/router';
import { ProductDetailComponent } from './product-detail.component';
import { productResolver } from './product.resolver';

export const routes: Routes = [
  {
    path: 'products/:id',
    component: ProductDetailComponent,
    data: { section: 'shop' },
    resolve: { productResolver },
  },
];

// product.resolver.ts
import { ResolveFn } from '@angular/router';
import { of } from 'rxjs';

export interface Product { id: string; name: string; }

export const productResolver: ResolveFn<Product> = (route) => {
  const id = route.paramMap.get('id') ?? 'unknown';
  return of({ id, name: `Produkt ${id}` });
};

// product-detail.component.ts
import { Component, Input, computed, input } from '@angular/core';
import { Product } from './product.resolver';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <h2>Produktdetail</h2>
    <ul>
      <li>ID (Param): <strong>{{ id }}</strong></li>
      <li>Ref (Query-Param): <strong>{{ ref || '–' }}</strong></li>
      <li>Section (data): <strong>{{ section }}</strong></li>
      <li>Resolver-Name: <strong>{{ productResolver?.name }}</strong></li>
      <li>Signal-ID uppercase: <strong>{{ upperCaseId() }}</strong></li>
    </ul>
  `,
})
export class ProductDetailComponent {
  @Input() id: string = '';
  @Input() ref: string = '';
  @Input() section: string = '';
  @Input() productResolver?: Product;

  // Signal-Input: wird ebenfalls automatisch aus dem :id-Param befüllt
  readonly signalId = input<string>('');
  readonly upperCaseId = computed(() => this.signalId().toUpperCase());
}

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

**Navigation zum Testen:**
```typescript
// In einer anderen Komponente
router.navigate(['/products', '42'], { queryParams: { ref: 'newsletter' } });
// → id='42', ref='newsletter', section='shop', productResolver={id:'42', name:'Produkt 42'}
```

## Weiterführendes

- **Testbarkeit**: Da alle Werte jetzt `@Input()` sind, kannst du `ProductDetailComponent` im Test einfach instanziieren und Inputs direkt setzen – kein `ActivatedRoute`-Mock mehr nötig.
- Lies die offizielle Doku zu [Router Input Bindings](https://angular.dev/guide/routing/router-tutorial-toh) und suche nach `withComponentInputBinding`.
- Kombiniere Router Inputs mit `linkedSignal()` (Dojo 2026-07-13), um abgeleitete Zustände reaktiv zu verwalten.
- Für komplexe Szenarien: Wenn ein Input-Name in `params` und `queryParams` kollidiert, gewinnt `params` – dokumentiere diese Priorität in deinem Team.

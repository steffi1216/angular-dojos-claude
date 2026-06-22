# Angular Dojo: Standalone Components & APIs
**Datum:** 2026-06-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Standalone Components funktionieren und wie du eine vollständige Feature-Einheit ohne NgModule aufbaust – inklusive Routing, Services und Lazy Loading mit der modernen Angular-API.

## Hintergrund & Theorie

Seit Angular 14 (stabil ab Angular 15) können Komponenten, Direktiven und Pipes ohne ein umgebendes `NgModule` deklariert werden. Mit `standalone: true` importieren sie ihre Abhängigkeiten direkt im `@Component`-Dekorator.

**Vorteile:**
- Weniger Boilerplate – kein `NgModule` mehr nötig
- Bessere Tree-Shakability durch explizite Imports
- Einfacheres Lazy Loading einzelner Komponenten (statt ganzer Module)
- Klarer Überblick über Abhängigkeiten direkt in der Komponente

**Wichtige APIs:**
- `bootstrapApplication(AppComponent, appConfig)` ersetzt `platformBrowserDynamic().bootstrapModule(AppModule)`
- `provideRouter(routes)` ersetzt `RouterModule.forRoot(routes)`
- `provideHttpClient()` ersetzt `HttpClientModule`
- `importProvidersFrom()` für Bibliotheken, die noch NgModules nutzen
- `Routes` mit `loadComponent` für Lazy Loading einzelner Komponenten

```typescript
// main.ts – neuer Einstiegspunkt
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
    provideHttpClient(withInterceptors([authInterceptor])),
  ]
});
```

Die gesamte Anwendung kann heute „NgModule-free" gebaut werden – das ist der empfohlene Weg für neue Angular-Projekte.

## Aufgabe

Baue eine kleine **Produkt-Übersicht** als vollständig standalone Feature. Die App soll:
1. Eine `ProductListComponent` (standalone) anzeigen, die Produkte von einem Service lädt
2. Eine `ProductCardComponent` (standalone) als wiederverwendbare Unterkomponente nutzen
3. Eine `HighlightDirective` (standalone) direkt in der Card importieren
4. Per `loadComponent` lazy geladen werden

### Schritte

1. **Erstelle `product.model.ts`** mit einem Interface `Product { id, name, price, featured }`.

2. **Erstelle `product.service.ts`** als Injectable-Service (ohne `providedIn` oder mit `providedIn: 'root'`), der eine Liste von Mock-Produkten als `signal<Product[]>` bereitstellt.

3. **Erstelle `highlight.directive.ts`** als standalone Direktive:
   - Selektor: `[appHighlight]`
   - Setzt `background-color: gold` per `HostBinding`, wenn ein Input `featured` true ist

4. **Erstelle `product-card.component.ts`** als standalone Component:
   - Importiert `HighlightDirective` direkt im `imports`-Array
   - Erhält ein `@Input() product: Product`
   - Zeigt Name und Preis an, nutzt `[appHighlight]="product.featured"` auf dem Host-Element

5. **Erstelle `product-list.component.ts`** als standalone Component:
   - Importiert `ProductCardComponent` und `CommonModule` (oder `NgFor`/`NgIf` direkt)
   - Injiziert `ProductService` per `inject()`
   - Zeigt alle Produkte als Liste von `app-product-card` an

6. **Konfiguriere das Routing** in `app.routes.ts` so, dass `/products` die `ProductListComponent` per `loadComponent` lädt:
   ```typescript
   {
     path: 'products',
     loadComponent: () =>
       import('./products/product-list.component')
         .then(m => m.ProductListComponent)
   }
   ```

7. **Passe `main.ts`** an, um `bootstrapApplication` mit `provideRouter(routes)` zu nutzen.

## Hints

<details>
<summary>Hint 1 – Standalone Direktive deklarieren</summary>

```typescript
@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @Input() appHighlight = false;

  @HostBinding('style.background-color')
  get bgColor() {
    return this.appHighlight ? 'gold' : 'transparent';
  }
}
```

Der Selektor und der Input haben denselben Namen – das erlaubt `[appHighlight]="true"` als kombiniertes Attribut + Binding.
</details>

<details>
<summary>Hint 2 – Standalone Component mit direktem Import</summary>

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [HighlightDirective],  // direkt importieren, kein Modul nötig
  template: `
    <div [appHighlight]="product.featured" class="card">
      <h3>{{ product.name }}</h3>
      <p>{{ product.price | currency:'EUR' }}</p>
    </div>
  `,
})
export class ProductCardComponent {
  @Input({ required: true }) product!: Product;
}
```

`CurrencyPipe` ist standalone seit Angular 15 – direkt in `imports` aufnehmen: `imports: [HighlightDirective, CurrencyPipe]`.
</details>

<details>
<summary>Hint 3 – inject() und Signals im Service</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {
  private products = signal<Product[]>([
    { id: 1, name: 'Angular Buch', price: 39.99, featured: true },
    { id: 2, name: 'RxJS Kurs', price: 79.00, featured: false },
    { id: 3, name: 'NgRx Workshop', price: 199.00, featured: true },
  ]);

  getProducts = this.products.asReadonly();
}

// In der Komponente:
export class ProductListComponent {
  private productService = inject(ProductService);
  products = this.productService.getProducts;
}
```
</details>

## Beispiellösung

```typescript
// product.model.ts
export interface Product {
  id: number;
  name: string;
  price: number;
  featured: boolean;
}

// product.service.ts
import { Injectable, signal } from '@angular/core';
import { Product } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private _products = signal<Product[]>([
    { id: 1, name: 'Angular Buch', price: 39.99, featured: true },
    { id: 2, name: 'RxJS Kurs', price: 79.0, featured: false },
    { id: 3, name: 'NgRx Workshop', price: 199.0, featured: true },
    { id: 4, name: 'TypeScript Deep Dive', price: 49.99, featured: false },
  ]);

  readonly products = this._products.asReadonly();
}

// highlight.directive.ts
import { Directive, HostBinding, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @Input() appHighlight = false;

  @HostBinding('style.background-color')
  get bgColor(): string {
    return this.appHighlight ? '#ffd70033' : 'transparent';
  }

  @HostBinding('style.border-left')
  get borderLeft(): string {
    return this.appHighlight ? '4px solid gold' : 'none';
  }
}

// product-card.component.ts
import { Component, Input } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { Product } from '../product.model';
import { HighlightDirective } from '../highlight.directive';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [HighlightDirective, CurrencyPipe],
  template: `
    <div [appHighlight]="product.featured" class="product-card">
      <span *ngIf="product.featured" class="badge">⭐ Featured</span>
      <h3>{{ product.name }}</h3>
      <p class="price">{{ product.price | currency:'EUR':'symbol':'1.2-2':'de' }}</p>
    </div>
  `,
  styles: [`
    .product-card {
      padding: 1rem;
      margin: 0.5rem 0;
      border: 1px solid #ddd;
      border-radius: 8px;
      transition: background-color 0.3s;
    }
    .badge { font-size: 0.8rem; color: goldenrod; }
    .price { font-weight: bold; color: #333; }
  `],
})
export class ProductCardComponent {
  @Input({ required: true }) product!: Product;
}

// product-list.component.ts
import { Component, inject } from '@angular/core';
import { NgFor } from '@angular/common';
import { ProductService } from '../product.service';
import { ProductCardComponent } from './product-card.component';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [NgFor, ProductCardComponent],
  template: `
    <h2>Produkte ({{ products().length }})</h2>
    <app-product-card
      *ngFor="let product of products(); trackBy: trackById"
      [product]="product"
    />
  `,
})
export class ProductListComponent {
  private productService = inject(ProductService);
  products = this.productService.products;

  trackById(_: number, product: { id: number }) {
    return product.id;
  }
}

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'products', pathMatch: 'full' },
  {
    path: 'products',
    loadComponent: () =>
      import('./products/product-list.component').then(
        (m) => m.ProductListComponent
      ),
  },
];

// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
  ],
}).catch(console.error);
```

## Weiterführendes

- **`@NgModule`-freie Migration:** Das Angular-Team stellt einen [Migrations-Schematic](https://angular.dev/reference/migrations/standalone) bereit (`ng generate @angular/core:standalone`), der bestehende Module automatisch in Standalone-Komponenten umwandelt.
- **Functional Guards & Resolvers:** Kombiniere Standalone-Routing mit funktionalen Guards: `canActivate: [() => inject(AuthService).isLoggedIn()]` – kein Klassen-Guard mehr nötig.
- **`withComponentInputBinding()`:** Mit `provideRouter(routes, withComponentInputBinding())` werden Router-Parameter automatisch als `@Input()` in die Komponente gemappt – kein `ActivatedRoute` mehr nötig.
- Offizielle Doku: https://angular.dev/guide/components/importing

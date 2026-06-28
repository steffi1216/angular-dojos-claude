# Angular Dojo: Angular Universal & Server-Side Rendering (SSR)
**Datum:** 2026-06-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehe, wie Angular Universal SSR funktioniert, und implementiere eine SSR-fähige Anwendung mit Transfer State, um doppelte HTTP-Requests zwischen Server und Client zu vermeiden.

## Hintergrund & Theorie
Angular Universal ermöglicht es, Angular-Anwendungen auf dem Server zu rendern (Server-Side Rendering, SSR). Das HTML wird dabei auf dem Server generiert und direkt an den Browser geliefert – noch bevor JavaScript ausgeführt wird. Das verbessert:

- **SEO**: Suchmaschinen-Crawler sehen sofort vollständig gerendertes HTML.
- **First Contentful Paint (FCP)**: Nutzer sehen Inhalte früher.
- **Core Web Vitals**: Bessere LCP- und FID-Werte.

Seit Angular 17 ist SSR direkt in die Angular CLI integriert (`@angular/ssr` ersetzt `@nguniversal/*`). Mit dem neuen App Engine werden `app.server.ts`, `server.ts` und `main.server.ts` automatisch generiert.

Ein kritisches Problem bei SSR ist **Double Fetching**: Der Server führt HTTP-Requests aus, serialisiert das Ergebnis in den HTML-Transfer-State (`TransferState`), und der Client liest diesen State, bevor er eigene Requests feuert – so wird der gleiche Request nicht zweimal ausgeführt.

Wichtig: Plattform-spezifischer Code (z.B. `window`, `document`, `localStorage`) darf nur im Browser laufen. Dafür gibt es `isPlatformBrowser()` und `PLATFORM_ID`.

## Aufgabe
Erstelle eine SSR-fähige Angular-Anwendung, die eine Liste von Produkten von einer API lädt. Nutze `TransferState`, um Double Fetching zu verhindern, und schütze browser-spezifischen Code mit `isPlatformBrowser`.

### Schritte

1. **Neues Projekt mit SSR anlegen**
   ```bash
   ng new ssr-dojo --ssr
   cd ssr-dojo
   ```
   Alternativ zu einem bestehenden Projekt SSR hinzufügen:
   ```bash
   ng add @angular/ssr
   ```

2. **Product-Service mit TransferState implementieren**

   Erstelle `src/app/products.service.ts`. Der Service soll:
   - Beim Server-Render die Daten von `https://fakestoreapi.com/products?limit=5` laden.
   - Das Ergebnis in den `TransferState` schreiben (mit einem `makeStateKey`).
   - Auf dem Client zuerst den `TransferState` prüfen und – falls vorhanden – die Daten direkt verwenden, ohne einen HTTP-Request zu senden.
   - Den State-Eintrag nach dem Lesen entfernen (`remove`).

3. **ProductListComponent erstellen**

   Erstelle `src/app/product-list/product-list.component.ts` als Standalone Component:
   - Injiziere den `ProductsService`.
   - Lade die Produkte im `ngOnInit`.
   - Zeige eine Ladeliste mit `*ngFor` (oder dem neuen `@for`) an.
   - Gib `title`, `price` und `id` jedes Produkts aus.

4. **Browser-Guard einbauen**

   Füge im Service oder in der Komponente Code ein, der `localStorage` nur im Browser-Kontext aufruft:
   ```typescript
   if (isPlatformBrowser(this.platformId)) {
     localStorage.setItem('last-loaded', new Date().toISOString());
   }
   ```

5. **SSR-Build und lokaler Test**
   ```bash
   npm run build
   npm run serve:ssr:ssr-dojo
   ```
   Öffne `http://localhost:4000` und prüfe im Browser den Page Source: Das HTML soll bereits die Produktliste enthalten (kein leeres `<app-root>`).

6. **Double Fetching verifizieren**

   Öffne die Browser DevTools → Network-Tab. Lade die Seite neu. Es sollte **kein** zweiter Request an `fakestoreapi.com` erscheinen – der Client liest aus dem Transfer State.

## Hints

<details>
<summary>Hint 1 – TransferState-Key und grundlegende Struktur</summary>

```typescript
import { makeStateKey, TransferState } from '@angular/core';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

// Im Service:
const cached = this.transferState.get(PRODUCTS_KEY, null);
if (cached) {
  this.transferState.remove(PRODUCTS_KEY);
  return of(cached);
}
return this.http.get<Product[]>(url).pipe(
  tap(products => this.transferState.set(PRODUCTS_KEY, products))
);
```

</details>

<details>
<summary>Hint 2 – PLATFORM_ID und isPlatformBrowser</summary>

```typescript
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

// Im Service oder Component:
private platformId = inject(PLATFORM_ID);

someMethod() {
  if (isPlatformBrowser(this.platformId)) {
    // Nur im Browser ausführen
    console.log(window.location.href);
  }
}
```

</details>

<details>
<summary>Hint 3 – server.ts anpassen (falls nötig)</summary>

Angular 17+ generiert automatisch eine `server.ts`. Falls du eigene Express-Middleware brauchst (z.B. für API-Proxying), kannst du sie dort eintragen:

```typescript
// server.ts
server.get('/api/**', (req, res) => {
  // Proxy-Logik
});
```

</details>

## Beispiellösung

```typescript
// src/app/product.model.ts
export interface Product {
  id: number;
  title: string;
  price: number;
}
```

```typescript
// src/app/products.service.ts
import { inject, Injectable, makeStateKey, PLATFORM_ID, TransferState } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { isPlatformBrowser } from '@angular/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { Product } from './product.model';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  getProducts(): Observable<Product[]> {
    const cached = this.transferState.get(PRODUCTS_KEY, null);
    if (cached) {
      this.transferState.remove(PRODUCTS_KEY);
      return of(cached);
    }

    return this.http
      .get<Product[]>('https://fakestoreapi.com/products?limit=5')
      .pipe(
        tap(products => {
          if (!isPlatformBrowser(this.platformId)) {
            this.transferState.set(PRODUCTS_KEY, products);
          }
        })
      );
  }

  trackLastLoaded(): void {
    if (isPlatformBrowser(this.platformId)) {
      localStorage.setItem('last-loaded', new Date().toISOString());
    }
  }
}
```

```typescript
// src/app/product-list/product-list.component.ts
import { Component, inject, OnInit, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductsService } from '../products.service';
import { Product } from '../product.model';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Produkte</h2>
    @if (loading()) {
      <p>Lade Produkte…</p>
    }
    @for (product of products(); track product.id) {
      <div class="product-card">
        <strong>{{ product.title }}</strong>
        <span>{{ product.price | currency:'EUR' }}</span>
      </div>
    }
  `,
  styles: [`
    .product-card {
      display: flex;
      justify-content: space-between;
      padding: 0.5rem;
      border-bottom: 1px solid #eee;
    }
  `]
})
export class ProductListComponent implements OnInit {
  private productsService = inject(ProductsService);

  products = signal<Product[]>([]);
  loading = signal(true);

  ngOnInit(): void {
    this.productsService.getProducts().subscribe({
      next: products => {
        this.products.set(products);
        this.loading.set(false);
        this.productsService.trackLastLoaded();
      },
      error: () => this.loading.set(false)
    });
  }
}
```

```typescript
// src/app/app.component.ts
import { Component } from '@angular/core';
import { ProductListComponent } from './product-list/product-list.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ProductListComponent],
  template: `<app-product-list />`
})
export class AppComponent {}
```

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    provideClientHydration(),  // Aktiviert Transfer State + Hydration
    provideHttpClient(withFetch()),  // SSR benötigt fetch statt XHR
  ]
};
```

## Weiterführendes
- **Incremental Hydration** (Angular 17+): Mit `@defer` und `withIncrementalHydration()` können Komponenten-Teile verzögert hydriert werden – ideal für Below-the-Fold-Inhalte.
- **Route-Level Rendering**: Angular 19 erlaubt per Route `renderMode: RenderMode.Server | RenderMode.Client | RenderMode.Prerender` in `app.routes.server.ts`.
- Offizielle Docs: [angular.dev/guide/ssr](https://angular.dev/guide/ssr)
- Für Prerendering (Static Site Generation) kann `ng build` mit `prerender: true` in `angular.json` konfiguriert werden.

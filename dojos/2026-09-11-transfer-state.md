# Angular Dojo: Transfer State
**Datum:** 2026-09-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular's `TransferState` API doppelte HTTP-Anfragen beim Wechsel von Server-Side Rendering (SSR) zu Client-Side Rendering verhindert und wie du eigene State-Transfers implementierst.

## Hintergrund & Theorie

Bei Angular Universal / SSR rendert der Server die Seite, führt dabei HTTP-Anfragen aus und schickt das fertige HTML an den Client. Ohne Transfer State wiederholt Angular im Browser dieselben HTTP-Anfragen kurz nach der Hydration — der User sieht kurz valide Daten, dann einen Lade-Spinner, dann wieder die Daten.

**`TransferState`** löst dieses Problem: Der Server serialisiert den abgerufenen Zustand als JSON in das gerenderte HTML (als eingebettetes `<script>`-Tag). Der Client liest diesen State beim Start aus, statt neue Requests abzufeuern.

**Wichtige APIs:**
- `makeStateKey<T>(key: string): StateKey<T>` — typsicherer Key für den State-Transfer
- `transferState.set(key, value)` — State auf dem Server speichern
- `transferState.get(key, defaultValue)` — State auf dem Client auslesen
- `transferState.hasKey(key)` — prüfen, ob State vorhanden ist
- `transferState.remove(key)` — State nach Verwendung bereinigen

Ab Angular 17+ ist `TransferState` direkt via DI verfügbar, ohne separates Modul.

Ein `HttpTransferCacheInterceptor` ist seit Angular 17 auch built-in verfügbar (`withHttpTransferCache()` im `provideHttpClient()`), der HTTP GET-Requests automatisch cached.

## Aufgabe

Implementiere einen `ProductService`, der `TransferState` nutzt, um Produktdaten aus einem SSR-Request an den Client zu übertragen. Erstelle außerdem eine manuelle Implementierung mit einem Guard, der `TransferState` für Custom State (kein HTTP) nutzt.

### Schritte

1. Richte das Projekt mit SSR-Support ein und konfiguriere `provideHttpClient(withHttpTransferCache())` in `app.config.ts` für automatisches HTTP-Caching.

2. Erstelle einen `ProductService` mit einer `getProducts()`-Methode, die manuell `TransferState` nutzt: Auf dem Server werden die Daten geladen und per `transferState.set()` gespeichert; auf dem Client werden sie per `transferState.get()` ausgelesen (Fallback: HTTP-Request).

3. Erstelle eine `ProductListComponent` (standalone), die den Service nutzt, und logge in der Konsole, ob die Daten vom Transfer State oder vom HTTP-Request kommen.

4. Füge einen Custom-State-Transfer für nicht-HTTP-Daten hinzu (z. B. Server-Timestamp oder Feature-Flags), die du im `APP_INITIALIZER` auf dem Server setzt und auf dem Client ausließt.

## Hints

<details>
<summary>Hint 1 – TransferState Setup und Injektion</summary>

`TransferState` ist direkt injizierbar — kein separates Modul nötig:

```typescript
import { TransferState, makeStateKey } from '@angular/core';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductService {
  private transferState = inject(TransferState);
  private http = inject(HttpClient);
  private platformId = inject(PLATFORM_ID);
}
```

Mit `isPlatformServer(this.platformId)` und `isPlatformBrowser(this.platformId)` kannst du zwischen Server und Client unterscheiden.
</details>

<details>
<summary>Hint 2 – Server/Client-Logik im Service</summary>

```typescript
getProducts(): Observable<Product[]> {
  if (this.transferState.hasKey(PRODUCTS_KEY)) {
    const products = this.transferState.get(PRODUCTS_KEY, []);
    this.transferState.remove(PRODUCTS_KEY); // Bereinigung nach Verwendung
    console.log('Daten aus Transfer State geladen');
    return of(products);
  }

  return this.http.get<Product[]>('/api/products').pipe(
    tap(products => {
      if (isPlatformServer(this.platformId)) {
        this.transferState.set(PRODUCTS_KEY, products);
      }
    })
  );
}
```

Das `remove()` nach dem Auslesen verhindert, dass alter State im Memory verbleibt.
</details>

## Beispiellösung

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withHttpTransferCache } from '@angular/common/http';
import { provideClientHydration } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    // withHttpTransferCache cached alle GET-Requests automatisch
    provideHttpClient(withHttpTransferCache()),
    provideClientHydration(),
  ],
};

// models/product.model.ts
export interface Product {
  id: number;
  name: string;
  price: number;
}

// services/product.service.ts
import { Injectable, inject, PLATFORM_ID } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { TransferState, makeStateKey } from '@angular/core';
import { isPlatformServer, isPlatformBrowser } from '@angular/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { Product } from '../models/product.model';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');
const SERVER_TIMESTAMP_KEY = makeStateKey<number>('serverTimestamp');

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  getProducts(): Observable<Product[]> {
    if (isPlatformBrowser(this.platformId) && this.transferState.hasKey(PRODUCTS_KEY)) {
      const products = this.transferState.get(PRODUCTS_KEY, []);
      this.transferState.remove(PRODUCTS_KEY);
      console.log('[Client] Produkte aus TransferState geladen — kein HTTP-Request!');
      return of(products);
    }

    console.log(
      isPlatformServer(this.platformId)
        ? '[Server] Produkte via HTTP laden und in TransferState speichern...'
        : '[Client] Kein TransferState gefunden — HTTP-Request wird ausgeführt.'
    );

    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(PRODUCTS_KEY, products);
        }
      })
    );
  }

  getServerTimestamp(): number | null {
    if (this.transferState.hasKey(SERVER_TIMESTAMP_KEY)) {
      const ts = this.transferState.get(SERVER_TIMESTAMP_KEY, 0);
      this.transferState.remove(SERVER_TIMESTAMP_KEY);
      return ts;
    }
    return null;
  }

  setServerTimestamp(): void {
    if (isPlatformServer(this.platformId)) {
      this.transferState.set(SERVER_TIMESTAMP_KEY, Date.now());
    }
  }
}

// components/product-list.component.ts
import { Component, OnInit, inject, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductService } from '../services/product.service';
import { Product } from '../models/product.model';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Produkte</h2>
    @if (serverTimestamp()) {
      <p>Server-Timestamp: {{ serverTimestamp() | date:'medium' }}</p>
    }
    @for (product of products(); track product.id) {
      <div>{{ product.name }} — {{ product.price | currency }}</div>
    } @empty {
      <p>Lädt...</p>
    }
  `,
})
export class ProductListComponent implements OnInit {
  private productService = inject(ProductService);

  products = signal<Product[]>([]);
  serverTimestamp = signal<number | null>(null);

  ngOnInit() {
    this.serverTimestamp.set(this.productService.getServerTimestamp());
    this.productService.getProducts().subscribe(p => this.products.set(p));
  }
}

// app.config.server.ts — APP_INITIALIZER für Custom State
import { ApplicationConfig, APP_INITIALIZER } from '@angular/core';
import { mergeApplicationConfig } from '@angular/core';
import { appConfig } from './app.config';
import { provideServerRendering } from '@angular/platform-server';
import { ProductService } from './services/product.service';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    {
      provide: APP_INITIALIZER,
      useFactory: (productService: ProductService) => () => {
        productService.setServerTimestamp();
      },
      deps: [ProductService],
      multi: true,
    },
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

## Weiterführendes

- Angular Docs: [Server-side rendering](https://angular.dev/guide/ssr) — offizielle Dokumentation zu SSR und Hydration
- Mit `withHttpTransferCache()` (Angular 17+) werden GET-Requests automatisch gecacht — manuelles `TransferState` ist nur noch für Non-HTTP-Daten (Feature Flags, Config, Timestamps) nötig
- Prüfe mit den Chrome DevTools → Network tab, dass beim Client-Startup keine doppelten API-Calls ausgeführt werden
- Für komplexere Szenarien: `provideClientHydration(withIncrementalHydration())` kombiniert Transfer State mit progressiver Hydration

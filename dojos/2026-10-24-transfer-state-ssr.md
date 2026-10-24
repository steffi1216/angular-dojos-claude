# Angular Dojo: TransferState – Server-zu-Client Datentransfer bei SSR

**Datum:** 2026-10-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du mit `TransferState` HTTP-Daten vom Server-Rendering-Prozess auf den Client überträgst und so unnötige doppelte HTTP-Anfragen nach der Hydration vermeidest.

## Hintergrund & Theorie

Bei Angular Universal bzw. SSR wird eine Seite zunächst auf dem Server gerendert und als HTML an den Browser geschickt. Sobald Angular im Browser bootstrappt (Hydration), werden Komponenten wieder initialisiert – und alle `ngOnInit`-Aufrufe mit HTTP-Requests laufen erneut. Der Nutzer sieht kurzzeitig Lade-Indikatoren, obwohl der Server die Daten bereits abgerufen hat.

`TransferState` löst dieses Problem: Ein Key-Value-Store wird auf dem Server befüllt und als JSON inline in die HTML-Response eingebettet (`<script type="application/json" id="ng-state">…</script>`). Im Browser liest Angular diesen Store, bevor ein echter HTTP-Request abgesetzt wird.

Seit Angular 14+ gibt es `withHttpTransferCache()` in `provideClientHydration()` als automatische Lösung für Standard-`HttpClient`-Calls. Für benutzerdefinierte Datenquellen (z.B. Datenbank-Direktzugriffe im Server-Code, komplexe Transformationen oder Drittanbieter-SDKs) bleibt das manuelle `TransferState`-API unverzichtbar.

**Kernkonzepte:**
- `makeStateKey<T>(token: string)` – typisierter Key
- `TransferState.set(key, value)` – auf dem Server befüllen
- `TransferState.get(key, defaultValue)` – auf dem Client auslesen
- `TransferState.hasKey(key)` – prüfen ob Daten vorhanden
- `TransferState.remove(key)` – nach Verbrauch bereinigen
- `isPlatformServer()` / `isPlatformBrowser()` – Plattformcheck

## Aufgabe

Erstelle einen `ProductService`, der `TransferState` nutzt, um Produktdaten vom Server auf den Client zu übertragen. Die Komponente soll beim Server-Rendering die Daten laden und im Browser direkt aus dem Transfer-Store lesen – ohne erneuten HTTP-Request.

### Schritte

1. **Setup**: Erstelle einen neuen Service `ProductTransferService` mit Injektion von `TransferState`, `HttpClient`, `PLATFORM_ID` und `isPlatformServer`.

2. **State Key definieren**: Definiere einen typisierten `StateKey` für ein Array von Produkten:
   ```typescript
   interface Product { id: number; name: string; price: number; }
   const PRODUCTS_KEY = makeStateKey<Product[]>('products');
   ```

3. **Service-Methode implementieren**: Erstelle eine `getProducts()`-Methode als Observable, die:
   - Auf dem **Server**: HTTP-Request ausführt, Ergebnis via `TransferState.set()` speichert und das Observable zurückgibt
   - Im **Browser**: Prüft ob `hasKey(PRODUCTS_KEY)` → gibt sofort `of(state.getAndRemove(key, []))` zurück; sonst macht sie einen neuen HTTP-Request

4. **Komponente verdrahten**: Erstelle eine `ProductListComponent` die den Service im `constructor`/`inject()` einbindet und `getProducts()` in `ngOnInit` oder als Signal via `toSignal()` nutzt.

5. **Verifikation**: Öffne den gerenderten HTML-Quelltext und prüfe, ob der `ng-state`-Block mit den Produktdaten vorhanden ist. Beobachte in den Browser-DevTools, dass kein zweiter HTTP-Request zu `/api/products` abgesetzt wird.

## Hints

<details>
<summary>Hint 1 – Service-Grundstruktur</summary>

```typescript
import { inject, Injectable, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { makeStateKey, TransferState } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';

export interface Product { id: number; name: string; price: number; }
const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductTransferService {
  private http = inject(HttpClient);
  private state = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  getProducts(): Observable<Product[]> {
    if (isPlatformServer(this.platformId)) {
      return this.http.get<Product[]>('/api/products').pipe(
        tap(products => this.state.set(PRODUCTS_KEY, products))
      );
    }
    // Browser-Pfad folgt in Hint 2
    return this.http.get<Product[]>('/api/products');
  }
}
```

</details>

<details>
<summary>Hint 2 – Browser-Pfad mit TransferState</summary>

```typescript
getProducts(): Observable<Product[]> {
  if (isPlatformServer(this.platformId)) {
    return this.http.get<Product[]>('/api/products').pipe(
      tap(products => this.state.set(PRODUCTS_KEY, products))
    );
  }

  // Im Browser: gecachte Daten verwenden wenn vorhanden
  if (this.state.hasKey(PRODUCTS_KEY)) {
    const cached = this.state.getAndRemove(PRODUCTS_KEY, []);
    return of(cached);
  }

  // Kein Cache vorhanden (z.B. Client-seitige Navigation)
  return this.http.get<Product[]>('/api/products');
}
```

`getAndRemove()` liest und löscht den Key in einem Schritt, damit Folgecalls einen echten Request machen.

</details>

<details>
<summary>Hint 3 – Komponente mit Signal-Integration</summary>

```typescript
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ProductTransferService } from './product-transfer.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    @if (products()) {
      <ul>
        @for (p of products(); track p.id) {
          <li>{{ p.name }} – {{ p.price | currency }}</li>
        }
      </ul>
    } @else {
      <p>Laden…</p>
    }
  `,
})
export class ProductListComponent {
  private service = inject(ProductTransferService);
  products = toSignal(this.service.getProducts());
}
```

</details>

## Beispiellösung

```typescript
// product-transfer.service.ts
import { inject, Injectable, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { makeStateKey, TransferState } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';

export interface Product {
  id: number;
  name: string;
  price: number;
}

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductTransferService {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  getProducts(): Observable<Product[]> {
    if (isPlatformServer(this.platformId)) {
      return this.http.get<Product[]>('/api/products').pipe(
        tap(products => this.transferState.set(PRODUCTS_KEY, products))
      );
    }

    if (this.transferState.hasKey(PRODUCTS_KEY)) {
      const products = this.transferState.getAndRemove(PRODUCTS_KEY, []);
      return of(products);
    }

    return this.http.get<Product[]>('/api/products');
  }
}

// product-list.component.ts
import { Component, inject } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { toSignal } from '@angular/core/rxjs-interop';
import { ProductTransferService } from './product-transfer.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CurrencyPipe],
  template: `
    @if (products(); as list) {
      <ul>
        @for (product of list; track product.id) {
          <li>{{ product.name }} – {{ product.price | currency:'EUR' }}</li>
        }
      </ul>
    } @else {
      <p>Produkte werden geladen…</p>
    }
  `,
})
export class ProductListComponent {
  private service = inject(ProductTransferService);
  readonly products = toSignal(this.service.getProducts());
}

// app.config.ts (Server)
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration, withHttpTransferCache } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { provideRouter } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    provideHttpClient(withFetch()),
    // withHttpTransferCache() für automatisches Caching von HttpClient-Calls
    provideClientHydration(withHttpTransferCache()),
  ],
};
```

## Weiterführendes

- **Automatisches HTTP-Caching**: `provideClientHydration(withHttpTransferCache())` übernimmt das Caching für Standard-`HttpClient`-Calls automatisch – `TransferState` manuell verwenden nur für custom Data Sources oder wenn du volle Kontrolle brauchst.
- **Sicherheitshinweis**: Sensible Daten (Auth-Tokens, PII) niemals in `TransferState` speichern – der Inhalt wird unverschlüsselt als inline-JSON in die HTML-Seite geschrieben und ist für jeden Nutzer sichtbar.
- Offizielle Docs: [Angular Server-side rendering](https://angular.dev/guide/ssr) und [TransferState API](https://angular.dev/api/core/TransferState)
- Für komplexe SSR-Szenarien mit NgRx: `@ngrx/router-store` in Kombination mit `TransferState` ermöglicht auch Store-State-Übertragung vom Server zum Client.

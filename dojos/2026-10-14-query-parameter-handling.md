# Angular Dojo: Query Parameter Handling mit dem Router
**Datum:** 2026-10-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Query Parameter sicher liest, setzt und reaktiv beobachtest – inklusive `queryParamsHandling`, typsicherer Konvertierung und dem Signal-basierten Zugriff über `toSignal`.

## Hintergrund & Theorie
Query Parameter (z. B. `?page=2&filter=active`) sind ein zentrales Werkzeug für teilbare, bookmarkbare UI-Zustände. Angular stellt dafür zwei Wege bereit:

**Reaktiv (Observable):** `ActivatedRoute.queryParams` oder `queryParamMap` liefern einen Stream, der bei jeder Änderung emittiert – auch wenn die Komponente nicht neu erzeugt wird (z. B. bei Component-Reuse durch den Router).

**Snapshot:** `ActivatedRoute.snapshot.queryParamMap` gibt den Zustand zum Aktivierungszeitpunkt zurück. Achtung: Er aktualisiert sich **nicht** automatisch bei Folgenavigationen auf derselben Route.

Für den Schreibzugriff nutzt man `Router.navigate()` mit dem `queryParams`-Objekt. Dabei steuert `queryParamsHandling`:
- `'merge'` – erhält alle bestehenden Query Params und überschreibt nur die genannten
- `'preserve'` – behält alle bestehenden Params unverändert
- `''` (Standard) – ersetzt alle bestehenden Params durch die neuen

Seit Angular 16+ lassen sich Query Params dank `withComponentInputBinding()` im Router auch direkt als `@Input()` binden. In modernen Standalone-Apps empfiehlt sich außerdem `toSignal(route.queryParamMap)` für eine nahtlose Integration ins Signals-Ökosystem.

## Aufgabe
Baue eine **Produktlisten-Komponente** mit serverseitiger Paginierung und einer Suchleiste, deren Zustand vollständig in den Query Params lebt. Die URL soll immer den aktuellen Zustand abbilden, sodass sie geteilt oder bookmarkt werden kann.

### Schritte
1. Erstelle eine Standalone-Komponente `ProductListComponent`. Injiziere `ActivatedRoute` und `Router`.
2. Lies `page` (default `1`) und `search` (default `''`) reaktiv aus `route.queryParamMap` – wandle `page` in eine Zahl um. Nutze `toSignal()` für die Signal-Integration.
3. Implementiere `setPage(page: number)` und `setSearch(term: string)`, die per `Router.navigate()` mit `queryParamsHandling: 'merge'` die URL aktualisieren.
4. Leite die gefilterten / paginierten Einträge als `computed()`-Signal aus den Query-Param-Signalen ab (Mock-Daten genügen).
5. Stelle sicher, dass beim Setzen eines neuen Suchbegriffs die Seite auf `1` zurückgesetzt wird – ohne das in jedem Handler einzeln zu codieren.
6. **Bonus:** Nutze `withComponentInputBinding()` im Router-Setup und binde `page` und `search` zusätzlich als `@Input()` – vergleiche beide Zugangswege.

## Hints
<details>
<summary>Hint 1 – Query Params reaktiv lesen</summary>

```typescript
// queryParamMap emittiert auch bei gleichbleibender Komponente
const queryParamMap = toSignal(this.route.queryParamMap);

const page = computed(() => Number(queryParamMap()?.get('page') ?? '1'));
const search = computed(() => queryParamMap()?.get('search') ?? '');
```

`toSignal` benötigt einen Injection-Kontext (Konstruktor oder `inject()`).
</details>

<details>
<summary>Hint 2 – Mit merge navigieren und Seite zurücksetzen</summary>

```typescript
setSearch(term: string): void {
  this.router.navigate([], {
    relativeTo: this.route,
    queryParams: { search: term || null, page: 1 },
    queryParamsHandling: 'merge',
  });
}

setPage(page: number): void {
  this.router.navigate([], {
    relativeTo: this.route,
    queryParams: { page },
    queryParamsHandling: 'merge',
  });
}
```

`null` als Wert entfernt den Query Param aus der URL.
</details>

<details>
<summary>Hint 3 – withComponentInputBinding im Router</summary>

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};

// component
@Input() page = '1';
@Input() search = '';
```

Angular befüllt `@Input()`-Properties automatisch aus den Query Params, wenn der Name übereinstimmt.
</details>

## Beispiellösung

```typescript
// product-list.component.ts
import { Component, computed, inject, Input, signal } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ActivatedRoute, Router } from '@angular/router';
import { FormsModule } from '@angular/forms';

const MOCK_PRODUCTS = Array.from({ length: 50 }, (_, i) => ({
  id: i + 1,
  name: `Produkt ${i + 1}`,
  category: i % 3 === 0 ? 'A' : i % 3 === 1 ? 'B' : 'C',
}));

const PAGE_SIZE = 10;

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [FormsModule],
  template: `
    <input [(ngModel)]="searchTerm" (ngModelChange)="setSearch($event)" placeholder="Suche..." />

    <ul>
      @for (p of pageItems(); track p.id) {
        <li>{{ p.name }} ({{ p.category }})</li>
      }
    </ul>

    <div>
      <button (click)="setPage(currentPage() - 1)" [disabled]="currentPage() <= 1">‹</button>
      Seite {{ currentPage() }} / {{ totalPages() }}
      <button (click)="setPage(currentPage() + 1)" [disabled]="currentPage() >= totalPages()">›</button>
    </div>
  `,
})
export class ProductListComponent {
  private route = inject(ActivatedRoute);
  private router = inject(Router);

  protected searchTerm = '';

  // Reaktive Query Params als Signals
  private queryParamMap = toSignal(this.route.queryParamMap);

  protected currentPage = computed(() => {
    const raw = this.queryParamMap()?.get('page');
    const n = Number(raw);
    return Number.isFinite(n) && n > 0 ? n : 1;
  });

  protected currentSearch = computed(() =>
    this.queryParamMap()?.get('search') ?? ''
  );

  private filtered = computed(() => {
    const term = this.currentSearch().toLowerCase();
    return term
      ? MOCK_PRODUCTS.filter(p => p.name.toLowerCase().includes(term))
      : MOCK_PRODUCTS;
  });

  protected totalPages = computed(() =>
    Math.ceil(this.filtered().length / PAGE_SIZE)
  );

  protected pageItems = computed(() => {
    const start = (this.currentPage() - 1) * PAGE_SIZE;
    return this.filtered().slice(start, start + PAGE_SIZE);
  });

  setSearch(term: string): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { search: term || null, page: 1 },
      queryParamsHandling: 'merge',
    });
  }

  setPage(page: number): void {
    if (page < 1 || page > this.totalPages()) return;
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { page },
      queryParamsHandling: 'merge',
    });
  }
}
```

```typescript
// app.config.ts (Bonus: withComponentInputBinding)
import { provideRouter, withComponentInputBinding } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

## Weiterführendes
- **Tipp:** Kapsle `queryParamsHandling: 'merge'`-Navigationen in einen wiederverwendbaren Service (`QueryParamService`), um Boilerplate zu reduzieren und die Navigation testbar zu halten.
- **Deep Dive:** Lies die Angular-Doku zu [`NavigationExtras`](https://angular.dev/api/router/NavigationExtras) – insbesondere `replaceUrl: true`, um das Browser-Verlauf-Rauschen bei schnellen Filter-Änderungen zu minimieren.
- **Weiterführend:** Kombiniere Query Params mit `debounceTime` + `distinctUntilChanged` in einem Service, der die Observable-Welt mit Signals verbindet, um übermäßige Server-Requests zu vermeiden.

# Angular Dojo: RxJS `expand` – Rekursive Streams für Paginierung und Baumstrukturen

**Datum:** 2026-11-03  
**Dauer:** ~25 Minuten  
**Level:** Fortgeschritten

## Lernziel

Du lernst den selten genutzten, aber mächtigen `expand`-Operator von RxJS kennen und wendest ihn für zwei klassische Angular-Szenarien an: automatische Paginierung einer REST-API und das rekursive Laden einer Baumstruktur.

## Hintergrund & Theorie

Der `expand`-Operator ist der rekursive Bruder von `mergeMap`. Er wendet eine Projektionsfunktion auf jeden emittierten Wert an – und **auch auf alle resultierenden Werte der Projektion selbst**. Das Ergebnis ist eine Art RxJS-Rekursion: Das Observable expandiert sich selbst, bis eine Abbruchbedingung eintritt.

```
Quelle:  A ──────────────────────────────>
expand:  A → f(A)=B → f(B)=C → f(C)=EMPTY
Ausgabe: A──B──C──|
```

**Typischer Anwendungsfall – Paginierung:**  
Viele APIs liefern einen `nextPageUrl` oder `nextCursor` im Response-Body. Mit `expand` kann man alle Seiten automatisch und reaktiv nachladen, ohne verschachtelte Schleifen oder manuelle Rekursion.

**Typischer Anwendungsfall – Baumstrukturen:**  
Lazy-Loading eines Dateibaums: Erst der Root-Knoten wird geladen, dann on-demand die Kind-Knoten jedes geladenen Eintrags.

**Wichtig:** `expand` emittiert den **Ausgangswert und alle Zwischenwerte** – nicht nur das Endergebnis. Mit `EMPTY` als Rückgabe der Projektionsfunktion signalisiert man das Ende der Rekursion. Für sequenzielles Nachladen eignet sich `concatMap` als innerer Operator; mit `mergeMap` laufen alle Branches parallel.

## Aufgabe

Baue einen Angular-Service, der eine paginierte Produktliste vollständig und reaktiv von einer Mock-API lädt und die kombinierten Ergebnisse als Signal bereitstellt.

### Schritte

1. Erstelle ein `ProductApiService` mit einer `getPage(cursor: string | null)` Methode, die ein simuliertes HTTP-Response-Observable zurückgibt (mit `of(...)` und `delay`).

2. Implementiere eine `loadAll()` Methode, die `expand` nutzt: Starte mit `cursor: null`, lade jede Seite, gib `EMPTY` zurück wenn kein `nextCursor` vorhanden ist.

3. Sammle alle Produkte über alle Seiten hinweg mit `reduce` (oder `scan` falls Zwischenstände benötigt werden) in einem einzigen Array.

4. Expose das Ergebnis als Signal via `toSignal()` und zeige die Gesamtanzahl geladener Produkte in der Komponente an.

5. **Bonus:** Ersetze `of(...).delay(...)` durch `HttpClient`-Calls gegen eine echte API und nutze einen `takeUntil(destroy$)` oder `takeUntilDestroyed()` um Memory Leaks zu vermeiden.

```typescript
// Zu implementierende Signatur
interface PageResponse {
  items: Product[];
  nextCursor: string | null;
}

interface Product {
  id: number;
  name: string;
}
```

## Hints

<details>
<summary>Hint 1 – Grundstruktur von expand</summary>

```typescript
import { expand, EMPTY, reduce } from 'rxjs';

loadAll(): Observable<Product[]> {
  return this.getPage(null).pipe(
    expand(response =>
      response.nextCursor ? this.getPage(response.nextCursor) : EMPTY
    ),
    reduce((acc, response) => [...acc, ...response.items], [] as Product[])
  );
}
```

`expand` ruft die Funktion für jede Page-Response auf. Solange `nextCursor` gesetzt ist, wird die nächste Seite geladen. `EMPTY` beendet den rekursiven Ast.

</details>

<details>
<summary>Hint 2 – Mock-API und Signal-Integration</summary>

```typescript
import { of } from 'rxjs';
import { delay } from 'rxjs/operators';

private mockData: PageResponse[] = [
  { items: [{ id: 1, name: 'A' }, { id: 2, name: 'B' }], nextCursor: 'page2' },
  { items: [{ id: 3, name: 'C' }, { id: 4, name: 'D' }], nextCursor: 'page3' },
  { items: [{ id: 5, name: 'E' }], nextCursor: null },
];

getPage(cursor: string | null): Observable<PageResponse> {
  const index = cursor ? parseInt(cursor.replace('page', '')) - 1 : 0;
  return of(this.mockData[index]).pipe(delay(200));
}

// In der Komponente
readonly products = toSignal(this.productApi.loadAll(), { initialValue: [] });
```

</details>

## Beispiellösung

```typescript
// product-api.service.ts
import { Injectable, inject } from '@angular/core';
import { Observable, of, EMPTY } from 'rxjs';
import { delay, expand, reduce } from 'rxjs/operators';

export interface Product { id: number; name: string; }
interface PageResponse { items: Product[]; nextCursor: string | null; }

@Injectable({ providedIn: 'root' })
export class ProductApiService {
  private readonly pages: PageResponse[] = [
    { items: [{ id: 1, name: 'Alpha' }, { id: 2, name: 'Beta' }], nextCursor: 'p2' },
    { items: [{ id: 3, name: 'Gamma' }, { id: 4, name: 'Delta' }], nextCursor: 'p3' },
    { items: [{ id: 5, name: 'Epsilon' }], nextCursor: null },
  ];

  getPage(cursor: string | null): Observable<PageResponse> {
    const idx = cursor ? Number(cursor.replace('p', '')) - 1 : 0;
    return of(this.pages[idx]).pipe(delay(300));
  }

  loadAll(): Observable<Product[]> {
    return this.getPage(null).pipe(
      expand(response =>
        response.nextCursor ? this.getPage(response.nextCursor) : EMPTY
      ),
      reduce((acc, response) => [...acc, ...response.items], [] as Product[])
    );
  }
}

// product-list.component.ts
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ProductApiService } from './product-api.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    @if (products().length === 0) {
      <p>Lade Produkte...</p>
    } @else {
      <p>{{ products().length }} Produkte geladen:</p>
      <ul>
        @for (p of products(); track p.id) {
          <li>{{ p.id }}: {{ p.name }}</li>
        }
      </ul>
    }
  `,
})
export class ProductListComponent {
  private readonly api = inject(ProductApiService);
  readonly products = toSignal(this.api.loadAll(), { initialValue: [] });
}
```

### Baumstruktur-Variante (Bonus)

```typescript
interface TreeNode { id: number; name: string; childIds: number[]; }

loadTree(rootId: number): Observable<TreeNode[]> {
  return this.fetchNode(rootId).pipe(
    expand(node =>
      node.childIds.length > 0
        ? merge(...node.childIds.map(id => this.fetchNode(id)))
        : EMPTY
    ),
    toArray()
  );
}
```

## Weiterführendes

- [`expand` Dokumentation auf rxjs.dev](https://rxjs.dev/api/operators/expand) – offizielle API-Referenz mit Marble-Diagramm
- Kombination mit `scan` statt `reduce`: Emittiere Zwischenstände (z. B. für Ladebalkene), indem du den Accumulator nach jeder Seite ausgibst
- Für Cursor-basierte APIs mit `HttpClient`: Nutze `takeWhile(res => res.nextCursor !== null, true)` als Alternative zu `EMPTY`, um den letzten Response noch zu emittieren
- `expand` mit `concatMap` (statt default `mergeMap`) für strikte sequenzielle Verarbeitung: `expand(res => res.next ? this.load(res.next) : EMPTY, 1)` (der zweite Parameter `concurrent` steuert die Parallelität)

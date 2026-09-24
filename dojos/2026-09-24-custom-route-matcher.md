# Angular Dojo: Custom Route Matcher
**Datum:** 2026-09-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man mit `UrlMatcher` eigene Route-Matching-Logik definiert, die über einfache Pfadmuster hinausgeht – zum Beispiel für Slugs, Legacy-URLs, sprachspezifische Routen oder beliebige reguläre Ausdrücke.

## Hintergrund & Theorie

Angular's Router gleicht URLs normalerweise über einfache Pfade (`path: 'products/:id'`) oder Wildcards (`path: '**'`) ab. Für komplexere Szenarien bietet der Router das `UrlMatcher`-Interface an.

Ein `UrlMatcher` ist eine Funktion mit folgender Signatur:

```typescript
type UrlMatcher = (
  segments: UrlSegment[],
  group: UrlSegmentGroup,
  route: Route
) => UrlMatchResult | null;
```

Gibt sie `null` zurück, gilt die Route als nicht gemacht. Gibt sie ein Objekt zurück, enthält dieses die gematchten `consumed` Segmente und optionale `posParams` (Positions-Parameter).

Typische Anwendungsfälle:
- **Slug-Matching**: Nur URLs mit einem bestimmten Format (z.B. `blog/my-post-slug-123`) matchen
- **Legacy-URL-Unterstützung**: Alte URL-Formate weiterleiten
- **Sprachpräfix-Routing**: `/de/produkte`, `/en/products` auf dieselbe Komponente mappen
- **Optionale Segmente**: Routen mit und ohne optionalen Pfadteil

Der `UrlMatcher` ersetzt die `path`-Eigenschaft einer Route – beide zusammen können nicht verwendet werden.

## Aufgabe

Erstelle eine Angular-Anwendung mit einem Custom Route Matcher, der Produkt-URLs mit einem spezifischen Format matcht: `products/<kategorie>-<id>` (z.B. `products/elektronik-42`). Nur URLs, bei denen das zweite Segment das Muster `<string>-<number>` hat, sollen auf die `ProductDetailComponent` weitergeleitet werden. Alle anderen sollen auf eine `NotFoundComponent` fallen.

### Schritte

1. **Erstelle die Komponenten** `ProductDetailComponent` und `NotFoundComponent` als Standalone-Komponenten.

2. **Implementiere den `UrlMatcher`** als reine Funktion in einer eigenen Datei `product-url-matcher.ts`:
   - Das erste Segment muss exakt `products` sein.
   - Das zweite Segment muss dem Muster `<kategorie>-<id>` entsprechen, wobei `<id>` eine Zahl ist.
   - Extrahiere `kategorie` und `id` als `posParams` (PositionalParameter) und gib sie als `UrlSegment` zurück.

3. **Registriere den Matcher in der Route-Konfiguration** mit `matcher: productUrlMatcher` (anstelle von `path`).

4. **Nutze die extrahierten Parameter** in `ProductDetailComponent` via `ActivatedRoute`:
   - Lese `posParams` aus `route.snapshot.params` aus (Matcher-Parameter landen im `params`-Map).
   - Zeige `kategorie` und `id` im Template an.

5. **Teste den Matcher** mit mehreren URLs:
   - `products/elektronik-42` → ProductDetailComponent
   - `products/buecher-7` → ProductDetailComponent
   - `products/invalid` → NotFoundComponent
   - `products/123-abc` → NotFoundComponent

## Hints

<details>
<summary>Hint 1 – UrlMatcher-Grundgerüst</summary>

```typescript
import { UrlMatcher, UrlSegment } from '@angular/router';

export const productUrlMatcher: UrlMatcher = (segments) => {
  if (segments.length === 2 && segments[0].path === 'products') {
    const match = segments[1].path.match(/^([a-z]+)-(\d+)$/);
    if (match) {
      return {
        consumed: segments,
        posParams: {
          kategorie: new UrlSegment(match[1], {}),
          id: new UrlSegment(match[2], {}),
        },
      };
    }
  }
  return null;
};
```

</details>

<details>
<summary>Hint 2 – Parameter aus ActivatedRoute lesen</summary>

`posParams` eines `UrlMatcher` werden im `ActivatedRoute` unter `route.snapshot.params` verfügbar – die Schlüssel entsprechen den Eigenschaftsnamen aus `posParams`:

```typescript
@Component({ ... })
export class ProductDetailComponent {
  private route = inject(ActivatedRoute);
  
  kategorie = this.route.snapshot.params['kategorie'];
  id = this.route.snapshot.params['id'];
}
```

Oder reaktiv mit Signals:

```typescript
params = toSignal(this.route.params);
kategorie = computed(() => this.params()?.['kategorie']);
id = computed(() => this.params()?.['id']);
```

</details>

<details>
<summary>Hint 3 – Route-Konfiguration mit Matcher</summary>

```typescript
export const routes: Routes = [
  {
    matcher: productUrlMatcher,
    component: ProductDetailComponent,
  },
  {
    path: '**',
    component: NotFoundComponent,
  },
];
```

Wichtig: `matcher` und `path` schließen sich gegenseitig aus. Wenn `matcher` gesetzt ist, darf `path` nicht gesetzt sein.

</details>

## Beispiellösung

```typescript
// product-url-matcher.ts
import { UrlMatcher, UrlSegment } from '@angular/router';

export const productUrlMatcher: UrlMatcher = (segments) => {
  if (segments.length === 2 && segments[0].path === 'products') {
    const match = segments[1].path.match(/^([a-z][a-z0-9]*)-(\d+)$/);
    if (match) {
      return {
        consumed: segments,
        posParams: {
          kategorie: new UrlSegment(match[1], {}),
          id: new UrlSegment(match[2], {}),
        },
      };
    }
  }
  return null;
};
```

```typescript
// product-detail.component.ts
import { Component, inject, computed } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <h2>Produktdetails</h2>
    <p>Kategorie: <strong>{{ kategorie() }}</strong></p>
    <p>ID: <strong>{{ id() }}</strong></p>
  `,
})
export class ProductDetailComponent {
  private route = inject(ActivatedRoute);
  private params = toSignal(this.route.params, { initialValue: {} });

  kategorie = computed(() => this.params()['kategorie']);
  id = computed(() => this.params()['id']);
}
```

```typescript
// not-found.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-not-found',
  standalone: true,
  template: `<h2>404 – Seite nicht gefunden</h2>`,
})
export class NotFoundComponent {}
```

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { productUrlMatcher } from './product-url-matcher';
import { ProductDetailComponent } from './product-detail.component';
import { NotFoundComponent } from './not-found.component';

export const routes: Routes = [
  {
    matcher: productUrlMatcher,
    component: ProductDetailComponent,
  },
  {
    path: '**',
    component: NotFoundComponent,
  },
];
```

```typescript
// matcher.spec.ts – Unit-Test des Matchers
import { UrlSegment } from '@angular/router';
import { productUrlMatcher } from './product-url-matcher';

function seg(path: string): UrlSegment {
  return new UrlSegment(path, {});
}

describe('productUrlMatcher', () => {
  it('matcht gültige Produkt-URLs', () => {
    const result = productUrlMatcher([seg('products'), seg('elektronik-42')], {} as any, {} as any);
    expect(result).not.toBeNull();
    expect(result!.posParams!['kategorie'].path).toBe('elektronik');
    expect(result!.posParams!['id'].path).toBe('42');
  });

  it('lehnt URLs ohne Zahlenteil ab', () => {
    const result = productUrlMatcher([seg('products'), seg('elektronik')], {} as any, {} as any);
    expect(result).toBeNull();
  });

  it('lehnt URLs mit falschem ersten Segment ab', () => {
    const result = productUrlMatcher([seg('shop'), seg('elektronik-42')], {} as any, {} as any);
    expect(result).toBeNull();
  });
});
```

## Weiterführendes

- **Kombinierter Matcher mit `or`-Logik**: Mehrere Matcher lassen sich mit einer Hilfsfunktion kombinieren, die die erste nicht-null Antwort zurückgibt – nützlich für Legacy-URL-Migration.
- **Offizielle Docs**: [Angular Router – UrlMatcher](https://angular.dev/api/router/UrlMatcher)
- **Praxistipp**: `UrlMatcher` sind reine Funktionen ohne Abhängigkeiten – sie sind ideal für Unit-Tests, da kein TestBed benötigt wird.
- **Lazy Loading mit Matcher**: `matcher` kann auch zusammen mit `loadComponent` oder `loadChildren` verwendet werden, um gematchte Routen lazy zu laden.

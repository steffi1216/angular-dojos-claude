# Angular Dojo: Incremental Hydration
**Datum:** 2026-07-30
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie Incremental Hydration in Angular 19+ funktioniert: Teile der Server-gerenderten Seite werden erst dann im Browser aktiviert ("hydriert"), wenn sie tatsächlich benötigt werden – mit `@defer`-Blöcken als Steuerungsmechanismus.

## Hintergrund & Theorie

Bei klassischem Angular SSR (Universal) wird die gesamte Seite auf dem Server gerendert und dann im Browser in einem einzigen Schritt hydriert – alle Komponenten werden gleichzeitig aktiviert. Das kostet Zeit und verzögert die Interaktivität (TTI).

**Incremental Hydration** (Angular 19, stabil in Angular 20) löst dieses Problem: Statt alles auf einmal zu hydrieren, werden Teile der Seite nur dann hydriert, wenn ein definierter Trigger eintritt:

| Trigger | Verhalten |
|---|---|
| `hydrate on viewport` | Hydriert, wenn das Element in den sichtbaren Bereich scrollt |
| `hydrate on interaction` | Hydriert bei erstem Klick oder `keydown` |
| `hydrate on hover` | Hydriert bei `mouseenter` oder `focusin` |
| `hydrate on idle` | Hydriert, wenn der Browser idle ist (`requestIdleCallback`) |
| `hydrate when (expr)` | Hydriert, wenn ein Signal/Observable-Ausdruck `true` wird |
| `hydrate never` | Bleibt dauerhaft als statisches Server-HTML, kein JS |

Bis zur Hydration bleibt das Server-HTML **sichtbar und statisch** im DOM – kein "Flash of Unstyled Content". Angular injiziert minimal JavaScript, um auf den Trigger zu warten.

**Voraussetzungen:**
- Angular 19+ mit SSR (`@angular/ssr`)
- `provideClientHydration(withIncrementalHydration())` in `app.config.ts`
- `@defer`-Blöcke mit `hydrate`-Trigger im Template

## Aufgabe

Erstelle eine **Produktseite** (E-Commerce), die SSR nutzt. Der sichtbare Hero-Bereich ist sofort voll interaktiv, aber der Bewertungsbereich und das Empfehlungs-Karussell am Ende der Seite werden erst bei Bedarf hydriert.

### Schritte

1. Aktiviere Incremental Hydration in `app.config.ts` durch Hinzufügen von `withIncrementalHydration()`.

2. Erstelle eine `ProductHeroComponent` (standalone), die sofort hydriert wird (keine `@defer`-Umhüllung nötig).

3. Erstelle eine `ReviewSectionComponent` mit einem Sterne-Rating und Kommentarliste. Wrapping in `@defer (hydrate on viewport)` – sie soll erst hydrieren, wenn der User dorthin scrollt.

4. Erstelle eine `RecommendationsComponent` (Produktkacheln). Wrap sie mit `@defer (hydrate on interaction)` – erst bei erstem Klick oder Tastendruck hydriert sie.

5. Füge einen `@placeholder`-Block für die `@defer`-Blöcke ein, damit das Server-HTML vor der Hydration korrekt angezeigt wird.

6. Erstelle eine `UserStatusComponent`, die mit `@defer (hydrate never)` markiert ist – sie soll rein statischer Server-Content bleiben (z. B. "Eingeloggt als: Max Mustermann").

### Erwartetes Ergebnis

- Die Seite rendert auf dem Server vollständig.
- Im Browser ist der Hero sofort interaktiv.
- Reviews hydrieren beim Hineinscrolling (kein Klick nötig).
- Recommendations hydrieren erst bei Interaktion.
- Der User-Status bleibt statisches HTML, kein JS.

## Hints

<details>
<summary>Hint 1 – withIncrementalHydration konfigurieren</summary>

In `app.config.ts` (Client-seitige Konfiguration):

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration, withIncrementalHydration } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter([]),
    provideClientHydration(
      withIncrementalHydration()
    ),
  ],
};
```

Für den Server (`app.config.server.ts`) ändert sich nichts – `provideServerRendering()` reicht weiterhin aus.
</details>

<details>
<summary>Hint 2 – @defer mit hydrate-Triggern im Template</summary>

```html
<!-- Sofort hydriert (kein defer nötig) -->
<app-product-hero [product]="product" />

<!-- Hydriert wenn sichtbar -->
@defer (hydrate on viewport) {
  <app-review-section [productId]="product.id" />
} @placeholder {
  <!-- Wichtig: placeholder enthält das Server-HTML vor der Hydration.
       Angular nutzt es, um den DOM zu vergleichen. -->
  <div class="reviews-placeholder">Bewertungen werden geladen...</div>
}

<!-- Hydriert bei erster Interaktion -->
@defer (hydrate on interaction) {
  <app-recommendations [category]="product.category" />
} @placeholder {
  <div class="recs-placeholder">Ähnliche Produkte</div>
}

<!-- Nie hydriert – bleibt statisches HTML -->
@defer (hydrate never) {
  <app-user-status />
}
```

</details>

<details>
<summary>Hint 3 – hydrate when mit Signal-Ausdruck</summary>

Mit einem Signal-Ausdruck kannst du die Hydration programmatisch auslösen:

```typescript
@Component({
  template: `
    <button (click)="showReviews.set(true)">Bewertungen anzeigen</button>

    @defer (hydrate when showReviews()) {
      <app-review-section />
    } @placeholder {
      <p>Bewertungen ausgeblendet</p>
    }
  `
})
export class ProductPageComponent {
  showReviews = signal(false);
}
```

Das `@defer`-Block hydriert erst, wenn `showReviews()` den Wert `true` zurückgibt. Auf dem Server wird der Inhalt trotzdem vollständig gerendert.
</details>

## Beispiellösung

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration, withIncrementalHydration } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration(withIncrementalHydration()),
  ],
};
```

```typescript
// product-page.component.ts
import { Component, signal } from '@angular/core';
import { ProductHeroComponent } from './product-hero.component';
import { ReviewSectionComponent } from './review-section.component';
import { RecommendationsComponent } from './recommendations.component';
import { UserStatusComponent } from './user-status.component';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  imageUrl: string;
}

@Component({
  selector: 'app-product-page',
  standalone: true,
  imports: [
    ProductHeroComponent,
    ReviewSectionComponent,
    RecommendationsComponent,
    UserStatusComponent,
  ],
  template: `
    <!-- Statischer User-Status – kein JavaScript, kein Rehydration-Overhead -->
    @defer (hydrate never) {
      <app-user-status [username]="'Max Mustermann'" />
    }

    <!-- Hero: sofort interaktiv nach SSR -->
    <app-product-hero [product]="product" />

    <!-- Reviews: Hydration erst beim Einblenden in den Viewport -->
    @defer (hydrate on viewport) {
      <app-review-section [productId]="product.id" />
    } @placeholder {
      <section class="reviews-shell" aria-busy="true">
        <h2>Kundenbewertungen</h2>
        <p>Bewertungen werden gleich geladen...</p>
      </section>
    }

    <!-- Empfehlungen: Hydration bei erster Interaktion des Users -->
    @defer (hydrate on interaction) {
      <app-recommendations [category]="product.category" />
    } @placeholder {
      <section class="recs-shell">
        <h2>Ähnliche Produkte</h2>
        <div class="recs-grid-placeholder"></div>
      </section>
    }

    <!-- Bedingte Hydration per Signal -->
    <button (click)="showDetails.set(true)" *ngIf="!showDetails()">
      Technische Details anzeigen
    </button>
    @defer (hydrate when showDetails()) {
      <app-tech-details [productId]="product.id" />
    } @placeholder {
      <div hidden></div>
    }
  `,
})
export class ProductPageComponent {
  showDetails = signal(false);

  product: Product = {
    id: 'prod-42',
    name: 'Wireless Noise-Cancelling Headphones',
    price: 299,
    category: 'audio',
    imageUrl: '/assets/headphones.jpg',
  };
}
```

```typescript
// review-section.component.ts
import { Component, input, OnInit, signal } from '@angular/core';

interface Review {
  author: string;
  rating: number;
  text: string;
}

@Component({
  selector: 'app-review-section',
  standalone: true,
  template: `
    <section class="reviews">
      <h2>Kundenbewertungen ({{ reviews().length }})</h2>
      @for (review of reviews(); track review.author) {
        <div class="review-card">
          <strong>{{ review.author }}</strong>
          <span class="stars">{{ '★'.repeat(review.rating) }}{{ '☆'.repeat(5 - review.rating) }}</span>
          <p>{{ review.text }}</p>
        </div>
      }
    </section>
  `,
})
export class ReviewSectionComponent implements OnInit {
  productId = input.required<string>();
  reviews = signal<Review[]>([]);

  ngOnInit() {
    // Im echten Projekt: HTTP-Call; hier statische Demo-Daten
    this.reviews.set([
      { author: 'Anna K.', rating: 5, text: 'Ausgezeichnete Klangqualität!' },
      { author: 'Tom M.', rating: 4, text: 'Sehr komfortabel, Akku könnte besser sein.' },
      { author: 'Lisa S.', rating: 5, text: 'Beste Kopfhörer die ich je hatte.' },
    ]);
  }
}
```

```typescript
// recommendations.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-recommendations',
  standalone: true,
  template: `
    <section class="recommendations">
      <h2>Ähnliche Produkte in "{{ category() }}"</h2>
      <div class="recs-grid">
        @for (item of mockRecs; track item.id) {
          <div class="rec-card">
            <img [src]="item.imageUrl" [alt]="item.name" loading="lazy" />
            <p>{{ item.name }}</p>
            <strong>{{ item.price | currency:'EUR' }}</strong>
          </div>
        }
      </div>
    </section>
  `,
})
export class RecommendationsComponent {
  category = input.required<string>();

  mockRecs = [
    { id: 1, name: 'Studio Headphones Pro', price: 199, imageUrl: '/assets/rec1.jpg' },
    { id: 2, name: 'Earbuds Ultra', price: 149, imageUrl: '/assets/rec2.jpg' },
    { id: 3, name: 'Portable Speaker', price: 89, imageUrl: '/assets/rec3.jpg' },
  ];
}
```

## Weiterführendes

- **`hydrate never` vs. Static Site Generation:** Komponenten mit `hydrate never` ersparen komplett den JS-Overhead für diesen DOM-Bereich – ideal für rein dekorative Inhalte oder statische Server-Daten.
- **Devtools-Check:** In den Angular DevTools (Chrome Extension) siehst du unter "Components" den Hydration-Status jeder Komponente – grün = hydriert, grau = noch nicht hydriert.
- **Kombination mit `@loading`:** Ein `@loading`-Block wird angezeigt, während der Lazy-Chunk für den `@defer`-Inhalt geladen wird – nützlich bei großen Komponenten, die erst nachgeladen werden.
- **Offizielle Docs:** [angular.dev/guide/incremental-hydration](https://angular.dev/guide/incremental-hydration)
- **Zusammenspiel mit Signals:** Da Hydration-Trigger reaktiv auf Signals reagieren können, lassen sich sehr granulare UX-Patterns umsetzen (z. B. "hydriere diesen Abschnitt, sobald der User das Formular oben abgeschickt hat").

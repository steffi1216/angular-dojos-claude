# Angular Dojo: Route-scoped Dependency Injection
**Datum:** 2026-10-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Services gezielt auf Route-Ebene bereitstellen kannst, sodass jede Feature-Route ihren eigenen isolierten Service-Kontext erhält – ohne globale Singleton-Konflikte.

## Hintergrund & Theorie
Seit Angular 14 können Route-Definitionen ein `providers`-Array enthalten. Dadurch wird ein eigener Injector-Knoten im Hierarchiebaum erstellt – direkt unter dem Router-Injector, aber oberhalb der Komponenten dieser Route. Das ermöglicht:

- **Feature-Isolation**: Jede Lazy-Route bekommt ihren eigenen Service-Zustand.
- **Kein Memory-Leak**: Der Injector (und alle seine Services) wird zerstört, sobald man die Route verlässt.
- **Keine SharedModule-Hacks**: Services müssen nicht mehr als Singleton im Root oder in einem freigegebenen Modul registriert werden.

Der Route-Injector lebt im Scope der Route-Hierarchie:
```
Root Injector
  └─ Environment Injector (lazy chunk)
       └─ Route Injector (providers: [...] in der Route)
            └─ Component Injector
```

Services im `providers`-Array einer Route-Definition werden mit `providedIn: 'root'` überschrieben, wenn sie explizit in `providers` der Route angegeben werden. Ein Service ohne `providedIn` oder mit `providedIn: 'any'` ist ideal für Route-scoped Services.

Die Route-Konfiguration unterstützt dasselbe Provider-Format wie `@NgModule.providers` oder `bootstrapApplication`'s `providers`.

## Aufgabe
Erstelle eine kleine Anwendung mit zwei Feature-Routen (`/cart` und `/wishlist`), die beide einen `ItemListService` verwenden – aber jeweils mit **eigenem isoliertem State**. Beim Navigieren zwischen den Routen soll der jeweilige State unabhängig bleiben.

### Schritte
1. Erstelle einen `ItemListService` **ohne** `providedIn: 'root'` (nur `Injectable()`).
2. Definiere zwei Routen `/cart` und `/wishlist` mit je einer eigenen `providers`-Deklaration für `ItemListService`.
3. Erstelle zwei Komponenten `CartComponent` und `WishlistComponent`, die `ItemListService` per `inject()` beziehen.
4. Füge in jeder Komponente Buttons zum Hinzufügen und Entfernen von Artikeln hinzu (nutze Signals im Service).
5. Navigiere zwischen den Routen und beobachte, dass jede Route ihre eigene unabhängige Liste hat.
6. Optional: Füge eine Parent-Route mit `providers: [SharedCartContextService]` hinzu und zeige, wie ein Service über mehrere Child-Routen geteilt werden kann.

## Hints
<details>
<summary>Hint 1 – Injectable ohne Root</summary>

Ein Service der **nicht** `providedIn: 'root'` hat, wird nur instanziiert wenn er explizit bereitgestellt wird:

```typescript
@Injectable()  // kein providedIn!
export class ItemListService {
  readonly items = signal<string[]>([]);

  add(item: string) {
    this.items.update(list => [...list, item]);
  }

  remove(item: string) {
    this.items.update(list => list.filter(i => i !== item));
  }
}
```

</details>
<details>
<summary>Hint 2 – providers in der Route-Definition</summary>

```typescript
export const routes: Routes = [
  {
    path: 'cart',
    component: CartComponent,
    providers: [ItemListService]  // eigene Instanz für /cart
  },
  {
    path: 'wishlist',
    component: WishlistComponent,
    providers: [ItemListService]  // eigene Instanz für /wishlist
  }
];
```

Jede Route erstellt eine **neue** `ItemListService`-Instanz. Die Instanz wird zerstört, wenn du die Route verlässt.

</details>
<details>
<summary>Hint 3 – Shared Context über Parent-Route</summary>

Um einen Service über mehrere Child-Routen zu teilen (aber nicht global):

```typescript
export const routes: Routes = [
  {
    path: 'shop',
    providers: [SharedCartContextService],  // geteilt zwischen allen Kinder-Routen
    children: [
      { path: 'cart', component: CartComponent },
      { path: 'wishlist', component: WishlistComponent }
    ]
  }
];
```

`SharedCartContextService` lebt im Route-Injector von `/shop` und ist für beide Kinder sichtbar.

</details>

## Beispiellösung

```typescript
// item-list.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable()  // KEIN providedIn: 'root'
export class ItemListService {
  readonly items = signal<string[]>([]);

  add(item: string) {
    this.items.update(list => [...list, item]);
  }

  remove(item: string) {
    this.items.update(list => list.filter(i => i !== item));
  }
}

// cart.component.ts
import { Component, inject } from '@angular/core';
import { ItemListService } from './item-list.service';

@Component({
  selector: 'app-cart',
  standalone: true,
  template: `
    <h2>Warenkorb</h2>
    <ul>
      @for (item of service.items(); track item) {
        <li>{{ item }} <button (click)="service.remove(item)">x</button></li>
      }
    </ul>
    <button (click)="service.add('Artikel ' + service.items().length)">
      Hinzufügen
    </button>
  `
})
export class CartComponent {
  protected service = inject(ItemListService);
}

// wishlist.component.ts – identisches Template, eigene Service-Instanz
@Component({
  selector: 'app-wishlist',
  standalone: true,
  template: `
    <h2>Wunschliste</h2>
    <ul>
      @for (item of service.items(); track item) {
        <li>{{ item }} <button (click)="service.remove(item)">x</button></li>
      }
    </ul>
    <button (click)="service.add('Wunsch ' + service.items().length)">
      Hinzufügen
    </button>
  `
})
export class WishlistComponent {
  protected service = inject(ItemListService);
}

// app.routes.ts
import { Routes } from '@angular/router';
import { ItemListService } from './item-list.service';

export const routes: Routes = [
  {
    path: 'cart',
    component: CartComponent,
    providers: [ItemListService]   // isolierte Instanz für Cart
  },
  {
    path: 'wishlist',
    component: WishlistComponent,
    providers: [ItemListService]   // isolierte Instanz für Wishlist
  },
  { path: '', redirectTo: 'cart', pathMatch: 'full' }
];

// app.component.ts – Navigation
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav>
      <a routerLink="/cart">Warenkorb</a> |
      <a routerLink="/wishlist">Wunschliste</a>
    </nav>
    <router-outlet />
  `
})
export class AppComponent {}
```

**Beobachtung**: Füge Artikel in `/cart` hinzu, navigiere zu `/wishlist` – die Liste ist leer. Kehre zu `/cart` zurück – ebenfalls leer, da die Instanz beim Verlassen der Route zerstört wurde. Für persistenten State brauchst du einen Root-Service oder einen State-Store.

## Weiterführendes
- [Angular Docs – Route-level providers](https://angular.dev/guide/di/hierarchical-dependency-injection#route-level-providers)
- Kombiniere Route-scoped Services mit `@defer` für maximale Lazy-Loading-Isolation
- Für persistenten Feature-State: `@ngrx/signals` mit Feature-Slices als Alternative zu Route-scoped Services
- Prüfe mit Angular DevTools den Injector-Baum: Route-Injektoren erscheinen als eigene Knoten im Baum

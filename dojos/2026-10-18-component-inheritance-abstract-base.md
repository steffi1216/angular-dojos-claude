# Angular Dojo: Component Inheritance & Abstract Base Components

**Datum:** 2026-10-18
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du mit abstrakten Basisklassen in Angular wiederverwendbare Komponentenlogik zentralisierst, ohne Code-Duplikation – und verstehst dabei die Fallstricke beim Vererben von Angular-Lifecycle-Hooks und Dependency Injection.

## Hintergrund & Theorie

Angular-Komponenten sind TypeScript-Klassen, die sich wie jede andere Klasse vererben lassen. Das eröffnet ein mächtiges Muster: eine **abstrakte Basisklasse** hält gemeinsame Logik (z.B. Loading-State, Destroy-Handling, Formularbasis), während konkrete Komponenten diese erben und spezialisieren.

Wichtige Regeln:

**1. `@Component` nicht in der Basisklasse:**  
Die Basisklasse ist eine normale TypeScript-Klasse oder trägt höchstens `@Directive()`. Angular benötigt den Decorator nur auf der konkreten Klasse.

**2. Lifecycle-Hooks werden vererbt, aber nicht automatisch aufgerufen:**  
Wenn die Basisklasse `ngOnInit()` definiert und die Kindklasse es überschreibt ohne `super.ngOnInit()`, läuft die Elternlogik nicht.

**3. Dependency Injection via `inject()`:**  
Seit Angular 14+ empfiehlt sich `inject()` statt Constructor-Injection in Basisklassen – der Kindklassen-Konstruktor muss keine Services weiterreichen.

**4. `DestroyRef` / `takeUntilDestroyed`:**  
Perfekt für Basisklassen, da `inject()` im Klassenfeld funktioniert und der Scope automatisch an die Kindinstanz gebunden ist.

Dieses Muster eignet sich besonders für: Listenseiten mit Filterung/Paginierung, Formular-Basis-Komponenten, Feature-Seiten mit einheitlichem Loading/Error-Handling.

## Aufgabe

Erstelle eine abstrakte Basiskomponente `AbstractListComponent<T>`, die gemeinsames Verhalten für Listen-Komponenten kapselt: Loading-State, Error-Handling, Suche und Destroy-Management. Leite zwei konkrete Komponenten davon ab.

### Schritte

1. **Abstrakte Basisklasse erstellen** (`abstract-list.base.ts`):
   - `@Directive()` Decorator (kein `@Component`)
   - Felder mit `inject()`: `destroyRef`, `loadingService` oder eigene Signals
   - Abstrakte Methode `loadData(): Observable<T[]>`
   - Signal-Felder: `items = signal<T[]>([])`, `loading = signal(false)`, `error = signal<string | null>(null)`, `searchQuery = signal('')`
   - `filteredItems = computed(() => ...)` – filtert `items()` anhand von `searchQuery()`
   - `ngOnInit()`: ruft `loadData()` auf, befüllt Signals, handled Fehler via `catchError`
   - `takeUntilDestroyed(this.destroyRef)` für automatisches Cleanup

2. **Erste konkrete Komponente** (`user-list.component.ts`):
   - Erbt `AbstractListComponent<User>`
   - Implementiert `loadData()` mit einem `HttpClient`-Call (oder einem Mock-Service)
   - Eigenes Template: zeigt `filteredItems()` in einer `@for`-Schleife, Suchfeld bindet an `searchQuery`

3. **Zweite konkrete Komponente** (`product-list.component.ts`):
   - Erbt ebenfalls `AbstractListComponent<Product>`
   - Überschreibt `ngOnInit()` mit `super.ngOnInit()` und zusätzlicher Logik (z.B. Kategorien laden)
   - Anderes Template, gleiche Basislogik

4. **Demo & Test:**
   - Verifiziere, dass ein Destroy der Komponente das Observable automatisch abmeldet
   - Teste, dass `filteredItems` reaktiv auf `searchQuery`-Änderungen reagiert

## Hints

<details>
<summary>Hint 1 – Basisklasse mit inject()</summary>

```typescript
import { Directive, OnInit, computed, signal } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';
import { Observable, catchError, finalize, of } from 'rxjs';

@Directive()
export abstract class AbstractListComponent<T> implements OnInit {
  private destroyRef = inject(DestroyRef);

  readonly items = signal<T[]>([]);
  readonly loading = signal(false);
  readonly error = signal<string | null>(null);
  readonly searchQuery = signal('');

  readonly filteredItems = computed(() => {
    const q = this.searchQuery().toLowerCase();
    if (!q) return this.items();
    return this.items().filter(item =>
      JSON.stringify(item).toLowerCase().includes(q)
    );
  });

  abstract loadData(): Observable<T[]>;

  ngOnInit(): void {
    this.loading.set(true);
    this.loadData().pipe(
      catchError(err => {
        this.error.set(err.message ?? 'Unbekannter Fehler');
        return of([]);
      }),
      finalize(() => this.loading.set(false)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => this.items.set(data));
  }
}
```

</details>

<details>
<summary>Hint 2 – Kindkomponente mit super.ngOnInit()</summary>

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { AbstractListComponent } from './abstract-list.base';
import { Observable } from 'rxjs';
import { UserService } from './user.service';

interface User { id: number; name: string; email: string; }

@Component({
  selector: 'app-user-list',
  standalone: true,
  template: `
    <input [value]="searchQuery()"
           (input)="searchQuery.set($any($event.target).value)"
           placeholder="Suchen..." />

    @if (loading()) { <p>Lade...</p> }
    @if (error()) { <p class="error">{{ error() }}</p> }

    @for (user of filteredItems(); track user.id) {
      <div>{{ user.name }} – {{ user.email }}</div>
    } @empty {
      <p>Keine Einträge gefunden.</p>
    }
  `
})
export class UserListComponent extends AbstractListComponent<User> {
  private userService = inject(UserService);

  override loadData(): Observable<User[]> {
    return this.userService.getUsers();
  }
}
```

Für die `ProductListComponent` mit zusätzlichem `ngOnInit`:

```typescript
@Component({ /* ... */ })
export class ProductListComponent extends AbstractListComponent<Product> implements OnInit {
  private productService = inject(ProductService);
  readonly categories = signal<string[]>([]);

  override ngOnInit(): void {
    super.ngOnInit(); // WICHTIG: Basislogik nicht vergessen!
    this.productService.getCategories()
      .subscribe(cats => this.categories.set(cats));
  }

  override loadData(): Observable<Product[]> {
    return this.productService.getProducts();
  }
}
```

</details>

## Beispiellösung

```typescript
// abstract-list.base.ts
import { Directive, OnInit, computed, signal, inject } from '@angular/core';
import { DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Observable, catchError, finalize, of } from 'rxjs';

@Directive()
export abstract class AbstractListComponent<T extends Record<string, unknown>>
  implements OnInit {

  private destroyRef = inject(DestroyRef);

  readonly items = signal<T[]>([]);
  readonly loading = signal(false);
  readonly error = signal<string | null>(null);
  readonly searchQuery = signal('');

  readonly filteredItems = computed(() => {
    const q = this.searchQuery().toLowerCase().trim();
    if (!q) return this.items();
    return this.items().filter(item =>
      Object.values(item).some(v =>
        String(v).toLowerCase().includes(q)
      )
    );
  });

  abstract loadData(): Observable<T[]>;

  ngOnInit(): void {
    this.loading.set(true);
    this.error.set(null);

    this.loadData().pipe(
      catchError(err => {
        this.error.set(err?.message ?? 'Fehler beim Laden der Daten');
        return of([] as T[]);
      }),
      finalize(() => this.loading.set(false)),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => this.items.set(data));
  }
}

// user-list.component.ts
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { AbstractListComponent } from './abstract-list.base';

interface User { id: number; name: string; email: string; username: string; }

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [],
  template: `
    <div class="list-container">
      <h2>Benutzerliste</h2>
      <input
        type="search"
        placeholder="Benutzer suchen..."
        [value]="searchQuery()"
        (input)="searchQuery.set($any($event.target).value)" />

      @if (loading()) {
        <div class="loading">Lade Benutzer...</div>
      }

      @if (error(); as err) {
        <div class="error">{{ err }}</div>
      }

      <ul>
        @for (user of filteredItems(); track user.id) {
          <li>
            <strong>{{ user.name }}</strong>
            <span>{{ user.email }}</span>
          </li>
        } @empty {
          <li>Keine Benutzer gefunden.</li>
        }
      </ul>

      <p>{{ filteredItems().length }} / {{ items().length }} Einträge</p>
    </div>
  `
})
export class UserListComponent extends AbstractListComponent<User> {
  private http = inject(HttpClient);

  override loadData(): Observable<User[]> {
    return this.http.get<User[]>('https://jsonplaceholder.typicode.com/users');
  }
}

// product-list.component.ts – mit erweitertem ngOnInit
interface Product { id: number; title: string; price: number; category: string; }

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    <div>
      <select (change)="searchQuery.set($any($event.target).value)">
        <option value="">Alle Kategorien</option>
        @for (cat of categories(); track cat) {
          <option [value]="cat">{{ cat }}</option>
        }
      </select>

      @for (product of filteredItems(); track product.id) {
        <div>{{ product.title }} – {{ product.price | currency }}</div>
      }
    </div>
  `
})
export class ProductListComponent extends AbstractListComponent<Product> {
  private http = inject(HttpClient);
  readonly categories = signal<string[]>([]);

  // Basislogik + eigene Initialisierung
  override ngOnInit(): void {
    super.ngOnInit(); // Pflicht: Basisklassen-Logik ausführen

    this.loadData().subscribe(products => {
      const cats = [...new Set(products.map(p => p.category))];
      this.categories.set(cats);
    });
  }

  override loadData(): Observable<Product[]> {
    return this.http.get<Product[]>('https://fakestoreapi.com/products');
  }
}
```

## Weiterführendes

- **Mixin-Pattern als Alternative:** Statt Vererbung können TypeScript-Mixins mehrere Verhaltensbausteine kombinieren – besonders nützlich, wenn Angular's Einfachvererbung limitiert (siehe: `type Constructor<T = {}> = new (...args: any[]) => T`)
- **Docs:** [Angular – Class inheritance](https://angular.dev/guide/components/inheritance)
- **Tipp:** Halte Basisklassen schlank – wenn die Basisklasse zu viel weiß, ist ein gemeinsamer Service oder eine Composition via `inject()` oft die bessere Wahl. Vererbung für strukturelle Wiederverwendung, Services für Logik-Wiederverwendung.

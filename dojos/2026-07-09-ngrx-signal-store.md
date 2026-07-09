# Angular Dojo: NgRx SignalStore
**Datum:** 2026-07-09
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, den modernen `@ngrx/signals`-basierten `signalStore()` zu verwenden, um lokalen und globalen Komponenten-State mit Signals zu verwalten – ohne RxJS-Overhead und mit deutlich weniger Boilerplate als klassisches NgRx.

## Hintergrund & Theorie

NgRx SignalStore wurde mit NgRx 17 eingeführt und ist der neue empfohlene Ansatz für State-Management in Angular-Signal-Anwendungen. Im Gegensatz zum klassischen `@ngrx/store` (mit Actions, Reducers, Effects) arbeitet SignalStore direkt mit Signals und ist deutlich schlanker.

**Kernkonzepte:**

- `signalStore()` – erzeugt einen Injectable Store als Klasse
- `withState(initialState)` – definiert den Zustand; jedes Property wird automatisch zu einem `Signal`
- `withGetters(store => ({...}))` – deklarative, memoized `computed()`-Signale (früher: `withComputed`)
- `withMethods((store) => ({...}))` – Methoden, die `patchState(store, ...)` aufrufen
- `withHooks({ onInit, onDestroy })` – Lifecycle-Hooks für Initialisierung und Cleanup
- `patchState(store, partial)` – immutables State-Update (wie `setState` in React)

**Vergleich zu klassischem NgRx:**
| Feature | NgRx Store | NgRx SignalStore |
|---|---|---|
| Reaktivität | Observable | Signal |
| Boilerplate | hoch (Action/Reducer/Effect) | minimal |
| Scope | global | lokal oder global |
| Async | Effects | `withMethods` + `inject(HttpClient)` |

```typescript
// Minimal-Beispiel
const CounterStore = signalStore(
  withState({ count: 0 }),
  withGetters(store => ({
    doubled: computed(() => store.count() * 2),
  })),
  withMethods(store => ({
    increment: () => patchState(store, s => ({ count: s.count + 1 })),
  })),
);
```

## Aufgabe

Baue einen `BookStore` mit NgRx SignalStore, der eine Bücherliste verwaltet. Der Store soll:

1. Eine Liste von Büchern (`Book[]`) im State halten
2. Einen `loading`-Flag und ein `filter`-String-Signal besitzen
3. Einen Getter `filteredBooks` anbieten, der die Bücher nach `filter` filtert (Titel, Autor)
4. Methoden `loadBooks()` (simulierter HTTP-Call via `delay`), `setFilter(term)` und `toggleFavorite(id)` bereitstellen
5. In einer `standalone` Komponente verwendet werden, die den Store per `inject()` bezieht

### Schritte

1. Definiere das `Book`-Interface mit `id`, `title`, `author`, `favorite: boolean`.
2. Erstelle `BookStore` mit `signalStore({ providedIn: 'root' })` und `withState()`, `withGetters()`, `withMethods()`.
3. Simuliere in `loadBooks()` einen asynchronen Aufruf via `Promise`/`setTimeout` oder `firstValueFrom(of([...]).pipe(delay(800)))`.
4. Implementiere `setFilter()` mit `patchState(store, { filter: term })`.
5. Implementiere `toggleFavorite(id)` – update das passende Buch immutabel im Array.
6. Erstelle eine `BookListComponent` (standalone, OnPush), die den Store injectet und Template-Signals bindet.
7. Zeige Ladeindikator, Filtereingabe und die gefilterte Bücherliste an.

## Hints

<details>
<summary>Hint 1 – Store-Grundstruktur</summary>

```typescript
import { signalStore, withState, withGetters, withMethods, patchState } from '@ngrx/signals';
import { computed, inject } from '@angular/core';

export interface Book {
  id: number;
  title: string;
  author: string;
  favorite: boolean;
}

interface BookState {
  books: Book[];
  loading: boolean;
  filter: string;
}

const initialState: BookState = {
  books: [],
  loading: false,
  filter: '',
};

export const BookStore = signalStore(
  { providedIn: 'root' },
  withState(initialState),
  withGetters(store => ({
    filteredBooks: computed(() => {
      const term = store.filter().toLowerCase();
      return store.books().filter(
        b => b.title.toLowerCase().includes(term) ||
             b.author.toLowerCase().includes(term)
      );
    }),
    favoriteCount: computed(() => store.books().filter(b => b.favorite).length),
  })),
  withMethods(store => ({
    setFilter(term: string): void {
      patchState(store, { filter: term });
    },
    // weitere Methoden ...
  })),
);
```

</details>

<details>
<summary>Hint 2 – Async loadBooks und toggleFavorite</summary>

```typescript
withMethods(store => ({
  async loadBooks(): Promise<void> {
    patchState(store, { loading: true });

    // Simulierter API-Call
    await new Promise(resolve => setTimeout(resolve, 800));
    const books: Book[] = [
      { id: 1, title: 'Clean Code', author: 'Robert C. Martin', favorite: false },
      { id: 2, title: 'The Pragmatic Programmer', author: 'Hunt & Thomas', favorite: false },
      { id: 3, title: 'Angular Development', author: 'Minko Gechev', favorite: true },
    ];

    patchState(store, { books, loading: false });
  },

  toggleFavorite(id: number): void {
    patchState(store, state => ({
      books: state.books.map(b =>
        b.id === id ? { ...b, favorite: !b.favorite } : b
      ),
    }));
  },

  setFilter(term: string): void {
    patchState(store, { filter: term });
  },
}))
```

</details>

<details>
<summary>Hint 3 – Komponente mit Store-Injection</summary>

```typescript
import { Component, inject, ChangeDetectionStrategy, OnInit } from '@angular/core';
import { BookStore } from './book.store';

@Component({
  selector: 'app-book-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input [value]="store.filter()"
           (input)="store.setFilter($any($event.target).value)"
           placeholder="Bücher filtern..." />

    <p>Favoriten: {{ store.favoriteCount() }}</p>

    @if (store.loading()) {
      <p>Lädt Bücher...</p>
    }

    <ul>
      @for (book of store.filteredBooks(); track book.id) {
        <li>
          <strong>{{ book.title }}</strong> – {{ book.author }}
          <button (click)="store.toggleFavorite(book.id)">
            {{ book.favorite ? '★' : '☆' }}
          </button>
        </li>
      }
    </ul>
  `,
})
export class BookListComponent implements OnInit {
  store = inject(BookStore);

  ngOnInit() {
    this.store.loadBooks();
  }
}
```

</details>

<details>
<summary>Hint 4 – withHooks für automatisches Laden</summary>

```typescript
// Statt ngOnInit in der Komponente: Initialisierung im Store selbst
withHooks({
  onInit(store) {
    // store.loadBooks() wird automatisch beim ersten Inject aufgerufen
    store.loadBooks();
  },
  onDestroy(store) {
    // Cleanup-Logik falls nötig
    console.log('BookStore destroyed');
  },
})
```

Mit `withHooks` wird der Store selbst verantwortlich für seine Initialisierung – die Komponente muss kein `ngOnInit` mehr implementieren.

</details>

## Beispiellösung

```typescript
// book.model.ts
export interface Book {
  id: number;
  title: string;
  author: string;
  favorite: boolean;
}

// book.store.ts
import {
  signalStore,
  withState,
  withGetters,
  withMethods,
  withHooks,
  patchState,
} from '@ngrx/signals';
import { computed } from '@angular/core';
import { Book } from './book.model';

interface BookState {
  books: Book[];
  loading: boolean;
  filter: string;
}

const MOCK_BOOKS: Book[] = [
  { id: 1, title: 'Clean Code',               author: 'Robert C. Martin', favorite: false },
  { id: 2, title: 'The Pragmatic Programmer', author: 'Hunt & Thomas',    favorite: false },
  { id: 3, title: 'Angular-Entwicklung',       author: 'Minko Gechev',    favorite: true  },
  { id: 4, title: 'RxJS in Action',            author: 'Daniels & Atencio', favorite: false },
];

export const BookStore = signalStore(
  { providedIn: 'root' },
  withState<BookState>({ books: [], loading: false, filter: '' }),

  withGetters(store => ({
    filteredBooks: computed(() => {
      const term = store.filter().toLowerCase();
      if (!term) return store.books();
      return store.books().filter(
        b => b.title.toLowerCase().includes(term) ||
             b.author.toLowerCase().includes(term)
      );
    }),
    favoriteCount: computed(() => store.books().filter(b => b.favorite).length),
    totalCount:    computed(() => store.books().length),
  })),

  withMethods(store => ({
    async loadBooks(): Promise<void> {
      patchState(store, { loading: true });
      await new Promise<void>(resolve => setTimeout(resolve, 800));
      patchState(store, { books: MOCK_BOOKS, loading: false });
    },

    setFilter(term: string): void {
      patchState(store, { filter: term });
    },

    toggleFavorite(id: number): void {
      patchState(store, state => ({
        books: state.books.map(b =>
          b.id === id ? { ...b, favorite: !b.favorite } : b
        ),
      }));
    },
  })),

  withHooks({
    onInit(store) {
      store.loadBooks();
    },
  }),
);

// book-list.component.ts
import { Component, inject, ChangeDetectionStrategy } from '@angular/core';
import { BookStore } from './book.store';

@Component({
  selector: 'app-book-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>Bücherliste</h1>

    <div class="controls">
      <input
        [value]="store.filter()"
        (input)="store.setFilter($any($event.target).value)"
        placeholder="Bücher filtern (Titel oder Autor)..."
      />
      <span class="meta">
        {{ store.filteredBooks().length }} / {{ store.totalCount() }} Bücher
        · {{ store.favoriteCount() }} Favoriten
      </span>
    </div>

    @if (store.loading()) {
      <p class="loading">⏳ Bücher werden geladen...</p>
    }

    <ul class="book-list">
      @for (book of store.filteredBooks(); track book.id) {
        <li [class.favorite]="book.favorite">
          <div class="book-info">
            <strong>{{ book.title }}</strong>
            <span class="author">{{ book.author }}</span>
          </div>
          <button
            class="fav-btn"
            (click)="store.toggleFavorite(book.id)"
            [attr.aria-label]="book.favorite ? 'Aus Favoriten entfernen' : 'Zu Favoriten hinzufügen'"
          >
            {{ book.favorite ? '★' : '☆' }}
          </button>
        </li>
      } @empty {
        <li class="empty">Keine Bücher gefunden.</li>
      }
    </ul>
  `,
  styles: [`
    .controls { display: flex; gap: 1rem; align-items: center; margin-bottom: 1rem; }
    input { padding: 0.4rem 0.6rem; flex: 1; border: 1px solid #ccc; border-radius: 4px; }
    .meta { font-size: 0.85rem; color: #666; white-space: nowrap; }
    .loading { color: #666; }
    .book-list { list-style: none; padding: 0; }
    .book-list li { display: flex; justify-content: space-between; align-items: center;
                    padding: 0.6rem 0.8rem; border-bottom: 1px solid #eee; }
    .book-list li.favorite { background: #fffbea; }
    .book-info { display: flex; flex-direction: column; }
    .author { font-size: 0.85rem; color: #666; }
    .fav-btn { background: none; border: none; font-size: 1.3rem; cursor: pointer; }
    .empty { color: #999; font-style: italic; }
  `],
})
export class BookListComponent {
  store = inject(BookStore);
}

// app.config.ts – keine zusätzliche Provider-Registrierung nötig,
// da BookStore mit { providedIn: 'root' } definiert ist.
```

**Bonus-Aufgabe:** Erweitere den Store mit `withEntities<Book>()` aus `@ngrx/signals/entities` und ersetze das manuelle Array-Management durch `addEntities()`, `updateEntity()` und `removeEntity()`.

## Weiterführendes

- **`withEntities()`** aus `@ngrx/signals/entities` – Entity-Adapter analog zu `@ngrx/entity`, reduziert CRUD-Boilerplate erheblich (`setAllEntities`, `addEntity`, `updateEntity`, `removeEntity`).
- **Custom Store Features** – wiederverwendbare Erweiterungen via `signalStoreFeature()`: z. B. ein `withLoadingState()`-Feature, das `loading`, `error` und `loadingMethods` für jeden Store bereitstellt.
- **Devtools-Integration** – `withDevtools('bookStore')` aus `@angular-architects/ngrx-toolkit` ermöglicht Redux-DevTools-Inspection auch für SignalStore.
- Offizielle Docs: [ngrx.io/guide/signals/signal-store](https://ngrx.io/guide/signals/signal-store)
- Vergleich SignalStore vs. ComponentStore vs. Store: [ngrx.io/guide/signals](https://ngrx.io/guide/signals)

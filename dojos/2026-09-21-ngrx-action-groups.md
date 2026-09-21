# Angular Dojo: NgRx createActionGroup – Aktionen strukturieren
**Datum:** 2026-09-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie `createActionGroup` aus NgRx 15+ Actions logisch gruppiert, Tipparbeit reduziert und durch automatisches Präfixing sowie strikte Typisierung die gesamte Store-Architektur sauberer macht.

## Hintergrund & Theorie

Klassisch in NgRx wurden Actions einzeln mit `createAction` und `props<>()` definiert. Bei wachsenden Features führt das zu vielen lose zusammenhängenden `const`-Exporten, die manuell gepflegt werden müssen. Seit **NgRx 15** löst `createActionGroup` dieses Problem:

```typescript
// Alt: einzelne createAction-Aufrufe
export const loadProducts = createAction('[Products] Load Products');
export const loadProductsSuccess = createAction('[Products] Load Products Success', props<{ products: Product[] }>());
export const loadProductsFailure = createAction('[Products] Load Products Failure', props<{ error: string }>());
```

`createActionGroup` bündelt alle verwandten Actions unter einem gemeinsamen **Source**-Prefix und gibt ein typisiertes Objekt zurück:

```typescript
export const ProductsActions = createActionGroup({
  source: 'Products',
  events: {
    'Load Products': emptyProps(),
    'Load Products Success': props<{ products: Product[] }>(),
    'Load Products Failure': props<{ error: string }>(),
  },
});
```

Die Keys werden automatisch in **camelCase** umgewandelt (`ProductsActions.loadProducts`, `ProductsActions.loadProductsSuccess`). Das Event-Type-String lautet `[Products] Load Products`. Reducer, Effects und Selektoren profitieren von der zentralen Quelle und vollständiger IDE-Autovervollständigung. Auch das Dispatchen aus Komponenten wird übersichtlicher, da nur noch ein Import nötig ist.

## Aufgabe

Refactore ein bestehendes Feature mit klassischen `createAction`-Definitionen auf `createActionGroup`. Implementiere anschließend einen Reducer und einen Effect, der die gruppierten Actions verwendet.

**Kontext:** Du hast eine `TodoList`-App. Bisher sind die Actions verstreut definiert. Deine Aufgabe ist es, sie mit `createActionGroup` zu konsolidieren und Reducer sowie Effect anzupassen.

### Schritte

1. **Action Group erstellen** – Definiere `TodosActions` mit `createActionGroup` für folgende Events:
   - `'Load Todos'` (kein Props)
   - `'Load Todos Success'` mit `{ todos: Todo[] }`
   - `'Load Todos Failure'` mit `{ error: string }`
   - `'Add Todo'` mit `{ title: string }`
   - `'Toggle Todo'` mit `{ id: string }`
   - `'Delete Todo'` mit `{ id: string }`

2. **Reducer anpassen** – Schreibe einen Reducer mit `createReducer` und `on()`, der alle Actions aus `TodosActions` verarbeitet. Nutze dabei `TodosActions.loadTodosSuccess` usw. (camelCase-Keys).

3. **Effect erstellen** – Schreibe einen Effect `loadTodos$`, der auf `TodosActions.loadTodos` reagiert, einen `TodoService.getAll()` (gibt `Observable<Todo[]>` zurück) aufruft und bei Erfolg `TodosActions.loadTodosSuccess` bzw. bei Fehler `TodosActions.loadTodosFailure` dispatcht.

4. **Komponente verwenden** – Dispatch `TodosActions.addTodo({ title: 'Dojo abschließen' })` und `TodosActions.toggleTodo({ id: '1' })` in einer Komponente. Beobachte in den DevTools, wie der Event-Type-String automatisch gesetzt ist.

5. **Bonus:** Erstelle eine zweite Action Group `TodosApiActions` (Source: `'Todos API'`) für serverseitige Ereignisse und demonstriere, wie zwei Groups koexistieren können.

## Hints

<details>
<summary>Hint 1 – Grundstruktur von createActionGroup</summary>

```typescript
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

export const TodosActions = createActionGroup({
  source: 'Todos',
  events: {
    'Load Todos': emptyProps(),
    'Load Todos Success': props<{ todos: Todo[] }>(),
    'Load Todos Failure': props<{ error: string }>(),
    'Add Todo': props<{ title: string }>(),
    'Toggle Todo': props<{ id: string }>(),
    'Delete Todo': props<{ id: string }>(),
  },
});
// Zugriff: TodosActions.loadTodos, TodosActions.addTodo, ...
```

</details>

<details>
<summary>Hint 2 – Reducer mit Action Group</summary>

```typescript
import { createReducer, on } from '@ngrx/store';
import { TodosActions, Todo } from './todos.actions';

export interface TodosState {
  todos: Todo[];
  loading: boolean;
  error: string | null;
}

const initialState: TodosState = {
  todos: [],
  loading: false,
  error: null,
};

export const todosReducer = createReducer(
  initialState,
  on(TodosActions.loadTodos, (state) => ({ ...state, loading: true, error: null })),
  on(TodosActions.loadTodosSuccess, (state, { todos }) => ({
    ...state,
    loading: false,
    todos,
  })),
  on(TodosActions.loadTodosFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),
  on(TodosActions.addTodo, (state, { title }) => ({
    ...state,
    todos: [...state.todos, { id: crypto.randomUUID(), title, completed: false }],
  })),
  on(TodosActions.toggleTodo, (state, { id }) => ({
    ...state,
    todos: state.todos.map((t) =>
      t.id === id ? { ...t, completed: !t.completed } : t
    ),
  })),
  on(TodosActions.deleteTodo, (state, { id }) => ({
    ...state,
    todos: state.todos.filter((t) => t.id !== id),
  }))
);
```

</details>

<details>
<summary>Hint 3 – Effect mit Action Group</summary>

```typescript
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { TodosActions } from './todos.actions';
import { TodoService } from './todo.service';

export const loadTodos$ = createEffect(
  (actions$ = inject(Actions), todoService = inject(TodoService)) =>
    actions$.pipe(
      ofType(TodosActions.loadTodos),
      switchMap(() =>
        todoService.getAll().pipe(
          map((todos) => TodosActions.loadTodosSuccess({ todos })),
          catchError((error: Error) =>
            of(TodosActions.loadTodosFailure({ error: error.message }))
          )
        )
      )
    ),
  { functional: true }
);
```

</details>

## Beispiellösung

```typescript
// todos.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

export const TodosActions = createActionGroup({
  source: 'Todos',
  events: {
    'Load Todos': emptyProps(),
    'Load Todos Success': props<{ todos: Todo[] }>(),
    'Load Todos Failure': props<{ error: string }>(),
    'Add Todo': props<{ title: string }>(),
    'Toggle Todo': props<{ id: string }>(),
    'Delete Todo': props<{ id: string }>(),
  },
});

// Bonus: separate Action Group für API-Events
export const TodosApiActions = createActionGroup({
  source: 'Todos API',
  events: {
    'Todo Created': props<{ todo: Todo }>(),
    'Todo Updated': props<{ todo: Todo }>(),
    'Todo Deleted': props<{ id: string }>(),
  },
});

// todos.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { TodosActions, TodosApiActions, Todo } from './todos.actions';

export interface TodosState {
  todos: Todo[];
  loading: boolean;
  error: string | null;
}

const initialState: TodosState = { todos: [], loading: false, error: null };

export const todosReducer = createReducer(
  initialState,
  on(TodosActions.loadTodos, (state) => ({ ...state, loading: true, error: null })),
  on(TodosActions.loadTodosSuccess, (state, { todos }) => ({ ...state, loading: false, todos })),
  on(TodosActions.loadTodosFailure, (state, { error }) => ({ ...state, loading: false, error })),
  on(TodosActions.addTodo, (state, { title }) => ({
    ...state,
    todos: [...state.todos, { id: crypto.randomUUID(), title, completed: false }],
  })),
  on(TodosActions.toggleTodo, (state, { id }) => ({
    ...state,
    todos: state.todos.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t)),
  })),
  on(TodosActions.deleteTodo, (state, { id }) => ({
    ...state,
    todos: state.todos.filter((t) => t.id !== id),
  })),
  // API Actions aus externer Quelle ebenfalls im selben Reducer verarbeiten
  on(TodosApiActions.todoCreated, (state, { todo }) => ({
    ...state,
    todos: [...state.todos, todo],
  }))
);

// todos.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { TodosActions } from './todos.actions';
import { TodoService } from './todo.service';

export const loadTodos$ = createEffect(
  (actions$ = inject(Actions), todoService = inject(TodoService)) =>
    actions$.pipe(
      ofType(TodosActions.loadTodos),
      switchMap(() =>
        todoService.getAll().pipe(
          map((todos) => TodosActions.loadTodosSuccess({ todos })),
          catchError((error: Error) =>
            of(TodosActions.loadTodosFailure({ error: error.message }))
          )
        )
      )
    ),
  { functional: true }
);

// todos.component.ts (Auszug)
@Component({
  template: `
    <button (click)="load()">Laden</button>
    <button (click)="add()">Hinzufügen</button>
    @for (todo of todos(); track todo.id) {
      <div (click)="toggle(todo.id)">{{ todo.title }}</div>
    }
  `,
})
export class TodosComponent {
  private store = inject(Store);
  todos = this.store.selectSignal(selectAllTodos);

  load() { this.store.dispatch(TodosActions.loadTodos()); }
  add() { this.store.dispatch(TodosActions.addTodo({ title: 'Neues Todo' })); }
  toggle(id: string) { this.store.dispatch(TodosActions.toggleTodo({ id })); }
}
```

## Weiterführendes

- **NgRx Docs – Action Groups:** https://ngrx.io/guide/store/action-groups – offizielle Dokumentation mit allen Details zu Key-Transformation und Typsystem
- **Tipp:** Kombiniere `createActionGroup` mit NgRx Schematics (`ng generate @ngrx/schematics:feature`) – die generierten Dateien nutzen bereits `createActionGroup` als Standard
- **Achtung:** Keys in `events` müssen eindeutige Strings sein; Sonderzeichen außer Leerzeichen und Bindestrichen können das camelCase-Mapping beeinflussen – halte dich an einfache Substantiv-Verb-Kombinationen

# Angular Dojo: NgRx Entity – Collection-Management leicht gemacht
**Datum:** 2026-08-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie `@ngrx/entity` wiederholenden CRUD-Boilerplate für Collections eliminiert, indem du einen `EntityAdapter` für eine `Todo`-Liste aufbaust und typesichere Selectors sowie Reducer mit den eingebauten Hilfsmethoden schreibst.

## Hintergrund & Theorie
In echten Anwendungen verwaltet man häufig Listen von Entitäten (Users, Todos, Products). Ohne `@ngrx/entity` baut man manuell Reducer mit array-basierten Operationen (`map`, `filter`, `findIndex`) – das ist fehleranfällig und repetitiv.

`@ngrx/entity` bietet einen `EntityAdapter<T>`, der eine normalisierte Datenstruktur (`EntityState<T>`) verwendet: ein Dictionary (`entities`) und ein geordnetes Array von IDs (`ids`). Dadurch sind Lookups O(1) statt O(n).

Der Adapter stellt fertige Reducer-Operationen bereit:
- `adapter.addOne(entity, state)` – fügt eine Entität hinzu
- `adapter.upsertOne(entity, state)` – fügt hinzu oder aktualisiert
- `adapter.updateOne({ id, changes }, state)` – partielles Update
- `adapter.removeOne(id, state)` – entfernt eine Entität
- `adapter.setAll(entities, state)` – ersetzt den gesamten Inhalt

Mit `adapter.getSelectors()` erhält man sofort nutzbare Selectors (`selectAll`, `selectEntities`, `selectIds`, `selectTotal`), die man an den Feature-State-Selector koppelt.

## Aufgabe
Baue ein vollständiges NgRx-Entity-Feature für eine `Todo`-Verwaltung:
1. Definiere das `Todo`-Interface und erstelle einen `EntityAdapter`
2. Implementiere `TodoState` mit `EntityState<Todo>` sowie einem `loading`-Flag
3. Schreibe Actions für Laden, Hinzufügen, Abhaken und Löschen von Todos
4. Implementiere den Reducer ausschließlich mit Adapter-Methoden
5. Erstelle Feature-Selectors (inkl. einem berechneten Selector für offene Todos)
6. Binde alles in eine `TodosComponent` ein

### Schritte
1. Installiere `@ngrx/entity` (`npm install @ngrx/entity`) und erstelle `src/app/todos/`.
2. Definiere `Todo` (`id: string`, `title: string`, `done: boolean`) und erstelle den Adapter mit `createEntityAdapter<Todo>()`.
3. Definiere `TodoState` als `EntityState<Todo> & { loading: boolean }` und erzeuge den Initial-State mit `adapter.getInitialState({ loading: false })`.
4. Schreibe die Actions: `loadTodos`, `loadTodosSuccess`, `addTodo`, `toggleTodo`, `deleteTodo`.
5. Implementiere den Reducer mit `createReducer` und `on`-Handlern; nutze ausschließlich Adapter-Methoden.
6. Erstelle Feature-Selectors mit `createFeatureSelector` und leite `selectAll`, `selectTotal` sowie `selectOpenTodos` (gefiltert) ab.
7. Erstelle `TodosComponent` als Standalone-Komponente, die die Selectors per `store.select` abonniert und die Actions dispatcht.

## Hints
<details>
<summary>Hint 1 – Adapter und InitialState</summary>

```typescript
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';

export interface Todo {
  id: string;
  title: string;
  done: boolean;
}

export const adapter: EntityAdapter<Todo> = createEntityAdapter<Todo>();

export interface TodoState extends EntityState<Todo> {
  loading: boolean;
}

export const initialState: TodoState = adapter.getInitialState({ loading: false });
```
</details>

<details>
<summary>Hint 2 – Selectors ableiten</summary>

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';

export const selectTodoState = createFeatureSelector<TodoState>('todos');

const { selectAll, selectTotal } = adapter.getSelectors();

export const selectAllTodos  = createSelector(selectTodoState, selectAll);
export const selectTotalTodos = createSelector(selectTodoState, selectTotal);
export const selectOpenTodos = createSelector(
  selectAllTodos,
  todos => todos.filter(t => !t.done)
);
```
</details>

## Beispiellösung

```typescript
// todo.model.ts
export interface Todo {
  id: string;
  title: string;
  done: boolean;
}

// todo.adapter.ts
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';
import { Todo } from './todo.model';

export const adapter: EntityAdapter<Todo> = createEntityAdapter<Todo>();

export interface TodoState extends EntityState<Todo> {
  loading: boolean;
}

export const initialState: TodoState = adapter.getInitialState({ loading: false });

// todo.actions.ts
import { createAction, props } from '@ngrx/store';
import { Todo } from './todo.model';

export const loadTodos        = createAction('[Todos] Load');
export const loadTodosSuccess = createAction('[Todos] Load Success', props<{ todos: Todo[] }>());
export const addTodo          = createAction('[Todos] Add',    props<{ todo: Todo }>());
export const toggleTodo       = createAction('[Todos] Toggle', props<{ id: string }>());
export const deleteTodo       = createAction('[Todos] Delete', props<{ id: string }>());

// todo.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { adapter, initialState } from './todo.adapter';
import * as TodoActions from './todo.actions';

export const todoReducer = createReducer(
  initialState,
  on(TodoActions.loadTodos,        state => ({ ...state, loading: true })),
  on(TodoActions.loadTodosSuccess, (state, { todos }) =>
    adapter.setAll(todos, { ...state, loading: false })
  ),
  on(TodoActions.addTodo,    (state, { todo })  => adapter.addOne(todo, state)),
  on(TodoActions.toggleTodo, (state, { id }) => {
    const current = state.entities[id];
    if (!current) return state;
    return adapter.updateOne({ id, changes: { done: !current.done } }, state);
  }),
  on(TodoActions.deleteTodo, (state, { id }) => adapter.removeOne(id, state)),
);

// todo.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { adapter, TodoState } from './todo.adapter';

export const selectTodoState   = createFeatureSelector<TodoState>('todos');
const { selectAll, selectTotal } = adapter.getSelectors();
export const selectAllTodos    = createSelector(selectTodoState, selectAll);
export const selectTotalTodos  = createSelector(selectTodoState, selectTotal);
export const selectOpenTodos   = createSelector(selectAllTodos, ts => ts.filter(t => !t.done));
export const selectLoading     = createSelector(selectTodoState, s => s.loading);

// todos.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { Store } from '@ngrx/store';
import { v4 as uuid } from 'uuid';
import * as TodoActions from './todo.actions';
import { selectAllTodos, selectOpenTodos, selectTotalTodos } from './todo.selectors';

@Component({
  selector: 'app-todos',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Todos ({{ total$ | async }} gesamt, {{ open$ | async | json | length }} offen)</h2>
    <button (click)="add()">+ Hinzufügen</button>
    <ul>
      @for (todo of todos$ | async; track todo.id) {
        <li [style.textDecoration]="todo.done ? 'line-through' : 'none'">
          <input type="checkbox" [checked]="todo.done" (change)="toggle(todo.id)" />
          {{ todo.title }}
          <button (click)="remove(todo.id)">✕</button>
        </li>
      }
    </ul>
  `,
})
export class TodosComponent {
  private store = inject(Store);
  todos$ = this.store.select(selectAllTodos);
  open$  = this.store.select(selectOpenTodos);
  total$ = this.store.select(selectTotalTodos);

  add(): void {
    const title = prompt('Titel?');
    if (title) {
      this.store.dispatch(TodoActions.addTodo({ todo: { id: uuid(), title, done: false } }));
    }
  }

  toggle(id: string): void { this.store.dispatch(TodoActions.toggleTodo({ id })); }
  remove(id: string): void { this.store.dispatch(TodoActions.deleteTodo({ id })); }
}

// app.config.ts (Auszug)
import { provideStore } from '@ngrx/store';
import { todoReducer }  from './todos/todo.reducer';

export const appConfig = {
  providers: [
    provideStore({ todos: todoReducer }),
  ],
};
```

## Weiterführendes
- **`sortComparer`**: Mit `createEntityAdapter<Todo>({ sortComparer: (a, b) => a.title.localeCompare(b.title) })` wird `selectAll` automatisch sortiert – kein manueller `sort`-Aufruf im Template nötig.
- Kombiniere `@ngrx/entity` mit Effects: Lade Todos via HTTP im Effect und dispatche `loadTodosSuccess` – so bleibt der Reducer rein synchron.
- Offizielle Doku: [NgRx Entity Guide](https://ngrx.io/guide/entity)

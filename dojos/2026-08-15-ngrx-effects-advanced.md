# Angular Dojo: NgRx Effects – Advanced Patterns
**Datum:** 2026-08-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, NgRx Effects professionell einzusetzen: mit `concatLatestFrom` auf Store-State zugreifen, Fehler robust behandeln, optimistische Updates implementieren und nicht-dispatching Side Effects korrekt schreiben.

## Hintergrund & Theorie

NgRx Effects sind der Ort für Side Effects – alles, was außerhalb des reinen State-Managements passiert: HTTP-Calls, Router-Navigation, localStorage, Analytics. Ein schlecht aufgesetzter Effect kann Race Conditions, unbehandelte Fehler oder State-Inkonsistenzen verursachen.

**Wichtige Konzepte:**

- **`concatLatestFrom`** (aus `@ngrx/operators`): Liest den aktuellen Store-State type-safe in einen Effect ein, ohne eine zweite `withLatestFrom`-Subscription aufzumachen. Er führt den inneren Observable erst aus, wenn der Store-Wert vorliegt.

- **Fehlerbehandlung im Effect**: `catchError` muss *innerhalb* des inneren Observables (z. B. `switchMap`) stehen – nicht außen –, sonst wird der Effect bei einem Fehler beendet und reagiert nicht mehr auf neue Actions.

- **Optimistisches Update**: UI wird sofort aktualisiert; bei einem Server-Fehler wird die Änderung zurückgerollt (`rollbackAction` dispatchen).

- **Non-dispatching Effects**: Mit `{ dispatch: false }` können Side Effects wie Logging, Navigation oder localStorage-Writes umgesetzt werden, ohne eine neue Action zu dispatchen.

- **`tapResponse`** (aus `@ngrx/operators`): Typsicheres Pendant zu `tap` + `catchError` in einem – vereinfacht die Fehlerbehandlung in HTTP-Effects erheblich.

## Aufgabe

Du hast eine kleine Todo-App mit NgRx. Implementiere drei Effects:

1. **`loadTodos$`**: Lädt Todos vom Backend. Fehler landen in `loadTodosFailure`. Nutze `tapResponse`.
2. **`toggleTodoOptimistic$`**: Implementiert ein optimistisches Update – dispatche sofort `toggleTodoSuccess`, und bei HTTP-Fehler `toggleTodoRollback` mit dem Original-Todo.
3. **`logActions$`** (non-dispatching): Loggt jede `loadTodosSuccess`-Action mit dem aktuellen Username aus dem Store in die Console.

### Schritte

1. Erstelle `todo.actions.ts` mit den nötigen Actions (load, success, failure, toggle, rollback).
2. Erstelle `todo.effects.ts` und implementiere `loadTodos$` mit `tapResponse`.
3. Implementiere `toggleTodoOptimistic$`: Speichere das Original-Todo vor dem HTTP-Call und dispatche bei Fehler `toggleTodoRollback`.
4. Implementiere `logActions$` mit `{ dispatch: false }` und `concatLatestFrom` für den Username aus dem Store.
5. Teste die Fehlerbehandlung, indem du den HTTP-Call absichtlich fehlschlagen lässt (z. B. via `throwError`).

## Hints

<details>
<summary>Hint 1 – tapResponse</summary>

`tapResponse` erwartet zwei Callbacks: `onSuccess` und `onError`. Es wrappt automatisch `catchError` und stellt sicher, dass der Effect-Stream nicht abbricht:

```typescript
import { tapResponse } from '@ngrx/operators';

loadTodos$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TodoActions.loadTodos),
    switchMap(() =>
      this.todoService.getAll().pipe(
        tapResponse({
          next: (todos) => TodoActions.loadTodosSuccess({ todos }),
          error: (error: HttpErrorResponse) => TodoActions.loadTodosFailure({ error: error.message }),
        })
      )
    )
  )
);
```

</details>

<details>
<summary>Hint 2 – concatLatestFrom & optimistic update</summary>

Für `concatLatestFrom` den Store-Selector übergeben:

```typescript
import { concatLatestFrom } from '@ngrx/operators';

logActions$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TodoActions.loadTodosSuccess),
    concatLatestFrom(() => this.store.select(selectUsername)),
    tap(([action, username]) => console.log(`[${username}] Todos geladen:`, action.todos))
  ),
  { dispatch: false }
);
```

Für das optimistische Update: Das Original-Todo *vor* dem `switchMap` aus der Action holen und im `catchError` für den Rollback nutzen:

```typescript
toggleTodoOptimistic$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TodoActions.toggleTodo),
    switchMap(({ todo }) =>
      this.todoService.toggle(todo.id).pipe(
        map((updated) => TodoActions.toggleTodoSuccess({ todo: updated })),
        catchError(() => of(TodoActions.toggleTodoRollback({ original: todo })))
      )
    )
  )
);
```

</details>

## Beispiellösung

```typescript
// todo.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';
import { Todo } from './todo.model';

export const TodoActions = createActionGroup({
  source: 'Todo',
  events: {
    'Load Todos': emptyProps(),
    'Load Todos Success': props<{ todos: Todo[] }>(),
    'Load Todos Failure': props<{ error: string }>(),
    'Toggle Todo': props<{ todo: Todo }>(),
    'Toggle Todo Success': props<{ todo: Todo }>(),
    'Toggle Todo Rollback': props<{ original: Todo }>(),
  },
});
```

```typescript
// todo.effects.ts
import { inject, Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { Store } from '@ngrx/store';
import { concatLatestFrom, tapResponse } from '@ngrx/operators';
import { switchMap, map, catchError, tap } from 'rxjs/operators';
import { of } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';
import { TodoActions } from './todo.actions';
import { TodoService } from './todo.service';
import { selectUsername } from '../auth/auth.selectors';

@Injectable()
export class TodoEffects {
  private readonly actions$ = inject(Actions);
  private readonly store = inject(Store);
  private readonly todoService = inject(TodoService);

  // Effect 1: Laden mit tapResponse (Fehler bricht Stream nicht ab)
  loadTodos$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TodoActions.loadTodos),
      switchMap(() =>
        this.todoService.getAll().pipe(
          tapResponse({
            next: (todos) => TodoActions.loadTodosSuccess({ todos }),
            error: (err: HttpErrorResponse) =>
              TodoActions.loadTodosFailure({ error: err.message }),
          })
        )
      )
    )
  );

  // Effect 2: Optimistisches Update mit Rollback
  toggleTodoOptimistic$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TodoActions.toggleTodo),
      switchMap(({ todo }) =>
        this.todoService.toggle(todo.id).pipe(
          map((updated) => TodoActions.toggleTodoSuccess({ todo: updated })),
          catchError(() => of(TodoActions.toggleTodoRollback({ original: todo })))
        )
      )
    )
  );

  // Effect 3: Non-dispatching Logging mit Store-State
  logActions$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(TodoActions.loadTodosSuccess),
        concatLatestFrom(() => this.store.select(selectUsername)),
        tap(([action, username]) =>
          console.log(`[${username}] ${action.todos.length} Todos geladen`)
        )
      ),
    { dispatch: false }
  );
}
```

```typescript
// im Reducer: Rollback zurücksetzen
on(TodoActions.toggleTodoRollback, (state, { original }) => ({
  ...state,
  todos: state.todos.map((t) => (t.id === original.id ? original : t)),
})),
```

## Weiterführendes

- **`@ngrx/operators` Doku**: `tapResponse`, `concatLatestFrom` und `mapResponse` sind der moderne Ersatz für manuelle `catchError`/`withLatestFrom`-Kombinationen – unbedingt lesen.
- **Long-running Effects**: Für Polling oder WebSocket-Streams statt `switchMap` besser `exhaustMap` oder ein manuell gemanagter Subject-Stream verwenden, um Race Conditions zu vermeiden.
- **Effect Isolation Testing**: `provideMockActions` + `cold()`/`hot()` aus `jasmine-marbles` erlaubt präzises Marble-Testing von Effects ohne echten HTTP-Call.

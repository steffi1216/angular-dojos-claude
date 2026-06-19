# Angular Dojo: NgRx Store
**Datum:** 2026-06-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man mit NgRx Store einen zentralen Zustand verwaltet – von der Definition von Actions und Reducers über typsichere Selectors bis hin zu Effects für asynchrone Operationen.

## Hintergrund & Theorie

NgRx ist eine reaktive State-Management-Bibliothek für Angular, inspiriert von Redux. Der Datenfluss ist strikt unidirektional:

1. **Action**: Ein einfaches Objekt mit `type` (und optionalen Nutzdaten), das ein Ereignis beschreibt.
2. **Reducer**: Eine reine Funktion `(state, action) => newState`, die nie den alten State mutiert.
3. **Store**: Der zentrale, unveränderliche Zustandsspeicher. Komponenten abonnieren ihn via `store.select(selector)`.
4. **Selector**: Eine Funktion, die einen Teil des States extrahiert und automatisch memoized (via `createSelector`).
5. **Effect**: Ein RxJS-Stream, der auf Actions lauscht, Seiteneffekte auslöst (z. B. HTTP-Calls) und neue Actions dispatcht.

Seit NgRx 15+ empfehlen sich die `createAction`/`createReducer`/`createFeature`-Helfer und `createActionGroup` für zusammenhängende Actions. `createEffect` kombiniert sich sauber mit dem `inject`-Pattern (keine Klassen-Dependency-Injection nötig).

Der Vorteil gegenüber einfachem Service-State: vollständige Nachvollziehbarkeit aller Zustandsänderungen (DevTools Time-Travel), klare Trennung von Logik und Darstellung.

## Aufgabe

Baue ein Mini-Feature „Task Manager" mit NgRx Store. Das Feature verwaltet eine Liste von Tasks (laden, hinzufügen, als erledigt markieren). Verwende `createFeature`, `createActionGroup` und einen `Effect` für das simulierte Laden der Tasks.

### Schritte

1. **Actions definieren** – Erstelle eine `tasks.actions.ts` mit `createActionGroup` für `loadTasks`, `loadTasksSuccess`, `loadTasksFailure`, `addTask` und `toggleTask`.

2. **State & Reducer** – Erstelle `tasks.reducer.ts` mit `createFeature`. Der State enthält `tasks: Task[]`, `loading: boolean` und `error: string | null`.

3. **Selectors** – Leite via `createSelector` einen Selector `selectPendingTasks` ab, der nur unerledigte Tasks zurückgibt.

4. **Effect** – Erstelle `tasks.effects.ts` mit `createEffect` und `inject(TasksService)`. Beim `loadTasks`-Action soll der Service aufgerufen werden; bei Erfolg wird `loadTasksSuccess`, bei Fehler `loadTasksFailure` dispatcht.

5. **Komponente** – Dispatch `loadTasks` im `ngOnInit` und zeige die Tasks über `store.select(selectAllTasks)` an. Füge Buttons hinzu, um einen neuen Task hinzuzufügen (`addTask`) und Tasks umzuschalten (`toggleTask`).

## Hints

<details>
<summary>Hint 1 – createActionGroup Syntax</summary>

```typescript
// tasks.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';
import { Task } from './task.model';

export const TasksActions = createActionGroup({
  source: 'Tasks',
  events: {
    'Load Tasks':       emptyProps(),
    'Load Tasks Success': props<{ tasks: Task[] }>(),
    'Load Tasks Failure': props<{ error: string }>(),
    'Add Task':         props<{ title: string }>(),
    'Toggle Task':      props<{ id: string }>(),
  },
});
// Zugriff: TasksActions.loadTasks(), TasksActions.addTask({ title: '...' })
```

</details>

<details>
<summary>Hint 2 – createFeature mit on()-Reducer</summary>

```typescript
// tasks.reducer.ts
import { createFeature, createReducer, on } from '@ngrx/store';
import { TasksActions } from './tasks.actions';
import { Task } from './task.model';

export interface TasksState {
  tasks: Task[];
  loading: boolean;
  error: string | null;
}

const initialState: TasksState = { tasks: [], loading: false, error: null };

export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(
    initialState,
    on(TasksActions.loadTasks, state => ({ ...state, loading: true, error: null })),
    on(TasksActions.loadTasksSuccess, (state, { tasks }) => ({ ...state, tasks, loading: false })),
    on(TasksActions.loadTasksFailure, (state, { error }) => ({ ...state, error, loading: false })),
    on(TasksActions.addTask, (state, { title }) => ({
      ...state,
      tasks: [...state.tasks, { id: crypto.randomUUID(), title, done: false }],
    })),
    on(TasksActions.toggleTask, (state, { id }) => ({
      ...state,
      tasks: state.tasks.map(t => t.id === id ? { ...t, done: !t.done } : t),
    })),
  ),
});

// Exportiert automatisch: selectTasksState, selectTasks, selectLoading, selectError
export const { selectTasks, selectLoading, selectError } = tasksFeature;
```

</details>

<details>
<summary>Hint 3 – Effect mit inject()</summary>

```typescript
// tasks.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, of, switchMap } from 'rxjs';
import { TasksActions } from './tasks.actions';
import { TasksService } from './tasks.service';

export const loadTasksEffect = createEffect(
  (actions$ = inject(Actions), tasksService = inject(TasksService)) =>
    actions$.pipe(
      ofType(TasksActions.loadTasks),
      switchMap(() =>
        tasksService.getAll().pipe(
          map(tasks => TasksActions.loadTasksSuccess({ tasks })),
          catchError(err => of(TasksActions.loadTasksFailure({ error: err.message }))),
        ),
      ),
    ),
  { functional: true },
);
```

</details>

<details>
<summary>Hint 4 – Selector für offene Tasks & App-Setup</summary>

```typescript
// tasks.selectors.ts
import { createSelector } from '@ngrx/store';
import { selectTasks } from './tasks.reducer';

export const selectPendingTasks = createSelector(
  selectTasks,
  tasks => tasks.filter(t => !t.done)
);
```

```typescript
// app.config.ts (Standalone)
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { tasksFeature } from './tasks/tasks.reducer';
import * as tasksEffects from './tasks/tasks.effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideState(tasksFeature),
    provideEffects(tasksEffects),
  ],
};
```

</details>

## Beispiellösung

```typescript
// task.model.ts
export interface Task {
  id: string;
  title: string;
  done: boolean;
}

// tasks.service.ts
import { Injectable } from '@angular/core';
import { delay, of } from 'rxjs';
import { Task } from './task.model';

@Injectable({ providedIn: 'root' })
export class TasksService {
  getAll() {
    const mockTasks: Task[] = [
      { id: '1', title: 'NgRx lernen', done: false },
      { id: '2', title: 'Effects verstehen', done: false },
    ];
    return of(mockTasks).pipe(delay(500));
  }
}

// task-list.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';
import { TasksActions } from './tasks.actions';
import { selectTasks, selectLoading, selectError } from './tasks.reducer';
import { selectPendingTasks } from './tasks.selectors';

@Component({
  selector: 'app-task-list',
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf],
  template: `
    <h2>Tasks</h2>

    <ng-container *ngIf="loading$ | async">Lädt...</ng-container>
    <p *ngIf="error$ | async as error" style="color:red">{{ error }}</p>

    <ul>
      <li *ngFor="let task of tasks$ | async">
        <label>
          <input type="checkbox" [checked]="task.done"
                 (change)="toggle(task.id)" />
          <span [style.textDecoration]="task.done ? 'line-through' : 'none'">
            {{ task.title }}
          </span>
        </label>
      </li>
    </ul>

    <p>Offen: {{ (pending$ | async)?.length }}</p>

    <button (click)="addTask()">+ Task hinzufügen</button>
  `,
})
export class TaskListComponent implements OnInit {
  private store = inject(Store);

  tasks$   = this.store.select(selectTasks);
  loading$ = this.store.select(selectLoading);
  error$   = this.store.select(selectError);
  pending$ = this.store.select(selectPendingTasks);

  ngOnInit() {
    this.store.dispatch(TasksActions.loadTasks());
  }

  toggle(id: string) {
    this.store.dispatch(TasksActions.toggleTask({ id }));
  }

  addTask() {
    const title = prompt('Task-Titel?');
    if (title?.trim()) {
      this.store.dispatch(TasksActions.addTask({ title: title.trim() }));
    }
  }
}
```

## Weiterführendes

- **NgRx Entity**: `@ngrx/entity` mit `createEntityAdapter` drastisch reduziert Boilerplate für CRUD-State (ids + entities Dictionary statt Array).
- **NgRx ComponentStore**: Für lokalen Komponenten-State ohne globalen Store – ideal wenn ein Feature seinen State nicht mit anderen teilen muss.
- **NgRx DevTools**: Browser-Extension für Time-Travel-Debugging; mit `provideStoreDevtools()` in der `app.config.ts` aktivieren.
- Offizielle Doku: https://ngrx.io/guide/store

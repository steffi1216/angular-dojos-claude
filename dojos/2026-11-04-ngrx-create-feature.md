# Angular Dojo: NgRx `createFeature` – Automatische Selektoren & Feature State
**Datum:** 2026-11-04
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie `createFeature()` aus `@ngrx/store` einen vollständigen Feature-State-Slice inklusive automatisch generierter Selektoren erstellt – und wie das den Boilerplate gegenüber manuellem `createSelector` drastisch reduziert.

## Hintergrund & Theorie

Seit NgRx 15 bietet `createFeature()` eine kompakte API, um einen Zustandsbereich zu definieren. Es nimmt einen `name` (Feature-Key) und einen `reducer` und liefert ein Objekt zurück, das folgendes enthält:

- `name` – den Feature-Key
- `reducer` – den fertigen Reducer für `StoreModule` bzw. `provideState()`
- `select<Key>` – für jeden Schlüssel im State-Interface wird automatisch ein typsicherer Selektor generiert (z.B. `selectLoading`, `selectItems`)

Das bedeutet: Kein manuelles `createFeatureSelector()` + `createSelector()` mehr für einfache Property-Selektoren. `createFeature` generiert diese on the fly aus dem State-Shape.

```typescript
export const counterFeature = createFeature({
  name: 'counter',
  reducer: createReducer(initialState, ...),
});
// Automatisch verfügbar:
// counterFeature.selectCounterState  (Root-Selektor)
// counterFeature.selectCount         (Property-Selektor)
// counterFeature.selectLoading       (Property-Selektor)
```

Zusätzlich können mit dem optionalen `extraSelectors`-Parameter abgeleitete Selektoren inline definiert werden, die Zugriff auf alle automatisch erzeugten Basis-Selektoren haben.

## Aufgabe

Baue einen einfachen **Task-Manager-State** mit `createFeature`. Der State soll eine Liste von Tasks, einen Loading-Indikator und eine optionale Fehlermeldung verwalten.

### Schritte

1. Definiere das `Task`-Interface und das `TasksState`-Interface (mit `tasks: Task[]`, `loading: boolean`, `error: string | null`).

2. Erstelle `initialState` und einen Reducer mit `createReducer` und folgenden Actions (via `createActionGroup`):
   - `TasksActions.loadTasks` – setzt `loading: true`
   - `TasksActions.loadTasksSuccess` – setzt `tasks`, `loading: false`
   - `TasksActions.loadTasksFailure` – setzt `error`, `loading: false`
   - `TasksActions.addTask` – fügt einen neuen Task hinzu
   - `TasksActions.removeTask` – entfernt einen Task per `id`

3. Verpacke Reducer und Name in `createFeature`. Exportiere alle generierten Selektoren via Destructuring.

4. Erweitere die Feature-Definition mit `extraSelectors`, um einen `selectCompletedTasks`-Selektor zu erzeugen, der nur abgeschlossene Tasks zurückgibt.

5. Registriere das Feature in der App mit `provideState(tasksFeature)` und injiziere die Selektoren in einer Komponente per `store.selectSignal()`.

## Hints

<details>
<summary>Hint 1 – createFeature Grundstruktur</summary>

```typescript
import { createFeature, createReducer, on } from '@ngrx/store';

export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(
    initialState,
    on(TasksActions.loadTasks, state => ({ ...state, loading: true })),
    // weitere ons ...
  ),
});

export const {
  name,
  reducer,
  selectTasksState,   // Root-Selektor
  selectTasks,        // Property-Selektor
  selectLoading,      // Property-Selektor
  selectError,        // Property-Selektor
} = tasksFeature;
```

</details>

<details>
<summary>Hint 2 – extraSelectors für abgeleitete Selektoren</summary>

```typescript
export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(initialState, /* ... */),
  extraSelectors: ({ selectTasks }) => ({
    selectCompletedTasks: createSelector(
      selectTasks,
      tasks => tasks.filter(t => t.completed)
    ),
    selectPendingCount: createSelector(
      selectTasks,
      tasks => tasks.filter(t => !t.completed).length
    ),
  }),
});
```

`extraSelectors` ist eine Funktion, die alle automatisch generierten Property-Selektoren erhält und ein Objekt mit weiteren Selektoren zurückgibt. Diese sind dann ebenfalls auf dem Feature-Objekt verfügbar.

</details>

## Beispiellösung

```typescript
// task.model.ts
export interface Task {
  id: string;
  title: string;
  completed: boolean;
}

// tasks.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export const TasksActions = createActionGroup({
  source: 'Tasks',
  events: {
    'Load Tasks': emptyProps(),
    'Load Tasks Success': props<{ tasks: Task[] }>(),
    'Load Tasks Failure': props<{ error: string }>(),
    'Add Task': props<{ task: Task }>(),
    'Remove Task': props<{ id: string }>(),
  },
});

// tasks.feature.ts
import { createFeature, createReducer, createSelector, on } from '@ngrx/store';
import { Task } from './task.model';
import { TasksActions } from './tasks.actions';

interface TasksState {
  tasks: Task[];
  loading: boolean;
  error: string | null;
}

const initialState: TasksState = {
  tasks: [],
  loading: false,
  error: null,
};

export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(
    initialState,
    on(TasksActions.loadTasks, state => ({
      ...state,
      loading: true,
      error: null,
    })),
    on(TasksActions.loadTasksSuccess, (state, { tasks }) => ({
      ...state,
      tasks,
      loading: false,
    })),
    on(TasksActions.loadTasksFailure, (state, { error }) => ({
      ...state,
      error,
      loading: false,
    })),
    on(TasksActions.addTask, (state, { task }) => ({
      ...state,
      tasks: [...state.tasks, task],
    })),
    on(TasksActions.removeTask, (state, { id }) => ({
      ...state,
      tasks: state.tasks.filter(t => t.id !== id),
    })),
  ),
  extraSelectors: ({ selectTasks }) => ({
    selectCompletedTasks: createSelector(
      selectTasks,
      tasks => tasks.filter(t => t.completed)
    ),
    selectPendingCount: createSelector(
      selectTasks,
      tasks => tasks.filter(t => !t.completed).length
    ),
  }),
});

export const {
  name,
  reducer,
  selectTasksState,
  selectTasks,
  selectLoading,
  selectError,
  selectCompletedTasks,
  selectPendingCount,
} = tasksFeature;

// app.config.ts
import { provideStore, provideState } from '@ngrx/store';
import { tasksFeature } from './tasks.feature';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideState(tasksFeature),
  ],
};

// tasks.component.ts
@Component({
  selector: 'app-tasks',
  template: `
    @if (loading()) {
      <p>Lädt...</p>
    }
    @for (task of tasks(); track task.id) {
      <div>
        {{ task.title }}
        <button (click)="remove(task.id)">Löschen</button>
      </div>
    }
    <p>Offen: {{ pendingCount() }}</p>
  `,
})
export class TasksComponent {
  private store = inject(Store);

  tasks = this.store.selectSignal(selectTasks);
  loading = this.store.selectSignal(selectLoading);
  pendingCount = this.store.selectSignal(selectPendingCount);

  remove(id: string) {
    this.store.dispatch(TasksActions.removeTask({ id }));
  }
}
```

## Weiterführendes

- **NgRx Docs – createFeature**: https://ngrx.io/guide/store/feature-creators – offizielle Dokumentation mit allen Optionen
- **Tipp**: Kombiniere `createFeature` mit `createFeature` aus `@ngrx/signals` (NgRx Signal Store), wenn du auf Signals-basiertes State Management umsteigen möchtest – die Konzepte ähneln sich, aber der Signal Store ist noch deklarativer.
- **Tipp**: Nutze `jest-auto-spies` oder `ngrx-mock-store` (`provideMockStore`) zusammen mit den generierten Selektoren für einfacheres Component-Testing.

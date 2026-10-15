# Angular Dojo: Optimistische Updates mit NgRx Effects
**Datum:** 2026-10-15
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit NgRx Effects optimistische UI-Updates implementierst, die sofort reagieren und bei einem Fehler automatisch zurückgerollt werden – für ein flüssiges Nutzererlebnis ohne Ladezeiten bei einfachen Operationen.

## Hintergrund & Theorie

Beim **optimistischen Update-Muster** wird die UI sofort aktualisiert, bevor die Server-Antwort eintrifft. Der pessimistische Ansatz wartet dagegen erst auf die Serverbestätigung, was die Anwendung träge wirken lässt.

**Ablauf eines optimistischen Updates:**
1. Nutzer löst eine Aktion aus (z. B. "Aufgabe als erledigt markieren")
2. Der Store wird **sofort** mit dem erwarteten Ergebnis aktualisiert
3. Im Hintergrund wird der HTTP-Request gesendet
4. **Erfolg:** Nichts weiter zu tun (oder Store mit Server-Daten synchronisieren)
5. **Fehler:** Der Store wird auf den vorherigen Zustand **zurückgerollt**

**Schlüsselelemente in NgRx:**
- Drei Actions pro Operation: `action`, `actionSuccess`, `actionFailure`
- Der Reducer reagiert sofort auf die Haupt-Action (nicht erst auf Success)
- Der Effect sendet den HTTP-Request, dispatcht Success oder Failure
- Bei Failure enthält die Action den **vorherigen Zustand** zum Rollback

Der Trick liegt darin, im Effect den aktuellen State vor dem Request zu lesen (`withLatestFrom(this.store.select(...))`) und diesen Snapshot in die Failure-Action mitzugeben.

## Aufgabe

Baue einen **Task-Manager** mit optimistischen Updates. Aufgaben können als erledigt markiert oder wieder geöffnet werden. Die UI reagiert sofort; bei einem simulierten Server-Fehler (zufällig ausgelöst) wird der vorherige Zustand wiederhergestellt und eine Fehlermeldung angezeigt.

### Schritte

1. **State und Actions definieren**: Erstelle einen `TasksState` mit `Task[]` und drei Actions für den Toggle: `toggleTask`, `toggleTaskSuccess`, `toggleTaskFailure` (mit `previousTasks: Task[]` im Failure-Payload).

2. **Reducer für sofortigen Update**: Der Reducer reagiert auf `toggleTask` und aktualisiert `completed` des Tasks direkt. Auf `toggleTaskFailure` stellt er `previousTasks` wieder her.

3. **Effect mit Rollback**: Implementiere einen Effect, der vor dem HTTP-Call den aktuellen State snapshoted (`withLatestFrom`), den Request sendet, bei Erfolg `toggleTaskSuccess` dispatcht und bei Fehler `toggleTaskFailure` mit dem Snapshot dispatcht.

4. **Fake HTTP-Service**: Erstelle einen `TaskService` mit einer `toggleTask(id)`-Methode, die mit 30% Wahrscheinlichkeit einen Fehler wirft (simulierter Server-Fehler).

5. **Komponente**: Zeige die Tasks als Liste an. Deaktiviere einen Task-Button kurz während des Requests nicht (da optimistisch). Zeige eine Fehlermeldung wenn `toggleTaskFailure` dispatcht wurde.

## Hints

<details>
<summary>Hint 1 – Actions mit Rollback-Payload</summary>

```typescript
export const TaskActions = createActionGroup({
  source: 'Tasks',
  events: {
    'Toggle Task': props<{ taskId: string }>(),
    'Toggle Task Success': props<{ task: Task }>(),
    'Toggle Task Failure': props<{ taskId: string; previousTasks: Task[] }>(),
  },
});
```

Im Reducer:
```typescript
on(TaskActions.toggleTask, (state, { taskId }) => ({
  ...state,
  tasks: state.tasks.map(t =>
    t.id === taskId ? { ...t, completed: !t.completed } : t
  ),
})),
on(TaskActions.toggleTaskFailure, (state, { previousTasks }) => ({
  ...state,
  tasks: previousTasks,
  error: 'Update fehlgeschlagen – Änderung wurde zurückgerollt.',
})),
```

</details>

<details>
<summary>Hint 2 – Effect mit withLatestFrom für den Snapshot</summary>

```typescript
toggleTask$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TaskActions.toggleTask),
    withLatestFrom(this.store.select(selectAllTasks)),
    switchMap(([{ taskId }, previousTasks]) =>
      this.taskService.toggleTask(taskId).pipe(
        map(task => TaskActions.toggleTaskSuccess({ task })),
        catchError(() =>
          of(TaskActions.toggleTaskFailure({ taskId, previousTasks }))
        )
      )
    )
  )
);
```

`previousTasks` ist der State **vor** dem optimistischen Update im Reducer – der Snapshot aus `withLatestFrom` wird ausgeführt bevor `switchMap` den HTTP-Call startet, aber da der Reducer synchron auf `toggleTask` reagiert hat, enthält der Store schon den optimistischen Wert. Den Snapshot **vor** dem Dispatch braucht man deshalb nicht selbst managen – `withLatestFrom` liest den State synchron nach dem Dispatch, also mit dem optimistischen Update. Für echten Rollback musst du den vorherigen State **vor** dem Dispatch kennen.

**Korrekte Lösung:** Lese den State im Effect mit einem zweistufigen Ansatz:
```typescript
// In der Action selbst den Snapshot mitgeben:
// 'Toggle Task': props<{ taskId: string; snapshot: Task[] }>()
// Snapshot in der Komponente vor dem Dispatch lesen:
this.store.select(selectAllTasks).pipe(take(1)).subscribe(snapshot => {
  this.store.dispatch(TaskActions.toggleTask({ taskId, snapshot }));
});
```

</details>

## Beispiellösung

```typescript
// task.model.ts
export interface Task {
  id: string;
  title: string;
  completed: boolean;
}

// task.actions.ts
export const TaskActions = createActionGroup({
  source: 'Tasks',
  events: {
    'Toggle Task': props<{ taskId: string; snapshot: Task[] }>(),
    'Toggle Task Success': props<{ task: Task }>(),
    'Toggle Task Failure': props<{ snapshot: Task[]; error: string }>(),
    'Load Tasks Success': props<{ tasks: Task[] }>(),
  },
});

// task.reducer.ts
export interface TasksState {
  tasks: Task[];
  error: string | null;
}

const initialState: TasksState = {
  tasks: [
    { id: '1', title: 'Angular lernen', completed: false },
    { id: '2', title: 'NgRx verstehen', completed: false },
    { id: '3', title: 'Optimistic Updates üben', completed: false },
  ],
  error: null,
};

export const tasksReducer = createReducer(
  initialState,
  on(TaskActions.toggleTask, (state, { taskId }) => ({
    ...state,
    error: null,
    // Optimistischer Update: sofort toggeln
    tasks: state.tasks.map(t =>
      t.id === taskId ? { ...t, completed: !t.completed } : t
    ),
  })),
  on(TaskActions.toggleTaskSuccess, (state, { task }) => ({
    ...state,
    // Mit Server-Daten synchronisieren
    tasks: state.tasks.map(t => (t.id === task.id ? task : t)),
  })),
  on(TaskActions.toggleTaskFailure, (state, { snapshot, error }) => ({
    ...state,
    tasks: snapshot, // Rollback auf den Snapshot vor dem Update
    error,
  }))
);

// task.service.ts
@Injectable({ providedIn: 'root' })
export class TaskService {
  private http = inject(HttpClient);

  toggleTask(taskId: string, completed: boolean): Observable<Task> {
    // Simulierter Server: 30% Fehlerrate
    if (Math.random() < 0.3) {
      return throwError(() => new Error('Server-Fehler'));
    }
    // In echter App: return this.http.patch<Task>(`/api/tasks/${taskId}`, { completed });
    return of({ id: taskId, title: 'Task', completed }).pipe(delay(800));
  }
}

// task.effects.ts
@Injectable()
export class TaskEffects {
  private actions$ = inject(Actions);
  private taskService = inject(TaskService);

  toggleTask$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TaskActions.toggleTask),
      switchMap(({ taskId, snapshot }) => {
        // Ziel-completed-Wert aus dem Snapshot ableiten (vor dem optimistischen Update)
        const prevTask = snapshot.find(t => t.id === taskId);
        const newCompleted = !prevTask?.completed;

        return this.taskService.toggleTask(taskId, newCompleted).pipe(
          map(task => TaskActions.toggleTaskSuccess({ task })),
          catchError(err =>
            of(
              TaskActions.toggleTaskFailure({
                snapshot,
                error: `Update fehlgeschlagen: ${err.message}. Änderung wurde zurückgerollt.`,
              })
            )
          )
        );
      })
    )
  );
}

// task-list.component.ts
@Component({
  selector: 'app-task-list',
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf],
  template: `
    <h2>Aufgaben</h2>

    @if (error$ | async; as error) {
      <div class="error-banner">{{ error }}</div>
    }

    <ul>
      @for (task of tasks$ | async; track task.id) {
        <li [class.completed]="task.completed">
          <input
            type="checkbox"
            [checked]="task.completed"
            (change)="onToggle(task.id)"
          />
          {{ task.title }}
        </li>
      }
    </ul>
  `,
  styles: [`
    .completed { text-decoration: line-through; opacity: 0.6; }
    .error-banner { background: #fee; color: #c00; padding: 8px; border-radius: 4px; margin-bottom: 8px; }
  `],
})
export class TaskListComponent {
  private store = inject(Store);

  tasks$ = this.store.select(selectAllTasks);
  error$ = this.store.select(selectTaskError);

  onToggle(taskId: string): void {
    // Snapshot VOR dem Dispatch lesen für zuverlässigen Rollback
    this.store
      .select(selectAllTasks)
      .pipe(take(1))
      .subscribe(snapshot => {
        this.store.dispatch(TaskActions.toggleTask({ taskId, snapshot }));
      });
  }
}
```

## Weiterführendes

- **Pessimistisch vs. Optimistisch**: Für destruktive Operationen (Löschen, Zahlungen) lieber pessimistisch vorgehen – das Nutzervertrauen wiegt schwerer als die Latenz.
- **NgRx Entity + Optimistic Updates**: Mit `updateOne`/`upsertOne` lässt sich der optimistische Update bei NgRx Entity eleganter ausdrücken – der Snapshot bleibt das Rollback-Instrument.
- **Konfliktbehandlung**: Bei parallelen Updates (mehrere User) braucht man Server-seitige Versionierung (ETag, `version`-Feld) und ggf. eine Merge-Strategie statt simpler Rollbacks.
- [NgRx Docs – Effects](https://ngrx.io/guide/effects)
- [Blog: Optimistic UI Updates – Patterns and Pitfalls](https://blog.angular.io)

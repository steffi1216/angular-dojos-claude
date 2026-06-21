# Angular Dojo: Angular Animations
**Datum:** 2026-06-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, mit dem Angular Animations-System komplexe, zustandsbasierte Animationen zu erstellen – inklusive `trigger`, `state`, `transition`, `query` und `stagger` für Listen-Animationen.

## Hintergrund & Theorie

Angular Animations basieren auf dem Web Animations API und sind eng in die Change Detection integriert. Das System arbeitet mit folgenden Kernkonzepten:

- **`trigger(name, [...])`** – Bindet eine Animation an ein Template-Property (`[@triggerName]`)
- **`state(name, style({...}))`** – Definiert CSS-Styles für einen bestimmten Zustand
- **`transition('a => b', [animate(...)])`** – Beschreibt den Übergang zwischen Zuständen; Wildcards wie `* => *` oder `:enter`/`:leave` sind möglich
- **`animate('Xs ease-in', style({...}))`** – Legt Dauer, Easing und Ziel-Style fest
- **`query` + `stagger`** – Ermöglichen animierte Listen, indem Child-Elemente verzögert animiert werden

Animationen müssen im `AppModule` (oder in Standalone-Komponenten via `provideAnimations()`) aktiviert werden:

```typescript
// app.config.ts (Standalone)
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';

export const appConfig: ApplicationConfig = {
  providers: [provideAnimationsAsync()]
};
```

Wichtig: Animationen mit `:enter` und `:leave` reagieren auf das Hinzufügen/Entfernen von DOM-Elementen (`*ngIf`, `@for`, `@if`).

## Aufgabe

Erstelle eine Standalone-Komponente `AnimatedTaskListComponent`, die eine Liste von Aufgaben verwaltet. Die Komponente soll folgendes Verhalten zeigen:

1. **Neue Tasks** fliegen beim Hinzufügen von links herein (`:enter`-Animation)
2. **Gelöschte Tasks** verschwinden mit einer Fade-out + Slide-left-Animation (`:leave`)
3. **Status-Toggle** (offen/erledigt): Der Hintergrund wechselt animiert zwischen Weiß und Grün
4. **Stagger-Effekt**: Beim ersten Laden erscheinen alle Tasks nacheinander mit 80ms Verzögerung

### Schritte

1. Erstelle `animated-task-list.component.ts` als Standalone-Komponente
2. Definiere ein `taskAnimation`-Trigger mit `:enter`- und `:leave`-Transitionen
3. Definiere einen `statusAnimation`-Trigger mit zwei States (`open` und `done`)
4. Erstelle einen `listAnimation`-Trigger auf dem Container, der `query + stagger` nutzt
5. Binde alle Trigger im Template und teste das Verhalten mit einem kleinen UI (Add/Delete/Toggle-Buttons)

## Hints

<details>
<summary>Hint 1 – Struktur der Trigger</summary>

```typescript
import {
  trigger, state, style, transition, animate, query, stagger
} from '@angular/animations';

export const taskAnimation = trigger('taskAnimation', [
  transition(':enter', [
    style({ transform: 'translateX(-100%)', opacity: 0 }),
    animate('300ms ease-out', style({ transform: 'translateX(0)', opacity: 1 }))
  ]),
  transition(':leave', [
    animate('250ms ease-in', style({ transform: 'translateX(-100%)', opacity: 0 }))
  ])
]);
```

</details>

<details>
<summary>Hint 2 – State-Animation für Status</summary>

```typescript
export const statusAnimation = trigger('statusAnimation', [
  state('open', style({ backgroundColor: '#ffffff', color: '#333' })),
  state('done', style({ backgroundColor: '#d4edda', color: '#155724' })),
  transition('open <=> done', animate('200ms ease-in-out'))
]);
```

</details>

<details>
<summary>Hint 3 – Stagger für die Liste</summary>

```typescript
export const listAnimation = trigger('listAnimation', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(-10px)' }),
      stagger(80, [
        animate('300ms ease-out', style({ opacity: 1, transform: 'translateY(0)' }))
      ])
    ], { optional: true })
  ])
]);
```

Im Template auf dem Container:
```html
<ul [@listAnimation]="tasks.length">
  <li *ngFor="let task of tasks" [@taskAnimation] [@statusAnimation]="task.status">
    ...
  </li>
</ul>
```

</details>

## Beispiellösung

```typescript
// animated-task-list.component.ts
import { Component, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import {
  trigger, state, style, transition,
  animate, query, stagger
} from '@angular/animations';

interface Task {
  id: number;
  title: string;
  status: 'open' | 'done';
}

const taskAnimation = trigger('taskAnimation', [
  transition(':enter', [
    style({ transform: 'translateX(-100%)', opacity: 0 }),
    animate('300ms ease-out', style({ transform: 'translateX(0)', opacity: 1 }))
  ]),
  transition(':leave', [
    animate('250ms ease-in', style({ transform: 'translateX(-100%)', opacity: 0 }))
  ])
]);

const statusAnimation = trigger('statusAnimation', [
  state('open', style({ backgroundColor: '#ffffff', color: '#333333' })),
  state('done', style({ backgroundColor: '#d4edda', color: '#155724' })),
  transition('open <=> done', animate('200ms ease-in-out'))
]);

const listAnimation = trigger('listAnimation', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(-10px)' }),
      stagger(80, [
        animate('300ms ease-out', style({ opacity: 1, transform: 'translateY(0)' }))
      ])
    ], { optional: true })
  ])
]);

@Component({
  selector: 'app-animated-task-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  animations: [taskAnimation, statusAnimation, listAnimation],
  template: `
    <div style="max-width: 500px; margin: 2rem auto; font-family: sans-serif;">
      <h2>Aufgabenliste</h2>

      <div style="display: flex; gap: 8px; margin-bottom: 1rem;">
        <input
          [(ngModel)]="newTaskTitle"
          placeholder="Neue Aufgabe..."
          style="flex: 1; padding: 8px; border: 1px solid #ccc; border-radius: 4px;"
          (keyup.enter)="addTask()"
        />
        <button
          (click)="addTask()"
          style="padding: 8px 16px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer;"
        >
          Hinzufügen
        </button>
      </div>

      <ul [@listAnimation]="tasks().length" style="list-style: none; padding: 0; margin: 0;">
        @for (task of tasks(); track task.id) {
          <li
            [@taskAnimation]
            [@statusAnimation]="task.status"
            style="display: flex; align-items: center; gap: 8px; padding: 12px;
                   margin-bottom: 8px; border-radius: 6px; border: 1px solid #dee2e6;
                   overflow: hidden;"
          >
            <span style="flex: 1;" [style.textDecoration]="task.status === 'done' ? 'line-through' : 'none'">
              {{ task.title }}
            </span>
            <button
              (click)="toggleStatus(task)"
              style="padding: 4px 10px; border-radius: 4px; border: 1px solid #28a745;
                     background: transparent; cursor: pointer; color: #28a745;"
            >
              {{ task.status === 'open' ? '✓ Erledigen' : '↩ Öffnen' }}
            </button>
            <button
              (click)="removeTask(task.id)"
              style="padding: 4px 10px; border-radius: 4px; border: 1px solid #dc3545;
                     background: transparent; cursor: pointer; color: #dc3545;"
            >
              ✕
            </button>
          </li>
        }
      </ul>

      <p style="color: #6c757d; margin-top: 1rem;">
        {{ openCount() }} offen · {{ doneCount() }} erledigt
      </p>
    </div>
  `
})
export class AnimatedTaskListComponent {
  tasks = signal<Task[]>([
    { id: 1, title: 'Angular Animations lernen', status: 'open' },
    { id: 2, title: 'Dojo abschließen', status: 'open' },
    { id: 3, title: 'Lösung committen', status: 'open' }
  ]);

  newTaskTitle = '';
  private nextId = 4;

  openCount = computed(() => this.tasks().filter(t => t.status === 'open').length);
  doneCount = computed(() => this.tasks().filter(t => t.status === 'done').length);

  addTask(): void {
    const title = this.newTaskTitle.trim();
    if (!title) return;
    this.tasks.update(tasks => [
      ...tasks,
      { id: this.nextId++, title, status: 'open' }
    ]);
    this.newTaskTitle = '';
  }

  removeTask(id: number): void {
    this.tasks.update(tasks => tasks.filter(t => t.id !== id));
  }

  toggleStatus(task: Task): void {
    this.tasks.update(tasks =>
      tasks.map(t =>
        t.id === task.id
          ? { ...t, status: t.status === 'open' ? 'done' : 'open' }
          : t
      )
    );
  }
}
```

## Weiterführendes

- **`AnimationBuilder`**: Für programmatisch gesteuerte Animationen außerhalb des Templates – nützlich für imperative Animation (z. B. auf Canvas-Overlays)
- **`useAnimation` + `animation()`**: Wiederverwendbare Animations-Factories, die sich wie Funktionen parametrieren lassen (`params: { duration: '300ms' }`)
- **Route-Animationen**: `RouterOutlet` mit `@routeAnimation`-Trigger für seitenweite Übergänge – kombinierbar mit `ActivatedRoute`-Daten als Animations-State
- [Offizielle Doku](https://angular.dev/guide/animations)

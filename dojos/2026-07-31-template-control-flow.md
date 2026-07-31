# Angular Dojo: Template Control Flow (@if, @for, @switch)
**Datum:** 2026-07-31
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst den neuen eingebauten Template Control Flow von Angular 17+ kennen – `@if`, `@for` und `@switch` – und verstehst, wie er sich von den alten Direktiven `*ngIf`, `*ngFor` und `*ngSwitch` unterscheidet und verbessert.

## Hintergrund & Theorie
Ab Angular 17 bietet das Framework eine neue, native Template-Syntax für Kontrollfluss-Logik, die direkt im Angular-Compiler verankert ist – kein Import von `CommonModule` oder `NgIf`/`NgFor` mehr nötig.

**Vorteile gegenüber der alten Direktiven-Syntax:**
- **Bessere Performance:** `@for` erfordert `track` (früher optional via `trackBy`) und optimiert das Rendering intern anders.
- **Lesbarkeit:** Klarer, näher an echtem TypeScript-Syntax.
- **Typ-Narrowing:** `@if` und `@else` funktionieren mit TypeScript-Typ-Narrowing direkt im Template.
- **`@empty`-Block:** `@for` hat einen eingebauten Leerfall ohne Extra-Direktive.
- **Kein Import nötig:** Standalone Components brauchen kein `CommonModule` mehr.

```html
<!-- Alt -->
<div *ngIf="user; else noUser">{{ user.name }}</div>
<ng-template #noUser>Kein User</ng-template>

<!-- Neu -->
@if (user) {
  <div>{{ user.name }}</div>
} @else {
  <span>Kein User</span>
}
```

```html
<!-- @for mit track (Pflicht) und @empty -->
@for (item of items; track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>Keine Einträge vorhanden</li>
}
```

## Aufgabe
Erstelle eine Standalone-Komponente `TaskListComponent`, die eine Liste von Aufgaben darstellt und folgende Features nutzt:

- `@for` mit `track` zum Rendern der Aufgaben
- `@empty`-Block wenn keine Aufgaben vorhanden
- `@if` / `@else if` / `@else` für den Status jeder Aufgabe (`'pending' | 'in-progress' | 'done'`)
- `@switch` zum Anzeigen einer Status-Badge-Farbe
- Ein Button zum Leeren der Liste (testet `@empty`)

### Schritte
1. Erstelle das Interface `Task` mit den Feldern `id: number`, `title: string`, `status: 'pending' | 'in-progress' | 'done'`.
2. Erstelle eine Standalone-Komponente mit einem `tasks` Signal, das initial 3 Demo-Tasks enthält.
3. Nutze `@for (task of tasks(); track task.id)` im Template.
4. Innerhalb jedes Items: zeige mit `@if` / `@else if` / `@else` einen passenden Statustext an.
5. Nutze `@switch (task.status)` um eine CSS-Klasse (`badge-pending`, `badge-progress`, `badge-done`) zu vergeben.
6. Füge einen "Alle löschen"-Button hinzu und zeige mit `@empty` einen Hinweis, wenn die Liste leer ist.
7. Füge optional einen `@if (tasks().length > 2)` Hinweis hinzu: "Du hast viele Aufgaben!".

## Hints
<details>
<summary>Hint 1 – @for Grundstruktur</summary>

```html
@for (task of tasks(); track task.id) {
  <div>{{ task.title }}</div>
} @empty {
  <p>Keine Aufgaben vorhanden.</p>
}
```

`track` ist kein optionales `trackBy` mehr – es ist Pflicht und der Compiler warnt, wenn es fehlt. Nutze eine eindeutige Eigenschaft wie `task.id`.

</details>

<details>
<summary>Hint 2 – @if mit @else if und @switch</summary>

```html
@if (task.status === 'done') {
  <span>Erledigt ✓</span>
} @else if (task.status === 'in-progress') {
  <span>In Bearbeitung…</span>
} @else {
  <span>Ausstehend</span>
}

@switch (task.status) {
  @case ('pending') { <span class="badge-pending">Offen</span> }
  @case ('in-progress') { <span class="badge-progress">Aktiv</span> }
  @case ('done') { <span class="badge-done">Fertig</span> }
}
```

</details>

## Beispiellösung

```typescript
import { Component, signal } from '@angular/core';

interface Task {
  id: number;
  title: string;
  status: 'pending' | 'in-progress' | 'done';
}

@Component({
  selector: 'app-task-list',
  standalone: true,
  template: `
    <h2>Aufgabenliste</h2>

    @if (tasks().length > 2) {
      <p class="warning">Du hast viele Aufgaben!</p>
    }

    <ul>
      @for (task of tasks(); track task.id) {
        <li>
          <strong>{{ task.title }}</strong>

          @switch (task.status) {
            @case ('pending') {
              <span class="badge badge-pending">Offen</span>
            }
            @case ('in-progress') {
              <span class="badge badge-progress">Aktiv</span>
            }
            @case ('done') {
              <span class="badge badge-done">Fertig</span>
            }
          }

          @if (task.status === 'done') {
            <em> – Erledigt ✓</em>
          } @else if (task.status === 'in-progress') {
            <em> – In Bearbeitung…</em>
          } @else {
            <em> – Noch nicht gestartet</em>
          }
        </li>
      } @empty {
        <li class="empty-state">Keine Aufgaben vorhanden. Gut gemacht!</li>
      }
    </ul>

    <button (click)="clearTasks()">Alle löschen</button>
    <button (click)="resetTasks()">Zurücksetzen</button>
  `,
  styles: [`
    .badge { padding: 2px 8px; border-radius: 4px; font-size: 0.8em; margin-left: 8px; }
    .badge-pending { background: #f0ad4e; }
    .badge-progress { background: #5bc0de; }
    .badge-done { background: #5cb85c; color: white; }
    .warning { color: orange; font-weight: bold; }
    .empty-state { font-style: italic; color: gray; }
  `],
})
export class TaskListComponent {
  private readonly initialTasks: Task[] = [
    { id: 1, title: 'Angular Control Flow lernen', status: 'done' },
    { id: 2, title: 'Signals vertiefen', status: 'in-progress' },
    { id: 3, title: 'Unit Tests schreiben', status: 'pending' },
  ];

  tasks = signal<Task[]>([...this.initialTasks]);

  clearTasks(): void {
    this.tasks.set([]);
  }

  resetTasks(): void {
    this.tasks.set([...this.initialTasks]);
  }
}
```

## Weiterführendes
- **Migration:** Das Angular CLI bietet eine automatische Migration: `ng generate @angular/core:control-flow` – sie konvertiert alle `*ngIf`, `*ngFor`, `*ngSwitch` in der gesamten App auf die neue Syntax.
- **`@defer` kombinieren:** Der neue `@defer`-Block (Angular 17+) kann mit `@if` und `@for` kombiniert werden – ideal für Lazy Loading von Template-Abschnitten (siehe Dojo vom 2026-07-01).
- **Offizielle Docs:** [angular.dev/guide/templates/control-flow](https://angular.dev/guide/templates/control-flow)

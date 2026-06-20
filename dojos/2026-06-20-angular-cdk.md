# Angular Dojo: Angular CDK – Virtual Scrolling, Drag & Drop, Overlay

**Datum:** 2026-06-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst das Angular Component Dev Kit (CDK) kennen und übst drei seiner mächtigsten Features: Virtual Scrolling für performante Listen, Drag & Drop für interaktive UIs und das Overlay-System für dynamische Popups.

## Hintergrund & Theorie

Das **Angular CDK** ist eine Sammlung von Low-Level-Primitiven, auf denen die Angular Material-Komponenten aufbauen – du kannst sie aber auch völlig unabhängig davon nutzen. Das CDK enthält keine opinionated Styles und lässt sich daher in jedes Design-System integrieren.

**Die drei wichtigsten Module heute:**

- **`ScrollingModule` (Virtual Scrolling):** Rendert bei langen Listen nur die sichtbaren Elemente im DOM. Statt 10.000 `<li>`-Elemente werden z. B. nur 20 gerendert und beim Scrollen ausgetauscht. Entscheidend für Performance bei großen Datensätzen.

- **`DragDropModule`:** Ermöglicht Drag & Drop zwischen Listen (`cdkDropList`) und innerhalb einer Liste mit automatischer Animations-Unterstützung. Die `moveItemInArray`- und `transferArrayItem`-Hilfsfunktionen machen das Sortieren trivial.

- **`OverlayModule`:** Ein mächtiges Low-Level-API für Tooltips, Dropdowns, Modale und Panels. Positionierung erfolgt über `ConnectedPosition`-Strategien, die automatisch auf verfügbaren Bildschirmraum reagieren.

Installation: `npm install @angular/cdk`

## Aufgabe

Erstelle eine **Task-Board-Komponente** mit drei Spalten (Todo, In Progress, Done). Aufgaben können per Drag & Drop zwischen den Spalten verschoben werden. Bei Hover über eine Aufgabe erscheint ein CDK-Overlay mit Details. Als Bonus: Nutze Virtual Scrolling, wenn eine Spalte mehr als 50 Einträge enthält.

### Schritte

1. Installiere `@angular/cdk` und importiere `DragDropModule`, `ScrollingModule` und `OverlayModule` in deiner Standalone-Komponente.

2. Definiere drei Arrays (`todo`, `inProgress`, `done`) mit Task-Objekten (`{ id, title, description, priority }`). Fülle `todo` mit mindestens 60 Einträgen für den Virtual-Scrolling-Test.

3. Baue das HTML-Template mit drei `cdkDropList`-Containern, die per `[cdkDropListConnectedTo]` miteinander verknüpft sind. Jedes Task-Item erhält `cdkDrag`.

4. Implementiere den `(cdkDropListDropped)`-Handler und nutze `transferArrayItem` bzw. `moveItemInArray` aus `@angular/cdk/drag-drop`.

5. Füge Virtual Scrolling mit `<cdk-virtual-scroll-viewport>` für die Todo-Spalte hinzu (Höhe fixieren, `itemSize` setzen).

6. Erstelle einen Overlay-Service oder nutze `Overlay` direkt in der Komponente, um bei `mouseenter` auf einem Task ein Panel mit `description` und `priority` anzuzeigen, das sich am Mauszeiger ausrichtet.

## Hints

<details>
<summary>Hint 1 – Drag & Drop zwischen Listen</summary>

Verknüpfe die drei `cdkDropList`-Referenzen gegenseitig:

```typescript
@ViewChild('todoList') todoList!: CdkDropList;
@ViewChild('inProgressList') inProgressList!: CdkDropList;
@ViewChild('doneList') doneList!: CdkDropList;
```

Im Template:
```html
<div cdkDropList #todoList="cdkDropList"
     [cdkDropListData]="todo"
     [cdkDropListConnectedTo]="[inProgressList, doneList]"
     (cdkDropListDropped)="drop($event)">
  <div *ngFor="let task of todo" cdkDrag>{{ task.title }}</div>
</div>
```

Im Handler:
```typescript
drop(event: CdkDragDrop<Task[]>) {
  if (event.previousContainer === event.container) {
    moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
  } else {
    transferArrayItem(
      event.previousContainer.data,
      event.container.data,
      event.previousIndex,
      event.currentIndex,
    );
  }
}
```
</details>

<details>
<summary>Hint 2 – Virtual Scrolling</summary>

Virtual Scrolling benötigt eine **fixe Höhe** auf dem Viewport und eine bekannte `itemSize` in Pixeln:

```html
<cdk-virtual-scroll-viewport itemSize="56" style="height: 400px;">
  <div *cdkVirtualFor="let task of todo" cdkDrag class="task-item">
    {{ task.title }}
  </div>
</cdk-virtual-scroll-viewport>
```

**Achtung:** `*cdkVirtualFor` ist das CDK-Pendant zu `*ngFor` – es rendert nur die sichtbaren Elemente. Kombiniere es mit `cdkDrag` wie gewohnt.
</details>

<details>
<summary>Hint 3 – Overlay für Task-Details</summary>

Injiziere `Overlay` und `ViewContainerRef`, erstelle ein `OverlayRef` bei `mouseenter`:

```typescript
constructor(private overlay: Overlay, private vcr: ViewContainerRef) {}

showOverlay(event: MouseEvent, task: Task) {
  const positionStrategy = this.overlay.position()
    .global()
    .left(`${event.clientX + 12}px`)
    .top(`${event.clientY + 12}px`);

  this.overlayRef = this.overlay.create({ positionStrategy });

  const portal = new TemplatePortal(this.detailTemplate, this.vcr, { $implicit: task });
  this.overlayRef.attach(portal);
}

hideOverlay() {
  this.overlayRef?.detach();
}
```

Das Template für das Panel:
```html
<ng-template #detailTemplate let-task>
  <div class="overlay-panel">
    <strong>{{ task.title }}</strong>
    <p>{{ task.description }}</p>
    <span [class]="'priority-' + task.priority">{{ task.priority }}</span>
  </div>
</ng-template>
```
</details>

## Beispiellösung

```typescript
import { Component, ViewChild, TemplateRef, ViewContainerRef, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { CdkDragDrop, DragDropModule, moveItemInArray, transferArrayItem } from '@angular/cdk/drag-drop';
import { ScrollingModule } from '@angular/cdk/scrolling';
import { Overlay, OverlayModule, OverlayRef } from '@angular/cdk/overlay';
import { TemplatePortal } from '@angular/cdk/portal';

interface Task {
  id: number;
  title: string;
  description: string;
  priority: 'low' | 'medium' | 'high';
}

@Component({
  selector: 'app-task-board',
  standalone: true,
  imports: [CommonModule, DragDropModule, ScrollingModule, OverlayModule],
  template: `
    <div class="board">

      <!-- Todo-Spalte mit Virtual Scrolling -->
      <div class="column">
        <h2>Todo ({{ todo.length }})</h2>
        <cdk-virtual-scroll-viewport itemSize="56" class="column-viewport">
          <div cdkDropList #todoList="cdkDropList"
               [cdkDropListData]="todo"
               [cdkDropListConnectedTo]="[inProgressList, doneList]"
               (cdkDropListDropped)="drop($event)"
               class="task-list">
            <div *cdkVirtualFor="let task of todo"
                 cdkDrag
                 class="task-item"
                 (mouseenter)="showOverlay($event, task)"
                 (mouseleave)="hideOverlay()">
              {{ task.title }}
              <span class="priority" [class]="'p-' + task.priority">{{ task.priority }}</span>
            </div>
          </div>
        </cdk-virtual-scroll-viewport>
      </div>

      <!-- In Progress -->
      <div class="column">
        <h2>In Progress ({{ inProgress.length }})</h2>
        <div cdkDropList #inProgressList="cdkDropList"
             [cdkDropListData]="inProgress"
             [cdkDropListConnectedTo]="[todoList, doneList]"
             (cdkDropListDropped)="drop($event)"
             class="task-list">
          <div *ngFor="let task of inProgress"
               cdkDrag
               class="task-item"
               (mouseenter)="showOverlay($event, task)"
               (mouseleave)="hideOverlay()">
            {{ task.title }}
            <span class="priority" [class]="'p-' + task.priority">{{ task.priority }}</span>
          </div>
        </div>
      </div>

      <!-- Done -->
      <div class="column">
        <h2>Done ({{ done.length }})</h2>
        <div cdkDropList #doneList="cdkDropList"
             [cdkDropListData]="done"
             [cdkDropListConnectedTo]="[todoList, inProgressList]"
             (cdkDropListDropped)="drop($event)"
             class="task-list">
          <div *ngFor="let task of done"
               cdkDrag
               class="task-item"
               (mouseenter)="showOverlay($event, task)"
               (mouseleave)="hideOverlay()">
            {{ task.title }}
            <span class="priority" [class]="'p-' + task.priority">{{ task.priority }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Overlay-Template für Task-Details -->
    <ng-template #detailTpl let-task>
      <div class="detail-overlay">
        <strong>{{ task.title }}</strong>
        <p>{{ task.description }}</p>
        <span class="priority" [class]="'p-' + task.priority">Priorität: {{ task.priority }}</span>
      </div>
    </ng-template>
  `,
  styles: [`
    .board { display: flex; gap: 16px; padding: 16px; }
    .column { flex: 1; background: #f4f5f7; border-radius: 8px; padding: 12px; }
    .column-viewport { height: 400px; }
    .task-list { min-height: 100px; }
    .task-item {
      background: white;
      border-radius: 4px;
      padding: 12px;
      margin-bottom: 8px;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
      cursor: grab;
      display: flex;
      justify-content: space-between;
      align-items: center;
      height: 40px;
      box-sizing: border-box;
    }
    .task-item.cdk-drag-preview { box-shadow: 0 4px 12px rgba(0,0,0,0.2); }
    .task-item.cdk-drag-placeholder { opacity: 0.3; }
    .priority { font-size: 11px; border-radius: 3px; padding: 2px 6px; }
    .p-high { background: #ffebe6; color: #bf2600; }
    .p-medium { background: #fffae6; color: #974f0c; }
    .p-low { background: #e3fcef; color: #006644; }
    .detail-overlay {
      background: white;
      border: 1px solid #ddd;
      border-radius: 6px;
      padding: 12px 16px;
      max-width: 260px;
      box-shadow: 0 4px 16px rgba(0,0,0,0.12);
      pointer-events: none;
    }
    .detail-overlay p { margin: 6px 0; font-size: 13px; color: #555; }
  `]
})
export class TaskBoardComponent {
  private overlay = inject(Overlay);
  private vcr = inject(ViewContainerRef);

  @ViewChild('detailTpl') detailTpl!: TemplateRef<{ $implicit: Task }>;

  private overlayRef: OverlayRef | null = null;

  todo: Task[] = Array.from({ length: 60 }, (_, i) => ({
    id: i + 1,
    title: `Task ${i + 1}`,
    description: `Beschreibung für Task ${i + 1}. Hier stehen weitere Details zur Aufgabe.`,
    priority: (['low', 'medium', 'high'] as const)[i % 3],
  }));

  inProgress: Task[] = [
    { id: 101, title: 'Auth-Modul refactoring', description: 'JWT-Handling vereinfachen', priority: 'high' },
    { id: 102, title: 'Unit Tests schreiben', description: 'Coverage auf 80% erhöhen', priority: 'medium' },
  ];

  done: Task[] = [
    { id: 201, title: 'CI/CD Pipeline', description: 'GitHub Actions eingerichtet', priority: 'high' },
  ];

  drop(event: CdkDragDrop<Task[]>) {
    if (event.previousContainer === event.container) {
      moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
    } else {
      transferArrayItem(
        event.previousContainer.data,
        event.container.data,
        event.previousIndex,
        event.currentIndex,
      );
    }
  }

  showOverlay(event: MouseEvent, task: Task) {
    this.hideOverlay();

    const positionStrategy = this.overlay.position()
      .global()
      .left(`${event.clientX + 16}px`)
      .top(`${event.clientY + 16}px`);

    this.overlayRef = this.overlay.create({
      positionStrategy,
      hasBackdrop: false,
      scrollStrategy: this.overlay.scrollStrategies.close(),
    });

    const portal = new TemplatePortal(this.detailTpl, this.vcr, { $implicit: task });
    this.overlayRef.attach(portal);
  }

  hideOverlay() {
    this.overlayRef?.detach();
    this.overlayRef = null;
  }
}
```

## Weiterführendes

- **CDK Portal:** Für noch flexiblere Overlays lies die [PortalModule-Dokumentation](https://material.angular.io/cdk/portal/overview) – `ComponentPortal` erlaubt komplette Komponenten als Overlay-Inhalt statt Templates.
- **Drag & Drop mit Backend:** Für Persistenz nach dem Drop kombiniere den `(cdkDropListDropped)`-Handler mit einem NgRx-Action-Dispatch oder einem HTTP-PATCH-Aufruf, um die neue Reihenfolge ans Backend zu schicken.
- **`CdkScrollable` & `ScrollDispatcher`:** Wenn du auf Scroll-Events von beliebigen Elementen reagieren willst (z. B. Sticky Headers), ist der `ScrollDispatcher` aus `@angular/cdk/scrolling` dein Werkzeug.
- Offizielle Doku: [angular.io/cdk](https://material.angular.io/cdk/categories)

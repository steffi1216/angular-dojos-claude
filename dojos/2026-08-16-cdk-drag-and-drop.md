# Angular Dojo: CDK Drag & Drop
**Datum:** 2026-08-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK sortierbare Listen und Transfer zwischen mehreren Listen implementierst – inklusive benutzerdefinierter Preview, Handles und reaktiver State-Verwaltung via Signals.

## Hintergrund & Theorie

Das `@angular/cdk/drag-drop`-Modul bietet eine vollständige, accessible Drag-&-Drop-Infrastruktur ohne externe Abhängigkeiten.

**Wichtigste Konzepte:**

- **`cdkDrag`** – Directive, die ein Element draggable macht.
- **`cdkDropList`** – Container, der draggable Elemente aufnimmt. Emittiert `cdkDropListDropped` nach einem Drop.
- **`moveItemInArray` / `transferArrayItem`** – Hilfsfunktionen aus dem CDK, die das Mutieren von Arrays nach einem Drop übernehmen.
- **`cdkDropListConnectedTo`** – Verbindet mehrere `cdkDropList`-Container, sodass Elemente zwischen ihnen verschoben werden können.
- **`cdkDragHandle`** – Schränkt den Drag auf ein bestimmtes Kindelement (z. B. einen Gripper-Icon) ein.
- **`*cdkDragPreview`** – Template für eine benutzerdefinierte Drag-Preview.
- **`*cdkDragPlaceholder`** – Template für den Platzhalter, der während des Drags an der Ursprungsposition erscheint.
- **`cdkDragData`** – Beliebige Daten, die an ein draggables Element gebunden werden und im `CdkDragDrop`-Event verfügbar sind.

Das CDK kümmert sich um Accessibility (ARIA-Attribute, Keyboard-Navigation) und Touch-Events out of the box.

## Aufgabe

Baue ein **Kanban-Board** mit drei Spalten (*Todo*, *In Progress*, *Done*). Jede Spalte ist eine `cdkDropList`. Karten können innerhalb einer Spalte umsortiert und zwischen Spalten verschoben werden. Jede Karte hat einen dedizierten **Drag-Handle** (ein Icon). Implementiere außerdem einen **Signal-basierten Counter**, der anzeigt, wie viele Karten sich aktuell in jeder Spalte befinden.

### Schritte

1. **Setup** – Installiere das CDK (falls noch nicht geschehen) und importiere `CdkDragDrop`, `moveItemInArray`, `transferArrayItem` sowie die CDK-Direktiven in deiner Standalone-Komponente.

2. **Datenmodell** – Definiere ein Interface `KanbanCard { id: number; title: string; priority: 'low' | 'medium' | 'high' }` und lege drei `signal<KanbanCard[]>`-Arrays an (eines pro Spalte).

3. **Template** – Erstelle drei Spalten mit je einer `cdkDropList`. Verbinde alle drei über `[cdkDropListConnectedTo]`. Iteriere die Karten mit `@for` und versehe jede mit `cdkDrag` und `[cdkDragData]`.

4. **Handle & Preview** – Füge innerhalb jeder Karte ein Element mit `cdkDragHandle` hinzu. Definiere außerdem ein `*cdkDragPreview`-Template mit einer kompakteren Darstellung der Karte.

5. **Drop-Handler** – Implementiere eine Methode `onDrop(event: CdkDragDrop<KanbanCard[]>)`. Unterscheide darin, ob der Drop in derselben Liste (`event.previousContainer === event.container`) oder zwischen Listen stattgefunden hat, und rufe die passende CDK-Hilfsfunktion auf. Aktualisiere die Signals anschließend.

6. **Counter** – Erstelle ein `computed`-Signal `columnCounts` das ein Objekt `{ todo: number, inProgress: number, done: number }` zurückgibt. Zeige die Counts in den Spalten-Headern an.

## Hints

<details>
<summary>Hint 1 – Imports & Setup</summary>

```typescript
import { CdkDragDrop, CdkDrag, CdkDropList, CdkDragHandle,
         CdkDragPreview, moveItemInArray, transferArrayItem } from '@angular/cdk/drag-drop';

@Component({
  imports: [CdkDrag, CdkDropList, CdkDragHandle, CdkDragPreview],
  // ...
})
```

Für Standalone-Komponenten genügt es, die einzelnen Direktiven zu importieren – kein `DragDropModule` nötig.

</details>

<details>
<summary>Hint 2 – Signals aktualisieren nach Drop</summary>

`moveItemInArray` und `transferArrayItem` mutieren das Array in-place. Signals erkennen Mutations nicht automatisch. Du musst deshalb eine neue Array-Referenz zuweisen:

```typescript
onDrop(event: CdkDragDrop<KanbanCard[]>) {
  if (event.previousContainer === event.container) {
    const arr = [...event.container.data];
    moveItemInArray(arr, event.previousIndex, event.currentIndex);
    // Signal mit neuer Referenz aktualisieren
    this.getSignalForList(event.container.id).set(arr);
  } else {
    const prev = [...event.previousContainer.data];
    const curr = [...event.container.data];
    transferArrayItem(prev, curr, event.previousIndex, event.currentIndex);
    this.getSignalForList(event.previousContainer.id).set(prev);
    this.getSignalForList(event.container.id).set(curr);
  }
}
```

</details>

<details>
<summary>Hint 3 – cdkDropListConnectedTo & IDs</summary>

```html
<div cdkDropList
     id="todo-list"
     [cdkDropListData]="todo()"
     [cdkDropListConnectedTo]="['in-progress-list', 'done-list']"
     (cdkDropListDropped)="onDrop($event)">
  @for (card of todo(); track card.id) {
    <div cdkDrag [cdkDragData]="card">
      <span cdkDragHandle>⠿</span>
      {{ card.title }}
      <ng-template cdkDragPreview>
        <span class="drag-preview">{{ card.title }}</span>
      </ng-template>
    </div>
  }
</div>
```

</details>

## Beispiellösung

```typescript
import { Component, signal, computed } from '@angular/core';
import {
  CdkDragDrop, CdkDrag, CdkDropList, CdkDragHandle,
  CdkDragPreview, moveItemInArray, transferArrayItem
} from '@angular/cdk/drag-drop';

interface KanbanCard {
  id: number;
  title: string;
  priority: 'low' | 'medium' | 'high';
}

@Component({
  selector: 'app-kanban',
  standalone: true,
  imports: [CdkDrag, CdkDropList, CdkDragHandle, CdkDragPreview],
  template: `
    <div class="board">
      @for (col of columns; track col.id) {
        <div class="column">
          <h3>{{ col.label }} ({{ columnCounts()[col.id] }})</h3>
          <div
            cdkDropList
            [id]="col.id"
            [cdkDropListData]="col.cards()"
            [cdkDropListConnectedTo]="connectedLists(col.id)"
            (cdkDropListDropped)="onDrop($event)"
            class="drop-zone">
            @for (card of col.cards(); track card.id) {
              <div cdkDrag [cdkDragData]="card" class="card" [class]="card.priority">
                <span cdkDragHandle class="handle">⠿</span>
                <span>{{ card.title }}</span>
                <ng-template cdkDragPreview>
                  <div class="preview">{{ card.title }}</div>
                </ng-template>
              </div>
            } @empty {
              <div class="empty-hint">Kein Eintrag</div>
            }
          </div>
        </div>
      }
    </div>
  `,
})
export class KanbanComponent {
  todo = signal<KanbanCard[]>([
    { id: 1, title: 'Login-Seite gestalten', priority: 'high' },
    { id: 2, title: 'Unit-Tests schreiben', priority: 'medium' },
    { id: 3, title: 'README aktualisieren', priority: 'low' },
  ]);

  inProgress = signal<KanbanCard[]>([
    { id: 4, title: 'API anbinden', priority: 'high' },
  ]);

  done = signal<KanbanCard[]>([
    { id: 5, title: 'Projektsetup', priority: 'low' },
  ]);

  columns = [
    { id: 'todo', label: 'Todo', cards: this.todo },
    { id: 'in-progress', label: 'In Progress', cards: this.inProgress },
    { id: 'done', label: 'Done', cards: this.done },
  ] as const;

  columnCounts = computed(() => ({
    todo: this.todo().length,
    'in-progress': this.inProgress().length,
    done: this.done().length,
  }));

  connectedLists(currentId: string): string[] {
    return this.columns.map(c => c.id).filter(id => id !== currentId);
  }

  private signalFor(id: string) {
    const col = this.columns.find(c => c.id === id);
    if (!col) throw new Error(`Unknown column: ${id}`);
    return col.cards as ReturnType<typeof signal<KanbanCard[]>>;
  }

  onDrop(event: CdkDragDrop<KanbanCard[]>) {
    if (event.previousContainer === event.container) {
      const arr = [...event.container.data];
      moveItemInArray(arr, event.previousIndex, event.currentIndex);
      this.signalFor(event.container.id).set(arr);
    } else {
      const prev = [...event.previousContainer.data];
      const curr = [...event.container.data];
      transferArrayItem(prev, curr, event.previousIndex, event.currentIndex);
      this.signalFor(event.previousContainer.id).set(prev);
      this.signalFor(event.container.id).set(curr);
    }
  }
}
```

## Weiterführendes

- **Sorting-Predicates** – Mit `[cdkDropListSortingDisabled]` und `[cdkDropListEnterPredicate]` kannst du steuern, ob und welche Elemente in eine Liste gedropped werden dürfen (z. B. nur `high`-Priority-Karten in *Done* erlauben).
- **Animations** – Das CDK lässt sich mit Angular Animations kombinieren: Die CSS-Klassen `cdk-drag-animating` und `cdk-drag-preview` bieten Einstiegspunkte für `@keyframes`-Übergänge.
- **Offizielle Docs:** [material.angular.io/cdk/drag-drop/overview](https://material.angular.io/cdk/drag-drop/overview)

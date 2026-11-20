# Angular Dojo: Angular Material Tree (MatTree)
**Datum:** 2026-11-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `MatTreeModule` hierarchische Daten als interaktiven Baum darstellst – einmal mit `FlatTreeControl` (flache Datenstruktur) und einmal mit `NestedTreeControl` (verschachtelte Objekte).

## Hintergrund & Theorie

Angular Material bietet mit `MatTreeModule` eine leistungsfähige Komponente für Baumstrukturen. Es gibt zwei grundlegende Ansätze:

**FlatTreeControl** arbeitet mit einer flachen Liste von Knoten, bei denen jeder Knoten ein `level`-Attribut besitzt. Der `MatTreeFlattener` übernimmt die Transformation von verschachtelten Daten in diese flache Form. Diese Variante eignet sich besonders für Virtual Scrolling, da alle Nodes in einem einzigen Array liegen.

**NestedTreeControl** arbeitet direkt mit verschachtelten Objekten. Jeder Knoten verweist auf seine Kindknoten über eine `childrenAccessor`-Funktion. Diese Variante ist einfacher aufzusetzen, aber schlechter für sehr große Datensätze geeignet.

Beide Ansätze benötigen:
- Einen `TreeControl` (für expand/collapse-Logik)
- Eine `MatTreeDataSource` oder `MatTreeNestedDataSource`
- Templates für Blatt- und Zweigknoten (`matTreeNodeDef`, `matNestedTreeNode`)

Ab Angular 17+ kann `childrenAccessor` direkt an `mat-tree` übergeben werden, was den Boilerplate für `NestedTreeControl` deutlich reduziert.

## Aufgabe

Erstelle einen **Datei-Explorer** als Standalone Component, der eine verschachtelte Ordnerstruktur als expandierbaren Baum anzeigt.

### Schritte

1. Generiere eine neue Standalone Component `FileExplorerComponent` und importiere `MatTreeModule`, `MatIconModule` und `MatButtonModule`.

2. Definiere eine Interface `FileNode` mit den Feldern `name: string`, `type: 'file' | 'folder'` und optional `children?: FileNode[]`.

3. Erstelle eine `fileData: FileNode[]` im Service oder direkt in der Komponente mit einer 3-Level-tiefen Struktur (z. B. Projekte → Ordner → Dateien).

4. Baue die Komponente mit **`NestedTreeControl`** und dem neuen `childrenAccessor`-Ansatz (Angular 17+):
   ```typescript
   childrenAccessor = (node: FileNode) => node.children ?? [];
   hasChild = (_: number, node: FileNode) => !!node.children?.length;
   ```

5. Erstelle das Template mit `mat-tree`, `mat-tree-node` für Blätter und `mat-nested-tree-node` für Zweige. Zeige ein `mat-icon` an (`folder`/`insert_drive_file`) und einen Toggle-Button für Ordner.

6. **Bonus:** Baue dieselbe Datenstruktur mit `FlatTreeControl` + `MatTreeFlattener` nach und vergleiche den Unterschied im Template.

### Schritte

1. Standalone Component anlegen, Imports setzen
2. `FileNode`-Interface und Testdaten definieren
3. `childrenAccessor` und `hasChild`-Funktion implementieren
4. HTML-Template mit `mat-tree` und den beiden `matTreeNodeDef`-Varianten schreiben
5. Styling: Einrückung über `[style.padding-left]="node.level * 16 + 'px'"` oder `matTreeNodePadding`
6. (Optional) FlatTree-Variante implementieren

## Hints

<details>
<summary>Hint 1 – NestedTreeControl Template-Grundgerüst</summary>

```html
<mat-tree [dataSource]="dataSource" [childrenAccessor]="childrenAccessor">
  <!-- Blatt-Knoten (kein Expand-Button) -->
  <mat-tree-node *matTreeNodeDef="let node" matTreeNodePadding>
    <button mat-icon-button disabled></button>
    <mat-icon>insert_drive_file</mat-icon>
    {{ node.name }}
  </mat-tree-node>

  <!-- Zweig-Knoten (mit Expand/Collapse) -->
  <mat-nested-tree-node *matTreeNodeDef="let node; when: hasChild">
    <div class="mat-tree-node">
      <button mat-icon-button matTreeNodeToggle>
        <mat-icon>
          {{ treeControl.isExpanded(node) ? 'expand_more' : 'chevron_right' }}
        </mat-icon>
      </button>
      <mat-icon>folder</mat-icon>
      {{ node.name }}
    </div>
    <div [class.hidden]="!treeControl.isExpanded(node)" role="group">
      <ng-container matTreeNodeOutlet></ng-container>
    </div>
  </mat-nested-tree-node>
</mat-tree>
```
</details>

<details>
<summary>Hint 2 – FlatTree mit MatTreeFlattener</summary>

Für den `FlatTreeControl`-Ansatz brauchst du einen `MatTreeFlattener<T, F>`, der deine verschachtelten Nodes in flache Nodes umwandelt:

```typescript
interface FlatFileNode {
  name: string;
  type: 'file' | 'folder';
  level: number;
  expandable: boolean;
}

private transformer = (node: FileNode, level: number): FlatFileNode => ({
  name: node.name,
  type: node.type,
  level,
  expandable: !!node.children?.length,
});

treeFlattener = new MatTreeFlattener(
  this.transformer,
  node => node.level,
  node => node.expandable,
  node => node.children,
);

treeControl = new FlatTreeControl<FlatFileNode>(
  node => node.level,
  node => node.expandable,
);

dataSource = new MatTreeFlatDataSource(this.treeControl, this.treeFlattener);
```

Im Template entfällt dann `mat-nested-tree-node` – du arbeitest ausschließlich mit `mat-tree-node` und `matTreeNodePadding` für die Einrückung.
</details>

## Beispiellösung

```typescript
// file-explorer.component.ts
import { Component } from '@angular/core';
import { MatTreeModule, MatTreeNestedDataSource } from '@angular/material/tree';
import { MatIconModule } from '@angular/material/icon';
import { MatButtonModule } from '@angular/material/button';
import { NestedTreeControl } from '@angular/cdk/tree';

interface FileNode {
  name: string;
  type: 'file' | 'folder';
  children?: FileNode[];
}

const FILE_DATA: FileNode[] = [
  {
    name: 'src',
    type: 'folder',
    children: [
      {
        name: 'app',
        type: 'folder',
        children: [
          { name: 'app.component.ts', type: 'file' },
          { name: 'app.routes.ts', type: 'file' },
          {
            name: 'features',
            type: 'folder',
            children: [
              { name: 'dashboard.component.ts', type: 'file' },
              { name: 'settings.component.ts', type: 'file' },
            ],
          },
        ],
      },
      { name: 'main.ts', type: 'file' },
      { name: 'styles.scss', type: 'file' },
    ],
  },
  {
    name: 'public',
    type: 'folder',
    children: [
      { name: 'favicon.ico', type: 'file' },
      { name: 'robots.txt', type: 'file' },
    ],
  },
  { name: 'angular.json', type: 'file' },
  { name: 'package.json', type: 'file' },
];

@Component({
  selector: 'app-file-explorer',
  standalone: true,
  imports: [MatTreeModule, MatIconModule, MatButtonModule],
  template: `
    <mat-tree [dataSource]="dataSource" [childrenAccessor]="childrenAccessor">
      <!-- Blatt-Knoten -->
      <mat-tree-node *matTreeNodeDef="let node" matTreeNodePadding>
        <button mat-icon-button disabled aria-label="Kein Kind"></button>
        <mat-icon class="file-icon">insert_drive_file</mat-icon>
        {{ node.name }}
      </mat-tree-node>

      <!-- Zweig-Knoten (Ordner) -->
      <mat-nested-tree-node *matTreeNodeDef="let node; when: hasChild">
        <div class="mat-tree-node">
          <button
            mat-icon-button
            matTreeNodeToggle
            [attr.aria-label]="'Toggle ' + node.name"
          >
            <mat-icon>
              {{ treeControl.isExpanded(node) ? 'expand_more' : 'chevron_right' }}
            </mat-icon>
          </button>
          <mat-icon class="folder-icon">folder</mat-icon>
          {{ node.name }}
        </div>
        <div
          role="group"
          [class.hidden]="!treeControl.isExpanded(node)"
        >
          <ng-container matTreeNodeOutlet></ng-container>
        </div>
      </mat-nested-tree-node>
    </mat-tree>
  `,
  styles: [`
    .hidden { display: none; }
    .folder-icon { color: #f9a825; }
    .file-icon { color: #78909c; }
    mat-tree-node, .mat-tree-node { align-items: center; }
  `],
})
export class FileExplorerComponent {
  childrenAccessor = (node: FileNode) => node.children ?? [];
  hasChild = (_: number, node: FileNode) => !!node.children?.length;

  treeControl = new NestedTreeControl<FileNode>(this.childrenAccessor);

  dataSource = new MatTreeNestedDataSource<FileNode>();

  constructor() {
    this.dataSource.data = FILE_DATA;
    // Ersten Level direkt aufklappen
    this.treeControl.expandDescendants(FILE_DATA[0]);
  }
}
```

```html
<!-- In app.component.html -->
<h2>Datei-Explorer</h2>
<app-file-explorer />
```

## Weiterführendes

- **Virtual Scrolling + FlatTree:** Kombiniere `MatTreeFlatDataSource` mit `cdk-virtual-scroll-viewport` für Bäume mit Tausenden von Knoten – nur der `FlatTreeControl` unterstützt dies, da alle Nodes in einem Array vorliegen.
- [Angular Material Tree Docs](https://material.angular.io/components/tree/overview) – Offizielle Dokumentation mit Beispielen für Checkboxes, Lazy Loading und Filtering.
- **Lazy Loading von Kindknoten:** Lade Children erst beim Expand-Event, indem du auf `treeControl.expansionModel.changed` reagierst und die `dataSource.data` reaktiv aktualisierst.

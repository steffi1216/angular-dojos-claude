# Angular Dojo: TemplateRef & ngTemplateOutlet
**Datum:** 2026-09-04
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Verstehe, wie `TemplateRef` und `ngTemplateOutlet` funktionieren, um wiederverwendbare, konfigurierbare Komponenten zu bauen, bei denen der Aufrufer eigene Template-Schnipsel einbringen kann.

## Hintergrund & Theorie

`TemplateRef` repräsentiert ein eingebettetes View-Template (definiert via `<ng-template>`). Es wird nicht sofort gerendert, sondern kann programmatisch oder deklarativ instanziiert werden.

`ngTemplateOutlet` ist eine Direktive, die ein `TemplateRef` an einer bestimmten Stelle im DOM rendert – optional mit einem Context-Objekt, das Template-Variablen definiert.

Das Muster ist die Grundlage für viele UI-Bibliotheken (Angular Material, PrimeNG) und ermöglicht das sog. **Template API Pattern**:

```html
<!-- Aufrufer definiert das Template -->
<my-list [items]="items">
  <ng-template #itemTpl let-item>
    <strong>{{ item.name }}</strong>
  </ng-template>
</my-list>
```

```typescript
// Komponente nimmt das Template entgegen und rendert es
@ContentChild('itemTpl') itemTemplate!: TemplateRef<{ $implicit: Item }>;
```

Der Context (`let-item`) wird aus dem `$implicit`-Feld des Context-Objekts befüllt. Weitere Felder werden mit `let-x="feldname"` abgerufen.

Typische Einsatzszenarien:
- Generische Listen- oder Tabellenkomponenten mit anpassbarem Zeilen-Template
- Wiederverwendbare Dialoge/Cards mit customisierbarem Header/Footer
- Skeleton-Loader mit konfigurierbarem Content-Slot

## Aufgabe

Baue eine generische `<app-data-table>`-Komponente, die eine Liste von Items rendert. Der Aufrufer kann über ein `<ng-template>`-API selbst bestimmen, wie jede Zeile aussieht – die Tabelle kümmert sich nur um Layout, Pagination und den Leer-Zustand.

### Schritte

1. **Erstelle die `DataTableComponent`** als Standalone Component mit folgenden Inputs:
   - `items: T[]` (generic, nutze `any[]` wenn nötig)
   - `pageSize: number` (Default: 5)
   - Optional: `emptyTemplate: TemplateRef<void>` für einen Custom-Leer-Zustand

2. **Definiere das Template-API** über `@ContentChild`:
   - `rowTemplate: TemplateRef<{ $implicit: any; index: number }>` – das Row-Template des Aufrufers
   - Fallback: rendere `{{ item | json }}` wenn kein Template übergeben wurde

3. **Implementiere einfache Pagination** (vorherige/nächste Seite) als Signal-State.

4. **Rendere die Items** der aktuellen Seite mit `*ngTemplateOutlet` (oder `@for` + `[ngTemplateOutlet]`):
   ```html
   <ng-container
     *ngTemplateOutlet="rowTemplate; context: { $implicit: item, index: i }"
   />
   ```

5. **Nutze die Komponente** in `AppComponent`:
   ```html
   <app-data-table [items]="users" [pageSize]="3">
     <ng-template #row let-user let-i="index">
       <div>{{ i + 1 }}. {{ user.name }} – {{ user.email }}</div>
     </ng-template>
   </app-data-table>
   ```

## Hints

<details>
<summary>Hint 1 – ContentChild mit TemplateRef</summary>

```typescript
import { ContentChild, TemplateRef } from '@angular/core';

@Component({ ... })
export class DataTableComponent {
  @ContentChild('row') rowTemplate?: TemplateRef<{ $implicit: any; index: number }>;
}
```

Im Template des Aufrufers:
```html
<ng-template #row let-item let-index="index">...</ng-template>
```

Der Selektor `'row'` muss mit der Template-Variable `#row` übereinstimmen.
</details>

<details>
<summary>Hint 2 – ngTemplateOutlet mit Context</summary>

```html
@for (item of currentPageItems(); track item.id; let i = $index) {
  <ng-container
    [ngTemplateOutlet]="rowTemplate ?? defaultRowTpl"
    [ngTemplateOutletContext]="{ $implicit: item, index: pageOffset() + i }"
  />
}

<ng-template #defaultRowTpl let-item>
  <pre>{{ item | json }}</pre>
</ng-template>
```

`$implicit` ist der Wert, der mit `let-x` (ohne `="feldname"`) gebunden wird.
</details>

<details>
<summary>Hint 3 – Pagination mit Signals</summary>

```typescript
readonly currentPage = signal(0);
readonly pageOffset = computed(() => this.currentPage() * this.pageSize());
readonly currentPageItems = computed(() =>
  this.items().slice(this.pageOffset(), this.pageOffset() + this.pageSize())
);
readonly totalPages = computed(() =>
  Math.ceil(this.items().length / this.pageSize())
);

next() { if (this.currentPage() < this.totalPages() - 1) this.currentPage.update(p => p + 1); }
prev() { if (this.currentPage() > 0) this.currentPage.update(p => p - 1); }
```

</details>

## Beispiellösung

```typescript
// data-table.component.ts
import {
  Component, ContentChild, TemplateRef, input, signal, computed
} from '@angular/core';
import { NgTemplateOutlet, JsonPipe } from '@angular/common';

@Component({
  selector: 'app-data-table',
  standalone: true,
  imports: [NgTemplateOutlet, JsonPipe],
  template: `
    <div class="table-wrapper">
      @if (currentPageItems().length === 0) {
        @if (emptyTemplate()) {
          <ng-container [ngTemplateOutlet]="emptyTemplate()!" />
        } @else {
          <p class="empty">Keine Einträge vorhanden.</p>
        }
      } @else {
        @for (item of currentPageItems(); track $index; let i = $index) {
          <div class="row">
            <ng-container
              [ngTemplateOutlet]="rowTemplate ?? defaultRowTpl"
              [ngTemplateOutletContext]="{ $implicit: item, index: pageOffset() + i }"
            />
          </div>
        }
      }

      <ng-template #defaultRowTpl let-item>
        <pre>{{ item | json }}</pre>
      </ng-template>
    </div>

    <div class="pagination">
      <button (click)="prev()" [disabled]="currentPage() === 0">‹ Zurück</button>
      <span>Seite {{ currentPage() + 1 }} / {{ totalPages() }}</span>
      <button (click)="next()" [disabled]="currentPage() >= totalPages() - 1">Weiter ›</button>
    </div>
  `,
  styles: [`
    .table-wrapper { border: 1px solid #ddd; border-radius: 4px; padding: 8px; }
    .row { padding: 6px 8px; border-bottom: 1px solid #eee; }
    .row:last-child { border-bottom: none; }
    .empty { color: #888; text-align: center; padding: 16px; }
    .pagination { display: flex; align-items: center; gap: 12px; margin-top: 8px; }
    button:disabled { opacity: 0.4; cursor: default; }
  `]
})
export class DataTableComponent {
  readonly items = input<any[]>([]);
  readonly pageSize = input<number>(5);
  readonly emptyTemplate = input<TemplateRef<void> | null>(null);

  @ContentChild('row') rowTemplate?: TemplateRef<{ $implicit: any; index: number }>;

  readonly currentPage = signal(0);
  readonly pageOffset = computed(() => this.currentPage() * this.pageSize());
  readonly totalPages = computed(() =>
    Math.max(1, Math.ceil(this.items().length / this.pageSize()))
  );
  readonly currentPageItems = computed(() =>
    this.items().slice(this.pageOffset(), this.pageOffset() + this.pageSize())
  );

  next() {
    if (this.currentPage() < this.totalPages() - 1)
      this.currentPage.update(p => p + 1);
  }
  prev() {
    if (this.currentPage() > 0)
      this.currentPage.update(p => p - 1);
  }
}

// app.component.ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [DataTableComponent],
  template: `
    <h2>Nutzerliste</h2>
    <app-data-table [items]="users" [pageSize]="3">
      <ng-template #row let-user let-index="index">
        <span style="color: gray;">{{ index + 1 }}.</span>
        <strong>{{ user.name }}</strong> – {{ user.email }}
      </ng-template>
    </app-data-table>

    <h2>Leere Liste</h2>
    <app-data-table [items]="[]" [emptyTemplate]="noData" />
    <ng-template #noData>
      <p style="text-align:center; color: tomato;">🚫 Keine Daten geladen.</p>
    </ng-template>
  `
})
export class AppComponent {
  users = [
    { id: 1, name: 'Anna', email: 'anna@example.com' },
    { id: 2, name: 'Ben', email: 'ben@example.com' },
    { id: 3, name: 'Clara', email: 'clara@example.com' },
    { id: 4, name: 'David', email: 'david@example.com' },
    { id: 5, name: 'Eva', email: 'eva@example.com' },
    { id: 6, name: 'Felix', email: 'felix@example.com' },
  ];
}
```

## Weiterführendes
- **Typed Templates**: Nutze generische Komponenten mit `ngTemplateContextGuard` für typsichere Template-Contexts: `static ngTemplateContextGuard<T>(dir: DataTableComponent, ctx: any): ctx is { $implicit: T; index: number } { return true; }` – Angular schlussfolgert dann den Typ von `let-user` automatisch.
- **Angular Material CDK Table** (`CdkTable`) verwendet dasselbe Prinzip im großen Maßstab – lohnt sich als Lektüre für ein produktionsreifes Beispiel.
- Offizielle Doku: [angular.dev/api/common/NgTemplateOutlet](https://angular.dev/api/common/NgTemplateOutlet)

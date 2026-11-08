# Angular Dojo: Angular Material DataTable mit Sort, Filter & Paginator
**Datum:** 2026-11-08
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `MatTableDataSource`, `MatSort` und `MatPaginator` eine vollwertige, reaktive Datentabelle baust – mit client- und serverseitiger Paginierung sowie einem anpassbaren `filterPredicate`.

## Hintergrund & Theorie
`mat-table` (Angular Material) baut auf dem CDK-Table auf, bietet aber zusätzlich `MatTableDataSource` – eine smarte DataSource-Implementierung, die Sortierung, Filterung und Paginierung out-of-the-box verknüpft.

**Schlüsselkonzepte:**

- **`MatTableDataSource<T>`**: Wrapper um ein Array, der `MatSort` und `MatPaginator` per Property automatisch verdrahtet.
- **`MatSort`**: Direktive auf dem `<table>`-Element; `MatSortHeader` auf den `<th>`-Zellen. Sortierstatus wird automatisch propagiert.
- **`MatPaginator`**: Steuert, welche Seite der DataSource aktiv ist. `pageSizeOptions` legt die Auswahl fest.
- **`filterPredicate`**: Eine Funktion `(data: T, filter: string) => boolean`, mit der du die Filterlogik vollständig anpassen kannst (z.B. nur bestimmte Spalten durchsuchen, case-insensitive, etc.).
- **Server-side vs. client-side**: `MatTableDataSource` deckt den client-seitigen Fall ab. Für Server-side implementierst du eine eigene `DataSource<T>` mit `connect()` / `disconnect()` und einem Observable.

Die drei Bausteine müssen in `ngAfterViewInit` (oder mit `viewChild()` als Signal) verknüpft werden, da die ViewChild-Referenzen erst dann verfügbar sind.

## Aufgabe

Erstelle eine `UserTableComponent`, die eine Liste von Nutzern in einer `mat-table` anzeigt, mit:

1. Spalten: `id`, `name`, `email`, `role`
2. Sortierung auf allen Spalten
3. Einem globalen Text-Filter, der nur auf `name` und `email` filtert
4. Einem Paginator mit 5 / 10 / 25 Einträgen pro Seite
5. Einer Methode `applyFilter(event: Event)` für das Input-Binding

### Schritte

1. `MatTableModule`, `MatSortModule`, `MatPaginatorModule`, `MatInputModule`, `MatFormFieldModule` im Component-Imports-Array registrieren
2. `MatTableDataSource<User>` mit Demodaten initialisieren
3. `viewChild` (Signal-API) oder `@ViewChild` verwenden, um `MatSort` und `MatPaginator` zu verdrahten
4. Eigenen `filterPredicate` setzen, der ausschließlich `name` und `email` berücksichtigt
5. Template bauen: Filter-Input, `mat-table` mit `matSort`, `matSortHeader` und `mat-paginator`

## Hints

<details>
<summary>Hint 1 – DataSource und ViewChild verknüpfen</summary>

Verknüpfe in `ngAfterViewInit` (oder im `afterNextRender`-Callback):

```typescript
ngAfterViewInit() {
  this.dataSource.sort = this.sort();
  this.dataSource.paginator = this.paginator();
}
```

Mit Signal-API:
```typescript
private sort = viewChild.required(MatSort);
private paginator = viewChild.required(MatPaginator);
```

</details>

<details>
<summary>Hint 2 – Eigener filterPredicate</summary>

```typescript
this.dataSource.filterPredicate = (data: User, filter: string) => {
  const search = filter.toLowerCase();
  return (
    data.name.toLowerCase().includes(search) ||
    data.email.toLowerCase().includes(search)
  );
};
```

Und im Template:
```typescript
applyFilter(event: Event) {
  const value = (event.target as HTMLInputElement).value;
  this.dataSource.filter = value.trim().toLowerCase();
  if (this.dataSource.paginator) {
    this.dataSource.paginator.firstPage();
  }
}
```

</details>

## Beispiellösung

```typescript
import { AfterViewInit, Component, viewChild } from '@angular/core';
import { MatTableDataSource, MatTableModule } from '@angular/material/table';
import { MatSort, MatSortModule } from '@angular/material/sort';
import { MatPaginator, MatPaginatorModule } from '@angular/material/paginator';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';

interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'editor' | 'viewer';
}

const USERS: User[] = Array.from({ length: 50 }, (_, i) => ({
  id: i + 1,
  name: `User ${i + 1}`,
  email: `user${i + 1}@example.com`,
  role: (['admin', 'editor', 'viewer'] as const)[i % 3],
}));

@Component({
  selector: 'app-user-table',
  standalone: true,
  imports: [
    MatTableModule,
    MatSortModule,
    MatPaginatorModule,
    MatFormFieldModule,
    MatInputModule,
  ],
  template: `
    <mat-form-field>
      <mat-label>Suche</mat-label>
      <input matInput (input)="applyFilter($event)" placeholder="Name oder E-Mail" />
    </mat-form-field>

    <table mat-table [dataSource]="dataSource" matSort>
      <ng-container matColumnDef="id">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>ID</th>
        <td mat-cell *matCellDef="let row">{{ row.id }}</td>
      </ng-container>

      <ng-container matColumnDef="name">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
        <td mat-cell *matCellDef="let row">{{ row.name }}</td>
      </ng-container>

      <ng-container matColumnDef="email">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>E-Mail</th>
        <td mat-cell *matCellDef="let row">{{ row.email }}</td>
      </ng-container>

      <ng-container matColumnDef="role">
        <th mat-header-cell *matHeaderCellDef mat-sort-header>Rolle</th>
        <td mat-cell *matCellDef="let row">{{ row.role }}</td>
      </ng-container>

      <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
      <tr mat-row *matRowDef="let row; columns: displayedColumns"></tr>

      <tr class="mat-row" *matNoDataRow>
        <td class="mat-cell" colspan="4">Keine Ergebnisse für "{{ dataSource.filter }}"</td>
      </tr>
    </table>

    <mat-paginator
      [pageSizeOptions]="[5, 10, 25]"
      showFirstLastButtons
    />
  `,
})
export class UserTableComponent implements AfterViewInit {
  displayedColumns = ['id', 'name', 'email', 'role'];
  dataSource = new MatTableDataSource<User>(USERS);

  private sort = viewChild.required(MatSort);
  private paginator = viewChild.required(MatPaginator);

  constructor() {
    this.dataSource.filterPredicate = (data, filter) => {
      const search = filter.toLowerCase();
      return (
        data.name.toLowerCase().includes(search) ||
        data.email.toLowerCase().includes(search)
      );
    };
  }

  ngAfterViewInit() {
    this.dataSource.sort = this.sort();
    this.dataSource.paginator = this.paginator();
  }

  applyFilter(event: Event) {
    const value = (event.target as HTMLInputElement).value;
    this.dataSource.filter = value.trim().toLowerCase();
    if (this.dataSource.paginator) {
      this.dataSource.paginator.firstPage();
    }
  }
}
```

## Weiterführendes

- **Server-side Paginierung**: Implementiere `DataSource<T>` direkt (statt `MatTableDataSource`) und lass `connect()` ein Observable zurückgeben, das auf `MatSort.sortChange` und `MatPaginator.page` reagiert – ideal für APIs mit cursor- oder offset-basierter Paginierung.
- [Angular Material Table Docs](https://material.angular.io/components/table/overview) – Abschnitte *Pagination*, *Sorting* und *Filtering*
- Für komplexere Fälle: CDK `CollectionViewer` und benutzerdefinierte `DataSource` mit `BehaviorSubject`

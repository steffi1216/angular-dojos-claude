# Angular Dojo: CDK Virtual Scrolling
**Datum:** 2026-08-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK's `CdkVirtualScrollViewport` sehr große Listen performant renderst, indem nur die sichtbaren Elemente im DOM existieren – und wie du das mit eigenen Item-Größen und dynamischen Datenquellen kombinierst.

## Hintergrund & Theorie

Wenn eine Liste Tausende von Einträgen enthält, ist das direkte Rendern aller DOM-Elemente ein erhebliches Performance-Problem: langer Initial-Render, hoher Speicherverbrauch und stockendes Scrollen.

**Virtual Scrolling** löst dieses Problem durch ein Sliding Window: Es werden nur die Elemente gerendert, die gerade im sichtbaren Bereich (Viewport) sind, plus ein kleiner Buffer darüber und darunter. Beim Scrollen werden DOM-Elemente recycelt und mit neuen Daten befüllt – ähnlich wie RecyclerView (Android) oder UICollectionView (iOS).

Das Angular CDK bietet dafür `CdkVirtualScrollViewport` mit zwei Strategien:

- **`itemSize`** (Fixed Size): Alle Items haben dieselbe Höhe. Einfach, maximal performant.
- **`AutoSizeVirtualScrollStrategy`** (aus `@angular/cdk-experimental`): Items können unterschiedliche Höhen haben. Komplexer, da CDK Höhen nachmessen muss.

Für die meisten Anwendungsfälle reicht die Fixed-Size-Strategie. Sie lässt sich nahtlos mit Signals, RxJS und großen Datensätzen kombinieren.

Wichtige Konzepte:
- `scrolledIndexChange` – Event beim Scrollindex-Wechsel
- `measureScrollOffset()` – programmatischer Scroll-Zustand
- `scrollToIndex()` – programmatisch zu einem Index scrollen
- `DataSource<T>` – CDP-Protokoll für lazy Datenquellen

## Aufgabe

Erstelle eine Komponente `VirtualContactListComponent`, die eine Liste von 10.000 Kontakten virtuell scrollt. Implementiere zusätzlich:

1. Eine Suchfunktion (Filter per Signal), die die Liste einschränkt
2. Einen "Jump to top"-Button
3. Eine Statuszeile, die den aktuell sichtbaren Index-Bereich anzeigt

### Schritte

1. Installiere `@angular/cdk` falls noch nicht vorhanden (`ng add @angular/cdk`) und importiere `ScrollingModule` in deiner Standalone-Komponente.

2. Generiere 10.000 Mock-Kontakte in einem Signal: `contacts = signal(generateContacts(10_000))`.

3. Erstelle ein `searchTerm = signal('')` und ein `computed`-Signal `filteredContacts`, das die Kontakte nach Name filtert.

4. Baue das Template mit `<cdk-virtual-scroll-viewport itemSize="64" class="viewport">` und `*cdkVirtualFor="let contact of filteredContacts()"` darin.

5. Höre auf `(scrolledIndexChange)` und speichere den ersten sichtbaren Index in einem Signal `firstVisibleIndex`. Berechne via `computed` auch den letzten sichtbaren Index (`firstVisibleIndex() + viewportSize`).

6. Injiziere eine `ViewChild`-Referenz auf den Viewport (`CdkVirtualScrollViewport`) und implementiere `scrollToTop()` mit `viewport.scrollToIndex(0)`.

7. Style den Viewport mit einer fixen Höhe (z. B. `500px`) und `overflow-y: auto`.

## Hints

<details>
<summary>Hint 1 – Import und Template-Grundstruktur</summary>

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  imports: [ScrollingModule, FormsModule],
  template: `
    <input [ngModel]="searchTerm()" (ngModelChange)="searchTerm.set($event)" placeholder="Suchen..." />
    <cdk-virtual-scroll-viewport itemSize="64" class="viewport"
      (scrolledIndexChange)="firstVisibleIndex.set($event)">
      <div *cdkVirtualFor="let c of filteredContacts()" class="contact-item">
        {{ c.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
```

</details>

<details>
<summary>Hint 2 – ViewChild und programmatisches Scrollen</summary>

```typescript
viewport = viewChild.required(CdkVirtualScrollViewport);

scrollToTop() {
  this.viewport().scrollToIndex(0, 'smooth');
}
```

`scrollToIndex` akzeptiert als zweiten Parameter `'auto' | 'smooth' | 'instant'` (ScrollBehavior).

</details>

<details>
<summary>Hint 3 – Sichtbaren Bereich berechnen</summary>

```typescript
readonly ITEM_SIZE = 64;
readonly VIEWPORT_HEIGHT = 500;
readonly visibleCount = Math.floor(this.VIEWPORT_HEIGHT / this.ITEM_SIZE);

firstVisibleIndex = signal(0);
lastVisibleIndex = computed(() =>
  Math.min(
    this.firstVisibleIndex() + this.visibleCount - 1,
    this.filteredContacts().length - 1
  )
);
```

</details>

## Beispiellösung

```typescript
import { Component, computed, signal, viewChild } from '@angular/core';
import { CdkVirtualScrollViewport, ScrollingModule } from '@angular/cdk/scrolling';
import { FormsModule } from '@angular/forms';

interface Contact {
  id: number;
  name: string;
  email: string;
}

function generateContacts(count: number): Contact[] {
  const firstNames = ['Anna', 'Ben', 'Clara', 'David', 'Eva', 'Felix', 'Greta', 'Hans'];
  const lastNames = ['Müller', 'Schmidt', 'Weber', 'Wagner', 'Fischer', 'Becker', 'Hoffmann'];
  return Array.from({ length: count }, (_, i) => ({
    id: i + 1,
    name: `${firstNames[i % firstNames.length]} ${lastNames[i % lastNames.length]}`,
    email: `kontakt${i + 1}@example.com`,
  }));
}

@Component({
  selector: 'app-virtual-contact-list',
  standalone: true,
  imports: [ScrollingModule, FormsModule],
  template: `
    <div class="toolbar">
      <input
        [ngModel]="searchTerm()"
        (ngModelChange)="searchTerm.set($event)"
        placeholder="Kontakt suchen..."
        class="search-input"
      />
      <button (click)="scrollToTop()">↑ Zum Anfang</button>
    </div>

    <div class="status">
      Zeige {{ filteredContacts().length }} Kontakte
      @if (filteredContacts().length > 0) {
        &nbsp;· Sichtbar: {{ firstVisibleIndex() + 1 }}–{{ lastVisibleIndex() + 1 }}
      }
    </div>

    <cdk-virtual-scroll-viewport
      itemSize="64"
      class="viewport"
      (scrolledIndexChange)="firstVisibleIndex.set($event)"
    >
      <div *cdkVirtualFor="let contact of filteredContacts(); trackBy: trackById" class="contact-item">
        <div class="contact-name">{{ contact.name }}</div>
        <div class="contact-email">{{ contact.email }}</div>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .toolbar { display: flex; gap: 8px; margin-bottom: 8px; }
    .search-input { flex: 1; padding: 6px 10px; font-size: 14px; }
    .status { font-size: 12px; color: #666; margin-bottom: 4px; }
    .viewport { height: 500px; border: 1px solid #ddd; }
    .contact-item {
      height: 64px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      padding: 0 16px;
      border-bottom: 1px solid #eee;
      box-sizing: border-box;
    }
    .contact-name { font-weight: 500; }
    .contact-email { font-size: 12px; color: #888; }
  `],
})
export class VirtualContactListComponent {
  private readonly allContacts = signal(generateContacts(10_000));

  readonly searchTerm = signal('');
  readonly firstVisibleIndex = signal(0);

  readonly filteredContacts = computed(() => {
    const term = this.searchTerm().toLowerCase().trim();
    if (!term) return this.allContacts();
    return this.allContacts().filter(c => c.name.toLowerCase().includes(term));
  });

  readonly lastVisibleIndex = computed(() => {
    const visibleCount = Math.floor(500 / 64);
    return Math.min(
      this.firstVisibleIndex() + visibleCount - 1,
      this.filteredContacts().length - 1
    );
  });

  private readonly viewport = viewChild.required(CdkVirtualScrollViewport);

  scrollToTop(): void {
    this.viewport().scrollToIndex(0, 'smooth');
  }

  trackById(_index: number, contact: Contact): number {
    return contact.id;
  }
}
```

## Weiterführendes

- **Dynamische Item-Höhen**: Schau dir `@angular/cdk-experimental` mit `AutoSizeVirtualScrollStrategy` an – nützlich, wenn Items variabler Höhe sind.
- **DataSource-Protokoll**: Für wirklich lazy Daten (API-Pagination) implementiere `CollectionViewer` und `DataSource<T>` – CDK ruft dann `connect()`/`disconnect()` auf und du lädst nur die benötigten Seiten.
- **Infinite Scroll**: Höre auf `scrolledIndexChange` und lade neue Daten nach, wenn der Index nahe am Ende ist – kombiniert gut mit `toObservable(firstVisibleIndex)` + `distinctUntilChanged()` + `switchMap()`.
- Offizielle Docs: [angular.dev/guide/cdk/virtual-scrolling](https://angular.dev/guide/cdk/virtual-scrolling)

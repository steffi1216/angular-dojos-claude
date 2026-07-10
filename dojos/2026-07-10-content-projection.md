# Angular Dojo: Content Projection & Slot-based Component APIs
**Datum:** 2026-07-10
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `ng-content`, `ContentChild`, `ContentChildren` und Multi-Slot-Projection flexible, wiederverwendbare Komponenten erstellst – das Herzstück jeder Angular-Komponentenbibliothek.

## Hintergrund & Theorie

**Content Projection** ermöglicht es, Inhalte von außen in eine Komponente „einzuschmuggeln", ähnlich wie `<slot>` in Web Components.

### Single-Slot vs. Multi-Slot

```html
<!-- Single-Slot: alles landet an einer Stelle -->
<ng-content />

<!-- Multi-Slot: gezieltes Platzieren per CSS-Selektor -->
<ng-content select="[slot=header]" />
<ng-content select=".footer" />
<ng-content />   <!-- default slot für den Rest -->
```

### ContentChild & ContentChildren

`@ContentChild` / `@ContentChildren` greifen auf *projizierte* Kinder zu (nicht auf View-Kinder wie `@ViewChild`).

```typescript
@ContentChild(MyBadgeComponent) badge!: MyBadgeComponent;
@ContentChildren(MyItemComponent) items!: QueryList<MyItemComponent>;
```

Die Lifecycle-Hook `ngAfterContentInit` garantiert, dass diese Queries aufgelöst sind.

### ngProjectAs

Wenn du ein `<ng-template>` oder einen Wrapper als einen bestimmten Selektor projizieren willst:

```html
<ng-container ngProjectAs="[slot=header]">
  <h2>Ich lande im Header-Slot</h2>
</ng-container>
```

> **Wichtig:** Content wird vom *Parent* gerendert, nicht von der Host-Komponente. Das hat Konsequenzen für Change Detection und Styling (`::ng-deep` sollte vermieden werden).

## Aufgabe

Erstelle eine wiederverwendbare `<app-card>` Komponente, die drei benannte Slots unterstützt: `header`, `body` und `footer`. Zusätzlich soll die Komponente per `ContentChildren` alle projizierten `<app-badge>` Elemente zählen und die Anzahl anzeigen.

### Schritte

1. **`BadgeComponent` erstellen** – eine einfache Standalone-Komponente, die einen Text-Input entgegennimmt und ihn in einem `<span class="badge">` anzeigt.

2. **`CardComponent` erstellen** – Standalone, mit drei `ng-content`-Slots (Selektoren: `[slot=header]`, `[slot=body]`, `[slot=footer]`). Zeige darunter einen Hinweis: „X Badge(s) in dieser Karte".

3. **`ContentChildren` verdrahten** – Injiziere `QueryList<BadgeComponent>` in `CardComponent` und abonniere `changes`, damit die Anzeige reaktiv bleibt (auch wenn Badges per `@if` ein- und ausgeblendet werden).

4. **Demo in `AppComponent`** – Nutze `<app-card>` mit allen drei Slots und platziere zwei `<app-badge>` Komponenten darin. Füge einen Button hinzu, der einen dritten Badge per `@if` ein- und ausblendet – die Zählung soll sich aktualisieren.

5. **Bonus:** Verwende `ngProjectAs` in einem `<ng-container>`, um Inhalt ohne zusätzliches DOM-Element in den Header-Slot zu projizieren.

## Hints

<details>
<summary>Hint 1 – Multi-Slot Template</summary>

```html
<!-- card.component.html -->
<div class="card">
  <div class="card__header">
    <ng-content select="[slot=header]" />
  </div>
  <div class="card__body">
    <ng-content select="[slot=body]" />
  </div>
  <div class="card__footer">
    <ng-content select="[slot=footer]" />
  </div>
  <p class="badge-count">{{ badgeCount }} Badge(s)</p>
</div>
```

</details>

<details>
<summary>Hint 2 – ContentChildren + changes</summary>

```typescript
@ContentChildren(BadgeComponent, { descendants: true })
badges!: QueryList<BadgeComponent>;

badgeCount = 0;

ngAfterContentInit() {
  this.badgeCount = this.badges.length;
  this.badges.changes.subscribe((list: QueryList<BadgeComponent>) => {
    this.badgeCount = list.length;
  });
}
```

Das `{ descendants: true }` sorgt dafür, dass auch tief verschachtelte Badges gefunden werden.

</details>

<details>
<summary>Hint 3 – ngProjectAs Bonus</summary>

```html
<!-- In app.component.html -->
<app-card>
  <ng-container ngProjectAs="[slot=header]">
    <h2>Kein extra div im DOM!</h2>
    <app-badge text="Neu" />
  </ng-container>

  <p slot="body">Hauptinhalt hier...</p>

  <button slot="footer">Speichern</button>
</app-card>
```

</details>

## Beispiellösung

```typescript
// badge.component.ts
import { Component, input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-badge',
  standalone: true,
  template: `<span class="badge">{{ text() }}</span>`,
  styles: [`
    .badge {
      display: inline-block;
      background: #3f51b5;
      color: white;
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 0.75rem;
      margin: 2px;
    }
  `],
})
export class BadgeComponent {
  text = input.required<string>();
}
```

```typescript
// card.component.ts
import {
  AfterContentInit,
  Component,
  ContentChildren,
  QueryList,
} from '@angular/core';
import { BadgeComponent } from './badge.component';

@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card__header">
        <ng-content select="[slot=header]" />
      </div>
      <div class="card__body">
        <ng-content select="[slot=body]" />
      </div>
      <div class="card__footer">
        <ng-content select="[slot=footer]" />
      </div>
      <p class="badge-info">{{ badgeCount }} Badge(s) in dieser Karte</p>
    </div>
  `,
  styles: [`
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      overflow: hidden;
      max-width: 400px;
      font-family: sans-serif;
    }
    .card__header { background: #f5f5f5; padding: 12px 16px; font-weight: bold; }
    .card__body   { padding: 16px; }
    .card__footer { background: #f5f5f5; padding: 12px 16px; text-align: right; }
    .badge-info   { margin: 0; padding: 8px 16px; font-size: 0.8rem; color: #666; border-top: 1px solid #eee; }
  `],
})
export class CardComponent implements AfterContentInit {
  @ContentChildren(BadgeComponent, { descendants: true })
  badges!: QueryList<BadgeComponent>;

  badgeCount = 0;

  ngAfterContentInit(): void {
    this.badgeCount = this.badges.length;
    this.badges.changes.subscribe((list: QueryList<BadgeComponent>) => {
      this.badgeCount = list.length;
    });
  }
}
```

```typescript
// app.component.ts
import { Component, signal } from '@angular/core';
import { CardComponent } from './card.component';
import { BadgeComponent } from './badge.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CardComponent, BadgeComponent],
  template: `
    <app-card>
      <ng-container ngProjectAs="[slot=header]">
        <h2 style="margin:0">Mein Profil</h2>
        <app-badge text="Admin" />
      </ng-container>

      <div slot="body">
        <p>Willkommen zurück!</p>
        <app-badge text="Verifiziert" />
        @if (showExtraBadge()) {
          <app-badge text="Premium" />
        }
      </div>

      <button slot="footer" (click)="showExtraBadge.set(!showExtraBadge())">
        Premium {{ showExtraBadge() ? 'entfernen' : 'hinzufügen' }}
      </button>
    </app-card>
  `,
})
export class AppComponent {
  showExtraBadge = signal(false);
}
```

## Weiterführendes

- **`@ContentChild` mit Signal-Queries (Angular 17+):** `contentChild(BadgeComponent)` gibt ein `Signal<BadgeComponent | undefined>` zurück – kein `ngAfterContentInit` mehr nötig. Schau dir `contentChildren()` als modernen Ersatz für `@ContentChildren` an.
- **Styling-Isolation:** Projizierter Content gehört dem Parent-StyleScope. Vermeide `::ng-deep`; nutze stattdessen CSS Custom Properties für theming.
- **Angular Material als Vorbild:** Studiere `MatCard`, `MatDialog` oder `MatTab` – sie alle basieren intensiv auf Multi-Slot Content Projection.
- **Offizielle Docs:** [angular.dev/guide/components/content-projection](https://angular.dev/guide/components/content-projection)

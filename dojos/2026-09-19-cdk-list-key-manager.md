# Angular Dojo: CDK ListKeyManager & Keyboard Navigation
**Datum:** 2026-09-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man mit dem Angular CDK `ListKeyManager` eine vollständig tastaturnavigierbare Liste baut – ein essenzielles Muster für zugängliche Custom Components wie Dropdowns, Autocomplete-Felder oder Toolbars.

## Hintergrund & Theorie

Der Angular CDK enthält das `@angular/cdk/a11y`-Paket mit `ListKeyManager` – einem Service, der Keyboard-Events auf eine Liste von Items abbildet. Er abstrahiert die komplexe Logik der Tastaturnavigation (ArrowUp/Down, Home, End, Tab, Enter) und erlaubt es, den aktuell "aktiven" Eintrag zu verwalten.

Es gibt drei Varianten:
- **`ActiveDescendantKeyManager`**: Setzt `aria-activedescendant` für Screenreader (ideal für `role="listbox"`)
- **`FocusKeyManager`**: Setzt den DOM-Fokus direkt auf das Item (ideal für `role="menu"` oder `role="toolbar"`)
- **`ListKeyManager`** (Basisklasse): Kein automatisches DOM-Handling, vollständig manuell steuerbar

Jedes Item muss das `FocusableOption`-Interface implementieren (beim `FocusKeyManager`) bzw. `Highlightable` (beim `ActiveDescendantKeyManager`). Die Klasse ist `QueryList`-kompatibel und reagiert auf Änderungen der Liste automatisch.

Zusätzliche Features:
- `withWrap()`: Navigation springt am Ende/Anfang zur anderen Seite um
- `withTypeAhead()`: Tippt man einen Buchstaben, springt der Manager zum ersten Item mit diesem Anfangsbuchstaben
- `withHomeAndEnd()`: Aktiviert Home/End-Tastenunterstützung

## Aufgabe

Baue eine accessible Custom-Dropdown-Komponente (`SelectListComponent`), die per Tastatur vollständig bedienbar ist. Die Komponente zeigt eine Liste von Optionen und verwendet `ActiveDescendantKeyManager`, um sowohl Screenreader als auch sehende Nutzer zu unterstützen.

### Schritte

1. **Setup**: Erstelle eine Standalone-Komponente `SelectListComponent`. Füge `@angular/cdk/a11y` als Import hinzu. Definiere ein `OptionItem`-Interface mit `id`, `label` und `disabled?`. Erstelle dazu ein `ListOptionComponent` für jedes Item, das das `Highlightable`-Interface implementiert.

2. **Key Manager initialisieren**: Nutze `@ViewChildren(ListOptionComponent)` für eine `QueryList`. Initialisiere `ActiveDescendantKeyManager` in `ngAfterViewInit` mit `.withWrap()` und `.withTypeAhead()`. Abonniere `keyManager.change` (Signal oder Observable), um das aktuell aktive Item zu verfolgen.

3. **Keyboard-Events weiterleiten**: Binde `(keydown)` auf den Host-Container (`role="listbox"`, `tabindex="0"`). Leite Events mit `keyManager.onKeydown($event)` weiter. Handle `Enter`-/`Space`-Taste manuell zur Selektion.

4. **Template & Aria-Attribute**: Setze `aria-activedescendant` auf die ID des aktiven Items. Setze `aria-selected` und optisch sichtbares Highlighting auf dem aktiven und selektierten Item.

5. **Optionals**: Füge `withHomeAndEnd()` hinzu. Überspringe disabled Items mit einem `skipPredicate`.

## Hints

<details>
<summary>Hint 1 – Highlightable Interface</summary>

`ListOptionComponent` muss das `Highlightable`-Interface aus `@angular/cdk/a11y` implementieren:

```typescript
import { Highlightable } from '@angular/cdk/a11y';

@Component({
  selector: 'app-list-option',
  template: `<div [class.active]="isActive" [id]="id" role="option">{{ label }}</div>`,
  standalone: true,
})
export class ListOptionComponent implements Highlightable {
  @Input() id!: string;
  @Input() label!: string;
  @Input() disabled = false;

  isActive = false;

  setActiveStyles() { this.isActive = true; }
  setInactiveStyles() { this.isActive = false; }
  // Optional – für withTypeAhead():
  getLabel() { return this.label; }
}
```

</details>

<details>
<summary>Hint 2 – Key Manager Setup</summary>

```typescript
import { ActiveDescendantKeyManager } from '@angular/cdk/a11y';
import { AfterViewInit, Component, QueryList, ViewChildren } from '@angular/core';

@Component({ /* ... */ })
export class SelectListComponent implements AfterViewInit {
  @ViewChildren(ListOptionComponent) options!: QueryList<ListOptionComponent>;

  keyManager!: ActiveDescendantKeyManager<ListOptionComponent>;
  activeId = signal<string | null>(null);

  ngAfterViewInit() {
    this.keyManager = new ActiveDescendantKeyManager(this.options)
      .withWrap()
      .withTypeAhead()
      .withHomeAndEnd();

    this.keyManager.change.subscribe(() => {
      const active = this.keyManager.activeItem;
      this.activeId.set(active?.id ?? null);
    });
  }

  onKeydown(event: KeyboardEvent) {
    if (event.key === 'Enter' || event.key === ' ') {
      this.selectActive();
      event.preventDefault();
    } else {
      this.keyManager.onKeydown(event);
    }
  }

  selectActive() {
    const item = this.keyManager.activeItem;
    if (item && !item.disabled) {
      // Selektion-Logik hier
    }
  }
}
```

</details>

<details>
<summary>Hint 3 – skipPredicate für disabled Items</summary>

```typescript
this.keyManager = new ActiveDescendantKeyManager(this.options)
  .withWrap()
  .skipPredicate(item => item.disabled);
```

Dadurch überspringt der Manager beim Navigieren automatisch deaktivierte Einträge.

</details>

## Beispiellösung

```typescript
// list-option.component.ts
import { Component, Input } from '@angular/core';
import { Highlightable } from '@angular/cdk/a11y';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-list-option',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div
      [id]="id"
      role="option"
      [attr.aria-disabled]="disabled || null"
      [attr.aria-selected]="isSelected"
      [class.option--active]="isActive"
      [class.option--selected]="isSelected"
      [class.option--disabled]="disabled"
    >
      {{ label }}
    </div>
  `,
  styles: [`
    div { padding: 8px 16px; cursor: pointer; }
    .option--active { background: #e3f2fd; }
    .option--selected { font-weight: bold; }
    .option--disabled { opacity: 0.4; pointer-events: none; }
  `],
})
export class ListOptionComponent implements Highlightable {
  @Input() id!: string;
  @Input() label!: string;
  @Input() disabled = false;
  @Input() isSelected = false;

  isActive = false;

  setActiveStyles() { this.isActive = true; }
  setInactiveStyles() { this.isActive = false; }
  getLabel() { return this.label; }
}

// select-list.component.ts
import {
  AfterViewInit, Component, EventEmitter, Input,
  OnChanges, Output, QueryList, ViewChildren, signal,
} from '@angular/core';
import { ActiveDescendantKeyManager } from '@angular/cdk/a11y';
import { ListOptionComponent } from './list-option.component';

export interface OptionItem {
  id: string;
  label: string;
  disabled?: boolean;
}

@Component({
  selector: 'app-select-list',
  standalone: true,
  imports: [ListOptionComponent],
  template: `
    <div
      role="listbox"
      tabindex="0"
      [attr.aria-activedescendant]="activeId()"
      [attr.aria-label]="ariaLabel"
      (keydown)="onKeydown($event)"
      style="border: 1px solid #ccc; border-radius: 4px; outline: none; min-width: 200px;"
    >
      @for (option of options; track option.id) {
        <app-list-option
          [id]="option.id"
          [label]="option.label"
          [disabled]="option.disabled ?? false"
          [isSelected]="selectedId === option.id"
          (click)="selectById(option.id)"
        />
      }
    </div>
    @if (selectedId) {
      <p>Ausgewählt: <strong>{{ selectedLabel }}</strong></p>
    }
  `,
})
export class SelectListComponent implements AfterViewInit, OnChanges {
  @Input() options: OptionItem[] = [];
  @Input() ariaLabel = 'Auswahlliste';
  @Output() selectionChange = new EventEmitter<OptionItem>();

  @ViewChildren(ListOptionComponent) optionComponents!: QueryList<ListOptionComponent>;

  keyManager!: ActiveDescendantKeyManager<ListOptionComponent>;
  activeId = signal<string | null>(null);
  selectedId: string | null = null;

  get selectedLabel() {
    return this.options.find(o => o.id === this.selectedId)?.label ?? '';
  }

  ngAfterViewInit() {
    this.keyManager = new ActiveDescendantKeyManager(this.optionComponents)
      .withWrap()
      .withTypeAhead(200)
      .withHomeAndEnd()
      .skipPredicate(item => item.disabled);

    this.keyManager.change.subscribe(() => {
      this.activeId.set(this.keyManager.activeItem?.id ?? null);
    });
  }

  ngOnChanges() {
    // Nach Änderungen der options-Liste den Manager zurücksetzen
    this.keyManager?.setActiveItem(-1);
  }

  onKeydown(event: KeyboardEvent) {
    if (event.key === 'Enter' || event.key === ' ') {
      const item = this.keyManager.activeItem;
      if (item) this.selectById(item.id);
      event.preventDefault();
    } else {
      this.keyManager.onKeydown(event);
    }
  }

  selectById(id: string) {
    const option = this.options.find(o => o.id === id);
    if (option && !option.disabled) {
      this.selectedId = id;
      this.selectionChange.emit(option);
    }
  }
}

// app.component.ts – Verwendung
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [SelectListComponent],
  template: `
    <h2>Wähle ein Framework:</h2>
    <app-select-list
      [options]="frameworks"
      ariaLabel="Framework-Auswahl"
      (selectionChange)="onSelect($event)"
    />
  `,
})
export class AppComponent {
  frameworks: OptionItem[] = [
    { id: 'ng', label: 'Angular' },
    { id: 'react', label: 'React', disabled: true },
    { id: 'vue', label: 'Vue' },
    { id: 'svelte', label: 'Svelte' },
    { id: 'solid', label: 'SolidJS' },
  ];

  onSelect(option: OptionItem) {
    console.log('Gewählt:', option);
  }
}
```

## Weiterführendes

- Für Menüs (`role="menu"`) den `FocusKeyManager` verwenden – er setzt den echten DOM-Fokus, was für flüchtige Overlays wichtiger ist als `aria-activedescendant`
- Die CDK A11y-Dokumentation zeigt, wie man `LiveAnnouncer` kombiniert, um Statusänderungen für Screenreader anzukündigen
- Das Angular Material `MatSelect`-Quellcode ist ein reales Produktionsbeispiel für `ActiveDescendantKeyManager` in einem komplexen Widget

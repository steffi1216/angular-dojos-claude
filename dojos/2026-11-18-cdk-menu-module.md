# Angular Dojo: CdkMenuModule – Barrierefreie Menüs und Kontextmenüs

**Datum:** 2026-11-18  
**Dauer:** ~25 Minuten  
**Level:** Fortgeschritten

## Lernziel

Du lernst, mit dem `CdkMenuModule` aus dem Angular CDK vollständig barrierefreie Menüs, Menüleisten und Kontextmenüs zu bauen – inklusive vollständiger Tastaturnavigation ohne eigenes Keyboard-Handling schreiben zu müssen.

## Hintergrund & Theorie

Das `CdkMenuModule` ist seit Angular CDK 14 stabil und implementiert das [ARIA Menu Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/menu/) vollständig out-of-the-box. Es besteht aus mehreren zusammenwirkenden Direktiven:

- **`cdkMenu`** / **`cdkMenuBar`** – Container-Direktiven für Menüs (vertikale Liste) und Menüleisten (horizontale Leiste)
- **`cdkMenuItem`** – Einzelner Menüeintrag; übernimmt automatisch Fokus- und Keyboard-Handling
- **`cdkMenuTriggerFor`** – Öffnet ein `cdkMenu` per Klick oder Tastatur (Pfeiltasten)
- **`cdkMenuItemRadio`** / **`cdkMenuItemCheckbox`** – Selektierbare Einträge mit ARIA-State
- **`cdkContextMenuTriggerFor`** – Öffnet ein Menü bei Rechtsklick (Kontextmenü)
- **`CdkMenuGroup`** – Logische Gruppierung für Radio/Checkbox-Items

Das CDK kümmert sich um: Fokus-Trap, Pfeiltasten-Navigation, Escape zum Schließen, Submenu-Öffnung via Hover/Pfeiltaste und korrekte `aria-*`-Attribute. Eigenes Keyboard-Handling ist nicht nötig.

Import: `CdkMenuModule` aus `@angular/cdk/menu`.

## Aufgabe

Baue eine **Editor-Toolbar** als Standalone-Component mit einer Menüleiste (`cdkMenuBar`), die drei Menüs enthält:

1. **„Datei"-Menü** mit einfachen Aktionen (Neu, Öffnen, Speichern) und einem Trennstrich
2. **„Ansicht"-Menü** mit `cdkMenuItemCheckbox`-Einträgen (Zeilennummern, Minimap) zur Toggles
3. **„Schriftgröße"-Menü** mit `cdkMenuItemRadio`-Einträgen (Small, Medium, Large)

Zusätzlich: Registriere ein **Kontextmenü** (`cdkContextMenuTriggerFor`) auf einem Textarea-Element, das bei Rechtsklick ein einfaches Menü mit „Ausschneiden", „Kopieren", „Einfügen" zeigt.

Alle Menüzustände (Checkboxen, Radio-Auswahl) sollen als Signals verwaltet werden.

### Schritte

1. Installiere / stelle sicher dass `@angular/cdk` in `package.json` vorhanden ist. Importiere `CdkMenuModule` in deiner Standalone-Component.

2. Erstelle das Component-Template mit `<nav cdkMenuBar>` und drei `<button cdkMenuItem [cdkMenuTriggerFor]="dateiMenu">` Triggern.

3. Definiere die Menü-Templates als `<ng-template>` mit `<div cdkMenu>` und `<button cdkMenuItem>` Einträgen. Füge einen `<hr cdkMenuGroup>` als Separator ein.

4. Implementiere das „Ansicht"-Menü mit `cdkMenuItemCheckbox`. Binde `[cdkMenuItemChecked]` an ein Signal und aktualisiere es im `(cdkMenuItemTriggered)`-Event.

5. Implementiere das „Schriftgröße"-Menü: Wrap alle Radio-Items in `<div cdkMenuGroup>` und nutze `cdkMenuItemRadio` mit `[cdkMenuItemChecked]="fontSize() === 'medium'"`.

6. Füge das Kontextmenü auf dem `<textarea>` hinzu: `[cdkContextMenuTriggerFor]="contextMenu"`. Definiere das Kontext-`<ng-template>` mit drei `cdkMenuItem`-Buttons.

7. Teste: Tab/Enter/Pfeiltasten öffnen Menüs, Escape schließt, Rechtsklick auf Textarea zeigt Kontextmenü.

## Hints

<details>
<summary>Hint 1 – Menü-Template Grundstruktur</summary>

Ein Menü wird als `<ng-template>` außerhalb der sichtbaren DOM-Struktur definiert:

```html
<ng-template #dateiMenu>
  <div cdkMenu class="menu-panel">
    <button cdkMenuItem (cdkMenuItemTriggered)="onNeu()">Neu</button>
    <hr />
    <button cdkMenuItem (cdkMenuItemTriggered)="onSpeichern()">Speichern</button>
  </div>
</ng-template>
```

Der Trigger im MenuBar verweist per Template-Referenz darauf:

```html
<button cdkMenuItem [cdkMenuTriggerFor]="dateiMenu">Datei</button>
```

</details>

<details>
<summary>Hint 2 – Radio-Items korrekt gruppieren</summary>

Für `cdkMenuItemRadio` müssen alle zusammengehörigen Items in **einem** `cdkMenuGroup`-Element liegen, damit das CDK sie als Radio-Gruppe behandelt:

```html
<ng-template #schriftMenu>
  <div cdkMenu class="menu-panel">
    <div cdkMenuGroup>
      <button cdkMenuItemRadio
              [cdkMenuItemChecked]="fontSize() === 'small'"
              (cdkMenuItemTriggered)="fontSize.set('small')">
        Klein
      </button>
      <button cdkMenuItemRadio
              [cdkMenuItemChecked]="fontSize() === 'medium'"
              (cdkMenuItemTriggered)="fontSize.set('medium')">
        Mittel
      </button>
      <button cdkMenuItemRadio
              [cdkMenuItemChecked]="fontSize() === 'large'"
              (cdkMenuItemTriggered)="fontSize.set('large')">
        Groß
      </button>
    </div>
  </div>
</ng-template>
```

</details>

<details>
<summary>Hint 3 – Kontextmenü auf beliebigem Element</summary>

`cdkContextMenuTriggerFor` kann auf jedem Host-Element stehen:

```html
<textarea
  [cdkContextMenuTriggerFor]="contextMenu"
  placeholder="Rechtsklick für Kontextmenü...">
</textarea>

<ng-template #contextMenu>
  <div cdkMenu class="menu-panel">
    <button cdkMenuItem>Ausschneiden</button>
    <button cdkMenuItem>Kopieren</button>
    <button cdkMenuItem>Einfügen</button>
  </div>
</ng-template>
```

Das CDK positioniert das Menü automatisch an der Mausposition.

</details>

## Beispiellösung

```typescript
import { Component, signal } from '@angular/core';
import { CdkMenuModule } from '@angular/cdk/menu';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-editor-toolbar',
  standalone: true,
  imports: [CdkMenuModule, CommonModule],
  styles: [`
    nav[cdkMenuBar] {
      display: flex;
      gap: 4px;
      background: #1e1e1e;
      padding: 4px 8px;
    }
    button[cdkMenuItem], button[cdkMenuItemRadio], button[cdkMenuItemCheckbox] {
      background: none;
      border: none;
      color: #ccc;
      padding: 4px 10px;
      cursor: pointer;
      border-radius: 3px;
      text-align: left;
      width: 100%;
    }
    button[cdkMenuItem]:hover, button[cdkMenuItemRadio]:hover,
    button[cdkMenuItemCheckbox]:hover {
      background: #2a2d2e;
      color: #fff;
    }
    button[cdkMenuBar] { font-size: 13px; }
    .menu-panel {
      background: #252526;
      border: 1px solid #454545;
      border-radius: 4px;
      padding: 4px 0;
      min-width: 160px;
      box-shadow: 0 4px 12px rgba(0,0,0,.4);
    }
    .menu-panel hr { border-color: #454545; margin: 4px 0; }
    .checked-indicator { margin-right: 6px; display: inline-block; width: 12px; }
    textarea { display: block; margin-top: 12px; width: 400px; height: 200px; }
  `],
  template: `
    <nav cdkMenuBar>
      <button cdkMenuItem [cdkMenuTriggerFor]="dateiMenu">Datei</button>
      <button cdkMenuItem [cdkMenuTriggerFor]="ansichtMenu">Ansicht</button>
      <button cdkMenuItem [cdkMenuTriggerFor]="schriftMenu">Schriftgröße</button>
    </nav>

    <!-- Datei-Menü -->
    <ng-template #dateiMenu>
      <div cdkMenu class="menu-panel">
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Neu')">Neu</button>
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Öffnen')">Öffnen...</button>
        <hr />
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Speichern')">Speichern</button>
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Speichern unter')">
          Speichern unter...
        </button>
      </div>
    </ng-template>

    <!-- Ansicht-Menü mit Checkboxen -->
    <ng-template #ansichtMenu>
      <div cdkMenu class="menu-panel">
        <button cdkMenuItemCheckbox
                [cdkMenuItemChecked]="showLineNumbers()"
                (cdkMenuItemTriggered)="showLineNumbers.update(v => !v)">
          <span class="checked-indicator">{{ showLineNumbers() ? '✓' : '' }}</span>
          Zeilennummern
        </button>
        <button cdkMenuItemCheckbox
                [cdkMenuItemChecked]="showMinimap()"
                (cdkMenuItemTriggered)="showMinimap.update(v => !v)">
          <span class="checked-indicator">{{ showMinimap() ? '✓' : '' }}</span>
          Minimap
        </button>
      </div>
    </ng-template>

    <!-- Schriftgröße-Menü mit Radio-Items -->
    <ng-template #schriftMenu>
      <div cdkMenu class="menu-panel">
        <div cdkMenuGroup>
          @for (size of fontSizes; track size.value) {
            <button cdkMenuItemRadio
                    [cdkMenuItemChecked]="fontSize() === size.value"
                    (cdkMenuItemTriggered)="fontSize.set(size.value)">
              <span class="checked-indicator">{{ fontSize() === size.value ? '●' : '' }}</span>
              {{ size.label }}
            </button>
          }
        </div>
      </div>
    </ng-template>

    <!-- Kontextmenü auf Textarea -->
    <textarea
      [cdkContextMenuTriggerFor]="contextMenu"
      placeholder="Rechtsklick für Kontextmenü...">
    </textarea>

    <ng-template #contextMenu>
      <div cdkMenu class="menu-panel">
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Ausschneiden')">Ausschneiden</button>
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Kopieren')">Kopieren</button>
        <button cdkMenuItem (cdkMenuItemTriggered)="log('Einfügen')">Einfügen</button>
      </div>
    </ng-template>

    <p>Zeilennummern: {{ showLineNumbers() }} | Minimap: {{ showMinimap() }} | Font: {{ fontSize() }}</p>
  `
})
export class EditorToolbarComponent {
  showLineNumbers = signal(true);
  showMinimap = signal(false);
  fontSize = signal<'small' | 'medium' | 'large'>('medium');

  fontSizes = [
    { value: 'small' as const, label: 'Klein (12px)' },
    { value: 'medium' as const, label: 'Mittel (14px)' },
    { value: 'large' as const, label: 'Groß (16px)' },
  ];

  log(action: string) {
    console.log('Aktion:', action);
  }
}
```

## Weiterführendes

- **Submenüs (Nested Menus):** Ein `cdkMenuItem` kann selbst wieder `[cdkMenuTriggerFor]` tragen – das CDK öffnet das Submenü bei Hover oder Pfeil-Rechts automatisch und setzt korrekte `aria-haspopup`/`aria-expanded`-Attribute.
- **Inline-Menüs:** `cdkMenu` ohne Trigger-Kontext rendert direkt im DOM (z. B. für Sidebar-Navigation) – einfach `<div cdkMenu>` ohne umgebendes Template verwenden.
- **Offizielle Docs:** [material.angular.io/cdk/menu/overview](https://material.angular.io/cdk/menu/overview)

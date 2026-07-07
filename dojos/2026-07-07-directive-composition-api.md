# Angular Dojo: Directive Composition API
**Datum:** 2026-07-07
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit der **Directive Composition API** (`hostDirectives`) mehrere wiederverwendbare Verhaltensbausteine zu einem Komponenten- oder Direktiven-Feature zusammensetzt, ohne Vererbung oder manuelle Inputs weiterzuverdrahten.

## Hintergrund & Theorie

Seit Angular 15 erlaubt die Directive Composition API, Direktiven direkt im `@Component`- oder `@Directive`-Decorator über das Feld `hostDirectives` einzubinden. Das bedeutet: Ein Host-Element kann die Logik mehrerer Direktiven automatisch aktivieren — inklusive selektivem Weiterleiten von `inputs` und `outputs`.

**Warum ist das nützlich?**

Klassische Ansätze leiden unter typischen Problemen:
- **Vererbung** (`extends`) bricht das Kompositionsprinzip und führt zu starker Kopplung.
- **Manuell weitergeleitete Inputs** erzeugen Boilerplate-Code.

Mit `hostDirectives` wird Verhalten kompositionell: Jede Direktive kümmert sich um genau eine Sache (z. B. Tooltip, Hover-State, Ripple-Effekt), und eine Komponente aggregiert diese, ohne selbst den Code zu duplizieren.

```typescript
@Component({
  selector: 'app-button',
  hostDirectives: [
    RippleDirective,
    { directive: TooltipDirective, inputs: ['tooltip: title'] },
  ],
})
export class ButtonComponent {}
```

- `inputs` und `outputs` können per Alias nach außen weitergegeben werden.
- Die Host-Direktiven werden im `ngOnInit`-Lifecycle der Komponente vollständig initialisiert.
- Sie sind über Dependency Injection innerhalb der Komponente injizierbar.

## Aufgabe

Erstelle eine kleine Angular-Anwendung mit zwei wiederverwendbaren Direktiven, die du über `hostDirectives` in einer Karte (`CardComponent`) zusammensetzt:

1. **`HighlightDirective`** – färbt den Hintergrund eines Elements bei Hover ein (via HostListener).
2. **`BadgeDirective`** – zeigt ein konfigurierbares Zahl-Badge (via `input`) oben rechts an, indem es ein `::after`-Pseudo-Element über Host-Binding setzt.

Die `CardComponent` soll beide Direktiven über `hostDirectives` einbinden und den `badge`-Input von `BadgeDirective` als eigenen `badgeCount`-Input nach außen weiterleiten.

### Schritte

1. Erstelle `HighlightDirective` als Standalone-Direktive mit einem `@HostListener('mouseenter')` und `@HostListener('mouseleave')`, der die Host-Hintergrundfarbe per `@HostBinding` wechselt.
2. Erstelle `BadgeDirective` als Standalone-Direktive mit einem Signal-Input `badge = input<number>(0)` und einem `@HostBinding('attr.data-badge')`, der den Input-Wert überträgt. Setze per `:host([data-badge])::after` in den Styles den Badge-Text.
3. Erstelle `CardComponent` als Standalone-Komponente. Binde beide Direktiven über `hostDirectives` ein und leite `badge` als `badgeCount` weiter.
4. Nutze `CardComponent` in `AppComponent` und übergib verschiedene `badgeCount`-Werte.

## Hints

<details>
<summary>Hint 1 – hostDirectives mit Input-Alias</summary>

```typescript
hostDirectives: [
  HighlightDirective,
  {
    directive: BadgeDirective,
    inputs: ['badge: badgeCount'],
  },
]
```

Der Syntax `'internerName: externerName'` erstellt einen Alias: außen schreibt man `badgeCount`, die Direktive erhält intern `badge`.

</details>

<details>
<summary>Hint 2 – Badge via CSS attr()</summary>

Damit das `::after`-Pseudo-Element den Wert des Attributes anzeigt, nutze in den Styles der Direktive (oder global):

```css
:host([data-badge])::after {
  content: attr(data-badge);
  position: absolute;
  top: -8px;
  right: -8px;
  background: red;
  color: white;
  border-radius: 50%;
  padding: 2px 6px;
  font-size: 12px;
}
```

Vergiss nicht `position: relative` am Host-Element.

</details>

<details>
<summary>Hint 3 – Signal-Input in HostBinding</summary>

`@HostBinding` und Signal-Inputs (`input()`) harmonieren nicht direkt. Nutze stattdessen `effect()` oder eine `computed()`-Eigenschaft in Kombination mit `Renderer2`, oder verwende einen klassischen `@Input()` und `@HostBinding` auf eine Klassen-Property:

```typescript
@Input() badge = 0;
@HostBinding('attr.data-badge') get badgeAttr() {
  return this.badge || null;
}
```

`null` entfernt das Attribut, wenn kein Badge nötig ist.

</details>

## Beispiellösung

```typescript
// highlight.directive.ts
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @HostBinding('style.backgroundColor') bg = 'transparent';

  @HostListener('mouseenter') onEnter() {
    this.bg = '#e8f4fd';
  }

  @HostListener('mouseleave') onLeave() {
    this.bg = 'transparent';
  }
}

// badge.directive.ts
import { Directive, HostBinding, Input } from '@angular/core';

@Directive({
  selector: '[appBadge]',
  standalone: true,
  styles: [`
    :host {
      position: relative;
      display: inline-block;
    }
    :host([data-badge])::after {
      content: attr(data-badge);
      position: absolute;
      top: -8px;
      right: -8px;
      background: #e53935;
      color: white;
      border-radius: 50%;
      min-width: 20px;
      height: 20px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 11px;
      font-weight: bold;
    }
  `],
})
export class BadgeDirective {
  @Input() badge: number | null = null;

  @HostBinding('attr.data-badge')
  get badgeAttr(): number | null {
    return this.badge ?? null;
  }
}

// card.component.ts
import { Component, Input } from '@angular/core';
import { HighlightDirective } from './highlight.directive';
import { BadgeDirective } from './badge.directive';

@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card-content">
      <ng-content />
    </div>
  `,
  styles: [`
    :host {
      display: block;
      border: 1px solid #ccc;
      border-radius: 8px;
      padding: 16px;
      cursor: pointer;
      transition: background-color 0.2s;
    }
  `],
  hostDirectives: [
    HighlightDirective,
    {
      directive: BadgeDirective,
      inputs: ['badge: badgeCount'],
    },
  ],
})
export class CardComponent {}

// app.component.ts
import { Component } from '@angular/core';
import { CardComponent } from './card.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CardComponent],
  template: `
    <h2>Directive Composition API Demo</h2>

    <app-card [badgeCount]="3" style="margin: 24px; max-width: 200px;">
      Benachrichtigungen
    </app-card>

    <app-card [badgeCount]="null" style="margin: 24px; max-width: 200px;">
      Keine Benachrichtigungen
    </app-card>

    <app-card [badgeCount]="count()" style="margin: 24px; max-width: 200px;">
      Dynamisch: {{ count() }}
      <br />
      <button (click)="inc()">+1</button>
    </app-card>
  `,
})
export class AppComponent {
  private _count = 0;
  count = () => this._count;
  inc() { this._count++; }
}
```

## Weiterführendes

- **Outputs weiterleiten:** Genau wie `inputs` können auch `outputs` im `hostDirectives`-Array per Alias exponiert werden — ideal für Klick- oder Fokus-Events, die die Direktive kapselt.
- **DI in der Komponente:** Injiziere eine `hostDirective` direkt per Constructor: `constructor(private badge: BadgeDirective) {}` — so kann die Komponente auf Direktiven-Zustand reagieren.
- **Offizielle Docs:** [angular.dev – Directive composition API](https://angular.dev/guide/directives/directive-composition-api)
- **Tipp:** Kombiniere die Composition API mit dem **CDK** (z. B. `CdkDrag`, `CdkDropList`), um interaktive Komponenten ohne Boilerplate zu bauen.

# Angular Dojo: HostListener & HostBinding
**Datum:** 2026-08-04
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne, wie du mit `@HostListener` und `@HostBinding` (sowie ihrer modernen `host`-Objekt-Alternative) auf Ereignisse des Host-Elements reagierst und dessen Eigenschaften, Klassen und Styles direkt bindest – ohne Template-Boilerplate in der nutzenden Komponente.

## Hintergrund & Theorie

Wenn du eine Direktive oder eine Komponente schreibst, ist der Host das DOM-Element, auf das sie angewendet wird. Angular bietet zwei klassische Dekoratoren und eine modernere Alternative, um dieses Element zu steuern:

**`@HostBinding(property)`** bindet einen Klassenpfad in der Direktive an eine Eigenschaft, Klasse (`class.active`) oder Style (`style.color`) des Host-Elements. Es ist das Äquivalent von `[disabled]` oder `[class.active]` direkt im Template.

**`@HostListener(event, args?)`** registriert einen Event-Listener auf dem Host-Element (oder `window`/`document`/`body` als optionalen ersten Parameter). Es ist das Äquivalent von `(click)` im Template.

**Moderne Alternative – `host`-Objekt im Dekorator:**
```typescript
@Directive({
  selector: '[appFoo]',
  host: {
    '(click)': 'onClick()',
    '[class.active]': 'isActive',
  }
})
```
Dieses deklarative `host`-Objekt ist der empfohlene Stil seit Angular 15+, da es statisch analysierbar ist, keinen Overhead durch Reflection erzeugt und gut mit dem Angular-Compiler zusammenarbeitet.

Beide Ansätze funktionieren mit Standalone Directives und sind SSR-kompatibel, weil Angular den Renderer-Layer nutzt.

## Aufgabe

Erstelle eine wiederverwendbare `appHighlight`-Direktive, die ein beliebiges Element interaktiv macht:

- Bei `mouseenter` wird das Element farblich hervorgehoben und eine CSS-Klasse `is-highlighted` gesetzt.
- Bei `mouseleave` wird die Hervorhebung entfernt.
- Bei `click` wird eine `is-selected`-Klasse getoggled und ein Output-Event `selected` emittiert.
- Die Highlight-Farbe soll als Input konfigurierbar sein (Default: `'lightyellow'`).
- Nutze für einen Teil der Bindings den deklarativen `host`-Ansatz, für den anderen Teil `@HostListener` / `@HostBinding`.

### Schritte

1. Erstelle eine neue Standalone-Direktive `HighlightDirective` (`selector: '[appHighlight]'`).
2. Definiere ein `@Input() highlightColor = 'lightyellow'` und einen `@Output() selected = new EventEmitter<void>()`.
3. Verwende `@HostBinding('style.backgroundColor')` für die Hintergrundfarbe und `@HostBinding('class.is-highlighted')` für die CSS-Klasse.
4. Implementiere `@HostListener('mouseenter')` und `@HostListener('mouseleave')`, um `isHighlighted` zu setzen.
5. Ergänze `@HostListener('click')` zum Toggling von `isSelected` und Emittieren des Events.
6. Migriere anschließend *einen* der `@HostListener`-Einträge (z. B. `click`) in das `host`-Objekt des `@Directive`-Dekorators und vergewissere dich, dass alles weiterhin funktioniert.
7. Verwende die Direktive in einer Demo-Komponente auf mindestens zwei verschiedenen Elementen (`<p>` und `<button>`).

## Hints

<details>
<summary>Hint 1 – HostBinding mit bedingter Klasse</summary>

```typescript
@HostBinding('class.is-highlighted') get highlightClass() {
  return this.isHighlighted;
}
// oder als Getter kürzer:
@HostBinding('class.is-highlighted') isHighlighted = false;
```
`@HostBinding` akzeptiert direkt ein Klassenpräfix `class.name` – Angular setzt oder entfernt die Klasse je nach Wahrheitswert.

</details>

<details>
<summary>Hint 2 – host-Objekt mit Output-Event</summary>

Im `host`-Objekt kannst du auch auf Events reagieren und Methoden aufrufen:

```typescript
@Directive({
  selector: '[appHighlight]',
  host: {
    '(click)': 'onHostClick()',
    '[class.is-selected]': 'isSelected',
  }
})
export class HighlightDirective {
  isSelected = false;

  onHostClick() {
    this.isSelected = !this.isSelected;
    this.selected.emit();
  }
}
```
`host` unterstützt dieselbe Bindungssyntax wie Angular-Templates: `[prop]`, `(event)`, `[class.x]`, `[style.x]`.

</details>

## Beispiellösung

```typescript
// highlight.directive.ts
import {
  Directive,
  HostBinding,
  HostListener,
  Input,
  Output,
  EventEmitter,
} from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
  // Moderne host-Syntax für den Click-Handler
  host: {
    '(click)': 'onHostClick()',
    '[class.is-selected]': 'isSelected',
    '[attr.tabindex]': '"0"',
  },
})
export class HighlightDirective {
  @Input() highlightColor = 'lightyellow';
  @Output() selected = new EventEmitter<void>();

  isHighlighted = false;
  isSelected = false;

  @HostBinding('style.backgroundColor')
  get bgColor(): string {
    return this.isHighlighted ? this.highlightColor : '';
  }

  @HostBinding('class.is-highlighted')
  get highlightClass(): boolean {
    return this.isHighlighted;
  }

  @HostListener('mouseenter')
  onEnter(): void {
    this.isHighlighted = true;
  }

  @HostListener('mouseleave')
  onLeave(): void {
    this.isHighlighted = false;
  }

  // Aufgerufen durch host-Objekt
  onHostClick(): void {
    this.isSelected = !this.isSelected;
    this.selected.emit();
  }
}
```

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { HighlightDirective } from './highlight.directive';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HighlightDirective],
  template: `
    <p appHighlight highlightColor="lightblue" (selected)="onSelected('Absatz')">
      Hover oder klicke mich (Absatz)
    </p>

    <button appHighlight highlightColor="lightgreen" (selected)="onSelected('Button')">
      Hover oder klicke mich (Button)
    </button>

    <p>Letztes Event: {{ lastEvent }}</p>
  `,
  styles: [`
    .is-selected { outline: 2px solid steelblue; }
    .is-highlighted { transition: background-color 0.2s; }
  `],
})
export class AppComponent {
  lastEvent = '–';

  onSelected(source: string): void {
    this.lastEvent = `${source} ausgewählt`;
  }
}
```

## Weiterführendes

- **`host`-Objekt vs. Dekoratoren**: Das offizielle Angular Style Guide empfiehlt seit v15+ das `host`-Objekt, da es zur Compile-Zeit analysiert werden kann. `@HostListener`/`@HostBinding` bleiben gültig, erzeugen aber mehr Reflection-Overhead.
- **Keyboard-Zugänglichkeit**: Kombiniere `@HostListener('keydown.enter')` mit `[attr.tabindex]="0"`, um nicht-interaktive Elemente (z. B. `<div>`) tastaturzugänglich zu machen.
- **Signal-Äquivalent**: Ab Angular 17+ kannst du `HostAttributeToken` und `input()`/`model()` kombinieren. Für Host-Events gibt es bisher kein Signal-Äquivalent; `@HostListener` bleibt hier der Standard.
- Docs: [angular.dev/guide/directives](https://angular.dev/guide/directives)

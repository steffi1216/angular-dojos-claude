# Angular Dojo: Angular Elements – Web Components mit Angular
**Datum:** 2026-07-11
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Komponenten mithilfe von `@angular/elements` in native Custom Elements (Web Components) verwandelst, die framework-unabhängig in jedem HTML-Kontext eingesetzt werden können.

## Hintergrund & Theorie

**Angular Elements** ermöglicht es, Angular-Komponenten als Custom Elements (`CustomElementRegistry`) zu registrieren. Ein Custom Element ist ein von der Browser-API definierter, erweiterbarer HTML-Tag, der eigene Logik kapselt.

### Warum Angular Elements?
- Wiederverwendung von Angular-Komponenten in React-, Vue- oder Vanilla-JS-Projekten
- Einbindung in Legacy-Systeme ohne vollständiges Angular
- Micro-Frontend-Ansätze, bei denen Teams unterschiedliche Frameworks nutzen

### Funktionsweise

```
Angular Component → createCustomElement() → browser.customElements.define() → <my-widget>
```

Der Kern ist die Funktion `createCustomElement()` aus `@angular/elements`. Sie nimmt eine Komponente und einen `Injector` entgegen und gibt eine Klasse zurück, die die `HTMLElement`-API implementiert:

- `@Input()`-Properties werden zu HTML-Attributen / DOM-Properties
- `@Output()` EventEmitter werden zu nativen DOM-CustomEvents
- Der Lifecycle (ngOnInit, ngOnDestroy) wird auf die Custom Element Callbacks (`connectedCallback`, `disconnectedCallback`) gemappt

Seit Angular 14+ mit Standalone Components ist die Integration besonders schlank – keine NgModule-Boilerplate nötig.

## Aufgabe

Erstelle ein wiederverwendbares `<rating-widget>`-Custom Element in Angular, das:
1. Eine Sternebewertung (1–5) rendert
2. Den aktuellen Wert über ein `@Input()` empfängt
3. Beim Klick auf einen Stern ein nativer DOM-Event (`CustomEvent`) auslöst
4. Das Custom Element im Browser-globalen Registry registriert

Das Element soll danach in einer einfachen `index.html` **ohne Angular-Bootstrap** funktionieren.

### Schritte

1. **Setup**: Installiere `@angular/elements` und erstelle eine neue Standalone-Komponente `RatingWidgetComponent`.

2. **Komponente implementieren**: Rendere 5 Sterne-Buttons. Nimm `value: number` als `@Input()` und emitte `ratingChange` als `@Output()`.

3. **Custom Element erzeugen**: Erstelle eine `main.ts`-ähnliche Bootstrapping-Datei, die `createCustomElement()` und `customElements.define()` aufruft.

4. **Ausprobieren**: Verwende das Element in einer `index.html`:
   ```html
   <rating-widget value="3"></rating-widget>
   <script>
     document.querySelector('rating-widget')
       .addEventListener('ratingChange', e => console.log(e.detail));
   </script>
   ```

5. **Bonus**: Setze `encapsulation: ViewEncapsulation.ShadowDom` und beobachte, wie Shadow DOM das Styling isoliert.

## Hints

<details>
<summary>Hint 1 – createCustomElement verwenden</summary>

```typescript
import { createCustomElement } from '@angular/elements';
import { createApplication } from '@angular/platform-browser';

(async () => {
  const appRef = await createApplication();
  const RatingElement = createCustomElement(RatingWidgetComponent, {
    injector: appRef.injector,
  });
  customElements.define('rating-widget', RatingElement);
})();
```

`createApplication()` (Angular 14+) bootstrapt Angular ohne Root-Komponente – ideal für reine Element-Bundles.
</details>

<details>
<summary>Hint 2 – Input/Output Mapping</summary>

Angular mapped `@Input()` automatisch auf camelCase-Attribute (z. B. `maxValue` → Attribut `max-value`).

Für `@Output()` gilt: Der EventEmitter-Name wird zum CustomEvent-Namen. Um `detail` im Event zu setzen, emitte einfach den Wert:

```typescript
@Output() ratingChange = new EventEmitter<number>();

selectStar(star: number): void {
  this.ratingChange.emit(star);
}
```

Im DOM empfängst du ihn so:
```javascript
element.addEventListener('ratingChange', (e: CustomEvent) => {
  console.log(e.detail); // der emittierte Wert
});
```
</details>

<details>
<summary>Hint 3 – Shadow DOM Encapsulation</summary>

```typescript
@Component({
  selector: 'app-rating-widget',
  encapsulation: ViewEncapsulation.ShadowDom,
  // ...
})
export class RatingWidgetComponent {}
```

Mit Shadow DOM sind deine CSS-Regeln komplett isoliert – externe Styles kommen nicht rein, interne nicht raus. Nutze `:host` für Styles am Element selbst.
</details>

## Beispiellösung

```typescript
// rating-widget.component.ts
import { Component, Input, Output, EventEmitter, ViewEncapsulation } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-rating-widget',
  standalone: true,
  imports: [CommonModule],
  encapsulation: ViewEncapsulation.ShadowDom,
  template: `
    <div class="stars">
      @for (star of stars; track star) {
        <button
          class="star"
          [class.active]="star <= currentValue"
          (click)="selectStar(star)"
          [attr.aria-label]="'Bewertung ' + star + ' von 5'"
        >★</button>
      }
    </div>
  `,
  styles: [`
    :host { display: inline-flex; font-size: 1.5rem; }
    .star { background: none; border: none; cursor: pointer; color: #ccc; padding: 0 2px; }
    .star.active { color: gold; }
    .star:hover, .star:hover ~ .star { color: #e0e0e0; }
  `],
})
export class RatingWidgetComponent {
  stars = [1, 2, 3, 4, 5];
  currentValue = 0;

  @Input()
  set value(v: number | string) {
    this.currentValue = Number(v);
  }

  @Output() ratingChange = new EventEmitter<number>();

  selectStar(star: number): void {
    this.currentValue = star;
    this.ratingChange.emit(star);
  }
}
```

```typescript
// main.ts (Element-Bootstrap)
import { createApplication } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { RatingWidgetComponent } from './rating-widget.component';

(async () => {
  const app = await createApplication({
    providers: [], // globale Provider hier
  });

  const RatingElement = createCustomElement(RatingWidgetComponent, {
    injector: app.injector,
  });

  customElements.define('rating-widget', RatingElement);
})();
```

```html
<!-- index.html – kein Angular-Bootstrapping nötig -->
<!DOCTYPE html>
<html lang="de">
<body>
  <h1>Bewerte diesen Artikel</h1>
  <rating-widget value="2"></rating-widget>

  <script type="module" src="main.js"></script>
  <script>
    document.querySelector('rating-widget')
      .addEventListener('ratingChange', (e) => {
        console.log('Neue Bewertung:', e.detail);
      });
  </script>
</body>
</html>
```

## Weiterführendes

- **Build als einzelnes Bundle**: Mit `ng build --output-hashing=none` und einem Post-Build-Script lassen sich `main.js` und `styles.css` zu einer einzelnen `rating-widget.js`-Datei zusammenführen, die per `<script>`-Tag eingebunden wird.
- **Mehrere Elemente pro App**: Du kannst mehrere Custom Elements aus einer einzigen Angular-Applikation registrieren, indem du `createCustomElement()` mehrfach aufrufst.
- **Offizielle Doku**: [angular.dev/guide/elements](https://angular.dev/guide/elements)
- **Interop-Tip**: In React 19+ können Custom Elements direkt als JSX-Tags verwendet werden – `<rating-widget value={3} onRatingChange={handler} />`.

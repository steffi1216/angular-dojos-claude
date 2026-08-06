# Angular Dojo: Renderer2 & Platform-agnostic DOM-Manipulation
**Datum:** 2026-08-06
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit Angulars `Renderer2`-API DOM-Manipulationen plattformunabhängig durchführst – also sowohl im Browser als auch in SSR-Umgebungen (Angular Universal / Server-Side Rendering) korrekt und sicher.

## Hintergrund & Theorie

Direkter DOM-Zugriff über `document`, `element.nativeElement.style` oder `innerHTML` ist in Angular ein Anti-Pattern: Er bricht bei Server-Side Rendering (kein DOM auf dem Server), bei Web Workers und beim Einsatz in nicht-browser Umgebungen.

`Renderer2` ist Angulars Abstraktionsschicht für DOM-Operationen. Angular wechselt intern die Implementierung je nach Plattform (`BrowserRenderer2`, `ServerRenderer2` usw.), während dein Code unverändert bleibt.

**Wichtige Methoden:**
| Methode | Beschreibung |
|---|---|
| `createElement(name)` | Neues Element erzeugen |
| `appendChild(parent, child)` | Kind-Element anhängen |
| `setStyle(el, style, value)` | Inline-Style setzen |
| `removeStyle(el, style)` | Inline-Style entfernen |
| `addClass(el, name)` | CSS-Klasse hinzufügen |
| `removeClass(el, name)` | CSS-Klasse entfernen |
| `setAttribute(el, name, value)` | Attribut setzen |
| `removeAttribute(el, name)` | Attribut entfernen |
| `setProperty(el, name, value)` | Eigenschaft setzen |
| `listen(el, event, callback)` | Event-Listener registrieren (gibt Unlisten-Fn zurück) |

`Renderer2` wird per Dependency Injection bereitgestellt und funktioniert in Komponenten, Direktiven und Services mit `inject()`.

## Aufgabe

Erstelle eine wiederverwendbare **`HighlightOnFocusDirective`**, die ein beliebiges Eingabefeld beim Fokussieren hervorhebt – vollständig mit `Renderer2`, ohne direkten DOM-Zugriff. Die Direktive soll:

1. Beim Fokus einen konfigurierbaren Hintergrundfarbe und einen Ring-Outline setzen.
2. Beim Blur alle gesetzten Stile wieder entfernen.
3. Optional ein `aria-describedby`-Attribut setzen, wenn ein `helpId`-Input übergeben wird.
4. In einem `<input>` und einem `<textarea>` funktionieren.

### Schritte

1. Erzeuge eine Standalone-Direktive `HighlightOnFocusDirective` mit dem Selektor `[appHighlightOnFocus]`.
2. Injiziere `Renderer2` und `ElementRef` via `inject()`.
3. Definiere zwei Inputs: `highlightColor` (default: `'#fffde7'`) und `helpId` (optional, `string | null`).
4. Registriere `focus`- und `blur`-Events über `renderer.listen(...)` im Konstruktor oder in `ngOnInit`.
5. Im `focus`-Handler: setze `background-color` und `outline` via `renderer.setStyle(...)`, setze `aria-describedby` via `renderer.setAttribute(...)` wenn `helpId` vorhanden ist.
6. Im `blur`-Handler: entferne die Styles mit `renderer.removeStyle(...)` und entferne das Attribut mit `renderer.removeAttribute(...)`.
7. Räume den Event-Listener im `ngOnDestroy` auf (die `listen`-Methode gibt eine Unlisten-Funktion zurück).
8. Verwende die Direktive in einer Demo-Komponente auf einem `<input>` und einem `<textarea>`.

## Hints

<details>
<summary>Hint 1 – Renderer2 und ElementRef injizieren</summary>

```typescript
import { Directive, ElementRef, inject, OnDestroy, OnInit, input } from '@angular/core';
import { Renderer2 } from '@angular/core';

@Directive({
  selector: '[appHighlightOnFocus]',
  standalone: true,
})
export class HighlightOnFocusDirective implements OnInit, OnDestroy {
  private readonly renderer = inject(Renderer2);
  private readonly el = inject(ElementRef);

  highlightColor = input<string>('#fffde7');
  helpId = input<string | null>(null);

  private unlistenFocus!: () => void;
  private unlistenBlur!: () => void;
  // ...
}
```
</details>

<details>
<summary>Hint 2 – Event-Listener mit renderer.listen und Cleanup</summary>

```typescript
ngOnInit(): void {
  const nativeEl = this.el.nativeElement;

  this.unlistenFocus = this.renderer.listen(nativeEl, 'focus', () => {
    this.renderer.setStyle(nativeEl, 'background-color', this.highlightColor());
    this.renderer.setStyle(nativeEl, 'outline', '2px solid #1976d2');
    this.renderer.setStyle(nativeEl, 'outline-offset', '2px');
    const id = this.helpId();
    if (id) {
      this.renderer.setAttribute(nativeEl, 'aria-describedby', id);
    }
  });

  this.unlistenBlur = this.renderer.listen(nativeEl, 'blur', () => {
    this.renderer.removeStyle(nativeEl, 'background-color');
    this.renderer.removeStyle(nativeEl, 'outline');
    this.renderer.removeStyle(nativeEl, 'outline-offset');
    this.renderer.removeAttribute(nativeEl, 'aria-describedby');
  });
}

ngOnDestroy(): void {
  this.unlistenFocus?.();
  this.unlistenBlur?.();
}
```
</details>

## Beispiellösung

```typescript
// highlight-on-focus.directive.ts
import {
  Directive,
  ElementRef,
  OnDestroy,
  OnInit,
  Renderer2,
  inject,
  input,
} from '@angular/core';

@Directive({
  selector: '[appHighlightOnFocus]',
  standalone: true,
})
export class HighlightOnFocusDirective implements OnInit, OnDestroy {
  private readonly renderer = inject(Renderer2);
  private readonly el = inject(ElementRef);

  highlightColor = input<string>('#fffde7');
  helpId = input<string | null>(null);

  private unlistenFocus!: () => void;
  private unlistenBlur!: () => void;

  ngOnInit(): void {
    const nativeEl = this.el.nativeElement;

    this.unlistenFocus = this.renderer.listen(nativeEl, 'focus', () => {
      this.renderer.setStyle(nativeEl, 'background-color', this.highlightColor());
      this.renderer.setStyle(nativeEl, 'outline', '2px solid #1976d2');
      this.renderer.setStyle(nativeEl, 'outline-offset', '2px');

      const id = this.helpId();
      if (id) {
        this.renderer.setAttribute(nativeEl, 'aria-describedby', id);
      }
    });

    this.unlistenBlur = this.renderer.listen(nativeEl, 'blur', () => {
      this.renderer.removeStyle(nativeEl, 'background-color');
      this.renderer.removeStyle(nativeEl, 'outline');
      this.renderer.removeStyle(nativeEl, 'outline-offset');
      this.renderer.removeAttribute(nativeEl, 'aria-describedby');
    });
  }

  ngOnDestroy(): void {
    this.unlistenFocus?.();
    this.unlistenBlur?.();
  }
}
```

```typescript
// demo.component.ts
import { Component } from '@angular/core';
import { HighlightOnFocusDirective } from './highlight-on-focus.directive';

@Component({
  selector: 'app-demo',
  standalone: true,
  imports: [HighlightOnFocusDirective],
  template: `
    <p id="name-help">Dein vollständiger Name.</p>
    <input
      appHighlightOnFocus
      [highlightColor]="'#e8f5e9'"
      [helpId]="'name-help'"
      placeholder="Name"
    />

    <p id="msg-help">Maximal 500 Zeichen.</p>
    <textarea
      appHighlightOnFocus
      [helpId]="'msg-help'"
      placeholder="Nachricht"
    ></textarea>
  `,
})
export class DemoComponent {}
```

**Bonus – Renderer2 vs. direkte DOM-Manipulation im Vergleich:**

```typescript
// ❌ Nicht SSR-kompatibel, bricht in Web Workers
this.el.nativeElement.style.backgroundColor = '#fffde7';
document.getElementById('foo')?.classList.add('active');

// ✅ Plattformunabhängig mit Renderer2
this.renderer.setStyle(this.el.nativeElement, 'background-color', '#fffde7');
this.renderer.addClass(someEl, 'active');
```

## Weiterführendes
- **`PLATFORM_ID` & `isPlatformBrowser()`**: Wenn du unbedingt plattformspezifischen Code schreiben musst, prüfe die Plattform explizit: `inject(PLATFORM_ID)` + `isPlatformBrowser(id)`.
- **`RendererFactory2`**: Ermöglicht das manuale Erstellen eines `Renderer2` außerhalb von Direktiven/Komponenten (z.B. in einem Service).
- **Offizielle Docs**: [https://angular.dev/api/core/Renderer2](https://angular.dev/api/core/Renderer2)
- **Tipp**: In Angular 17+ kannst du Host-Direktiven (`hostDirectives`) nutzen, um `HighlightOnFocusDirective` automatisch auf Formular-Komponenten anzuwenden, ohne sie manuell im Template zu schreiben.

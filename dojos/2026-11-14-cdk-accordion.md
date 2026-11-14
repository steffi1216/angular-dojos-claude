# Angular Dojo: CDK Accordion
**Datum:** 2026-11-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `@angular/cdk/accordion` zugängliche, animierte Akkordeon-Komponenten baust, die vollständig tastaturbedienbar sind und sich nahtlos mit Angular Animations verbinden lassen.

## Hintergrund & Theorie
Das CDK-Paket `@angular/cdk/accordion` stellt zwei Direktiven bereit: `CdkAccordion` und `CdkAccordionItem`. Sie kümmern sich um die gesamte Zustandsverwaltung (welches Panel ist offen/geschlossen, Multi-Open vs. Single-Open) und geben dir volle Kontrolle über das Markup – ohne aufgezwungenes Styling.

**Kernkonzepte:**

- `CdkAccordion` ist der Container. Mit `[multi]="true"` dürfen mehrere Panels gleichzeitig geöffnet sein; ohne `multi` schließt ein öffnendes Panel alle anderen automatisch.
- `CdkAccordionItem` repräsentiert ein einzelnes Panel. Es stellt `expanded`, `toggle()`, `open()`, `close()` und einen `id`-String bereit. Über `#item="cdkAccordionItem"` greifst du per Template-Referenz darauf zu.
- Die Verbindung zu Angular Animations erfolgt über die `expanded`-Property als Trigger-Zustand – du steuerst Ein-/Ausblenden vollständig selbst.
- Für Barrierefreiheit bindest du `[attr.aria-expanded]`, `[attr.aria-controls]` und `[id]` manuell, was dir maximale Kontrolle über die ARIA-Struktur gibt.

Das Muster eignet sich ideal für FAQ-Sektionen, Einstellungsseiten oder komplexe Formulare, die in Abschnitte gegliedert werden sollen.

## Aufgabe
Erstelle eine standalone `FaqComponent`, die eine Liste von Frage-Antwort-Paaren als Akkordeon darstellt. Nur jeweils eine Antwort soll gleichzeitig sichtbar sein. Die Ausklapp-Animation soll über Angular Animations (`@angular/animations`) realisiert werden.

### Schritte
1. Lege eine neue standalone Komponente `FaqComponent` an und importiere `CdkAccordionModule` aus `@angular/cdk/accordion` sowie die benötigten Animations-Importe.
2. Definiere im Component eine `faqItems`-Liste mit mindestens vier Objekten à `{ question: string; answer: string }`.
3. Baue das Template mit `<cdk-accordion [multi]="false">`. Iteriere per `@for` über `faqItems` und verwende jeweils `<cdk-accordion-item #item="cdkAccordionItem">`.
4. Erstelle für den Header-Button korrekte ARIA-Attribute: `[attr.aria-expanded]="item.expanded"` und `[attr.aria-controls]="item.id + '-body'"`.
5. Definiere eine Angular-Animation `expandCollapse` mit zwei States (`void`/`collapsed` → `height: 0`, `expanded` → `height: *`) und einer `300ms ease-in-out`-Transition.
6. Binde die Animation am Panel-Body: `[@expandCollapse]="item.expanded ? 'expanded' : 'collapsed'"`.
7. Stelle sicher, dass Klick auf den Header `item.toggle()` aufruft und der Body nur gerendert wird, wenn `item.expanded` true ist (nutze `@if` oder bedingtes `*ngIf`-Äquivalent, **aber** die Animation soll Ein-/Ausblenden steuern – render die Div immer und verstecke sie per Animation, nicht per structural directive, damit die Leave-Animation sichtbar ist).

## Hints
<details>
<summary>Hint 1 – Animation Definition</summary>

```typescript
import { trigger, state, style, transition, animate } from '@angular/animations';

export const expandCollapse = trigger('expandCollapse', [
  state('collapsed, void', style({ height: '0px', overflow: 'hidden' })),
  state('expanded', style({ height: '*', overflow: 'hidden' })),
  transition('collapsed <=> expanded', animate('300ms ease-in-out')),
  transition(':enter', animate('300ms ease-in-out')),
  transition(':leave', animate('300ms ease-in-out')),
]);
```

Füge `animations: [expandCollapse]` im `@Component`-Decorator hinzu und setze `provideAnimations()` im App-Bootstrap.

</details>
<details>
<summary>Hint 2 – Template-Struktur</summary>

```html
<cdk-accordion [multi]="false">
  @for (item of faqItems; track item.question) {
    <cdk-accordion-item #acc="cdkAccordionItem">
      <div class="faq-item">
        <button
          class="faq-header"
          (click)="acc.toggle()"
          [attr.aria-expanded]="acc.expanded"
          [attr.aria-controls]="acc.id + '-body'"
        >
          {{ item.question }}
        </button>
        <div
          [id]="acc.id + '-body'"
          role="region"
          [@expandCollapse]="acc.expanded ? 'expanded' : 'collapsed'"
          class="faq-body"
        >
          <p>{{ item.answer }}</p>
        </div>
      </div>
    </cdk-accordion-item>
  }
</cdk-accordion>
```

</details>

## Beispiellösung

```typescript
import { Component } from '@angular/core';
import { CdkAccordionModule } from '@angular/cdk/accordion';
import {
  trigger, state, style, transition, animate
} from '@angular/animations';

@Component({
  selector: 'app-faq',
  standalone: true,
  imports: [CdkAccordionModule],
  animations: [
    trigger('expandCollapse', [
      state('collapsed, void', style({ height: '0px', overflow: 'hidden' })),
      state('expanded',        style({ height: '*',  overflow: 'hidden' })),
      transition('collapsed <=> expanded', animate('300ms ease-in-out')),
    ]),
  ],
  template: `
    <cdk-accordion [multi]="false">
      @for (item of faqItems; track item.question) {
        <cdk-accordion-item #acc="cdkAccordionItem">
          <div class="faq-item">
            <button
              class="faq-header"
              (click)="acc.toggle()"
              [attr.aria-expanded]="acc.expanded"
              [attr.aria-controls]="acc.id + '-body'"
            >
              {{ item.question }}
              <span class="icon">{{ acc.expanded ? '▲' : '▼' }}</span>
            </button>

            <div
              [id]="acc.id + '-body'"
              role="region"
              [@expandCollapse]="acc.expanded ? 'expanded' : 'collapsed'"
            >
              <div class="faq-body">
                <p>{{ item.answer }}</p>
              </div>
            </div>
          </div>
        </cdk-accordion-item>
      }
    </cdk-accordion>
  `,
  styles: [`
    .faq-item    { border-bottom: 1px solid #e0e0e0; }
    .faq-header  { width: 100%; text-align: left; padding: 16px;
                   background: none; border: none; cursor: pointer;
                   font-size: 1rem; display: flex; justify-content: space-between; }
    .faq-header:focus { outline: 2px solid #1976d2; outline-offset: -2px; }
    .faq-body    { padding: 0 16px 16px; color: #555; }
  `],
})
export class FaqComponent {
  readonly faqItems = [
    {
      question: 'Was ist Angular CDK?',
      answer:   'Das Component Dev Kit (CDK) bietet primitive Bausteine für ' +
                'UI-Komponenten – ohne Material-Styling, aber mit vollständiger ' +
                'Accessibility- und Interaction-Logik.',
    },
    {
      question: 'Wann verwende ich multi="true"?',
      answer:   'Wenn mehrere Panels gleichzeitig geöffnet sein dürfen, z. B. in ' +
                'Einstellungsseiten, bei denen der Nutzer mehrere Abschnitte parallel ' +
                'bearbeitet.',
    },
    {
      question: 'Wie funktioniert die ARIA-Verknüpfung?',
      answer:   'Der Button erhält aria-expanded und aria-controls (auf die ID des ' +
                'Panel-Body). Das Panel-Div bekommt role="region" und die passende ID. ' +
                'So können Screenreader die Beziehung korrekt ankündigen.',
    },
    {
      question: 'Kann ich das CDK-Akkordeon mit OnPush nutzen?',
      answer:   'Ja. CdkAccordionItem nutzt intern Signals bzw. BehaviorSubjects. ' +
                'Bei OnPush reicht async-Pipe oder eine Signal-Bindung; die Animation ' +
                'läuft unabhängig von der Change Detection.',
    },
  ];
}
```

**main.ts (Bootstrap):**
```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimations }    from '@angular/platform-browser/animations';
import { FaqComponent }         from './faq.component';

bootstrapApplication(FaqComponent, {
  providers: [provideAnimations()],
});
```

## Weiterführendes
- **Keyboard-Navigation hinzufügen:** Ergänze `(keydown.ArrowDown)` / `(keydown.ArrowUp)` am Header-Button und steuere den Fokus manuell – so erhält dein Akkordeon vollständige WCAG 2.1-Konformität für das Accordion-Pattern.
- **Lazy Content:** Nutze `*cdkAccordionItemLazy` (oder bedingtes Rendering via `@if (acc.expanded || acc.initialized)`), um den DOM-Inhalt erst beim ersten Öffnen zu rendern und die initiale Ladezeit zu optimieren.
- [CDK Accordion Docs](https://material.angular.io/cdk/accordion/overview)
- [ARIA Accordion Pattern (WAI)](https://www.w3.org/WAI/ARIA/apg/patterns/accordion/)

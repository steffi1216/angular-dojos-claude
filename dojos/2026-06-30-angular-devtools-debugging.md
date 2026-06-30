# Angular Dojo: Angular DevTools & Debugging
**Datum:** 2026-06-30
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, Angular DevTools effektiv einzusetzen, um Component Trees zu inspizieren, Change Detection zu analysieren und Signals/Injectors zu debuggen. Außerdem übst du Debugging-Techniken direkt im Code.

## Hintergrund & Theorie

Angular DevTools ist eine Browser-Extension (Chrome/Firefox), die speziell für Angular-Anwendungen entwickelt wurde. Sie bietet:

**Component Explorer:** Zeigt den gesamten Component Tree der Anwendung. Du kannst einzelne Komponenten auswählen, ihre Inputs/Outputs und State in Echtzeit einsehen und direkt im Browser verändern.

**Profiler:** Zeichnet Change Detection Cycles auf und zeigt, welche Komponenten wie oft und wie lange gerendert wurden. Unverzichtbar bei Performance-Problemen.

**Injector Tree:** Seit Angular 17+ zeigt DevTools den Dependency Injection Graph – wo ein Service instanziiert wurde und welche Providers aktiv sind.

**Programmatisches Debugging:** Angular stellt globale Hilfsfunktionen bereit, die in der Browser-Konsole verwendbar sind:
- `ng.getComponent(element)` – gibt die Komponenteninstanz zurück
- `ng.getContext(element)` – gibt den `NgIf`/`NgFor`-Kontext zurück
- `ng.applyChanges(component)` – triggert manuell Change Detection
- `ng.getOwningComponent(element)` – gibt die übergeordnete Komponente zurück
- `ng.getInjector(element)` – gibt den Injector der Komponente zurück

Signals lassen sich in DevTools ebenfalls live beobachten: Änderungen an Signals werden im Component Explorer sofort reflektiert.

## Aufgabe

Baue eine kleine Debug-Demo-Anwendung, die verschiedene Debugging-Szenarien demonstriert und dabei gezielt Angular DevTools einsetzt.

### Schritte

1. **Setup:** Erstelle eine neue Standalone-Komponente `DebugDemoComponent` mit einem `signal<number>` (Counter), einer berechneten `computed()`-Eigenschaft und einem `effect()`, der Änderungen loggt.

2. **Change Detection sichtbar machen:** Füge der Komponente ein `OnPush`-Change-Detection-Schema hinzu. Baue eine Child-Komponente `CounterDisplayComponent`, die den Counter-Wert via `@Input()` erhält. Implementiere einen Button, der den Wert mit `setTimeout` asynchron ändert – beobachte im Profiler, wann und wie oft Change Detection läuft.

3. **Injector debuggen:** Erstelle einen `DebugService` mit einem `InjectionToken`, der auf Komponentenebene (nicht Root) bereitgestellt wird. Nutze `ng.getInjector()` in der Konsole, um den Service zu inspizieren.

4. **Profiler-Auswertung:** Öffne Angular DevTools → Profiler → Starte Aufzeichnung → Klicke mehrfach auf den Button → Stoppe Aufzeichnung. Identifiziere welche Komponenten re-rendern und warum.

5. **Source Maps & Breakpoints:** Setze in den DevTools einen Breakpoint in der `ngOnChanges`-Methode der Child-Komponente und beobachte, wie oft sie aufgerufen wird.

```typescript
// Zielstruktur
@Component({
  selector: 'app-debug-demo',
  standalone: true,
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class DebugDemoComponent {
  counter = signal(0);
  doubled = computed(() => this.counter() * 2);
  
  constructor() {
    effect(() => console.log('[effect] counter:', this.counter()));
  }
  
  incrementAsync() {
    setTimeout(() => this.counter.update(v => v + 1), 500);
  }
}

@Component({
  selector: 'app-counter-display',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>Value: {{ value }} | Doubled: {{ doubled }}</p>`,
})
export class CounterDisplayComponent implements OnChanges {
  @Input() value = 0;
  @Input() doubled = 0;
  
  ngOnChanges(changes: SimpleChanges) {
    console.log('[OnChanges]', changes);
  }
}
```

## Hints

<details>
<summary>Hint 1 – OnPush mit Signals</summary>

Bei `OnPush`-Komponenten triggern Signals automatisch Change Detection, wenn sie im Template verwendet werden – auch ohne `async`-Pipe oder `ChangeDetectorRef`. Das ist ein wesentlicher Vorteil von Signals gegenüber `BehaviorSubject`. Wenn du aber einen normalen `setTimeout` ohne Signal verwendest, musst du manuell `cdr.markForCheck()` aufrufen.

```typescript
// Mit Signal – funktioniert automatisch auch bei OnPush:
counter = signal(0);

// Ohne Signal – braucht manuelles markForCheck:
value = 0;
incrementAsync() {
  setTimeout(() => {
    this.value++;
    this.cdr.markForCheck(); // notwendig!
  }, 500);
}
```

</details>

<details>
<summary>Hint 2 – ng.getComponent() in der Konsole</summary>

Um eine Komponenteninstanz in der Browser-Konsole zu inspizieren:

```javascript
// Element im Elements-Tab auswählen, dann in der Konsole:
const comp = ng.getComponent($0);
console.log(comp.counter()); // Signal-Wert lesen

// Signal direkt aus der Konsole ändern:
comp.counter.set(42);
ng.applyChanges(comp); // Change Detection triggern (bei OnPush nötig, wenn kein Signal im Template)
```

Für Signals im Template ist `ng.applyChanges()` meist nicht nötig, da Angular die Abhängigkeit automatisch trackt.

</details>

<details>
<summary>Hint 3 – InjectionToken auf Komponentenebene</summary>

```typescript
const DEBUG_TOKEN = new InjectionToken<string>('DEBUG_TOKEN');

@Component({
  providers: [
    { provide: DEBUG_TOKEN, useValue: 'component-level-value' }
  ]
})
export class DebugDemoComponent {
  value = inject(DEBUG_TOKEN);
}

// In der Konsole:
const injector = ng.getInjector($0);
injector.get(DEBUG_TOKEN); // 'component-level-value'
```

</details>

## Beispiellösung

```typescript
import {
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  Component,
  computed,
  effect,
  inject,
  InjectionToken,
  Input,
  OnChanges,
  signal,
  SimpleChanges,
} from '@angular/core';
import { CommonModule } from '@angular/common';

const DEBUG_LABEL = new InjectionToken<string>('DEBUG_LABEL');

@Component({
  selector: 'app-counter-display',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="display">
      <p>Value: <strong>{{ value }}</strong></p>
      <p>Doubled: <strong>{{ doubled }}</strong></p>
    </div>
  `,
})
export class CounterDisplayComponent implements OnChanges {
  @Input() value = 0;
  @Input() doubled = 0;

  ngOnChanges(changes: SimpleChanges): void {
    // Breakpoint hier setzen um OnChanges-Aufrufe zu beobachten
    console.log('[CounterDisplay OnChanges]', changes);
  }
}

@Component({
  selector: 'app-debug-demo',
  standalone: true,
  imports: [CommonModule, CounterDisplayComponent],
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [
    { provide: DEBUG_LABEL, useValue: 'DebugDemoComponent@component-level' },
  ],
  template: `
    <h2>Angular DevTools Debug Demo</h2>
    <p>Label (from InjectionToken): {{ label }}</p>

    <app-counter-display
      [value]="counter()"
      [doubled]="doubled()"
    />

    <div class="controls">
      <button (click)="increment()">Increment (sync)</button>
      <button (click)="incrementAsync()">Increment (async +500ms)</button>
      <button (click)="reset()">Reset</button>
    </div>

    <p class="hint">
      Öffne Angular DevTools → Component Explorer → wähle diese Komponente.<br/>
      Im Profiler: Aufzeichnung starten, Buttons klicken, auswerten.
    </p>
  `,
})
export class DebugDemoComponent {
  counter = signal(0);
  doubled = computed(() => this.counter() * 2);
  label = inject(DEBUG_LABEL);

  private cdr = inject(ChangeDetectorRef);

  constructor() {
    effect(() => {
      // Wird jedes Mal ausgeführt, wenn counter() sich ändert
      console.log('[effect] counter changed to:', this.counter());
    });
  }

  increment(): void {
    this.counter.update((v) => v + 1);
  }

  incrementAsync(): void {
    // Signal-Update innerhalb setTimeout – bei OnPush trotzdem reaktiv,
    // da Signals Zone-unabhängig sind und Angular selbst informieren.
    setTimeout(() => {
      this.counter.update((v) => v + 1);
    }, 500);
  }

  reset(): void {
    this.counter.set(0);
  }
}

// --- Konsolen-Snippets zum Ausprobieren (in DevTools Console) ---
//
// 1) Komponente selektieren ($0 = aktuell ausgewähltes Element im Inspector):
//    const c = ng.getComponent($0)
//    c.counter()        // Signal-Wert lesen
//    c.counter.set(99)  // Signal-Wert setzen
//
// 2) Injector inspizieren:
//    const inj = ng.getInjector($0)
//    inj.get(DEBUG_LABEL)  // 'DebugDemoComponent@component-level'
//
// 3) Übergeordnete Komponente:
//    ng.getOwningComponent($0)
```

## Weiterführendes

- **Angular DevTools Extension:** [Chrome Web Store – Angular DevTools](https://chrome.google.com/webstore/detail/angular-devtools/ienfalfjdbdpebioblfackkekamfmbnh) – kostenlos, von Angular Team gepflegt
- **Profiler-Guide:** Die offizielle Angular-Doku unter `angular.dev/tools/devtools` erklärt den Profiler im Detail, inkl. der Flame-Graph-Ansicht für Change Detection
- **Zoneless Debugging:** Ab Angular 18+ mit `provideExperimentalZonelessChangeDetection()` entfällt Zone.js – das macht den Profiler noch aussagekräftiger, da jedes Re-Render explizit auf einen Signal-Trigger zurückgeführt werden kann
- **`ng.probe()` ist deprecated:** In älteren Tutorials findest du `ng.probe()` – das ist seit Angular Ivy entfernt; verwende stattdessen `ng.getComponent()` u.a.
- **VS Code Angular Language Service:** Ergänzt DevTools um statische Analyse direkt im Editor (Autocomplete, Template-Typprüfung)

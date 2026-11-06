# Angular Dojo: State Machine Pattern mit Signals
**Datum:** 2026-11-06
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, einen endlichen Zustandsautomaten (Finite State Machine) mit `signal()`, `computed()` und `update()` zu implementieren – ohne externe Bibliothek. Du verstehst, wie Signals die explizite Zustandsführung vereinfachen und ungültige Übergänge zur Laufzeit verhindern.

## Hintergrund & Theorie

Ein Finite State Machine (FSM) modelliert ein System als Menge von **Zuständen** und **Übergängen**. Zu jedem Zeitpunkt befindet sich das System in genau einem Zustand, und nur bestimmte Übergänge sind erlaubt.

In Angular gab es klassisch zwei Lösungen:
- Enum + boolean-Flags (fehleranfällig, Flags können inkonsistent werden)
- XState / Robot (externe Bibliotheken, Boilerplate)

Mit **Signals** gibt es einen dritten Weg: ein `signal<State>()` als Single Source of Truth. `computed()` leitet daraus alle UI-Eigenschaften ab:

```typescript
type TrafficLightState = 'red' | 'yellow' | 'green';
const state = signal<TrafficLightState>('red');
const canGo = computed(() => state() === 'green');
```

Der entscheidende Vorteil: `computed()`-Werte sind **garantiert konsistent** – es ist unmöglich, einen Zustand zu haben, in dem `canGo` falsch berechnet wurde. Zustandsübergänge werden in einer Transition-Map zentralisiert, sodass nur erlaubte Sprünge möglich sind.

## Aufgabe

Implementiere einen **HTTP-Request-Lifecycle** als State Machine in einem Angular Service und einem dazugehörigen Component.

Die Zustände lauten: `idle → loading → success | error`. Von `success` und `error` kann man zurück zu `idle` (Reset).

### Schritte

1. Erstelle eine Datei `request-state-machine.ts` mit dem Type, den erlaubten Transitions und einem `createRequestMachine()`-Helper.

2. Erstelle einen Service `DataService`, der die State Machine nutzt. Er soll:
   - `fetch()` aufrufen können (nur wenn `idle`)
   - `reset()` erlauben (von `success` oder `error`)
   - Simuliere den HTTP-Call mit `setTimeout` oder einem RxJS-Operator

3. Erstelle einen Component `RequestDemoComponent` mit Template-Bindings auf `machine.state()`, `machine.isLoading()`, `machine.hasError()` und `machine.data()`.

4. Schreibe einen Unit-Test mit `TestBed` und prüfe, dass ein `fetch()`-Aufruf während `loading` ignoriert wird (kein doppelter Request).

**Ziel-UX im Template:**
```
[Idle]   → Button "Laden" sichtbar
[Loading]→ Spinner sichtbar, Button deaktiviert
[Success]→ Daten sichtbar + Button "Reset"
[Error]  → Fehlermeldung + Button "Erneut versuchen"
```

### Schritte
1. Definiere den `RequestState`-Typ mit einem Discriminated Union: `{ status: 'idle' } | { status: 'loading' } | { status: 'success'; data: string[] } | { status: 'error'; message: string }`.
2. Implementiere die Transition-Map und eine `transition()`-Funktion, die nur erlaubte Übergänge ausführt (sonst `console.warn`).
3. Erstelle den Service mit `signal<RequestState>()` und exponiere `computed()`-Derivate.
4. Binde den Service im Component via `inject()` ein und rendere alle Zustände im Template.
5. Schreibe den Guard-Test (fetch im `loading`-Zustand bleibt wirkungslos).

## Hints

<details>
<summary>Hint 1 – Discriminated Union und Transition-Map</summary>

```typescript
export type RequestState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; message: string };

// Erlaubte Übergänge
const TRANSITIONS: Partial<Record<RequestState['status'], RequestState['status'][]>> = {
  idle:    ['loading'],
  loading: ['success', 'error'],
  success: ['idle'],
  error:   ['idle'],
};

function canTransition(from: RequestState['status'], to: RequestState['status']): boolean {
  return TRANSITIONS[from]?.includes(to) ?? false;
}
```

</details>

<details>
<summary>Hint 2 – Service mit Signals und computed-Derivaten</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  private _state = signal<RequestState>({ status: 'idle' });

  readonly state    = this._state.asReadonly();
  readonly isIdle   = computed(() => this._state().status === 'idle');
  readonly isLoading = computed(() => this._state().status === 'loading');
  readonly data     = computed(() =>
    this._state().status === 'success' ? this._state().data : null
  );
  readonly error    = computed(() =>
    this._state().status === 'error' ? this._state().message : null
  );

  private transition(next: RequestState): void {
    if (!canTransition(this._state().status, next.status)) {
      console.warn(`Invalid transition: ${this._state().status} → ${next.status}`);
      return;
    }
    this._state.set(next);
  }

  fetch(): void {
    this.transition({ status: 'loading' });
    // Simulierter async Call
    setTimeout(() => {
      Math.random() > 0.3
        ? this.transition({ status: 'success', data: ['Item A', 'Item B'] })
        : this.transition({ status: 'error', message: 'Netzwerkfehler' });
    }, 1500);
  }

  reset(): void {
    this.transition({ status: 'idle' });
  }
}
```

</details>

## Beispiellösung

```typescript
// request-state.ts
export type RequestState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; message: string };

const TRANSITIONS: Partial<Record<RequestState['status'], RequestState['status'][]>> = {
  idle:    ['loading'],
  loading: ['success', 'error'],
  success: ['idle'],
  error:   ['idle'],
};

export function canTransition(
  from: RequestState['status'],
  to: RequestState['status']
): boolean {
  return TRANSITIONS[from]?.includes(to) ?? false;
}

// data.service.ts
import { Injectable, computed, signal } from '@angular/core';
import { RequestState, canTransition } from './request-state';

@Injectable({ providedIn: 'root' })
export class DataService {
  private _state = signal<RequestState>({ status: 'idle' });

  readonly state     = this._state.asReadonly();
  readonly isIdle    = computed(() => this._state().status === 'idle');
  readonly isLoading = computed(() => this._state().status === 'loading');
  readonly isSuccess = computed(() => this._state().status === 'success');
  readonly isError   = computed(() => this._state().status === 'error');
  readonly data      = computed(() =>
    this._state().status === 'success' ? (this._state() as { status: 'success'; data: string[] }).data : null
  );
  readonly errorMsg  = computed(() =>
    this._state().status === 'error' ? (this._state() as { status: 'error'; message: string }).message : null
  );

  private go(next: RequestState): void {
    if (!canTransition(this._state().status, next.status)) {
      console.warn(`FSM: Übergang ${this._state().status} → ${next.status} nicht erlaubt`);
      return;
    }
    this._state.set(next);
  }

  fetch(): void {
    this.go({ status: 'loading' });
    setTimeout(() => {
      Math.random() > 0.3
        ? this.go({ status: 'success', data: ['Eintrag 1', 'Eintrag 2', 'Eintrag 3'] })
        : this.go({ status: 'error', message: 'Server nicht erreichbar' });
    }, 1500);
  }

  reset(): void {
    this.go({ status: 'idle' });
  }
}

// request-demo.component.ts
import { Component, inject } from '@angular/core';
import { DataService } from './data.service';

@Component({
  selector: 'app-request-demo',
  standalone: true,
  template: `
    <div class="demo">
      <p>Status: <strong>{{ svc.state().status }}</strong></p>

      @if (svc.isIdle()) {
        <button (click)="svc.fetch()">Daten laden</button>
      }

      @if (svc.isLoading()) {
        <p>⏳ Wird geladen...</p>
      }

      @if (svc.isSuccess()) {
        <ul>
          @for (item of svc.data(); track item) {
            <li>{{ item }}</li>
          }
        </ul>
        <button (click)="svc.reset()">Zurücksetzen</button>
      }

      @if (svc.isError()) {
        <p class="error">❌ {{ svc.errorMsg() }}</p>
        <button (click)="svc.reset()">Erneut versuchen</button>
      }
    </div>
  `,
})
export class RequestDemoComponent {
  protected svc = inject(DataService);
}

// data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { DataService } from './data.service';

describe('DataService – State Machine Guard', () => {
  let service: DataService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(DataService);
  });

  it('ignoriert fetch() im loading-Zustand', () => {
    const warnSpy = spyOn(console, 'warn');
    service.fetch();                    // idle → loading ✓
    service.fetch();                    // loading → loading ✗ (guard)
    expect(warnSpy).toHaveBeenCalled();
    expect(service.state().status).toBe('loading');
  });

  it('erlaubt reset() nur von success oder error', () => {
    const warnSpy = spyOn(console, 'warn');
    service.reset();                    // idle → idle ✗ (guard)
    expect(warnSpy).toHaveBeenCalledTimes(1);
  });
});
```

## Weiterführendes

- **XState v5** bietet typsichere FSMs mit Guards, Actions und Services – lohnt sich bei komplexen Workflows mit vielen Zuständen.
- Schau dir `@ngrx/signals` `withState` an: es folgt ähnlichen Prinzipien, aber auf Store-Ebene.
- Für serverseitige Validierung von Transitionen: kombiniere das Muster mit `httpResource()` und einem `effect()`, der auf Statusänderungen reagiert.
- Lies [David Khourshid – Statecharts intro](https://statecharts.dev) für eine formale Einführung in hierarchische Zustandsmaschinen.

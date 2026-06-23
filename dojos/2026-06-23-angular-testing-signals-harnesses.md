# Angular Dojo: Testing mit Signals & Component Harnesses
**Datum:** 2026-06-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Komponenten, die Signals verwenden, korrekt testest und wie du Angular CDK Component Harnesses nutzt, um robuste, implementierungsunabhängige Tests zu schreiben.

## Hintergrund & Theorie

### Signals testen
Seit Angular 16/17 sind Signals das neue Reaktivitätsmodell. In Tests gibt es einige Besonderheiten:
- `signal()`, `computed()` und `effect()` benötigen einen reaktiven Kontext.
- In Unit-Tests ohne `TestBed` kannst du Signals direkt lesen (kein `.value`, sondern `mySignal()`).
- Mit `TestBed` musst du nach Signaländerungen `fixture.detectChanges()` aufrufen – oder `TestBed.flushEffects()` für `effect()`.
- `toSignal()` aus `@angular/core/rxjs-interop` benötigt einen Injection Context; in Tests wird dieser durch `TestBed.runInInjectionContext()` bereitgestellt.

### Component Harnesses
Angular CDK bietet das `HarnessLoader`-System, um UI-Interaktionen in Tests von der DOM-Struktur zu entkoppeln. Harnesses sind typsichere Abstractions über Komponenten – z. B. `MatButtonHarness`, `MatInputHarness`. Du kannst eigene Harnesses für deine Komponenten erstellen, die `ComponentHarness` von `@angular/cdk/testing` erweitern.

Vorteil: Tests brechen nicht, wenn sich internes HTML ändert – nur die öffentliche API zählt.

## Aufgabe

Erstelle eine `CounterComponent` mit Signals und schreibe dazu Tests mit einem eigenen Component Harness.

### Schritte

1. **Erstelle `CounterComponent`** als Standalone Component mit folgenden Signals:
   - `count = signal(0)` – aktueller Zählerstand
   - `doubled = computed(() => this.count() * 2)` – doppelter Wert
   - Methoden: `increment()`, `decrement()`, `reset()`
   - Template: Zeige `count()` und `doubled()` an, dazu drei Buttons

2. **Erstelle `CounterHarness`** (`counter.harness.ts`), der `ComponentHarness` erweitert:
   - `static hostSelector = 'app-counter'`
   - Methoden: `getCount()`, `getDoubled()`, `clickIncrement()`, `clickDecrement()`, `clickReset()`
   - Nutze `this.locatorFor('button[data-testid="increment"]')` etc.

3. **Schreibe Tests** in `counter.component.spec.ts`:
   - Test 1: Initialwert ist 0, doubled ist 0
   - Test 2: Nach 3x `clickIncrement()` ist count 3 und doubled 6
   - Test 3: Nach `clickDecrement()` bei 0 bleibt count bei 0 (Clamp)
   - Test 4: `clickReset()` setzt count auf 0 zurück
   - Nutze `TestbedHarnessEnvironment.loader(fixture)` aus `@angular/cdk/testing/testbed`

4. **Bonus:** Teste einen `effect()` in der Komponente, der ein Log schreibt, wenn `count` 10 überschreitet – nutze `TestBed.flushEffects()`.

## Hints

<details>
<summary>Hint 1 – CounterComponent Grundstruktur</summary>

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p data-testid="count">{{ count() }}</p>
    <p data-testid="doubled">{{ doubled() }}</p>
    <button data-testid="increment" (click)="increment()">+</button>
    <button data-testid="decrement" (click)="decrement()">-</button>
    <button data-testid="reset" (click)="reset()">Reset</button>
  `
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment() { this.count.update(v => v + 1); }
  decrement() { this.count.update(v => Math.max(0, v - 1)); }
  reset() { this.count.set(0); }
}
```

</details>

<details>
<summary>Hint 2 – CounterHarness</summary>

```typescript
import { ComponentHarness } from '@angular/cdk/testing';

export class CounterHarness extends ComponentHarness {
  static hostSelector = 'app-counter';

  private countEl = this.locatorFor('[data-testid="count"]');
  private doubledEl = this.locatorFor('[data-testid="doubled"]');
  private incrementBtn = this.locatorFor('[data-testid="increment"]');
  private decrementBtn = this.locatorFor('[data-testid="decrement"]');
  private resetBtn = this.locatorFor('[data-testid="reset"]');

  async getCount(): Promise<number> {
    return +(await (await this.countEl()).text());
  }

  async getDoubled(): Promise<number> {
    return +(await (await this.doubledEl()).text());
  }

  async clickIncrement() { await (await this.incrementBtn()).click(); }
  async clickDecrement() { await (await this.decrementBtn()).click(); }
  async clickReset() { await (await this.resetBtn()).click(); }
}
```

</details>

<details>
<summary>Hint 3 – Test-Setup mit Harness</summary>

```typescript
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { TestBed } from '@angular/core/testing';

describe('CounterComponent', () => {
  async function setup() {
    await TestBed.configureTestingModule({
      imports: [CounterComponent],
    }).compileComponents();

    const fixture = TestBed.createComponent(CounterComponent);
    fixture.detectChanges();
    const loader = TestbedHarnessEnvironment.loader(fixture);
    const harness = await loader.getHarness(CounterHarness);
    return { fixture, harness, component: fixture.componentInstance };
  }

  it('should start at 0', async () => {
    const { harness } = await setup();
    expect(await harness.getCount()).toBe(0);
    expect(await harness.getDoubled()).toBe(0);
  });
});
```

</details>

<details>
<summary>Hint 4 – effect() testen (Bonus)</summary>

```typescript
// In der Komponente:
logEffect = effect(() => {
  if (this.count() > 10) {
    console.log('Count exceeded 10!');
  }
});

// Im Test:
it('should trigger effect when count exceeds 10', () => {
  const consoleSpy = jest.spyOn(console, 'log');
  component.count.set(11);
  TestBed.flushEffects(); // Führt alle pending effects aus
  expect(consoleSpy).toHaveBeenCalledWith('Count exceeded 10!');
});
```

</details>

## Beispiellösung

```typescript
// counter.component.ts
import { Component, computed, effect, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p data-testid="count">{{ count() }}</p>
    <p data-testid="doubled">{{ doubled() }}</p>
    <button data-testid="increment" (click)="increment()">+</button>
    <button data-testid="decrement" (click)="decrement()">-</button>
    <button data-testid="reset" (click)="reset()">Reset</button>
  `
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  logEffect = effect(() => {
    if (this.count() > 10) console.log('Count exceeded 10!');
  });

  increment() { this.count.update(v => v + 1); }
  decrement() { this.count.update(v => Math.max(0, v - 1)); }
  reset() { this.count.set(0); }
}

// counter.harness.ts
import { ComponentHarness } from '@angular/cdk/testing';

export class CounterHarness extends ComponentHarness {
  static hostSelector = 'app-counter';

  private countEl = this.locatorFor('[data-testid="count"]');
  private doubledEl = this.locatorFor('[data-testid="doubled"]');
  private incrementBtn = this.locatorFor('[data-testid="increment"]');
  private decrementBtn = this.locatorFor('[data-testid="decrement"]');
  private resetBtn = this.locatorFor('[data-testid="reset"]');

  async getCount(): Promise<number> {
    return +(await (await this.countEl()).text());
  }
  async getDoubled(): Promise<number> {
    return +(await (await this.doubledEl()).text());
  }
  async clickIncrement() { await (await this.incrementBtn()).click(); }
  async clickDecrement() { await (await this.decrementBtn()).click(); }
  async clickReset() { await (await this.resetBtn()).click(); }
}

// counter.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { CounterComponent } from './counter.component';
import { CounterHarness } from './counter.harness';

describe('CounterComponent with Harness', () => {
  async function setup() {
    await TestBed.configureTestingModule({
      imports: [CounterComponent],
    }).compileComponents();

    const fixture = TestBed.createComponent(CounterComponent);
    fixture.detectChanges();
    const loader = TestbedHarnessEnvironment.loader(fixture);
    const harness = await loader.getHarness(CounterHarness);
    return { fixture, harness, component: fixture.componentInstance };
  }

  it('should initialize count and doubled to 0', async () => {
    const { harness } = await setup();
    expect(await harness.getCount()).toBe(0);
    expect(await harness.getDoubled()).toBe(0);
  });

  it('should increment count and update doubled', async () => {
    const { harness } = await setup();
    await harness.clickIncrement();
    await harness.clickIncrement();
    await harness.clickIncrement();
    expect(await harness.getCount()).toBe(3);
    expect(await harness.getDoubled()).toBe(6);
  });

  it('should not decrement below 0', async () => {
    const { harness } = await setup();
    await harness.clickDecrement();
    expect(await harness.getCount()).toBe(0);
  });

  it('should reset count to 0', async () => {
    const { harness, component, fixture } = await setup();
    component.count.set(5);
    fixture.detectChanges();
    await harness.clickReset();
    expect(await harness.getCount()).toBe(0);
  });

  it('should trigger effect when count exceeds 10 (Bonus)', () => {
    TestBed.configureTestingModule({ imports: [CounterComponent] });
    const fixture = TestBed.createComponent(CounterComponent);
    fixture.detectChanges();
    const consoleSpy = jest.spyOn(console, 'log');
    fixture.componentInstance.count.set(11);
    TestBed.flushEffects();
    expect(consoleSpy).toHaveBeenCalledWith('Count exceeded 10!');
  });
});
```

## Weiterführendes
- Erstelle Harnesses für komplexere Komponenten (z. B. mit `MatTable`) und teste pagination/sorting ohne DOM-Abhängigkeiten.
- Lese die offizielle Doku zu [Angular CDK Testing](https://material.angular.io/cdk/testing/overview) für Parallel-Harnesses und `HarnessPredicate`-Filter.
- Erkunde `jest-auto-spies` oder `ng-mocks` für noch schlankere Testaufbauten mit Signals.

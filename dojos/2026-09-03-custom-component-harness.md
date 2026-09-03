# Angular Dojo: Custom Component Harnesses schreiben
**Datum:** 2026-09-03
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK eigene `ComponentHarness`-Klassen für deine UI-Komponenten erstellst, damit Tests stabil, lesbar und wartungsfreundlich bleiben.

## Hintergrund & Theorie

Das Angular CDK stellt mit `@angular/cdk/testing` eine Harness-Infrastruktur bereit, die ursprünglich für Angular Material entwickelt wurde. Statt direkt DOM-Elemente in Tests per `querySelector` zu suchen, abstrahiert ein `ComponentHarness` die Interaktion mit einer Komponente hinter einer stabilen, semantischen API.

**Vorteile:**
- Tests brechen nicht, wenn sich interne DOM-Struktur ändert
- Harnesses können in Unit- und E2E-Tests gleichermaßen verwendet werden
- Wiederverwendbare Test-Utilities über das ganze Projekt hinweg

**Kernkonzepte:**
- `ComponentHarness` – Basisklasse, die jede Harness-Klasse erweitert
- `HarnessPredicate<T>` – ermöglicht das Filtern nach Optionen (z. B. Label-Text)
- `this.locatorFor(selector)` – lazy locator für ein Kind-Element
- `this.locatorForAll(selector)` – lazy locator für mehrere Kind-Elemente
- `TestElement` – abstrahiert ein DOM-Element (click, text, getAttribute, …)
- `HarnessLoader` – Entry-Point, um Harnesses im Test zu laden (`TestbedHarnessEnvironment.loader(fixture)`)

Ein Harness sollte **keine Assertions** enthalten – er liefert nur Daten und führt Interaktionen aus. Die eigentlichen Assertions bleiben im Test.

## Aufgabe

Gegeben ist eine `RatingComponent`, die 1–5 Sterne anzeigt und beim Klick auf einen Stern den Wert ändert. Schreibe:

1. Die `RatingComponent` selbst (einfach gehalten)
2. Eine `RatingHarness`-Klasse, die folgende API bereitstellt:
   - `getStarCount(): Promise<number>` – Anzahl der Sterne
   - `getSelectedRating(): Promise<number>` – aktuell gewählter Wert
   - `clickStar(index: number): Promise<void>` – klickt auf Stern `index` (1-basiert)
   - `isStarActive(index: number): Promise<boolean>` – ist Stern `index` aktiv?
3. Zwei Tests, die `RatingHarness` statt direkter DOM-Abfragen verwenden

### Schritte

1. Erstelle `rating.component.ts` mit `@Input() value = 0` und einem `@Output() valueChange`.
   Das Template rendert 5 `<button class="star">` Elemente; aktive Sterne erhalten die Klasse `active`.
2. Erstelle `rating.harness.ts` und erweitere `ComponentHarness`.
   Definiere `static hostSelector = 'app-rating'`.
3. Implementiere `getStarCount()` mit `this.locatorForAll('.star')`.
4. Implementiere `getSelectedRating()`: zähle die Elemente mit Klasse `active`.
5. Implementiere `clickStar(index)`: hole alle Sterne, klicke auf Index `index - 1`.
6. Implementiere `isStarActive(index)`: prüfe via `hasClass('active')`.
7. Schreibe einen Test der klickt & prüft, einen der den initialen Zustand prüft.

## Hints

<details>
<summary>Hint 1 – ComponentHarness Grundgerüst</summary>

```typescript
import { ComponentHarness, HarnessPredicate } from '@angular/cdk/testing';

export class RatingHarness extends ComponentHarness {
  static hostSelector = 'app-rating';

  private stars = this.locatorForAll('.star');

  async getStarCount(): Promise<number> {
    return (await this.stars()).length;
  }
}
```

`locatorForAll` gibt eine Funktion zurück, die beim Aufruf ein `Promise<TestElement[]>` liefert.

</details>

<details>
<summary>Hint 2 – hasClass und HarnessPredicate</summary>

`TestElement` hat die Methode `hasClass(className: string): Promise<boolean>`.

Für `isStarActive(index)`:
```typescript
const allStars = await this.stars();
return allStars[index - 1].hasClass('active');
```

Willst du Harnesses nach eigenen Optionen filtern (z. B. ein Label), nutze:
```typescript
static with(options: { label?: string } = {}): HarnessPredicate<RatingHarness> {
  return new HarnessPredicate(RatingHarness, options);
}
```

Im Test kannst du dann nach Label filtern:
```typescript
const harness = await loader.getHarness(RatingHarness.with({ label: 'Quality' }));
```

</details>

## Beispiellösung

```typescript
// rating.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-rating',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="rating">
      @for (star of stars; track star) {
        <button
          class="star"
          [class.active]="star <= value"
          (click)="setValue(star)"
          [attr.aria-label]="'Rate ' + star + ' stars'"
        >★</button>
      }
    </div>
  `,
})
export class RatingComponent {
  @Input() value = 0;
  @Output() valueChange = new EventEmitter<number>();

  stars = [1, 2, 3, 4, 5];

  setValue(rating: number): void {
    this.value = rating;
    this.valueChange.emit(rating);
  }
}
```

```typescript
// rating.harness.ts
import { ComponentHarness } from '@angular/cdk/testing';

export class RatingHarness extends ComponentHarness {
  static hostSelector = 'app-rating';

  private stars = this.locatorForAll('.star');

  async getStarCount(): Promise<number> {
    return (await this.stars()).length;
  }

  async getSelectedRating(): Promise<number> {
    const allStars = await this.stars();
    let count = 0;
    for (const star of allStars) {
      if (await star.hasClass('active')) count++;
    }
    return count;
  }

  async clickStar(index: number): Promise<void> {
    const allStars = await this.stars();
    await allStars[index - 1].click();
  }

  async isStarActive(index: number): Promise<boolean> {
    const allStars = await this.stars();
    return allStars[index - 1].hasClass('active');
  }
}
```

```typescript
// rating.component.spec.ts
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { RatingComponent } from './rating.component';
import { RatingHarness } from './rating.harness';

describe('RatingComponent via Harness', () => {
  let fixture: ComponentFixture<RatingComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [RatingComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(RatingComponent);
    fixture.detectChanges();
  });

  it('should show 5 stars and none active by default', async () => {
    const loader = TestbedHarnessEnvironment.loader(fixture);
    const harness = await loader.getHarness(RatingHarness);

    expect(await harness.getStarCount()).toBe(5);
    expect(await harness.getSelectedRating()).toBe(0);
  });

  it('should activate stars up to clicked star', async () => {
    const loader = TestbedHarnessEnvironment.loader(fixture);
    const harness = await loader.getHarness(RatingHarness);

    await harness.clickStar(3);

    expect(await harness.getSelectedRating()).toBe(3);
    expect(await harness.isStarActive(1)).toBeTrue();
    expect(await harness.isStarActive(3)).toBeTrue();
    expect(await harness.isStarActive(4)).toBeFalse();
  });
});
```

## Weiterführendes

- **`HarnessPredicate`** nutzen, um Harnesses nach eigenen Optionen zu filtern (z. B. `RatingHarness.with({ maxValue: 5 })`).
- **`BaseHarnessFilters`** erweitern: standardmäßig stehen `selector` und `ancestor` zur Verfügung.
- Schau dir an, wie Angular Material Harnesses aufgebaut sind (z. B. `MatButtonHarness` im CDK-Source), um Best Practices für größere Komponentenbibliotheken zu lernen.
- Mit **`parallel()`** aus `@angular/cdk/testing` kannst du mehrere `Promise`-Abfragen gleichzeitig ausführen und so Tests deutlich beschleunigen.

# Angular Dojo: Signal Effects Deep Dive
**Datum:** 2026-08-01
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst die `effect()`-Funktion in Angular Signals in der Tiefe: Wann und wie sie ausgeführt wird, wie du Cleanup-Logik registrierst, was `untracked()` leistet und warum der Injection Context entscheidend ist.

## Hintergrund & Theorie

`effect()` ist das reaktive Seiteneffekt-Primitiv in Angular Signals. Es registriert eine Funktion, die automatisch neu ausgeführt wird, sobald sich ein darin gelesenes Signal ändert.

**Wichtige Eigenschaften:**

- `effect()` muss innerhalb eines Injection Context aufgerufen werden (z. B. im Konstruktor, in einer Fabrikfunktion mit `runInInjectionContext()` oder über `inject()`).
- Der Effekt läuft **asynchron** – erst beim nächsten Microtask-Flush, nicht sofort.
- Gelesene Signals werden **automatisch getrackt** – kein explizites `subscribe()` nötig.
- Signals, die via `untracked()` gelesen werden, lösen keine neue Effekt-Ausführung aus.
- Cleanup: Die Callback-Funktion kann eine `onCleanup`-Funktion entgegennehmen, die vor jedem Re-Run und beim Zerstören des Effekts aufgerufen wird.
- Mit `{ allowSignalWrites: true }` darf der Effekt selbst Signals schreiben (Vorsicht: Zyklen!).
- `effect()` gibt ein `EffectRef` zurück, mit dem der Effekt manuell via `.destroy()` beendet werden kann.

```typescript
const count = signal(0);

effect(() => {
  console.log('count ist:', count()); // wird bei jeder Änderung ausgeführt
});
```

## Aufgabe

Implementiere einen `LoggingService`, der mit `effect()` automatisch auf Signal-Änderungen reagiert, Cleanup sauber abwickelt und `untracked()` gezielt einsetzt.

### Schritte

1. **Service erstellen:** Erzeuge einen `LoggingService` mit zwei Signals: `searchTerm = signal('')` und `resultCount = signal(0)`.

2. **Haupt-Effekt:** Registriere in der `constructor`-Methode einen `effect()`, der bei jeder Änderung von `searchTerm` eine Fetch-Simulation startet (z. B. `setTimeout`). Nutze `onCleanup`, um den vorherigen Timeout abzubrechen (wie ein manuelles `switchMap`-Verhalten).

3. **`untracked()` einsetzen:** Im Effekt soll `resultCount` **gelesen**, aber **nicht getrackt** werden, sodass eine Änderung von `resultCount` allein keinen Re-Run auslöst.

4. **Manuelles Zerstören:** Füge eine Methode `stopLogging()` hinzu, die das `EffectRef` per `.destroy()` deaktiviert.

5. **Demo-Komponente:** Binde `LoggingService` in eine Komponente ein; zeige `searchTerm` und `resultCount` per Template-Binding an und ändere `searchTerm` per Input-Feld.

## Hints

<details>
<summary>Hint 1 – Cleanup mit onCleanup</summary>

```typescript
effect((onCleanup) => {
  const term = this.searchTerm();
  const id = setTimeout(() => {
    // Simulierter API-Call
    this.resultCount.set(Math.floor(Math.random() * 100));
  }, 300);

  onCleanup(() => clearTimeout(id)); // wird vor dem nächsten Run aufgerufen
});
```

</details>

<details>
<summary>Hint 2 – untracked()</summary>

```typescript
import { untracked } from '@angular/core';

effect(() => {
  const term = this.searchTerm(); // getrackt
  const count = untracked(() => this.resultCount()); // NICHT getrackt
  console.log(`Suche nach "${term}", zuletzt ${count} Ergebnisse`);
});
```

</details>

<details>
<summary>Hint 3 – EffectRef und manuelles Zerstören</summary>

```typescript
import { effect, EffectRef } from '@angular/core';

private logRef: EffectRef = effect(() => { ... });

stopLogging() {
  this.logRef.destroy();
}
```

</details>

## Beispiellösung

```typescript
// logging.service.ts
import { Injectable, effect, EffectRef, signal, untracked } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoggingService {
  searchTerm = signal('');
  resultCount = signal(0);

  private logRef: EffectRef;

  constructor() {
    this.logRef = effect((onCleanup) => {
      const term = this.searchTerm();
      const previousCount = untracked(() => this.resultCount());

      console.log(`[Effect] Neue Suche: "${term}" (vorherige Trefferzahl: ${previousCount})`);

      const timeoutId = setTimeout(() => {
        const fakeCount = term.length * 7;
        this.resultCount.set(fakeCount);
        console.log(`[Effect] Ergebnis für "${term}": ${fakeCount} Treffer`);
      }, 300);

      onCleanup(() => {
        console.log(`[Cleanup] Timeout für "${term}" abgebrochen`);
        clearTimeout(timeoutId);
      });
    });
  }

  stopLogging(): void {
    this.logRef.destroy();
    console.log('[LoggingService] Effekt zerstört');
  }
}

// search.component.ts
import { Component, inject } from '@angular/core';
import { LoggingService } from './logging.service';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-search',
  standalone: true,
  imports: [FormsModule],
  template: `
    <input
      type="text"
      [ngModel]="svc.searchTerm()"
      (ngModelChange)="svc.searchTerm.set($event)"
      placeholder="Suchbegriff eingeben..."
    />
    <p>Suchbegriff: <strong>{{ svc.searchTerm() }}</strong></p>
    <p>Letzte Trefferzahl: <strong>{{ svc.resultCount() }}</strong></p>
    <button (click)="svc.stopLogging()">Logging stoppen</button>
  `,
})
export class SearchComponent {
  svc = inject(LoggingService);
}
```

**Ausgabe in der Konsole bei Eingabe von "ng":**
```
[Effect] Neue Suche: "n" (vorherige Trefferzahl: 0)
[Cleanup] Timeout für "n" abgebrochen   ← onCleanup vor Re-Run
[Effect] Neue Suche: "ng" (vorherige Trefferzahl: 0)
[Effect] Ergebnis für "ng": 14 Treffer
```

## Weiterführendes

- `runInInjectionContext(injector, () => effect(...))` ermöglicht `effect()` außerhalb des Konstruktors – z. B. in lazy-geladenen Routen oder Services ohne Konstruktor-Injection.
- Vermeide Schreib-Zyklen: Wenn ein Effekt ein Signal schreibt, das er auch liest, entsteht eine Endlosschleife. Nutze `untracked()` oder `allowSignalWrites: true` mit Bedacht.
- Offizielle Doku: [angular.dev/guide/signals#effects](https://angular.dev/guide/signals#effects)

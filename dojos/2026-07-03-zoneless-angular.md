# Angular Dojo: Zoneless Angular (ohne Zone.js)
**Datum:** 2026-07-03
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du eine Angular-Anwendung ohne Zone.js betreibst und verstehst, wie das neue zoneless Change-Detection-Modell mit Signals zusammenarbeitet.

## Hintergrund & Theorie

Traditionell nutzt Angular **Zone.js**, um asynchrone Operationen (setTimeout, Promises, HTTP-Calls) zu "patchen" und automatisch eine Change Detection auszulösen. Das funktioniert, hat aber einen Preis: Zone.js monkey-patcht Browser-APIs global und löst Change Detection oft unnötig häufig aus.

Seit Angular 18 (stabil ab v19) gibt es einen offiziellen **Zoneless-Modus**. Dabei wird Change Detection ausschließlich durch explizite Signale ausgelöst:

- **Signals** (`signal()`, `computed()`) – markieren Komponenten direkt als "dirty"
- **`ChangeDetectorRef.markForCheck()`** – manueller Trigger für zonebasierte Flows
- **`ChangeDetectorRef.detectChanges()`** – sofortige lokale CD
- **`AsyncPipe`** – funktioniert weiterhin, intern nutzt sie `markForCheck()`

Zoneless-Anwendungen starten schneller (kein Zone.js-Overhead beim Booten), verbrauchen weniger Speicher und sind leichter zu debuggen. Der Wechsel ist in bestehenden Apps ein inkrementeller Prozess: Man kann `provideExperimentalZonelessChangeDetection()` aktivieren und Zone.js schrittweise aus dem Projekt entfernen.

**Kernregel im Zoneless-Modus:** Wenn du Daten aktualisierst, die in der UI sichtbar sein sollen, musst du entweder Signals verwenden oder manuell Change Detection anstoßen.

## Aufgabe

Erstelle eine kleine **Timer-Komponente**, die einen laufenden Sekundenzähler anzeigt. Implementiere sie einmal "klassisch" mit Zone.js (funktioniert automatisch) und dann **zoneless** mit einem Signal. Teste anschließend, was passiert, wenn du im Zoneless-Modus eine normale Variable statt eines Signals benutzt.

### Schritte

1. Konfiguriere die App im `bootstrapApplication()`-Aufruf für den Zoneless-Modus via `provideExperimentalZonelessChangeDetection()` und entferne Zone.js aus den `polyfills` in `angular.json`.

2. Erstelle eine Standalone-Komponente `ZonelessTimerComponent` mit:
   - Einem `signal<number>(0)` für den Zählerstand (`count`)
   - Einem `computed()` das die Sekunden in `MM:SS`-Format formatiert
   - Einem `setInterval` in `ngOnInit`, der das Signal jede Sekunde inkrementiert
   - Einem `inject(DestroyRef)` mit `onDestroy()` zum Aufräumen des Intervals

3. Füge zu Demonstrationszwecken eine zweite Property `plainCount = 0` hinzu, die du parallel im selben Interval erhöhst. Zeige beide in der Template-View an und beobachte: Nur `count()` (Signal) aktualisiert sich in der UI – `plainCount` bleibt bei 0.

4. Füge einen Button hinzu, der über `ChangeDetectorRef.detectChanges()` die View manuell aktualisiert. Prüfe, ob danach auch `plainCount` sichtbar wird.

## Hints

<details>
<summary>Hint 1 – Zoneless konfigurieren</summary>

In `main.ts`:
```typescript
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection(),
    // weitere Provider...
  ]
});
```

In `angular.json` unter `projects > architect > build > options`:
```json
"polyfills": []
```
(statt `"polyfills": ["zone.js"]`)

</details>

<details>
<summary>Hint 2 – DestroyRef für Cleanup</summary>

```typescript
import { DestroyRef, inject } from '@angular/core';

export class ZonelessTimerComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    const id = setInterval(() => { /* ... */ }, 1000);
    this.destroyRef.onDestroy(() => clearInterval(id));
  }
}
```

</details>

<details>
<summary>Hint 3 – computed() für Formatierung</summary>

```typescript
import { signal, computed } from '@angular/core';

count = signal(0);

formatted = computed(() => {
  const s = this.count();
  const mm = String(Math.floor(s / 60)).padStart(2, '0');
  const ss = String(s % 60).padStart(2, '0');
  return `${mm}:${ss}`;
});
```

Im Template: `{{ formatted() }}`

</details>

## Beispiellösung

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection(),
  ]
});
```

```typescript
// zoneless-timer.component.ts
import {
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  Component,
  DestroyRef,
  computed,
  inject,
  signal,
  OnInit,
} from '@angular/core';

@Component({
  selector: 'app-zoneless-timer',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>Zoneless Timer</h2>

    <p>Signal-Zähler: <strong>{{ formatted() }}</strong></p>
    <p>Normale Variable: <strong>{{ plainCount }}</strong></p>

    <button (click)="forceDetect()">Manuell aktualisieren</button>
  `,
})
export class ZonelessTimerComponent implements OnInit {
  private cdr = inject(ChangeDetectorRef);
  private destroyRef = inject(DestroyRef);

  count = signal(0);
  plainCount = 0;

  formatted = computed(() => {
    const s = this.count();
    const mm = String(Math.floor(s / 60)).padStart(2, '0');
    const ss = String(s % 60).padStart(2, '0');
    return `${mm}:${ss}`;
  });

  ngOnInit() {
    const id = setInterval(() => {
      this.count.update(c => c + 1); // Signal → triggert automatisch CD
      this.plainCount++;              // Normale Variable → UI bleibt stehen
    }, 1000);

    this.destroyRef.onDestroy(() => clearInterval(id));
  }

  forceDetect() {
    this.cdr.detectChanges(); // Zeigt den aktuellen plainCount-Wert
  }
}
```

**Beobachtung:** Der Signal-Zähler läuft in der UI selbstständig. `plainCount` bleibt bei `0`, bis du den Button klickst – weil ohne Zone.js kein automatischer CD-Trigger existiert.

## Weiterführendes

- **`effect()`** in Kombination mit Zoneless: Side-Effects reagieren auf Signal-Änderungen ohne manuellen CD-Trigger – ideal für Logging, Analytics oder DOM-Manipulationen.
- **NgRx Signal Store** und `@ngrx/signals` sind von Grund auf zoneless-kompatibel gebaut.
- Offizielle Docs: [angular.dev/guide/experimental/zoneless](https://angular.dev/guide/experimental/zoneless)
- Für bestehende Projekte empfiehlt sich ein schrittweiser Wechsel: erst `provideExperimentalZonelessChangeDetection()` aktivieren (ohne Zone.js zu entfernen), alle Komponenten auf Signals umstellen, dann Zone.js entfernen.

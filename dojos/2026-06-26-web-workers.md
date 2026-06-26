# Angular Dojo: Web Workers in Angular
**Datum:** 2026-06-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du rechenintensive Aufgaben aus dem Haupt-Thread in einen Web Worker auslagerst, damit die Angular-UI flüssig bleibt – und wie du die Angular CLI dabei optimal nutzt.

## Hintergrund & Theorie

JavaScript läuft standardmäßig single-threaded. Rechenintensive Operationen (z. B. große Datensätze sortieren, Bildverarbeitung, Kryptographie) blockieren den Haupt-Thread und frieren die UI ein.

**Web Workers** sind Browser-Threads, die parallel zum Haupt-Thread laufen. Sie kommunizieren über ein Message-System (`postMessage` / `onmessage`) und haben **keinen** Zugriff auf das DOM. Angular-Change-Detection läuft im Haupt-Thread – Web Workers berühren sie daher nicht direkt.

Die Angular CLI kann Web Workers automatisch einrichten:
```bash
ng generate web-worker <name>
```
Das erzeugt eine `<name>.worker.ts` und passt `tsconfig.json` an (`"lib": ["webworker"]`).

Im Service instanzierst du den Worker mit:
```typescript
const worker = new Worker(new URL('./my.worker', import.meta.url));
```
Das `import.meta.url`-Muster stellt sicher, dass Webpack/esbuild den Worker korrekt bündelt.

**Typisches Kommunikationsmuster:**
- Haupt-Thread: `worker.postMessage(data)` + `worker.onmessage = e => handle(e.data)`
- Worker: `addEventListener('message', e => { ...; postMessage(result) })`

Für reaktiven Code lässt sich das eleganter in einen `Observable`-basierten Service einwickeln.

## Aufgabe

Erstelle einen Angular-Service, der eine **rechenintensive Primzahlsuche** in einen Web Worker auslagert. Die Komponente soll dabei vollständig responsiv bleiben (Eingaben, Animationen laufen weiter), während der Worker rechnet.

### Schritte

1. **Worker generieren** – Erzeuge via CLI einen Web Worker `prime-finder.worker.ts` in `src/app/`. Implementiere darin die Logik, alle Primzahlen bis zu einer übergebenen Zahl `n` zu finden (Sieb des Eratosthenes oder naiver Ansatz).

2. **Service erstellen** – Erstelle `PrimeWorkerService`. Der Service soll:
   - Den Worker lazy (beim ersten Aufruf) instantiieren
   - Eine Methode `findPrimes(n: number): Observable<number[]>` anbieten
   - Das `onmessage`-Ergebnis des Workers in ein `Observable` (via `fromEvent` oder manuell mit `new Observable(...)`) wrappen
   - Den Worker bei `ngOnDestroy` terminieren

3. **Komponente verdrahten** – Erstelle eine `PrimeDemoComponent` mit:
   - Einem `<input type="number">` für `n` (gebunden via `signal` oder `FormControl`)
   - Einem Button "Berechnen"
   - Einem Ladeindikator (einfaches `isLoading`-Signal)
   - Anzeige der Anzahl gefundener Primzahlen und der Berechnungszeit (ms)
   - Einem kleinen "Heartbeat"-Counter (zählt jede Sekunde hoch via `setInterval`), der beweist, dass die UI während der Berechnung **nicht** einfriert

4. **Vergleich** – Kommentiere kurz in der Komponente, was passieren würde, wenn du denselben Algorithmus **ohne** Worker direkt im Template-Event-Handler aufrufst.

## Hints

<details>
<summary>Hint 1 – Worker-Kommunikation als Observable</summary>

```typescript
findPrimes(n: number): Observable<number[]> {
  return new Observable(observer => {
    this.worker.postMessage(n);
    const handler = (e: MessageEvent<number[]>) => {
      observer.next(e.data);
      observer.complete();
    };
    this.worker.addEventListener('message', handler, { once: true });
    return () => this.worker.removeEventListener('message', handler);
  });
}
```
`{ once: true }` stellt sicher, dass der Handler nur einmal feuert und sich dann selbst entfernt.
</details>

<details>
<summary>Hint 2 – Sieb des Eratosthenes im Worker</summary>

```typescript
// prime-finder.worker.ts
addEventListener('message', ({ data }: MessageEvent<number>) => {
  const n = data;
  const sieve = new Uint8Array(n + 1).fill(1);
  sieve[0] = sieve[1] = 0;
  for (let i = 2; i * i <= n; i++) {
    if (sieve[i]) {
      for (let j = i * i; j <= n; j += i) sieve[j] = 0;
    }
  }
  const primes: number[] = [];
  for (let i = 2; i <= n; i++) if (sieve[i]) primes.push(i);
  postMessage(primes);
});
```
`Uint8Array` spart Speicher gegenüber einem normalen `boolean[]`.
</details>

<details>
<summary>Hint 3 – Lazy Worker-Initialisierung</summary>

```typescript
@Injectable({ providedIn: 'root' })
export class PrimeWorkerService implements OnDestroy {
  private worker?: Worker;

  private getWorker(): Worker {
    if (!this.worker) {
      this.worker = new Worker(
        new URL('./prime-finder.worker', import.meta.url)
      );
    }
    return this.worker;
  }

  ngOnDestroy(): void {
    this.worker?.terminate();
  }
}
```
</details>

## Beispiellösung

```typescript
// prime-finder.worker.ts
/// <reference lib="webworker" />

addEventListener('message', ({ data }: MessageEvent<number>) => {
  const n = data;
  const sieve = new Uint8Array(n + 1).fill(1);
  sieve[0] = sieve[1] = 0;
  for (let i = 2; i * i <= n; i++) {
    if (sieve[i]) {
      for (let j = i * i; j <= n; j += i) sieve[j] = 0;
    }
  }
  const primes: number[] = [];
  for (let i = 2; i <= n; i++) if (sieve[i]) primes.push(i);
  postMessage(primes);
});
```

```typescript
// prime-worker.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class PrimeWorkerService implements OnDestroy {
  private worker?: Worker;

  private getWorker(): Worker {
    if (!this.worker) {
      this.worker = new Worker(
        new URL('./prime-finder.worker', import.meta.url)
      );
    }
    return this.worker;
  }

  findPrimes(n: number): Observable<number[]> {
    const worker = this.getWorker();
    return new Observable(observer => {
      const handler = (e: MessageEvent<number[]>) => {
        observer.next(e.data);
        observer.complete();
      };
      worker.addEventListener('message', handler, { once: true });
      worker.onerror = err => observer.error(err);
      worker.postMessage(n);
      return () => worker.removeEventListener('message', handler);
    });
  }

  ngOnDestroy(): void {
    this.worker?.terminate();
  }
}
```

```typescript
// prime-demo.component.ts
import { Component, signal, OnDestroy } from '@angular/core';
import { PrimeWorkerService } from './prime-worker.service';

@Component({
  selector: 'app-prime-demo',
  standalone: true,
  template: `
    <h2>Primzahl-Finder (Web Worker)</h2>
    <input type="number" [(ngModel)]="limit" min="2" max="10000000" />
    <button (click)="calculate()" [disabled]="isLoading()">Berechnen</button>

    @if (isLoading()) {
      <p>Berechne... (UI-Heartbeat läuft weiter: {{ heartbeat() }})</p>
    } @else {
      <p>Heartbeat: {{ heartbeat() }}</p>
    }

    @if (result()) {
      <p>Gefunden: {{ result()!.count }} Primzahlen in {{ result()!.ms }} ms</p>
    }

    <!--
      OHNE Worker: primes = findPrimesSync(n) direkt im click-Handler
      → UI friert für mehrere Sekunden ein, Heartbeat stoppt,
        keine weiteren Klicks/Eingaben möglich.
    -->
  `,
  imports: [/* FormsModule, ... */],
})
export class PrimeDemoComponent implements OnDestroy {
  protected limit = 1_000_000;
  protected isLoading = signal(false);
  protected heartbeat = signal(0);
  protected result = signal<{ count: number; ms: number } | null>(null);

  private intervalId = setInterval(() => this.heartbeat.update(v => v + 1), 1000);

  constructor(private primeService: PrimeWorkerService) {}

  calculate(): void {
    this.isLoading.set(true);
    const start = performance.now();
    this.primeService.findPrimes(this.limit).subscribe(primes => {
      this.result.set({ count: primes.length, ms: Math.round(performance.now() - start) });
      this.isLoading.set(false);
    });
  }

  ngOnDestroy(): void {
    clearInterval(this.intervalId);
  }
}
```

## Weiterführendes

- **Comlink** (Google Chrome Labs): Bibliothek, die Worker-Kommunikation hinter einem Proxy-Objekt versteckt – kein manuelles `postMessage` mehr nötig. Ideal für komplexere Worker-APIs.
- Für **SharedArrayBuffer** und **Atomics** (thread-safe gemeinsamer Speicher) benötigt die Seite Cross-Origin-Isolation-Header (`COOP`/`COEP`) – relevant bei sehr großen Datentransfers zwischen Threads.
- Angular-Signals funktionieren nur im Haupt-Thread; Worker-Ergebnisse müssen explizit per `postMessage` zurück und dann in ein Signal geschrieben werden.

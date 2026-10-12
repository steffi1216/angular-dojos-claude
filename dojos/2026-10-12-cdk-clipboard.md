# Angular Dojo: CDK Clipboard
**Datum:** 2026-10-12
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, das Angular CDK `Clipboard`-Modul einzusetzen, um Text programmatisch und per Direktive in die Zwischenablage zu kopieren — inkl. Feedback-Handling, langer Texte und eigener Pending-States.

## Hintergrund & Theorie
Das Angular CDK stellt zwei Wege bereit, Text in die Zwischenablage zu schreiben:

**1. `cdkCopyToClipboard`-Direktive** (deklarativ)
Wird direkt auf einem Button oder Element gesetzt. Das `cdkCopied`-Event feuert nach dem Kopierversuch mit einem `boolean` (Erfolg/Misserfolg).

**2. `Clipboard`-Service** (programmatisch)
Wird per DI eingebunden und ermöglicht `clipboard.copy(text)`. Für sehr lange Texte (> ~500 KB im Firefox-Umfeld) gibt es `clipboard.beginCopy(text)` + `pendingCopy.copy()` + `pendingCopy.destroy()`, was das DOM-Element länger am Leben hält.

Das CDK löst selbst das Cross-Browser-Problem (execCommand-Fallback vs. Clipboard API) und bereinigt intern erzeugte `<textarea>`-Elemente nach dem Kopiervorgang automatisch.

**Import:**
```typescript
import { ClipboardModule } from '@angular/cdk/clipboard';
import { Clipboard } from '@angular/cdk/clipboard';
```

## Aufgabe
Erstelle eine `CodeSnippetComponent`, die einen Code-Block mit einem "Kopieren"-Button anzeigt. Nach dem Klick soll der Button-Text für 2 Sekunden auf "Kopiert ✓" wechseln, bevor er zurückgesetzt wird. Implementiere außerdem eine `CopyLargeTextService`-Methode, die für Texte > 1 MB den `PendingCopy`-Mechanismus nutzt.

### Schritte
1. Installiere `@angular/cdk` (falls noch nicht vorhanden) und importiere `ClipboardModule` in der Standalone-Component.
2. Erstelle `CodeSnippetComponent` mit einem Input `code: string` und einem Template, das den Code in einem `<pre>` rendert.
3. Füge einen Button mit `cdkCopyToClipboard` und dem `(cdkCopied)`-Event hinzu. Setze im Event-Handler einen `copied`-State (Signal) für 2 Sekunden auf `true`.
4. Binde den Button-Label dynamisch: `copied() ? 'Kopiert ✓' : 'Kopieren'`.
5. Erstelle einen `CopyService` mit einer Methode `copyLargeText(text: string): boolean`, die `Clipboard.beginCopy` / `PendingCopy.copy` / `PendingCopy.destroy` einsetzt und mindestens 3 Copy-Versuche unternimmt (Retry-Loop), bevor sie aufgibt.

## Hints
<details>
<summary>Hint 1 – cdkCopied-Event</summary>

`cdkCopied` liefert einen `boolean`. Im Template:
```html
<button
  [cdkCopyToClipboard]="code"
  (cdkCopied)="onCopied($event)">
  {{ copied() ? 'Kopiert ✓' : 'Kopieren' }}
</button>
```
Im Handler:
```typescript
onCopied(success: boolean) {
  if (success) {
    this.copied.set(true);
    setTimeout(() => this.copied.set(false), 2000);
  }
}
```
</details>

<details>
<summary>Hint 2 – PendingCopy für große Texte</summary>

```typescript
import { Clipboard, PendingCopy } from '@angular/cdk/clipboard';

copyLargeText(text: string): boolean {
  const pending: PendingCopy = this.clipboard.beginCopy(text);
  let attempts = 0;
  let result = false;
  do {
    result = pending.copy();
  } while (!result && ++attempts < 3);
  pending.destroy();
  return result;
}
```
`beginCopy` erzeugt ein verstecktes `<textarea>` im DOM. `destroy()` muss immer aufgerufen werden, sonst bleibt das Element erhalten.
</details>

## Beispiellösung

```typescript
// code-snippet.component.ts
import { Component, input, signal } from '@angular/core';
import { ClipboardModule } from '@angular/cdk/clipboard';

@Component({
  selector: 'app-code-snippet',
  standalone: true,
  imports: [ClipboardModule],
  template: `
    <div class="snippet-wrapper">
      <pre><code>{{ code() }}</code></pre>
      <button
        [cdkCopyToClipboard]="code()"
        (cdkCopied)="onCopied($event)"
        [class.copied]="copied()">
        {{ copied() ? 'Kopiert ✓' : 'Kopieren' }}
      </button>
    </div>
  `,
  styles: [`
    .snippet-wrapper { position: relative; }
    button { transition: background 0.2s; }
    button.copied { background: #4caf50; color: white; }
  `],
})
export class CodeSnippetComponent {
  code = input.required<string>();
  copied = signal(false);

  private timer: ReturnType<typeof setTimeout> | null = null;

  onCopied(success: boolean): void {
    if (!success) return;
    this.copied.set(true);
    if (this.timer) clearTimeout(this.timer);
    this.timer = setTimeout(() => this.copied.set(false), 2000);
  }
}

// copy.service.ts
import { Injectable } from '@angular/core';
import { Clipboard } from '@angular/cdk/clipboard';

@Injectable({ providedIn: 'root' })
export class CopyService {
  constructor(private clipboard: Clipboard) {}

  copyLargeText(text: string): boolean {
    const pending = this.clipboard.beginCopy(text);
    let attempts = 0;
    let result = false;
    do {
      result = pending.copy();
    } while (!result && ++attempts < 3);
    pending.destroy();
    return result;
  }

  copyText(text: string): boolean {
    return this.clipboard.copy(text);
  }
}

// app.component.ts (Verwendungsbeispiel)
import { Component } from '@angular/core';
import { CodeSnippetComponent } from './code-snippet.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CodeSnippetComponent],
  template: `
    <app-code-snippet
      [code]="sampleCode" />
  `,
})
export class AppComponent {
  sampleCode = `const greet = (name: string) => \`Hello, \${name}!\`;`;
}
```

## Weiterführendes
- [Angular CDK Clipboard Doku](https://material.angular.io/cdk/clipboard/overview) — offizielle Übersicht mit allen Optionen
- Kombiniere `cdkCopyToClipboard` mit `cdkCopyToClipboardAttempts` (Input), um die Anzahl der internen Wiederholversuche zu konfigurieren — nützlich für ältere Browser mit langsamen execCommand-Aufrufen
- Für reaktives Clipboard-Feedback in einer Liste (`*ngFor` / `@for`): Speichere den State pro Item in einer `Map<id, boolean>` statt in einem einzelnen Signal, um mehrere gleichzeitige Hover-States zu unterstützen

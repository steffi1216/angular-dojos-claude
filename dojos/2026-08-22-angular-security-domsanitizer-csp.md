# Angular Dojo: Angular Security – DomSanitizer & Content Security Policy
**Datum:** 2026-08-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular Cross-Site Scripting (XSS) verhindert, wie `DomSanitizer` mit `bypassSecurityTrust*`-Methoden korrekt und sicher eingesetzt wird, und wie eine Content Security Policy (CSP) die Sicherheitslage zusätzlich absichert.

## Hintergrund & Theorie

Angular schützt die Anwendung standardmäßig gegen XSS, indem es alle Interpolationen (`{{ }}`) und Property Bindings (`[src]`, `[href]`) automatisch escaped bzw. sanitisiert. Das funktioniert für den Großteil der Anwendungen ohne weiteres Zutun.

Es gibt jedoch Szenarien, in denen man vertrauenswürdigen HTML-/CSS-/URL-Inhalt aus einer kontrollierten Quelle rendern möchte (z. B. serverseitig generiertes HTML, SVG-Icons, sichere Video-URLs). Dafür bietet Angular `DomSanitizer` mit folgenden Methoden:

| Methode | Kontext |
|---|---|
| `bypassSecurityTrustHtml` | `[innerHTML]` |
| `bypassSecurityTrustStyle` | `[style]` |
| `bypassSecurityTrustUrl` | `[href]`, `[src]` |
| `bypassSecurityTrustResourceUrl` | `<iframe src>`, `<script src>` |
| `bypassSecurityTrustScript` | Inline-Skripte |

**Wichtig:** Diese Methoden umgehen Angulars Sanitizer vollständig. Der Entwickler übernimmt damit die Verantwortung, dass der Inhalt sicher ist. Sie sollten **nie** auf Benutzereingaben angewendet werden.

Eine ergänzende Maßnahme ist die **Content Security Policy (CSP)** als HTTP-Header (oder `<meta>`-Tag), die dem Browser mitteilt, welche Ressourcen geladen und ausgeführt werden dürfen. Seit Angular 16+ unterstützt das Framework **Nonce-basierte CSP**, bei der ein zufälliger `nonce`-Wert für inline styles vergeben wird, anstatt `'unsafe-inline'` zuzulassen.

## Aufgabe

Erstelle eine Angular-Komponente `SafeHtmlDemoComponent`, die dynamischen HTML-Inhalt sicher rendert – und demonstriere dabei den Unterschied zwischen sanitisiertem und unsanitisiertem Rendering.

### Schritte

1. **Erstelle die Komponente** `SafeHtmlDemoComponent` als Standalone-Komponente.

2. **Injiziere `DomSanitizer`** mit der `inject()`-Funktion und erstelle eine Methode `sanitize(html: string): SafeHtml`, die `bypassSecurityTrustHtml` verwendet.

3. **Erstelle im Template** zwei Bereiche:
   - Einen `[innerHTML]`-Binding mit **rohem** String (Angular sanitisiert automatisch → `<script>` wird entfernt).
   - Einen `[innerHTML]`-Binding mit dem Rückgabewert von `bypassSecurityTrustHtml` für vertrauenswürdiges HTML.

4. **Erstelle eine `SafeUrlPipe`** (Pure Pipe), die eine URL für `[src]` oder `[href]` als vertrauenswürdig markiert:
   ```typescript
   transform(url: string): SafeUrl
   ```

5. **Simuliere einen XSS-Angriffsversuch**: Übergib einen String wie `<img src=x onerror="alert('XSS')">` als rohen `[innerHTML]`-Wert und beobachte (im Template-Kommentar), wie Angular die `onerror`-Eigenschaft entfernt.

6. **Ergänze einen CSP-Kommentar** im `index.html` oder einer separaten Datei, der zeigt, wie ein realer CSP-Header aussehen würde:
   ```
   Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-RANDOM'; style-src 'self' 'nonce-RANDOM'; img-src 'self' data:;
   ```

7. **Optional (Bonus):** Erstelle ein `nonce`-Attribut auf einem `<style>`-Tag und zeige, wie Angular's `ngCspNonce`-Provider in `bootstrapApplication` gesetzt wird.

## Hints

<details>
<summary>Hint 1 – DomSanitizer injizieren</summary>

```typescript
import { inject } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

private sanitizer = inject(DomSanitizer);

getSafeHtml(html: string): SafeHtml {
  return this.sanitizer.bypassSecurityTrustHtml(html);
}
```

Merke: Diese Methode **nur** für Inhalte verwenden, die du vollständig kontrollierst (z. B. aus deinem eigenen Backend).

</details>

<details>
<summary>Hint 2 – SafeUrl Pipe</summary>

```typescript
import { Pipe, PipeTransform, inject } from '@angular/core';
import { DomSanitizer, SafeUrl } from '@angular/platform-browser';

@Pipe({ name: 'safeUrl', standalone: true, pure: true })
export class SafeUrlPipe implements PipeTransform {
  private sanitizer = inject(DomSanitizer);

  transform(url: string): SafeUrl {
    return this.sanitizer.bypassSecurityTrustUrl(url);
  }
}
```

Verwendung im Template:
```html
<iframe [src]="videoUrl | safeUrl"></iframe>
```

</details>

<details>
<summary>Hint 3 – ngCspNonce für Angular 16+</summary>

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    { provide: CSP_NONCE, useValue: 'abc123xyz' } // Wert vom Server pro Request generieren
  ]
});
```

Damit setzt Angular den `nonce`-Wert auf alle von ihm generierten `<style>`-Tags, was `'unsafe-inline'` in der CSP überflüssig macht.

</details>

## Beispiellösung

```typescript
// safe-html-demo.component.ts
import { Component, inject } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';
import { SafeUrlPipe } from './safe-url.pipe';

@Component({
  selector: 'app-safe-html-demo',
  standalone: true,
  imports: [SafeUrlPipe],
  template: `
    <h2>Angular Security Demo</h2>

    <!-- Angular sanitisiert automatisch: <script> und onerror werden entfernt -->
    <section>
      <h3>Automatisch sanitisiertes HTML</h3>
      <div [innerHTML]="rawHtml"></div>
      <small>Angreifer-Input: {{ rawHtml }}</small>
    </section>

    <!-- bypassSecurityTrustHtml: Angular vertraut dem Inhalt -->
    <section>
      <h3>Vertrauenswürdiges HTML (bypassSecurityTrustHtml)</h3>
      <div [innerHTML]="trustedHtml"></div>
    </section>

    <!-- SafeUrl Pipe für externe URLs -->
    <section>
      <h3>Vertrauenswürdige URL (SafeUrlPipe)</h3>
      <a [href]="externalUrl | safeUrl" target="_blank" rel="noopener">Externer Link</a>
    </section>
  `,
})
export class SafeHtmlDemoComponent {
  private sanitizer = inject(DomSanitizer);

  // Simulierter XSS-Angriff – Angular entfernt onerror automatisch
  rawHtml = `<b>Fett</b> <img src=x onerror="alert('XSS')"> <script>alert('XSS')</script>`;

  // Vertrauenswürdiger Inhalt vom eigenen Backend
  trustedHtml: SafeHtml = this.sanitizer.bypassSecurityTrustHtml(
    `<b>Willkommen!</b> <em>Dieser Inhalt kommt vom Server.</em>`
  );

  externalUrl = 'https://angular.dev';
}
```

```typescript
// safe-url.pipe.ts
import { Pipe, PipeTransform, inject } from '@angular/core';
import { DomSanitizer, SafeUrl } from '@angular/platform-browser';

@Pipe({ name: 'safeUrl', standalone: true, pure: true })
export class SafeUrlPipe implements PipeTransform {
  private sanitizer = inject(DomSanitizer);

  transform(url: string): SafeUrl {
    return this.sanitizer.bypassSecurityTrustUrl(url);
  }
}
```

```typescript
// main.ts – ngCspNonce Bonus
import { bootstrapApplication } from '@angular/platform-browser';
import { CSP_NONCE } from '@angular/core';
import { AppComponent } from './app/app.component';

// In der Praxis: Nonce vom Server in das HTML gerendert, z. B. window.__nonce__
const nonce = (document.querySelector('meta[name="csp-nonce"]') as HTMLMetaElement)?.content;

bootstrapApplication(AppComponent, {
  providers: [
    { provide: CSP_NONCE, useValue: nonce ?? null },
  ],
});
```

```html
<!-- index.html – CSP als Meta-Tag (nur für lokale Entwicklung; in Prod als HTTP-Header) -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self'; style-src 'self' 'nonce-REPLACE_AT_RUNTIME'; img-src 'self' data:;">
<meta name="csp-nonce" content="REPLACE_AT_RUNTIME">
```

**Was passiert beim Test?**
- `rawHtml` mit `onerror` → Angular entfernt das Attribut, kein Alert ausgelöst.
- `rawHtml` mit `<script>` → Tag wird komplett entfernt.
- `trustedHtml` → Inhalt wird unverändert gerendert (kein `<script>` übergeben!).

## Weiterführendes

- **Angular Security Guide (offiziell):** [angular.dev/best-practices/security](https://angular.dev/best-practices/security) – erklärt alle `bypassSecurityTrust*`-Methoden und wann sie legitim sind.
- **OWASP XSS Prevention Cheat Sheet:** Konkrete Regeln, wie man XSS in Web-Anwendungen vermeidet.
- **CSP mit Nonces:** Statt `'unsafe-inline'` eine server-seitige Nonce-Generierung implementieren (z. B. mit Express-Middleware `helmet`) und den Wert als `CSP_NONCE` providen.
- **Trusted Types API:** Die nächste Sicherheitsstufe – API, die im Browser erzwingt, dass nur typisierte, sichere Werte in DOM-Sinks übergeben werden.

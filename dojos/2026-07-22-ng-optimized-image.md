# Angular Dojo: NgOptimizedImage
**Datum:** 2026-07-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, die `NgOptimizedImage`-Direktive (`ngSrc`) einzusetzen, um Bilder automatisch zu optimieren, LCP-Metriken zu verbessern und maßgeschneiderte Image Loader für CDN-Anbindungen zu schreiben.

## Hintergrund & Theorie

Seit Angular 15 ist `NgOptimizedImage` Teil des Angular-Kerns (`@angular/common`). Die Direktive ersetzt das Standard-`src`-Attribut durch `ngSrc` und übernimmt automatisch:

- **Lazy Loading** per `loading="lazy"` (Standard) und explizites `priority`-Flag für LCP-Bilder
- **Srcset-Generierung** anhand konfigurierbarer Breakpoints, sodass der Browser die passende Auflösung lädt
- **Intrinsic-Size-Enforcement** — `width` und `height` sind Pflicht, was Cumulative Layout Shift (CLS) verhindert
- **Preconnect-Warnung** wenn kein `<link rel="preconnect">` für die Bild-Domain gesetzt ist
- **Fill-Modus** für Bilder, die ihren Container vollständig ausfüllen sollen (kein festes `width`/`height` nötig)
- **Placeholder** (Low Quality Image Placeholder, LQIP) via `placeholder` für ein wahrgenommenes schnelles Laden

Für den Zugriff auf ein CDN (Cloudinary, ImageKit, Imgix, …) stellt Angular fertige `ImageLoader`-Provider bereit. Eigene Loader lassen sich mit einer einzeiligen Factory-Funktion definieren.

## Aufgabe

Baue eine `ImageGalleryComponent`, die eine Liste von Produktbildern darstellt. Das erste Bild ist das Haupt-LCP-Bild, die restlichen werden lazy geladen. Alle Bilder werden über einen eigenen custom Image Loader von einer fiktiven CDN-URL serviert, die automatisch Breite und Format (`webp`) anhängt.

### Schritte

1. **Setup** — Importiere `NgOptimizedImage` in deiner Standalone Component.

2. **Custom Image Loader** erstellen — Schreibe eine `IMAGE_LOADER`-Factory, die URLs nach dem Schema  
   `https://cdn.example.com/<src>?w=<width>&fmt=webp` erzeugt.  
   Registriere den Loader mit `provideImageKitLoader` oder dem Low-Level-Token `IMAGE_LOADER`.

3. **Template** — Zeige das erste Bild mit `priority` (LCP), die restlichen ohne (lazy).  
   Verwende `fill`-Modus für ein Hero-Banner-Bild mit `position: relative`-Container.

4. **Placeholder** — Aktiviere `placeholder` für das LCP-Bild und beobachte das Blur-to-sharp-Verhalten.

5. **Prüfung** — Öffne die Browser-DevTools → Network-Tab und verifiziere:
   - Das Priority-Bild hat `fetchpriority="high"` und kein `loading="lazy"`.
   - Die generierten `srcset`-Werte enthalten mehrere Breiten.
   - Die CDN-URLs tragen `?w=...&fmt=webp`.

## Hints

<details>
<summary>Hint 1 – IMAGE_LOADER Token</summary>

```typescript
import { IMAGE_LOADER, ImageLoaderConfig } from '@angular/common';

export const customCdnLoader = {
  provide: IMAGE_LOADER,
  useValue: (config: ImageLoaderConfig) =>
    `https://cdn.example.com/${config.src}?w=${config.width ?? 'auto'}&fmt=webp`,
};
```

Registriere den Provider in `bootstrapApplication` oder im `providers`-Array der Route.

</details>

<details>
<summary>Hint 2 – Fill-Modus & priority</summary>

Das `fill`-Attribut ersetzt `width`/`height`. Der Container braucht `position: relative` (oder `absolute`/`fixed`) und eine definierte Höhe:

```html
<div style="position: relative; height: 400px;">
  <img ngSrc="hero.jpg" fill priority placeholder />
</div>
```

Für reguläre Bilder mit bekannten Dimensionen:
```html
<img ngSrc="product.jpg" width="800" height="600" priority />
<img ngSrc="other.jpg"   width="400" height="300" />
```

</details>

<details>
<summary>Hint 3 – Srcset-Breakpoints anpassen</summary>

Mit `provideNgOptimizedImageConfig` (seit Angular 17) kannst du die Standard-Breakpoints überschreiben:

```typescript
import { provideNgOptimizedImageConfig } from '@angular/common';

providers: [
  provideNgOptimizedImageConfig({
    breakpoints: [320, 480, 768, 1024, 1280, 1920],
    // disableImageSizeWarning: true,  // nur im Test
  }),
]
```

</details>

## Beispiellösung

```typescript
// custom-cdn-loader.ts
import { IMAGE_LOADER, ImageLoaderConfig } from '@angular/common';

export const customCdnLoader = {
  provide: IMAGE_LOADER,
  useValue: (config: ImageLoaderConfig): string => {
    const width = config.width ? `&w=${config.width}` : '';
    return `https://cdn.example.com/${config.src}?fmt=webp${width}`;
  },
};
```

```typescript
// image-gallery.component.ts
import { Component } from '@angular/core';
import { NgFor, NgOptimizedImage } from '@angular/common';

interface Product {
  src: string;
  alt: string;
  width: number;
  height: number;
}

@Component({
  selector: 'app-image-gallery',
  standalone: true,
  imports: [NgFor, NgOptimizedImage],
  template: `
    <!-- Hero/LCP-Bild im Fill-Modus -->
    <div class="hero-container">
      <img
        ngSrc="hero/banner.jpg"
        fill
        priority
        placeholder
        alt="Hero Banner"
      />
    </div>

    <!-- Produktbilder – lazy per Default -->
    <ul class="product-grid">
      @for (product of products; track product.src; let i = $index) {
        <li>
          <img
            [ngSrc]="product.src"
            [width]="product.width"
            [height]="product.height"
            [alt]="product.alt"
            [priority]="i === 0"
          />
        </li>
      }
    </ul>
  `,
  styles: [`
    .hero-container {
      position: relative;
      height: 400px;
      overflow: hidden;
    }
    .product-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
      gap: 1rem;
      list-style: none;
      padding: 0;
    }
    .product-grid img {
      max-width: 100%;
      height: auto;
    }
  `],
})
export class ImageGalleryComponent {
  products: Product[] = [
    { src: 'products/shoe-1.jpg', alt: 'Rote Sneakers', width: 400, height: 400 },
    { src: 'products/shoe-2.jpg', alt: 'Blaue Turnschuhe', width: 400, height: 400 },
    { src: 'products/bag-1.jpg',  alt: 'Schwarze Tasche',  width: 400, height: 300 },
    { src: 'products/hat-1.jpg',  alt: 'Grüne Mütze',     width: 400, height: 400 },
  ];
}
```

```typescript
// main.ts – Provider-Registrierung
import { bootstrapApplication } from '@angular/platform-browser';
import { provideNgOptimizedImageConfig } from '@angular/common';
import { AppComponent } from './app/app.component';
import { customCdnLoader } from './app/custom-cdn-loader';

bootstrapApplication(AppComponent, {
  providers: [
    customCdnLoader,
    provideNgOptimizedImageConfig({
      breakpoints: [320, 640, 960, 1280, 1920],
    }),
  ],
});
```

## Weiterführendes

- **Fertige CDN-Loader**: Angular liefert `provideCloudflareLoader`, `provideCloudinaryLoader`, `provideImageKitLoader` und `provideImgixLoader` out of the box — kein eigener Loader nötig, wenn du eines dieser CDNs nutzt.
- **`loaderParams`**: Übergib CDN-spezifische Parameter (z. B. Zuschnitt, Qualität) direkt am `<img>`-Tag mit dem Input `[loaderParams]="{ quality: 80, crop: 'fill' }"` und lese sie im Loader via `config.loaderParams` aus.
- **Offizielle Doku**: [angular.dev/guide/image-optimization](https://angular.dev/guide/image-optimization)
- **Core Web Vitals Messungen**: Kombiniere `NgOptimizedImage` mit `web-vitals` (npm) um LCP-Verbesserungen zu quantifizieren.

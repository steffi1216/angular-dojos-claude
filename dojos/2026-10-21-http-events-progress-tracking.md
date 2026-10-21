# Angular Dojo: HttpClient Events & Upload/Download-Fortschritt
**Datum:** 2026-10-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `HttpClient` und der Option `observe: 'events'` den gesamten HTTP-Lebenszyklus – inkl. Upload- und Download-Fortschritt – als Observable beobachten kannst.

## Hintergrund & Theorie

Standardmäßig gibt `HttpClient` nur den Response-Body zurück. Mit `observe: 'response'` erhältst du zusätzlich Header und Statuscode. Mit `observe: 'events'` bekommst du jedoch den **vollständigen Event-Stream** des HTTP-Requests:

- `HttpSentEvent` – Request wurde abgeschickt
- `HttpUploadProgressEvent` – Upload-Fortschritt (% gesendet)
- `HttpDownloadProgressEvent` – Download-Fortschritt (% empfangen)
- `HttpResponseEvent<T>` – vollständige Response
- `HttpUserEvent<T>` – benutzerdefinierter Event

Für Fortschrittsanzeigen muss zusätzlich `reportProgress: true` gesetzt werden. Angular liefert den `HttpEventType`-Enum, um Events typsicher zu unterscheiden.

Das Pattern ist besonders nützlich für:
- Datei-Uploads mit Fortschrittsbalken
- Große Downloads mit Anzeige des Ladefortschritts
- Timeout-Handling auf Basis von Sent/Response-Events

## Aufgabe

Erstelle einen `FileUploadService` und eine `FileUploadComponent`, die eine Datei per POST an einen Mock-Endpoint hochlädt und den Upload-Fortschritt (0–100 %) in Echtzeit anzeigt.

### Schritte

1. Erstelle `FileUploadService` mit einer Methode `uploadFile(file: File): Observable<number>`, die den Fortschritt in Prozent emittiert. Nutze `HttpClient` mit `{ observe: 'events', reportProgress: true }` und filtere/mappe die Events mit `HttpEventType`.

2. Erstelle `FileUploadComponent` als Standalone Component mit:
   - Einem `<input type="file">` zum Auswählen einer Datei
   - Einem `upload()`-Button
   - Einer Fortschrittsanzeige (z.B. `<progress [value]="progress()" max="100">`)
   - Nutze `signal<number>(0)` für den Fortschrittswert

3. Abonniere den Service im Component und update das Signal bei jedem Fortschritts-Event. Handle den abgeschlossenen Upload (100 %) und mögliche Fehler.

4. **Bonus:** Zeige zusätzlich den Download-Fortschritt an, wenn der Server nach dem Upload eine große Datei zurückschickt (simuliere mit `HttpDownloadProgressEvent`).

## Hints

<details>
<summary>Hint 1 – HttpClient-Optionen für Events</summary>

```typescript
this.http.post('/api/upload', formData, {
  reportProgress: true,
  observe: 'events',
})
```

Das gibt ein `Observable<HttpEvent<any>>` zurück. Ohne `reportProgress: true` werden keine `UploadProgressEvent`s emittiert.

</details>

<details>
<summary>Hint 2 – Events filtern und Fortschritt berechnen</summary>

```typescript
import { HttpEventType, HttpUploadProgressEvent } from '@angular/common/http';
import { filter, map } from 'rxjs/operators';

uploadFile(file: File): Observable<number> {
  const formData = new FormData();
  formData.append('file', file);

  return this.http
    .post('/api/upload', formData, { reportProgress: true, observe: 'events' })
    .pipe(
      filter((event): event is HttpUploadProgressEvent =>
        event.type === HttpEventType.UploadProgress
      ),
      map(event => Math.round((100 * event.loaded) / (event.total ?? event.loaded)))
    );
}
```

</details>

<details>
<summary>Hint 3 – Signal im Component updaten</summary>

```typescript
progress = signal(0);
uploadState = signal<'idle' | 'uploading' | 'done' | 'error'>('idle');

upload(file: File) {
  this.uploadState.set('uploading');
  this.fileUploadService.uploadFile(file).subscribe({
    next: (pct) => this.progress.set(pct),
    complete: () => this.uploadState.set('done'),
    error: () => this.uploadState.set('error'),
  });
}
```

</details>

## Beispiellösung

```typescript
// file-upload.service.ts
import { Injectable, inject } from '@angular/core';
import {
  HttpClient,
  HttpEventType,
  HttpUploadProgressEvent,
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { filter, map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class FileUploadService {
  private http = inject(HttpClient);

  uploadFile(file: File): Observable<number> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http
      .post('/api/upload', formData, {
        reportProgress: true,
        observe: 'events',
      })
      .pipe(
        filter((event): event is HttpUploadProgressEvent =>
          event.type === HttpEventType.UploadProgress
        ),
        map(event =>
          Math.round((100 * event.loaded) / (event.total ?? event.loaded))
        )
      );
  }
}

// file-upload.component.ts
import { Component, inject, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FileUploadService } from './file-upload.service';

@Component({
  selector: 'app-file-upload',
  standalone: true,
  imports: [CommonModule],
  template: `
    <input type="file" #fileInput (change)="onFileSelected(fileInput.files)" />

    <button
      [disabled]="!selectedFile() || uploadState() === 'uploading'"
      (click)="upload()"
    >
      Hochladen
    </button>

    @if (uploadState() === 'uploading' || uploadState() === 'done') {
      <div>
        <progress [value]="progress()" max="100"></progress>
        <span>{{ progress() }}%</span>
      </div>
    }

    @if (uploadState() === 'done') {
      <p>Upload abgeschlossen!</p>
    }

    @if (uploadState() === 'error') {
      <p>Fehler beim Upload.</p>
    }
  `,
})
export class FileUploadComponent {
  private uploadService = inject(FileUploadService);

  selectedFile = signal<File | null>(null);
  progress = signal(0);
  uploadState = signal<'idle' | 'uploading' | 'done' | 'error'>('idle');

  onFileSelected(files: FileList | null) {
    this.selectedFile.set(files?.[0] ?? null);
    this.progress.set(0);
    this.uploadState.set('idle');
  }

  upload() {
    const file = this.selectedFile();
    if (!file) return;

    this.uploadState.set('uploading');
    this.progress.set(0);

    this.uploadService.uploadFile(file).subscribe({
      next: pct => this.progress.set(pct),
      complete: () => {
        this.progress.set(100);
        this.uploadState.set('done');
      },
      error: () => this.uploadState.set('error'),
    });
  }
}
```

## Weiterführendes

- Für vollständige Response-Header und Status: `observe: 'response'` statt `'events'` verwenden – gibt `HttpResponse<T>` zurück
- `HttpDownloadProgressEvent` funktioniert nur, wenn der Server `Content-Length`-Header setzt und Angular die Response als Text/Arraybuffer streamt (`responseType: 'text'` + `reportProgress: true`)
- Für Abbruch eines laufenden Uploads: `takeUntil(cancel$)` im Pipe nutzen – Angular sendet automatisch einen `AbortController`-Signal an den Browser
- Offizielle Doku: [Angular – HTTP: Tracking and showing request progress](https://angular.dev/guide/http/making-requests#http-request-progress)

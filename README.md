# Songbook Studio Vue

A static Vue 3 application for managing song lyrics and arranging them into a printable songbook. It uses JavaScript, Vue, and Vite; there is no Java service or database server.

## Run locally

1. Install Node.js (includes npm).
2. Open this folder in Visual Studio Code.
3. In **Terminal → New Terminal**, run:

   ```sh
   npm install
   npm run dev
   ```

4. Open the local URL Vite prints in the terminal.

## Build for publishing

Run `npm run build`. Upload the contents of the generated `dist` folder to a static website host. `npm run preview` previews the production build locally.

## Song data and privacy

Songs and songbook order are stored as JSON in the browser's local storage. This is per browser and device; it does not synchronize between visitors or devices. Use **Download JSON backup** and **Import JSON** to move data. Publishing the app does not publish private song data.

## Exporting

- **Word document** downloads a `.doc` file that Microsoft Word can open.
- **Download PDF** opens the browser print dialog. Choose **Save as PDF**.

The app UI and exports use system fonts and local code, so it does not require external web assets at runtime.

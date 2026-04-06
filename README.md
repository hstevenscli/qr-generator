# Slop Coded QR Generator (Frontend-Only Svelte[Pretty much vanilla JS] because these robots have a hard time writing svelte)

Simple web app to generate QR codes with:
- custom foreground/background colors
- full directional gradients (horizontal, vertical, diagonal)
- optional center logo overlay
- QR eye/finder styling (outer/inner shapes and colors)
- one-click eye style presets (classic, soft rounded, circular eyes, dot matrix, and more)

## Tech Stack

- Frontend: Svelte + Vite + `qrcode`

## Project Structure

- `frontend` - standalone Svelte UI (QR generation runs in browser)
- `backend` - optional legacy Go server (no longer required)

## Frontend Run

```bash
cd frontend
npm install
npm run dev
```

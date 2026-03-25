---
title: "LibGDX – Lezione 3: Input, Tastiera e Mouse"
date: 2024-01-01
draft: false
tags: ["libgdx", "java", "gamedev", "input"]
categories: ["Laboratorio di Programmazione 1"]
description: "Come gestire tastiera, mouse e controller in LibGDX tramite Gdx.input."
Cosa ricordate?
Prima di iniziare — domande rapide sulla lezione precedente.
Cos'è il game loop e quante volte si ripete al secondo?
Cosa fa `ScreenUtils.clear()` e dove va chiamato?
Qual è la differenza tra `SpriteBatch` e `ShapeRenderer`?
Cosa succede se non chiami `dispose()`?
---
Il Sistema di Input in LibGDX
`Gdx.input` è l'oggetto che raccoglie tutti gli eventi di input — tastiera, mouse e controller.
Dispositivo	Metodi principali
Tastiera	`isKeyPressed()`, `isKeyJustPressed()`
Mouse	`justTouched()`, `getX()` / `getY()`
Controller	`gdx-controllers`, `Controller.getAxes()`
> **Regola d'oro:** Tutti i metodi di input si leggono dentro `render()`.
---
`isKeyPressed()` vs `isKeyJustPressed()`
La differenza più importante — e la più comune fonte di bug.
`isKeyPressed()`
Restituisce `true` finché il tasto è tenuto premuto. Ideale per movimento continuo.
```java
if (Gdx.input.isKeyPressed(Input.Keys.SPACE)) {
    // Flappy Bird vola SEMPRE se tieni premuto ⚠️
}
```
`isKeyJustPressed()`
Restituisce `true` solo nel frame in cui il tasto viene premuto, poi torna `false`.
```java
if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE)) {
    // Un flap per pressione — comportamento corretto ✅
}
```
---
Live Coding — Il Flap con la Tastiera
Aggiungiamo il flap a Flappy Bird usando `SPACE` con `isKeyJustPressed()`.
Step 1 — Aggiungi i campi per la fisica
```java
float velY = 0;
float GRAVITY = -500;
float FLAP_FORCE = 200;
```
Step 2 — In `render()`, input + fisica
```java
float dt = Gdx.graphics.getDeltaTime();

if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE))
    velY = FLAP_FORCE;

velY   += GRAVITY * dt;
flappyY += velY * dt;
```
---
La Gravità — come funziona
La gravità è un'accelerazione costante verso il basso. Agisce sulla velocità, non sulla posizione.
```
FLAP                  OGNI FRAME              POSIZIONE
──────────────        ──────────────          ──────────────
velY = +400     →     velY += -800 × dt  →    birdY += velY × dt
(forza verso l'alto)  (gravità abbassa velY)  (posizione aggiornata)
```
> **Perché `-500`?** In LibGDX l'asse Y parte dal basso, quindi i valori negativi indicano la direzione verso il basso.
---
Live Coding — Flap con il Mouse
Aggiungiamo un secondo modo di flappare: il click del mouse.
Sostituiamo (o affianchiamo) il controllo `SPACE` con `justTouched()`:
```java
// Prima: solo tastiera
if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE))
    velY = FLAP_FORCE;

// Ora: tastiera OPPURE mouse
if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE) || Gdx.input.justTouched())
    velY = FLAP_FORCE;
```
> **Prova live:** sostituisci `justTouched()` con `isTouched()` — cosa cambia nel comportamento di Flappy?
---
Codice Completo — Flappy con Gravità
Tutto insieme: sprite, fisica, tastiera e mouse in un unico `render()`.
Campi della classe
```java
float birdY = 200, velY = 0;
float GRAVITY = -500, FLAP_FORCE = 350;
```
In `render()`
```java
float dt = Gdx.graphics.getDeltaTime();

if (Gdx.input.isKeyJustPressed(Input.Keys.SPACE) || Gdx.input.justTouched())
    velY = FLAP_FORCE;

velY  += GRAVITY * dt;
birdY += velY * dt;

ScreenUtils.clear(0, 0, 0, 1);
batch.begin();
batch.draw(bgTex,     0,   0,     280, 480);
batch.draw(flappyTex, 120, birdY,  40,  40);
batch.end();
```
---
Input.Keys — Tasti Comuni
I tasti più usati nello sviluppo di giochi con LibGDX.
Costante	Descrizione
`Input.Keys.SPACE`	Barra spaziatrice
`Input.Keys.UP` / `DOWN`	Frecce direzionali
`Input.Keys.LEFT` / `RIGHT`	Frecce sinistra/destra
`Input.Keys.W` / `A` / `S` / `D`	Controlli WASD
`Input.Keys.ESCAPE`	Uscita / pausa
`Input.Keys.ENTER`	Conferma
`Input.Keys.NUM_0..9`	Tasti numerici
`Input.Keys.ANY_KEY`	Qualsiasi tasto
---
Riepilogo — Input in LibGDX
Metodo	Quando usarlo
`isKeyPressed()`	Tasto tenuto — per movimento continuo
`isKeyJustPressed()`	Tasto premuto una volta — per azioni singole come il flap
`justTouched()`	Click del mouse — equivalente di `isKeyJustPressed` per il mouse
`Controllers.getControllers()`	Recupera i gamepad collegati — usa `isEmpty()` prima di accedere
Flag booleano `flap`	Pattern per unificare più sorgenti di input in un punto solo

---
title: "LibGDX: Sviluppo Game Java"
date: 2026-03-25
description: "Come configurare l'ambiente e creare il primo ciclo di gioco (Game Loop)."
draft: false
---

### Cos'è LibGDX?
LibGDX è un framework open-source per lo sviluppo di videogiochi cross-platform in **Java**. A differenza di Unity, non ha un editor visuale: tutto viene gestito tramite codice.

#### Il Ciclo di Vita (Life Cycle)
Ogni gioco in LibGDX segue un ciclo preciso. Ecco i metodi fondamentali della classe `ApplicationAdapter`:

```java
public class MyGame extends ApplicationAdapter {
    @Override
    public void create () {
        // Carica le immagini e i suoni qui (eseguito una volta)
    }

    @Override
    public void render () {
        // Qui avviene la magia: eseguito 60 volte al secondo!
        ScreenUtils.clear(1, 0, 0, 1); // Pulisce lo schermo
    }
}
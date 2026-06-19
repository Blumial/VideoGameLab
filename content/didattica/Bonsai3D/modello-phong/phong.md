---
title: "Il modello di illuminazione di Phong"
description: "Storia, teoria e implementazione del modello Phong"
date: 2026-06-19
draft: false
tags: ["opengl", "shader", "illuminazione", "bonsai3d"]
categories: ["Grafica 3D"]
showTableOfContents: true
---

# Il modello di illuminazione di Phong

> Perché una mela e una pallina di metallo sembrano diverse sotto la stessa luce?

---

## Un po' di storia — chi era Phong

Il modello che porta questo nome nasce da una persona reale, con una storia breve ma straordinaria.

Bui Tuong Phong nacque a Hanoi il 14 dicembre 1942. Si trasferì negli Stati Uniti, all'Università dello Utah, dove ottenne il dottorato nel 1973 sotto la supervisione di Ivan Sutherland e David Evans — due dei fondatori della computer grafica moderna.

Nella sua tesi di dottorato del 1973, intitolata *"Illumination for Computer-Generated Images"*, Phong propose un modello empirico di illuminazione che combina tre componenti — riflessione ambientale per la luce indiretta uniforme, riflessione diffusa per la dispersione opaca basata sull'orientamento della superficie, e riflessione speculare per i riflessi lucidi che dipendono dalla posizione di chi osserva.

Prima del suo lavoro, le immagini generate al computer erano piatte e prive di profondità — ogni superficie aveva un colore uniforme, senza alcuna sensazione di forma tridimensionale. Phong, insieme a Robert McDermott, Jim Clark e Raphael Rom, creò la prima immagine generata al computer che assomigliasse davvero al suo modello fisico — il celebre maggiolino Volkswagen, un'immagine che divenne un punto di riferimento storico nella computer grafica.

Phong morì nel luglio 1975, a soli 32 anni — per leucemia, poco dopo aver accettato una cattedra a Stanford. Non vide mai l'impatto enorme che il suo modello avrebbe avuto: oggi resta uno degli elementi fondamentali di qualunque pipeline di rendering, nonostante la sua relativa semplicità matematica. Più di cinquant'anni dopo, ogni motore grafico moderno — incluso quello che stai costruendo — calcola ancora la luce seguendo la stessa idea che Phong descrisse in quella tesi.

---

## Un esempio reale — gli occhi dei personaggi Pixar

Negli studi di animazione 3D — Pixar, DreamWorks, qualsiasi produzione moderna — il riflesso speculare (lo "specular highlight") negli occhi dei personaggi è uno degli elementi più curati di tutta la scena.

È quel piccolo punto bianco lucido che vedi nella pupilla di un personaggio animato. Sembra un dettaglio minuscolo, ma è uno dei segnali visivi più potenti che il cervello umano usa per percepire vita e profondità in un volto — senza quel punto gli occhi sembrano spenti e morti.

Un caso documentato durante la produzione de *Il Pianeta delle Scimmie* è particolarmente illuminante: gli animatori scoprirono che **cambiando solo la dimensione dello specular highlight negli occhi del personaggio Caesar**, senza toccare nient'altro del modello, il personaggio poteva sembrare percepito come maschile o femminile dallo spettatore. Solo la dimensione di un riflesso.

Questo perché lo specular non è un dettaglio decorativo — è **l'unico componente del modello di Phong che dice all'occhio dove si trova la fonte di luce e qual è la forma curva della superficie**. Una cornea (la parte trasparente dell'occhio) è leggermente bombata — quella curvatura è esattamente ciò che la formula `reflect()` e `viewDir` calcolano matematicamente.

Lo stesso principio — alla base di un occhio animato o di una sfera di metallo in un videogioco — è quello che costruiremo passo per passo in questo documento.

---

## Applicazione pratica — cosa vedi in un gioco

Guarda due oggetti nella stessa scena, sotto la stessa luce:

**Un blocco di legno**
La luce sembra "appoggiarsi" sulla superficie. Da qualsiasi angolo tu guardi, le facce illuminate restano chiare e quelle in ombra restano scure — il colore non cambia muovendo la camera.

**Una sfera di metallo lucido**
Vedi un piccolo punto bianco intenso che si sposta sulla superficie mentre ti muovi attorno all'oggetto. Quel punto è il riflesso diretto della luce — e segue il tuo occhio.

Questa differenza visiva — superficie opaca contro superficie lucida — è esattamente quello che il modello di Phong permette di simulare con tre numeri: **ambient**, **diffuse**, **specular**.

```
colore_finale = ambient + diffuse + specular
```

---

## Teoria — cosa succede fisicamente

### Ambient — la luce che rimbalza ovunque

Anche un angolo della stanza che non riceve luce diretta non è completamente nero — la luce rimbalza tra le pareti, il soffitto, gli altri oggetti, e arriva comunque, attenuata. L'ambient simula questa illuminazione di fondo con una costante.

### Diffuse — la luce che si disperde

Una superficie ruvida — legno, gesso, carta — ha micro-irregolarità che disperdono la luce incidente in **tutte le direzioni**. Il risultato: non importa da dove guardi, vedi sempre la stessa intensità su quella faccia. Conta solo l'angolo tra la superficie e la luce.

### Specular — la luce che riflette in una direzione

Una superficie liscia — plastica, metallo, vetro, acqua — si comporta più come uno specchio. Il raggio di luce riflette in una **direzione precisa**, calcolabile con la legge della riflessione (angolo di incidenza = angolo di riflessione). Il punto lucido appare solo se la camera si trova esattamente in quella direzione.

| | Diffuse | Specular |
|---|---|---|
| Comportamento fisico | luce dispersa in tutte le direzioni | luce riflessa in una direzione |
| Dipende dalla camera | No | Sì |
| Materiali tipici | legno, gesso, tessuto, carta | plastica, metallo, vetro, acqua |

---

## Concetto matematico — il prodotto scalare

Entrambi i componenti diffuse e specular si calcolano con lo stesso strumento — il **prodotto scalare (dot product)** — applicato a coppie di vettori diverse.

### Diffuse — normal · lightDir

```
intensità = dot(normale, direzione_luce)
```

| Angolo tra normale e luce | Dot product | Intensità |
|---|---|---|
| 0° — stessa direzione | 1.0 | massima |
| 90° — perpendicolari | 0.0 | nulla |
| 180° — direzioni opposte | -1.0 (clampato a 0) | nulla |

Il processo seguito per ogni frammento:

```
1. Prendi il frammento
2. Leggi la sua normale (interpolata dal rasterizzatore)
3. Calcoli lightDir per quel frammento — normalize(posizioneLuce - posizioneFrammento)
4. Calcoli dot(normale, lightDir)
5. Se i due vettori sono allineati nella stessa direzione → intensità massima
```

### Specular — viewDir · reflectDir

Il calcolo risponde a una domanda diversa: *"il riflesso della luce punta verso il mio occhio?"*

```
reflectDir = reflect(-lightDir, normale)   // dove rimbalza la luce
intensità  = dot(viewDir, reflectDir)      // l'occhio è lì?
```

`reflect()` applica la legge della riflessione — restituisce il vettore che descrive dove rimbalza il raggio di luce rispetto alla normale della superficie. `viewDir` è la direzione dal frammento verso la camera.

Il risultato viene elevato a potenza per concentrare il punto lucido:

```
specular = pow(max(dot(viewDir, reflectDir), 0.0), shininess)
```

| Shininess | Effetto |
|---|---|
| basso (2-8) | riflesso largo e diffuso |
| alto (128+) | punto piccolo e intenso |

---

## Codice — implementazione in GLSL

### Vertex Shader
```glsl
#version 410 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;

uniform mat4 uModel;
uniform mat4 uView;
uniform mat4 uProjection;

out vec3 FragNormal;
out vec3 FragPosition;

void main()
{
    FragPosition = vec3(uModel * vec4(aPos, 1.0));
    FragNormal   = aNormal;
    gl_Position  = uProjection * uView * uModel * vec4(aPos, 1.0);
}
```

### Fragment Shader
```glsl
#version 410 core
in vec3 FragNormal;
in vec3 FragPosition;
out vec4 FragColor;

uniform vec4  BaseColor;
uniform vec3  uLightPosition;
uniform vec3  uLightColor;
uniform float uLightIntensity;
uniform vec3  uCameraPosition;
uniform float uShininess;

void main()
{
    vec3 normal   = normalize(FragNormal);
    vec3 lightDir = normalize(uLightPosition - FragPosition);
    vec3 viewDir  = normalize(uCameraPosition - FragPosition);

    // Ambient
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * uLightColor;

    // Diffuse — normal · lightDir
    float diff   = max(dot(normal, lightDir), 0.0);
    vec3 diffuse = diff * uLightColor * uLightIntensity;

    // Specular — viewDir · reflectDir
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec       = pow(max(dot(viewDir, reflectDir), 0.0), uShininess);
    vec3 specular     = spec * uLightColor;

    vec3 result = (ambient + diffuse + specular) * BaseColor.rgb;
    FragColor   = vec4(result, BaseColor.a);
}
```

### Codice C# — passaggio delle uniform

```csharp
SetUniform("uCameraPosition", CameraView.Position);
SetUniform("uShininess", Shininess); // proprietà del materiale, es. 32.0f
```

---

## In sintesi

```
diffuse  → dot(normal, lightDir)              → "quanta luce arriva qui?"
specular → dot(viewDir, reflectDir)           → "il riflesso punta verso il mio occhio?"
```

Lo stesso strumento matematico — il prodotto scalare — risponde a due domande fisiche diverse, applicato a coppie di vettori diverse. È quello che rende il modello di Phong elegante: poche righe di codice per simulare due comportamenti fisici completamente distinti della luce.

## Fonti

- Bui, Tuong-Phong. *Illumination for Computer-Generated Images*. PhD thesis, University of Utah, 1973.
- Kim, Oh, Tran. *The Life and Legacy of Bui Tuong Phong*. ACM SIGGRAPH History Archives, 2024.
- Wikipedia — Bui Tuong Phong, Phong reflection model, Phong shading

Documento redatto con l'assistenza di Claude (Anthropic) per la ricerca
storica e la strutturazione dei contenuti.
---
title: "UniformMatrix4: come i dati viaggiano dalla CPU alla GPU"
date: 2025-01-01
draft: false
tags: ["opengl", "csharp", "silk.net", "shader", "grafica-3d", "glsl"]
description: "Una guida pratica per sviluppatori C# su come funziona UniformMatrix4fv: parametri, flusso dati, errori comuni e glossario."
---

## Introduzione

Quando scrivi `gl.UniformMatrix4(...)` nel tuo codice C#, stai eseguendo
una delle operazioni più importanti nella pipeline grafica: **stai scrivendo
sulla lavagna della GPU** i calcoli che la CPU ha appena completato.

Questa chiamata è il ponte tra la logica del tuo programma e ciò che appare
sullo schermo. Senza di essa, lo shader non saprebbe dove, come e quanto
ruotare o traslare i tuoi oggetti 3D.

In questa guida analizzeremo ogni parametro nel dettaglio, vedremo il flusso
completo dei dati e raccoglieremo gli errori più comuni in cui si incorre.

---

## La firma del metodo

```csharp
gl.UniformMatrix4(location, count, transpose, in matrixData);
```

In Silk.NET, la firma completa è:

```csharp
public void UniformMatrix4(
    int location,       // indirizzo dello uniform nello shader
    uint count,         // quante matrici stai inviando
    bool transpose,     // invertire righe e colonne?
    in float matrixData // puntatore ai 16 float della matrice
)
```

---

## Analisi dei parametri

### 1. `location` — L'indirizzo del cassetto

```csharp
// Durante OnLoad — recuperi l'indirizzo una volta sola
_uModelLocation = gl.GetUniformLocation(shader.Handle, "uModel");

// Durante OnRender — lo riusi ad ogni frame
gl.UniformMatrix4(_uModelLocation, 1, false, ref matrixData[0]);
```

`location` è un intero che OpenGL usa come **indice interno** per sapere
dove si trova la variabile `uModel` dentro lo shader compilato. Puoi
immaginarlo come il numero di uno scaffale in un magazzino: non ti interessa
cosa c'è vicino, vuoi solo aprire *quel* cassetto specifico.

> **Nota pratica:** `GetUniformLocation` restituisce `-1` se il nome non
> esiste nello shader o se la variabile è stata ottimizzata via dal
> compilatore GLSL perché non usata. Controlla sempre questo valore durante
> il debug.

```csharp
// Buona abitudine: validare la location durante lo sviluppo
_uModelLocation = gl.GetUniformLocation(shader.Handle, "uModel");
if (_uModelLocation == -1)
    Console.WriteLine("[WARN] uModel non trovato nello shader!");
```

---

### 2. `count` — Quante matrici

```csharp
gl.UniformMatrix4(_uModelLocation, 1, false, ref matrixData[0]);
//                                 ^ una sola matrice
```

Nel caso più comune invii **una singola matrice** (count = 1), quella che
descrive posizione, rotazione e scala del tuo oggetto.

Esistono però scenari in cui è utile inviare **un array di matrici** in
una sola chiamata:

| Scenario | Count tipico |
|---|---|
| Oggetto singolo | `1` |
| Skeletal Animation (ossa) | `32` – `256` |
| Instanced Rendering | `N` (quante istanze) |

Per il Skeletal Animation, ad esempio, ogni osso del personaggio ha la
propria matrice di trasformazione. Inviarle tutte insieme è molto più
efficiente che fare `N` chiamate separate.

---

### 3. `transpose` — L'ordine delle matrici

Questo parametro è fonte di confusione per chi viene dal mondo C#. Ecco
perché esiste:

| Libreria | Ordine in memoria |
|---|---|
| C# / .NET (e Silk.NET) | **Row-Major** (riga per riga) |
| OpenGL / GLSL | **Column-Major** (colonna per colonna) |

In una matrice **Row-Major**, i dati sono disposti così in memoria:

```
[m00, m01, m02, m03,  // riga 0
 m10, m11, m12, m13,  // riga 1
 m20, m21, m22, m23,  // riga 2
 m30, m31, m32, m33]  // riga 3
```

In una matrice **Column-Major**, l'ordine è invece per colonne:

```
[m00, m10, m20, m30,  // colonna 0
 m01, m11, m21, m31,  // colonna 1
 m02, m12, m22, m32,  // colonna 2
 m03, m13, m23, m33]  // colonna 3
```

**Perché usiamo `false`?**
Silk.NET (tramite `System.Numerics.Matrix4x4`) memorizza già le matrici
in formato compatibile con OpenGL. Il driver non deve quindi invertire
nulla: passiamo `false` e i dati arrivano corretti.

> **Attenzione:** Se usi un'altra libreria matematica (es. una tua
> implementazione custom), potresti dover passare `true`. Un oggetto che
> appare distorto o nella posizione sbagliata è spesso sintomo di un
> problema di trasposizione.

---

### 4. `matrixData` — I 16 float che muovono il mondo

```csharp
// Esempio completo: costruire e inviare una matrice di trasformazione
var model = Matrix4x4.Identity;
model *= Matrix4x4.CreateRotationZ(MathF.PI / 4f);   // 45° di rotazione
model *= Matrix4x4.CreateTranslation(xOffset, 0, 0); // spostamento sull'asse X

// Estrarre i 16 float e inviarli
gl.UniformMatrix4(_uModelLocation, 1, false, ref model.M11);
```

`matrixData` contiene i **16 numeri in virgola mobile** (4×4) che descrivono
completamente la trasformazione dell'oggetto: rotazione, traslazione e scala.

Quando OpenGL riceve questi dati, li copia fisicamente nella **memoria
uniforme della GPU** (Uniform Storage / Constant Buffer). Da quel momento
la variabile `uniform mat4 uModel` dentro lo shader GLSL ha un valore reale
e può essere usata per ogni vertice.

```glsl
// Vertex Shader (GLSL) — come viene usata la matrice
#version 330 core

layout(location = 0) in vec3 aPosition;

uniform mat4 uModel;      // <-- riceve i tuoi 16 float
uniform mat4 uView;
uniform mat4 uProjection;

void main()
{
    // La moltiplicazione applica rotazione + traslazione + scala
    gl_Position = uProjection * uView * uModel * vec4(aPosition, 1.0);
}
```

---

## Il flusso completo dei dati

```
┌─────────────────────────────────────────────────────────────────┐
│  CPU (il tuo codice C#)                                         │
│                                                                 │
│  1. Calcola la matrice                                          │
│     Matrix4x4.CreateRotationZ(...) * CreateTranslation(...)     │
│                                                                 │
│  2. Chiama UniformMatrix4(location, 1, false, ref matrix.M11)   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │  Bus PCIe
                             │  (trasferimento fisico dei dati)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Driver Video                                                   │
│                                                                 │
│  3. Riceve i 16 float                                           │
│  4. Valida e impacchetta i dati                                 │
│  5. Invia alla VRAM tramite DMA                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  GPU                                                            │
│                                                                 │
│  6. Salva i dati nel Constant Buffer (memoria ad alta vel.)     │
│  7. Al momento della Draw Call, ogni Vertex Shader legge        │
│     uModel e applica la trasformazione al vertice corrente      │
└─────────────────────────────────────────────────────────────────┘
```

> **Perché la chiamata va fatta in `OnRender` e non in `OnLoad`?**
> Perché ogni frame il valore di `xOffset` (o di qualsiasi altra variabile
> di stato) può essere cambiato. Se non aggiorni la matrice ad ogni frame,
> la GPU continua ad usare l'ultima matrice ricevuta: il tuo oggetto sembra
> "congelato" anche se la logica C# sta calcolando nuove posizioni.

---

## Esempio completo: quadrato in rotazione

```csharp
public class SquareRenderer
{
    private int _shaderHandle;
    private int _vao;
    private int _uModelLocation;
    private float _angle = 0f;

    public void OnLoad(GL gl)
    {
        // 1. Compila e linka gli shader
        _shaderHandle = CreateShaderProgram(gl, vertexSource, fragmentSource);

        // 2. Recupera la location di uModel — una volta sola
        _uModelLocation = gl.GetUniformLocation(_shaderHandle, "uModel");
        if (_uModelLocation == -1)
            throw new Exception("Uniform 'uModel' non trovato nello shader.");

        // 3. Carica la geometria del quadrato nel VAO
        _vao = CreateQuadVao(gl);
    }

    public void OnUpdate(double deltaTime)
    {
        // Aggiorna l'angolo di rotazione (60°/sec)
        _angle += (float)(deltaTime * 60.0 * MathF.PI / 180f);
    }

    public void OnRender(GL gl)
    {
        gl.UseProgram(_shaderHandle);
        gl.BindVertexArray(_vao);

        // 4. Costruisci la matrice con la rotazione aggiornata
        var model = Matrix4x4.CreateRotationZ(_angle);

        // 5. Invia i 16 float alla GPU
        gl.UniformMatrix4(_uModelLocation, 1, false, ref model.M11);

        // 6. Disegna
        gl.DrawElements(PrimitiveType.Triangles, 6, DrawElementsType.UnsignedInt, 0);
    }
}
```

---

## Errori comuni e come risolverli

### ❌ L'oggetto non si muove

**Causa più probabile:** stai chiamando `UniformMatrix4` in `OnLoad` invece
che in `OnRender`.

```csharp
// ❌ Sbagliato — la matrice viene inviata una volta sola al caricamento
public void OnLoad(GL gl)
{
    var model = Matrix4x4.CreateTranslation(xOffset, 0, 0);
    gl.UniformMatrix4(_uModelLocation, 1, false, ref model.M11);
}

// ✅ Corretto — la matrice viene aggiornata ad ogni frame
public void OnRender(GL gl)
{
    var model = Matrix4x4.CreateTranslation(xOffset, 0, 0);
    gl.UniformMatrix4(_uModelLocation, 1, false, ref model.M11);
}
```

---

### ❌ L'oggetto appare distorto o fuori posizione

**Causa più probabile:** problema di trasposizione o ordine errato nelle
moltiplicazioni di matrice.

```csharp
// ❌ Sbagliato — l'ordine delle moltiplicazioni conta!
var model = Matrix4x4.CreateTranslation(pos) * Matrix4x4.CreateRotationZ(angle);

// ✅ Corretto — prima ruota, poi trasla (leggi da destra a sinistra)
var model = Matrix4x4.CreateRotationZ(angle) * Matrix4x4.CreateTranslation(pos);
```

---

### ❌ `GetUniformLocation` restituisce `-1`

**Cause possibili:**

1. Il nome della variabile è scritto diversamente tra C# e GLSL
   (`"uModel"` vs `"umodel"` — GLSL è **case-sensitive**).
2. La variabile uniform è dichiarata nello shader ma non viene mai usata
   nel `main()`: il compilatore GLSL la elimina come ottimizzazione.
3. Stai chiamando `GetUniformLocation` prima di aver chiamato `gl.UseProgram`.

```csharp
// ✅ Assicurati di avere il programma attivo prima di cercare le location
gl.UseProgram(_shaderHandle);
_uModelLocation = gl.GetUniformLocation(_shaderHandle, "uModel");
```

---

### ❌ Performance basse con molti oggetti

**Causa:** una chiamata `UniformMatrix4` per oggetto è costosa se hai
migliaia di oggetti.

**Soluzione:** usa **Instanced Rendering** — invia tutte le matrici in
un'unica chiamata con `count > 1`, oppure usa un **Uniform Buffer Object
(UBO)** per aggiornamenti in batch.

---

## Glossario

| Termine | Definizione |
|---|---|
| **Uniform** | Variabile nello shader che rimane costante per tutti i vertici di una singola Draw Call. Si aggiorna dalla CPU tramite chiamate come `UniformMatrix4`. |
| **mat4** | Tipo GLSL per una matrice 4×4 di float. Contiene 16 valori che codificano traslazione, rotazione e scala. |
| **Vertex Shader** | Programma eseguito dalla GPU per ogni singolo vertice. Riceve la posizione grezza e la trasforma usando le matrici uniform. |
| **Draw Call** | Il comando (`DrawArrays` o `DrawElements`) che ordina alla GPU di elaborare i vertici e produrre i pixel. |
| **PCIe Bus** | Il canale fisico di comunicazione ad alta velocità tra la RAM della CPU e la VRAM della GPU. |
| **VRAM** | Memoria dedicata della scheda video. Molto più veloce della RAM per gli accessi paralleli della GPU. |
| **Constant Buffer / Uniform Storage** | Area speciale della VRAM ottimizzata per dati che non cambiano durante una Draw Call, come le uniform. |
| **Row-Major** | Convenzione di memorizzazione matriciale in cui i dati sono disposti riga per riga (usata da C# / .NET). |
| **Column-Major** | Convenzione di memorizzazione matriciale in cui i dati sono disposti colonna per colonna (usata da OpenGL / GLSL). |
| **VAO** | *Vertex Array Object* — oggetto OpenGL che memorizza la configurazione degli attributi dei vertici (posizione, colore, UV...). |
| **Skeletal Animation** | Tecnica di animazione 3D in cui ogni parte del modello è controllata da una "osso" con la propria matrice di trasformazione. |
| **Instanced Rendering** | Tecnica per disegnare molte copie dello stesso oggetto in una singola Draw Call, ognuna con una propria matrice. |
| **DMA** | *Direct Memory Access* — meccanismo che permette al driver di trasferire dati in VRAM senza impegnare la CPU per ogni byte. |

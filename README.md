# Rain Filter (Noise Filter)
**Autor:** David Rodríguez

Usar el ruido direccionado como fuerza disipadora inspirada en la física del mundo real.

## Introducción
Hay un restaurante frente al mar, en una isla donde la costa no es propiamente una playa sino una suerte de malecón o paseo marítimo. Allí, algo profundamente contraintuitivo se mostró frente a mis ojos: en días de lluvia, lejos de tornarse más agitado, el mar se volvía notablemente más calmo. La expectativa lógica —que la lluvia generaría más perturbación en la superficie— se desmoronó ante la evidencia: las gotas parecían ejercer un efecto disipador.

Aún más curioso: las **crestas finas**, las pequeñas olas rápidas y puntiagudas, desaparecían primero. Lo que quedaba era una gran masa ondulante, más lenta y pesada. La lluvia actuaba como un filtro de paso bajo natural.

¿Puede este efecto trasladarse al audio digital o al procesamiento de geometrías?

---

## Analogía con Audio Digital
* **La Marea:** Una señal de audio cualquiera.
* **Crestas:** Los armónicos, las frecuencias por encima de la fundamental.
* **La Lluvia:** Ruido blanco, actuando como agente disipador.

El detalle crucial es que la lluvia no actúa de forma instantánea, sino mediante una miríada de impactos distribuidos en tiempo y espacio.

---

## Fundamento Matemático: Tensión Superficial Digital
Defino la **tensión superficial digital** como la resistencia local que oponen los datos (samples de audio o vértices de malla) a ser modificados por la gota. La resistencia se estima con derivadas discretas:

| Orden             | Fórmula (1‑D)                         | Interpretación física                       |
| :---------------- | :------------------------------------ | :------------------------------------------ |
| 1.ª derivada (Pendiente) | $\tau_1(n) = x(n) − x(n−1)$         | Inclinación instantánea.                    |
| 2.ª derivada (Curvatura) | $\tau_2(n) = x(n+1) − 2x(n) + x(n−1)$ | Detección de puntos de inflexión (picos y valles). |
| 3.ª y 4.ª derivadas | $\tau_k = \Delta^k x$                 | Detectan transitorios finos.                |

Cuando la **magnitud de curvatura es alta** en **crestas agudas**, la **resistencia de la superficie es baja** y cede con facilidad, permitiendo un impacto significativo. En contraste, cuando la **magnitud de curvatura es alta** en **valles profundos**, la **resistencia es extremadamente alta**, haciendo que estas zonas sean **casi inamovibles** por el impacto de la gota. Las zonas planas o suaves (con **baja magnitud de curvatura**) ofrecen una resistencia alta, resultando en un impacto mínimo.

### Mezcla de derivadas
Se pueden combinar órdenes para resaltar distintos rasgos, por ejemplo
$\tau = |\tau_1| \cdot |\tau_2|$ que enfatiza pendientes fuertes y curvaturas extremas.
El coeficiente de resistencia $R = f(\tau)$ se traspasa al bloque de fuerza. Cuanto mayor sea la **magnitud de $\tau$**, menor será $R$ y más violento el impacto.

---

## Ruido en Antifase
En la propuesta del Rain Filter, las gotas de lluvia aplican una fuerza disipadora cuya **dirección se opone a la polaridad actual de la señal**, empujándola siempre hacia el eje central (cero) o un estado de menor amplitud. Esto introduce una asimetría estructural en cómo el filtro interactúa con las variaciones rápidas.

La **intensidad de este impacto** está fuertemente modulada por la **curvatura local** de la superficie, actuando como un coeficiente de resistencia variable, independientemente de la polaridad de la señal:

* **Puntos de Extremo (picos o valles pronunciados, es decir, alta magnitud de curvatura):** La resistencia es **baja**, permitiendo que la gota ejerza una fuerza considerable, **aplastando eficazmente** estas formaciones abruptas.
* **Zonas Planas o Suaves (baja magnitud de curvatura):** La resistencia es **alta**, haciendo que el efecto de la gota sea **mínimo**, preservando estas áreas.

Por tanto, el ruido no actúa como una perturbación neutra o aleatoria pura, sino como una fuerza disipadora direccionada. El uso del ruido en antifase no es decorativo ni arbitrario:
* Es una condición necesaria para imitar el comportamiento físico de una gota actuando sobre una superficie.
* Convierte al Rain Filter en un modelo asimétrico y dependiente de la forma: la energía no se disemina por igual en todas las direcciones, sino que sigue una lógica de impacto orientado, inspirada en el mundo real.

---

## Ruido Direccionado según el Signo de la Señal
En una señal bipolar, la dirección del golpe depende únicamente de la fase (signo) de la muestra, nunca de su curvatura.

$\text{dir}(n) = -\text{sign}(x(n))$
\# fase positiva $\rightarrow$ fuerza descendente (–)
\# fase negativa $\rightarrow$ fuerza ascendente (+)

La magnitud de curvatura solo regula la intensidad de la gota:
$\text{coef}(n) = g(\tau(n))$ \# $0 \le g \le 1$ (resistencia local)
$\text{gota}(n) = \text{dir}(n) \cdot \text{coef}(n) \cdot |\eta(n)| \cdot \text{intensidad}$

Cuando la **magnitud de curvatura es grande** (picos afilados, ya sea positivos o negativos) $\text{coef}(n)$ se acerca a 1 $\rightarrow$ la gota impacta con toda su fuerza.

En **zonas planas** $\text{coef}(n)$ se reduce $\rightarrow$ el impacto es leve.

Así, fase determina la dirección del empuje y curvatura su potencia; ambos factores permanecen independientes.
Si la señal es unipolar, basta con fijar $\text{dir}(n) = -1$, lo que equivale al caso clásico de “lluvia desde arriba”.

---

## El Algoritmo

**Duplicar la Marea**
* Divide la señal en rama seca y rama procesada.

**Calcular la Curvatura ($\tau$)**
* Evalúa la derivada discreta (o mezcla de ellas).

**Generar Lluvia Direccionada**
* Calcula primero la dirección del golpe: $\text{dir}(n) = -\text{sign}(x(n))$.
* Genera ruido blanco $\eta(n)$ y toma su magnitud $|\eta(n)|$.
* Obtén la resistencia $\text{coef}(n) = g(\tau(n))$ y construye la gota:
    $\text{gota}(n) = \text{dir}(n) \cdot \text{coef}(n) \cdot |\eta(n)| \cdot \text{intensidad}$.

**Interpolación Log‑Lin**
* Para suavizar la curva de fuerza se usa:
    $\text{impacto} = \text{dir}(n) \cdot [(1-\text{mix}) \cdot \text{log}(1+\text{k} \cdot \text{coef}) + \text{mix} \cdot \text{coef}]$
    donde $\text{mix}$ (0 = log, 1 = lineal) y $\text{k}$ ajustan la escala.

* Este $\text{impacto}$ representa la fuerza disipadora de la gota. Gracias a la propiedad del logaritmo (y la combinación lineal), la **resistencia de la superficie a la deformación se modela directamente**: a medida que la magnitud de curvatura (`|τ|`) se reduce (tendiendo a zonas planas o valles con baja curvatura), la $\text{impacto}$ disminuye drásticamente, haciendo que el impacto sea mínimo en estas zonas. En **valles profundos**, la lógica del `coef(n)` (que se vuelve muy pequeño para alta curvatura en valles) y la dirección `dir(n)` (que empuja hacia arriba) resultan en un movimiento casi nulo, haciéndolos **casi inamovibles**.

**Acción por Zonas (radio R)**
* Reparte el impacto sobre 2R+1 muestras contiguas.

**Consistencia de Pitch**
* Ajusta R y la densidad de gotas según la frecuencia fundamental.

**Retroalimentar y Repetir**
* Envía un porcentaje de la señal filtrada al paso 2; itera.

**Disipación Secundaria (slew limiter)**
* Limita la velocidad máxima de cambio para emular relajación física , la tendencia de los fluidos a ir al estado de minima energia.

---

## Parámetros Principales

| Parámetro           | Descripción                         | Efecto auditivo                   |
| :------------------ | :---------------------------------- | :-------------------------------- |
| Intensidad de lluvia | Ganancia global del ruido           | Fuerza de cada gota               |
| Densidad de gotas   | Frecuencia / probabilidad de impacto | Lluvia torrencial vs. llovizna    |
| Feedback            | Porcentaje reenviado al ciclo       | Duración del efecto               |
| Mix log‑lin         | Curva de fuerza aplicada (0 = log, 1 = lineal) | Agresividad / distorsión          |
| Radio de impacto (R) | Nº de muestras afectadas por cada gota | Textura y dispersión              |

---

## Resultado
El resultado es un efecto emergente donde la señal se va suavizando de manera orgánica. Las partes tranquilas permanecen casi intactas, mientras que las agitadas son golpeadas una y otra vez por gotas aleatorias hasta calmarse. Se trata de una forma de procesamiento no lineal, sensible al contenido dinámico de la señal.
Aunque el efecto final puede recordar a un filtro de paso bajo, lo más valioso de la propuesta no es solo el espectro limpio que se obtiene al final, sino los pasos intermedios: las formas transitorias, los patrones momentáneos, la textura en movimiento. Estas microvariaciones aportan riqueza sonora y visual, haciendo del Rain Filter no solo una técnica de limpieza, sino también de expresión.

---

## Aplicaciones fuera del campo del DSP
El algoritmo del Rain Filter ha sido implementado de forma temprana con éxito en el suavizado de mallas poligonales, proponiendo un enfoque de suavizado en el que el ruido aleatorio actúa como filtro emergente. A diferencia de métodos deterministas, las gotas actúan de forma inversa a la fase del vector normal: las crestas se erosionan y los valles **se rellenan**, reflejando la acción de una fuerza que los empuja hacia un punto de equilibrio. El resultado final es un suavizado global, pero lo interesante del método reside en los pasos intermedios y su aleatoriedad controlada.

[PureRainSmooth](https://github.com/infamedavid/PureRainSmooth)

---

## Conclusión

Inspirarse en fenómenos naturales como la lluvia y el mar no solo es poético: también puede ser profundamente útil como base para procesos algorítmicos. La propuesta del Rain Filter ofrece una alternativa original a los delays filtrados clásicos, incorporando aleatoriedad, adaptabilidad, y un vínculo con el comportamiento físico del mundo. Es importante destacar que, aunque ya se han explorado estas ideas en implementaciones prácticas como Rain Smooth sobre mallas, su aplicación en procesamiento digital de señales (DSP) sigue en el terreno especulativo.

---

# Rain Filter (Noise Filter)

**Autor:** David Rodríguez

Usar el ruido antifase para modelar una fuerza disipadora inspirada en la física del mundo real.

---

## Introducción

Hay un restaurante frente al mar, en una isla donde la costa no es propiamente una playa sino una suerte de malecón o paseo marítimo. Allí, algo profundamente contraintuitivo se mostró frente a mis ojos: en días de lluvia, lejos de tornarse más agitado, el mar se volvía notablemente más calmo. La expectativa lógica —que la lluvia generaría más perturbación en la superficie— se desmoronó ante la evidencia: las gotas de agua parecían tener un efecto disipador sobre las olas.

Lo más curioso era que no se trataba simplemente de una atenuación uniforme. Los picos más afilados, las pequeñas olas rápidas y puntiagudas, desaparecían primero. Lo que quedaba era una gran masa ondulante, más pesada, como si solo las olas lentas y profundas pudieran resistir el efecto de la lluvia. La lluvia actuaba como una suerte de **filtro de paso bajo natural**.

¿Puede este efecto transpolarse al audio digital o al procesamiento de geometrías?

---

## Analogía con Audio Digital

* **La Marea:** Una señal de audio arbitraria.
* **Crestas:** Los armónicos, son las frecuencias que van sobre la fundamental.
* **La Lluvia:** Es ruido blanco, actuando como un modulador.

Un elemento importante a considerar es que la lluvia no ejerce un efecto instantáneo en la marea sino que es la suma de muchísimas interacciones en largas distancias y tiempo.

---

## Fundamento Matemático: Tensión Superficial

En el contexto del **Rain Filter**, el significado de **tensión superficial digital**: es la resistencia que ofrece cada punto de la señal (o de la malla en el Rain smooth) frente a la energía de la gota.

* **Alta tensión superficial** → la superficie es difícil de deformar; la gota apenas actúa.
* **Baja tensión superficial** → la cresta presenta poca resistencia; la gota tiene gran efecto.

Matemáticamente se estima esa resistencia con **derivadas discretas**; cuanto mayor sea la curvatura absoluta, menor la tensión superficial y, por tanto, mayor la vulnerabilidad.

### Medidas discretas

| Orden                     | Fórmula (1‑D)                 | Interpretación física            |
| :------------------------ | :---------------------------- | :------------------------------- |
| 1.ª derivada (Pendiente)  | $\tau_1(n) = x(n) − x(n−1)$   | Inclinación instantánea.         |
| 2.ª derivada (Laplaciano) | $\tau_2(n) = x(n+1) − 2x(n) + x(n−1)$ | Curvatura: picos y valles.       |
| 3.ª y 4.ª derivadas       | $\tau_k = \Delta^k x$         | Detectan transitorios complejos. |

### Producto o mezcla de derivadas

Es posible multiplicar o combinar estos términos; por ejemplo $\tau = |\tau_1| \cdot |\tau_2|$ acentúa zonas donde la pendiente es grande y la curvatura es extrema. El valor final $\tau$ se pasa al bloque de fuerza (interpolación log‑lin).

De esta forma, la tensión superficial digital se convierte en un coeficiente de resistencia directamente proporcional a la curvatura local: una cima afilada tiene baja resistencia y es fácilmente “aplastada” por la gota; una llanura ofrece resistencia máxima y apenas se ve afectada.

Por otro lado un empluje lineal ejercera un empuje  constante , mientra que uno logaritmico  imitaria la resistencia  creciente de la superfice a ser desplazado

---

## Ruido en Antifase

En la propuesta del **Rain Filter**, la señal base —la "marea"— puede entenderse como una señal unipolar (por ejemplo, elevación respecto a un eje base). Las "gotas de lluvia", en cambio, siempre aplican fuerza desde una dirección definida: desde arriba, como en el mundo físico. Esto introduce una asimetría estructural en el comportamiento del filtro.

El impacto de la gota debe estar siempre en dirección opuesta a la curvatura local: si se encuentra una cresta, se aplasta desde arriba; si se encuentra un valle, se empuja desde abajo. Pero como la dirección del ruido es fija, esto implica que debe invertirse su fase cuando la señal base cambia de polaridad.

Por tanto, el ruido no actúa como una perturbación neutra o aleatoria pura, sino como una **fuerza disipadora direccionada**. El uso del ruido en antifase no es decorativo ni arbitrario:

* Es una condición necesaria para imitar el comportamiento físico de una gota actuando sobre una superficie.
* Esta propiedad convierte al Rain Filter en un modelo asimétrico y dependiente de la forma: la energía no se disemina por igual en todas las direcciones, sino que sigue una lógica de impacto orientado, inspirada en el mundo real.

---

## El Algoritmo

El Rain Filter se implementa como un **delay difuso con retroalimentación**, ahora ampliado para actuar por zonas y mantener consistencia entre distintos tonos.

### Paso 1 – Duplicar la Marea

División de la señal en una rama seca y una rama procesada.

### Paso 2 – Calcular la Tensión ($\tau$)

Evaluación de la curvatura local con derivadas discretas (1.ª, 2.ª, 3.ª, 4.ª) y, opcionalmente, combinaciones con distintos pesos.

### Paso 3 – Generar Lluvia

Ruido blanco continuo; solo se acepta el ruido en antifase con la señal.

Además de su forma (blanco, antifase), la frecuencia de aparición de las gotas determina la tasa de acción del filtro: una lluvia densa (alta frecuencia) moldea la señal más rápido, mientras que una lluvia escasa (frecuencia baja) actúa con mayor dispersión y puede requerir más repeticiones para lograr el mismo efecto.

### Paso 4 – Interpolación Log‑Lin

La fuerza aplicada se calcula con:
`impacto = (1 - mix) * log(1 + k * |τ|) + mix * |τ|`
`impacto *= sign(−τ)` # cresta ↓ / valle ↑

`mix` controla la mezcla (0 = log puro, 1 = lineal puro) y `k` fija la escala del log. Así, la resistencia crece a medida que la curvatura se reduce.

### Paso 5 – Acción por Zonas (radio R)

En lugar de afectar un solo sample, el impacto se reparte sobre un vecindario de `2R+1` muestras. Esto acelera la convergencia y crea una textura más densa.

### Paso 6 – Consistencia de Pitch

Para que el carácter del filtro sea igual en distintos tonos:
* **Opción A · Detección de pitch:** se calcula la longitud de onda $\lambda$ y se ajusta el radio `R` $\propto \lambda$ y la densidad de gotas.
* **Opción B · Señal CV 1 V/oct:** en entornos modulares, un control 1 V/oct modula `R` y la frecuencia de lluvia directamente.

### Paso 7 – Retroalimentar y Repetir

Una fracción (`feedback`) de la señal procesada vuelve a la etapa de tensión y el ciclo se repite hasta revelar la fundamental.

Esta iteración gradual no solo es una necesidad técnica: es parte esencial del efecto buscado. A diferencia de un filtro pasabajo convencional, aquí el interés no reside únicamente en el resultado final (una forma suavizada), sino en la **evolución intermedia: las pequeñas deformaciones, las microacciones, la textura viva del proceso**.

### Paso 8 – Disipación secundaria

Para simular el comportamiento físico de los fluidos —su tendencia a buscar un estado de mínima energía—, se añade una etapa opcional de suavizado correctivo posterior al filtro.
Se implementa como un **slew limiter digital**, que impone una velocidad máxima de cambio entre muestras consecutivas. Esta etapa no aplica nueva fuerza, sino que reduce gradualmente los excesos residuales, como haría un sistema físico al estabilizarse naturalmente.
El resultado es más natural y libre de artefactos, especialmente útil para rematar la forma final o limpiar distorsiones inesperadas.

---

### Parámetros del Rain Filter

| Parámetro                | Descripción                                         | Efecto auditivo                         |
| :----------------------- | :-------------------------------------------------- | :-------------------------------------- |
| **Intensidad de lluvia** | Multiplicador global aplicado a la señal de lluvia  | Más o menos fuerza por gota             |
| **Densidad de gotas** | Frecuencia o probabilidad con que ocurre una gota   | Lluvia torrencial vs. llovizna          |
| **Feedback** | Porcentaje de la señal procesada que vuelve al circuito | Duración del efecto                     |
| **Caída de amplitud** | Atenuación adicional en la ruta de feedback         | Controla desvanecimiento                |
| **Interpolación log-lin**| Valor entre 0 y 1 que define la curva de fuerza aplicada | Controla agresividad/distorsión         |
| **Zona de impacto (samples)** | Cantidad de muestras impactadas por una sola gota | Controla textura y dispersión           |

### Nota sobre el número de samples impactados

Aunque en esta versión del algoritmo se contempla la posibilidad de que una gota afecte a un grupo de samples adyacentes (una zona de impacto), esto no es un requerimiento fundamental. La acción sobre un solo sample sigue siendo válida y funcional, solo que requerirá mayor densidad de lluvia (más ruido y más interacciones sucesivas) para obtener un resultado perceptualmente equivalente.

Por tanto, el número de samples impactados por cada gota se considera un parámetro estético, no estructural: influye en la velocidad de convergencia del filtro, su comportamiento espacial, y su textura sonora, pero no en su principio fundamental.

---

## Resultado

El resultado es un efecto emergente donde la señal se va suavizando de manera orgánica. Las partes tranquilas permanecen casi intactas, mientras que las agitadas son golpeadas una y otra vez por gotas aleatorias hasta calmarse. Se trata de una forma de **procesamiento no lineal**, sensible al contenido dinámico de la señal.

Aunque el efecto final puede recordar a un filtro de paso bajo, lo más valioso de la propuesta no es solo el espectro limpio que se obtiene al final, sino los pasos intermedios: las formas transitorias, los patrones momentáneos, la **textura en movimiento**. Estas microvariaciones aportan riqueza sonora y visual, haciendo del Rain Filter no solo una técnica de limpieza, sino también de expresión.

---

## Aplicaciones fuera del campo del DSP

El algoritmo del Rain Filter ha sido implementado de forma temprana con éxito en el suavizado de **mallas poligonales**, proponiendo un enfoque de suavizado en el que el ruido aleatorio actúa como filtro emergente. A diferencia de métodos deterministas, las gotas actúan de forma inversa a la fase del vector normal: las crestas se erosionan y los valles se rellenan. El resultado final es un suavizado global, pero lo interesante del método reside en los pasos intermedios y su aleatoriedad controlada.

https://github.com/infamedavid/PureRainSmooth

---

## Conclusión

Inspirarse en fenómenos naturales como la lluvia y el mar no solo es poético: también puede ser profundamente útil como base para procesos algorítmicos. La propuesta del Rain Filter ofrece una alternativa original a los delays filtrados clásicos, incorporando aleatoriedad, adaptabilidad, y un vínculo con el comportamiento físico del mundo.

Es importante destacar que, aunque ya se han explorado estas ideas en implementaciones prácticas como Rain Smooth sobre mallas, su aplicación en procesamiento digital de señales (DSP) sigue en el terreno especulativo. mis limitaciones técnicas actuales impiden una implementación completa en tiempo real o con motores DSP optimizados, aunque los principios están claros y ofrecen una base sólida para futuros desarrollos.

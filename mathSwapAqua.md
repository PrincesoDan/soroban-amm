# Protocolo Aqua - Matemáticas del StableSwap

El protocolo Aquarius implementa un algoritmo StableSwap similar al de Curve Finance para su pool de liquidez estable. Este documento explica en detalle los modelos matemáticos fundamentales que rigen el funcionamiento del contrato [CBQDHNBFBZYE4MKPWBSJOPIYLW4SFSXAXUTSXJN76GNKYVYPCKWC6QUK](https://stellar.expert/explorer/public/contract/CBQDHNBFBZYE4MKPWBSJOPIYLW4SFSXAXUTSXJN76GNKYVYPCKWC6QUK).

## La Ecuación Invariante del StableSwap

La esencia del modelo StableSwap es su ecuación invariante. A diferencia del modelo de producto constante de Uniswap ($x \cdot y = k$), el StableSwap utiliza una función invariante más compleja:

$$A \cdot n^n \cdot \sum_{i=1}^{n} x_i + D = A \cdot D \cdot n^n + \frac{D^{n+1}}{n^n \cdot \prod_{i=1}^{n} x_i}$$

Donde:
- $x_i$ son los balances normalizados de cada token en el pool
- $n$ es el número de tokens en el pool
- $A$ es el coeficiente de amplificación (explicado más adelante)
- $D$ es el invariante que se mantiene constante durante los intercambios

Esta ecuación proporciona un comportamiento híbrido:
- Cuando $A \to 0$: Se aproxima a un AMM de suma constante ($\sum x_i = k$)
- Cuando $A \to \infty$: Se aproxima a un AMM de producto constante ($\prod x_i = k$)
- Para valores intermedios de $A$: Balance óptimo para tokens estables

## Funciones Matemáticas Principales

### 1. Swap: El Intercambio de Tokens

La función `swap` es la operación principal del pool que permite intercambiar un token por otro.

**Firma:**

fn swap(user, in_idx, out_idx, in_amount, out_min) → out_amount

**Modelo matemático:**

1. **Preparación de datos normalizados:**
   $$xp_i = \text{reserves}_i \cdot \text{precision\_mul}_i$$

2. **Cálculo del nuevo balance de entrada:**
   $$x' = xp_{in\_idx} + in\_amount \cdot \text{precision\_mul}_{in\_idx}$$

3. **Determinación del nuevo balance de salida:**
   $$y = \text{\_get\_y}(in\_idx, out\_idx, x', xp)$$

4. **Cálculo de la cantidad bruta a recibir:**
   $$dy_{raw} = xp_{out\_idx} - y - 1$$

5. **Aplicación de la comisión:**
   $$dy_{fee} = dy_{raw} \cdot \frac{fee}{FEE\_DENOMINATOR}$$

6. **Cantidad final a recibir:**
   $$out\_amount = \frac{dy_{raw} - dy_{fee}}{\text{precision\_mul}_{out\_idx}}$$

7. **Verificación del mínimo:**
   Si $out\_amount < out\_min$, la transacción falla con error `OutMinNotSatisfied`

Esta operación mantiene el invariante $D$ constante después del intercambio, asegurando la estabilidad del pool.

### 2. _get_y: Cálculo del Balance de Salida

Esta función resuelve para encontrar el valor de $y$ (balance del token de salida) que mantiene el invariante del pool después de modificar el balance del token de entrada.

**Firma:**

_get_y:

fn get_y(in_idx, out_idx, x, xp) → y

**Modelo matemático:**

1. **Cálculo del invariante actual:**
   $$D = \text{\_get\_d}(xp, A)$$

2. **Preparación de coeficientes:**
   $$c = \frac{D^{n+1}}{(A \cdot n^n) \cdot \prod_{i \neq out\_idx} x_i}$$
   
   $$s = \sum_{i \neq out\_idx} x_i$$
   
   $$b = s + \frac{D}{A \cdot n}$$

3. **Resolución iterativa (método de Newton):**
   $$y_{i+1} = \frac{y_i^2 + c}{2y_i + b - D}$$

4. **Criterio de convergencia:**
   La iteración se detiene cuando $|y_{i+1} - y_i| \leq 1$

Este algoritmo encuentra con precisión el nuevo balance del token de salida que mantiene el invariante $D$ constante.

### 3. a: El Coeficiente de Amplificación

El coeficiente $A$ determina la curvatura de la función StableSwap y puede ajustarse a lo largo del tiempo.

**Firma:**

fn a() → amp
**Modelo matemático:**

El valor de $A$ puede seguir una rampa lineal en el tiempo:

$$A(t) = 
\begin{cases}
A_0 + (A_1 - A_0) \cdot \frac{t - t_0}{t_1 - t_0}, & \text{si } A_1 > A_0 \text{ y } t_0 \leq t < t_1 \\
A_0 - (A_0 - A_1) \cdot \frac{t - t_0}{t_1 - t_0}, & \text{si } A_1 < A_0 \text{ y } t_0 \leq t < t_1 \\
A_1, & \text{si } t \geq t_1
\end{cases}$$

Donde:
- $A_0$ es el valor inicial (`initial_a`)
- $A_1$ es el valor objetivo (`future_a`)
- $t_0$ es el tiempo de inicio de la rampa (`initial_a_time`)
- $t_1$ es el tiempo de finalización (`future_a_time`)
- $t$ es el tiempo actual (`now`)

Este mecanismo de rampa permite ajustar gradualmente la curvatura del pool sin causar discontinuidades en los precios.

### 4. _get_d: Cálculo del Invariante

Esta función calcula el valor del invariante $D$ para un conjunto dado de balances y un coeficiente de amplificación.

**Firma:**

fn _get_d(xp, amp) → D


**Modelo matemático:**

1. **Cálculo de la suma de balances:**
   $$S = \sum_{i=1}^{n} x_i$$

2. **Inicialización:**
   Si $S = 0$, entonces $D = 0$ y terminamos
   De lo contrario, $D_{0} = S$

3. **Proceso iterativo:**
   Para cada iteración $j$:
   
   $$D_{p,j} = D_j \cdot \prod_{i=1}^{n} \frac{D_j}{n \cdot x_i}$$
   
   $$ann = A \cdot n^n$$
   
   $$D_{j+1} = \frac{(ann \cdot S + D_{p,j} \cdot n) \cdot D_j}{(ann - 1) \cdot D_j + (n + 1) \cdot D_{p,j}}$$

4. **Criterio de convergencia:**
   La iteración se detiene cuando $|D_{j+1} - D_j| \leq 1$

Este algoritmo converge rápidamente (típicamente en menos de 10 iteraciones) al valor correcto de $D$.

## Implicaciones Económicas

El modelo StableSwap proporciona varias ventajas clave:

1. **Slippage reducido:** Para intercambios entre tokens estables (de valor similar), el deslizamiento es significativamente menor que en AMMs tradicionales.

2. **Flexibilidad adaptativa:** El parámetro $A$ permite ajustar dinámicamente la curva de intercambio según las condiciones del mercado.

3. **Zona de precio estable:** Cuando los tokens mantienen su paridad, el intercambio ocurre con mínima pérdida, similar a un intercambio 1:1.

4. **Resistencia a desviaciones:** Si un token pierde su paridad, la curva se ajusta automáticamente para desincentivizar arbitrajes excesivos.

La implementación de Aqua Protocol adopta este modelo sofisticado para proporcionar intercambios óptimos entre tokens estables en el ecosistema Stellar.
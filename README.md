[README (1).md](https://github.com/user-attachments/files/31956271/README.1.md)
# Practica I — De los pixeles a la integral

**Curso:** ST0244 — Programación de Lenguajes (Programming Paradigms)
**Profesor:** Alexander Narváez Berrío
**Universidad EAFIT — Escuela de Ciencias Aplicadas e Ingeniería**

## Integrantes del equipo

- Sebastián Quintero
- Juan José Cadavid

## Entorno de desarrollo

| Herramienta | Versión usada en el desarrollo |
|---|---|
| Sistema operativo | Windows 11 |
| GHC (Haskell) | instalado con [GHCup](https://www.haskell.org/ghcup/) |
| SWI-Prolog | instalado desde [swi-prolog.org/download/stable](https://www.swi-prolog.org/download/stable) |
| Terminal | PowerShell |

Ambos programas solo usan librerías estándar (`bytestring` viene con GHC; ninguna librería extra de Prolog además de las built-in de SWI-Prolog). No se necesita instalar nada más.

## Estructura del repositorio

```
.
├── README.md
├── curva_binaria_P4.pbm
├── Haskell/
│   └── Main.hs
└── Prolog/
    └── area.pl
```

## Cómo ejecutar la solución en Haskell (Windows / PowerShell)

1. Instalar GHC con [GHCup](https://www.haskell.org/ghcup/) (instalador para Windows), aceptando instalar GHC durante el proceso. Cerrar y volver a abrir PowerShell después de instalar.
2. Verificar la instalación:

   ```powershell
   ghc --version
   ```

3. Desde la carpeta `Haskell/` (el programa busca el `.pbm` un nivel arriba por defecto):

   ```powershell
   cd Haskell
   ghc -O2 -o area.exe Main.hs
   .\area.exe
   ```

También se puede indicar explícitamente la ruta del archivo:

```powershell
.\area.exe ..\curva_binaria_P4.pbm
```

O ejecutarlo sin compilar, con el intérprete:

```powershell
runghc Main.hs
```

## Cómo ejecutar la solución en Prolog (Windows / PowerShell)

1. Instalar [SWI-Prolog](https://www.swi-prolog.org/download/stable) (instalador para Windows). Cerrar y volver a abrir PowerShell después de instalar.
2. Verificar la instalación:

   ```powershell
   swipl --version
   ```

3. Desde la carpeta `Prolog/` (también busca el `.pbm` un nivel arriba por defecto):

   ```powershell
   cd Prolog
   swipl area.pl
   ```

O indicando la ruta explícitamente:

```powershell
swipl area.pl ..\curva_binaria_P4.pbm
```

`area.pl` tiene una directiva `:- initialization(main).`, así que se ejecuta automáticamente al cargar el archivo y termina con `halt`.

> **Nota sobre los caracteres del sparkline (▁▂▃▄▅▆▇█):** en PowerShell moderno se ven bien por defecto. Si aparecen como cuadros o signos de interrogación, ejecutar antes `chcp 65001` para forzar codificación UTF-8 en la terminal.

## Área obtenida

Con el archivo `curva_binaria_P4.pbm` suministrado (567 × 319 píxeles):

```
Área = 108660 píxeles cuadrados
```

Este valor lo calculan **ambos programas, con dos paradigmas distintos, y coinciden exactamente**. Es también el mismo resultado que produce el programa de referencia en C++ mostrado en el enunciado (Figura 1), lo que confirma que la interpretación de la imagen (bit `1` = negro = región bajo la curva, conteo de negros consecutivos desde abajo) es correcta.

Como verificación adicional, los diez valores de muestra `x_i -> f(x_i)` obtenidos por ambos programas son idénticos entre sí **y** idénticos a los que aparecen en la Figura 1 del enunciado (x₀=0→224, x₁=62→239, ..., x₉=566→145).

## Estrategia para mostrar la imagen en la consola

La imagen original (567×319 píxeles) es mucho más ancha que una terminal típica (80–120 columnas) y mucho más alta que una pantalla de texto razonable, así que no se imprime píxel a píxel. La estrategia, **igual en Haskell y en Prolog** para que ambas visualizaciones sean comparables, es:

1. **Reducción horizontal (muestreo por bloques/"bucketing"):** las `ancho` columnas originales (567) se agrupan en `AnchoTerminal` (100) grupos contiguos de tamaño aproximadamente `ancho / AnchoTerminal`. Cada grupo se representa con el **promedio** de las alturas `f(x)` que caen en él. Promediar (en vez de tomar solo la primera o la máxima) evita que un único píxel ruidoso distorsione la forma general de la curva.
2. **Reducción vertical (escalado lineal):** se toma la altura máxima entre esos promedios y se usa como referencia para reescalar linealmente los valores a `AltoTerminal` (25) filas de texto.
3. **Dibujo de abajo hacia arriba:** para cada fila `r` (de arriba hacia abajo) se calcula un umbral `AltoTerminal - r`; una columna se "pinta" (`#`) en esa fila si su altura escalada alcanza el umbral. Esto reproduce el efecto de barras verticales acumuladas, igual que en la Figura 1 del enunciado.

Además, como una visualización complementaria (requisito 7, la función de alturas `M[x] = f(x)`), ambos programas imprimen una segunda representación de una sola línea usando los caracteres Unicode de bloque (`▁▂▃▄▅▆▇█`, 8 niveles). Aquí cada carácter corresponde directamente a una muestra de `M`, sin dibujar toda la grilla 2D; es una especie de "sparkline" que hace muy visible dónde sube y baja la curva de un vistazo.

Ambas técnicas de reducción (agrupar columnas por promedio, escalar filas linealmente al máximo) están implementadas dos veces de forma independiente —como `muestrear`/`dibujarCurva`/`sparkline` en Haskell y como `muestrear`/`visualizar_curva`/`visualizar_alturas` en Prolog— siguiendo el mismo criterio matemático pero expresado con las herramientas propias de cada paradigma (`map`/listas por comprensión en Haskell; `findall`/recursión en Prolog).

## Explicación de las dos soluciones

### Parte I — Haskell (funcional)

La transformación completa se ve como una tubería de funciones puras:

```
PBM (bytes) -> pixel(x,y) -> f(x) -> M -> area
```

- `cargarPBM` lee el archivo y separa el encabezado de los bytes de píxeles (`ByteString`).
- `esNegro img x y` es una función pura que, dado un `PBM` y una coordenada, dice si ese píxel es negro, manipulando bits (`testBit`) sobre el byte correspondiente.
- `f img x` es la función discreta pedida en el enunciado: cuenta píxeles negros consecutivos desde abajo con `takeWhile`.
- El vector de alturas es, literalmente, **la aplicación de una función a todo el dominio**:

  ```haskell
  vectorAlturas img = map (f img) [0 .. ancho img - 1]
  ```

- El área es un **plegado (fold)** sobre esa estructura:

  ```haskell
  area = sum
  ```

No hay un bucle imperativo "recorriendo con un índice y acumulando en una variable mutable": el programa describe **qué transformación aplicar** (`map f`) y **cómo combinar el resultado** (`sum`), y GHC se encarga de la ejecución. La recursión que sí aparece (por ejemplo en `esNegro`/`f` a través de listas por comprensión y `takeWhile`) opera sobre estructuras inmutables, no sobre estado mutable.

### Parte II — Prolog (lógico/declarativo)

En vez de programar el "cómo", se declaran las relaciones que deben cumplirse:

```
f(X, Datos, Alto, BytesPorFila, Altura)
```

es una relación entre una posición `X` y su altura `Altura`, no una función con un cuerpo secuencial. Prolog la resuelve por unificación y backtracking sobre los hechos y reglas de conteo de píxeles negros.

El vector de alturas nunca se "llena" con un bucle: se **pregunta** por todos los valores que satisfacen la relación, para cada `X` entre `0` y `MaxX`:

```prolog
findall(Altura,
        ( between(0, MaxX, X),
          f(X, Datos, Alto, BytesPorFila, Altura) ),
        M)
```

y el área es, de nuevo, una relación entre una lista y su suma:

```prolog
sum_list(M, Area)
```

Aquí el énfasis está en **qué relación debe existir** entre la imagen, una posición y su altura (y entre la lista de alturas y el área), no en la secuencia de pasos para calcularla — aunque, por supuesto, SWI-Prolog internamente sí ejecuta una búsqueda determinista para resolver esas relaciones.

### Comparación de paradigmas

| | Haskell (funcional) | Prolog (lógico) |
|---|---|---|
| Visión del problema | dominio → alturas → área (transformaciones) | relaciones → valores que las satisfacen → área |
| Construcción de M | `map f [0..n-1]` (aplicar función) | `findall/3` (recolectar soluciones de una relación) |
| Cálculo del área | `sum` (fold/plegado) | `sum_list/2` (mismo fold, expresado como relación built-in) |
| "Bucle" de conteo de píxeles negros | recursión sobre listas inmutables (`takeWhile`) | recursión sobre una relación (`contar_negros_desde/5`) con unificación |
| Qué se declara | *qué función aplicar y cómo combinar resultados* | *qué debe cumplirse entre entrada y salida* |

Ambos programas, a pesar de partir de paradigmas muy distintos, calculan exactamente **108660 píxeles cuadrados** como área bajo la curva, confirmando que la Suma de Riemann

```
A = Σ_{x=0}^{n-1} f(x) · Δx     (Δx = 1 píxel)
```

es la misma expresión matemática detrás de las dos implementaciones.

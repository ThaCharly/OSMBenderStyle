# DOCUMENTO DE ARQUITECTURA — FOOTBALL MATCH ENGINE (v4)

## Filosofía Fundamental

El motor simula posesiones de fútbol a través de una sola barra compartida: `field_progress`.

0 = arco propio del equipo con el balón  
50 = mitad de cancha  
100 = arco rival  

Cada acción mueve esta barra hacia adelante o hacia atrás. El equipo rival presiona simultáneamente — su presión sube mientras dura la posesión. Si llega a 100, roban la pelota y se invierte todo.

El riesgo no es intrínseco a cada acción como una tirada de dados independiente. Es acumulativo: cada decisión cambia el estado actual del equipo con posesión (exposición) y del rival (presión). Cuando el umbral de pérdida se alcanza, la posesión termina.

---

# Estado Mínimo del Partido

```cpp
struct MatchContext
{
    Team home;
    Team away;

    bool home_possession;        // quién tiene el balón ahora

    int32_t field_progress;      // 0..100, medido desde el equipo con posesión actual
    
    uint8_t accumulated_exposure; // se acumula con cada acción del equipo con posesión
    uint8_t rival_pressure;       // sube con exposición + tipo de acción

    uint8_t home_score;
    uint8_t away_score;
};
```

No existen zonas. No existen tácticas. No existen jugadores individuales. Solo field_progress, exposición acumulada y presión rival acumulada.

---

# Acción Fundamental

El equipo con posesión elige una acción. Cada acción tiene:

- **Exposición generada**: cuánto riesgo acumula la posesión
- **Efecto sobre field_progress**: cuánto avanza o retrocede si se ejecuta
- **Efecto sobre rival_pressure**: cuánto sube la presión del rival
- **Acciones disponibles según posición actual** (ver más abajo)

```cpp
struct ActionDefinition
{
    int exposure;        // +exposición acumulada de la posesión
    int progress_delta;  // cambio en field_progress si se ejecuta
    int pressure_delta;  // cambio en rival_pressure
};
```

---

# Acciones Disponibles Según Posición

El campo es un recurso compartido. No todas las acciones están disponibles en todas las posiciones:

**0–25** (propia mitad, cerca del propio arco):
- GO_BACK
- KEEP_BALL
- PROGRESS
- *(no podés finalizar — estás lejos del arco rival)*

**25–60** (mediocampo y entrada a campo rival):
- GO_BACK
- KEEP_BALL
- PROGRESS
- DANGEROUS_ACTION
- FINISH *(posible, aunque arriesgado)*

**60–100** (campo rival, cerca del arco rival):
- TODAS las acciones anteriores
- FINISH es la acción natural aquí

---

# Definición de Acciones (IMPORTANTÍSIMO, TODAS LAS ACCIONES PUEDEN TENER VARIACIONES)

```cpp
GO_BACK:
  exposición: -5
  field_progress: -10
  rival_pressure: -10 o +10

KEEP_BALL:
  exposición: +5
  field_progress: +5
  rival_pressure: +10

PROGRESS:
  exposición: +15
  field_progress: +20
  rival_pressure: +12

DANGEROUS_ACTION:
  exposición: +35
  field_progress: +40
  rival_pressure: +25

FINISH:
  exposición: +50
  field_progress: (definición al arco)
  rival_pressure: +30
```

Estos valores son iniciales. Más adelante se ajustarán con estadísticas de jugadores y presión modelada. Por ahora son constantes fijas.

---

# Mecánica de Pérdida de Posesión

La pérdida nace del umbral, no de cada acción individualmente:

```cpp
bool IsPossessionLost(MatchContext& ctx)
{
    return (ctx.exposure + ctx.rival_pressure) >= LOSS_THRESHOLD;
}
```

Donde `LOSS_THRESHOLD` es una constante configurable (ej: 80).

Esto genera comportamientos emergentes:

- Una secuencia `GO_BACK, GO_BACK, KEEP_BALL, GO_BACK` reduce activamente el riesgo acumulado — es legítima
- Una secuencia larga de progresiones acumulando exposición inevitablemente termina en pérdida
- El rival presiona mientras dura la posesión; si no avanzás lo suficiente, te roban

---

# Bucle Interno del Motor

```cpp
void ProcessPossession(MatchContext& ctx)
{
    auto action = SelectAction(ctx.field_progress);  // elige según posición
    
    // Aplicar efectos de la acción
    ctx.accumulated_exposure += action.exposure;
    ctx.field_progress += action.progress_delta;
    ctx.rival_pressure += action.pressure_delta;
    
    // Verificar si la posesión se perdió por acumulación
    if (IsPossessionLost(ctx)) {
        EndPossession();  // robo por presión/exposición acumulada
        return;
    }
    
    // Verificar gol en FINISH
    if (action == FINISH && Roll() < action.goal_chance) {
        ScoreGoal();
        EndPossession();
        return;
    }
    
    // Verificar si llegó al arco rival sin definir
    if (ctx.field_progress >= 100) {
        ctx.field_progress = 95;  // mantener posesión en posición de definición
    }
}
```

---

# Objetivo de la Primera Iteración

Generar un partido con:

1. Posesiones identificables (secuencias de acciones entre cambios de posesión)
2. Recuperaciones por pérdida directa y por presión rival acumulada
3. Progresiones naturales hacia el arco rival
4. Finalizaciones al arco
5. Goles distribuidos entre ambos equipos
6. Duración de posesiones entre 3 y 12 acciones en promedio

---

# Fase Posterior

Una vez validado el sistema base:

- Las estadísticas de jugadores modificarán exposición y efectos de acción
- La presión modelada reemplazará los valores fijos
- Las tácticas no modificarán probabilidades — modificarán distribuciones de decisión (qué acciones se prefieren en cada posición)
- El campo sigue siendo una sola barra compartida

Las tácticas son personalidad, no bonos.
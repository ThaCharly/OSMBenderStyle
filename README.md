# DOCUMENTO DE ARQUITECTURA — OSMBenderStyle

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

    uint8_t match_time;           // minutos simulados (0..90)
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
- KEEP_BALL
- PROGRESS
- GO_BACK

**25–60** (mediocampo y entrada a campo rival):
- KEEP_BALL
- PROGRESS
- DANGEROUS_ACTION
- FINISH *(posible, aunque arriesgado)*
- GO_BACK

**60–100** (campo rival, cerca del arco rival):
- TODAS las acciones anteriores
- FINISH es la acción natural aquí

---

# Definición de Acciones

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
  goal_chance: 0.15 (15%)
  rival_pressure: +30
```

Estos valores son iniciales. Más adelante se ajustarán con estadísticas de jugadores y presión modelada. Por ahora son constantes fijas.

---

# Mecánica de Pérdida de Posesión

La pérdida nace del umbral, no de cada acción individualmente:

```cpp
bool IsPossessionLost(MatchContext& ctx)
{
    return (ctx.accumulated_exposure + ctx.rival_pressure) >= LOSS_THRESHOLD;
}
```

Donde `LOSS_THRESHOLD = 80`.

Esto genera comportamientos emergentes:

- Una secuencia `GO_BACK, GO_BACK, KEEP_BALL, GO_BACK` reduce activamente el riesgo acumulado — es legítima
- Una secuencia larga de progresiones acumulando exposición inevitablemente termina en pérdida
- El rival presiona mientras dura la posesión; si no avanzás lo suficiente, te roban

---

# Bucle Interno del Motor

```cpp
void ProcessPossession(MatchContext& ctx)
{
    while (true) {
        // Elegir acción según posición actual
        auto action = SelectAction(ctx.field_progress);
        
        // Aplicar efectos de la acción
        ctx.accumulated_exposure += action.exposure;
        ctx.field_progress += action.progress_delta;
        ctx.rival_pressure += action.pressure_delta;
        
        // Verificar si la posesión se perdió por acumulación
        if (IsPossessionLost(ctx)) {
            return;  // robo por presión/exposición acumulada
        }
        
        // Verificar gol en FINISH
        if (action == FINISH && Roll() < GOAL_CHANCE_FINISH) {
            ScoreGoal();  // actualiza marcador
            return;       // posesión termina tras gol
        }
        
        // Si llegó al arco rival sin definir, mantenerse cerca
        if (ctx.field_progress >= 100) {
            ctx.field_progress = 95;
        }
    }
}
```

---

# Selección de Acción por Probabilidad Fija

```cpp
ActionDefinition SelectAction(int32_t field_progress)
{
    // Zona 0-25 (propia mitad, cerca del propio arco):
    if (field_progress <= 25) {
        auto roll = Random(0.0, 1.0);
        if (roll < 0.6) return KEEP_BALL;      // 60% mantener
        if (roll < 0.9) return PROGRESS;        // 30% progresar
        return GO_BACK;                          // 10% retroceder
    }
    
    // Zona 25-60 (mediocampo y entrada a campo rival):
    if (field_progress <= 60) {
        auto roll = Random(0.0, 1.0);
        if (roll < 0.4) return KEEP_BALL;       // 40% mantener
        if (roll < 0.7) return PROGRESS;        // 30% progresar
        if (roll < 0.9) return DANGEROUS_ACTION; // 20% peligroso
        return FINISH;                           // 10% finalizar
    }
    
    // Zona 60-100 (campo rival, cerca del arco rival):
    auto roll = Random(0.0, 1.0);
    if (roll < 0.2) return PROGRESS;           // 20% progresar
    if (roll < 0.5) return DANGEROUS_ACTION;   // 30% peligroso
    return FINISH;                              // 50% finalizar
}
```

---

# Bucle Completo del Partido

```cpp
MatchContext SimulateMatch(Team home, Team away)
{
    auto ctx = MatchContext();
    
    // Sorteo inicial — equipo A saca en mitad de cancha
    bool coin_flip = Random(0.0, 1.0) > 0.5;
    ctx.home_possession = coin_flip ? true : false;
    ctx.field_progress = 50;
    
    ctx.match_time = 0;
    
    while (ctx.match_time < 90) {
        ProcessPossession(ctx);
        
        // Procesar el resultado de la posesión
        if (ctx.last_action == FINISH && ctx.goal_scored) {
            UpdateScore(ctx);
            
            // Reiniciar posesión desde mitad de cancha
            ctx.field_progress = 50;
            ctx.accumulated_exposure = 0;
            ctx.rival_pressure = 0;
        }
        else if (ctx.possession_lost) {
            // Elegir equipo que saca (aleatorio, ~50/50)
            ctx.home_possession = Random(0.0, 1.0) > 0.5;
            
            // Reiniciar posesión desde mitad de cancha
            ctx.field_progress = 50;
            ctx.accumulated_exposure = 0;
            ctx.rival_pressure = 0;
        }
        
        // Avanzar tiempo (simulado)
        ctx.match_time += RollTimeAdvance();
    }
    
    return ctx;
}
```

---

# Fase Posterior

Una vez validado el sistema base:

- Las estadísticas de jugadores modificarán exposición y efectos de acción
- La presión modelada reemplazará los valores fijos
- Las tácticas no modificarán probabilidades — modificarán distribuciones de decisión (qué acciones se prefieren en cada posición)
- El campo sigue siendo una sola barra compartida

Las tácticas son personalidad, no bonos.
# Contexto Maestro del Proyecto: `F1-2026-power-unit-ml`

> **Instrucción para asistentes de IA:** Este documento contiene todo el contexto técnico, académico y arquitectónico del proyecto. Léelo por completo antes de proponer código, arquitecturas o análisis. Mantén coherencia estricta con las restricciones del temario, la normativa FIA 2026 y la estructura del repositorio.

---

## 1. Perfil del Desarrollador y Contexto Académico

* **Autor / Usuario:** Borja (`BorjaBG`)
* **Formación previa:** Curso de Especialización en Inteligencia Artificial y Big Data.
* **Formación actual:** Bootcamp de Data Science.
* **Propósito del proyecto:** Proyecto Individual de Machine Learning (correspondiente al hito **3.8: Proyecto Individual - ML**, aunque el alumno cursará **todo el Módulo 3 completo** antes de su entrega final).
* **Idioma de trabajo:** Español para explicaciones, documentación y notebooks; inglés para nombres de variables, funciones, archivos y commits.

### Stack Técnico Cubierto en el Bootcamp
Cualquier solución propuesta debe apoyarse en las herramientas y algoritmos de este temario:
1. **Módulo 1 (Programación):** Python, colecciones, funciones, Programación Orientada a Objetos (POO).
2. **Módulo 2 (Análisis y Datos):** NumPy, Pandas, Fundamentos Estadísticos, Fuentes de Datos (APIs), Visualización (Matplotlib/Seaborn), Bases de Datos, BI y Data Wrangling.
3. **Módulo 3 (Machine Learning e IA):**
   * *Supervisado Clásico:* Regresión Lineal, Regresión Polinómica, Regularización (Ridge/Lasso), Clasificación y Regresión Logística, Árboles de Decisión, Ensamblados (Bagging y Boosting: Random Forest, XGBoost, LightGBM), KNN, SVM.
   * *Ingeniería de ML:* Pipelines de Scikit-Learn, Hiperparametrización (`GridSearchCV` / `RandomizedSearchCV`), Selección de variables.
   * *No Supervisado:* Clustering (K-Means, DBSCAN, GMM), Reducción de Dimensionalidad (PCA).
   * *Deep Learning y Avanzado:* Fundamentos de Redes Neuronales (MLP), Computer Vision (CV), NLP, Series Temporales y RNN (LSTM/GRU), Aprendizaje por Refuerzo (RL) e IA Generativa.

---

## 2. Objetivo y Visión del Proyecto

* **Repositorio GitHub:** `BorjaBG/F1-2026-power-unit-ml`
* **Tema:** Análisis, ingeniería inversa y optimización de **Mapas Motor (gestión energética de la Unidad de Potencia)** en la Fórmula 1 bajo el reglamento técnico de **2026**.
* **Problema de negocio / ingeniería:** En la F1 de 2026, un mapa motor ya no regula solo el motor de combustión (ICE), sino la estrategia óptima de despliegue y regeneración eléctrica entre el ICE y el MGU-K a lo largo de una vuelta. Dado que la batería tiene una capacidad limitada y existen curvas estrictas de recorte de potencia (*derating*) por velocidad, los equipos utilizan Machine Learning para simular el comportamiento del coche y encontrar la política óptima de despliegue energético.

---

## 3. Marco Físico y Reglamentario (FIA F1 2026)

Toda ingeniería de características (*Feature Engineering*) y entorno de simulación debe respetar las especificaciones de las Unidades de Potencia de 2026:

* **Eliminación del MGU-H:** Toda la recuperación energética proviene de la frenada o del *super-clipping* en recta a través del MGU-K.
* **Reparto de Potencia (aproximadamente equilibrado, no 50/50 exacto):**
  * Motor de Combustión Interna (ICE): $\approx 400\text{ kW}$ ($\sim 536\text{ CV}$).
  * Motor Eléctrico (MGU-K): hasta $350\text{ kW}$ ($\sim 469\text{ CV}$).
  * Nota: el reparto real es $\sim$53/47; la cifra "~50/50" es una licencia narrativa, no un dato de cálculo.
* **Límites Energéticos de Batería (ES):**
  * Capacidad máxima útil de almacenamiento: $4\text{ MJ}$ por ciclo.
  * Energía máxima recuperable por vuelta: hasta $9\text{ MJ}$ (en circuitos con suficientes zonas de frenada), lo que obliga a múltiples ciclos de carga y descarga dentro de una misma vuelta.
* **Curva Oficial de *Derating* del MGU-K (Modo Estándar):**
  La potencia eléctrica máxima permitida $P_{\text{MGU-K, max}}(v)$ decae linealmente con la velocidad instantánea $v$ (en $\text{km/h}$):
  $$P_{\text{MGU-K, max}}(v) = \begin{cases} 350\text{ kW} & \text{si } v \le 290 \\ 1800 - 5v\text{ kW} & \text{si } 290 < v < 340 \\ 6900 - 20v\text{ kW} & \text{si } 340 \le v \le 345 \\ 0\text{ kW} & \text{si } v > 345 \end{cases}$$
* **Modo *Override* (Adelantamiento):** Sustituye al antiguo DRS. Cuando un piloto está a menos de $1\text{ segundo}$ del coche precedente en el punto de detección, se elimina el recorte temprano de los $290\text{ km/h}$, permitiendo mantener los $350\text{ kW}$ completos hasta los $337\text{ km/h}$ y cortando a $0\text{ kW}$ a los $355\text{ km/h}$.

---

## 4. Arquitectura del Sistema (Pipeline en 3 Fases)

El proyecto integra distintas técnicas del Módulo 3 en un flujo modular y progresivo:

### Fase 1: Descubrimiento de Modos Motor Reales (No Supervisado)
* **Datos:** Telemetría real de Grandes Premios de la temporada 2026 extraída mediante la librería Python **`fastf1`** (`Speed`, `RPM`, `nGear`, `Throttle`, `Brake`, `Distance`, `DRS/Aero`).
* **Feature Engineering:** Cálculo de aceleración longitudinal $a_x = \frac{\Delta v}{\Delta t}$, comparación con la envolvente teórica de $350\text{ kW}$, detección de *super-clipping* (caída de aceleración con acelerador al $100\%$) y segmentación de vueltas en micro-sectores (ej. tramos de $50\text{ m}$).
* **Modelado (Módulos 3.9 y 3.10):** Reducción de dimensionalidad con **PCA** y segmentación mediante **Clustering** (K-Means / GMM) para clasificar automáticamente tramos de telemetría en modos operativos reales sin etiquetas previas (*Quali Max Attack*, *Carrera Equilibrado*, *Harvest/Clipping*, *Override Mode*).

### Fase 2: Gemelo Digital / Modelo Subrogado (Supervisado / Series Temporales)
* **Objetivo:** Construir un simulador rápido capaz de predecir el tiempo por micro-sector, la ganancia de velocidad y el consumo energético estimado dado un nivel de despliegue eléctrico y las condiciones del circuito.
* **Modelado (Módulos 3.5, 3.7, 3.11, 3.14):**
  * *Enfoque Tabular:* `Pipeline` de Scikit-Learn con `RobustScaler`, selección de variables y regresores de ensamblado (**XGBoost / LightGBM / Random Forest**) validados con `GroupKFold` (agrupando por piloto o carrera para evitar *data leakage*).
  * *Enfoque Secuencial (opcional/complementario):* Red neuronal recurrente (**LSTM / GRU**) o MLP para modelar la evolución continua de velocidad y estado de carga (SoC) a lo largo de la vuelta.

### Fase 3: Optimización del Mapa Motor y Tabla ECU (Reinforcement Learning)
* **Objetivo:** Replicar cómo los equipos generan mapas óptimos desde cero para un circuito específico (ej. Monza, Spa o Baréin).
* **Entorno (Módulo 3.15):** Entorno personalizado compatible con `gymnasium` que utiliza el Modelo Subrogado de la Fase 2 como motor físico y aplica las restricciones estrictas de la FIA ($4\text{ MJ}$ de batería, $9\text{ MJ}$ de recarga/vuelta, curva de *derating*).
* **Agente:** Entrenamiento de un agente de **Reinforcement Learning** (**PPO** o **DQN** con `stable-baselines3`) cuya función de recompensa minimiza el tiempo total por vuelta sin agotar la batería antes de meta.
* **Entregable Final:** Extracción de la política aprendida en formato de **tabla de consulta (*Lookup Table*) 2D/3D** (Velocidad $\times$ Marcha $\times$ SoC $\rightarrow$ kW MGU-K) y mapas 2D del circuito coloreados por intensidad de despliegue/recarga.

---

## 5. Estado Actual y Reglas de Colaboración para las IAs

* **Última actualización:** 15 nov 2026 — Revisión de formato: usuario corregido a `BorjaBG`, precisión en el reparto de potencia (~53/47), añadidas sección de entregables pendientes y glosario.
* **Estado actual del repositorio (`BorjaBG/F1-2026-power-unit-ml`):** Recién inicializado con `README.md` y `.gitignore` estándar de Python. Aún no se ha escrito código fuente ni descargado datos.
* **Reglas de interacción:**
  1. **Paso a paso:** No generar decenas de archivos de golpe si el usuario no lo pide; avanzar de forma incremental y validada.
  2. **Buenas prácticas de Data Science:** Evitar siempre el *data leakage* (usar `Pipeline` y `GroupKFold` cuando haya datos secuenciales/telemetría por vuelta).
  3. **Explicabilidad:** Traducir siempre las métricas técnicas ($RMSE$, $R^2$, $F1$, Silhouette Score) a implicaciones reales de ingeniería de Fórmula 1.
  4. **Actualización del documento:** Cada vez que se tome una decisión arquitectónica importante, se elija un circuito banco de pruebas o se complete una fase, actualizar la sección 5 de este archivo (incluida la línea "Última actualización").

---

## 6. Entregables y Evaluación — PENDIENTE DE ENUNCIADO

> **⚠️ Aún no se ha publicado el enunciado/instrucciones oficiales del hito 3.8.** Esta sección se completará cuando lleguen. Hasta entonces, las IAs **no deben asumir** requisitos de formato, estructura de repositorio, pesos de evaluación ni fechas de entrega. Los entregables mencionados en la Fase 3 (lookup table y mapas del circuito) son objetivos técnicos del proyecto, no requisitos oficiales confirmados.

* Pendiente: requisitos del repositorio (estructura, README, memoria).
* Pendiente: criterios de evaluación y pesos.
* Pendiente: fecha de entrega y formatos de presentación.

---

## 7. Glosario

* **ICE:** *Internal Combustion Engine*, motor de combustión interna (~400 kW en 2026).
* **MGU-K:** *Motor Generator Unit - Kinetic*, motor eléctrico acoplado al eje que despliega y recupera energía (hasta 350 kW en 2026).
* **MGU-H:** *Motor Generator Unit - Heat*, recuperador de energía del turbo. **Eliminado en el reglamento 2026.**
* **ES:** *Energy Store*, batería. Capacidad útil máxima de $4\text{ MJ}$ por ciclo en 2026.
* **SoC:** *State of Charge*, estado de carga de la batería (energía disponible en un instante dado).
* **Mapa Motor:** estrategia de gestión energética: cuánta energía eléctrica desplegar/recuperar en cada punto de la vuelta.
* **Derating:** recorte reglamentario de la potencia eléctrica máxima en función de la velocidad (curva de la sección 3).
* **Super-clipping:** recuperación de energía en recta levantando parcialmente el acelerador, con el MGU-K actuando como generador.
* **Override:** modo de adelantamiento de 2026 (sustituye al DRS): potencia eléctrica completa mantenida hasta ~337 km/h para el coche que persigue a menos de 1 segundo.

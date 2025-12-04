# **Simulación de Subasta Inteligente con Agentes Artificiales**

## **Materia: Inteligencia Artificial Distribuida – UTN-FRT**

### **Alumnos:**

* **Fonts, Ignacio Esteban (53524)**
* **Mercado, Mariano Agustín (53843)**
* **Farfan, Sebastián (19633)**

---

# **1. Montaje del entorno (Docker Compose)**

Para garantizar reproducibilidad y despliegue consistente, la simulación completa corre dentro de un contenedor **n8n** administrado por Docker.
El entorno se define mediante un archivo `docker-compose.yml`:

Este servicio ejecuta n8n en modo servidor, habilitando:

* Capacidad de **ejecutar workflows de agentes** cada minuto.
* Endpoints webhook GET/POST persistentes.
* Persistencia mediante Data Tables (almacenadas en SQLite interna del contenedor).

Para levantar el proyecto, insertar en la terminal:

```bash
docker compose up --build

```

---

# **2. Descripción de la simulación de subasta con agentes inteligentes**

El objetivo del experimento es estudiar **comportamientos de agentes autónomos** dentro de un escenario competitivo: **una subasta dinámica** donde cada agente recibe el estado actual y decide si realizar o no una puja.

El flujo general:

1. Cada minuto, el `Schedule Trigger` inicia la simulación.
2. Se obtiene el precio actual desde el Webhook GET (actuando como “sensor”).
3. Cada agente analiza este precio y decide:

   * si puja,
   * cuánto puja,
   * bajo qué estrategia.
4. Cada agente envía su decisión mediante Webhook POST.
5. El POST actualiza la tabla de estado global (Data Table):

   * registra puja,
   * actualiza mejor oferta,
   * define ganador,
   * actualiza precio actual.
6. El GET genera una interfaz HTML que visualiza:

   * estadísticas,
   * estado,
   * historial de pujas,
   * ganador actual.

Esto implementa un sistema de **IA Distribuida**, donde agentes autónomos colaboran/compiten sin comunicación directa entre sí, interactuando únicamente a través del entorno.

---

# **3. Comportamiento de los agentes**

Cada agente es un nodo “Code” con lógica propia. Sus decisiones se basan en:

* **precioActual** (visto mediante el nodo Sensor GET)
* **algoritmos probabilísticos**
* **actitudes estratégicas diferentes**

---

## **Agente 1 – Conservador**

* Presupuesto dinámico: **200–400 ARS**
* Solo puja cuando el precio es bajo.
* Incrementos pequeños y prudentes.

```js
const precio = parseInt($input.first().json.precioActual);
const presupuesto = Math.floor(Math.random() * 200) + 200; // 200–400

let pujar = precio < presupuesto;
let monto = pujar ? precio + Math.floor(Math.random() * 10) + 1 : 0;

return [{
  json: { agente: 'Agente Conservador', pujar, monto, estrategia: 'Minimizar Gasto' }
}];
```

---

## **Agente 2 – Magnate**

* Presupuesto muy alto: **700–1100 ARS**
* Pero ahora tiene un **30% de probabilidad de NO participar**, lo que elimina dominancia absoluta.
* Incrementos medianos.

```js
const precio = parseInt($input.first().json.precioActual);
const presupuesto = Math.floor(Math.random() * 400) + 700;
const quierePujar = Math.random() > 0.7;

let pujar = precio < presupuesto && quierePujar;
let monto = pujar ? precio + Math.floor(Math.random() * 25) + 5 : 0;

return [{
  json: { agente: 'Agente Magnate', pujar, monto, estrategia: 'Intimidación' }
}];
```

---

## **Agente 3 – Francotirador**

* Evalúa oportunidades reales.
* Solo puja si el precio está por debajo del **55% del valor real**.
* Incrementa moderadamente.

```js
const precio = parseInt($input.first().json.precioActual);
const valorReal = Math.floor(Math.random() * 200) + 500;

let pujar = precio < valorReal * 0.55;
let monto = pujar ? precio + (Math.floor(Math.random() * 15) + 5) : 0;

return [{
  json: { agente: 'Agente Francotirador', pujar, monto, estrategia: 'Oportunista' }
}];
```

---

## **Agente 4 – Caótico**

* Completamente azaroso.
* A veces puja ridículamente alto; otras no hace nada.

```js
const precio = parseInt($input.first().json.precioActual);

const pujar = Math.random() > 0.5;
const monto = pujar ? precio + Math.floor(Math.random() * 20) + 1 : 0;

return [{
  json: { agente: 'Agente Caótico', pujar, monto, estrategia: 'Aleatoria' }
}];
```

---

# **4. Construcción de la simulación**

La arquitectura está compuesta por dos workflows:

---

## **Workflow 1: TRIGGERS (Simulación Automática)**

Archivo: **Simulacion Subasta – TRIGGERS.json**

Este workflow:

1. Se ejecuta cada minuto.
2. Lee el estado actual vía GET.
3. Distribuye el precio actual a los 4 agentes.
4. Recibe 4 decisiones de puja y las envía al POST.
5. Mergea los resultados.
6. Obtiene el nuevo estado actualizado.
7. Renderiza HTML para visualizar la simulación.

<img width="1196" height="432" alt="{9BABCAC5-FE42-4ADB-8646-A615AE3BADC3}" src="https://github.com/user-attachments/assets/451b80de-6176-43eb-b2b1-9f14934a163b" />

---

## **Workflow 2: Lógica Central (GET + POST)**

Archivo: **Simulación Subasta Inteligente.json**

Incluye:

### **Webhook POST**

* Recibe la puja.
* Mezcla datos del agente con estado previo.
* Convierte el array `pujas` a string JSON.
* Actualiza mejor puja, ganador y precio actual.
* Guarda en Data Table.

### **Webhook GET**

* Lee la Data Table.
* Convierte la cadena JSON de pujas a objetos.
* Inyecta datos en la página HTML para visualizar los resultados.

Esta estructura separa **interacción**, **persistencia**, **decisiones** y **visualización**.

<img width="1225" height="479" alt="image" src="https://github.com/user-attachments/assets/b9285f13-4753-4f4e-ae51-6ee984d297cf" />

---

# **5. Casos de prueba (10 iteraciones)**

Se realizaron múltiples ciclos de ejecución, encontrando los siguientes patrones:

### ✔ **El Magnate ya no domina todas las rondas**

Gracias a:

* su probabilidad del 70% de participar,
* límites más suaves en monto.

### ✔ **El Francotirador gana cuando el precio es bajo al iniciar**

Funciona según intención: *solo compra oportunidades reales*.

### ✔ **El Conservador suele ganar primeras rondas**

Sus incrementos bajos lo mantienen competitivo en precios bajos.

### ✔ **El Caótico genera impredecibilidad real**

A veces sorprende y supera a todos los demás.

<img width="537" height="710" alt="{90AB1398-9DFE-40A8-B4AC-68EBE6D9B651}" src="https://github.com/user-attachments/assets/b10d2118-cccf-4130-ba08-3d59189b5c8e" />


---

# **6. Conclusiones**

* El montaje de agentes autónomos en un ambiente distribuido demuestra la interacción emergente entre estrategias opuestas.
* La simulación evidencia cómo **reglas simples generan comportamientos complejos**, un principio fundamental de la Inteligencia Artificial Distribuida.
* Ajustar parámetros probabilísticos permitió equilibrar la competencia, evitando agentes dominantes.
* La arquitectura con **Workflow + Data Table + Webhooks** funciona como un entorno multiagente totalmente descentralizado.
* El sistema puede escalar fácilmente para:

  * más agentes,
  * reglas evolutivas,
  * competencia cooperativa,
  * aprendizaje reforzado.

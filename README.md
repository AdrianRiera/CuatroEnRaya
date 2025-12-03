# Proyecto Acceso a Datos: API REST & WebSockets (Serverless Architecture)

**Asignatura:** Acceso a Datos (2º DAM)  
**Entrega:** 2.1 - Contrato formal API y WebSockets  

---

## 📖 Introducción

Este repositorio contiene el desarrollo correspondiente a la Entrega 2 del módulo de Acceso a Datos. El proyecto demuestra la implementación de patrones de diseño modernos para el intercambio de información:

1. **Práctica 1:** Diseño de una **API RESTful** para la gestión de datos (Base de datos Sakila).  
2. **Práctica 2:** Desarrollo de una aplicación en tiempo real (**Connect 4 Multiplayer**) utilizando el patrón **Event-Driven** sobre WebSockets, desplegada en una arquitectura 100% Serverless en **AWS**.

---

## ☁️ Arquitectura e Infraestructura (AWS Serverless)

Para la implementación de los WebSockets (Práctica 2), se ha diseñado una arquitectura en la nube de alto rendimiento, escalable y sin servidores. Esta infraestructura ha sido validada como solución técnica avanzada para el proyecto.

### 📊 Diagrama de Flujo


Client (Vue.js) <--> AWS CloudFront (CDN)
AWS CloudFront (CDN) <--> AWS S3 (Hosting Static)
Client (Vue.js) -->> AWS API Gateway (WebSocket): WSS
AWS API Gateway --> AWS Lambda: Invoke
AWS Lambda --> Amazon DynamoDB: Read/Write
🧩 Componentes del Sistema
Componente	Servicio AWS	Función
Frontend Host	S3 + CloudFront	Alojamiento SPA (Vue.js) + CDN
API Socket	API Gateway v2	Gestión de conexiones WebSocket
Backend Logic	AWS Lambda	Lógica del juego (turnos, victoria, emparejamiento)
Persistencia	DynamoDB	Estado de partidas y jugadores

🛠 Tecnologías y Versiones
Backend & Cloud
Lenguaje: Python 3.12

AWS SDK: Boto3 1.26+

Base de Datos: Amazon DynamoDB (On-Demand)

Infraestructura: AWS Lambda, API Gateway WebSocket V2

Frontend
Framework: Vue.js 3.x (Composition API)

Build Tool: Vite (latest)

Lenguajes: JavaScript (ES6+), HTML5, CSS3

📦 Dependencias
Backend (Python – requirements.txt)
Incluye librerías nativas del entorno Lambda:

boto3

json

decimal

os, random

Frontend (Node.js – package.json)
vue

@vitejs/plugin-vue

vite

⚙️ Manual de Instalación y Despliegue
1. Backend (AWS)
📌 Base de Datos
Crear tabla en DynamoDB:

Nombre: Connect4Games

Partition Key: GameId (String)

📌 Lógica (AWS Lambda)
Crear función Lambda con el código de /backend/lambda_function.py

Asignar permisos IAM:

AmazonDynamoDBFullAccess

AmazonAPIGatewayInvokeFullAccess

📌 API WebSocket (API Gateway)
Crear rutas:

$connect

$disconnect

joinGame

makeMove

voteTurn

requestRematch

declineRematch

Integrarlas todas con la Lambda y desplegar en stage: production.

2. Frontend
📦 Instalación local
bash
Copiar código
npm install
🔧 Configurar endpoint WSS
Editar el archivo de configuración del proyecto Vue e introducir la URL WebSocket generada por API Gateway.

🚀 Compilar y desplegar en S3
bash
Copiar código
npm run build
# Subir el contenido de /dist al bucket S3 conectado a CloudFront
🎮 Manual de Uso
🌐 Acceso a la aplicación
El proyecto se encuentra desplegado en AWS y accesible aquí:

👉 https://dole06ju1i10f.cloudfront.net/

## CI/CD
He usado Github Actions para automatizar la sincronización de archivos en el bucket S3 con la carpeta dist que crea al hacer npm run build y seguidamente fuerza la caché de Cloudfront para ver los cambios más recientes. El deploy.yml donde declaro estas acciones está en .github -> workflows -> deploy.yml.

🕹️ Guía paso a paso (Connect 4)
1. Iniciar sesión de juego
El cliente establece la conexión WebSocket ($connect).

El usuario elige entre:

Crear Partida

Unirse a Partida

2. Crear una Partida
Clic en Crear Partida Nueva

El sistema genera un Game ID único

Copiar y enviar al contrincante

3. Unirse a una Partida
Introducir el Game ID recibido

Pulsar Unirse

4. Sistema de Votación (Novedad)
Ambos jugadores votan quién empieza (Rojo o Amarillo):

Si ambos coinciden → empieza ese color

Si no → el servidor elige aleatoriamente

5. Desarrollo del juego
Cada acción del jugador envía un evento makeMove

El backend valida:

Turnos

Jugadas válidas

Estado del tablero

Victoria o empate

DynamoDB se actualiza

El nuevo tablero se envía a ambos jugadores en tiempo real

6. Finalización y Revancha
El servidor detecta:

4 en raya

Empate

Sistema de revancha:

requestRematch

declineRematch

Si ambos aceptan → partida reiniciada manteniendo conexión

👥 Autores
Adrián Riera Granada – Desarrollo Fullstack & Arquitectura Cloud AWS

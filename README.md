# SM2_EXAMEN_PRACTICO

**Nombre:** Luis Eduardo Caxi Calani  
**Proyecto:** CcontaPub  

## Descripción del Proyecto

**CcontaPub** es una aplicación que tiene como objetivo mantener a los Contadores Públicos de Tacna actualizados e informados a través de materiales digitales, promoviendo una plataforma de comunicación y recursos en línea para el gremio. La aplicación ofrece funcionalidades clave para asegurar la autenticidad de los usuarios y facilitar la interacción entre ellos.

---

## Historias de Usuario

### Historia de Usuario 01 - Scrapeo de Validación de Usuarios

**Descripción:**  
Como usuario que se registra en la aplicación, quiero que mi estado como contador habilitado sea validado automáticamente al registrarme, de modo que se asegure mi autenticidad y se extraigan los datos relevantes para mi perfil.

**Criterios de Aceptación:**
- La aplicación debe realizar un proceso de validación automática del registro del contador mediante scrapeo de una fuente de datos confiable (https://ccptacna.pe/).
- Si el contador no es habilitado, se debe notificar al usuario con un mensaje de error y no permitir la creación del perfil.
- La información extraída debe ser precisa y almacenada en el perfil del usuario.

# Código de Backend

## 1. Scrapeo de Validación de Usuarios

Este endpoint utiliza la librería `puppeteer` para realizar el scrapeo de datos en la página de https://ccptacna.pe/, verificando el estado de habilitación de un contador.

```javascript
const express = require('express');
const puppeteer = require('puppeteer');
const app = express();

app.use(express.json());

app.post('/validate-user', async (req, res) => {
  const { identifier } = req.body; // El usuario ingresa su código o nombre completo
  try {
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    
    await page.goto('https://ccptacna.pe/web/miembros.php');

    await page.type('#searchInput', identifier);
    await page.click('#searchButton');
    
    await page.waitForSelector('#resultContainer', { timeout: 10000 });

    const isEnabled = await page.evaluate(() => {
      const statusElement = document.querySelector('#status'); 
      return statusElement && statusElement.innerText.includes('Habilitado');
    });

    await browser.close();

    if (isEnabled) {
      res.status(200).json({ message: 'Usuario habilitado' });
    } else {
      res.status(403).json({ message: 'Usuario no habilitado' });
    }
  } catch (error) {
    console.error('Error en la validación:', error);
    res.status(500).json({ message: 'Error en la validación', error: error.message });
  }
});

```

### Historia de Usuario 02 - Chat Privado entre Usuarios

**Descripción:**  
Como contador público registrado en la aplicación, quiero poder comunicarme con otros contadores mediante un chat privado, de modo que podamos realizar consultas o resolver dudas de manera oportuna, asegurando que las conversaciones se mantengan solo por un tiempo limitado.

**Criterios de Aceptación:**
- Los usuarios pueden iniciar chats privados con otros contadores registrados para realizar consultas.
- Cada conversación debe tener una duración máxima de 24 horas, tras lo cual el chat se eliminará automáticamente.
- El usuario debe recibir una notificación antes de que el chat expire y se elimine.

# Código de Backend

## 2. Chat Privados

Esta funcionalidad permite manejar conversaciones privadas. Los controladores permiten iniciar y gestionar conversaciones, obtener mensajes, unirse y salir de conversaciones, y actualizar fotos de conversación. Para la comunicación en tiempo real, el sistema utiliza WebSockets.

```javascript
const conversationService = require('../services/conversationService');
const messageService = require('../services/messageService');
const appError = require('../utils/appError');
const resSend = require('../utils/resSend');
const cryptoFeatures = require('../utils/crytoFeatures');
const catchAsync = require('../utils/catchAsync');
const requireField = require('./../utils/requireField');
const translatorNext = require('../utils/translatorNext');
const socketController = require('./socketController')
const emitToActiveUsersInRoom = require('../utils/socketEmitActiveUsersRoom');
const emitMessageToUser = require('../utils/socketEmitMessageToUser');
const handleClientError = (socket, statusCode, message) => {
  socket.emit('clientError', { statusCode, status: message });
};

const emitUserConversations = async (socket, senderId) => {
  const responseConversations = await conversationService.getUserConversationsService(senderId);
  
  if (!responseConversations.success) {
    socket.emit('receiveUserConversations', {
      statusCode: 200,
      status: true,
      data: null,
    });
    return;
  }

  const responseDetailsMessageConversations = await Promise.all(
    responseConversations.data.map(async (message) => {
      const responseUnreadMessage = await messageService.getUnreadMessagesService(message._id, senderId);
      message._doc.countUnread = responseUnreadMessage.data;
      message.lastMessage.content = cryptoFeatures.decrypt(message.lastMessage.content);
      return message;
    })
  );
  socket.emit('receiveUserConversations', {
    statusCode: 200,
    status: true,
    data: responseDetailsMessageConversations,
  });
};

exports.initiateConversationController = catchAsync(async (req, res, next) => {
  const { userId, memberUserId, content } = req.body;

  if (requireField(userId, memberUserId, content)) {
    return next(new appError(translatorNext(req, 'MISSING_REQUIRED_FIELDS'), 400));
  }

  const response = await conversationService.initiateConversationService([{ userId: memberUserId }], userId, content);

  const socket = socketController()
  const io=socket.io

  const usersInvolved = [userId, memberUserId];

  usersInvolved.forEach(userId => {
    emitMessageToUser(io,userId, 'getUserConversations', {
      statusCode: response.status,
      status: response.success,
      data: response.data,
    });
  });

  resSend(res, { statusCode: response.status, status: response.success, data: response.data, message: response.code });
});

exports.getUserConversationsController = async (io, socket) => {
  await emitUserConversations(socket, socket.userId);
};

exports.getConversationMessagesController = async (io, socket, conversationId) => {
  socket.currentRoomId = conversationId;
  const senderId = socket.userId;

  const responseMarkMessage = await messageService.markMessagesAsReadService(conversationId, senderId);
  
  if (!responseMarkMessage.success) {
    return handleClientError(socket, 400, 'Error al marcar mensajes');
  }

  const response = await messageService.getAllMessagesService(conversationId);
  
  if (!response.success) {
    return handleClientError(socket, response.statusCode, 'Error al obtener los mensajes');
  }

  await emitUserConversations(socket, senderId);

  await emitToActiveUsersInRoom(io, conversationId, 'getConversationMessages', {
    statusCode: response.status,
    status: response.success,
    data: response.data,
  });
};

exports.joinConversationController = async (io, socket, conversationId) => {
  socket.join(conversationId);
  io.in(conversationId).emit('joinConversation', { statusCode: 200, status: 'Se unio a la conversacion' });
};

exports.leaveConversationController = async (io, socket, conversationId) => {
  socket.leave(conversationId);
  io.in(conversationId).emit('leaveConversation', { statusCode: 200, status: 'Salio de la conversacion' });
};

exports.editPhotoConversationController = catchAsync(async (req, res, next) => {
  const { conversationId } = req.params;

  if (requireField(conversationId)) {
    return next(new appError(translatorNext(req, 'MISSING_REQUIRED_FIELDS'), 400));
  }

  // Implement the photo editing logic here
  // const response = await conversationService.editPhotoConversationService(conversationId, req.fileDetails);

  // For now, we'll just return a placeholder response
  const response = { status: 200, success: true, data: null, code: 'PHOTO_UPDATED' };
  resSend(res, { statusCode: response.status, status: response.success, data: response.data, message: response.code });
});
```


## Enlaces y Referencias

### Librerías y Herramientas
- [Express](https://expressjs.com/) - Framework web rápido y minimalista para Node.js.
- [Socket.IO](https://socket.io/) - Biblioteca para comunicación en tiempo real entre clientes y servidores.
- [Puppeteer](https://pptr.dev/) - Herramienta para el scrapeo de sitios web mediante un navegador sin cabeza.
- [Mongoose](https://mongoosejs.com/) - ODM (Mapeo de objetos a documentos) para MongoDB, utilizado para gestionar la base de datos de manera eficiente.
- [Crypto](https://nodejs.org/api/crypto.html) - Biblioteca nativa de Node.js para cifrado de datos, usada en el proyecto para la encriptación y desencriptación de mensajes.

### Servicios y APIs Externas
- [Colegio de Contadores Públicos de Tacna](https://ccptacna.pe/) - Fuente de datos oficial utilizada para validar a los contadores públicos registrados mediante scrapeo de datos.

### Otros Recursos
- [Node.js](https://nodejs.org/) - Entorno de ejecución de JavaScript en el lado del servidor.



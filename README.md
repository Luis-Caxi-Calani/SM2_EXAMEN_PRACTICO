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

### Historia de Usuario 02 - Chat Privado entre Usuarios

**Descripción:**  
Como contador público registrado en la aplicación, quiero poder comunicarme con otros contadores mediante un chat privado, de modo que podamos realizar consultas o resolver dudas de manera oportuna, asegurando que las conversaciones se mantengan solo por un tiempo limitado.

**Criterios de Aceptación:**
- Los usuarios pueden iniciar chats privados con otros contadores registrados para realizar consultas.
- Cada conversación debe tener una duración máxima de 24 horas, tras lo cual el chat se eliminará automáticamente.
- El usuario debe recibir una notificación antes de que el chat expire y se elimine.

---

## Enlaces y Referencias

Si has utilizado recursos externos (librerías, APIs, etc.), menciona los enlaces o referencias aquí.

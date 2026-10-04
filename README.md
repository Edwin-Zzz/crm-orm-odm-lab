# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas
**1. Dos motores.**
Una razon por la cual Activity se implementa dentro de una base de datos documental es por su atributo "metadata" que como podemos observar en la implementacion del reto 6 es un atributo que cambia de estructura segun el tipo de actividad (CALL, EMAIL, MEETING). Es esta flexibilidad que la hace candidata para una BD NoSQL. Company y Contact son candidatas para una estructura relacional por las relaciones estructuradas que existen entre ellas (un contacto pertenece a una empresa) algo que refuerza una base de datos relacional y facilita las consultas sobre ella, algo que pudimos observar tambien en el reto 5.

**2. ORM vs ODM.**
Un ORM y un ODM son "traductores" que nos dejan trabajar con las bases de datos usando objetos y funciones de JavaScript en vez de escribir las consultas o inserts a mano. Aqui usamos Sequelize como el ORM para conectar a PostgreSQL y Mongoose como ODM para conectar a MongoDB. Una diferencia importante es en donde se aplica la estructura: en Sequelize la respalda PostgreSQL en su Schema con "allowNull: false" en contact.js, pero el esquema de Mongoose solo lo aplica Mongoose (el ODM, no la base de datos), y en las actualizaciones hay que pedir las validaciones con "runValidators: true", como en el reto 8.

**3. Configuracion por variables de entorno.**
Las variables estan definidas en .devcontainer/docker-compose.yml, en el bloque environment conteniendo: DB_HOST: postgres, MONGODB_URI: mongodb://mongo:27017/crm, usuario, password, etc. Docker las inyecta al contenedor y el codigo las lee con process.env. Estas no se escriben en los .js porque el codigo se sube al repositorio y cualquiera que lo vea tendria las credenciales (esto cae dentro del OWASP Top 10 2025 como el A07 Authentication Failures). Los hosts no son localhost porque app, postgres y mongo son contenedores separados. localhost dentro de app seria la app misma, asi que se usan los nombres de los servicios.

**4. Asociaciones.**
En models/sequelize/index.js podemos ver que un contacto tiene 1 empresa (Contact.belongsTo) y una empresa puede tener varios contactos (Company.hasMany), generando una relacion 1 -> N entre contacts y company. La llave foranea utilizada es companyId que vive en la tabla de contacts. El alias as: "contacts" sirve para que, al usar include: { model: Contact, as: "contacts" }, la información incluida del modelo Contact aparezca listada bajo ese nombre de campo (contacts) en el resultado.

**5. Eager loading.**
Traer la compañía y luego hacer una segunda consulta para sus contactos implicaria hacer dos viajes separados a la base de datos, mientras que usar include hace que Sequelize genere una sola consulta con un JOIN que trae todo junto. Es preferible el include porque reduce la latencia total y evita múltiples viajes a la base de datos.

**6. Instancia vs consulta.**
Buscar primero el registro con findByPk y luego .update() nos permite validar si el registro existe primero (y mandar un 404 en caso de que no este) y permite regresar el objeto actualizado. Mientras que usar Model.update({...}, { where }) directo, aunque si nos permite validar si un registro existe, no regresa el objeto actualizado, sino solo la cantidad de filas afectadas. La ventaja de usar find + update es que nos facilita la revision y uso directo del objeto, pero cuesta dos consultas, mientras que usar un .update + where es mas eficiente, sobre todo para las actualizaciones masivas, aunque para revisar los cambios se requieren consultas adicionales.

   ## Evidencia
   <img width="811" height="540" alt="npm test con las 9 pruebas en verde" src="https://github.com/user-attachments/assets/da46b840-156b-45cb-9dde-ce5e5c993e37" />


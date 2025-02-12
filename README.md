# Documentación General para Crear una AWS Lambda con Serverless Framework


## 📌 Índice
1. Introducción  
2. Requisitos Previos  
3. Configuración del Entorno  
4. Estructura del Proyecto  
5. Cuándo Usar un Repositorio (repository.ts)  
6. Configuración de Serverless Framework  
7. Implementación del Código  
8. Gestión de Variables de Entorno con AWS Secrets Manager  
9. Despliegue con Serverless  
10. Pruebas Locales y en AWS  
11. Conclusión  

---

## 1️⃣ Introducción
Este documento proporciona una guía para configurar una AWS Lambda con Serverless Framework, siguiendo una arquitectura modular y utilizando AWS Secrets Manager para gestionar variables de entorno de manera segura.

## 2️⃣ Requisitos Previos
Antes de comenzar, asegúrate de tener instalado:
- **AWS CLI** → Para interactuar con AWS
- **Node.js (v18.x o superior)** → Para ejecutar código JavaScript/TypeScript
- **Serverless Framework** → Para gestionar la AWS Lambda

### 📌 Instalación de Herramientas
```sh
# Instalar Serverless Framework
npm install -g serverless

# Configurar credenciales de AWS
aws configure
```

## 3️⃣ Configuración del Entorno

### 📍 Crear un Proyecto Serverless
```sh
serverless create --template aws-nodejs --path my-lambda-project
cd my-lambda-project
npm init -y
```

### 📍 Instalar Dependencias
```sh
npm install dotenv aws-sdk serverless-offline
npm install --save-dev typescript @types/node
```

### 📍 Inicializar TypeScript
```sh
npx tsc --init
```

## 4️⃣ Estructura del Proyecto
```sh
src/
│── modules/
│   ├── example/
│   │   ├── handlers/
│   │   │   ├── example.handler.ts
│   │   ├── service.ts
│   │   ├── repository.ts
│   │   ├── utils.ts
│
│── common/
│   ├── utils/
│   │   ├── responseHandler.ts
│   │   ├── secretsManager.ts
│
│── config/
│   ├── db.ts
│
│── tests/
│   ├── example.test.ts
│── serverless.yml
```

## 5️⃣ Cuándo Usar un Repositorio (repository.ts)
El repositorio es una capa dedicada a la interacción con bases de datos, APIs externas o almacenamiento.

✅ Cuando necesitas hacer consultas a bases de datos (SQL, DynamoDB, MongoDB, etc.).
✅ Cuando interactúas con una API externa y quieres mantener la lógica separada.
✅ Cuando quieres evitar que el servicio (service.ts) se encargue de la conexión a datos.

### 📍 Ejemplo de repository.ts (Conexión a DynamoDB)
```ts
import AWS from 'aws-sdk';

const dynamoDB = new AWS.DynamoDB.DocumentClient();

export const getItemById = async (id: string) => {
    const params = {
        TableName: 'MyTable',
        Key: { id }
    };
    const result = await dynamoDB.get(params).promise();
    return result.Item || null;
};
```

### 📍 Integración en service.ts
```ts
import { getItemById } from './repository';

export const fetchData = async (id: string) => {
    return await getItemById(id);
};
```

## 6️⃣ Configuración de Serverless Framework
```yaml
service: my-lambda-project

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  environment:
    ENV: ${opt:stage, 'dev'}
    SECRET_VALUE: ${ssm:/my-lambda/${self:provider.environment.ENV}/secret}

functions:
  exampleFunction:
    handler: src/modules/example/handlers/example.handler
    events:
      - http:
          path: example
          method: get
```

## 7️⃣ Implementación del Código
### 📍 Manejador de Lambda - example.handler.ts
```ts
import { APIGatewayEvent } from 'aws-lambda';
import { fetchData } from '../service';
import { sendResponse } from '../../../common/utils/responseHandler';

export const exampleHandler = async (event: APIGatewayEvent) => {
    try {
        const id = event.queryStringParameters?.id || 'default';
        const result = await fetchData(id);
        return sendResponse(200, { message: 'Success', data: result });
    } catch (error) {
        return sendResponse(500, { message: 'Internal Server Error', error: error.message });
    }
};
```

## 8️⃣ Gestión de Variables de Entorno con AWS Secrets Manager

### 📍 Crear Secretos en AWS SSM Parameter Store
```sh
aws ssm put-parameter --name "/my-lambda/dev/secret" \
  --value "MyDevSecretValue" --type "SecureString" --overwrite

aws ssm put-parameter --name "/my-lambda/prod/secret" \
  --value "MyProdSecretValue" --type "SecureString" --overwrite
```

### 📍 Obtener Secretos en secretsManager.ts
```ts
import AWS from 'aws-sdk';

const ssm = new AWS.SSM({ region: 'us-east-1' });

export async function getSecretValue(paramName: string): Promise<string | null> {
    try {
        const response = await ssm.getParameter({ Name: paramName, WithDecryption: true }).promise();
        return response.Parameter?.Value || null;
    } catch (error) {
        console.error(`Error retrieving parameter: ${error}`);
        return null;
    }
}
```

## 9️⃣ Despliegue con Serverless
```sh
serverless deploy --stage dev
serverless deploy --stage prod
```

## 🔟 Pruebas Locales y en AWS
```sh
serverless offline
curl -X GET "http://localhost:3000/example?id=123"
```

## 🔟 Conclusión
Ahora tienes una AWS Lambda bien estructurada con Serverless Framework. 
🚀 ¡Listo para producción! 🎉


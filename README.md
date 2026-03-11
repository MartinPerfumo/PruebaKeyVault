# Guía: Conectar Azure Key Vault con GitHub Actions

Esta guía te explica paso a paso cómo configurar el acceso desde GitHub Actions a secretos almacenados en Azure Key Vault.

---

## 📋 Índice

1. [Conceptos básicos](#conceptos-básicos)
2. [Crear un Key Vault en Azure](#1-crear-un-key-vault-en-azure)
3. [Almacenar secretos en el Key Vault](#2-almacenar-secretos-en-el-key-vault)
4. [Crear un Service Principal](#3-crear-un-service-principal)
5. [Dar permisos al Service Principal](#4-dar-permisos-al-service-principal)
6. [Configurar secretos en GitHub](#5-configurar-secretos-en-github)
7. [Usar en GitHub Actions](#6-usar-en-github-actions)

---

## Conceptos básicos

### ¿Qué es un Key Vault?
Es un servicio de Azure que almacena de forma segura secretos, contraseñas, certificados y claves de cifrado. Es como una caja fuerte digital.

### ¿Qué es un Service Principal?
Es una identidad para aplicaciones y servicios automatizados. Como tu aplicación o GitHub Actions no puede usar tu cuenta personal, necesita su propia "cuenta de servicio" para autenticarse en Azure.

### ¿Por qué hay que dar permisos?
Por seguridad. Cuando creas un Service Principal, por defecto no tiene acceso a nada. Debes darle explícitamente solo los permisos mínimos que necesita.

---

## 1. Crear un Key Vault en Azure

### Desde el Portal de Azure:

1. **Acceder al Portal**
   - Ve a [https://portal.azure.com](https://portal.azure.com)
   - Inicia sesión con tu cuenta

2. **Buscar el servicio Key Vault**
   - En la barra de búsqueda superior, escribe "Key Vault"
   - Haz clic en "Key vaults" en los resultados

3. **Crear nuevo Key Vault**
   - Haz clic en el botón **"+ Create"** o **"+ Crear"**
   
4. **Configuración básica**
   - **Subscription**: Selecciona tu suscripción
   - **Resource group**: Selecciona un grupo de recursos existente o crea uno nuevo
   - **Key vault name**: Elige un nombre único (ej: "mi-app-keyvault")
   - **Region**: Selecciona la región más cercana a tus usuarios
   - **Pricing tier**: Normalmente "Standard" es suficiente

5. **Configuración de acceso**
   - En la pestaña **"Access configuration"**
   - **Permission model**: Selecciona **"Azure role-based access control (RBAC)"** (recomendado)
   - Marca la opción **"Azure Virtual Machines for deployment"** si necesitas que VMs accedan

6. **Configuración de red (opcional)**
   - En **"Networking"** puedes dejar "All networks" para desarrollo
   - Para producción, es mejor restringir a IPs o redes específicas

7. **Revisar y crear**
   - Haz clic en **"Review + create"**
   - Revisa la configuración
   - Haz clic en **"Create"**

**Tiempo de creación**: 1-2 minutos aproximadamente.

---

## 2. Almacenar secretos en el Key Vault

> **⚠️ IMPORTANTE**: Si configuraste tu Key Vault con RBAC (recomendado en el paso 1), primero necesitas darte permisos a ti mismo para poder crear secretos. Sigue el paso adicional a continuación antes de intentar crear secretos.

### Paso previo: Darte permisos para gestionar secretos (solo si usas RBAC)

Cuando creas un Key Vault con RBAC, **ni siquiera tú tienes permisos automáticos** para trabajar con los secretos. Necesitas asignarte el rol apropiado:

1. **Ve a tu Key Vault** en el Portal de Azure

2. **Ve a "Access Control (IAM)"** en el menú lateral

3. **Haz clic en "+ Add"** → **"Add role assignment"**

4. **Selecciona el rol "Key Vault Secrets Officer"**:
   - En la pestaña **"Role"**, busca **"Key Vault Secrets Officer"**
   - Este rol te permite crear, leer, modificar y eliminar secretos
   - Selecciónalo y haz clic en **"Next"**

5. **Selecciónate a ti mismo como miembro**:
   - En **"Assign access to"**, deja **"User, group, or service principal"**
   - Haz clic en **"+ Select members"**
   - Busca tu nombre de usuario o email
   - Selecciónalo y haz clic en **"Select"**

6. **Asignar el rol**:
   - Haz clic en **"Review + assign"**

7. **Espera 1-2 minutos** para que los permisos se propaguen

**Error común**: Si intentas crear un secreto sin estos permisos, verás un error como:
```
Caller is not authorized to perform action on resource.
Action: 'Microsoft.KeyVault/vaults/secrets/setSecret/action'
```

### Ahora sí: Crear secretos

Una vez que tienes los permisos necesarios, puedes crear secretos:

1. **Ir a tu Key Vault**
   - En el Portal de Azure, busca "Key vaults"
   - Haz clic en el Key Vault que acabas de crear

2. **Acceder a la sección de Secrets**
   - En el menú lateral izquierdo, busca **"Objects"** o **"Objetos"**
   - Haz clic en **"Secrets"** o **"Secretos"**

3. **Crear un nuevo secreto**
   - Haz clic en **"+ Generate/Import"** o **"+ Generar/Importar"**

4. **Configurar el secreto**
   - **Upload options**: Deja "Manual"
   - **Name**: Pon un nombre descriptivo (ej: "database-password")
   - **Value**: Escribe el valor del secreto (lo que quieres proteger)
   - **Content type** (opcional): Puedes poner una descripción
   - **Activation date** (opcional): Cuándo estará disponible
   - **Expiration date** (opcional): Cuándo expirará
   - **Enabled**: Deja activado

5. **Guardar**
   - Haz clic en **"Create"**

**Repite este proceso** para cada secreto que necesites almacenar.

---

## 3. Crear un Service Principal

Un Service Principal es la "identidad" que usará GitHub Actions para autenticarse en Azure.

### Opción A: Desde Azure CLI (más rápido)


Primero obtén tu subscription ID:
```bash
az account show --query id --output tsv
```

Si tienes Azure CLI instalado, puedes usar este comando:

```bash
az ad sp create-for-rbac --name "github-actions-keyvault" --role contributor --scopes /subscriptions/{tu-subscription-id} --sdk-auth
```


Este comando te devolverá un JSON completo con todas las credenciales que necesitas. **Copia todo el JSON.**

### Opción B: Desde Azure Portal (más visual)

1. **Ir a App registrations**
   - En el menú lateral, busca **"App registrations"** o **"Registros de aplicaciones"**
   - Haz clic ahí

2. **Crear nueva aplicación**
   - Haz clic en **"+ New registration"** o **"+ Nuevo registro"**

3. **Configurar la aplicación**
   - **Name**: Pon un nombre descriptivo (ej: "github-actions-keyvault")
   - **Supported account types**: Deja la primera opción (Accounts in this organizational directory only)
   - **Redirect URI**: Déjalo en blanco
   - Haz clic en **"Register"**

4. **Copiar información importante**
   Una vez creada, verás una página con información. **Copia y guarda**:
   - **Application (client) ID**: Lo necesitarás más tarde
   - **Directory (tenant) ID**: También lo necesitarás

5. **Crear un secreto (contraseña) para la aplicación**
   - En el menú lateral, ve a **"Certificates & secrets"** o **"Certificados y secretos"**
   - Haz clic en la pestaña **"Client secrets"**
   - Haz clic en **"+ New client secret"** o **"+ Nuevo secreto de cliente"**
   - **Description**: Pon algo como "GitHub Actions secret"
   - **Expires**: Selecciona cuándo expirará (ej: 12 meses, 24 meses)
   - Haz clic en **"Add"**

6. **Copiar el valor del secreto**
   - **¡IMPORTANTE!** Se mostrará un **"Value"** - **Cópialo AHORA**
   - **Solo se mostrará una vez**. Si no lo copias, tendrás que crear uno nuevo
   - Guárdalo en un lugar seguro temporalmente


---

## 4. Dar permisos al Service Principal

Ahora que tienes el Service Principal creado, necesitas darle permisos para acceder a los secretos del Key Vault.

### Pasos:

1. **Ir a tu Key Vault**
   - Busca "Key vaults" en el Portal
   - Haz clic en tu Key Vault

2. **Acceder a Control de Acceso**
   - En el menú lateral, busca **"Access Control (IAM)"**
   - Haz clic ahí

3. **Agregar asignación de rol**
   - Haz clic en **"+ Add"** → **"Add role assignment"** o **"Agregar asignación de roles"**

4. **Seleccionar el rol**
   - En la pestaña **"Role"**, busca:
     - **"Key Vault Secrets User"** - Si solo necesitas LEER secretos (recomendado para GitHub Actions)
     - **"Key Vault Secrets Officer"** - Si necesitas leer, crear, modificar y eliminar secretos
   - Selecciona el rol y haz clic en **"Next"**

5. **Seleccionar el miembro (tu Service Principal)**
   - En la pestaña **"Members"**
   - En **"Assign access to"**, deja seleccionado **"User, group, or service principal"**
   - Haz clic en **"+ Select members"**
   - En el buscador, escribe el **nombre** que le diste a tu aplicación (ej: "github-actions-keyvault")
   - Haz clic en el resultado para seleccionarlo
   - Haz clic en **"Select"** abajo

6. **Revisar y asignar**
   - Haz clic en **"Next"**
   - Revisa que todo esté correcto
   - Haz clic en **"Review + assign"** o **"Revisar y asignar"**

**Listo**: Ahora tu Service Principal tiene permisos para leer secretos del Key Vault.

---

## 5. Configurar secretos en GitHub

Ahora necesitas guardar las credenciales del Service Principal en GitHub para que tu workflow pueda usarlas.

### Pasos:

1. **Ir a tu repositorio en GitHub**
   - Abre GitHub y ve a tu repositorio

2. **Acceder a Settings**
   - Haz clic en la pestaña **"Settings"** del repositorio
   - (Necesitas permisos de administrador del repositorio)

3. **Ir a Secrets and variables**
   - En el menú lateral izquierdo, busca **"Secrets and variables"**
   - Haz clic en **"Actions"**

4. **Crear secretos nuevos**
   - Haz clic en **"New repository secret"**

### Opción A: Si usaste Azure CLI (tienes el JSON completo):

Crea UN SOLO secreto:
- **Name**: `AZURE_CREDENTIALS`
- **Value**: Pega todo el JSON que te devolvió Azure CLI
- Haz clic en **"Add secret"**

### Opción B: Si usaste el Portal (tienes valores separados):

Crea TRES secretos separados:

**Secreto 1:**
- **Name**: `AZURE_CLIENT_ID`
- **Value**: El "Application (client) ID" que copiaste
- Haz clic en **"Add secret"**

**Secreto 2:**
- **Name**: `AZURE_TENANT_ID`
- **Value**: El "Directory (tenant) ID" que copiaste
- Haz clic en **"Add secret"**

**Secreto 3:**
- **Name**: `AZURE_CLIENT_SECRET`
- **Value**: El "Value" del secreto que creaste en el Service Principal
- Haz clic en **"Add secret"**

**También necesitarás:**

**Secreto 4:**
- **Name**: `AZURE_SUBSCRIPTION_ID`
- **Value**: Tu Subscription ID de Azure (puedes encontrarlo en el Portal, en "Subscriptions")
- Haz clic en **"Add secret"**

---

## 6. Usar en GitHub Actions

Ahora ya puedes acceder a tus secretos de Key Vault desde un workflow de GitHub Actions.

### Ejemplo de workflow completo:

#### Si usas AZURE_CREDENTIALS (JSON):

```yaml
name: Acceder a Azure Key Vault

on:
  workflow_dispatch:  # Se ejecuta manualmente
  push:
    branches: [ main ]

jobs:
  get-secrets:
    runs-on: ubuntu-latest
    
    steps:
    # 1. Checkout del código
    - name: Checkout código
      uses: actions/checkout@v3
    
    # 2. Login en Azure con las credenciales
    - name: Login en Azure
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    # 3. Obtener secreto del Key Vault
    - name: Obtener secreto
      uses: azure/get-keyvault-secrets@v1
      with:
        keyvault: "mi-app-keyvault"  # Nombre de tu Key Vault
        secrets: 'database-password, api-key'  # Nombres de los secretos separados por comas
      id: keyvault-secrets
    
    # 4. Usar el secreto (ejemplo: mostrar que existe sin revelar el valor)
    - name: Usar secreto
      run: |
        echo "Secreto obtenido correctamente"
        # Los secretos están disponibles como:
        # ${{ steps.keyvault-secrets.outputs.database-password }}
        # ${{ steps.keyvault-secrets.outputs.api-key }}
```

#### Si usas secretos separados:

```yaml
name: Acceder a Azure Key Vault

on:
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  get-secrets:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout código
      uses: actions/checkout@v3
    
    - name: Login en Azure
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        # Para autenticación con secret:
        client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
    
    - name: Obtener secretos del Key Vault
      uses: azure/get-keyvault-secrets@v1
      with:
        keyvault: "mi-app-keyvault"
        secrets: 'database-password, api-key'
      id: keyvault-secrets
    
    - name: Usar secreto
      run: |
        echo "Secretos obtenidos correctamente"
        # Usa: ${{ steps.keyvault-secrets.outputs.database-password }}
```

---

## 📝 Resumen del flujo completo

1. **Creas un Key Vault** en Azure → Es tu caja fuerte
2. **Guardas secretos** en el Key Vault → Son tus valores protegidos
3. **Creas un Service Principal** → Es la "identidad" de GitHub Actions
4. **Das permisos** al Service Principal para acceder al Key Vault → Autorizas el acceso
5. **Guardas las credenciales** del Service Principal en GitHub Secrets → GitHub puede autenticarse
6. **Tu workflow** se autentica en Azure y obtiene los secretos del Key Vault → ¡Funciona!

---

## 🔒 Buenas prácticas de seguridad

1. **Principio de mínimo privilegio**: Da solo el rol "Key Vault Secrets User" si solo necesitas leer
2. **Expira los secretos**: Configura fechas de expiración en los client secrets
3. **Rota credenciales**: Cambia periódicamente los client secrets
4. **No comprometas secretos**: Nunca hagas echo o log de los valores de secretos
5. **Separa ambientes**: Usa Key Vaults diferentes para desarrollo, staging y producción
6. **Configura red**: En producción, restringe el acceso por IP o red virtual
7. **Monitorea**: Activa los logs de auditoría del Key Vault para ver quién accede

---

## ❓ Preguntas frecuentes

### ¿Qué pasa si pierdo/olvido el client secret?
No se puede recuperar. Debes crear uno nuevo en "Certificates & secrets" del Service Principal.

### ¿Puedo usar el mismo Service Principal para múltiples repositorios?
Sí, pero por seguridad es mejor crear uno por repositorio o por aplicación.

### ¿Access Policies vs RBAC?
RBAC es el método moderno y recomendado. Access Policies es el método antiguo pero aún funciona.

### ¿Cómo sé si mi Key Vault usa RBAC o Access Policies?
Ve a "Access configuration" en tu Key Vault y verás el "Permission model" activo.

### ¿Puedo tener múltiples secretos con el mismo nombre?
No, cada secreto debe tener un nombre único en el Key Vault. Pero puedes tener versiones diferentes del mismo secreto.

---

## 🔗 Referencias útiles

- [Documentación oficial de Azure Key Vault](https://docs.microsoft.com/azure/key-vault/)
- [GitHub Action azure/login](https://github.com/Azure/login)
- [GitHub Action azure/get-keyvault-secrets](https://github.com/Azure/get-keyvault-secrets)
- [Roles RBAC para Key Vault](https://docs.microsoft.com/azure/key-vault/general/rbac-guide)

---

**¡Listo!** Ahora tienes todo configurado para acceder de forma segura a tus secretos de Azure Key Vault desde GitHub Actions.

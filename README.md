# Prueba de Acceso a Azure Key Vault

Este repositorio contiene un workflow de GitHub Actions para probar el acceso a Azure Key Vault.

## Configuración

### 1. Crear un Service Principal en Azure

```bash
az ad sp create-for-rbac --name "github-keyvault-test" \
  --role contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group} \
  --sdk-auth
```

Este comando generará un JSON con las credenciales. Copia todo el output.

### 2. Configurar Secrets en GitHub

En tu repositorio de GitHub, ve a **Settings > Secrets and variables > Actions** y añade los siguientes secrets:

- **AZURE_CREDENTIALS**: El JSON completo del comando anterior
- **KEY_VAULT_NAME**: El nombre de tu Azure Key Vault (ejemplo: `mi-keyvault`)
- **SECRET_NAME**: El nombre del secret que quieres leer del Key Vault (ejemplo: `test-secret`)

### 3. Dar permisos al Service Principal al Key Vault

```bash
# Obtener el Object ID del Service Principal
SP_OBJECT_ID=$(az ad sp list --display-name "github-keyvault-test" --query [0].id -o tsv)

# Dar permisos de lectura de secretos
az keyvault set-policy --name {tu-keyvault-name} \
  --object-id $SP_OBJECT_ID \
  --secret-permissions get list
```

### 4. Ejecutar el Workflow

El workflow se ejecutará automáticamente cuando:
- Hagas push a la rama `main`
- Lo ejecutes manualmente desde la pestaña **Actions** en GitHub

## Estructura del Workflow

El workflow realiza las siguientes acciones:

1. ✅ Checkout del código
2. ✅ Login en Azure usando las credenciales
3. ✅ Acceso al Key Vault y lectura de un secret
4. ✅ Verificación de que el acceso fue exitoso

## Solución de Problemas

### Error: "Caller is not authorized to perform action"

Asegúrate de que el Service Principal tiene permisos en el Key Vault:

```bash
az keyvault set-policy --name {tu-keyvault-name} \
  --spn {client-id} \
  --secret-permissions get list
```

### Error: "Secret not found"

Verifica que el secret existe en el Key Vault:

```bash
az keyvault secret list --vault-name {tu-keyvault-name}
```

## Recursos

- [Azure Login Action](https://github.com/Azure/login)
- [Azure CLI Action](https://github.com/Azure/cli)
- [Documentación Azure Key Vault](https://docs.microsoft.com/azure/key-vault/)

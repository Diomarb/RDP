# RDP
# Laboratorio GNS3 — Guía paso a paso completa
## RD RemoteApp + RD Web Client + IIS + NPS (RADIUS) + AAA en Router
Link video: https://youtu.be/IRjlGvf4GNM

## Topología y direccionamiento (tal como la armaste)

```
        SW ---- e0/0 (Router) ---- e0/1 (Router) ---- Nube VMnet4 (Windows-Client)
        |
     Nube VMnet5 (WinServer)
```
<img width="211" height="191" alt="Captura de pantalla 2026-07-11 224753" src="https://github.com/user-attachments/assets/ec0c7d92-683e-41c9-abe5-298b25586bf3" />


| Dispositivo      | Interfaz  | IP                 | Máscara           | Gateway         |
|-------------------|-----------|--------------------|--------------------|-----------------|
| Router            | e0/0      | 192.168.10.1       | 255.255.255.0      | —               |
| WinServer         | Ethernet  | 192.168.10.10      | 255.255.255.0      | 192.168.10.1    |
| Router            | e0/1      | 192.168.20.1       | 255.255.255.0      | —               |
| Windows-Client    | Ethernet  | 192.168.20.10      | 255.255.255.0      | 192.168.20.1    |

> Cambia `e0/0`/`e0/1` por el nombre real de tus interfaces si tu router en GNS3 usa otra nomenclatura (`GigabitEthernet0/0`, `FastEthernet0/0`, etc). Verifica con `show ip interface brief`.

---

# PARTE A — Direccionamiento IP básico

## A.1 IP fija en WinServer (VMnet5)

**Por GUI:**
1. `Panel de Control > Redes e Internet > Centro de redes > Cambiar configuración del adaptador`.
2. Clic derecho en el adaptador > Propiedades > `Protocolo de Internet versión 4 (TCP/IPv4)` > Propiedades.
3. IP: `192.168.10.10`, Máscara: `255.255.255.0`, Gateway: `192.168.10.1`, DNS: `192.168.10.10` (si es DC) o `127.0.0.1`.

**Por PowerShell (más rápido):**
```powershell
# Ver nombre del adaptador
Get-NetAdapter

# Asignar IP fija (ajusta "Ethernet0" al nombre real que te dio el comando anterior)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.10.10 -PrefixLength 24 -DefaultGateway 192.168.10.1

# DNS (si el server es su propio DNS/DC)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.10
```

Verifica:
```powershell
ipconfig /all
ping 192.168.10.1
```

## A.2 IP fija en Windows-Client (VMnet4)

**Por PowerShell:**
```powershell
Get-NetAdapter

New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.20.10 -PrefixLength 24 -DefaultGateway 192.168.20.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.10
```

Verifica que llegue hasta el servidor **cruzando el router**:
```powershell
ping 192.168.20.1
ping 192.168.10.10
```

Si el segundo ping falla, revisa primero el Router (parte D) antes de seguir con los servicios.

---

# PARTE B — Windows Server: todo lo que hay que crear

## B.1 (Opcional recomendado) Nombre del equipo y reinicio

```powershell
Rename-Computer -NewName "WINSRV" -Restart
```

## B.2 (Opcional) Promover a Controlador de Dominio

Si quieres usar grupos de Active Directory para el NPS (más prolijo, no obligatorio):

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -InstallDns -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd2026" -AsPlainText -Force)
```
El servidor se reiniciará solo. Si prefieres no usar dominio, usa grupos **locales** (`net localgroup`) en los pasos siguientes; el resto de la guía funciona igual.

## B.3 Crear usuarios y grupos de prueba

**Con dominio (AD):**
```powershell
New-ADUser -Name "admin15" -SamAccountName "admin15" -AccountPassword (ConvertTo-SecureString "Passw0rd15!" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "user01" -SamAccountName "user01" -AccountPassword (ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force) -Enabled $true

New-ADGroup -Name "RADIUS-Nivel15" -GroupScope Global
New-ADGroup -Name "RADIUS-Nivel1" -GroupScope Global

Add-ADGroupMember -Identity "RADIUS-Nivel15" -Members "admin15"
Add-ADGroupMember -Identity "RADIUS-Nivel1" -Members "user01"
```

**Sin dominio (grupos locales):**
```powershell
New-LocalUser -Name "admin15" -Password (ConvertTo-SecureString "Passw0rd15!" -AsPlainText -Force)
New-LocalUser -Name "user01" -Password (ConvertTo-SecureString "Passw0rd01!" -AsPlainText -Force)

New-LocalGroup -Name "RADIUS-Nivel15"
New-LocalGroup -Name "RADIUS-Nivel1"

Add-LocalGroupMember -Group "RADIUS-Nivel15" -Member "admin15"
Add-LocalGroupMember -Group "RADIUS-Nivel1" -Member "user01"
```

## B.4 Instalar Remote Desktop Services (RemoteApp)

**Opción GUI (recomendada la primera vez):**
1. `Administrador del servidor > Agregar roles y características > Instalación de Servicios de Escritorio remoto`.
2. Selecciona **Instalación basada en escenarios de inicio rápido**.
3. Tipo de implementación: **Implementación de escritorio basada en sesiones**.
4. Selecciona el propio servidor `WINSRV` para RD Connection Broker, RD Web Access y RD Session Host.
5. Marca "Reiniciar automáticamente el servidor de destino si es necesario" y da clic en **Implementar**.
6. Espera a que termine y reinicie.

**Opción PowerShell (equivalente):**
```powershell
Import-Module RemoteDesktop
New-RDSessionDeployment -ConnectionBroker "WINSRV.lab.local" -WebAccessServer "WINSRV.lab.local" -SessionHost "WINSRV.lab.local"
```

## B.5 Crear la colección de sesiones

**GUI:**
1. `Administrador del servidor > Servicios de Escritorio remoto > Colecciones > Tareas > Crear colección de sesiones`.
2. Nombre: `ColeccionLab`.
3. Selecciona el RD Session Host `WINSRV`.
4. Grupo de usuarios: agrega `RADIUS-Nivel15` y `RADIUS-Nivel1` (o "Usuarios del dominio"/"Users" si prefieres simplificar).
5. Finaliza el asistente.

**PowerShell:**
```powershell
New-RDSessionCollection -CollectionName "ColeccionLab" -SessionHost "WINSRV.lab.local" -CollectionDescription "Laboratorio RemoteApp"
```

## B.6 Publicar el RemoteApp (navegador apuntando al IIS)

**GUI:**
1. Dentro de `ColeccionLab > Programas RemoteApp > Tareas > Publicar programas RemoteApp`.
2. Selecciona **Internet Explorer** (o Microsoft Edge si tu versión de Server lo permite publicar).
3. Finaliza el asistente.
4. Clic derecho sobre el programa publicado > **Propiedades**.
5. En la pestaña de parámetros: marca **"Permitir solo esta línea de comandos"** y escribe:
   ```
   http://192.168.10.10/
   ```
6. Aplica y cierra.

**PowerShell (equivalente):**
```powershell
New-RDRemoteApp -CollectionName "ColeccionLab" -DisplayName "PaginaIIS" -FilePath "C:\Program Files\Internet Explorer\iexplore.exe" -Alias "iexplore" -CommandLineSetting Require -RequiredCommandLine "http://192.168.10.10/"
```

## B.7 Instalar y configurar IIS con página personalizada

```powershell
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Laboratorio Completado</title>
    <style>
        /* Estilos generales y colores */
        :root {
            --primary-color: #0078D4; /* Azul tecnológico */
            --bg-color: #f3f4f6;
            --card-bg: #ffffff;
            --text-main: #1f2937;
            --text-muted: #6b7280;
        }
        body {
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
            background-color: var(--bg-color);
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            color: var(--text-main);
        }
        /* Contenedor principal estilo tarjeta */
        .container {
            background-color: var(--card-bg);
            padding: 40px 50px;
            border-radius: 12px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.05);
            text-align: center;
            max-width: 500px;
            border-top: 5px solid var(--primary-color);
            animation: fadeIn 0.8s ease-in-out;
        }
        .icon {
            font-size: 50px;
            margin-bottom: 15px;
        }
        h1 {
            margin: 0 0 10px 0;
            font-size: 28px;
            font-weight: 600;
            color: var(--text-main);
        }
        p {
            margin: 0 0 25px 0;
            font-size: 16px;
            line-height: 1.6;
            color: var(--text-muted);
        }
        /* Etiqueta de estado */
        .badge {
            display: inline-block;
            background-color: #e0f2fe;
            color: #0284c7;
            padding: 6px 16px;
            border-radius: 20px;
            font-size: 14px;
            font-weight: 600;
            margin-bottom: 20px;
            letter-spacing: 0.5px;
        }
        /* Animación suave de entrada */
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(-10px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
</head>
<body>

    <div class="container">
        <div class="icon">🚀</div>
        <div class="badge">Servidor IIS Activo</div>
        <h1>¡Laboratorio Completado!</h1>
        <p>El servidor web está respondiendo correctamente. La infraestructura de red y las políticas de seguridad han sido configuradas con éxito.</p>
    </div>

</body>
</html>
```

Verifica en el propio servidor:
```powershell
Start-Process "http://localhost"
```

## B.8 Instalar y publicar el RD Web Client (HTML5)

```powershell
Install-Module -Name PowerShellGet -Force
Install-Module -Name RDWebClientManagement -Force
Install-RDWebClientPackage
Import-RDWebClientBrokerCert
Publish-RDWebClientPackage -Type Production -Latest
```

Comprueba que quedó publicado:
```powershell
Get-RDWebClientPackage
```

URLs finales:
- RD Web Access clásico: `https://192.168.10.10/RDWeb`
- RD Web Client HTML5: `https://192.168.10.10/RDWeb/webclient/index.html`

## B.9 Instalar el rol NPS (RADIUS)

```powershell
Install-WindowsFeature NPAS -IncludeManagementTools
```

Si tienes dominio, regístralo:
```powershell
netsh nps add registeredserver
```

## B.10 Agregar el Router como cliente RADIUS

**GUI:**
1. Abre la consola **NPS** (`nps.msc`).
2. `Clientes y servidores RADIUS > Clientes RADIUS > Nuevo`.
3. Nombre descriptivo: `R1`. Dirección: `192.168.10.1`. Secreto compartido: `Cisco123RADIUS`.
4. Aceptar.

**Por línea de comandos (netsh):**
```
netsh nps add client name="R1" address="192.168.10.1" secret="Cisco123RADIUS" vendorname="RADIUS Standard"
```

## B.11 Crear las Network Policies (Nivel 15 y Nivel 1)

**Política "Nivel15" (GUI):**
1. `NPS > Directivas > Directivas de red > Nueva`.
2. Nombre: `Nivel15`.
3. Tipo de servidor de acceso a la red: **Sin especificar**.
4. Condiciones: **Agregar > Grupos de Windows > RADIUS-Nivel15**.
5. Conceder el acceso.
6. Métodos de autenticación: marca **PAP, SPAP** (Cisco IOS con AAA + RADIUS usa PAP por defecto salvo que definas CHAP).
7. En **Configuración > Atributos RADIUS > Específico del proveedor**:
   - Agregar > Proveedor: **Cisco** > Atributo: **Cisco-AV-Pair**.
   - Valor: `shell:priv-lvl=15`
8. Finalizar.

**Política "Nivel1":** repite el proceso con grupo `RADIUS-Nivel1` y valor `shell:priv-lvl=1`.

9. Ordena ambas políticas por encima de "Connections to other access servers" (arriba en la lista = mayor prioridad).

Verifica con:
```powershell
Get-Service IAS   # servicio del NPS, debe estar "Running"
```

---

# PARTE C — Windows-Client: configuración y pruebas

## C.1 Verificar IP y conectividad

```powershell
ipconfig /all
ping 192.168.20.1
ping 192.168.10.10
```

## C.2 Probar RD Web Access

1. Abre el navegador: `https://192.168.10.10/RDWeb`
2. Acepta el certificado (si es autofirmado).
3. Inicia sesión con `admin15` o `user01`.
4. Haz clic en el RemoteApp publicado; debe abrir la página IIS personalizada.

## C.3 Probar RD Web Client (HTML5)

1. Navegador: `https://192.168.10.10/RDWeb/webclient/index.html`
2. Inicia sesión y ejecuta el mismo RemoteApp.

## C.4 Probar SSH al Router vía RADIUS

Desde PowerShell o PuTTY:
```
ssh admin15@192.168.20.1
```
Ingresa la contraseña del usuario (`Passw0rd15!` si seguiste el ejemplo). Debe entrar directo en modo privilegiado (`R1#`) si pertenece al grupo Nivel15.

Repite con `user01@192.168.20.1` — debe entrar en modo usuario (`R1>`).

---

# PARTE D — Router: comandos completos desde cero

## D.1 Configuración básica de interfaces (lo primero)

```
enable
configure terminal
hostname R1

interface e0/0
 description LAN_SERVIDOR
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

interface e0/1
 description LAN_CLIENTE
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit

end
write memory
```

Verifica:
```
show ip interface brief
ping 192.168.10.10
ping 192.168.20.10
```

## D.2 Usuario local (respaldo si RADIUS falla)

```
configure terminal
username admin privilege 15 secret Local123
username soporte privilege 1 secret Local456
end
write memory
```

## D.3 Contraseña de modo de configuración (enable)

```
configure terminal
enable secret ClaseConfig2026
end
write memory
```

## D.4 Habilitar SSH

```
configure terminal
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
end
write memory
```

(Al generar la llave, IOS puede preguntar `How many bits in the modulus`, confirma `2048` con Enter si no lo tomó del comando).

## D.5 AAA con RADIUS

```
configure terminal
aaa new-model

radius server RAD1
 address ipv4 192.168.10.10 auth-port 1812 acct-port 1813
 key Cisco123RADIUS
exit

aaa group server radius RADIUS-GROUP
 server name RAD1
exit

aaa authentication login default group RADIUS-GROUP local
aaa authorization exec default group RADIUS-GROUP local
aaa accounting exec default start-stop group RADIUS-GROUP

end
write memory
```

## D.6 Aplicar autenticación a las líneas VTY (SSH)

```
configure terminal
line vty 0 4
 login authentication default
 transport input ssh
exit
end
write memory
```

## D.7 Verificación y depuración (los pedidos en la consigna)

```
show aaa servers
show aaa sessions

debug aaa authentication
debug aaa authorization
debug radius
```

Con los `debug` activos, desde el Windows-Client haz `ssh admin15@192.168.20.1` y observa en la consola del router el intercambio Access-Request/Access-Accept y el atributo `Cisco-AV-Pair`.

Al terminar de revisar:
```
undebug all
```

## D.8 Guardar configuración final

```
write memory
```
o
```
copy running-config startup-config
```

---

# PARTE E — Checklist de verificación de extremo a extremo

1. `ping` entre Router y ambas VMs — OK.
2. `ping 192.168.10.10` desde Windows-Client — cruza el router — OK.
3. `https://192.168.10.10/RDWeb` abre y el RemoteApp muestra el IIS — OK.
4. `https://192.168.10.10/RDWeb/webclient/index.html` abre y el RemoteApp muestra el IIS — OK.


# Actividad Práctica Individual: Registro de Préstamo de Equipamiento en Hyperledger Fabric

**Alumno:** Facundo Arana  
**Legajo:** 96098
**Materia:** Desarrollo con Tecnologías Blockchain  
**Clase:** 04  Desarrollo con Hyperledger Fabric  
**Fecha:** 13/09/2026  

---

## 1. Entorno y prerequisitos

Se configuró un entorno Linux dentro de Windows 11 utilizando WSL2 con Ubuntu. Se instaló Node.js v18 mediante NVM y se activó la integración del socket de Docker Desktop hacia la distribución Ubuntu de WSL2.

**Comando ejecutado:**
```bash
docker --version
docker compose version
git --version
node --version
npm --version
```

**Salida obtenida:**

```text
Docker version 29.4.3, build 055a478
Docker Compose version v5.1.3
git version 2.53.0
v18.20.8
10.8.2
```

**Explicación:** Se verificó que todas las herramientas del entorno cumplan con los requerimientos mínimos de la práctica (Docker >= 24, Docker Compose v2+, Git >= 2, Node.js >= 18 y npm >= 9).

---

## 2. Descarga de Fabric

Se descargaron los binarios oficiales de Hyperledger Fabric (v2.5.16), Fabric CA (v1.5.22) y el repositorio de pruebas `fabric-samples`.

**Comandos ejecutados:**

```bash
test -d fabric-samples/test-network && echo "fabric-samples OK"
docker images | grep hyperledger
```

**Salida obtenida:**

```text
fabric-samples OK
hyperledger/fabric-ca:1.5          a70b6ba64a08        398MB          129MB
hyperledger/fabric-orderer:2.5     e322c57331d3        182MB         50.7MB
hyperledger/fabric-peer:2.5        09ee75042de9        232MB         66.8MB
hyperledger/fabric-tools:2.5       1d163b251345        930MB          255MB
```

**Explicación:** Se comprobó la existencia del directorio de automatización de la red local (`test-network`) y la disponibilidad local de las imágenes Docker oficiales de Hyperledger necesarias para inicializar la red permisionada.

---

## 3. Compilación del chaincode

Se generó el código fuente del contrato inteligente en TypeScript bajo `~/fabric-labs/prestamos-ts` utilizando la API de contratos de Fabric (`fabric-contract-api`), y se procedió con la instalación de dependencias y compilación.

**Comandos ejecutados:**

```bash
cd ~/fabric-labs/prestamos-ts
npm install
npm run build
test -f dist/index.js && echo "compilación OK"
```

**Salida obtenida:**

```text
added 82 packages, and audited 83 packages in 3s
found 0 vulnerabilities

> prestamos@1.0.0 build
> tsc

compilación OK
```

**Explicación:** El compilador de TypeScript (`tsc`) procesó el contrato `prestamos.ts` y generó el punto de entrada JavaScript en `dist/index.js` sin errores sintácticos ni de tipado.

---

## 4. Red local y canal

Se limpió cualquier estado remanente con `./network.sh down` y se levantó la red local creando el canal permisionado `equipamiento`.

**Comandos ejecutados:**

```bash
cd ~/fabric-labs/fabric-prestamos-equipamiento/fabric-samples/test-network
./network.sh down
./network.sh up createChannel -c equipamiento
docker ps --format "table {{.Names}}\t{{.Status}}"
```

**Salida obtenida:**

```text
NAMES                   STATUS
orderer.example.com     Up 21 seconds
peer0.org2.example.com  Up 21 seconds
peer0.org1.example.com  Up 21 seconds
```

**Explicación del rol de cada contenedor:**

* `peer0.org1.example.com`: Nodo peer del Laboratorio de Redes (Org1). Almacena una copia del ledger (blockchain y estado mundial) y simula/endosa las transacciones firmándolas con su clave criptográfica de organización.
* `peer0.org2.example.com`: Nodo peer del Laboratorio de Sistemas (Org2). Mantiene una réplica sincronizada del ledger y valida transacciones actuando como contraparte endosante.
* `orderer.example.com`: Nodo de ordenamiento (consenso Raft). Recibe las transacciones ya endosadas por ambos peers, las empaqueta en bloques de forma cronológica e inmutable y las distribuye a los peers del canal para su commit definitivo.

---

## 5. Despliegue del chaincode

Se desplegó el chaincode `prestamos` en el canal `equipamiento` utilizando el ciclo de vida de Fabric v2.x, requiriendo aprobación y endoso explícito de ambas organizaciones.

**Comandos ejecutados:**

```bash
CHAINCODE_PATH="$HOME/fabric-labs/prestamos-ts"

./network.sh deployCC \
  -c equipamiento \
  -ccn prestamos \
  -ccp "$CHAINCODE_PATH" \
  -ccl typescript \
  -ccep "AND('Org1MSP.peer','Org2MSP.peer')"

docker ps --format "table {{.Names}}\t{{.Status}}" | grep prestamos
```

**Salida obtenida:**

```text
Committed chaincode definition for chaincode 'prestamos' on channel 'equipamiento':
Version: 1.0, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc,
Approvals: [Org1MSP: true, Org2MSP: true]

dev-peer0.org1.example.com-prestamos_1.0-23207a01fbaf05380d56ea4587b5a2f99b984ade149078f0880f0540b3dfe54d   Up 31 seconds
dev-peer0.org2.example.com-prestamos_1.0-23207a01fbaf05380d56ea4587b5a2f99b984ade149078f0880f0540b3dfe54d   Up 31 seconds
```

**Explicación de la política de endoso:**
La política `AND('Org1MSP.peer','Org2MSP.peer')` establece a nivel de red que ninguna transacción sobre el chaincode será válida para commitearse en el ledger a menos que cuente con la firma digital (endoso) de al menos un peer de Org1 **y** de al menos un peer de Org2 tras simular la ejecución.

Si la política fuera `OR` en lugar de `AND`, bastaría con la firma de una sola organización para que el cambio sea válido en el ledger. En este caso de uso rompería la confianza distribuida: Org1 podría transferirse o asignarse equipamiento unilateralmente sin que Org2 participe ni audite la operación.

---

## 6. Invocaciones y consultas

### Configuración inicial de la CLI como Org1

```bash
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/[org1.example.com/users/Admin@org1.example.com/msp](https://org1.example.com/users/Admin@org1.example.com/msp)
export CORE_PEER_ADDRESS=localhost:7051
```

### Inicialización del ledger (`InitLedger`)

**Comando:**

```bash
peer chaincode invoke \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer.example.com \
  --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/[example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem](https://example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem)" \
  -C equipamiento \
  -n prestamos \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org2.example.com/peers/peer0.org2.example.com/tls/ca.crt](https://org2.example.com/peers/peer0.org2.example.com/tls/ca.crt)" \
  -c '{"function":"InitLedger","Args":[]}'
```

**Salida:**

```text
2026-09-13 20:09:39.898 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200

```

### Consulta de estado inicial de EQ-01

**Comando:**

```bash
for intento in 1 2 3 4 5; do
  peer chaincode query \
    -C equipamiento \
    -n prestamos \
    -c '{"Args":["ReadEquipo","EQ-01"]}' && break
  sleep 2
done
```

**Salida:**

```json
{"id":"EQ-01","responsable":"Org1","destino":"Depósito"}
```

### Transferencia de EQ-01 a Org2

**Comando:**

```bash
peer chaincode invoke \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer.example.com \
  --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/[example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem](https://example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem)" \
  -C equipamiento \
  -n prestamos \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org2.example.com/peers/peer0.org2.example.com/tls/ca.crt](https://org2.example.com/peers/peer0.org2.example.com/tls/ca.crt)" \
  -c '{"function":"TransferirEquipo","Args":["EQ-01","Org2","Laboratorio B"]}'
```

**Salida:**

```text
2026-09-13 20:10:59.064 UTC 0001 INFO [chaincodeCmd] chaincodeInvokeOrQuery -> Chaincode invoke successful. result: status:200
```

### Consulta posterior a la transferencia (desde Org1)

**Comando:**

```bash
for intento in 1 2 3 4 5; do
  RESULTADO=$(peer chaincode query \
    -C equipamiento \
    -n prestamos \
    -c '{"Args":["ReadEquipo","EQ-01"]}' 2>/dev/null || true)
  echo "$RESULTADO"
  echo "$RESULTADO" | grep "Laboratorio B" && break
  sleep 2
done
```

**Salida:**

```json
{"id":"EQ-01","responsable":"Org2","destino":"Laboratorio B"}
```

### Consulta de consistencia desde Org2

Se reconfiguraron las variables de entorno para consultar conectando al nodo `peer0.org2.example.com`:

```bash
export CORE_PEER_LOCALMSPID=Org2MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/[org2.example.com/peers/peer0.org2.example.com/tls/ca.crt](https://org2.example.com/peers/peer0.org2.example.com/tls/ca.crt)
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/[org2.example.com/users/Admin@org2.example.com/msp](https://org2.example.com/users/Admin@org2.example.com/msp)
export CORE_PEER_ADDRESS=localhost:9051

peer chaincode query \
  -C equipamiento \
  -n prestamos \
  -c '{"Args":["ReadEquipo","EQ-01"]}'
```

**Salida:**

```json
{"id":"EQ-01","responsable":"Org2","destino":"Laboratorio B"}
```

**Explicación de consistencia:** Ambos peers tienen exactamente el mismo estado porque ambos están unidos al canal `equipamiento` y reciben del servicio de ordering la misma secuencia de bloques confirmados. Al aplicar determinísticamente las escrituras validadas, sus bases de datos locales (World State) permanecen sincronizadas e idénticas.

### Prueba de autorización rechazada

Se volvió a configurar la identidad de Org1 e intentó forzarse una transferencia cuando el responsable ya era Org2:

```bash
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/[org1.example.com/users/Admin@org1.example.com/msp](https://org1.example.com/users/Admin@org1.example.com/msp)
export CORE_PEER_ADDRESS=localhost:7051

peer chaincode invoke \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer.example.com \
  --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/[example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem](https://example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem)" \
  -C equipamiento \
  -n prestamos \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)" \
  --peerAddresses localhost:9051 \
  --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/[org2.example.com/peers/peer0.org2.example.com/tls/ca.crt](https://org2.example.com/peers/peer0.org2.example.com/tls/ca.crt)" \
  -c '{"function":"TransferirEquipo","Args":["EQ-01","Org2","Laboratorio C"]}'
```

**Salida obtenida:**

```text
Error: endorsement failure during invoke. response: status:500 message:"solo Org2 puede transferir el equipo EQ-01; cliente recibido: Org1MSP"
```

**Explicación de la diferencia de fallos:**
Este rechazo se produce durante la fase de simulación dentro del peer endosante. El código del contrato inteligente evalúa la condición de negocio (`MSP_POR_ORG[equipo.responsable] !== clienteMSP`) y arroja una excepción, provocando que el peer aborte y no firme la transacción. En cambio, si la regla de negocio fuera aprobada pero fallara la política de endoso (por ejemplo, omitiendo intencionalmente la firma de Org2), la transacción llegaría al orderer o al peer validador (VSCC), siendo rechazada al momento de commitear por falta de firmas requeridas (`INVALID_ENDORSEMENT_POLICY`).

### Limpieza de la red

**Comando:**

```bash
cd ~/fabric-labs/fabric-prestamos-equipamiento/fabric-samples/test-network
./network.sh down
docker ps --format "{{.Names}}" | grep -E "peer|orderer|prestamos" || echo "red detenida OK"
```

**Salida obtenida:**

```text
Using docker and docker-compose
Stopping network
[+] down 7/7
 ✔ Container peer0.org2.example.com      Removed
 ✔ Container peer0.org1.example.com      Removed
 ✔ Container orderer.example.com          Removed
 ✔ Volume compose_peer0.org2.example.com Removed
 ✔ Network fabric_test                    Removed
 ✔ Volume compose_orderer.example.com     Removed
 ✔ Volume compose_peer0.org1.example.com Removed
Removing remaining containers
Removing generated chaincode docker images
red detenida OK
```

---

## 7. Problemas encontrados

1. **Ausencia de integración entre Docker Desktop y Ubuntu en WSL2:** Al intentar verificar los prerrequisitos, la terminal devolvió que `docker` no existía. Se resolvió ingresando a la configuración de Docker Desktop en Windows (`Settings` -> `Resources` -> `WSL Integration`) y habilitando manualmente la casilla de integración con la distribución Ubuntu.
2. **Descarga fallida del script `install-fabric.sh` (Proxy Cache 503):** La llamada directa vía `curl` a `raw.githubusercontent.com` devolvió un documento XML/HTML con error 503 generado por un servidor de caché Varnish. Se resolvió consumiendo la API REST oficial de GitHub (`api.github.com`), extrayendo el contenido mediante `jq` y decodificando el archivo base64 en Bash.
3. **Permisos denegados en el socket de Docker:** Al consultar las imágenes descargadas, el usuario de Ubuntu no tenía permisos sobre `/var/run/docker.sock`. Se solucionó incorporando el usuario al grupo `docker` mediante `sudo usermod -aG docker "$USER"` y recargando el grupo activo con `newgrp docker`.
4. **Falta de carpeta fuente en el repositorio clonado:** El directorio `chaincode/prestamos-ts` no venía provisto en el árbol raíz. Se solucionó creando la estructura de directorios y los archivos `package.json`, `tsconfig.json`, `src/index.ts` y `src/prestamos.ts` manualmente desde la terminal usando la implementación prevista en la consigna.
---

## 8. Respuestas finales

**1. ¿Qué parte de la práctica corresponde a crear la red y qué parte corresponde a desplegar el chaincode?**

La creación de la red corresponde al **Paso 3** (`./network.sh up createChannel -c equipamiento`), donde se generan los certificados criptográficos x509 de las organizaciones, se levantan los contenedores Docker de los peers y del orderer, y se inicializa el canal permisionado `equipamiento`. El despliegue del chaincode corresponde al **Paso 4** (`./network.sh deployCC ...`), en donde se ejecuta el ciclo de vida de Fabric v2.x: empaquetado del código TypeScript compilado, instalación en los nodos peer, aprobación individual por parte de cada organización (`Org1MSP` y `Org2MSP`) y la confirmación final de la definición en el canal con su política de endoso.

**2. ¿Por qué el invoke apunta a los peers de Org1 y Org2 al mismo tiempo?**

Porque la política de endoso configurada durante el despliegue es `AND('Org1MSP.peer','Org2MSP.peer')`. Para que una transacción de escritura sea aceptada en el ledger, debe contener las propuestas de lectura y escritura (*Read-Write Set*) firmadas criptográficamente por al menos un peer de cada organización. Al especificar `--peerAddresses localhost:7051` y `--peerAddresses localhost:9051`, el cliente envía la propuesta en paralelo a ambos endosantes para recopilar los dos respaldos necesarios.

**3. ¿Qué hace el orderer durante la invocación? ¿Ejecuta el contrato?**

El orderer **no** ejecuta el contrato ni procesa la lógica de negocio del chaincode. Su función dentro del modelo *Execute-Order-Validate* consiste únicamente en recibir las transacciones ya simuladas y firmadas por los peers endosantes, verificar las firmas del cliente, ordenarlas de forma cronológica y determinista en bloques, y distribuir dichos bloques a todos los peers del canal.

**4. ¿Por qué una consulta no necesita pasar por el orderer?**

Porque una consulta (`peer chaincode query`) es una operación de solo lectura que evalúa el estado actual directamente sobre el *World State* local del peer al que se conecta el cliente. Al no generar modificaciones de datos ni crear nuevas transacciones para el ledger, no requiere consenso global, ordenamiento ni distribución de bloques por parte del orderer.

**5. ¿Qué diferencia hay entre la política de endoso `AND(Org1MSP.peer, Org2MSP.peer)` y la regla de autorización implementada en `TransferirEquipo`?**

La **política de endoso** es una regla de consenso a nivel de red y protocolo que define **qué organizaciones deben atestiguar y firmar** que la ejecución fue legítima para que el bloque sea validado en la blockchain. La **regla de autorización en `TransferirEquipo**` es una validación de negocio a nivel de aplicación dentro del código del chaincode que define **quién tiene permiso para solicitar la acción**, restringiendo la operación para que solo la entidad que actualmente figura registrada como custodio del equipo pueda cederlo.



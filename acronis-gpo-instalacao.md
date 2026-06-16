# Instalacao do Futura Cyber Protection via GPO

**Cliente:** Global Logistica (global.log)  
**Data:** 16/06/2026  
**Ambiente:** Windows Server AD + GPO  
**Versao instalada:** 26.3.42183

---

## Contexto

Instalacao do agente Futura Cyber Protection (Acronis rebrandeado) via GPO de inicializacao em maquinas do dominio global.log.

---

## Arquivos necessarios

Caminho no servidor de arquivos:

```
\\SRV-FILES\GPO\antivirus\acronis\
```

Arquivos presentes na pasta:

```
FuturaCyberProtection.exe   <- instalador principal (20MB)
BackupClient64.msi          <- MSI extraido pelo .exe
BackupClient64.msi.mst      <- arquivo de transformacao com token de registro
bc6410.cab ... bc6449.cab   <- arquivos de cabine do instalador
```

> O `FuturaCyberProtection.exe` ja contem o token de registro no console Acronis embutido.
> Nao e necessario referenciar o `.msi` ou `.mst` separadamente ao usar o `.exe`.

---

## Metodo utilizado

**Script de Inicializacao via GPO** (Configuracao do Computador).

> Metodo via Software Installation com `.msi` + `.mst` foi testado e resultou em erro 1603
> (falha fatal do Windows Installer). O `.exe` com parametro `--quiet` foi o metodo que funcionou.

---

## Parametros do instalador

O `FuturaCyberProtection.exe` usa sintaxe com duplo traco (`--`), nao barra (`/`):

| Parametro | Funcao |
|---|---|
| `--quiet` | Instalacao silenciosa sem interface grafica |
| `--help` | Exibe todos os parametros disponiveis |
| `--log-dir=<path>` | Salva log de instalacao no caminho especificado |
| `--language=<code>` | Define idioma da instalacao |

---

## Configuracao da GPO

### Caminho no GPMC

```
Editor de GPO > Acronis Antivirus
  > Configuracao do Computador
    > Configuracoes do Windows
      > Scripts (Inicializacao/Encerramento)
        > Inicializacao
```

### Parametros do script

| Campo | Valor |
|---|---|
| Nome do Script | `\\SRV-FILES\GPO\antivirus\acronis\FuturaCyberProtection.exe` |
| Parametros | `--quiet` |

---

## Aplicacao na maquina cliente

Apos configurar a GPO, na maquina cliente:

```cmd
gpupdate /force
```

Reiniciar a maquina. O script roda no boot antes do logon.

---

## Validacao pos-instalacao

### 1. Verificar processos ativos

```cmd
tasklist | findstr /i "futura acronis cyber"
```

Resultado esperado:

```
FuturaCyberProtection.exe    XXXX  Services  0  XX.XXX K
cyber-desktop-service.exe    XXXX  Services  0  XX.XXX K
cyber-scripting-executor.    XXXX  Services  0  XX.XXX K
```

### 2. Verificar em Programas e Recursos

```
Painel de Controle > Programas e Recursos
```

Deve aparecer: **Futura Cyber Protection** com editor Acronis.

### 3. Verificar aplicacao da GPO

```cmd
gpresult /h c:\gpo.html && start c:\gpo.html
```

Conferir:
- Scripts: **Exito**
- Software Installation: **Exito**
- Nenhum Erro Detectado

### 4. Verificar registro no console Acronis

Acessar o painel web do Acronis e confirmar se a maquina aparece como dispositivo registrado na conta do cliente.

---

## Erros encontrados durante o processo

### Erro 1 - Parametro invalido `/quiet`

**Sintoma:** Janela do instalador abria com mensagem de parametro nao reconhecido.

**Causa:** O `FuturaCyberProtection.exe` usa `--quiet` (duplo traco), nao `/quiet` (barra).

**Solucao:** Trocar parametro para `--quiet`.

---

### Erro 2 - Software Installation com erro 1603

**Sintoma:** GPO com Software Installation apontando para `BackupClient64.msi` + `.mst` resultava em falha fatal apos 1h50min.

**Log de evento:**
```
Event ID: 1033 - MsiInstaller
Status de erro ou exito da instalacao: 1603
```

**Causa provavel:** Timeout ou incompatibilidade do `.mst` com o contexto de instalacao via GPO.

**Solucao:** Abandonar metodo via Software Installation e usar Script de Inicializacao com o `.exe`.

---

### Erro 3 - Nome do servico diferente do esperado

**Sintoma:** `sc query "Acronis Cyber Protect Agent"` retornava FALHA 1060.

**Causa:** O produto e rebrandeado como Futura Cyber Protection, nao Acronis.

**Validacao correta:**

```cmd
tasklist | findstr /i "futura acronis cyber"
```

# Instalacao do Agente Futura Cyber Protection (Acronis) via GPO

**Cliente:** Global Logistica (global.log)
**Objetivo:** Implantar o agente Futura Cyber Protection (Acronis rebrandeado) automaticamente nas maquinas do dominio via GPO.
**Servidor AD:** SRV-AD.global.log
**Maquina de teste:** PC-GPO (Windows 11, build 26200)

---

## 1. Descobertas importantes

Pontos que custaram tempo e que evitam refazer o diagnostico do zero:

- O `.msi` (`BackupClient64.msi`) sozinho **nao instala** - da erro **1603**. Ele depende do `.exe` (`FuturaCyberProtection.exe`), que aplica permissoes e o token de registro.
- O `.exe` usa parametro com **duplo traco**: `--quiet` (e **nao** `/quiet`).
- **Software Installation** via GPO (atribuir o MSI) falha com **1085/1603**. Descartado.
- GPO de **Script** aceita `.exe` direto, mas o parametro vai no campo separado **"Parametros de Script"**, nunca junto ao nome do arquivo.
- Sem `--quiet`, o instalador roda em **modo interativo** e abre a janela na tela do usuario.
- O **caminho do script tem que ser exato**. Bug real encontrado: o gatilho estava registrado como `\\Srv-files\gpo\antivirus\instala_acronis.bat`, faltando a subpasta `\acronis\`. No boot/logon o Windows nao achava o arquivo e nao rodava nada.
- A instalacao em contexto **SYSTEM / Sessao 0** funciona 100% (provado via PsExec, retorno **code 0**).
- A mensagem `Assertion failed ... com_library_holder.cpp, line 19` que aparece **depois** do `code 0` e erro **cosmetico** de limpeza de COM na saida do instalador. A instalacao ja foi concluida. Pode ignorar.
- Resquicio de **Acronis legado** (`HKLM\SOFTWARE\Acronis\BackupAndRecovery`) pode atrapalhar a instalacao. Limpar antes de testar em maquina "limpa".
- **Windows 11 + Fast Logon Optimization:** script de Inicializacao recem-configurado pode so rodar no **2o boot**. A GPO "Sempre aguardar a rede na inicializacao e no logon do computador" desativa esse comportamento e faz rodar ja no 1o boot.

---

## 2. Arquivos no compartilhamento

| Arquivo | Caminho |
|---|---|
| Instalador | `\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe` |
| MSI (nao usado sozinho) | `\\Srv-files\gpo\antivirus\acronis\BackupClient64.msi` |
| Script (.bat) | `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat` |
| Log de teste | `C:\Windows\Temp\instala_acronis_logon.log` |

A conta de computador (`MAQUINA$`) precisa de **Leitura** no compartilhamento e no NTFS. O grupo "Usuarios autenticados" ja inclui as contas de computador, entao normalmente nao precisa ajuste.

---

## 3. Solucao que funcionou neste ambiente (Logon de usuario)

GPO **Acronis Antivirus** > **Configuracao do Usuario** > Politicas > Configuracoes do Windows > **Scripts (Logon/Logoff)** > **Logon** > aba **Scripts** > Adicionar:

- **Nome do Script:** `\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe`
- **Parametros de Script:** `--quiet`

Aplicar (OK / Aplicar).

> **Importante:** essa via funcionou porque o usuario de teste (`talles.geovani`) e **Administrador do Dominio**. O logon roda no contexto do usuario; a instalacao cria servico, escreve no registro e aplica token, o que **exige admin**. Para **usuario comum (nao-admin)**, esta via pode falhar. Ver secao 5 (via recomendada para producao).

---

## 4. Validacao

Depois de aplicar:

```cmd
gpupdate /force
```

- [ ] Deslogar o usuario
- [ ] Logar novamente
- [ ] Aguardar de **2 a 3 minutos** (o `--quiet` instala em background; o usuario ja fica no desktop)
- [ ] Rodar as verificacoes abaixo

```cmd
sc query state= all | findstr /i "AcronisAgent cyber-desktop futura"
```

```cmd
tasklist | findstr /i "futura cyber acronis"
```

- [ ] Confirmar em **Painel de Controle > Programas e Recursos** que "Futura Cyber Protection" esta listado

Confirmar que a GPO chegou no usuario:

```cmd
gpresult /r /scope:user
```
Procurar "Acronis Antivirus" em **"Objetos de politica de grupo aplicados"**.

Relatorio completo em HTML (mais facil de ler):

```cmd
gpresult /h C:\Windows\Temp\rsop.html /f
start C:\Windows\Temp\rsop.html
```
Ir em **Configuracoes do Usuario > Scripts** e confirmar o `.exe` com `--quiet` no Logon.

---

## 5. Via recomendada para producao (Inicializacao / SYSTEM)

Mais robusta porque roda como **SYSTEM** (admin local pleno), **nao depende do privilegio do usuario logado** e funciona em qualquer maquina, mesmo antes do logon. Foi **provada** funcionando via PsExec (code 0).

### 5.1 Script com log e verificacao de "ja instalado"

Salvar em `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat`:

```bat
@echo off
set LOG=C:\Windows\Temp\instala_acronis.log
set ORIGEM=\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe

echo ================================>> "%LOG%"
echo %date% %time% - Inicio>> "%LOG%"

sc query state= all | findstr /i "AcronisAgent cyber-desktop" >nul 2>&1
if %errorlevel%==0 (
    echo %date% %time% - Ja instalado, saindo>> "%LOG%"
    exit /b 0
)

set /a N=0
:check
if exist "%ORIGEM%" goto instalar
set /a N+=1
if %N% geq 20 (
    echo %date% %time% - Share inacessivel, abortando>> "%LOG%"
    exit /b 1
)
timeout /t 5 /nobreak >nul
goto check

:instalar
echo %date% %time% - Instalando...>> "%LOG%"
"%ORIGEM%" --quiet
echo %date% %time% - Instalador retornou %errorlevel%>> "%LOG%"
exit /b 0
```

Notas do script:
- O check de "ja instalado" usa o **servico**, nao o codigo de saida - por isso a assertion cosmetica do instalador nao atrapalha.
- O loop espera o **arquivo** no compartilhamento (nao so ping), com limite para nao travar o boot.
- Grava log em `C:\Windows\Temp\instala_acronis.log` - fonte de verdade para saber se rodou.

### 5.2 Configuracao da GPO

GPO **Acronis Antivirus** > **Configuracao do Computador** > Politicas > Configuracoes do Windows > **Scripts (Inicializacao/Encerramento)** > **Inicializacao** > aba **Scripts** > Adicionar:

- **Nome do Script:** `\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat`
- **Parametros:** vazio

Para rodar ja no 1o boot (desativa Fast Logon Optimization do Win11):

GPO > **Configuracao do Computador** > Modelos Administrativos > Sistema > Logon > **"Sempre aguardar a rede na inicializacao e no logon do computador"** > **Habilitado**.

(Opcional) Para o boot nao esperar a instalacao terminar antes da tela de login:
Modelos Administrativos > Sistema > Scripts > **"Executar scripts de inicializacao de modo assincrono"** > **Habilitado**. O boot vai direto para o login e a instalacao corre em background.

### 5.3 Validacao da via de Inicializacao

```cmd
gpupdate /force
shutdown /r /t 0
```

Apos o boot:

```cmd
type C:\Windows\Temp\instala_acronis.log
sc query state= all | findstr /i "AcronisAgent cyber-desktop futura"
```

Confirmar que o gatilho disparou (o `ExecTime` deixa de ser `0x0`):

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup\0\0" /v ExecTime
```

---

## 6. Comandos uteis de diagnostico

Testar a instalacao como **SYSTEM** (Sessao 0, exatamente como o boot faz), sem reiniciar:

```cmd
PsExec64.exe -s -accepteula cmd
whoami
"\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe" --quiet
```
`whoami` tem que mostrar `nt authority\system`.

Ver scripts registrados na maquina:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup" /s
reg query "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Logon" /s
```
(Script de **computador** fica em HKLM\...\Startup; script de **usuario** fica em HKCU\...\Logon.)

Verificar se a maquina enxerga o compartilhamento como conta de maquina (rodar dentro do shell SYSTEM do PsExec):

```cmd
dir \\Srv-files\gpo\antivirus\acronis\
```

Verificar resquicio de Acronis legado:

```cmd
reg query "HKLM\SOFTWARE\Acronis" /s
sc query state= all | findstr /i "acronis"
```

Remover resquicio de registro (somente se nao houver instalacao real, so chave orfa):

```cmd
reg delete "HKLM\SOFTWARE\Acronis" /f
```

---

## 7. Observacoes e cuidados

- **Admin vs usuario comum:** a via de Logon so e garantida se o usuario for admin local. Testar logado como admin da **falso positivo** - pode parecer que funciona e quebrar nas maquinas dos funcionarios comuns. A via de Inicializacao (SYSTEM) nao tem esse risco.
- **GPOs duplicadas:** existiam 3 GPOs de Acronis conflitando (Acronis Antivirus, Deploy Acronis, GPO - Acronis Antivirus). Mantida apenas **Acronis Antivirus**. Conferir periodicamente se nao voltaram duplicatas.
- **Sem MSI atribuido:** garantir que **Configuracao do Computador > Configuracoes de Software > Instalacao de software** esteja **vazio** na GPO, senao o MSI quebrado tenta instalar em paralelo e gera 1603.
- **Filtro WMI:** a GPO "Acronis Antivirus" nao tem filtro WMI (confere em GPO > aba Escopo > Filtragem WMI = "nenhum"). Se um dia for adicionado um filtro, conferir se a maquina-alvo casa com a consulta.

---

## 8. Status final

- [x] Instalador `.exe --quiet` funciona manualmente (100%)
- [x] Instalacao como SYSTEM via PsExec (code 0, aparece em Programas e Recursos)
- [x] Caminho do script corrigido (incluida a subpasta `\acronis\`)
- [x] GPOs duplicadas removidas (mantida so "Acronis Antivirus")
- [x] Resquicio de Acronis legado limpo
- [x] **Logon de usuario com `--quiet` instalou automaticamente** (ambiente de teste, usuario admin)
- [ ] Validar com **usuario nao-admin** (definir se vai para producao via Logon ou via Inicializacao)
- [ ] Implantar a via escolhida em todas as maquinas do dominio

---

## 9. Resumo rapido (cola de implantacao)

**Logon (funcionou com admin):**
```
Config. Usuario > Scripts (Logon) > Logon
Nome:       \\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe
Parametros: --quiet
```

**Inicializacao (recomendado para producao):**
```
Config. Computador > Scripts (Inicializacao) > Inicializacao
Nome:       \\Srv-files\gpo\antivirus\acronis\instala_acronis.bat
Parametros: (vazio)
+ Habilitar "Sempre aguardar a rede na inicializacao e no logon do computador"
```

**Validar:**
```cmd
gpupdate /force
sc query state= all | findstr /i "AcronisAgent cyber-desktop futura"
```

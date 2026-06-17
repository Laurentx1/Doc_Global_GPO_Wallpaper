# Instalacao do Agente Milvus via GPO

**Cliente:** Global Logistica (global.log)
**Objetivo:** Instalar o agente de monitoramento Milvus automaticamente nas maquinas do dominio via GPO.
**GPO:** GPO - Milvus Monitoramento
**Produto instalado:** Agente Milvus Core

---

## 1. Resumo

Mesma logica usada no Acronis: GPO de **Script de Inicializacao** (Configuracao do Computador) chama um `.bat` que executa o instalador do Milvus em modo silencioso (`--quiet`). Rodar como Inicializacao garante contexto **SYSTEM** (admin local pleno), independente do usuario logado, e funciona ja antes do logon.

---

## 2. Arquivos no compartilhamento

| Arquivo | Caminho |
|---|---|
| Script (.bat) | `\\Srv-files\gpo\monitoramento\Milvus\Instalar_Milvus.bat` |
| Instalador (.exe) | `\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe` |

> Ajustar `<NOME_DO_INSTALADOR>` para o nome real do executavel que esta na pasta `Milvus`.

---

## 3. Script

### Versao basica (a que foi feita)
```bat
@echo off
"\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe" --quiet
exit /b 0
```

### Versao recomendada (com log e verificacao de ja instalado)
```bat
@echo off
set LOG=C:\Windows\Temp\instala_milvus.log
set ORIGEM=\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe

echo ================================>> "%LOG%"
echo %date% %time% - Inicio>> "%LOG%"

sc query state= all | findstr /i "milvus" >nul 2>&1
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

> Confirmar o nome real do servico do Milvus com `sc query state= all | findstr /i milvus` e ajustar o filtro do check, se necessario.

---

## 4. Configuracao da GPO

GPO **GPO - Milvus Monitoramento** > **Configuracao do Computador** > Politicas > Configuracoes do Windows > **Scripts (Inicializacao/Encerramento)** > **Inicializacao** > aba **Scripts** > Adicionar:

- **Nome do Script:** `\\Srv-files\gpo\monitoramento\Milvus\Instalar_Milvus.bat`
- **Parametros:** vazio

---

## 5. Validacao

```cmd
gpupdate /force
shutdown /r /t 0
```

Apos o boot:

- [ ] Conferir em **Programas e Recursos** que "Agente Milvus Core" esta listado
```cmd
sc query state= all | findstr /i "milvus"
```
```cmd
tasklist | findstr /i "milvus"
```

Confirmar que o gatilho disparou:
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup" /s
```
(procurar a entrada com `Instalar_Milvus.bat`; o `ExecTime` deixa de ser `0x0` apos rodar.)

---

## 6. Observacoes

- Em Windows 11, por causa do Fast Logon Optimization, o script de Inicializacao recem-configurado pode so rodar no 2o boot. Habilitar **"Sempre aguardar a rede na inicializacao e no logon do computador"** (Config. Computador > Modelos Administrativos > Sistema > Logon) para rodar ja no 1o boot.
- A conta de computador (`MAQUINA$`) precisa de **Leitura** no compartilhamento `\\Srv-files\gpo\monitoramento\Milvus\`.

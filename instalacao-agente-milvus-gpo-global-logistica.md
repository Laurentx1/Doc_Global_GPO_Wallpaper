# Instalação do Agente Milvus via GPO

**Cliente:** Global Logística (global.log)
**Objetivo:** Instalar o agente de monitoramento Milvus automaticamente nas máquinas do domínio via GPO.
**GPO:** GPO - Milvus Monitoramento
**Produto instalado:** Agente Milvus Core

---

## 1. Resumo

Mesma lógica usada no Acronis: GPO de **Script de Inicialização** (Configuração do Computador) chama um `.bat` que executa o instalador do Milvus em modo silencioso (`--quiet`). Rodar como Inicialização garante contexto **SYSTEM** (admin local pleno), independente do usuário logado, e funciona já antes do logon.

---

## 2. Arquivos no compartilhamento

| Arquivo | Caminho |
|---|---|
| Script (.bat) | `\\Srv-files\gpo\monitoramento\Milvus\Instalar_Milvus.bat` |
| Instalador (.exe) | `\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe` |

> Ajustar `<NOME_DO_INSTALADOR>` para o nome real do executável que está na pasta `Milvus`.

---

## 3. Script

### Versão básica (a que foi feita)
```bat
@echo off
"\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe" --quiet
exit /b 0
```

### Versão recomendada (com log e verificação de já instalado)
```bat
@echo off
set LOG=C:\Windows\Temp\instala_milvus.log
set ORIGEM=\\Srv-files\gpo\monitoramento\Milvus\<NOME_DO_INSTALADOR>.exe

echo ================================>> "%LOG%"
echo %date% %time% - Início>> "%LOG%"

sc query state= all | findstr /i "milvus" >nul 2>&1
if %errorlevel%==0 (
    echo %date% %time% - Já instalado, saindo>> "%LOG%"
    exit /b 0
)

set /a N=0
:check
if exist "%ORIGEM%" goto instalar
set /a N+=1
if %N% geq 20 (
    echo %date% %time% - Share inacessível, abortando>> "%LOG%"
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

> Confirmar o nome real do serviço do Milvus com `sc query state= all | findstr /i milvus` e ajustar o filtro do check, se necessário.

---

## 4. Configuração da GPO

GPO **GPO - Milvus Monitoramento** > **Configuração do Computador** > Políticas > Configurações do Windows > **Scripts (Inicialização/Encerramento)** > **Inicialização** > aba **Scripts** > Adicionar:

- **Nome do Script:** `\\Srv-files\gpo\monitoramento\Milvus\Instalar_Milvus.bat`
- **Parâmetros:** vazio

---

## 5. Validação

```cmd
gpupdate /force
shutdown /r /t 0
```

Após o boot:

- [ ] Conferir em **Programas e Recursos** que "Agente Milvus Core" está listado
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
(procurar a entrada com `Instalar_Milvus.bat`; o `ExecTime` deixa de ser `0x0` após rodar.)

---

## 6. Observações

- Em Windows 11, por causa do Fast Logon Optimization, o script de Inicialização recém-configurado pode só rodar no 2º boot. Habilitar **"Sempre aguardar a rede na inicialização e no logon do computador"** (Config. Computador > Modelos Administrativos > Sistema > Logon) para rodar já no 1º boot.
- A conta de computador (`MAQUINA$`) precisa de **Leitura** no compartilhamento `\\Srv-files\gpo\monitoramento\Milvus\`.

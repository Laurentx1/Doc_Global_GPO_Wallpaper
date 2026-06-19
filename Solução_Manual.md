# IMPORTANTE : NÃO USADO MAIS DESSA FORMA
# Solução de Problemas - Tela de Bloqueio Não Aplica

Se uma máquina específica não aplicar a tela de bloqueio pelo ciclo normal da GPO,
execute os comandos abaixo diretamente nela (CMD ou PowerShell como administrador):

## 1. Gravar as chaves de registro manualmente

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImagePath /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageUrl /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageStatus /t REG_DWORD /d 1 /f
```

## 2. Confirmar que as chaves foram gravadas

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP"
```

## 3. Confirmar que o arquivo existe na máquina

```cmd
dir C:\bg\
```

## 4. Testar a tela de bloqueio

Pressione **Win + L** e verifique se a arte correta está sendo exibida.

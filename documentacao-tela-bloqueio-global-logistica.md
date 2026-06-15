# Correcao da Tela de Bloqueio (Lock Screen) via GPO - Global Logistica

## Informacoes do Chamado

| Campo | Valor |
|---|---|
| Cliente | Global Logistica |
| Dominio | global.log |
| Chamados | 0626-000345 e 0626-000347 |
| Tecnico responsavel | Talles Geovani |
| Data | 15/06/2026 |
| GPO afetada | Tela de Bloqueio e Plano de Fundo |
| GUID da GPO | {06047A9B-B307-4DFC-8D97-7716ABEBEF9B} |
| Vinculo da GPO | global.log (raiz do dominio) |
| Sistema operacional das estacoes | Windows 11 Pro (build 26100) |

---

## 1. Problema Relatado

Em todo o parque de maquinas da Global Logistica, o plano de fundo da area de trabalho (wallpaper) era trocado normalmente pela GPO, porem a imagem da tela de bloqueio (lock screen) nao era aplicada em nenhuma maquina, tanto nas antigas quanto nas recem-adicionadas ao Active Directory.

Nao havia mensagem de erro na tela das estacoes. A tela de bloqueio simplesmente nao trocava para a arte definida pela equipe.

Observacao do cliente: a arte da tela de bloqueio e trocada mensalmente e e usada pelo RH como canal de informacoes basicas, portanto a funcao e considerada essencial.

---

## 2. Diagnostico Realizado

Foram verificados todos os pontos da configuracao, e todos estavam corretos:

1. A GPO estava habilitada e vinculada a raiz do dominio (global.log), alcancando todas as maquinas.
2. O filtro de seguranca estava como Usuarios Autenticados (correto).
3. O relatorio `gpresult /h` confirmou que a GPO Tela de Bloqueio e Plano de Fundo estava em GPOs Aplicadas, no escopo do computador.
4. O arquivo da arte existia localmente na maquina em `C:\bg\lockScreen.jpg`, com data recente (arte nova).
5. A chave de registro `HKLM\SOFTWARE\Policies\Microsoft\Windows\Personalization\LockScreenImage` estava gravada apontando para `C:\bg\lockScreen.jpg`.
6. A chave `LockScreenOverlaysDisabled` estava com valor 1 (Windows Spotlight desativado).
7. A maquina foi reiniciada e mesmo assim a arte nao foi aplicada.

Conclusao do diagnostico: a GPO estava chegando, sendo processada e gravando o registro corretamente. O problema nao era permissao, nao era a GPO e nao era o arquivo. Por isso, refazer a GPO do zero nao resolveria.

---

## 3. Causa Raiz

A politica utilizada era:

```
Configuracao do Computador
  > Politicas
    > Modelos Administrativos
      > Painel de Controle
        > Personalizacao
          > Forcar uma imagem padrao especifica de tela de bloqueio e de logon
```

Essa politica funciona apenas nas edicoes Enterprise, Education e Server do Windows. Em maquinas Windows 11 Pro, ela ate grava a chave de registro e aparece como aplicada no gpresult, mas o sistema ignora essa configuracao na hora de renderizar a tela de bloqueio.

Como todo o parque da Global Logistica utiliza Windows 11 Pro, a tela de bloqueio nunca era aplicada. O plano de fundo da area de trabalho continuava funcionando porque utiliza um mecanismo diferente, que o Windows Pro respeita.

Resumo da causa raiz: limitacao de edicao do Windows. A politica de tela de bloqueio por Modelos Administrativos nao e imposta em edicoes Pro.

---

## 4. Solucao Aplicada

A solucao foi substituir o metodo de aplicacao da imagem por chaves de registro do PersonalizationCSP, que funcionam em todas as edicoes do Windows, incluindo a Pro. O arquivo da arte continua sendo copiado do servidor para a pasta local, garantindo funcionamento tambem fora da rede da empresa.

A configuracao foi feita dentro da propria GPO Tela de Bloqueio e Plano de Fundo, em tres partes.

### 4.1. Copia do arquivo (ja existente, mantida)

```
Configuracao do Computador
  > Preferencias
    > Configuracoes do Windows
      > Arquivos
```

| Acao | Origem | Destino |
|---|---|---|
| Substituir | \\Srv-files\gpo\papeldeparede\lockScreen.jpg | C:\bg\lockScreen.jpg |
| Substituir | \\Srv-files\gpo\papeldeparede\backgroundScreen.jpg | C:\bg\backgroundScreen.jpg |

A acao Substituir garante que, quando a arte for trocada no servidor, a maquina recebe a versao nova no proximo ciclo de atualizacao da GPO.

### 4.2. Criacao da pasta (ja existente, mantida)

```
Configuracao do Computador
  > Preferencias
    > Configuracoes do Windows
      > Pastas
```

| Acao | Caminho |
|---|---|
| Atualizar | C:\bg\ |

### 4.3. Chaves de registro do PersonalizationCSP (novo, o que resolveu)

```
Configuracao do Computador
  > Preferencias
    > Configuracoes do Windows
      > Registro
```

Foram criados tres itens de registro, todos com Acao Atualizar, Hive HKEY_LOCAL_MACHINE, no caminho:

```
SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP
```

| Nome do Valor | Tipo | Dado |
|---|---|---|
| LockScreenImagePath | REG_SZ | C:\bg\lockScreen.jpg |
| LockScreenImageUrl | REG_SZ | C:\bg\lockScreen.jpg |
| LockScreenImageStatus | REG_DWORD | 1 |

### 4.4. Politicas mantidas habilitadas

As politicas abaixo foram mantidas, pois impedem que o usuario altere a imagem manualmente:

- Impedir alteracao de imagem da tela de bloqueio e de logon (Habilitado)
- Impedir alteracao de plano de fundo do menu Iniciar (Habilitado)

A politica Forcar uma imagem padrao especifica de tela de bloqueio pode ser mantida habilitada sem prejuizo, mas quem efetivamente aplica a imagem no Windows Pro sao as chaves do PersonalizationCSP.

---

## 5. Validacao na Maquina de Teste

Antes de confiar apenas na GPO, as chaves foram aplicadas manualmente em uma estacao Windows 11 Pro para confirmar que o metodo funciona. Comandos executados no CMD como administrador:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImagePath /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageUrl /t REG_SZ /d "C:\bg\lockScreen.jpg" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP" /v LockScreenImageStatus /t REG_DWORD /d 1 /f
```

Em seguida, a tela foi bloqueada com a combinacao de teclas Windows + L. A arte correta foi exibida com sucesso, confirmando a solucao.

Para conferir as chaves gravadas em qualquer maquina:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP"
```

---

## 6. Como Propagar para Todo o Parque

A GPO se propaga sozinha. Cada maquina busca a atualizacao automaticamente em duas situacoes:

1. Na inicializacao (boot) da maquina.
2. No ciclo automatico de atualizacao da Politica de Grupo, a cada 90 minutos aproximadamente.

As maquinas que estiverem desligadas recebem a configuracao no proximo boot.

### Opcoes para acelerar a aplicacao (opcional)

Forcar a atualizacao em uma maquina especifica (CMD como administrador na propria maquina):

```cmd
gpupdate /force
```

Forcar a atualizacao em todas as maquinas de uma Unidade Organizacional, pelo Console de Gerenciamento de Politica de Grupo (gpmc.msc): clicar com o botao direito na OU COMPUTADORES e selecionar Atualizacao de Politica de Grupo.

Forcar a atualizacao em todas as maquinas do dominio via PowerShell no servidor (como administrador):

```powershell
Get-ADComputer -Filter * | ForEach-Object { Invoke-GPUpdate -Computer $_.Name -Force -RandomDelayInMinutes 0 }
```

Observacao: as opcoes de atualizacao remota dependem de o firewall das maquinas estar liberado para gerenciamento remoto (regras de Tarefas Agendadas Remotas e WMI) e so atingem maquinas ligadas no momento. O metodo via PowerShell pode demorar, pois aguarda o tempo limite de cada maquina desligada antes de seguir para a proxima.

---

## 7. Manutencao Mensal da Arte

Para trocar a arte da tela de bloqueio nos meses seguintes:

1. Substituir o arquivo no servidor em `\\Srv-files\gpo\papeldeparede\lockScreen.jpg`, mantendo exatamente o mesmo nome de arquivo.
2. Nao e necessario alterar a GPO.
3. Como a acao de copia esta como Substituir, cada maquina copia a versao nova no proximo ciclo de atualizacao da GPO e a tela de bloqueio passa a exibir a arte atualizada.

---

## 8. Checklist de Verificacao

- [ ] GPO Tela de Bloqueio e Plano de Fundo habilitada e vinculada a global.log
- [ ] Preferencia de Pasta criando C:\bg\ (Acao Atualizar)
- [ ] Preferencia de Arquivo copiando lockScreen.jpg do servidor para C:\bg\ (Acao Substituir)
- [ ] Preferencia de Arquivo copiando backgroundScreen.jpg do servidor para C:\bg\ (Acao Substituir)
- [ ] Item de Registro LockScreenImagePath (REG_SZ) = C:\bg\lockScreen.jpg
- [ ] Item de Registro LockScreenImageUrl (REG_SZ) = C:\bg\lockScreen.jpg
- [ ] Item de Registro LockScreenImageStatus (REG_DWORD) = 1
- [ ] Politicas de impedir alteracao mantidas habilitadas
- [ ] Arquivo lockScreen.jpg presente em C:\bg\ na maquina de teste
- [ ] Chaves do PersonalizationCSP confirmadas com reg query na maquina de teste
- [ ] Tela de bloqueio exibindo a arte correta apos Windows + L
- [ ] Validado em pelo menos uma estacao que recebeu a configuracao via GPO (sem o reg add manual)

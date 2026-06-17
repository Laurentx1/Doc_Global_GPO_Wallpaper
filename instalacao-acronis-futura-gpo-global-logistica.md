Instalação do Agente Futura Cyber Protection (Acronis) via GPO

Cliente: Global Logística (global.log)
Objetivo: Implantar o agente Futura Cyber Protection (Acronis rebrandado) automaticamente nas máquinas do domínio via GPO.
Servidor AD: SRV-AD.global.log
Máquina de teste: PC-GPO (Windows 11, build 26200)

1. Descobertas importantes

Pontos que custaram tempo e que evitam refazer o diagnóstico do zero:

O .msi (BackupClient64.msi) sozinho não instala - dá erro 1603. Ele depende do .exe (FuturaCyberProtection.exe), que aplica permissões e o token de registro.
O .exe usa parâmetro com duplo traço: --quiet (e não /quiet).
Software Installation via GPO (atribuir o MSI) falha com 1085/1603. Descartado.
GPO de Script aceita .exe direto, mas o parâmetro vai no campo separado "Parâmetros de Script", nunca junto ao nome do arquivo.
Sem --quiet, o instalador roda em modo interativo e abre a janela na tela do usuário.
O caminho do script tem que ser exato. Bug real encontrado: o gatilho estava registrado como \\Srv-files\gpo\antivirus\instala_acronis.bat, faltando a subpasta \acronis\. No boot/logon o Windows não achava o arquivo e não rodava nada.
A instalação em contexto SYSTEM / Sessão 0 funciona 100% (provado via PsExec, retorno code 0).
A mensagem Assertion failed ... com_library_holder.cpp, line 19 que aparece depois do code 0 é erro cosmético de limpeza de COM na saída do instalador. A instalação já foi concluída. Pode ignorar.
Resquício de Acronis legado (HKLM\SOFTWARE\Acronis\BackupAndRecovery) pode atrapalhar a instalação. Limpar antes de testar em máquina "limpa".
Windows 11 + Fast Logon Optimization: script de Inicialização recém-configurado pode só rodar no 2º boot. A GPO "Sempre aguardar a rede na inicialização e no logon do computador" desativa esse comportamento e faz rodar já no 1º boot.
2. Arquivos no compartilhamento
Arquivo	Caminho
Instalador	\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe
MSI (não usado sozinho)	\\Srv-files\gpo\antivirus\acronis\BackupClient64.msi
Script (.bat)	\\Srv-files\gpo\antivirus\acronis\instala_acronis.bat
Log de teste	C:\Windows\Temp\instala_acronis_logon.log

A conta de computador (MAQUINA$) precisa de Leitura no compartilhamento e no NTFS. O grupo "Usuários autenticados" já inclui as contas de computador, então normalmente não precisa de ajuste.

3. Solução que funcionou neste ambiente (Logon de usuário)

GPO Acronis Antivirus > Configuração do Usuário > Políticas > Configurações do Windows > Scripts (Logon/Logoff) > Logon > aba Scripts > Adicionar:

Nome do Script: \\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe
Parâmetros de Script: --quiet

Aplicar (OK / Aplicar).

Importante: essa via funcionou porque o usuário de teste (talles.geovani) é Administrador do Domínio. O logon roda no contexto do usuário; a instalação cria serviço, escreve no registro e aplica token, o que exige admin. Para usuário comum (não-admin), esta via pode falhar. Ver seção 5 (via recomendada para produção).

4. Validação

Depois de aplicar:

gpupdate /force
 Deslogar o usuário
 Logar novamente
 Aguardar de 2 a 3 minutos (o --quiet instala em background; o usuário já fica no desktop)
 Rodar as verificações abaixo
sc query state= all | findstr /i "AcronisAgent cyber-desktop futura"
tasklist | findstr /i "futura cyber acronis"
 Confirmar em Painel de Controle > Programas e Recursos que "Futura Cyber Protection" está listado.

Confirmar que a GPO chegou no usuário:

gpresult /r /scope:user

Procurar "Acronis Antivirus" em "Objetos de política de grupo aplicados".

Relatório completo em HTML (mais fácil de ler):

gpresult /h C:\Windows\Temp\rsop.html /f
start C:\Windows\Temp\rsop.html

Ir em Configurações do Usuário > Scripts e confirmar o .exe com --quiet no Logon.

5. Via recomendada para produção (Inicialização / SYSTEM)

Mais robusta porque roda como SYSTEM (admin local pleno), não depende do privilégio do usuário logado e funciona em qualquer máquina, mesmo antes do logon. Foi provada funcionando via PsExec (code 0).

5.1 Script com log e verificação de "já instalado"

Notas do script:

O check de "já instalado" usa o serviço, não o código de saída - por isso a assertion cosmética do instalador não atrapalha.
O loop espera o arquivo no compartilhamento (não só ping), com limite para não travar o boot.
Grava log em C:\Windows\Temp\instala_acronis.log - fonte de verdade para saber se rodou.
5.2 Configuração da GPO

Para rodar já no 1º boot (desativa Fast Logon Optimization do Win11):

GPO > Configuração do Computador > Modelos Administrativos > Sistema > Logon > "Sempre aguardar a rede na inicialização e no logon do computador" > Habilitado.

(Opcional) Para o boot não esperar a instalação terminar antes da tela de login:

Modelos Administrativos > Sistema > Scripts > "Executar scripts de inicialização de modo assíncrono" > Habilitado. O boot vai direto para o login e a instalação corre em background.

5.3 Validação da via de Inicialização
gpupdate /force
shutdown /r /t 0

Após o boot:

type C:\Windows\Temp\instala_acronis.log
sc query state= all | findstr /i "AcronisAgent cyber-desktop futura"

Confirmar que o gatilho disparou (o ExecTime deixa de ser 0x0):

reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup\0\0" /v ExecTime
6. Comandos úteis de diagnóstico

Testar a instalação como SYSTEM (Sessão 0, exatamente como o boot faz), sem reiniciar:

PsExec64.exe -s -accepteula cmd
whoami
"\\Srv-files\gpo\antivirus\acronis\FuturaCyberProtection.exe" --quiet

whoami tem que mostrar nt authority\system.

Ver scripts registrados na máquina:

reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup" /s
reg query "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Logon" /s

(Script de computador fica em HKLM...\Startup; script de usuário fica em HKCU...\Logon.)

Verificar se a máquina enxerga o compartilhamento como conta de máquina (rodar dentro do shell SYSTEM do PsExec):

dir \\Srv-files\gpo\antivirus\acronis\

Verificar resquício de Acronis legado:

reg query "HKLM\SOFTWARE\Acronis" /s
sc query state= all | findstr /i "acronis"

Remover resquício de registro (somente se não houver instalação real, só chave órfã):

reg delete "HKLM\SOFTWARE\Acronis" /f
7. Observações e cuidados
Admin vs usuário comum: a via de Logon só é garantida se o usuário for admin local. Testar logado como admin dá falso positivo - pode parecer que funciona e quebrar nas máquinas dos funcionários comuns. A via de Inicialização (SYSTEM) não tem esse risco.
GPOs duplicadas: existiam 3 GPOs de Acronis conflitando (Acronis Antivirus, Deploy Acronis, GPO - Acronis Antivirus). Mantida apenas Acronis Antivirus. Conferir periodicamente se não voltaram duplicatas.
Sem MSI atribuído: garantir que Configuração do Computador > Configurações de Software > Instalação de Software esteja vazio na GPO, senão o MSI quebrado tenta instalar em paralelo e gera 1603.
Filtro WMI: a GPO "Acronis Antivirus" não tem filtro WMI (confere em GPO > aba Escopo > Filtragem WMI = "nenhum"). Se um dia for adicionado um filtro, conferir se a máquina-alvo casa com a consulta.
8. Status final
 Instalador .exe --quiet funciona manualmente (100%)
 Instalação como SYSTEM via PsExec (code 0, aparece em Programas e Recursos)
 Caminho do script corrigido (incluída a subpasta \acronis\)
 GPOs duplicadas removidas (mantida só "Acronis Antivirus")
 Resquício de Acronis legado limpo
 Logon de usuário com --quiet instalou automaticamente (ambiente de teste, usuário admin)
 Validar com usuário não-admin (definir se vai para produção via Logon ou via Inicialização)
 Implantar a via escolhida em todas as máquinas do domínio.

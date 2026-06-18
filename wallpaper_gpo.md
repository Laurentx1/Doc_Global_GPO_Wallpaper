# Configuração de LockScreen e Wallpaper via GPO

## Contexto

Configuração da imagem de tela de bloqueio (**LockScreen**) e papel de parede (**Wallpaper**) via GPO em máquinas **Windows 11 Pro** do domínio **global.log**.

A imagem aplicada é referente à campanha interna (**Festa Junina**).

---

# Problema Encontrado

O wallpaper da área de trabalho aplicava corretamente, mas o lockscreen continuava exibindo uma imagem antiga (**caminhão Global com trigo**), mesmo com o registro apontando para a imagem nova.

## Causa Raiz

Uma GPO antiga (já excluída) entregava a imagem antiga na pasta local:

```text
C:\temp\WallpaperLockScreen
```

A origem da imagem antiga vinha do caminho de servidor, também já desativado:

```text
\\192.168.0.9\compartilhados\gpo
```

A imagem antiga permanecia na pasta local das máquinas e era lida pelo Windows, sobrescrevendo a configuração nova.

---

# Solução Aplicada

Padronizar a entrega das imagens novas no mesmo caminho local que a GPO antiga utilizava:

```text
C:\temp\WallpaperLockScreen
```

Substituindo os arquivos antigos pelos novos e alinhando o registro para apontar para esse mesmo caminho.

---

# Configuração Final da GPO

## 1. Preferências > Arquivos (Cópia das Imagens para a Máquina)

Duas entregas, cada arquivo com seu próprio nome de destino:

| Origem | Destino |
|--------|----------|
| `\\srv-files\gpo\papeldeparede\backgroundScreen.jpg` | `C:\temp\WallpaperLockScreen\backgroundScreen.jpg` |
| `\\srv-files\gpo\papeldeparede\lockScreen.jpg` | `C:\temp\WallpaperLockScreen\lockScreen.jpg` |

**Ação:** `Substituir` em ambos.

> **IMPORTANTE:** Cada arquivo deve ter destino com seu próprio nome. Um erro comum foi apontar os dois para `lockScreen.jpg`, fazendo com que um sobrescrevesse o outro.

---

## 2. Preferências > Registro (3 Valores)

Todos com Hive:

```text
HKEY_LOCAL_MACHINE
```

Caminho da chave:

```text
SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP
```

| Nome do Valor | Tipo | Dados |
|---------------|-------|--------|
| LockScreenImageStatus | REG_DWORD | 1 |
| LockScreenImagePath | REG_SZ | `C:\temp\WallpaperLockScreen\lockScreen.jpg` |
| LockScreenImageUrl | REG_SZ | `C:\temp\WallpaperLockScreen\lockScreen.jpg` |

> Os três valores devem utilizar exatamente o mesmo caminho de chave e o mesmo caminho da imagem. Erros anteriores ocorreram porque o campo **Caminho da Chave** estava vazio nos valores **Path** e **Url**.

---

## 3. Modelos Administrativos (Opcional / Reforço)

```text
Configuração do Computador
└── Modelos Administrativos
    └── Painel de Controle
        └── Personalização
            └── "Forçar uma imagem padrão específica de tela de bloqueio e de logon"
                ├── Habilitado
                └── Caminho:
                    C:\temp\WallpaperLockScreen\lockScreen.jpg
```

> **OBSERVAÇÃO:** Esta política é oficialmente suportada apenas em Windows Enterprise/Education. No Windows 11 Pro, o método confiável é via **PersonalizationCSP** juntamente com a entrega do arquivo no caminho correto.

---

# Aplicação e Validação

Na máquina cliente:

```cmd
gpupdate /force
shutdown /r /t 0
```

Após o boot, validar:

```cmd
dir "C:\temp\WallpaperLockScreen\"
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\PersonalizationCSP"
```

Verificar também visualmente a tela de bloqueio (`Win + L`).

---

## Resultado Esperado na Pasta

```text
backgroundScreen.jpg
lockScreen.jpg
```

## Resultado Esperado no Registro

```text
LockScreenImageStatus  REG_DWORD  0x1
LockScreenImagePath    REG_SZ     C:\temp\WallpaperLockScreen\lockScreen.jpg
LockScreenImageUrl     REG_SZ     C:\temp\WallpaperLockScreen\lockScreen.jpg
```

---

# Observações para Atualizações Futuras

- Para trocar a imagem (nova campanha), basta substituir os arquivos em:

```text
\\srv-files\gpo\papeldeparede\
```

Mantendo os mesmos nomes:

- `backgroundScreen.jpg`
- `lockScreen.jpg`

A GPO realizará a entrega automaticamente.

- Não alterar os nomes dos arquivos, pois o registro aponta para nomes fixos.
- O Windows 11 Pro não suporta o método **Forçar Imagem** dos Modelos Administrativos de forma confiável. A combinação **Entrega de Arquivo + PersonalizationCSP** é o método que funciona.

---

# Checklist

- [ ] GPO de Arquivos com 2 entregas, cada uma com nome próprio de destino
- [ ] Registro com os 3 valores no caminho `PersonalizationCSP`
- [ ] Caminho da Chave preenchido nos 3 valores de registro
- [ ] Dados de valor apontando para `C:\temp\WallpaperLockScreen\lockScreen.jpg`
- [ ] `gpupdate /force` executado
- [ ] Máquina reiniciada
- [ ] Pasta `C:\temp\WallpaperLockScreen` com os 2 arquivos novos
- [ ] Tela de bloqueio exibindo a imagem correta (`Win + L`)

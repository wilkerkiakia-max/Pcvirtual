nome: PC criado pelo @alves7.pcm

sobre:
  despacho de fluxo de trabalho:
    entradas:
      número_da_instância:
        Descrição: 'Número da instância (1-10)'
        obrigatório: verdadeiro
        padrão: '1'
        tipo: string

empregos:
  implantar-rdp:
    executa em: windows-latest
    tempo limite em minutos: 360
    
    passos:
    - nome: Repositório de Checkout
      utiliza: ações/checkout@v4
      
    - nome: Informações do sistema
      executar: |
        Write-Host "=== IMPLANTAÇÃO ENIGMANO RDP ===" -ForegroundColor Verde
        Write-Host "Número da instância: ${{ github.event.inputs.instance_number }}" -ForegroundColor Yellow
        Write-Host "Executor: $env:RUNNER_NAME" -ForegroundColor Yellow
        Write-Host "Fluxo de trabalho: $env:GITHUB_WORKFLOW" -ForegroundColor Yellow
        
        Write-Host "`n=== ESPECIFICAÇÕES DO SISTEMA ===" -ForegroundColor Cyan
        Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, TotalPhysicalMemory
        Get-WmiObject -Class Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
        
    - nome: Habilitar RDP
      executar: |
        Write-Host "Habilitando a Área de Trabalho Remota..." -ForegroundColor Verde
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -Value 0
        Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 1
        
    - nome: Criar conta de usuário
      executar: |
        Write-Host "Configurando a conta de usuário..." -ForegroundColor Verde
        $Password = ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force
        Set-LocalUser -Name "runneradmin" -Password $Password
        Adicionar-LocalGroupMember -Grupo "Usuários da Área de Trabalho Remota" -Membro "runneradmin"
        Write-Host "Nome de usuário: runneradmin" -ForegroundColor Amarelo
        Write-Host "Senha: P@ssw0rd!" -ForegroundColor Amarelo
        
    - nome: Instalar Tailscale (Silencioso)
      executar: |
        Write-Host "Baixando Tailscale..." -ForegroundColor Verde
        $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
        $installerPath = "$env:TEMP\tailscale.msi"
        Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
        Write-Host "Instalando o Tailscale silenciosamente..." -ForegroundColor Verde
        Iniciar-Processo msiexec.exe -ArgumentList "/i `"$installerPath`" /quiet /norestart" -Wait
        Remover-Item $installerPath -Forçar
        Write-Host "Tailscale instalado com sucesso!" -ForegroundColor Verde
        
    - nome: Iniciar Escama de Cauda
      executar: |
        Write-Host "Iniciando a escama da cauda..." -ForegroundColor Verde
        & "C:\Program Files\Tailscale\tailscale.exe" up --authkey ${{ secrets.TAILSCALE_AUTH_KEY }} --hostname "enigmano-${{ github.run_id }}"
        Iniciar-Dormir -Segundos 15
        
    - nome: Obter IP do Tailscale
      executar: |
        $tsIP = & "C:\Program Files\Tailscale\tailscale.exe" ip -4
        Write-Host "IP do Tailscale: $tsIP" -ForegroundColor Verde
        echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV
        
    - nome: Instalar o Google Chrome (Direto)
      executar: |
        Write-Host "Baixando o Google Chrome..." -ForegroundColor Verde
        $chromeUrl = "https://dl.google.com/chrome/install/latest/chrome_installer.exe"
        $installerPath = "$env:TEMP\chrome_installer.exe"
        Invoke-WebRequest -Uri $chromeUrl -OutFile $installerPath
        Write-Host "Instalando o Google Chrome silenciosamente..." -ForegroundColor Verde
        Iniciar-Processo -CaminhoDoArquivo $caminhoDoInstalador -ListaDeArgumentos "/silencioso /instalar" -Aguardar
        Remover-Item $installerPath -Forçar
        Write-Host "Google Chrome instalado!" -ForegroundColor Verde
        
    - nome: Instalar o 7-Zip (Direto)
      executar: |
        Write-Host "Baixando 7-Zip..." -ForegroundColor Verde
        $zipUrl = "https://www.7-zip.org/a/7z2409-x64.exe"
        $installerPath = "$env:TEMP\7z-installer.exe"
        Invoke-WebRequest -Uri $zipUrl -OutFile $installerPath
        Write-Host "Instalando o 7-Zip silenciosamente..." -ForegroundColor Green
        Iniciar-Processo -CaminhoDoArquivo $caminhoDoInstalador -ListaDeArgumentos "/S" -Aguardar
        Remover-Item $installerPath -Forçar
        Write-Host "7-Zip instalado!" -ForegroundColor Verde
        
    - nome: Detalhes da conexão
      executar: |
        Write-Host "`n=== DETALHES DA CONEXÃO (TAILSCALE) ===" -ForegroundColor Verde
        Write-Host "╔═══════════════════════════════════════╗" -ForegroundColor Cyan
        Write-Host "║ ENIGMANO RDP ACCESS ║" -ForegroundColor Cyan  
        Write-Host "╠═══════════════════════════════════════╣" -ForegroundColor Cyan
        Write-Host "║ Host: $env:TAILSCALE_IP ║" -ForegroundColor White
        Write-Host "║ Nome de usuário: runneradmin ║" -ForegroundColor White
        Write-Host "║ Senha: P@ssw0rd! ║" -ForegroundColor White
        Write-Host "╚═══════════════════════════════════════╝" -ForegroundColor Cyan
        
        Write-Host "`nInstruções de conexão:" -ForegroundColor Amarelo
        Write-Host "1. Instale o Tailscale no seu dispositivo (celular, tablet, Mac, Linux)" -ForegroundColor White
        Write-Host "2. Faça login com a mesma conta Tailscale" -ForegroundColor White
        Write-Host "3. Abra o aplicativo Área de Trabalho Remota e insira o IP acima" -ForegroundColor White
        Write-Host "4. Use as credenciais fornecidas" -ForegroundColor White
        
    - nome: Manter Sessão Ativa
      executar: |
        Write-Host "`n=== SESSÃO ATIVA ===" -ForegroundColor Verde
        Write-Host "A instância RDP do EnigMano agora está em execução!" -ForegroundColor Cyan
        Write-Host "A instância permanecerá ativa por até 6 horas." -ForegroundColor Yellow
        Write-Host "Não feche esta janela para manter a conexão." -ForegroundColor Red
        
        $contador = 0
        enquanto ($verdadeiro) {
          $contador++
          Write-Host "Sessão ativa - Pulsação #$counter ($(Get-Date))" -ForegroundColor Verde
          
          $tsStatus = & "status de C:\Program Files\Tailscale\tailscale.exe"
          se ($tsStatus -match "ativo") {
            Write-Host "Status do Tailscale: ATIVO" -ForegroundColor Verde
          } outro {
            Write-Host "Status do Tailscale: ERRO - Reiniciando..." -ForegroundColor Vermelho
            & "C:\Program Files\Tailscale\tailscale.exe" up --authkey ${{ secrets.TAILSCALE_AUTH_KEY }} --hostname "enigmano-${{ github.run_id }}"
          }
          
          Iniciar-Dormir -Segundos 60
        }
